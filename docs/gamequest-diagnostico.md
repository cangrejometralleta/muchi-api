# GameQuest: coste de búsqueda e integración

Fecha de observación: 18 de septiembre de 2026.

GameQuest está integrada y habilitada para Magic en la configuración local del
repositorio. Una búsqueda directa de Sol Ring terminó correctamente, pero tomó
678,66 segundos y al menos 120 solicitudes para devolver dos ofertas disponibles.
Esto valida ese recorrido del adaptador; no acredita despliegue, exhaustividad
del catálogo ni rendimiento de las tareas de producción.

## Recorrido y coste

`internal/jumpseller/search.go` consulta
`https://gamequest.cl/api/search/Sol%20Ring?page=N`, con los encabezados públicos
del escaparate `X-Requested-With: XMLHttpRequest` y `Referer: https://gamequest.cl/`.
Acumula IDs, selecciona títulos mediante `offer.MatchesCard` y deduplica enlaces.
`internal/jumpseller/client.go` abre después cada producto coincidente, interpreta
sus variantes y devuelve únicamente ofertas disponibles.

Cada solicitud se serializa dentro del proceso y espera cuatro segundos antes
de salir. Esa pausa es una decisión de Muchi para moderar el tráfico: la duración
medida no debe atribuirse íntegramente al tiempo de respuesta de GameQuest.
La caché amortiza búsquedas repetidas; una búsqueda que requiere consultar la
fuente vuelve a pagar el recorrido.

El problema tiene dos costes distintos:

- Paginar resultados que incluyen agotados. Descartarlos localmente no evita
  solicitar las páginas posteriores.
- Abrir fichas de productos que el JSON ya podría permitir descartar. El lector
  de búsqueda actual conserva ID, nombre y enlace; no aprovecha stock y variantes
  de esa respuesta para reducir las consultas de detalle.

## Evidencia observada

| Comprobación | Resultado | Alcance |
| --- | --- | --- |
| Primera página JSON de Sol Ring | 40 productos de Sol Ring; 39 declaran stock cero y uno declara una unidad | Dos ofertas finales no significa dos coincidencias en toda la búsqueda |
| Páginas 40 y 41 | Comparten 15 IDs y contienen IDs distintos | Hay solapamiento; no son páginas idénticas |
| Página 1000 | `products: []` | Existe una respuesta vacía; no determina por sí sola la última página válida |
| `limit` y `per_page`, con 100 y 10000 | Primera página de 40 productos, sin el aumento solicitado | No se verificó un tamaño de página configurable |
| `in_stock=true`, `stock=1`, `status=available` | Mismos 40 productos, 39 con stock cero | Estos parámetros no consiguieron filtrar stock; no prueba que no exista otro |
| Prueba inicial | Error de productos repetidos tras más de 40 páginas; 248,04 segundos | El lector abortaba ante una página sin IDs nuevos |
| Prueba con tolerancia | Dos ofertas disponibles, primer precio 5104 CLP, stock de la primera reconfirmado; 678,66 segundos | Al menos 120 solicitudes entre búsqueda y productos; no 120 páginas de búsqueda |

Las lecturas ocurrieron en momentos distintos y no constituyen una instantánea.
No se instrumentó el total de IDs únicos, coincidencias de título ni agotados
del recorrido completo. No podemos afirmar que se recorrió todo el inventario,
ni explicar todavía la selección y el orden de todos los resultados.

El formulario visual `/search` publica `q`/`query`, `page`, `sorting` y filtros
de atributos `filter[cfv][ID][]`; no se encontró un filtro de disponibilidad.
El autocompletado usa `/api/suggest` con `q`, `locale` y `cat`, pero la consulta
directa probada respondió 403. Los parámetros de esas rutas no deben asumirse
compatibles con `/api/search/{nombre}`.

## Ajuste actual y límites

El lector tolera una página sin IDs nuevos y continúa. Si la siguiente tampoco
aporta IDs, devuelve error. Una página vacía termina normalmente. El estado de
repetición se reinicia al encontrar un ID nuevo. El cambio aplica al adaptador
Jumpseller compartido, no únicamente a GameQuest.

La prueba local `TestSearchSurvivesRepeatedPage` cubre una repetición seguida de
un producto nuevo; `TestInvalidData` conserva el rechazo de repetición persistente.
Se incrementó el namespace de caché a `search-providers-v9` porque cambió el
comportamiento de búsqueda. Este ajuste evitó el fallo observado, pero no garantiza
que no existan omisiones por orden inestable y no reduce la paginación.

No es seguro detenerse ante un producto agotado o una página sin coincidencias:
no hay un contrato verificado de orden global por disponibilidad o coincidencia.
`status=available` tampoco equivale a stock positivo; deben contemplarse las
variantes y el stock ilimitado.

## Siguientes medidas

1. Solicitar un filtro de disponibilidad aplicado en origen, antes de paginar,
   y una búsqueda exacta por nombre que conserve ediciones y variantes. Es la
   medida que puede eliminar páginas innecesarias para Muchi y tráfico para la tienda.
2. Solicitar el contrato de paginación y tamaño máximo: orden estable, cursor o
   indicador de fin. Un tamaño alto solo ayuda si el servidor lo respeta.
3. Aprovechar el JSON en Muchi para evitar detalles inequívocamente agotados,
   validando primero la semántica de stock, variantes, moneda y descuentos.
   Medir páginas, IDs únicos, títulos coincidentes, agotados y consultas de detalle.

Las tres son propuestas; el filtrado de stock desde el JSON no está implementado.
Un feed de inventario actualizado sería una alternativa si el escaparate no ofrece
estos controles. La decisión de uso en producción debe considerar la latencia
medida; el repositorio sigue habilitando GameQuest y no se cambió su estado al
escribir este diagnóstico.

Para repetir la prueba directa, con acceso de red:

```sh
MUCHI_TEST_JUMPSELLER_LIVE=1 go test ./internal/jumpseller \
  -run 'TestLiveSolRing/gamequest.cl' -v -count=1 -timeout 21m
```

La prueba genera tráfico real y puede tardar varios minutos. No hace falta
repetirla para verificar cambios exclusivamente documentales.

Referencias locales: [mediciones anteriores](stores-sol-ring.md),
[flujo de fuentes](busqueda-proveedores.md) y
[reporte propuesto para GameQuest](gamequest-reporte.md).
