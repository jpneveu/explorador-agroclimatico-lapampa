# La Pampa | Explorador Agroclimático: fuentes y metodología

Prototipo educativo para la exploración de información agroclimática. Lic. Juan Pablo Neveu.

Datos generados el 8 de noviembre de 2026 (UTC). Período: 1 de enero de 2020 al 31 de diciembre de 2025 (2.192 días, seis años completos).

## 1. Fuentes

| Variable | Producto | Tipo de dato | Resolución | Unidades |
|---|---|---|---|---|
| Precipitación diaria | CHIRPS v3.0 diario (final, desagregado con ERA5). Climate Hazards Center, UC Santa Barbara | Estimación satelital calibrada con estaciones | 0,05° (≈ 5 km), diaria | mm/día |
| Temperatura máxima diaria | CHIRTS-ERA5 diario (producto experimental). Climate Hazards Center, UC Santa Barbara | Estimación satelital combinada con reanálisis ERA5 | 0,05° (≈ 5 km), diaria | °C |
| Temperatura mínima diaria | CHIRTS-ERA5 diario (producto experimental) | Ídem | 0,05° (≈ 5 km), diaria | °C |
| Temperatura media | Calculada a partir de CHIRTS-ERA5 | Indicador derivado | 0,05°, mensual | °C |
| Límite provincial | GADM 4.1, nivel 1, `ARG.11_1` (La Pampa) | Cartografía vectorial | — | — |
| Departamentos | GADM 4.1, nivel 2, 22 departamentos de `ARG.11_1` | Cartografía vectorial | — | — |

Ninguna variable es una observación directa de estación meteorológica: todas son estimaciones en grilla.

**Acceso a los datos.** La descarga se hizo con el paquete `aggeodata` (CGIAR Climate Data Hub), siguiendo la skill `climate-data-download` (pipeline YAML, `use_zarr: true`). Se leyó desde los almacenes Zarr que replican los archivos del Climate Hazards Center:

- CHIRPS v3.0: `https://huggingface.co/datasets/aaguilar90/chirps-v3-daily-rnl` (variable `precip`, cobertura verificada 1981-01-01 a 2026-08-31).
- CHIRTS-ERA5: `https://huggingface.co/datasets/aaguilar90/chirts-era5-daily` (variables `Tmax`, `Tmin`, cobertura verificada 1981-01-01 a 2026-10-21).
- GADM 4.1: `https://geodata.ucdavis.edu/gadm/gadm4.1/json/gadm41_ARG_1.json` y `gadm41_ARG_2.json`.

Referencias:

- Funk, C. et al. (2015). The climate hazards infrared precipitation with stations — a new environmental record for monitoring extremes. *Scientific Data* 2, 150066.
- Verdin, A. et al. (2020). Development and validation of the CHIRTS-daily quasi-global high-resolution daily temperature data set. *Scientific Data* 7, 303.
- Climate Hazards Center. CHIRPS v3.0 readme: https://data.chc.ucsb.edu/products/CHIRPS/v3.0/daily/readme.txt
- GADM, versión 4.1: https://gadm.org

## 2. Procesamiento

