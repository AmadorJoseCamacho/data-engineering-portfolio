# 🥈 Arquitectura medallion: de Bronze a Silver en Databricks

![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)
![Spark SQL](https://img.shields.io/badge/Spark%20SQL-E25A1C?logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-00ADD4?logo=delta&logoColor=white)
![Unity Catalog](https://img.shields.io/badge/Unity%20Catalog-1B3139)

> Construcción de la **capa Silver** a partir de **1,2 millones de avisos inmobiliarios** crudos: perfilado estadístico, tabla de referencia geográfica, tipado y normalización, deduplicación por aviso y flags de calidad, con validaciones automáticas al final del pipeline.

📓 **[Ver el notebook con resultados →](notebooks/silver_propiedades.ipynb)**

---

## 📌 Contexto

En la [semana 1](../semana-01-eda-propiedades) se cargaron los avisos en Bronze y se diagnosticó su calidad. Esta semana se diseña la capa Silver: **datos tipados, normalizados, sin duplicados y sin valores centinela**, todavía neutrales respecto de las reglas de negocio (que quedan para Gold).

**Objetivo:** que cada fila de Silver represente **un aviso único** con tipos correctos y que los problemas de calidad queden **marcados con flags**, no borrados.

## 🏗️ Arquitectura

```
bootcamp.bronze.properties_bronze          ← BRONZE  (texto crudo, 1.246.131 filas)
        │
        │  + bootcamp.silver.ref_partidos   (tabla de referencia: partido → región)
        ▼
propiedades_tipadas  (vista temporal)       ← tipado, normalización, centinelas → null
        │  row_number() por url
        ▼
bootcamp.silver.propiedades                 ← SILVER  (1 fila por aviso, 1.090.779 filas)
        │
        ▼
Validaciones: balance, centinelas, regiones, flags, nulos, medianas
```

## 🔍 Qué se hizo

| Etapa | Detalle | Técnicas |
|---|---|---|
| Perfilado | Análisis de moneda, zona, fecha y nulos sobre la vista filtrada por percentiles | `percentile`, `parse_url`, `stack`, `count_if` |
| Referencia geográfica | Tabla `ref_partidos` que traduce las variantes de zona de cada portal a partido y región | `regexp_extract`, `left anti join` |
| Tipado | Conversión segura de texto a números, fechas y booleanos; rangos válidos para piso y antigüedad | `try_cast`, `case`, `coalesce` |
| Normalización | Operación, moneda, estado y orientación llevados a categorías cerradas | `case … in (…)`, `regexp` |
| Deduplicación | Una fila por `url`, conservando la versión más reciente y el historial de scraping | `row_number()`, `min() over`, `count() over` |
| Calidad | Flags de outlier de precio (P1/P99 por moneda × operación), moneda sospechosa y m² inconsistentes | CTEs, `left join` |
| Persistencia | Tabla Delta creada con CTAS idempotente y clave sustituta estable | `create or replace table … as`, `xxhash64` |
| Validación | Chequeos que deben dar 0 o cuadrar contra Bronze | consultas de control |

## 💡 Hallazgos del perfilado

- **Zona:** cada portal escribe la zona con su propio formato. Las **99 variantes** se reducen a **38 partidos** reales.
- **Fecha:** `fecha` cubre **183 días** (2025-07-08 a 2026-01-09) y es la **fecha de scraping**, no de publicación. `hora` tiene un único valor (`13:15:22`), así que no aporta información.
- **Avisos repetidos:** el mismo aviso aparece en hasta **10 fechas distintas**, por lo que Silver debe deduplicar por `url`.
- **Moneda mal etiquetada:** las ventas en ARS tienen una mediana de **$700.000**, un precio irreal para una venta, lo que indica avisos en USD cargados como ARS.
- **El portal sesga la moneda:** el porcentaje de alquileres publicados en USD es del 21,8 % en Argenprop y del 46,9 % en Zonaprop. Una comparación en el tiempo sin separar por portal mezcla cambios del scraping con cambios del mercado.
- **Nulos disfrazados:** `antiguedad` es nula o `999` en **1.197.858** de 1.197.911 registros, y hay **6.525** avisos con más m² cubiertos que totales.

## ✅ Resultado de Silver

**Balance Bronze → Silver**

| Filas Bronze | Avisos únicos en Bronze | Filas Silver | Versiones anteriores descartadas | URLs repetidas en Silver |
|---|---|---|---|---|
| 1.246.131 | 1.090.779 | 1.090.779 | 155.352 | 0 |

- **0 zonas sin mapeo** a partido y **0 valores centinela** (`999`, m² `2147483647`, precios o ambientes ≤ 0, estado `-1`, fechas futuras).
- **Distribución por región:** GBA Norte 45,9 %, Capital Federal 38,94 %, GBA Sur 8,86 % y GBA Oeste 6,3 %.
- **Moneda sospechosa:** **8.454 de 9.487** ventas en ARS quedan marcadas como probables precios en USD.
- **Precios de referencia** sobre avisos aptos para análisis:

| Segmento | Avisos | Mediana | Mediana por m² |
|---|---|---|---|
| Venta (USD) | 495.958 | US$ 139.900 | US$ 1.509,62 |
| Alquiler (ARS) | 316.560 | $ 670.000 | $ 13.333,33 |
| Alquiler (USD) | 226.489 | US$ 1.600 | US$ 12,63 |

## 🧠 Decisiones de ingeniería

- **No se borran filas por calidad:** Silver conserva todos los avisos únicos de Bronze. Los problemas quedan en flags (`flag_precio_outlier`, `flag_moneda_sospechosa`, `flag_m2_inconsistente`) y en `apto_analisis_precio`, para que cada consumidor decida qué filtrar.
- **Centinelas a `null`, no al revés:** un `999` o un `-1` contamina promedios y conteos; un `null` es honesto.
- **Tabla de referencia en lugar de un `case` largo:** el mapeo partido → región es un dato que se mantiene, no lógica. Una tabla se puede auditar y extender sin tocar el pipeline, y un `left anti join` detecta zonas nuevas sin mapear.
- **Deduplicar conservando historia:** se guarda la versión más reciente de cada aviso junto con `primera_fecha_scraping` y `veces_scrapeado`.
- **Percentiles calculados después de deduplicar:** un aviso scrapeado 10 veces no pesa 10 veces en los límites de outliers.
- **CTAS idempotente:** `create or replace table … as select` reconstruye la tabla completa en cada ejecución, sin riesgo de duplicar filas como un `insert into` repetido, y conserva el historial Delta.
- **Clave sustituta estable:** `xxhash64(url)` da el mismo ID a un aviso en cada reconstrucción, a diferencia de una columna `identity`.
- **Validar antes de dar por terminado:** el notebook cierra con chequeos que deben dar 0 o cuadrar con Bronze.

## ➡️ Próximos pasos: capa Gold

1. Modelado dimensional (estrella) sobre `silver.propiedades`.
2. Reglas de negocio: qué avisos entran en los análisis de precio y cómo tratar la moneda sospechosa.
3. Métricas por partido, región y período.

## 📁 Estructura

```
semana-02-medallion-silver/
├── README.md
└── notebooks/
    └── silver_propiedades.ipynb   ← notebook de Databricks con resultados
```

> El notebook parte del material del bootcamp (ejercicios 1 a 3) y agrega el perfilado estadístico y un pipeline Silver propio.
