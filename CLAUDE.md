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
    hmm.csv                           # Producido por HMM.ipynb (original); extiende el anterior
    hmm2.csv                          # Producido por HMM2.ipynb; referencia simplificada de 2 features
    hmm3.csv                          # Producido por HMM3.ipynb (HMM2 + veg); idéntico esquema a hmm2.csv
    hmm5.csv                          # Producido por HMM5.ipynb (CANÓNICO TFG); HMM2 + veg + horas_luz
    hmm_wind.csv                      # Extiende hmm.csv con viento ERA5 a 850 hPa → usado por ML6
```

`hmm.csv` y `hmm2.csv` añaden sobre `aves_procesado_markov.csv`: `step_length` (km), `bearing` (grados), `turning_angle` (radianes), `estado_hmm` (0=migración, 1=estacionario). `hmm.csv` añade además `grid_x/grid_y/cell_id/target_cell/next_lat/next_lon` (columnas de Markov/ML). `hmm2.csv` solo añade las 4 columnas HMM, sin las de Markov.

`hmm_wind.csv` añade sobre `hmm.csv`: `u_wind`, `v_wind`, `wind_speed`, `tail_wind`, `cross_wind` (componentes de viento ERA5 a 850 hPa).

---

## Pipeline de notebooks

Deben ejecutarse en orden — cada uno alimenta al siguiente:

1. `dataExploration1.ipynb` → limpieza del GPS bruto, produce `aves_procesado_markov.csv`
2. `HMM.ipynb` → HMM original (referencia); produce `hmm.csv` con features de movimiento + etiqueta HMM + columnas Markov/ML
   - `HMM2.ipynb` → HMM con 2 features (referencia simplificada); produce `hmm2.csv`
   - `HMM5.ipynb` → **HMM canónico del TFG** con 4 features recomendadas por la tutora (step + turn + vegetación + horas de luz); produce `hmm5.csv`. Justificación de elegirlo sobre HMM2 en `docs/O3_hmm.md`.
3. `markov1.ipynb` → construye 12 matrices de transición mensuales desde `aves_procesado_markov.csv`
4. `ML3–ML5.ipynb` → entrenan clasificadores sobre `hmm.csv` para predecir `target_cell` (ML1 y ML2 fueron eliminados; sus resultados quedan en la tabla de evolución como contexto)
5. `ML6.ipynb` → igual que ML5 pero sobre `hmm_wind.csv`, añade 5 features de viento ERA5
6. `ML0.ipynb` → línea paralela limpia sobre `hmm2.csv` con corrección de leakage (recalcula step/bearing como salto t-1 → t)

Split **por animal**: primer 80% cronológico → train, último 20% → test. El `LabelEncoder` se ajusta solo sobre celdas de train; las filas de test con celdas no vistas se descartan.

---

## Estado por objetivo

- **O1 — Datos**: ver `docs/O1_datos.md`
- **O2 — Markov**: ver `docs/O2_markov.md`
- **O3 — HMM**: ver `docs/O3_hmm.md` — implementaciones HMM1/2/3/4/5, resultados y decisiones de diseño
- **O4 — ML**: ver `docs/O4_ml.md` — evolución ML0–ML6, features, conclusiones y error geográfico
- **O5 — Visualización**: ver `docs/O5_visualizacion.md`

---

## Registro de conversación (conversation_log.md)

Al final de **cada respuesta relacionada con el proyecto** (análisis de datos, modelos ML, visualización, notebooks, datos, resultados), añadir una entrada en `conversation_log.md`. **No registrar** preguntas sobre el funcionamiento de Claude Code (modos, skills, atajos, configuración, etc.).

```markdown
## [YYYY-MM-DD HH:MM] Prompt
<texto exacto del prompt del usuario>

### Resumen de respuesta
<resumen que incluya: qué archivos se editaron/crearon, qué secciones concretas se modificaron, y con qué propósito>

---
```

El archivo está en `/home/jllorens/Desktop/TFG/version2/conversation_log.md`.

---

## Git workflow

Hacer commit y push tras cada unidad de trabajo significativa (nueva celda de análisis, nuevo notebook, resultado de modelo). Nunca usar `git add .` — añadir ficheros por nombre para evitar subir `tfg_env/` o `data/`.

```bash
git add notebooks/MLx.ipynb img/nombre.png
git commit -m "descripción concisa de qué cambió y por qué"
git push
```

Los ficheros `tfg_env/` y `data/` están en `.gitignore` y nunca deben commitearse.
