# 5. Incidente: órdenes duplicadas

Que solo pase con el proveedor lento apunta a reintentos: el usuario o algún componente reenvía el POST mientras el primero espera la sesión de pago. Lo confirmaría así, descartando una hipótesis a la vez.

1. Medir y acotar. Cuántas órdenes duplicadas hay, desde cuándo, de qué clientes y de qué app y versión vienen. Cruzo esas horas con la latencia del proveedor en las métricas: si los duplicados aparecen solo cuando la llamada pasa de unos segundos, la relación está confirmada.
2. Mirar los pares duplicados. Para cada par, busco qué `Idempotency-Key` trajo cada orden (`idempotency_keys` guarda la respuesta con el id) y sigo las dos requests en los logs por id de traza y por cliente.
3. Keys distintas. El cliente generó una nueva key al reintentar: una recarga de página, una versión vieja de la app que no la conserva, un botón que se reactiva tras el timeout o un cliente HTTP que reintenta solo y arma la key en cada intento. Reviso el código del cliente, su configuración de reintentos y si los duplicados se concentran en una versión.
4. Misma key y aun así dos órdenes. Algo está saltando la idempotencia: un endpoint o una versión del backend que no usa la tabla, o la key guardada fuera de la transacción de la orden. Reviso qué versión y qué tarea atendió cada request.
5. Reproducir y cerrar. En un ambiente de prueba agrego latencia al proveedor (un stub que tarda 15 segundos) y repito el flujo de la app hasta ver el duplicado. Con la causa confirmada, corrijo, agrego una prueba que cubra ese caso y una alarma por órdenes del mismo cliente con el mismo carrito en menos de un minuto.

Con las órdenes ya duplicadas, cancelo la que sobra, libero su stock y, si se cobraron las dos, reembolso una por el proveedor y aviso al cliente.
