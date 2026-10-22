# Modelo de datos y reglas del piloto

Diseño, sin migraciones ni endpoints implementados. Referencia pública: [`../contracts/api-v1.md`](../contracts/api-v1.md). Las decisiones aquí no modifican el contrato v1. Suposiciones del piloto: moneda ARS, sin impuestos/cargos/propinas separados, sin modificadores de producto, sin inventario numérico, sin reembolsos parciales y una sola sede operativa por bar. IDs opacos, generados en servidor; todos los importes son enteros en centavos.

## Entidades y relaciones

Se propone una base relacional transaccional (por ejemplo PostgreSQL). Cada tabla de negocio lleva `bar_id` incluso cuando puede inferirse por una relación; esto permite claves foráneas compuestas e índices por bar. Nunca se acepta `bar_id` suministrado por cliente o personal como autoridad.

| Entidad | Campos principales | Relaciones y restricciones |
|---|---|---|
| `bars` | `id`, `name`, `currency`, `active`, `created_at` | `currency = ARS` en v1. `id` PK. |
| `bar_tables` | `id`, `bar_id`, `label`, `qr_token_hash`, `active`, `created_at` | FK `bar_id → bars`; único `(bar_id, label)` entre mesas activas; `qr_token_hash` único global. El token real nunca se guarda en claro. |
| `menu_categories` | `id`, `bar_id`, `name`, `sort_order`, `active` | FK a bar; `UNIQUE(bar_id, id)`. |
| `products` | `id`, `bar_id`, `category_id`, `name`, `description`, `image_url`, `price_minor`, `available`, `active`, `updated_at` | FK compuesta `(bar_id, category_id) → menu_categories(bar_id, id)`; `price_minor >= 0`. `active=false` oculta producto, sin borrar snapshots. |
| `menu_revisions` | `bar_id`, `revision`, `updated_at` | Una fila por bar. Se incrementa atómicamente ante cambios de carta/precio/disponibilidad. |
| `staff_users` | `id`, `identity_subject`, `active` | Identidad global; `identity_subject` único. No guardar credenciales de proveedor en esta tabla. |
| `staff_memberships` | `id`, `bar_id`, `staff_user_id`, `role`, `active` | FK a bar y usuario; único `(bar_id, staff_user_id)`; rol `barra`, `salon`, `administrador`. El token de personal refleja membresía activa; validar también en servidor al escribir. |
| `quotes` | `id`, `bar_id`, `table_id`, `menu_revision`, `subtotal_minor`, `total_minor`, `expires_at`, `created_at` | FK compuesta mesa/bar; no reserva stock. TTL lógico 5 minutos. |
| `quote_items` | `quote_id`, `line_no`, `bar_id`, `product_id`, `name_snapshot`, `quantity`, `unit_price_minor`, `line_total_minor` | PK `(quote_id, line_no)`; producto del mismo bar; `quantity` 1..20; totales verificados por servidor. |
| `orders` | `id`, `bar_id`, `table_id`, `quote_id`, `public_code`, `total_minor`, `payment_state`, `service_state`, `version`, `created_at`, `updated_at`, `paid_at` | FK compuestas a mesa y cotización del mismo bar. `quote_id` único: una cotización no crea dos pedidos. `public_code` único por bar y jornada. `version >= 1`. |
| `order_items` | `order_id`, `line_no`, `bar_id`, `product_id`, `name_snapshot`, `quantity`, `unit_price_minor`, `line_total_minor` | Copia inmutable de `quote_items` al crear pedido. FK compuesta a pedido/bar; producto puede ser desactivado después. |
| `order_access_grants` | `id`, `bar_id`, `order_id`, `token_hash`, `created_at`, `revoked_at` | Token aleatorio de alta entropía entregado una vez. `token_hash` único global; comparación de hash en servidor. No se usa el QR como credencial para leer pedidos. |
| `payment_attempts` | `id`, `bar_id`, `order_id`, `provider`, `provider_preference_id`, `provider_payment_id`, `provider_state`, `checkout_url`, `checkout_expires_at`, `created_at`, `updated_at` | FK compuesta a pedido/bar. `provider = mercado_pago` en v1; IDs de proveedor únicos si no son nulos. Historial de varios intentos por pedido, nunca sobrescribir uno con otro. URL temporal protegida como dato sensible. |
| `payment_notifications` | `id`, `provider`, `provider_event_id`, `provider_payment_id`, `received_at`, `verified_at`, `processing_state`, `payload_digest` | Inbox técnico. Único `(provider, provider_event_id)` cuando el ID existe; deduplicación adicional por `(provider, provider_payment_id, payload_digest)` para notificaciones sin ID estable. Payload bruto con retención limitada fuera de vistas públicas. |
| `order_events` | `id`, `bar_id`, `order_id`, `sequence`, `kind`, `actor_type`, `actor_id`, `from_state`, `to_state`, `occurred_at`, `metadata` | Auditoría append-only. `UNIQUE(order_id, sequence)`. Actor `guest`, `staff`, `system` o `provider`; `metadata` sin secretos ni datos de tarjeta. El evento se escribe en la misma transacción que el cambio de pedido. |
| `idempotency_records` | `scope`, `actor_hash`, `key`, `request_digest`, `response_status`, `response_body`, `expires_at` | Único `(scope, actor_hash, key)`. 24 horas. El replay del alta puede devolver el mismo `orderAccessToken` cifrado hasta vencer el registro; después no se vuelve a emitir. |

