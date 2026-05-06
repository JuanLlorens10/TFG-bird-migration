# O1 — Preparación de datos

> Referencia específica para el objetivo O1 del TFG.
> Documento de referencia para futuros cambios en `dataExploration1.ipynb` y para la redacción de la memoria.

## Objetivo

Limpieza de los registros GPS brutos de gaviotas (*Larus fuscus*) para obtener una tabla de **una localización diaria por ave** con cobertura temporal homogénea, segmentada en trayectorias continuas y lista para consumirse por los módulos de Markov (O2), HMM (O3) y ML (O4).

## Notebook

`notebooks/dataExploration1.ipynb` → produce `data/processed/aves_procesado_markov.csv`.

## Datos de entrada

`data/raw/migration_original.csv` — descarga del estudio *"Navigation experiments in lesser black-backed gulls"* (Movebank).

| Atributo | Valor |
|---|---|
| Registros | 89 867 |
| Aves únicas (`individual-local-identifier`) | 126 |
| Rango temporal | 2009-05-25 00:05 → 2015-08-27 09:00 |
| Sensor | 100 % `gps` (no hay registros de otro tipo de sensor) |
| Columnas originales | 15 |

### Columnas brutas y tratamiento

| Columna original | Acción | Justificación |
|---|---|---|
| `timestamp` | renombrada → `time` | Formato más corto |
| `location-long` | renombrada → `lon` | |
| `location-lat` | renombrada → `lat` | |
| `individual-local-identifier` | renombrada → `animal_id` | Identificador del ave |
| `ECMWF Interim Full Daily Invariant Low Vegetation Cover` | renombrada → `veg_low` | Cobertura vegetal baja en la celda (ya viene anexada al GPS por Movebank/ECMWF) |
| `ECMWF Interim Full Daily Invariant High Vegetation Cover` | renombrada → `veg_high` | Cobertura vegetal alta en la celda |
| `manually-marked-outlier` | descartada | 100 % NaN; sin información |
| `NCEP NARR SFC Vegetation at Surface` | descartada | 100 % NaN; sin información |
| `visible.1` | descartada | Idéntica a `visible` (todos `True`) |
| `visible` | descartada | Constante a `True` tras la verificación anterior |
| `event-id`, `sensor-type`, `individual-taxon-canonical-name`, `tag-local-identifier`, `study-name` | descartadas | Metadatos no necesarios para el análisis |

> **Importante**: `veg_low` y `veg_high` **no se calculan en este notebook**. Vienen ya anotadas por Movebank desde el reanalysis ECMWF Interim. Solo se renombran y se conservan.

## Pipeline de limpieza

### Paso 1 — Análisis de la rejilla temporal de los GPS

Los collares emiten en 4 ventanas diarias de ±1 hora cada una alrededor de las 5h, 8h, 14h y 20h UTC. La distribución horaria de los 89 867 registros confirma esta estructura:

| Hora pico | Registros en la hora | Suma con vecinas (h±1) |
|---|---|---|
| 14h | 15 656 | 22 375 |
| 20h | 15 613 | 22 361 |
| 8h | 15 591 | 22 313 |
| 5h | 15 466 | 22 149 |
| Resto de horas | ≤ 530 cada una | (ruido residual) |

Cada ave tiene típicamente 2–4 emisiones por día; rara vez una sola. Esto justifica reducir el dato a una emisión diaria por ave para igualar el intervalo temporal del análisis posterior.

### Paso 2 — Selección de un registro diario por ave

Para cada par `(animal_id, fecha)` se conserva el registro **más cercano a las 14:00 UTC**. Se eligen las 14h por ser el pico con mayor cobertura (22 375 emisiones en ±1 h).

Tras esta operación: **23 585 filas** (un registro/día/ave).

### Paso 3 — Filtro de tolerancia horaria

De los 23 585 registros seleccionados, se conservan únicamente los que caen en la ventana **13h–15h UTC**. Los restantes (~5 %) son días en los que el ave no emitió cerca de las 14h y el "más cercano" quedó a 20h o más, lo que rompería la homogeneidad temporal.

Resultado: **22 375 filas** (94,87 % de retención respecto al paso anterior). Distribución final por hora seleccionada:

| Hora | Registros |
|---|---|
| 14h | 15 408 |
| 13h | 4 548 |
| 15h | 2 085 |

### Paso 4 — Segmentación en trayectorias continuas (`trayectoria_id`)

Para cada ave se calcula la diferencia en días entre registros consecutivos. Se inicia un nuevo segmento cada vez que esa diferencia es **mayor que 1 día**, lo que indica un hueco en la cobertura GPS.

El identificador resultante tiene la forma `{animal_id}_Seg{n}` (p. ej. `91743A_Seg2`).

