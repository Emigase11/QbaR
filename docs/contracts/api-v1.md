# Contrato de API v1 — piloto de pedidos en bares

Estado: **publicado para mocks de frontend**. Dueño: backend. Fecha: 2026-10-22. Base: `/api/v1`. El piloto usa un bar, pero todas las operaciones y credenciales se aíslan por bar. Este documento fija nombres de rutas, campos, estados y formato de errores; cualquier cambio posterior se registra en `CHANGELOG.md`.

## Convenciones y acceso

- JSON UTF-8; nombres de campos en `camelCase`; IDs opacos en string; fechas RFC 3339 en UTC. Importes en unidades menores de ARS (`amountMinor`: centavos enteros), nunca números decimales. `currency` vale `ARS` en v1.
- El QR contiene una URL del frontend con `tableToken` opaco, por ejemplo `https://app.example.com/t/tbl_qr_7F2a`. El token identifica la mesa y permite leer contexto/carta, cotizar y crear pedidos en ella. No autentica al personal ni permite consultar pedidos ajenos. Se debe servir solo por HTTPS y no registrar el token completo en logs.
- Crear un pedido devuelve `orderAccessToken` **una sola vez**. El cliente lo guarda en almacenamiento de sesión y lo usa como `Authorization: Bearer <orderAccessToken>` para checkout y estado. La URL compartible del pedido debe llevar un secreto separado equivalente; v1 no ofrece recuperación si se pierden ambos. No incluir el token en analítica ni en URLs de terceros.
- El personal usa `Authorization: Bearer <staffAccessToken>` emitido por el sistema de acceso del personal, con `barId` y rol `barra`, `salon` o `administrador`. La emisión/renovación del token queda fuera de v1. El backend deriva bar y rol del token; nunca confía en ellos si vienen en el cuerpo.
- Todas las operaciones de escritura aceptan `Idempotency-Key` cuando se indica. Es un UUID generado por cliente, único por operación y actor durante 24 horas. Repetir clave y cuerpo devuelve el mismo resultado; repetir clave con otro cuerpo da `409 IDEMPOTENCY_KEY_REUSED`.
- No hay estados de servicio disponibles para barra antes de pago confirmado por el backend. La redirección de Checkout Pro **no confirma** el pago. El estado puede demorar en reflejar una notificación; frontend consulta el pedido al volver del checkout.

## Tipos públicos

```ts
type Money = { currency: "ARS"; amountMinor: number }; // entero >= 0
type EstadoPago = "pendiente" | "aprobado" | "reembolsado" | "contracargo" | "cancelado";
type EstadoServicio =
  | "bloqueado_pago" | "en_cola" | "en_preparacion"
  | "listo" | "entregado" | "requiere_atencion" | "cancelado";

type Producto = {
  id: string;
  name: string;
  description: string | null;
  imageUrl: string | null;
  categoryId: string;
  price: Money;
  available: boolean;
};
type Carrito = {
  items: Array<{ productId: string; quantity: number }>;
}; // 1..30 renglones distintos; quantity 1..20; sin duplicar productId
type Quote = {
  id: string;
  tableId: string;
  items: Array<{
    productId: string; name: string; quantity: number;
    unitPrice: Money; lineTotal: Money;
  }>;
  subtotal: Money;
  total: Money;
  expiresAt: string;
};
type Pedido = {
  id: string;
  publicCode: string;
  barId: string;
  tableId: string;
  items: Quote["items"];
  total: Money;
  paymentState: EstadoPago;
  serviceState: EstadoServicio;
  createdAt: string;
  updatedAt: string;
  version: number;
};
type ErrorAPI = {
  error: {
    code: string;
    message: string;
    details?: Array<{ field: string; reason: string }>;
    requestId: string;
  };
};
```

`Producto.available` refleja la disponibilidad actual. El precio y nombre dentro de `Quote` y `Pedido` son snapshots y pueden diferir de la carta actual. `Pedido.version` aumenta con cada cambio visible; ayuda a evitar escrituras concurrentes y a refrescar pantallas.

### Visibilidad

