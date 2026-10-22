# Decisiones pendientes para producto y frontend

Estas preguntas no bloquean los mocks de v1. Hasta resolverlas, las suposiciones del contrato son la referencia compartida.

| Tema | Suposición vigente | Decisión pendiente |
|---|---|---|
| Pedido tras volver de Checkout Pro | Mostrar `pendiente` y consultar cada 5 segundos hasta confirmación. | Texto y plazo antes de ofrecer ayuda si el proveedor demora. |
| Pedido en `requiere_atencion` | Mostrar estado y pedir hablar con personal; el administrador resuelve. | Flujo visual para sustituir producto o reembolsar. |
| Pérdida de acceso al pedido | El cliente conserva token durante la sesión; no se recupera con solo QR. | Si hace falta recuperación en el piloto, definir mecanismo seguro y su experiencia. |
| Producto agotado | Se deshabilita en carta, pero el pedido pagado conserva snapshot y pasa a incidencia. | Mensaje al cliente y opción de sustitución. |