Claves foráneas compuestas requieren `UNIQUE(bar_id, id)` en mesas, categorías, productos, cotizaciones y pedidos. `order_items` y `quote_items` también guardan `bar_id` y se comprueba que coincida con el padre y el producto. `CHECK` para cantidades, importes no negativos, estados permitidos y `line_total_minor = quantity * unit_price_minor`; total de cabecera se calcula dentro de la transacción, no se acepta del cliente. Evitar `ON DELETE CASCADE` en pedidos/pagos/eventos; las bajas operativas son lógicas.

## Índices iniciales

| Índice | Uso |
|---|---|
| `bar_tables(qr_token_hash)` único | Resolver QR sin barrer mesas. |
| `menu_categories(bar_id, active, sort_order)` y `products(bar_id, category_id, active, available)` | Carta pública del bar. |
| `quotes(bar_id, table_id, expires_at)` | Crear pedido y limpiar cotizaciones vencidas. |
| `orders(bar_id, service_state, updated_at DESC, id DESC)` | Bandeja de barra/salón y cursor estable. |
| `orders(bar_id, table_id, created_at DESC)` | Auditoría y soporte por mesa, nunca para acceso público por QR. |
| `orders(bar_id, payment_state, created_at)` | Conciliación de pagos. |
| `payment_attempts(bar_id, order_id, created_at DESC)` | Reintentos y conciliación. |
| `payment_attempts(provider, provider_preference_id)` y `(provider, provider_payment_id)` únicos parciales | Vincular callbacks sin ambigüedad. |
| `payment_notifications(processing_state, received_at)` | Reprocesar notificaciones pendientes. |
| `order_events(bar_id, order_id, sequence)` | Reconstrucción y auditoría de un pedido. |

Para la bandeja, se filtra `bar_id` y estados autorizados **en SQL** antes de paginar. Cursor firmado que codifica `(updated_at, id)` y filtros; paginación keyset. Las consultas públicas por `order_id` siempre exigen que el hash del token pertenezca al mismo pedido. La política de aislamiento se aplica en repositorios/consultas y puede reforzarse con row level security; pruebas entre dos bares son obligatorias antes de abrir el piloto.

## Flujo transaccional y snapshots

