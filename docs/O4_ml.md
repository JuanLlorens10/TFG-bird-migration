# O4 — Predicción con Machine Learning

> Referencia específica para el objetivo O4 del TFG.
> Actualizar tras cada sesión de trabajo en este objetivo.

## Objetivo

Entrenar y comparar modelos de clasificación (RF, XGBoost, LightGBM) para predecir la `target_cell` (celda que ocupará el ave al día siguiente) usando fecha, inercia del movimiento y estado HMM.

## Notebooks

Línea principal (todavía con leakage en `step_length`/`bearing` por venir como salto t→t+1):

`ML3.ipynb` → `ML4.ipynb` → `ML5.ipynb` → `ML6.ipynb`

ML3–ML5 leen `data/processed/hmm5.csv` (versión canónica del HMM; `estado_hmm` de HMM5). ML6 lee `data/processed/hmm_wind.csv` (extiende `hmm.csv`/HMM1 con viento ERA5 — no migrado a HMM5 porque requeriría reconstruir el dataset de viento).

> **Nota sobre leakage**: cambiar a `hmm5.csv` actualiza el `estado_hmm` a las etiquetas de HMM5, pero `step_length` y `bearing` en el CSV siguen siendo el salto t→t+1 (leakage respecto al target `cell(t+1)`). ML3–ML5 heredan este leakage; solo ML0 lo corrige.

Línea paralela limpia:

`ML0.ipynb` — baseline rediseñado desde cero sobre `hmm5.csv`, recalculando las features de movimiento como salto t-1 → t para eliminar el leakage. Pensado como referencia limpia para construir variantes posteriores (ML0b, ML0c…) midiendo qué aporta cada decisión adicional.

> Nota: `ML1.ipynb` y `ML2.ipynb` se eliminaron del repositorio (commit de limpieza del 2026-05-06). Sus aportaciones — fase lunar y día de la semana como features temporales — quedaron descartadas en ML3 sin aportar señal. Las filas ML1/ML2 de la tabla siguiente se conservan como contexto histórico de la evolución; los notebooks ya no existen en disco. Si se necesita recuperarlos, están en el historial de git.

## Split por animal

Primer 80 % cronológico → train; último 20 % → test. El `LabelEncoder` se ajusta solo sobre celdas de train; las filas de test con celdas no vistas se descartan.

---

## Evolución de los modelos

| Versión | Cambio principal | Migración Top-1 | Estacionario Top-1 | Global Top-1 |
|---------|-----------------|-----------------|-------------------|--------------|
| ML1 † | Baseline: mes + día semana + fase lunar | — | — | 80,3 % |
| ML2 † | Quita fase lunar | — | — | 80,8 % |
| ML3 (HMM1) ‡ | Solo semana del año | 47,9 % | 87,2 % | 81,1 % |
| **ML3 (HMM5)** | Solo semana del año, etiquetas HMM5 | **26,7 %** | **86,4 %** | **81,1 %** |
| ML4 | Modelos separados por estado + lag-1 | 44,5 % ‡ | 89,2 % ‡ | 82,3 % ‡ |
| **ML5** | +lag-2, racha, episodio, dist. latitudinal, delta_bearing | **43,4 % ‡** | **89,8 % ‡** | **82,9 % ‡** |
| ML6 | +viento ERA5 a 850 hPa (u, v, speed, tail, cross) | 41,6 % | 88,4 % | — |
| ML0 | Línea paralela limpia sobre `hmm5.csv`, anti-leakage (12 features) | pendiente | pendiente | pendiente |

† Notebook eliminado del repositorio el 2026-05-06; el resultado se conserva como contexto histórico (recuperable desde el historial de git).

‡ Resultados históricos obtenidos sobre `hmm.csv` (etiquetas HMM1). ML4 y ML5 todavía están **pendientes de re-ejecución** sobre `hmm5.csv`; ML3 ya re-ejecutado el 2026-05-09.

