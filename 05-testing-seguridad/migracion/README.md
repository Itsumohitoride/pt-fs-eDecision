# 5. Migración sin cortar servicio

Uso expandir y contraer. Ejemplo: renombrar `orders.total` a `total_amount`. Un `RENAME` directo no sirve porque durante el despliegue la versión anterior del backend sigue atendiendo requests y falla al no encontrar `total`.

Cada paso es un despliegue aparte, y cada uno se puede revertir sin romper la versión que lo precede:

1. Agregar `total_amount` sin `NOT NULL` y quitarle el `NOT NULL` a `total`, porque en el paso 5 se deja de escribir. La versión actual no nota ninguno de los dos cambios.
2. Desplegar código que escribe en las dos columnas y sigue leyendo `total`.
3. Copiar los datos viejos por lotes, para no bloquear la tabla ni generar una transacción enorme.
4. Desplegar código que lee `total_amount`.
5. Desplegar código que deja de escribir `total`.
6. En un despliegue posterior, cuando ya no corre ninguna versión que use `total`, borrarla.

Paso 1:

```sql
ALTER TABLE orders ADD COLUMN total_amount numeric(12, 2);
ALTER TABLE orders ALTER COLUMN total DROP NOT NULL;
```

En PostgreSQL las dos solo cambian el catálogo; el lock dura milisegundos.

Paso 3, repetido por rangos de id (por ejemplo de 10.000 en 10.000) hasta que no queden filas:

```sql
UPDATE orders
SET total_amount = total
WHERE id BETWEEN $1 AND $2
  AND total_amount IS NULL;
```

El `IS NULL` hace que se pueda cortar y reanudar sin pisar lo que ya escribió la versión nueva. Al terminar, verifico que no queden filas sin `total_amount`.

Paso 6:

```sql
ALTER TABLE orders DROP COLUMN total;
```

El costo es hacer seis pasos en vez de uno y unos días con la columna duplicada. A cambio, en ningún momento hay una versión del backend corriendo contra un esquema que no entiende.
