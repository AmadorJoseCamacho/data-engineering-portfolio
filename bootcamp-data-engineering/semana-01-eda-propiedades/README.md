# EDA de avisos inmobiliarios en Bronze

Notebook: [eda_properties_bronze.ipynb](notebooks/eda_properties_bronze.ipynb)

## De qué se trata

El dataset son 1,2 millones de avisos de venta y alquiler de Buenos Aires, scrapeados de varios portales (MercadoLibre, Argenprop, Zonaprop, Liderprop). Lo cargué en Databricks como capa Bronze: un CSV en un Volume de Unity Catalog y, desde ahí, una tabla Delta con todas las columnas como texto.

Antes de limpiar nada quería saber qué tan malos venían los datos, para decidir las reglas de la capa Silver con evidencia y no a ojo.

```
properties_raw.csv → Volume landing → bronze.properties_bronze → EDA
```

## Lo que encontré

Lo que más me llamó la atención fueron los nulos que no parecen nulos. `antiguedad` vale `999` en el 99,4 % de los registros, y hay metros cuadrados con `2.147.483.647`, que es el entero más grande posible. Un conteo de `is null` no detecta ninguno de los dos.

La moneda tampoco es confiable. Las ventas en pesos tienen una mediana de $700.000, un precio imposible para comprar una propiedad, así que muchas son avisos en dólares mal etiquetados. Además, cerca del 40 % de los alquileres se publican en dólares, por lo que no se puede hablar de "precio promedio" sin separar por moneda.

Otros problemas:

- 8 columnas tienen más del 50 % de nulos (`orientacion_cardinal` 92,5 %, `tipo_vendedor` 84,6 %, `orientacion_inmueble` 83,6 %).
- El tipo de operación tiene 28 variantes, con errores como `alquier` o `alguilar`, y el estado tiene 46.
- Cada portal escribe la zona en un formato distinto (vi 3).
- Hay 130.750 registros duplicados por URL y precio; un mismo aviso aparece hasta 12 veces.

Sacando outliers (P1–P99 por moneda y operación), los precios típicos quedan así:

| Segmento | Mediana |
|---|---|
| Venta (USD) | US$ 135.000 |
| Alquiler (ARS) | $ 650.000 |
| Alquiler (USD) | US$ 1.600 |

## Qué me llevo para Silver

- Usar mediana y no promedio: hay precios de hasta 1.000 millones que arrastran cualquier promedio.
- Siempre separar por moneda y tipo de operación antes de agregar precios.
- No tocar Bronze. Las pruebas de limpieza las hice en vistas temporales.
- Convertir los centinelas (`999`, `2147483647`) a `null`, normalizar operación, moneda, estado y zona, y deduplicar por URL quedándome con la versión más reciente.

El CSV original no está en el repo por tamaño y porque son datos de terceros.
