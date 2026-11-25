# AgroTwin — Datasets del modelo de rendimiento

Datos con los que se entrena el Random Forest de **AgroTwin**, que predice el rendimiento (kg/ha) de soja de primera, maíz y trigo por departamento y campaña en Argentina. Forma parte de la tesis de Ingeniería Informática (UADE) de Julian Maistruk y Manuel Figuerero Mantilla.

Este repositorio contiene solo los datos. El código que los descarga, los procesa y entrena el modelo está en el repositorio AgroTwin, carpeta `ml/`.

## Contenido

| Archivo | Filas | Qué es |
|---|---:|---|
| `data/raw/estimaciones-agricolas.csv` | 160.499 | Estimaciones Agrícolas del MAGyP: 41 cultivos, campañas 1969/1970 a 2024/2025 |
| `data/raw/departamentos.csv` | 529 | Departamentos argentinos con su centroide (Georef) |
| `data/raw/clima_por_campania.csv` | 13.060 | Clima de Open-Meteo, ya agregado por departamento, campaña y estación |
| `data/processed/dataset_entrenamiento.csv` | 19.174 | **Dataset con el que se entrena el modelo** |
| `data/processed/climatologia_normal.csv` | 923 | Clima promedio por departamento y cultivo; se usa al predecir, no al entrenar |

Los CSV están en UTF-8, separados por coma y con encabezado. Los códigos `departamento_id` y `provincia_id` son del INDEC y llevan ceros a la izquierda, así que hay que leerlos como texto:

```python
import pandas as pd

df = pd.read_csv("data/processed/dataset_entrenamiento.csv", dtype={"departamento_id": str})
```

## Cómo se arma el dataset de entrenamiento

1. Se toman del MAGyP los cultivos `maíz`, `soja 1ra` y `trigo total`, desde la campaña 2000/2001: 19.433 filas.
2. Se agregan las coordenadas del departamento, uniendo por `departamento_id`.
3. Se agrega el clima, uniendo por `departamento_id`, `campania` y `estacion`.
4. Se descartan las 259 filas sin unión completa: 179 sin coordenadas y 80 sin clima descargado.

Quedan 19.174 filas (98,7 %), una por combinación de departamento, campaña y cultivo, sin duplicados ni valores nulos. Cubren 337 departamentos de 15 provincias y 25 campañas.

| Cultivo | Filas | Rendimiento medio (kg/ha) | Mediana | Máximo |
|---|---:|---:|---:|---:|
| `MAIZ` | 7.108 | 5.502 | 5.600 | 14.933 |
| `SOJA` | 6.405 | 2.545 | 2.600 | 7.253 |
| `TRIGO` | 5.661 | 2.575 | 2.498 | 7.186 |

### Separación entre entrenamiento y prueba

La separación es temporal y se calcula al entrenar; no hay archivos separados.

| Conjunto | Campañas | Filas |
|---|---|---:|
| Entrenamiento | 2000/2001 a 2019/2020 | 15.006 |
| Prueba | 2020/2021 a 2024/2025 | 4.168 |

Las métricas se miden sobre el conjunto de prueba. El modelo desplegado se reentrena después con las 19.174 filas.

## Diccionario de datos

### `dataset_entrenamiento.csv`

| Columna | Unidad | Descripción | Uso en el modelo |
|---|---|---|---|
| `tipo_cultivo` | — | `MAIZ`, `SOJA` o `TRIGO` | Entrada (categórica) |
| `campania` | — | Campaña, por ejemplo `2000/2001` | Define la separación temporal |
| `anio_inicio_campania` | año | Primer año de la campaña | Entrada |
| `provincia`, `departamento` | — | Nombres según el MAGyP | No se usa |
| `departamento_id` | — | Código INDEC de 5 dígitos | No se usa |
| `lat`, `lon` | grados | Centroide del departamento | Entrada |
| `superficie_sembrada_ha` | ha | Superficie sembrada | No se usa |
| `precipitacion_acumulada_mm` | mm | Lluvia total en la ventana de cultivo | Entrada |
| `temp_max_promedio` | °C | Promedio de las máximas diarias | Entrada |
| `temp_min_promedio` | °C | Promedio de las mínimas diarias | Entrada |
| `temp_max_absoluta` | °C | Máxima más alta de la ventana | Entrada |
| `et0_acumulado_mm` | mm | Evapotranspiración de referencia (FAO) total | Entrada |
| `dias_con_lluvia` | días | Días con más de 1 mm | Entrada |
| `racha_seca_max_dias` | días | Racha más larga de días con 1 mm o menos | Entrada |
| `dias_calor_extremo` | días | Días con máxima de 34 °C o más | Entrada |
| `rendimiento_kgxha` | kg/ha | Rendimiento observado | **Variable objetivo** |