| Dato | Invitado con QR | Invitado con token de pedido | Barra | Salón | Administrador |
|---|---:|---:|---:|---:|---:|
| Bar, mesa, categorías y productos públicos | Sí | Sí | Sí | Sí | Sí |
| Cotización y pedido propio, estados, total, código | Puede crear | Sí | Solo pedidos del bar con pago aprobado | Solo pedidos del bar con pago aprobado | Pedidos del bar |
| `orderAccessToken` | Solo al crear | No se vuelve a emitir | No | No | No |
| URL de checkout | No | Pedido propio | No | No | No |
| Intentos de pago, IDs de proveedor, datos de personal, auditoría | No | No | No | No | Solo vistas administrativas futuras |
| Cambio de estado / disponibilidad | No | No | Preparación | Entrega | Ambos y disponibilidad |

La vista de personal nunca devuelve `orderAccessToken` ni la URL de checkout. La vista pública nunca devuelve proveedor de pagos, datos de tarjeta, identidad de personal ni notas internas.

## Operaciones

Los ejemplos siguen el mismo bar `bar_1`, mesa `mesa_12`, producto `prod_cafe_1`, cotización `quo_1` y pedido `ord_1`. Las respuestas de error usan siempre `ErrorAPI`.

### 1. Obtener bar, mesa y carta

`GET /api/v1/public/tables/{tableToken}/context` — permiso: poseer el QR. Sin cuerpo. Respuesta `200`:

```json
{
  "bar": { "id": "bar_1", "name": "Bar Central", "currency": "ARS" },
  "table": { "id": "mesa_12", "label": "Mesa 12" },
  "menu": {
    "revision": 7,
    "categories": [{ "id": "cat_cafes", "name": "Cafés", "sortOrder": 1 }],
    "products": [{ "id": "prod_cafe_1", "name": "Café con leche", "description": "Taza grande", "imageUrl": null, "categoryId": "cat_cafes", "price": { "currency": "ARS", "amountMinor": 350000 }, "available": true }]
  }
}
```

Errores: `404 TABLE_NOT_FOUND` (token inválido o mesa desactivada), `503 BAR_UNAVAILABLE` (bar inactivo). Se evita distinguir tokens inexistentes de mesas desactivadas.

### 2. Cotizar y validar carrito

`POST /api/v1/public/tables/{tableToken}/quotes` — permiso: poseer el QR. Solicitud `Carrito`:

```json
{ "items": [{ "productId": "prod_cafe_1", "quantity": 2 }] }
```

Respuesta `201`:

```json
{
  "id": "quo_1", "tableId": "mesa_12",
  "items": [{ "productId": "prod_cafe_1", "name": "Café con leche", "quantity": 2, "unitPrice": { "currency": "ARS", "amountMinor": 350000 }, "lineTotal": { "currency": "ARS", "amountMinor": 700000 } }],
  "subtotal": { "currency": "ARS", "amountMinor": 700000 },
  "total": { "currency": "ARS", "amountMinor": 700000 },
  "expiresAt": "2026-10-22T15:10:00Z"
}
```

La cotización dura 5 minutos y no reserva stock. v1 no incluye cargos, descuentos ni propinas; por eso `subtotal = total`. Errores: `400 INVALID_CART` (vacío, cantidades/renglones fuera de rango o duplicados), `404 TABLE_NOT_FOUND`, `409 PRODUCT_UNAVAILABLE` (no existe en ese bar o está agotado), `503 BAR_UNAVAILABLE`.

### 3. Crear pedido pendiente de pago

`POST /api/v1/public/tables/{tableToken}/orders` — permiso: poseer el QR. Requiere `Idempotency-Key`. Solicitud:

```json
{ "quoteId": "quo_1" }
```

Respuesta `201` (repetición idempotente: `200` con el mismo cuerpo):

