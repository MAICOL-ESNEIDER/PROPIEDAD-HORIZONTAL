# Distribución de la propiedad horizontal en las comunas de Medellín

Proyecto individual de ciencia de datos del programa Tecnólogo en Análisis y Desarrollo de Software (SENA), ficha 3223873.

**Autor:** Maicol Esneider Posada Callejas · **Instructor:** Carlos Navia

## Descripción

El proyecto analiza la información catastral abierta de Medellín para saber dónde se concentran las viviendas en propiedad horizontal (conjuntos residenciales y edificios de apartamentos) y qué proporción representan en cada comuna.

| Etapa | Evidencia | Notebook | Documento |
|---|---|---|---|
| Diagnóstico de calidad de los datos | EV05 | `notebooks/data_quality.ipynb` | `reports/data_quality.md` |
| Limpieza, análisis y estructuración | EV07 | `notebooks/data_preparation.ipynb` | Este README (sección *Transformaciones realizadas*) |

## Propósito

Identificar las comunas con más propiedad horizontal, y en especial con más conjuntos y edificios grandes, para decidir dónde enfocar primero Domux, un software de administración de propiedades horizontales que se desarrolla como proyecto de grado.

## Pregunta analítica

> ¿En qué comunas de Medellín se concentran los predios habitacionales en propiedad horizontal y qué proporción representan frente al total de predios habitacionales de cada comuna?

## Fuentes de datos

Los CSV originales **no se publican en el repositorio** por su tamaño (el de predios pesa unos 92 MB). Se descargan y se guardan en `data/raw/` con los nombres indicados, sin modificarlos.

| Dataset | Archivo en `data/raw/` | Fuente y descarga | Papel en el proyecto |
|---|---|---|---|
| Información Predios | `informacion_predios.csv` | MEData, Alcaldía de Medellín (2021): https://medata.gov.co/node/16594 | Fuente principal: un registro por predio |
| Información Lotes | `informacion_lotes.csv` | MEData, Alcaldía de Medellín (2021): https://medata.gov.co/node/16593 | Área y pisos de cada lote; se une con Predios por `cbml` |
| Medellín Properties (2023) | `medellin_properties.csv` | Kaggle: https://www.kaggle.com/datasets/cesaregr/medelln-properties | Apoyo: valor de la administración por estrato |
| Precios inmuebles Medellín | `houses_medellin.csv` | Kaggle: https://www.kaggle.com/datasets/valentinafeve/precios-inmuebles-medelln | Apoyo |

Los diccionarios de datos de MEData (`Informacion_Predios.json` e `Informacion_Lotes.json`) están incluidos en `data/raw/`.

## Estructura del repositorio

```
PROPIEDAD-HORIZONTAL/
├── README.md
├── .gitignore
├── requirements.txt
├── data/
│   ├── raw/                              # CSV originales (ignorados por Git) y diccionarios JSON
│   └── processed/                        # Datasets limpios y estructurados
│       ├── cleaning_log.csv
│       ├── properties_clean.parquet
│       ├── lots_clean.parquet
│       ├── medellin_properties_clean.csv
│       ├── houses_medellin_clean.csv
│       ├── residential_properties.parquet
│       ├── horizontal_property_lots.csv
│       ├── comuna_summary.csv
│       └── territory_summary.csv
├── notebooks/
│   ├── data_quality.ipynb                # Diagnóstico de calidad (EV05)
│   └── data_preparation.ipynb            # Limpieza, análisis y estructuración (EV07)
└── reports/
    └── data_quality.md                   # Informe del diagnóstico de calidad
```

## Transformaciones realizadas

### 1. Limpieza y estandarización

Cada dataset se limpia con un pipeline de funciones encadenadas con `pipe()`. Cada paso queda en `data/processed/cleaning_log.csv` y las reglas de calidad se comprueban con `assert`.

| Dataset | Transformaciones |
|---|---|
| Predios | Columnas renombradas a un estándar único; `cbml` convertido a texto de 11 dígitos (recupera el cero inicial); `matricula` a texto; columna `nro_propietarios` eliminada (siempre 0); "--" en `urbanizacion` y estrato 0 convertidos a nulo; comunas mal escritas corregidas ("Manrrique" → "Manrique", "La Ameria" → "La América"); texto estandarizado; nueva columna `es_vivienda` (uso predial 1) |
| Lotes | `cbml` y `zona_geo` a texto; área de lote 0 convertida a nulo; columna `pisos_incoherentes` para los lotes con área construida y pisos contradictorios |
| Medellín Properties | 116 duplicados eliminados; coordenadas a número y fuera de Medellín a nulo; "Más de 10" en garajes, área 0, estrato 0 y barrio "Otro" a nulo; antigüedad como categoría ordenada; columna `valor_atipico` para precios y áreas imposibles |
| Precios inmuebles | Columna de índice eliminada; 121 filas que solo tenían el precio eliminadas; montos y áreas convertidos de texto a número; "No definida" y "Otro" a nulo; 98 duplicados eliminados |