1. **Descarga.** Se descargó el rectángulo envolvente de La Pampa con un margen de 0,1° (`[-68,40; -39,42; -63,28; -34,90]`). El rectángulo solo sirvió para la descarga y no se usó en ningún cálculo.
2. **Recorte por geometría real.** Se usaron solo las celdas cuyo **centro** cae dentro del polígono GADM de La Pampa (5.780 celdas; 142.741 km² frente a 142.951 km² del polígono). Se verificó que la función `mask_to_admin` de la skill `geospatial-cube-processor` (`rio.clip`) selecciona exactamente las mismas celdas. A diferencia del valor por defecto de la skill, se usó `all_touched=False`, para no incluir celdas que solo rozan el límite.
3. **Asignación a departamentos.** Cada celda se asignó al departamento que contiene su centro. Las 5.780 celdas quedaron asignadas.
4. **Acumulados de precipitación.** Para cada celda se sumó la precipitación diaria dentro de cada mes. El acumulado anual es la suma de los 12 meses.
5. **Promedios espaciales.** El valor provincial o departamental es el promedio de las celdas **ponderado por área** (coseno de la latitud). Nunca se suman lluvias de distintas celdas: el acumulado provincial es la lámina media de agua sobre el territorio.
6. **Temperaturas máxima y mínima.** Promedio mensual de los valores diarios de cada celda, y luego promedio espacial ponderado. El promedio anual pondera cada mes por su cantidad de días.
7. **Temperatura media.** Se calculó como (Tmax + Tmin) / 2 para cada día y celda, y luego se promedió como las demás temperaturas. Es una aproximación habitual en agrometeorología y **no** equivale al promedio de 24 observaciones horarias.
8. **Días con precipitación.** Días con al menos **1 mm** en cada celda (umbral habitual de la OMM y del ETCCDI). Se cuentan por celda y se promedian espacialmente. Por eso el valor puede tener decimales.
9. **Mapa.** Muestra el valor de cada celda. La escala de colores es común a los seis años para el mismo período (año completo o un mismo mes), así los colores se pueden comparar entre años. La marca negra de la leyenda indica la media provincial ponderada, que coincide con el indicador correspondiente.
10. **Referencia 2020–2025.** Los indicadores y la serie anual muestran el promedio de los seis años como referencia descriptiva. **No es una normal climática**, que requiere 30 años.

## 3. Controles de integridad

| Control | Resultado |
|---|---|
| Días por variable | 2.192 en las tres variables (sin días faltantes) |
| Valores faltantes dentro de la provincia | 0 en precipitación, 0 en Tmax y 0 en Tmin |
| Grillas | CHIRPS y CHIRTS comparten la misma grilla de 0,05° (diferencias de coordenadas < 1e-13°) |
| Rango de precipitación diaria | 0 a 150,2 mm (sin negativos) |
| Rango de Tmax diaria | 1,2 a 44,1 °C |
| Rango de Tmin diaria | −10,9 a 27,8 °C |
| Casos con Tmin > Tmax | 0 |
| Acumulado provincial calculado por dos caminos (suma de medias diarias / media de sumas por celda) | Diferencia máxima 5e-12 mm |
| Promedio de departamentos ponderado por área frente al valor provincial | Diferencia máxima 2e-13 mm |
| Contraste puntual con el almacén de origen (4 celdas y fechas al azar) | Coincidencia exacta en precipitación y Tmax |
| Recorte propio frente a `rio.clip` de la skill | Mismo número de celdas (5.780) |
| Tablero frente al CSV (8.970 filas) | Diferencia máxima 0,03 (redondeo de los valores incrustados) |
| Mapa frente a indicadores (390 combinaciones de variable, año y mes) | Diferencia máxima 0,02 mm y 0,005 °C |

El detalle está en `trazabilidad/qc_report.json`.

## 4. Limitaciones

- **Escala y comparación con registros locales.** Cada valor del mapa representa una celda de 0,05° (≈ 5,5 km norte-sur × 4,5 km este-oeste en La Pampa). Los indicadores y los gráficos muestran la lámina media o la temperatura media de la provincia o del departamento elegido, no un punto. Por esa escala y por el error propio de toda estimación, los valores pueden diferir de las mediciones puntuales, como los pluviómetros del SMN o del INTA Anguil. Las diferencias son mayores a escala diaria y en las tormentas convectivas, frecuentes de primavera a comienzos del otoño, y se reducen en los acumulados mensuales y anuales. Comparar estos valores con registros locales es una actividad de aula recomendada.
- **Precipitación diaria.** CHIRPS v3 diario distribuye en días los acumulados de 5 días (péntadas) usando ERA5. Los acumulados mensuales y anuales son más confiables que la lluvia de un día puntual. La cantidad de días con lluvia depende de esa desagregación y debe leerse con cautela.
- **Temperatura.** CHIRTS-ERA5 es un producto **experimental**, validado por sus autores a escala global (Verdin et al., 2020). En este prototipo **no** se contrastó con estaciones de la provincia (SMN, INTA). Antes de usar los valores absolutos de temperatura, especialmente la mínima, se recomienda compararlos con registros de estaciones de la provincia.
- **Período corto.** Seis años permiten describir la variabilidad interanual, pero no tendencias. Las diferencias entre dos años **no** son evidencia de cambio climático.
- **Relaciones entre variables.** Las coincidencias que se ven en los gráficos (por ejemplo, entre temperatura y lluvia) son descriptivas y no implican causalidad.
- **Límites administrativos.** Se usaron los límites de GADM 4.1. Pueden diferir en detalle de la cartografía oficial del IGN o de la Dirección Provincial de Catastro.

