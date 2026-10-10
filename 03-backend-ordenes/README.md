# 3. POST /api/v1/orders

Asumo Express 4 y PostgreSQL con `pg`, como en el 1.1. El middleware de autenticación deja el cliente en `req.user.customerId` y responde 401 si no hay token válido. El precio sale de una tabla `products (sku, price)` que ya existe, nunca del request. El pago sigue el flujo del punto 2.

## 21. Validaciones

`Idempotency-Key` obligatoria y con formato UUID; `items` con 1 a 50 elementos; cada `sku` un string no vacío, cada `quantity` un entero entre 1 y 99 y sin SKU repetidos. Si algo falla, 400. Además, `customerId` debe coincidir con `req.user.customerId` o respondo 403: es el mismo IDOR del 1.1. Si el SKU existe y si hay stock solo lo sabe la base, así que eso se revisa en la transacción.

`orders.router.js`

```js
const { Router } = require('express');

const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

function validate(key, body) {
  if (!UUID.test(key || '')) return 'Idempotency-Key debe ser un UUID';
  const items = body && body.items;
  if (!Array.isArray(items) || items.length < 1 || items.length > 50) return 'items debe tener entre 1 y 50 productos';
  for (const item of items) {
    if (!item || typeof item.sku !== 'string' || item.sku.trim() === '') return 'Cada producto necesita un sku';
    if (!Number.isInteger(item.quantity) || item.quantity < 1 || item.quantity > 99) {
      return `La cantidad de ${item.sku} debe ser un entero entre 1 y 99`;
    }
  }
  if (new Set(items.map((i) => i.sku)).size !== items.length) return 'Hay productos repetidos';
  return null;
}

function ordersRouter(service) {
  const router = Router();

  router.post('/', async (req, res, next) => {
    const key = req.get('Idempotency-Key');
    const invalid = validate(key, req.body);
    if (invalid) return res.status(400).json({ error: { code: 'VALIDATION_ERROR', message: invalid } });
    if (req.body.customerId !== req.user.customerId) {
      return res.status(403).json({ error: { code: 'FORBIDDEN', message: 'No puedes crear órdenes para otro cliente' } });
    }

    try {
      const { status, order } = await service.createOrder(req.user.customerId, key, req.body.items);
      const paymentUrl = await service.ensurePaymentSession(order);
      res.status(status).json({ ...order, paymentUrl });
    } catch (err) {
      if (!err.status) return next(err);
      res.status(err.status).json({ error: { code: err.code, message: err.message } });
    }
  });

  return router;
}

module.exports = { ordersRouter };
```

Los errores sin `status` caen en el error handler global del 1.1, que responde 500.

## 22. Stock sin negativos

Cada item se descuenta con `UPDATE inventory SET available = available - $2 WHERE sku = $1 AND available >= $2`; si no afecta filas, rollback y 409. Si dos clientes compran la última unidad a la vez, el segundo UPDATE espera el lock de la fila, la relee cuando el primero hace commit, ve `available = 0` y no afecta filas. Los items van ordenados por SKU para que dos carritos con productos en común bloqueen filas en el mismo orden y no haya deadlock. `CHECK (available >= 0)` es la segunda barrera.

## 23. Transacción

Empieza después de validar y termina antes de llamar al proveedor de pago, así que adentro no hay llamadas de red y los locks duran milisegundos:

1. Registrar la `Idempotency-Key` (o devolver la respuesta guardada).
2. Leer los precios del catálogo.
3. Crear la orden y, por cada item, descontar stock, crear la línea y la reserva de 15 minutos.
4. Calcular el total y guardar la respuesta en la key.
5. `COMMIT`.

Cualquier error hace `ROLLBACK` y no queda nada. Uso una clase porque aquí sí hay lógica de negocio y dos dependencias que conviene reemplazar en los tests.

`order.service.js`

