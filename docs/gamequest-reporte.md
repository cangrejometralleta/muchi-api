# GameQuest y Muchi: mejoras para consultar ofertas disponibles

Reporte preparado el 18 de septiembre de 2026 (Sí en serio uyuyui).

Hola, equipo de GameQuest: 🐱

Ya integramos su catálogo de Magic en el código de Muchi, nuestro buscador de
ofertas de cartas. Pudimos completar una búsqueda real de Sol Ring y recuperar
dos ofertas disponibles, con verificación de stock de la primera. Nos gustaría
coordinar algunas mejoras para que esta integración responda más rápido y genere
menos solicitudes a su tienda.

## Qué observamos

Consultamos el buscador público del escaparate Jumpseller mediante
`/api/search/Sol%20Ring?page=N`. La primera página devolvió 40 productos de Sol Ring:
39 declaraban stock cero y uno declaraba una unidad. Tener muchas ediciones
agotadas es normal; la dificultad para Muchi es tener que recorrerlas para
encontrar las que sí se pueden comprar.

En una prueba completa, búsqueda y consultas de producto sumaron al menos
120 solicitudes y 11 minutos y 19 segundos. Muchi introduce una pausa de cuatro
segundos entre solicitudes para moderar el tráfico, por lo que este tiempo
incluye nuestras pausas y no representa solamente la latencia de su servidor.

También observamos solapamiento entre páginas y una página sin productos nuevos
durante el recorrido. Adaptamos nuestro lector para tolerar una repetición
aislada y la prueba pudo terminar. Esto no permite concluir que todo su buscador
esté roto; agradeceríamos confirmar el funcionamiento previsto de la paginación.

## Primera alternativa: una opción de Jumpseller

Jumpseller documenta dos ajustes que podrían ayudar sin desarrollar un endpoint.
La [guía oficial en español de Configuración General](https://jumpseller.cl/support/general-settings/)
explica ambos en las secciones «Configuración General» y «No ocultes productos
si volverán a estar disponibles». Estos ajustes afectan al catálogo de la tienda;
la documentación no confirma su efecto en `/api/search`.

### Opción A: ocultar agotados

1. Entrar al panel de administración de Jumpseller y abrir **General → Preferencias**.
2. En **Configuración General**, activar **Ocultar productos sin stock** y guardar
   el cambio si el panel lo solicita.
3. Avisarnos para comprobar juntos el resultado con el procedimiento de abajo.

La guía advierte que ocultar productos puede afectar su visibilidad en buscadores.
Si desean mantener visibles las ediciones que repondrán, pueden evaluar la opción B.

### Opción B: conservar agotados al final

1. Abrir **General → Preferencias** en el panel.
2. Activar **Mostrar productos agotados al final de la lista** y guardar si corresponde.
3. Avisarnos para verificar si ese orden también se aplica a la búsqueda JSON.

Los nombres anteriores son los publicados por Jumpseller; no hemos accedido a su
panel. Ambas opciones se explican en la [guía paso a paso de ajustes generales](https://jumpseller.cl/support/general-settings/).

### Cómo comprobaremos que ayuda a Muchi

Esta comprobación la propone Muchi; no forma parte del procedimiento documentado
por Jumpseller. Guardaremos el valor anterior para poder restablecerlo si el
resultado no es el deseado. Probaremos una opción a la vez.

Primero revisaremos la [búsqueda visual de Sol Ring](https://gamequest.cl/search?q=Sol%20Ring)
y después repetiremos la consulta JSON con los encabezados del escaparate:

```sh
curl --fail --silent --show-error \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Referer: https://gamequest.cl/' \
  'https://gamequest.cl/api/search/Sol%20Ring?page=1'
```

Con la opción A, comprobaremos que se excluyan los productos realmente agotados
antes de paginar y se conserven aquellos con variantes disponibles o stock
ilimitado. Compararemos también las ofertas y el número de solicitudes con la
medición anterior; el inventario puede haber cambiado desde entonces.

Con la opción B, necesitamos que Jumpseller confirme un orden global por
disponibilidad para todas las páginas de esta ruta. Ver primero un producto
disponible no basta: solo con ese contrato podremos adaptar Muchi para detenerse
al comenzar los agotados. El lector actual aún no hace ese corte.

Si solo cambia la presentación visual, Muchi seguirá recorriendo las páginas.
En ese caso agradeceríamos trasladar a soporte de Jumpseller esta pregunta:
«¿Estos ajustes se aplican a `/api/search/{query}?page=N`, incluyendo variantes
y stock ilimitado? Si no, ¿qué filtro o endpoint soportado permite consultar
únicamente productos comprables antes de paginar?».

## Medidas si la configuración no resuelve la consulta

1. **Consultar solo productos con stock desde el servidor.** ¿Existe una opción
   soportada para excluir agotados antes de paginar, considerando variantes y
   stock ilimitado? Si existe, agradeceríamos el endpoint y un ejemplo de uso.
   Probamos `in_stock=true`, `stock=1` y `status=available`, pero devolvieron la
   misma primera página con 39 productos de stock cero. Filtrar después en Muchi
   evita algunas consultas de detalle, pero no evita descargar las demás páginas.

2. **Reducir y estabilizar las páginas de búsqueda.** Nos ayudaría una búsqueda
   exacta por nombre de carta que conserve sus ediciones, idiomas y condiciones,
   junto con un tamaño de página configurable, un orden estable y una señal clara
   de fin. Probamos `limit` y `per_page` con valores de hasta 10000 y seguimos
   recibiendo 40 productos. Preferimos solicitar un lote grande dentro del límite
   que ustedes recomienden para reducir el número de llamadas.

3. **Confirmar una fuente estructurada para la integración.** El JSON público ya
   incluye precio, descuento, stock y variantes. Una descripción de esos campos,
   incluida moneda, disponibilidad y frecuencia de actualización, nos permitiría
   aprovecharlos con confianza y abrir menos fichas. Si el buscador no admite
   los filtros anteriores, un feed JSON/CSV de inventario o un endpoint acordado
   podría resolver la misma necesidad. Agradeceríamos también sus límites de
   solicitudes y recomendaciones de caché.

## Qué mejoraríamos de nuestro lado

Podemos adaptar Muchi al mecanismo soportado que nos indiquen, reutilizar los
datos de inventario y evitar consultas de detalle innecesarias. La mejora más
importante sería que una búsqueda nos entregue directamente las ofertas
disponibles, con sus precios y enlaces de compra.

Entendemos que algunas opciones dependen de Jumpseller o de la configuración del
tema. Si corresponde, agradeceríamos que pudieran consultar con su soporte qué
alternativa está disponible. No necesitamos cambiar su catálogo visible para
todos los clientes si existe una vía de consulta específica para integraciones.

Muchas gracias por ayudarnos a mostrar sus ofertas en Muchi de manera más ágil
y con menos carga para su tienda.

UYuuuui, Cangrejo Metralleta 🦀