Tras la segmentación: **663 segmentos**, con una longitud media de 33,75 días.

### Paso 5 — Filtro de longitud mínima por segmento

Se descartan los segmentos con menos de **4 días consecutivos**. Justificación:
- 4 días es el mínimo necesario para que el HMM (O3) calcule al menos un `turning_angle` interior y disponga de varias transiciones internas dentro del segmento.
- Segmentos cortos generan trayectorias degeneradas para el cálculo de `step_length` y `bearing` en el HMM.

| Métrica | Antes del filtro | Después del filtro |
|---|---|---|
| Segmentos | 663 | **480** |
| Longitud media | 33,75 días | **45,92 días** |
| Registros | 22 375 | **22 041** |
| Pérdida | — | 334 filas (1,49 %) |

### Paso 6 — Exportación

Se guarda en `data/processed/aves_procesado_markov.csv` (`index=False`).

## Output: `aves_procesado_markov.csv`

| Columna | Tipo | Descripción |
|---|---|---|
| `animal_id` | str | Identificador del ave (p. ej. `91732A`) |
| `date` | date (YYYY-MM-DD) | Fecha del registro |
| `hora` | int | Hora del registro seleccionado: 13, 14 o 15 |
| `lon` | float | Longitud WGS84 |
| `lat` | float | Latitud WGS84 |
| `veg_low` | float ∈ [0, 1] | Cobertura vegetal baja (ECMWF Interim) |
| `veg_high` | float ∈ [0, 1] | Cobertura vegetal alta (ECMWF Interim) |
| `trayectoria_id` | str | Segmento continuo (`{animal_id}_Seg{n}`) |

### Estadísticas finales

| Métrica | Valor |
|---|---|
| Filas | **22 041** |
| Aves únicas | **117** (de 126 iniciales; 9 perdidas por no tener ningún segmento ≥ 4 días) |
| Trayectorias | **480** |
| Rango temporal | 2009-05-25 → 2015-08-23 |
| Longitud media de trayectoria | 45,9 días |

## Decisiones de diseño clave

1. **Una localización diaria, no una posición media** — el HMM (O3) y el cálculo de `step_length`/`bearing` requieren posiciones puntuales. Promediar varios registros del mismo día introduciría un sesgo en el desplazamiento.
2. **Ventana 13–15h en lugar de "todos los registros del día más cercanos a 14h"** — al exigir cercanía estricta a las 14h, se uniformiza el intervalo temporal entre días consecutivos en ~24 h. Sin este filtro, un día con emisión a las 20h y el siguiente a las 5h darían un `step_length` calculado sobre 9 horas, no 24.
3. **`trayectoria_id` y no series temporales por ave directamente** — un ave con un hueco GPS de varios días puede haber recorrido cientos de km sin observación. Calcular `step_length` sobre ese hueco daría un dato espurio. La segmentación garantiza que toda transición ocurre entre dos días consecutivos reales.
4. **Mínimo de 4 días por segmento** — crítico para que en O3 (HMM) se pueda calcular al menos un `turning_angle` interior (necesita 3 puntos consecutivos) y para que cada segmento tenga al menos 3 transiciones internas que aporten información a `transmat_`.
5. **`veg_low` y `veg_high` se conservan aunque este notebook no las use** — son features candidatas para HMM3 y ML5/ML6. Mantenerlas evita reabrir el CSV bruto.
6. **No se añaden aquí `grid_x`, `grid_y` ni `cell_id`** — la rejilla espacial (Markov/ML) se construye en `markov1.ipynb` a partir de este CSV. Mantener O1 estrictamente como limpieza temporal facilita reutilizar la salida con otras rejillas.

## Limitaciones y notas para la memoria

- **Pérdida total entre bruto y procesado**: 89 867 → 22 041 filas (75,5 %). La mayor parte (≈74 %) viene de quedarse con un solo registro/día; solo el 1,5 % es por filtro de longitud de segmento.
- **9 aves se pierden por completo** al no tener ningún segmento ≥ 4 días. Son aves con emisiones muy esporádicas o tags que fallaron pronto.
- **Cobertura estacional desigual**: hay más registros en primavera-verano (paso migratorio Norte→Sur de las gaviotas escandinavas) que en invierno. Esto se documenta en las gráficas mensuales/estacionales del notebook y debe tenerse en cuenta al interpretar los modelos por estación de O2.
- **Sesgo geográfico**: el estudio rastrea principalmente *Larus fuscus* del Báltico; las latitudes de cría se concentran en torno a 60° N y las de invernada en torno a 10° N (África subsahariana). Los modelos entrenados con estos datos no son extrapolables a otras poblaciones de gaviotas.
