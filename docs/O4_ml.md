# O4 — Predicción con Machine Learning

> Referencia específica para el objetivo O4 del TFG.
> Actualizar tras cada sesión de trabajo en este objetivo.

## Objetivo

Entrenar y comparar modelos de clasificación (RF, XGBoost, LightGBM) para predecir la `target_cell` (celda que ocupará el ave al día siguiente) usando fecha, inercia del movimiento y estado HMM.

## Notebooks

Línea principal (sobre `hmm.csv`, todavía con leakage en `step_length`/`bearing` por venir como salto t→t+1):

`ML3.ipynb` → `ML4.ipynb` → `ML5.ipynb` → `ML6.ipynb`

Todos leen `data/processed/hmm.csv` (ML3–ML5) o `data/processed/hmm_wind.csv` (ML6).

Línea paralela limpia:

`ML0.ipynb` — baseline rediseñado desde cero sobre `hmm2.csv`, recalculando las features de movimiento como salto t-1 → t para eliminar el leakage. Pensado como referencia limpia para construir variantes posteriores (ML0b, ML0c…) midiendo qué aporta cada decisión adicional.

> Nota: `ML1.ipynb` y `ML2.ipynb` se eliminaron del repositorio (commit de limpieza del 2026-05-06). Sus aportaciones — fase lunar y día de la semana como features temporales — quedaron descartadas en ML3 sin aportar señal. Las filas ML1/ML2 de la tabla siguiente se conservan como contexto histórico de la evolución; los notebooks ya no existen en disco. Si se necesita recuperarlos, están en el historial de git.

## Split por animal

Primer 80 % cronológico → train; último 20 % → test. El `LabelEncoder` se ajusta solo sobre celdas de train; las filas de test con celdas no vistas se descartan.

---

## Evolución de los modelos

| Versión | Cambio principal | Migración Top-1 | Estacionario Top-1 | Global Top-1 |
|---------|-----------------|-----------------|-------------------|--------------|
| ML1 † | Baseline: mes + día semana + fase lunar | — | — | 80,3 % |
| ML2 † | Quita fase lunar | — | — | 80,8 % |
| ML3 | Solo semana del año | 47,9 % | 87,2 % | 81,1 % |
| ML4 | Modelos separados por estado + lag-1 | 44,5 % | 89,2 % | 82,3 % |
| **ML5** | +lag-2, racha, episodio, dist. latitudinal, delta_bearing | **43,4 %** | **89,8 %** | **82,9 %** |
| ML6 | +viento ERA5 a 850 hPa (u, v, speed, tail, cross) | 41,6 % | 88,4 % | — |
| ML0 | Línea paralela limpia sobre `hmm2.csv`, anti-leakage (12 features) | pendiente | pendiente | pendiente |

† Notebook eliminado del repositorio el 2026-05-06; el resultado se conserva como contexto histórico (recuperable desde el historial de git).

**ML5 es la mejor versión de la línea histórica.** ML6 empeora a pesar de añadir viento ERA5. ML0 está pendiente de ejecución completa (el `RandomizedSearchCV` se reajustó para evitar OOM).

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
Comportamiento: estado_hmm   (con caveat: estado_hmm en hmm2.csv se obtuvo a partir de las features con leakage)
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