**ML5 sigue siendo la mejor versión de la línea histórica** (con etiquetas HMM1). El global Top-1 de ML3 con HMM5 es prácticamente idéntico al original (81,1 %), y la métrica de estacionario apenas cambia (87,2 → 86,4 %). Sin embargo la métrica de migración cae en aparente empeoramiento (47,9 → 26,7 %); el análisis detallado de por qué ocurre esto está en la sección "Análisis: por qué cambia la métrica de migración al pasar a HMM5" más abajo. ML6 empeora a pesar de añadir viento ERA5. ML0 está pendiente de ejecución completa.

---

## Features de ML5 (18 en total)

```
Posición:       grid_x, grid_y
Movimiento:     step_length, bearing, turning_angle
Inercia lag-1:  step_lag1, bearing_lag1
Inercia lag-2:  step_lag2, bearing_lag2
Episodio:       streak_dias, cum_dist_episodio, mean_bearing_episodio
Dirección:      delta_bearing
Latitud:        dist_cria (|lat-60°|), dist_invernada (|lat-10°|)
Hábitat/tiempo: veg_low, veg_high, semana_num
```

## Features de ML6 (23 en total = ML5 + viento ERA5)

```
[todas las de ML5]
Viento ERA5:    u_wind, v_wind, wind_speed, tail_wind, cross_wind
```

## Features de ML0 (12 en total)

ML0 corrige el leakage de movimiento de `hmm.csv` recalculando el salto t-1 → t en lugar de heredar el t → t+1 del CSV. Codifica la circularidad del calendario y del rumbo con sin/cos.

```
Posición:       grid_x, grid_y
Tiempo:         mes_num, sin_mes, cos_mes
Movimiento:     step_prev, sin_bearing, cos_bearing
Hábitat:        veg_low, veg_high
Fotoperiodo:    daylight_h
Comportamiento: estado_hmm   (etiquetas de HMM5 — modelo canónico del TFG)
```

Detalles del baseline en `notebooks/O4_plan.md` §1.

---

## Error geográfico (ML5, Random Forest)

| Estado | Mediana | ≤50 km | ≤100 km |
|--------|---------|--------|---------|
| Migración | 32 km | 62 % | 78 % |
| Estacionario | 5 km | 98,6 % | 98,8 % |

---

## Conclusiones clave

- **Brecha migración/estacionario**: el modelo predice bien dónde estará un ave en reposo (~90 %) pero mal adónde irá en migración (~44 %). La causa es que los saltos migratorios dependen del viento, que no está en los datos.
- **Techo de migración sin viento**: ~44–48 % Top-1. Para superarlo habría que integrar datos ERA5 (Copernicus) a baja altitud.
- **El viento ERA5 a 850 hPa no mejora (ML6)**: empeora ambos estados (migración: 41,6 % vs 43,4 %; estacionario: 88,4 % vs 89,8 %). El viento a ese nivel de presión no captura bien el viento que experimenta el ave a baja altitud.
- **LightGBM no funciona** con la configuración actual (~8–41 % según versión). Con +850 clases y `class_weight='balanced'` en la API sklearn de LightGBM 4.6 necesita tuning específico. No usarlo sin esa búsqueda.
- **XGBoost ≈ Random Forest** en resultados pero tarda 3–4× más en entrenar.

---

## Análisis: por qué cambia la métrica de migración al pasar a HMM5

Tras re-ejecutar ML3 con `hmm5.csv` (2026-05-09) la métrica de migración pasa de 47,9 % a 26,7 % (Top-1) mientras que la métrica global se mantiene en 81,1 % y la estacionaria apenas baja (87,2 → 86,4 %). El "empeoramiento" es **superficial**: refleja un cambio en qué días se etiquetan como migración, no una pérdida de capacidad predictiva del modelo.

### Comparativa numérica entre ejecuciones

