# 1.3 SQL / datos

## 9. Problemas

1. `DATE(o.created_at)` aplica una función a la columna: la BD no puede usar un índice sobre `created_at` y recorre toda `orders` para encontrar las de un día.
2. `GROUP BY c.name`: dos clientes distintos con el mismo nombre se suman como si fueran uno.
3. `COUNT(o.id)` cuenta items, no órdenes: por el JOIN con `order_items`, una orden con 3 items cuenta 3.

## 10. Consulta reescrita

```sql
SELECT c.id, c.name,
       COUNT(DISTINCT o.id) AS orders,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.created_at >= '2026-10-01'
  AND o.created_at < '2026-10-02'
GROUP BY c.id, c.name
ORDER BY revenue DESC;
```

El rango semiabierto (desde el inicio del día, hasta antes del inicio del siguiente) cubre el día completo sin tocar la columna. Agrupo por `c.id` para que cada cliente sea una fila, y `COUNT(DISTINCT o.id)` cuenta cada orden una vez aunque tenga varios items.

## 11. Índices

```sql
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders (created_at);
CREATE INDEX CONCURRENTLY idx_order_items_order ON order_items (order_id);
```

El primero permite leer solo las órdenes del día en vez de toda la tabla. El segundo es el mismo del 1.1: Postgres no indexa las foreign keys, y sin él el JOIN con `order_items` recorre la tabla completa. `customers` ya se busca por su primary key, que tiene índice.

## 12. Cientos de millones de registros

Supongo que la tabla ya pasó a miles de millones de órdenes y sigue creciendo.

Particionaría `orders` por rango de `created_at`, una partición por mes:

```sql
CREATE TABLE orders (
  id          bigserial,
  customer_id bigint NOT NULL,
  created_at  timestamptz NOT NULL,
  -- resto de columnas
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_10 PARTITION OF orders
  FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
```

La primary key incluye `created_at` porque Postgres lo exige en tablas particionadas. El costo es que las foreign keys hacia `orders` (como la de `order_items`) también tienen que llevar `created_at`.

La consulta del día solo lee la partición de octubre; Postgres descarta las demás (partition pruning). Un índice creado en la tabla padre se crea en cada partición, así que el de `created_at` sigue sirviendo dentro del mes.

Siempre por mes. Mezclar tamaños (meses recientes, años viejos) obliga a reorganizar datos cada año; con mes, 10 años son 120 particiones y Postgres las maneja bien.

Un job programado crea la partición del mes siguiente por adelantado. Si llega una orden y no existe su partición, la inserción falla.

Los meses que ya nadie consulta se sacan con `ALTER TABLE orders DETACH PARTITION ...` y se archivan. Es casi instantáneo, a diferencia de un `DELETE` de millones de filas. Toma un lock breve sobre `orders`; con `DETACH PARTITION ... CONCURRENTLY` no bloquea las consultas.

Los reportes como este irían a una réplica de lectura, para no cargar la base que recibe las escrituras del checkout. El costo es que la réplica puede ir unos segundos atrás, lo cual no importa para un reporte del día anterior.

Pasar la tabla actual a particionada no se hace en sitio: se crea la tabla nueva particionada y se mueven los datos por lotes.
