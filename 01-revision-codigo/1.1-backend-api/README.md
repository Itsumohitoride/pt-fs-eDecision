# 1.1 Backend / API

Asumo Express 4 y PostgreSQL con `pg`; el código va en JavaScript, igual que el fragmento original. También asumo que ya hay un middleware de autenticación que valida el token y deja el usuario en `req.user.id`; sin token válido responde 401 antes de llegar al endpoint. `orders.id` es serial, así que un id más alto es una orden más reciente. Como el original usa `SELECT *`, asumo columnas típicas (por ejemplo `order_items` con `sku`, `quantity` y `unit_price`, como en el carrito del punto 3).

## 1. Problemas

1. Inyección SQL: `userId` se interpola en el texto de la consulta. Con `?userId=1 OR 1=1` devuelve las órdenes de todos los usuarios.
2. IDOR: el `userId` viene del query y nadie lo compara con el usuario autenticado. Cualquiera puede leer las órdenes de otro cambiando un número.
3. Sin validación de entrada: un `userId` ausente o no numérico llega al SQL (`user_id = undefined`) y falla en la BD en vez de responder 400.
4. N+1 con `await` en serie: una consulta por orden, una detrás de otra. Con 200 órdenes son 201 viajes a la BD.
5. Sin paginación ni `ORDER BY`: un usuario con miles de órdenes las recibe todas, en un orden que la BD no promete mantener. Además la respuesta es un arreglo sin envoltorio, así que no hay dónde poner el cursor.
6. `SELECT *`: trae columnas que no se usan (si alguna es pesada, viaja en cada request), expone columnas internas y el contrato de la API cambia cada vez que alguien agrega una columna.
7. Sin manejo de errores: en Express 4 una promesa rechazada no llega al error handler. Desde Node 15 eso tumba el proceso, y con él todas las requests en curso.
8. Las consultas están dentro del handler: mezcla HTTP con acceso a datos y es difícil de probar.

## 2. Clasificación

- Seguridad: 1, 2, 3, 6.
- Rendimiento: 4, 5, 6.
- Mantenibilidad: 6, 7, 8.
- Diseño de API: 3, 5, 7.

## 3. Versión mejorada

`orders.repository.js`

```js
class OrdersRepository {
  constructor(pool) {
    this.pool = pool;
  }

  async findByUser(userId, limit, before) {
    const orders = before
      ? await this.pool.query(
          `SELECT id, status, total, created_at FROM orders
           WHERE user_id = $1 AND id < $2
           ORDER BY id DESC LIMIT $3`,
          [userId, before, limit],
        )
      : await this.pool.query(
          `SELECT id, status, total, created_at FROM orders
           WHERE user_id = $1
           ORDER BY id DESC LIMIT $2`,
          [userId, limit],
        );

    if (orders.rows.length === 0) return [];

    const items = await this.pool.query(
      `SELECT order_id, sku, quantity, unit_price FROM order_items
       WHERE order_id = ANY($1)
       ORDER BY order_id, id`,
      [orders.rows.map((o) => o.id)],
    );

    return orders.rows.map((o) => ({
      id: o.id,
      status: o.status,
      total: o.total,
      createdAt: o.created_at,
      items: items.rows
        .filter((i) => i.order_id === o.id)
        .map((i) => ({ sku: i.sku, quantity: i.quantity, unitPrice: i.unit_price })),
    }));
  }
}

module.exports = { OrdersRepository };
```

`orders.router.js`

```js
const { Router } = require('express');

function badRequest(res, message) {
  return res.status(400).json({ error: { code: 'INVALID_PARAMETER', message } });
}

function ordersRouter(repo) {
  const router = Router();

  router.get('/', async (req, res, next) => {
    const limit = req.query.limit ? Number(req.query.limit) : 20;
    const before = req.query.before ? Number(req.query.before) : undefined;

    if (!Number.isInteger(limit) || limit < 1 || limit > 100) {
      return badRequest(res, 'limit debe ser un entero entre 1 y 100');
    }
    // Tope de integer en Postgres; más allá la BD falla y sería un 500.
    if (before !== undefined && (!Number.isInteger(before) || before < 1 || before > 2147483647)) {
      return badRequest(res, 'before debe ser un id válido');
    }

    try {
      // El usuario sale del token, nunca del query.
      const orders = await repo.findByUser(req.user.id, limit, before);
      const nextCursor = orders.length === limit ? orders[orders.length - 1].id : null;
      res.json({ data: orders, nextCursor });
    } catch (err) {
      next(err);
    }
  });

  return router;
}

module.exports = { ordersRouter };
```

Montaje y error handler global:

```js
app.use('/api/v1/orders', ordersRouter(new OrdersRepository(pool)));

app.use((err, req, res, next) => {
  logger.error({ err, path: req.path });
  res.status(500).json({ error: { code: 'INTERNAL_ERROR', message: 'Error interno' } });
});
```

Uso dos consultas en vez de un JOIN porque se leen fácil, cada una usa su índice y el costo es un viaje extra fijo, no uno por orden. Pagino con cursor (el último id) y no con offset porque offset obliga a la BD a leer y descartar las filas anteriores, y si entran órdenes nuevas entre una página y otra se repiten filas. El repositorio entra por constructor para poder probar el router con un doble; no puse capa de servicio porque no hay lógica de negocio que meter ahí.

Cambios en el contrato: la ruta pasa a `/api/v1/orders`, ya no recibe `userId` en el query, acepta `limit` (default 20, máximo 100) y `before`, responde `{ data, nextCursor }` en vez de un arreglo, y los errores 400, 401 y 500 usan el mismo formato `{ error: { code, message } }`.

## 4. Índices y ajustes de BD

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id, id DESC);
CREATE INDEX CONCURRENTLY idx_order_items_order ON order_items (order_id);
```

El primero sirve para el filtro por usuario, el orden y la condición del cursor, así que la consulta lee solo las filas de la página. El segundo hace falta porque Postgres no indexa las foreign keys automáticamente; sin él, cada página recorre `order_items` completa. En producción los crearía con `CONCURRENTLY` para no bloquear escrituras; no corre dentro de una transacción, así que va en una migración aparte.

También pondría un `statement_timeout` (por ejemplo 5 s) en el rol de la app, para que una consulta lenta no se quede con conexiones del pool.
