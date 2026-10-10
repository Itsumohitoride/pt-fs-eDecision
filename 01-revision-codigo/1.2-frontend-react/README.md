# 1.2 Frontend / React

Asumo que `api.get(url, { signal })` se comporta como axios: devuelve una promesa con `{ data }` y acepta un `signal` de `AbortController` para cancelar. También asumo que cada producto tiene un `id` único. Con la búsqueda vacía no consulto y la lista queda vacía; el original pedía todos los productos, pero no tiene sentido cargar el catálogo entero antes de que el usuario escriba algo.

## 5. Problemas

1. `key={index}`: si la lista cambia, React asocia cada elemento por su posición y puede mostrar datos o estado de un producto equivocado. La key debe ser `p.id`.
2. Sin loading ni error: mientras espera no hay indicador de carga, y el `.then` no tiene `.catch`. Si la API falla, el usuario no ve nada y queda una promesa rechazada sin manejar.
3. Una request por cada tecla: escribir "camiseta" son 8 llamadas.
4. Se muta el estado: `products.push` modifica el mismo arreglo y `setProducts(products)` pasa la misma referencia, así que React no re-renderiza. Además los resultados de cada búsqueda se suman a los anteriores.
5. `products` en las dependencias del efecto: hoy no explota porque la referencia nunca cambia, pero al corregir el 4 cada respuesta dispararía el efecto otra vez y quedaría un loop infinito de requests.
6. Respuestas fuera de orden y sin cancelación: si "cam" responde después que "camiseta", la lista muestra lo de "cam". Tampoco se cancela la request al cambiar la búsqueda ni al desmontar el componente.

También hay detalles menores que corrijo en el código sin listarlos aparte: el `console.log` en el `onClick`, el input sin `label` y la búsqueda sin `encodeURIComponent`.

## 6. Solicitudes mientras se escribe

Pongo un `setTimeout` de 300 ms dentro del `useEffect`. Si el usuario sigue escribiendo, el cleanup del efecto limpia el timer anterior, así que solo sale una request cuando deja de escribir. "camiseta" pasa de 8 llamadas a 1.

## 7. Fuera de orden, loading, error y cancelación

Cada búsqueda crea su propio `AbortController` y el cleanup del efecto llama a `controller.abort()`. Eso cubre tanto el cambio de búsqueda como el desmontaje. Como la request vieja se cancela antes de lanzar la nueva, una respuesta atrasada nunca pisa a la actual.

Loading y error son estados aparte. Loading se marca al cambiar la búsqueda y solo la búsqueda vigente lo apaga. Si la request fue abortada, el error se ignora: lo compruebo con `controller.signal.aborted`, que no depende de cómo cada librería nombra el error de cancelación.

## 8. Código corregido

```jsx
import { useEffect, useState } from 'react';

function useProductSearch(api, search) {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!search.trim()) {
      setProducts([]);
      setError(null);
      setLoading(false);
      return;
    }

    const controller = new AbortController();
    setLoading(true);
    setError(null);

    const timer = setTimeout(() => {
      api
        .get('/products?search=' + encodeURIComponent(search), { signal: controller.signal })
        .then((response) => setProducts(response.data))
        .catch((err) => {
          // Una búsqueda cancelada no es un error para el usuario.
          if (!controller.signal.aborted) setError(err);
        })
        .finally(() => {
          if (!controller.signal.aborted) setLoading(false);
        });
    }, 300);

    return () => {
      clearTimeout(timer);
      controller.abort();
    };
  }, [api, search]);

  return { products, loading, error };
}

function ProductList({ api, onSelect }) {
  const [search, setSearch] = useState('');
  const { products, loading, error } = useProductSearch(api, search);

  return (
    <div>
      <label>
        Buscar productos{' '}
        <input type="search" value={search} onChange={(e) => setSearch(e.target.value)} />
      </label>

      {loading && <p>Cargando...</p>}
      {error && <p role="alert">No pudimos cargar los productos. Intenta de nuevo.</p>}
      {!loading && !error && search.trim() && products.length === 0 && <p>Sin resultados.</p>}

      {products.map((p) => (
        <Product key={p.id} data={p} onClick={() => onSelect(p)} />
      ))}
    </div>
  );
}
```

`onSelect` lo pasa el componente padre y decide qué hacer con el producto elegido, en vez del `console.log`. Marco loading apenas cambia la búsqueda y no cuando sale la request, para que durante los 300 ms de espera no aparezca "Sin resultados" por un instante.

Saqué la búsqueda a un hook porque al corregir los problemas 2 a 6 la lógica crece, y el componente queda solo con el render. `api` ya entra como prop, así que el hook y el componente se pueden probar con un cliente falso.
