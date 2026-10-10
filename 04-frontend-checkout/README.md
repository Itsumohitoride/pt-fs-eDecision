# 4. Checkout

Asumo una app React que llama a `POST /api/v1/orders`. Supuesto: existe un `GET /api/v1/orders/:id` para que la app consulte el estado de la orden.

## 28. Estados

- Idle: el formulario está listo y el botón "Pagar" activo.
- Validando: reviso en el cliente la dirección y el carrito. Si falta algo, vuelvo a idle con el error junto a cada campo.
- Enviando: el POST está en curso. El botón dice "Procesando pago..." y no se puede presionar.
- Verificando: se cortó la red o pasó el tiempo límite, y no sé si la orden se creó. Muestro "Estamos confirmando tu orden".
- Redirigiendo al pago: llegó la orden con `paymentUrl` y llevo al usuario a la página del proveedor.
- Confirmado: al volver del proveedor, la app consulta la orden hasta verla pagada y muestra el resumen.
- Error recuperable: falla de red, 500 o `paymentUrl: null`. Ofrezco "Reintentar", que repite el POST con la misma key.
- Error definitivo: 400, 404 o 409 sin stock. Reintentar daría lo mismo, así que llevo al usuario al carrito con el mensaje del backend para que cambie algo.

## 29. Doble clic

El botón queda deshabilitado mientras envía. Eso no alcanza, porque dos clics muy seguidos pueden salir antes de que la pantalla se actualice. Lo que evita la orden duplicada es la `Idempotency-Key`: la genero al abrir el checkout y la reutilizo en cada reintento. Aunque lleguen dos POST, el backend crea una sola orden y devuelve la misma a ambos.

Para no dejar al usuario bloqueado, corto la request a los 15 segundos. El backend tarda como mucho 4 s, y la llamada al proveedor de pago tiene su propio límite de 10 s, así que una respuesta lenta pero válida no se corta. Al cortarse paso a verificando, con la opción de reintentar.

Si el usuario vuelve al carrito y lo cambia, descarto la key y al abrir el checkout se genera otra. Con la misma key y otro carrito el backend respondería 409 `IDEMPOTENCY_KEY_REUSED`.

## 30. Corte de red después del POST

Paso a verificando y no muestro "falló", porque la orden puede existir. Cuando vuelve la conexión, o si el usuario toca "Reintentar", repito el mismo POST con la misma key. Si la orden ya se había creado, el backend devuelve la guardada y sigo hacia el pago; si no, la crea ahora. En los dos casos queda una sola orden.

La key se guarda en sessionStorage. Si el usuario recarga la página en medio del corte, el checkout recupera la misma key y no crea otra orden.

## 31. Accesibilidad

- Cada campo tiene su etiqueta visible, no solo un texto de ejemplo dentro del campo.
- El error se escribe junto al campo, dice qué corregir y se destaca con otro estilo de letra, no solo con color.
- Si el envío falla, el foco va al primer campo con error.
- Los mensajes de estado, como "Procesando pago...", se anuncian a los lectores de pantalla.
- "Pagar" es un botón real con texto claro, que se puede usar con teclado.

## 32. Estado

Local: los campos del formulario, sus errores y el estado del checkout.

Global: el carrito y la sesión del usuario, por ejemplo en un context. Los usan el encabezado, el carrito y el checkout, y tienen que coincidir en todos lados.

Server state: productos, precios, stock y estado de la orden. Vienen del backend y pueden cambiar mientras el usuario mira la pantalla, así que no los guardo en el context. Los leo con hooks propios que manejan carga, error y cancelación. El total que se ve antes de pagar es una referencia: el que vale es el que calcula el backend al crear la orden, y es el que muestro después.