```js
const crypto = require('crypto');

class OrderService {
  constructor(pool, paymentProvider) {
    this.pool = pool;
    this.paymentProvider = paymentProvider;
  }

  async createOrder(customerId, idempotencyKey, items) {
    // Mismo orden de bloqueo en todas las transacciones para evitar deadlocks.
    const sorted = items
      .map((i) => ({ sku: i.sku, quantity: i.quantity }))
      .sort((a, b) => (a.sku < b.sku ? -1 : 1));
    const hash = crypto.createHash('sha256').update(JSON.stringify({ customerId, sorted })).digest('hex');

    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      // Si otra request con la misma key sigue en curso, este INSERT espera a que termine.
      const inserted = await client.query(
        `INSERT INTO idempotency_keys (customer_id, key, request_hash)
         VALUES ($1, $2, $3) ON CONFLICT DO NOTHING`,
        [customerId, idempotencyKey, hash],
      );
      if (inserted.rowCount === 0) {
        const { rows } = await client.query(
          'SELECT request_hash, response_status, response_body FROM idempotency_keys WHERE customer_id = $1 AND key = $2',
          [customerId, idempotencyKey],
        );
        if (rows[0].request_hash !== hash) {
          throw { status: 409, code: 'IDEMPOTENCY_KEY_REUSED', message: 'La Idempotency-Key ya se usó con otro carrito' };
        }
        await client.query('ROLLBACK');
        return { status: rows[0].response_status, order: rows[0].response_body };
      }

      const products = await client.query('SELECT sku, price FROM products WHERE sku = ANY($1)', [
        sorted.map((i) => i.sku),
      ]);
      const prices = new Map(products.rows.map((p) => [p.sku, p.price]));
      const missing = sorted.find((i) => !prices.has(i.sku));
      if (missing) throw { status: 404, code: 'SKU_NOT_FOUND', message: `No existe el producto ${missing.sku}` };

      const created = await client.query(
        `INSERT INTO orders (customer_id, status, total) VALUES ($1, 'pending_payment', 0) RETURNING id`,
        [customerId],
      );
      const orderId = created.rows[0].id;

      for (const { sku, quantity } of sorted) {
        const stock = await client.query(
          'UPDATE inventory SET available = available - $2 WHERE sku = $1 AND available >= $2',
          [sku, quantity],
        );
        if (stock.rowCount === 0) {
          throw { status: 409, code: 'OUT_OF_STOCK', message: `No hay stock suficiente de ${sku}` };
        }
        await client.query(
          'INSERT INTO order_items (order_id, sku, quantity, unit_price) VALUES ($1, $2, $3, $4)',
          [orderId, sku, quantity, prices.get(sku)],
        );
        await client.query(
          `INSERT INTO reservations (order_id, sku, quantity, expires_at)
           VALUES ($1, $2, $3, now() + interval '15 minutes')`,
          [orderId, sku, quantity],
        );
      }

      // La suma va en SQL porque numeric es exacto; en JS serían floats.
      const totals = await client.query(
        `UPDATE orders SET total = (SELECT sum(quantity * unit_price) FROM order_items WHERE order_id = $1)
         WHERE id = $1 RETURNING total`,
        [orderId],
      );

      const order = { id: orderId, status: 'pending_payment', total: totals.rows[0].total, items: sorted };
      await client.query(
        'UPDATE idempotency_keys SET response_status = 201, response_body = $3 WHERE customer_id = $1 AND key = $2',
        [customerId, idempotencyKey, order],
      );

      await client.query('COMMIT');
      return { status: 201, order };
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
}

module.exports = { OrderService };
```

## 24. Idempotencia

`idempotency_keys` tiene clave primaria `(customer_id, key)` y el primer paso de la transacción es `INSERT ... ON CONFLICT DO NOTHING`. Si no inserta, la key ya existe: si el sha256 del body (con los items ordenados por SKU) coincide, devuelvo la respuesta guardada; si no, 409 `IDEMPOTENCY_KEY_REUSED`. Si un reintento llega mientras la primera request sigue en curso, Postgres hace esperar su INSERT hasta que la primera termine, y si la primera hace rollback (por ejemplo, sin stock) la key tampoco queda y el reintento se evalúa de nuevo. Guardo la respuesta y no solo la key porque el reintento tiene que recibir la misma orden; un job diario borra las keys de más de 24 horas.

