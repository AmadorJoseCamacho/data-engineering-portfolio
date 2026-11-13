# De Bronze a Silver: avisos inmobiliarios

Notebook: [silver_propiedades.ipynb](notebooks/silver_propiedades.ipynb)

## De qué se trata

Con el diagnóstico de la [semana anterior](../semana-01-eda-propiedades), esta semana tocaba construir la capa Silver: una tabla donde cada fila sea un aviso único, con tipos correctos y sin valores basura.

El curso traía una solución propuesta, pero no me convenció el paso de limpieza. Filtraba las zonas por texto (que contuvieran `gba`, `caba` o `capital`). El problema es que Zonaprop y Argenprop, los dos portales con más avisos, a veces escriben solo el partido (`tigre`, `vicente-lopez`), sin la región. Ese filtro descartaba avisos válidos solo por cómo venía escrito el campo, y además los borraba en Silver, donde ya no se pueden recuperar.

Así que armé mi propio pipeline. Su regla principal: en Silver no se borra un aviso por problemas de calidad, se marca.

```
bronze.properties_bronze
  + silver.ref_partidos        (partido → región)
  → vista propiedades_tipadas  (tipos, normalización, centinelas a null)
  → silver.propiedades         (un aviso por fila, con flags)
  → validaciones
```

## Primero, mirar los datos

Antes de escribir el pipeline perfilé la vista filtrada del curso (P1–P99 de precio):

- **Zona:** hay 99 formas distintas de escribir la zona, pero corresponden a solo 38 partidos.
- **Fecha:** cubre 183 días (del 8 de julio de 2025 al 9 de enero de 2026) y es la fecha en que se scrapeó el aviso, no la de publicación. La hora siempre es la misma (`13:15:22`), así que no sirve.
- **Repetidos:** el mismo aviso aparece en hasta 10 fechas distintas.
- **Moneda:** las ventas en pesos vuelven a dar una mediana de $700.000. El porcentaje de alquileres en dólares depende mucho del portal: 21,8 % en Argenprop contra 46,9 % en Zonaprop. Si comparo meses sin separar por portal, mezclo cambios del scraping con cambios del mercado.
- **Nulos disfrazados:** `antiguedad` es nula o `999` en 1.197.858 de 1.197.911 registros, y 6.525 avisos tienen más m² cubiertos que totales.

## Cómo lo resolví

- **Zonas:** en vez de un `case` gigante, creé una tabla chica, `ref_partidos`, que dice a qué región pertenece cada partido. Si mañana aparece un partido nuevo, se agrega una fila y no se toca el pipeline. Una consulta con `left anti join` me dice si quedó alguna zona sin mapear.
- **Tipos:** uso `try_cast`, para que un valor como `consultar` en el precio quede en `null` en vez de romper la carga. Los `999` y los `-1` también pasan a `null`.
- **Duplicados:** con `row_number()` por URL me quedo con la versión más reciente de cada aviso, pero guardo cuántas veces se scrapeó y la primera fecha en que apareció.
- **Calidad:** marco los outliers de precio (P1/P99 por moneda y operación, calculados después de deduplicar), las ventas en pesos sospechosas de ser dólares y los m² inconsistentes. Quien use la tabla decide qué filtrar; para eso dejé una columna `apto_analisis_precio`.
- **Carga:** la tabla se crea con `create or replace table … as select`. Se puede volver a ejecutar las veces que sea sin duplicar filas y conserva el historial de Delta.

## Resultado

El balance cuadra: Bronze tiene 1.246.131 filas y 1.090.779 URLs únicas, y Silver queda con exactamente 1.090.779 avisos, sin URLs repetidas, sin zonas sin mapear y sin centinelas.

Por región, GBA Norte concentra el 45,9 % de los avisos, Capital Federal el 38,94 %, GBA Sur el 8,86 % y GBA Oeste el 6,3 %. De las 9.487 ventas en pesos, 8.454 quedaron marcadas como sospechosas.

Sobre los avisos aptos para analizar precio:

| Segmento | Avisos | Mediana | Mediana por m² |
|---|---|---|---|
| Venta (USD) | 495.958 | US$ 139.900 | US$ 1.509,62 |
| Alquiler (ARS) | 316.560 | $ 670.000 | $ 13.333,33 |
| Alquiler (USD) | 226.489 | US$ 1.600 | US$ 12,63 |

## Siguiente paso

Modelar Gold en estrella sobre esta tabla y definir ahí las reglas de negocio: qué avisos entran al análisis de precios y qué hacer con las ventas en pesos sospechosas.

Los ejercicios 1 a 3 del notebook son del curso; el perfilado y el pipeline Silver son míos, desarrollados con ayuda de IA.