## 5. Nota visible en el explorador

Debajo del título, el explorador muestra una franja que se puede desplegar y ocultar, y que vuelve a aparecer cada vez que se abre:

> **Estos datos son estimaciones, no mediciones de estación.** Pueden diferir de los registros locales.
>
> **Sobre estos datos.** La precipitación (CHIRPS v3) y las temperaturas (CHIRTS-ERA5) son estimaciones en grillas de ≈ 5 km, elaboradas con satélites, estaciones y reanálisis. Los indicadores y gráficos muestran la lámina media o la temperatura media de la provincia o del departamento elegido; el mapa muestra el valor de cada celda.
>
> Por esa escala y por el error propio de toda estimación, los valores pueden diferir de las mediciones puntuales, como los pluviómetros del SMN o del INTA Anguil. Las diferencias son mayores a escala diaria y en las tormentas convectivas de primavera a comienzos del otoño, y se reducen en los acumulados mensuales y anuales. Por el mismo motivo, los «días con precipitación» son el indicador menos preciso.
>
> Las temperaturas aún no fueron contrastadas con estaciones de la provincia, y seis años permiten observar la variabilidad entre años, no tendencias. Comparar estos valores con registros locales es una buena actividad para el aula.

## 6. Archivos

| Archivo | Contenido |
|---|---|
| `Explorador_Agroclimatico_La_Pampa.html` | Tablero autónomo. Incluye Chart.js 4.4.0 y los datos comprimidos. Funciona sin conexión en Chrome, Edge y Firefox actuales. |
| `datos_agroclimaticos_lapampa.csv` | Datos procesados: provincia y 22 departamentos, valores mensuales y anuales de 5 variables (8.970 filas, UTF-8). |
| `fuentes_y_metodologia.md` | Este documento. |
| `trazabilidad/lapampa.yaml` | Configuración de descarga de `aggeodata`. |
| `trazabilidad/process_lapampa.py` | Script de recorte, agregación y controles. |
| `trazabilidad/build_html.py` y `template.html` | Armado del HTML autónomo. |
| `trazabilidad/qc_report.json` | Resultados de los controles de integridad. |

### Columnas del CSV

| Columna | Descripción |
|---|---|
| `nivel` | `provincia` o `departamento` |
| `unidad` | Nombre de la provincia o del departamento |
| `gid_gadm` | Identificador GADM 4.1 |
| `variable` | `precipitacion`, `temp_maxima`, `temp_minima`, `temp_media`, `dias_con_precipitacion` |
| `anio`, `mes` | Año y mes (el mes queda vacío en las filas anuales) |
| `periodo` | `mensual` o `anual` |
| `valor` | Valor del indicador |
| `unidad_medida` | mm, °C o días |
| `agregacion_temporal` | `suma` o `promedio` |
| `n_celdas` | Celdas de 0,05° incluidas en el promedio espacial |
| `celdas_faltantes` | Celdas sin dato en ese mes |
| `tipo_de_dato` | Estimación satelital, satelital-reanálisis o indicador derivado |
| `fuente` | Producto de origen |

### Reproducir el procesamiento

```bash
python -m aggeodata.pipelines.download trazabilidad/lapampa.yaml
python -I trazabilidad/process_lapampa.py <carpeta_datos>
```

Requiere `aggeodata[download]` (CGIAR Climate Data Hub), con xarray, rioxarray, rasterio y geopandas.