```json
{
  "order": {
    "id": "ord_1", "publicCode": "A104", "barId": "bar_1", "tableId": "mesa_12",
    "items": [{ "productId": "prod_cafe_1", "name": "Café con leche", "quantity": 2, "unitPrice": { "currency": "ARS", "amountMinor": 350000 }, "lineTotal": { "currency": "ARS", "amountMinor": 700000 } }],
    "total": { "currency": "ARS", "amountMinor": 700000 },
    "paymentState": "pendiente", "serviceState": "bloqueado_pago",
    "createdAt": "2026-10-22T15:06:00Z", "updatedAt": "2026-10-22T15:06:00Z", "version": 1
  },
  "orderAccessToken": "oat_example_secret"
}
```

Antes de crear, el servidor comprueba vigencia de la cotización y disponibilidad actual. El total del pedido usa el snapshot cotizado; si cambió el precio o la carta, devuelve `409 QUOTE_STALE` y el cliente debe cotizar de nuevo. Errores: `400 INVALID_REQUEST`, `400 IDEMPOTENCY_KEY_REQUIRED`, `404 TABLE_NOT_FOUND`, `404 QUOTE_NOT_FOUND` (incluye cotización de otra mesa), `409 QUOTE_EXPIRED`, `409 QUOTE_STALE`, `409 PRODUCT_UNAVAILABLE`, `409 IDEMPOTENCY_KEY_REUSED`, `503 BAR_UNAVAILABLE`.

### 4. Obtener URL de Checkout Pro

`POST /api/v1/guest/orders/{orderId}/checkout` — permiso: token del pedido propio. Requiere `Idempotency-Key`; cuerpo `{}`. Respuesta `200`:

```json
{
  "orderId": "ord_1",
  "checkoutUrl": "https://www.mercadopago.com.ar/checkout/v1/redirect?pref_id=example",
  "expiresAt": "2026-10-22T15:21:00Z"
}
```

El backend puede reutilizar un intento vigente. `checkoutUrl` es una URL HTTPS temporal de Mercado Pago; frontend la abre en navegación completa. No infiere pago aprobado de la vuelta. Errores: `400 IDEMPOTENCY_KEY_REQUIRED`, `401 UNAUTHORIZED`, `403 ORDER_FORBIDDEN`, `404 ORDER_NOT_FOUND`, `409 ORDER_NOT_PAYABLE` (aprobado, cancelado o expirado), `409 IDEMPOTENCY_KEY_REUSED`, `502 CHECKOUT_UNAVAILABLE`.

### 5. Consultar pedido propio

`GET /api/v1/guest/orders/{orderId}` — permiso: token del pedido propio. Sin cuerpo. Respuesta `200`:

```json
{
  "order": {
    "id": "ord_1", "publicCode": "A104", "barId": "bar_1", "tableId": "mesa_12",
    "items": [{ "productId": "prod_cafe_1", "name": "Café con leche", "quantity": 2, "unitPrice": { "currency": "ARS", "amountMinor": 350000 }, "lineTotal": { "currency": "ARS", "amountMinor": 700000 } }],
    "total": { "currency": "ARS", "amountMinor": 700000 },
    "paymentState": "pendiente", "serviceState": "bloqueado_pago",
    "createdAt": "2026-10-22T15:06:00Z", "updatedAt": "2026-10-22T15:06:00Z", "version": 1
  }
}
```

Tras pago confirmado, el mismo pedido pasa a `paymentState: "aprobado"`, `serviceState: "en_cola"` y `version: 2`. El cliente consulta al volver del checkout y luego cada 5 segundos mientras la página esté visible; detiene el sondeo al llegar a `entregado` o `cancelado`. Errores: `401 UNAUTHORIZED`, `403 ORDER_FORBIDDEN`, `404 ORDER_NOT_FOUND`.

### 6. Bandeja de pedidos del personal

`GET /api/v1/staff/orders?serviceState=en_cola&limit=20&cursor=...` — permiso: `barra`, `salon` o `administrador`; el bar se toma del token. `serviceState` es opcional y acepta cualquiera de los estados de servicio salvo `bloqueado_pago`; `limit` 1..100, por defecto 20; `cursor` opaco. Respuesta `200`:

