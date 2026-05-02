# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Contexto del proyecto

Trabajo Fin de Grado del **Grado en Matemáticas e Informática** de la **Universidad Politécnica de Madrid (UPM)**. El objetivo es construir un sistema completo de predicción de la posición diaria de gaviotas (*Larus fuscus*) combinando tres enfoques complementarios:

1. **Cadenas de Markov** — probabilidad de transición entre celdas según la estación del año
2. **Hidden Markov Models (HMM)** — detección automática de estados de comportamiento (reposo vs. migración activa) a partir de desplazamiento y rumbo
3. **Machine Learning (RF, XGBoost, LightGBM)** — predicción del siguiente salto diario usando fecha, inercia del movimiento y estado HMM

El trabajo incluye también un **módulo de visualización** con mapas interactivos que comparan la ruta real del ave con la predicción.

### Objetivos oficiales del TFG

| ID | Objetivo |
|----|----------|
| O1 | Preparación de datos: limpieza de +80k registros GPS, selección de secuencias diarias sin interrupciones |
| O2 | Predicción estadística con Markov: tablas de probabilidad de movimiento entre celdas por estación |
| O3 | Detección de comportamiento (HMM): identificar reposo vs. viaje analizando velocidad y rumbo |
| O4 | Predicción con ML: entrenar y comparar RF, XGBoost y LightGBM para predecir el siguiente salto |
| O5 | Módulo de visualización en mapas interactivos con análisis del error en km |

Un eje central del trabajo es **analizar qué cambios mejoran o empeoran los modelos** y justificarlo.

---

## Environment

Todo el trabajo corre dentro del entorno virtual `tfg_env/`. Usar siempre sus binarios:

```bash
# Lanzar JupyterLab
tfg_env/bin/jupyter lab

# Ejecutar un notebook completo sin interacción (re-ejecuta todas las celdas)
tfg_env/bin/jupyter nbconvert --to notebook --execute --inplace notebooks/<nombre>.ipynb

# Ejecutar un script Python puntual
tfg_env/bin/python script.py
```

Paquetes clave: pandas 3.x, numpy, scikit-learn, xgboost, lightgbm, hmmlearn, matplotlib, seaborn, plotly, scipy (Python 3.12).

**Nota pandas**: usar `.map()` en lugar de `.applymap()` (renombrado en pandas 3.x).

---

## Data layout

```
data/
  raw/migration_original.csv          # GPS bruto: 126 gaviotas, 2009-2015, ~90k registros
  processed/
    aves_procesado_markov.csv         # Una localización/día/ave (~22k filas) → usado por Markov
    hmm.csv                           # Extiende el anterior con features de movimiento y etiqueta HMM
```

`hmm.csv` añade sobre `aves_procesado_markov.csv`: `step_length`, `bearing`, `turning_angle`, `estado_hmm` (0=migración, 1=estacionario), `grid_x/grid_y/cell_id` (discretización 0,5°×0,5°), `target_cell` (celda objetivo del día siguiente), `next_lat/next_lon`.

---

## Pipeline de notebooks

Deben ejecutarse en orden — cada uno alimenta al siguiente:

1. `dataExploration1.ipynb` → limpieza del GPS bruto, produce `aves_procesado_markov.csv`
2. `HMM.ipynb` → ajusta HMM gaussiano, añade features de movimiento y etiquetas de estado, produce `hmm.csv`
3. `markov1.ipynb` → construye 12 matrices de transición mensuales desde `aves_procesado_markov.csv`
4. `ML1–ML5.ipynb` → entrenan clasificadores sobre `hmm.csv` para predecir `target_cell`

Split **por animal**: primer 80% cronológico → train, último 20% → test. El `LabelEncoder` se ajusta solo sobre celdas de train; las filas de test con celdas no vistas se descartan.

---

## Estado actual y resultados (Random Forest)

### Evolución de los modelos ML

| Versión | Cambio principal | Migración Top-1 | Estacionario Top-1 | Global Top-1 |
|---------|-----------------|-----------------|-------------------|--------------|
| ML1 | Baseline: mes + día semana + fase lunar | — | — | 80.3% |
| ML2 | Quita fase lunar | — | — | 80.8% |
| ML3 | Solo semana del año | 47.9% | 87.2% | 81.1% |
| ML4 | Modelos separados por estado + lag-1 | 44.5% | 89.2% | 82.3% |
| **ML5** | +lag-2, racha, episodio, dist. latitudinal, delta_bearing | **43.4%** | **89.8%** | **82.9%** |

**ML5 es la mejor versión** en todos los métricas. El análisis de la evolución ML3→ML4→ML5 es parte central del trabajo (objetivo de comparar qué mejora y qué no).

### Features de ML5 (18 en total)

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

### Conclusiones clave

- **Brecha migración/estacionario**: el modelo predice bien dónde estará un ave en reposo (~90%) pero mal adónde irá en migración (~44%). La causa es que los saltos migratorios dependen del viento, que no está en los datos.
- **Techo de migración sin viento**: ~44-48% Top-1. Para superarlo habría que integrar datos ERA5 (Copernicus).
- **LightGBM no funciona** con la configuración actual (~8-44% según versión). Con +850 clases y `class_weight='balanced'` en la API sklearn de LightGBM 4.6 necesita tuning específico de hiperparámetros. No usarlo sin esa búsqueda.
- **XGBoost ≈ Random Forest** en resultados (~81.8% vs 82.9%) pero tarda 3-4× más en entrenar.

### Error geográfico (ML5, Random Forest)

| Estado | Mediana | ≤50 km | ≤100 km |
|--------|---------|--------|---------|
| Migración | 32 km | 62% | 78% |
| Estacionario | 5 km | 98.6% | 98.8% |

---

## Git workflow

Hacer commit y push tras cada unidad de trabajo significativa (nueva celda de análisis, nuevo notebook, resultado de modelo). Nunca usar `git add .` — añadir ficheros por nombre para evitar subir `tfg_env/` o `data/`.

```bash
git add notebooks/MLx.ipynb img/nombre.png
git commit -m "descripción concisa de qué cambió y por qué"
git push
```

Los ficheros `tfg_env/` y `data/` están en `.gitignore` y nunca deben commitearse.