1. Resolver `tableToken` por hash y bar activo. `context` devuelve solo carta activa del mismo bar, incluida disponibilidad.
2. Cotizar: validar líneas distintas/cantidades/productos del bar; leer revisión y precios en una misma instantánea; guardar nombre, precio, cantidades y suma en `quotes`/`quote_items`. La cotización no reserva inventario.
3. Crear pedido: bloquear cotización, verificar mesa, expiración, revisión de carta y disponibilidad de cada producto. Si el precio o la revisión cambió, `QUOTE_STALE`; si un producto está agotado, `PRODUCT_UNAVAILABLE`. Copiar ítems inmutables y total. Insertar pedido `pendiente / bloqueado_pago`, token de acceso y evento. Unicidad de `quote_id` más idempotencia impide doble alta.
4. Checkout: crear o reutilizar un intento vigente para el mismo pedido pendiente. El acceso al proveedor se coordina con una marca de intento en curso para que dos solicitudes simultáneas no creen dos preferencias; se guarda resultado y URL al volver. No se expone preferencia de otro pedido.
5. Confirmación: únicamente una consulta autenticada al proveedor hecha por backend, disparada por webhook o reconciliación, autoriza `aprobado`. En una transacción bloqueada por pedido, aplicar estado, incrementar `version`, escribir eventos y hacerlo visible a personal. No se cambia el total ni los ítems tras crear el pedido.

`publicCode` sirve para mostrar/llamar el pedido; no es secreto ni autoriza lectura. Las cantidades de la carta son ilustrativas: `available` es una señal de aceptación, no inventario reservado. El sistema no conserva datos de tarjeta.

## Estados y actores

`payment_state` es el resultado agregado del pedido. Un intento `rechazado` deja el pedido en `pendiente` para permitir reintento; el rechazo queda en `payment_attempts`. No hay estado público `rechazado` de pedido. `service_state` describe cumplimiento y nunca se deduce únicamente de la vuelta del navegador.

| Estado origen de pago | Destino | Actor / condición |
|---|---|---|
| `pendiente` | `aprobado` | Sistema, tras verificar pago exacto por monto/moneda/bar/pedido con proveedor. |
| `pendiente` | `cancelado` | Sistema o administrador, solo si no hay pago aprobado y se cierra el pedido. |
| `aprobado` | `reembolsado` | Sistema, tras confirmar reembolso total en proveedor. |
| `aprobado` | `contracargo` | Sistema, tras confirmación del proveedor. |
| `cancelado` | `aprobado` | Reconciliación excepcional de pago tardío: bloquear servicio, abrir incidencia y gestionar devolución. |
| `reembolsado` / `contracargo` | otro | Sin transición automática; cualquier corrección requiere evento administrativo auditado y diseño posterior. |

| Estado origen de servicio | Destino | Actor / condición |
|---|---|---|
| `bloqueado_pago` | `en_cola` | Sistema al aprobar pago y verificar disponibilidad actual de todos los productos. |
| `bloqueado_pago` | `requiere_atencion` | Sistema al aprobar pago con un producto ya agotado. |
| `bloqueado_pago` | `cancelado` | Sistema al cancelar pedido sin cobro. |
| `cancelado` | `requiere_atencion` | Sistema únicamente si se confirma tarde un pago de un pedido ya cancelado; se gestiona devolución y no entra a barra. |
| `en_cola` | `en_preparacion` | Barra o administrador, solo con pago `aprobado`. |
| `en_preparacion` | `listo` | Barra o administrador, solo con pago `aprobado`. |
| `listo` | `entregado` | Salón o administrador, solo con pago `aprobado`. |
| `en_cola` / `en_preparacion` / `listo` | `requiere_atencion` | Administrador o sistema ante incidencia de cumplimiento; la barra solicita intervención al administrador fuera de v1. |
| `requiere_atencion` | `en_cola` | Administrador, solo si el pedido original puede cumplirse y el pago sigue aprobado. No se editan ítems. |
| `requiere_atencion` / `en_cola` / `en_preparacion` / `listo` | `cancelado` | Administrador o sistema después de reembolso total confirmado; una cancelación con pago aprobado sin devolución no se considera resuelta. |
| `entregado` | otro | Sin transición de servicio en v1; reembolso/contracargo financiero no borra la entrega histórica. |