```json
{
  "orders": [{
    "id": "ord_1", "publicCode": "A104", "barId": "bar_1", "tableId": "mesa_12", "tableLabel": "Mesa 12",
    "items": [{ "productId": "prod_cafe_1", "name": "Café con leche", "quantity": 2, "unitPrice": { "currency": "ARS", "amountMinor": 350000 }, "lineTotal": { "currency": "ARS", "amountMinor": 700000 } }],
    "total": { "currency": "ARS", "amountMinor": 700000 }, "paymentState": "aprobado", "serviceState": "en_cola",
    "createdAt": "2026-10-22T15:06:00Z", "updatedAt": "2026-10-22T15:07:00Z", "version": 2
  }],
  "nextCursor": null
}
```

Por defecto lista estados activos `en_cola`, `en_preparacion`, `listo` y `requiere_atencion`, ordenados por `updatedAt` descendente. Nunca devuelve pedidos sin pago aprobado ni de otro bar. `tableLabel` es una ampliación exclusiva de la vista de personal. Errores: `400 INVALID_QUERY`, `401 UNAUTHORIZED`, `403 ROLE_FORBIDDEN`.

### 7. Cambiar estado de preparación o entrega

`PATCH /api/v1/staff/orders/{orderId}/service-state` — permiso según transición: `barra` cambia `en_cola → en_preparacion → listo`; `salon` cambia `listo → entregado`; `administrador` puede esas transiciones y las de incidencia/cancelación. Solicitud:

```json
{ "targetState": "en_preparacion", "expectedVersion": 2 }
```

Respuesta `200`: `{ "order": { ...Pedido, "paymentState": "aprobado", "serviceState": "en_preparacion", "version": 3 } }`. El objeto completo sigue el tipo `Pedido` y conserva los valores del ejemplo anterior. El estado de incidencia se escribe `requiere_atencion`. Errores: `400 INVALID_REQUEST`, `401 UNAUTHORIZED`, `403 ROLE_FORBIDDEN`, `404 ORDER_NOT_FOUND` (también otro bar), `409 INVALID_TRANSITION`, `409 PAYMENT_NOT_CONFIRMED`, `409 VERSION_CONFLICT`.

### 8. Cambiar disponibilidad de producto

`PATCH /api/v1/staff/products/{productId}/availability` — permiso: `administrador` del bar. Solicitud:

```json
{ "available": false }
```

Respuesta `200`:

```json
{ "product": { "id": "prod_cafe_1", "name": "Café con leche", "description": "Taza grande", "imageUrl": null, "categoryId": "cat_cafes", "price": { "currency": "ARS", "amountMinor": 350000 }, "available": false }, "menuRevision": 8 }
```

La revisión sube solo si cambió el valor. No altera pedidos existentes. Errores: `400 INVALID_REQUEST`, `401 UNAUTHORIZED`, `403 ROLE_FORBIDDEN`, `404 PRODUCT_NOT_FOUND` (también otro bar).

## Errores comunes

Estados HTTP: `400` solicitud inválida; `401` falta credencial o expiró; `403` credencial válida sin permiso; `404` recurso no visible; `409` conflicto de estado; `429` límite de frecuencia; `502` proveedor de checkout no disponible; `503` bar o servicio no disponible. Todos usan el mismo sobre:

```json
{
  "error": {
    "code": "PRODUCT_UNAVAILABLE",
    "message": "Hay productos no disponibles; actualizá el carrito.",
    "details": [{ "field": "items[0].productId", "reason": "unavailable" }],
    "requestId": "req_8e71"
  }
}
```

`message` es para mostrar o registrar y puede cambiar; frontend debe ramificar por `code`. `details` es opcional. `429` y `503` pueden incluir `Retry-After` en segundos. Los errores de autenticación no exponen si existe un pedido ajeno.

## Notas para mocks e integración

El flujo mínimo es `context → quotes → orders → checkout → GET order`. El mock puede simular la confirmación cambiando simultáneamente `paymentState` a `aprobado` y `serviceState` a `en_cola`, con `version` incrementada. La bandeja del personal empieza a mostrar el pedido en ese momento. La interfaz debe tolerar `requiere_atencion`, `reembolsado` y `contracargo` aunque no ocurran en el camino feliz del piloto. Las decisiones abiertas de interacción se anotan en `pending-decisions.md`.