**Uso predial de vivienda.** El código 1 de `uso_predial` se identificó como vivienda porque el 100 % de esos predios tiene estrato socioeconómico, que en Colombia solo se asigna a la vivienda.

### 2. Filtrado, agrupación e indicadores

- **Vivienda en propiedad horizontal:** predio de vivienda con porcentaje de copropiedad (`por_des`) menor que 100, es decir, que comparte el lote con otros predios.
- **Tamaño de la propiedad horizontal:** según las viviendas en PH del lote: pequeña (1 a 4), mediana (5 a 20) y grande (más de 20).
- **Indicadores por comuna:** viviendas en PH, proporción de PH en la comuna, participación en la ciudad, viviendas en PH grande y proporción de PH grande. Se completan con tablas dinámicas por estrato y por tamaño, y con la mediana de la administración por estrato (Kaggle).

### 3. Combinación y estructuración

- Predios y Lotes se unen por `cbml` con `merge(validate='m:1')`. La unión conserva las 575.614 viviendas de las 16 comunas, sin perder ni duplicar filas.
- Se crea una tabla con un lote en propiedad horizontal por fila y una tabla final de indicadores por comuna (unión 1:1). Las comunas y los corregimientos se apilan con `concat()`.
- Los datasets de Kaggle **no se combinan** con los de MEData, porque no tienen una clave común: unirlos por nombre de barrio sería una combinación artificial.

### Datasets procesados

| Archivo | Contenido |
|---|---|
| `cleaning_log.csv` | Registro de cambios de la limpieza |
| `properties_clean.parquet` | Predios limpios (1.041.413) |
| `lots_clean.parquet` | Lotes limpios (308.129) |
| `medellin_properties_clean.csv` | Medellín Properties limpio (9.883) |
| `houses_medellin_clean.csv` | Precios inmuebles limpio (211) |
| `residential_properties.parquet` | Viviendas de las 16 comunas unidas con su lote (575.614) |
| `horizontal_property_lots.csv` | Un lote en propiedad horizontal por fila (83.005) |
| `comuna_summary.csv` | Indicadores por comuna (dataset final para responder la pregunta) |
| `territory_summary.csv` | Viviendas y propiedad horizontal de las 16 comunas y los 5 corregimientos |

### Resultados principales

- El 75,2 % de las viviendas de las 16 comunas está en propiedad horizontal.
- Belén, El Poblado, Robledo, Laureles y Buenos Aires reúnen el 49,5 % de las viviendas en propiedad horizontal de la ciudad.
- La proporción va del 39,0 % en Popular al 97,1 % en El Poblado.
- El Poblado, Belén, Robledo, La Candelaria y Laureles concentran el 71,6 % de las viviendas en conjuntos y edificios grandes (más de 20 viviendas).

## Instrucciones de ejecución

Requisitos: Python 3.11 o superior y Git.

1. Clonar el repositorio y entrar a la carpeta:
   ```
   git clone https://github.com/MAICOL-ESNEIDER/PROPIEDAD-HORIZONTAL.git
   cd PROPIEDAD-HORIZONTAL
   ```
2. Crear y activar el entorno virtual:
   ```
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1        # Windows PowerShell
   source .venv/bin/activate          # macOS y Linux
   ```
3. Instalar las dependencias (pandas, pyarrow para los archivos Parquet y JupyterLab):
   ```
   python -m pip install -r requirements.txt
   ```
4. Descargar los cuatro CSV (tabla de *Fuentes de datos*) y guardarlos en `data/raw/`.
5. Abrir JupyterLab desde la raíz del proyecto:
   ```
   jupyter lab
   ```
6. Ejecutar los notebooks de la carpeta `notebooks/` con *Run → Run All Cells*:
   - `data_quality.ipynb`: diagnóstico de calidad.
   - `data_preparation.ipynb`: limpieza, análisis y estructuración. Al terminar, los datasets procesados quedan en `data/processed/`.

## Limitaciones

- Los datos de MEData son de 2021.
- El código de vivienda (`uso_predial = 1`) es una inferencia basada en los datos, no una tabla oficial de Catastro.
- La propiedad horizontal se estima con el porcentaje de copropiedad, porque los datos no la marcan directamente.