### `clima_por_campania.csv`

Clave: `departamento_id`, `campania`, `estacion`. Las ocho variables climáticas son las de la tabla anterior, calculadas sobre esta ventana:

| `estacion` | Cultivos | Ventana |
|---|---|---|
| `verano` | Maíz y soja | 1 de octubre al 30 de abril del año siguiente |
| `invierno` | Trigo | 1 de mayo al 15 de diciembre |

### `climatologia_normal.csv`

Clave: `departamento_id`, `tipo_cultivo`. Promedio de las ocho variables climáticas en las campañas del dataset. Reemplaza al clima real cuando se predice una campaña que todavía no terminó.

### `estimaciones-agricolas.csv`

`cultivo`, `anio`, `campania`, `provincia`, `provincia_id`, `departamento`, `departamento_id`, `superficie_sembrada_ha`, `superficie_cosechada_ha`, `produccion_tm` (toneladas) y `rendimiento_kgxha`. Es la descarga completa del MAGyP, sin filtrar.

### `departamentos.csv`

`departamento_id`, `departamento_nombre`, `provincia_id`, `provincia_nombre`, `lat` y `lon`.

## Limitaciones conocidas

- **El clima de `raw/` no es diario.** La descarga agrega los datos al vuelo y no conserva la serie diaria. Para calcular otra variable climática hay que volver a descargar.
- **No se fijó un modelo climático.** Las consultas a Open-Meteo usan el modelo por defecto de su Archive API, que puede cambiar con el tiempo; una nueva descarga podría dar valores algo distintos.
- **Chascomús queda afuera.** Son 75 filas que el MAGyP informa con el código `06217`, que no figura en el listado de Georef. Las otras 104 filas sin coordenadas son de departamentos «sin definir».
- **Hay 213 filas con rendimiento 0.** El modelo las usa, pero se excluyen del cálculo del MAPE.
- **Hay lluvias muy altas en departamentos de Tucumán y Jujuy.** El máximo es 4.687,7 mm, en el departamento `90021` (campaña 2018/2019). Son valores tal como los devolvió Open-Meteo, sin corregir.
- **La ventana de cultivo es única para todo el país.** No sigue los calendarios regionales de siembra.
- **No hay datos de suelo ni de manejo,** y el clima es el del centroide del departamento.
- **En el archivo del MAGyP,** 8 filas no tienen departamento, y en té y yerba mate `anio` no siempre coincide con el inicio de la campaña. Ninguno de los dos casos afecta a los cultivos usados.

## Verificar y reproducir

Comprobar que los archivos no cambiaron:

```bash
shasum -a 256 -c SHA256SUMS
```

Regenerar los archivos de `processed/` a partir de los de `raw/`: copiar `data/raw/` en `ml/data/raw/` del repositorio AgroTwin, instalar `ml/requirements.txt` y ejecutar `python src/build_dataset.py` desde `ml/`. Con pandas 3.0.5 el resultado es idéntico byte a byte al publicado acá.

## Fuentes y licencias

| Fuente | Archivos | Licencia |
|---|---|---|
| [MAGyP, Dirección de Estimaciones Agrícolas](https://datos.magyp.gob.ar/dataset/estimaciones-agricolas) | `estimaciones-agricolas.csv` | CC BY 4.0 |
| [Georef, datos.gob.ar](https://datosgobar.github.io/georef-ar-api/terms/) | `departamentos.csv` | CC BY 4.0 |
| [Open-Meteo, Archive API](https://open-meteo.com/en/licence) (reanálisis de Copernicus) | `clima_por_campania.csv` | CC BY 4.0 |

[Texto de la licencia CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.es). Los archivos de `processed/` son obras derivadas de las tres fuentes: datos filtrados, unidos y agregados por los autores de AgroTwin. `clima_por_campania.csv` agrega los datos diarios de Open-Meteo. `departamentos.csv` conserva un subconjunto de campos de Georef.
