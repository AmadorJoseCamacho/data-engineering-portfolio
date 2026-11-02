# 🏠 EDA de avisos inmobiliarios: capa Bronze en Databricks

![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)
![Spark SQL](https://img.shields.io/badge/Spark%20SQL-E25A1C?logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-00ADD4?logo=delta&logoColor=white)
![Unity Catalog](https://img.shields.io/badge/Unity%20Catalog-1B3139)

> Ingesta de **1,2 millones de avisos inmobiliarios** de Buenos Aires en una arquitectura medallion y **análisis exploratorio de calidad de datos** sobre la capa bronze, para definir las reglas de limpieza de la capa silver.

📓 **[Ver el notebook con resultados →](notebooks/eda_properties_bronze.ipynb)**

---

## 📌 Contexto

El dataset contiene avisos de venta y alquiler scrapeados de portales inmobiliarios argentinos (MercadoLibre, Argenprop, Liderprop, entre otros). Los datos llegan **crudos y heterogéneos**: todas las columnas son texto, cada portal usa sus propios formatos y hay errores de carga, duplicados y valores imposibles.

**Objetivo:** cargar los datos en la capa bronze y diagnosticar su calidad antes de construir la capa silver.

## 🏗️ Arquitectura

```
properties_raw.csv
        │  carga manual
        ▼
/Volumes/bootcamp/landing/archivos        ← LANDING  (Volume de Unity Catalog)
        │  read_files() + filtro de URL válida
        ▼
bootcamp.bronze.properties_bronze         ← BRONZE   (tabla Delta)
        │  EDA
        ▼
Hallazgos  →  reglas de limpieza para SILVER
```

## 🔍 Qué se hizo

| Etapa | Detalle | Técnicas |
|---|---|---|
| Ingesta | Catálogo, esquemas `landing`/`bronze`, Volume *managed* y tabla Delta desde CSV | `read_files`, Unity Catalog, Volumes |
| Nulos | Conteo y % de nulos por columna, transpuesto a una fila por columna | CTEs, `unpivot` |
| Categóricas | Cardinalidad y distribución de operación, moneda, ambientes, zona y estado | `sum(count(*)) over()` |
| Numéricas | Precio por moneda × operación, m², percentiles y filtrado de outliers | `percentile`, vistas temporales, `rlike` |
| Calidad | Valores centinela, duplicados por URL y outliers P1/P99 | `count_if`, `having`, joins |
| Análisis | Ranking de zonas por precio y comparación con el promedio global | `row_number() over()` |

## 💡 Hallazgos principales

- **8 columnas con más del 50 % de nulos.** `orientacion_cardinal` (92,5 %), `tipo_vendedor` (84,6 %) y `orientacion_inmueble` (83,6 %) son candidatas a eliminarse.
- **Nulos disfrazados:** `antiguedad = 999` en el **99,4 %** de los registros, y m² con valor `2.147.483.647` (el máximo de un entero). Ninguno aparece en un conteo de nulos tradicional.
- **Categóricas sin estandarizar:** 28 variantes de tipo de operación (con typos como `alquier` o `alguilar`), 46 de estado (incluidos `-1`, `luminoso`…) y la misma zona escrita en 3 formatos distintos según el portal.
- **Moneda mal cargada:** las ventas en ARS tienen una mediana de $700.000, un valor irreal para una venta, lo que indica avisos en USD etiquetados como ARS. Además, cerca del **40 % de los alquileres se publican en USD**, así que cualquier análisis de precio debe separar por moneda.
- **130.750 registros duplicados** (misma URL y precio), con avisos repetidos hasta 12 veces.
- **Precios de referencia** (sin outliers, P1–P99):

| Segmento | Mediana |
|---|---|
| Venta (USD) | US$ 135.000 |
| Alquiler (ARS) | $ 650.000 |
| Alquiler (USD) | US$ 1.600 |

## 🧠 Decisiones de ingeniería

- **Mediana en lugar de promedio:** con outliers de hasta 1.000 millones, el promedio no representa el precio típico.
- **Separar por moneda y operación** antes de cualquier agregación de precio.
- **Bronze se mantiene crudo:** toda la limpieza se prueba en vistas temporales y se materializará recién en silver.
- **Detectar centinelas** (`999`, `2147483647`, `NaN`) además de los nulos reales.

## ➡️ Próximos pasos: capa Silver

1. Tipar columnas numéricas y convertir centinelas a `null`.
2. Normalizar `tipo_de_operacion`, `moneda`, `estado` y `zona`.
3. Deduplicar por `url`, conservando el registro más reciente.
4. Eliminar columnas sin información útil.
5. Marcar outliers con un flag en lugar de borrarlos.

## 📁 Estructura

```
semana-01-eda-propiedades/
├── README.md
└── notebooks/
    └── eda_properties_bronze.ipynb   ← notebook de Databricks con resultados
```

> El CSV original no se incluye en el repositorio por su tamaño y por tratarse de datos de terceros.
