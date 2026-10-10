# 7. Comunicación técnica

La decisión que considero más riesgosa es reservar el stock al crear la orden, durante 15 minutos, mientras el usuario paga.

El riesgo va en dos sentidos. En una campaña muchos abren el pago y lo abandonan, y esas unidades quedan bloqueadas aunque nadie las pague. Y si el pago falla o la reserva vence, el usuario pierde lo que tenía separado. Me pasó comprando boletos con asientos numerados: el pago falló y cuando volví a intentarlo mis asientos ya no estaban.

La alternativa que evalué fue descontar el stock solo al confirmar el pago. No bloquea unidades, pero permite vender algo que ya no existe si dos personas pagan a la vez. También pensé en una reserva más corta, que bloquea menos pero deja más casos como el mío.

Para confirmar la decisión y ajustar los 15 minutos necesitaría saber cuánto tarda un usuario en pagar, cuántos abandonan o reintentan después de un fallo, y cuánto stock tienen los productos de campaña. Con eso ajustaría la duración, o extendería la reserva cuando el pago falla y el usuario reintenta.