| | Ejecución previa (HMM1) | Ejecución actual (HMM5) |
|---|---|---|
| Train total | 16 323 | 16 415 (+92) |
| Test total | 4 137 | 4 162 (+25) |
| **Migración en test** | **584** | **334** (−43 %) |
| Estacionario en test | 3 181 | 3 448 |
| Random Forest Top-1 | 81,14 % | 81,15 % |
| Random Forest Top-3 | 88,21 % | 88,29 % |
| Random Forest Top-5 | 89,67 % | 89,79 % |
| XGBoost Top-1 | 81,09 % | 80,62 % |
| LightGBM Top-1 | 42,87 % | 36,81 % |
| Migración Top-1 | 47,95 % | 26,65 % |
| Estacionario Top-1 | 87,24 % | 86,43 % |

### Causas

**1. El test set tiene mucha menos migración esta vez (584 → 334, −43 %).** HMM5 es más estricto que HMM1 etiquetando migración (% global: 14,4 % vs 17–25 % de HMM1). El split cronológico 80/20 por animal hace que esa caída se concentre en el último 20 % del histórico de cada ave. Con n=334 la métrica también tiene más varianza estadística.

**2. El split cronológico descarta sistemáticamente migraciones reales.** El `LabelEncoder` se ajusta solo sobre celdas vistas en train; las filas de test cuyo `target_cell` nunca aparece en train se descartan. De los 509 días de migración del test (calculados antes del filtrado), **175 (34 %)** caen fuera porque su celda destino nunca fue visitada en train — son "primeras llegadas" del ave a un sitio invernal nuevo. Solo el 5,6 % de los días estacionarios sufren ese descarte. Las migraciones que sobreviven al filtrado son commutes recurrentes a celdas ya conocidas, no las migraciones largas y novedosas.

**3. Las migraciones que quedan son intrínsecamente difíciles.** Las features disponibles (`grid_x`, `grid_y` actual, `step_length` previo, `bearing`, `semana_num`) tienen muy poca capacidad predictiva sobre el destino de un salto de 100–500 km. Un ave en migración puede aterrizar en cualquiera de ~50 celdas alcanzables.

**4. La cifra anterior (47,9 %) estaba inflada por commutes "falsos positivos" de HMM1.** HMM1 etiquetaba como migración días de movimiento medio (10–50 km, commute Galicia ↔ vertedero) que terminaban en celdas visitadas con frecuencia — fáciles de predecir. Al limpiar HMM5 ese tipo de día, el accuracy de "migración" baja porque ahora la clase contiene migraciones de verdad, donde el modelo no tiene señal.

**5. LightGBM cae más que RF/XGB (−6 pp) por sensibilidad al binning.** LightGBM cuantiza las features en histogramas antes de buscar splits; cambios pequeños en la distribución (las 92 filas adicionales de train) producen splits diferentes que se propagan en los 200 árboles. Random Forest, al ser un ensemble bootstrap independiente, promedia esa varianza. Con ~1 100 clases y 86 % desbalanceado, LightGBM ya está en zona de inestabilidad numérica.

### Lectura para la memoria

- **El número que mide la calidad del modelo es el desglose por estado HMM, no el global.** El 81,1 % global está dominado por la facilidad del estacionario (86 %), donde "se queda en la misma celda" funciona como heurística trivial.
- **La métrica de migración con HMM5 es más honesta**: refleja la dificultad real de predecir migraciones largas, no el inflado por commutes.
- El descenso aparente de 47,9 → 26,7 % **no contradice la decisión de adoptar HMM5 como canónico**. HMM5 etiqueta menos pero más limpio; la métrica antigua medía una mezcla, la nueva mide la clase real.
- Para reportar la "mejor versión" del modelo en la memoria conviene re-ejecutar también ML4 y ML5 sobre `hmm5.csv` antes de comparar — sus métricas históricas siguen siendo sobre HMM1 y no son directamente comparables con ML3 actual.
