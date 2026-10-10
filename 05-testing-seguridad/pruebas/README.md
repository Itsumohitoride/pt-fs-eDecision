# 5. Pruebas

Las ocho son unitarias, con Jest y dobles del pool de Postgres y del proveedor de pago, para que corran en segundos y sin infraestructura. Las del router montan solo el router en una app de Express con supertest y un `OrderService` falso; las del servicio le pasan un pool cuyo `client.query` responde según el SQL.

1. Validación. `Idempotency-Key` ausente o que no es UUID, `items` vacío o con 51 productos, `quantity` 0, 100 o con decimales, y un SKU repetido devuelven 400 `VALIDATION_ERROR` sin llamar al servicio.
2. Cliente ajeno. Si `customerId` no es el del token, responde 403 y el servicio no se llama.
3. Mismo carrito en otro orden. Dos llamadas con los mismos items en distinto orden mandan el mismo hash al `INSERT` de `idempotency_keys`, así que cuentan como el mismo request.
4. Camino feliz. Verifica la secuencia: `BEGIN`, registro de la key, descuento de stock en orden de SKU, `COMMIT`, y que devuelve 201 con el total que calculó la base.
5. Sin stock. El `UPDATE inventory` devuelve `rowCount` 0, el servicio lanza 409 `OUT_OF_STOCK`, hace `ROLLBACK` y no crea líneas. Así se ve la última unidad desde el código.
6. SKU inexistente. El catálogo no trae uno de los SKU: 404 `SKU_NOT_FOUND`, `ROLLBACK` y ningún `UPDATE inventory`.
7. Idempotencia. Si la key ya existe con el mismo hash, devuelve la respuesta guardada sin crear orden; con otro hash, 409 `IDEMPOTENCY_KEY_REUSED`.
8. Pago. Si ya hay fila en `payments`, no llama al proveedor; si el proveedor falla, `ensurePaymentSession` devuelve `null` sin lanzar error.

La concurrencia real, dos transacciones sobre la última unidad, la resuelve la base con el `UPDATE` condicionado; con dobles se prueba cómo responde el código cuando la base dice que no alcanzó.