## 25. Códigos y errores

Todos los errores usan `{ error: { code, message } }`:

- 400 `VALIDATION_ERROR`: datos inválidos.
- 404 `SKU_NOT_FOUND`: un SKU no existe en el catálogo.
- 409 `OUT_OF_STOCK` o `IDEMPOTENCY_KEY_REUSED`: conflicto con el estado actual.
- 500 `INTERNAL_ERROR`: sin detalles internos, que van al log.

401 sin token válido y 403 si el `customerId` no es el del token. El éxito es 201 con la orden y la URL de pago.

```json
{ "error": { "code": "OUT_OF_STOCK", "message": "No hay stock suficiente de SKU-050" } }
```

## 26. Modelo de datos

```sql
CREATE TABLE inventory (
  sku       text PRIMARY KEY,
  available integer NOT NULL CHECK (available >= 0)
);

CREATE TABLE orders (
  id          bigserial PRIMARY KEY,
  customer_id text NOT NULL,
  status      text NOT NULL CHECK (status IN ('pending_payment', 'paid', 'cancelled')),
  total       numeric(12, 2) NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
  order_id   bigint NOT NULL REFERENCES orders (id),
  sku        text NOT NULL,
  quantity   integer NOT NULL CHECK (quantity > 0),
  unit_price numeric(12, 2) NOT NULL,
  PRIMARY KEY (order_id, sku)
);

CREATE TABLE reservations (
  order_id   bigint NOT NULL REFERENCES orders (id),
  sku        text NOT NULL,
  quantity   integer NOT NULL,
  expires_at timestamptz NOT NULL,
  status     text NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'consumed', 'released')),
  PRIMARY KEY (order_id, sku)
);

CREATE TABLE idempotency_keys (
  customer_id     text NOT NULL,
  key             uuid NOT NULL,
  request_hash    text NOT NULL,
  response_status integer,
  response_body   jsonb,
  created_at      timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (customer_id, key)
);

CREATE TABLE payments (
  order_id            bigint PRIMARY KEY REFERENCES orders (id),
  provider_session_id text NOT NULL,
  payment_url         text NOT NULL,
  status              text NOT NULL DEFAULT 'pending'
);

CREATE INDEX idx_reservations_active ON reservations (expires_at) WHERE status = 'active';
CREATE INDEX idx_idempotency_keys_created ON idempotency_keys (created_at);
```

`payments.order_id` es la clave primaria: una sola sesión de pago por orden, como en el punto 2. Los dos índices sirven a los jobs de reservas vencidas y de limpieza de keys.

## 27. Pago fuera de la transacción

Después del `COMMIT`, el router llama a este método de `OrderService`:

```js
async ensurePaymentSession(order) {
  const { rows } = await this.pool.query('SELECT payment_url FROM payments WHERE order_id = $1', [order.id]);
  if (rows.length > 0) return rows[0].payment_url;

  try {
    // Con el id de la orden como key, un reintento recibe la misma sesión del proveedor.
    const session = await this.paymentProvider.createSession(
      { orderId: order.id, amount: order.total },
      { idempotencyKey: `order-${order.id}`, timeout: 10000 },
    );
    await this.pool.query(
      `INSERT INTO payments (order_id, provider_session_id, payment_url)
       VALUES ($1, $2, $3) ON CONFLICT (order_id) DO NOTHING`,
      [order.id, session.id, session.url],
    );
    return session.url;
  } catch (err) {
    logger.warn({ err, orderId: order.id }, 'payment session failed');
    return null;
  }
}
```

Mientras se espera al proveedor no hay transacción abierta ni conexión tomada, así que un proveedor lento no bloquea stock ni otras requests. Si falla o tarda más de 10 segundos, respondo 201 con `paymentUrl: null`; el cliente reintenta el mismo POST con la misma key, recibe la orden guardada y se vuelve a pedir la sesión, como en el paso 3 del punto 2. La reserva de 15 minutos sostiene el stock mientras tanto, y el pago se confirma por webhook como describe el punto 2.