En `PATCH service-state`, el backend bloquea pedido y compara `expectedVersion`; si difiere, `VERSION_CONFLICT` y frontend recarga. Rechaza transición fuera de la tabla (`INVALID_TRANSITION`) o preparación/entrega sin `payment_state=aprobado` (`PAYMENT_NOT_CONFIRMED`). `administrador` puede ejecutar las transiciones manuales de incidencia; no puede crear un aprobado. La confirmación de pago y el paso a cola/incidencia son atómicos, por eso barra nunca ve `en_cola` con pago pendiente. Un reembolso o contracargo confirmado retira el pedido de la bandeja activa; si aún no estaba entregado, queda `cancelado` tras el reembolso o `requiere_atencion` ante contracargo para investigación administrativa.

## Eventos, duplicados y orden de llegada

- Verificar firma/origen del webhook según el mecanismo del proveedor, registrar inbox y responder rápido. El cuerpo del webhook es una pista; consultar estado canónico de pago al proveedor con credenciales del bar antes de cambiar el pedido.
- Duplicados se reconocen por ID de evento y/o huella de pago. Un reenvío ya procesado responde éxito sin escribir otro `order_event` ni volver a habilitar barra.
- Notificaciones fuera de orden se resuelven con el estado canónico actual, no con el orden de llegada. Por ejemplo, un aviso `pending` recibido después de `approved` no retrocede `aprobado` a `pendiente`. Reconciliación periódica revisa intentos pendientes y notificaciones fallidas.
- `provider_payment_id` es único. Si dos pagos distintos resultan aprobados para el mismo pedido, solo el primero mueve el pedido a `aprobado` y a la cola. El segundo se registra como cobro duplicado en eventos/incidencia financiera y se inicia devolución o revisión manual; jamás genera otro pedido ni segunda preparación.
- Validar que monto y moneda del proveedor coincidan exactamente con snapshot de pedido, y que referencia externa/preferencia apunten al bar y pedido correctos. Una discrepancia queda en cuarentena para conciliación, sin habilitar servicio.
- Cada cambio visible aumenta `orders.version` y registra `order_events.sequence` en la misma transacción. Un outbox transaccional sería la evolución si se agregan avisos push; el piloto se apoya en consulta periódica.

## Agotados, reconexión y seguridad de acceso

Si un producto pasa a `available=false` **antes de crear** el pedido, se rechaza la creación y se vuelve a cotizar. Si pasa a agotado **después del cobro y antes de preparar**, se conserva el pedido/snapshot y se marca `requiere_atencion`; no se sustituye ni se recalcula el total silenciosamente. El administrador confirma si aún puede cumplir el ítem original; si no, tramita reembolso total por el canal operativo y el sistema cancela el servicio al confirmar ese reembolso. Al cambiar disponibilidad a falso, se inspeccionan pedidos pagados en `en_cola` que contienen el producto. Los pedidos `en_preparacion` se revisan con barra antes de declararlos afectados. Esto supone que `available=false` significa que no pueden iniciarse nuevas preparaciones del producto.

Tras reconectar, invitado y personal hacen `GET` de su vista autorizada y reemplazan el estado local por la versión del servidor. Frontend no reconstruye estados desde la URL de retorno de Mercado Pago. El invitado conserva `orderAccessToken` en sesión, con manejo de borrado al cerrar sesión/pestaña según plataforma; si lo pierde, no puede recuperar el pedido solo con el QR. El personal recarga bandeja al volver a foco y usa `expectedVersion` en cambios. El sondeo de cliente indicado en v1 (5 segundos con página visible) es suficiente para el piloto; se puede agregar SSE después sin romper este contrato.

En todas las rutas se aplica rate limit por IP/token y se ocultan pedidos de otro bar. Tokens y URLs de checkout se redactan de logs y trazas. Personal solo ve pedidos del bar de su membresía; barra y salón ven únicamente los que tuvieron pago aprobado, sin credenciales ni datos del proveedor. Las operaciones administrativas sobre cobros/reembolsos y soporte de recuperación de token requieren un contrato futuro; este diseño no las presenta como endpoints existentes.
