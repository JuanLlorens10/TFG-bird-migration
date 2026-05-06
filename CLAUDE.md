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
    hmm2.csv                          # Producido por HMM2.ipynb (recomendado); step/bearing/turning/estado_hmm
    hmm3.csv                          # Producido por HMM3.ipynb (HMM2 + veg); idéntico esquema a hmm2.csv
    hmm_wind.csv                      # Extiende hmm.csv con viento ERA5 a 850 hPa → usado por ML6
```

`hmm.csv` y `hmm2.csv` añaden sobre `aves_procesado_markov.csv`: `step_length` (km), `bearing` (grados), `turning_angle` (radianes), `estado_hmm` (0=migración, 1=estacionario). `hmm.csv` añade además `grid_x/grid_y/cell_id/target_cell/next_lat/next_lon` (columnas de Markov/ML). `hmm2.csv` solo añade las 4 columnas HMM, sin las de Markov.

`hmm_wind.csv` añade sobre `hmm.csv`: `u_wind`, `v_wind`, `wind_speed`, `tail_wind`, `cross_wind` (componentes de viento ERA5 a 850 hPa).

---

## Pipeline de notebooks

Deben ejecutarse en orden — cada uno alimenta al siguiente:

1. `dataExploration1.ipynb` → limpieza del GPS bruto, produce `aves_procesado_markov.csv`
2. `HMM.ipynb` → HMM original (referencia); produce `hmm.csv` con features de movimiento + etiqueta HMM + columnas Markov/ML
   - `HMM2.ipynb` → HMM mejorado (versión recomendada); produce `hmm2.csv` solo con step/bearing/turning/estado_hmm
3. `markov1.ipynb` → construye 12 matrices de transición mensuales desde `aves_procesado_markov.csv`
4. `ML3–ML5.ipynb` → entrenan clasificadores sobre `hmm.csv` para predecir `target_cell` (ML1 y ML2 fueron eliminados; sus resultados quedan en la tabla de evolución como contexto)
5. `ML6.ipynb` → igual que ML5 pero sobre `hmm_wind.csv`, añade 5 features de viento ERA5
6. `ML0.ipynb` → línea paralela limpia sobre `hmm2.csv` con corrección de leakage (recalcula step/bearing como salto t-1 → t)

Split **por animal**: primer 80% cronológico → train, último 20% → test. El `LabelEncoder` se ajusta solo sobre celdas de train; las filas de test con celdas no vistas se descartan.

---

## Estado actual — O3: HMM

### Tres implementaciones: HMM1 vs HMM2 vs HMM3

| Aspecto | HMM1 (`HMM.ipynb`) | HMM2 (`HMM2.ipynb`) | HMM3 (`HMM3.ipynb`) |
|---|---|---|---|
| Features | `step_length`, `turning_angle`, `veg_low`, `veg_high` | `step_length`, `cos(turn)` | `step_length`, `cos(turn)`, `veg_low`, `veg_high` |
| Transformación | Ninguna | Ninguna | Ninguna |
| `covariance_type` | `'diag'` | `'diag'` | `'diag'` |
| `lengths=` en `.fit()` | **No** — bug crítico | **Sí** | **Sí** |
| Inicializaciones | 1 (`random_state=42`) | 15 seeds | 15 seeds |
| Asignación 0/1 | Manual post-hoc | Automática por `means_` | Automática por `means_` |
| Validación | Sin análisis de dinámica | Completa | Completa + concordancia vs HMM2 |

### Resultados de HMM2 (v2, valores reales)

| Métrica | Migración (0) | Estacionario (1) |
|---|---|---|
| n días | 4 476 (21,2 %) | 16 605 (78,8 %) |
| Media `step_length` | **129,6 km** | **4,1 km** |
| Mediana `step_length` | 43,6 km | 1,7 km |
| % días con step > 100 km | 27,6 % | 0,0 % |
| % días con step < 10 km | 0,6 % | 85,1 % |
| Persistencia diagonal | 0,706 | **0,917** |
| Duración media (días) | 3,4 | **12,1** |
| % migración en abril | 34,0 % | — |
| % migración en septiembre | 31,9 % | — |
| % migración en enero | 7,1 % | — |
| Estabilidad (15 inits) | todos convergen al mismo LL = −113 068,9 |

HMM1: media migración 119,87 km, media estacionario 3,90 km (similares en emisiones, pero transmat y duración no son confiables por el bug de `lengths`).

### Resultados de HMM3 y comparación con HMM2

| Métrica | HMM2 (sin veg) | HMM3 (con veg) | Δ |
|---|---|---|---|
| Media step migración (km) | 129,57 | 125,84 | −3 |
| Media step estacionario (km) | 4,09 | 4,00 | ≈0 |
| % días migración | 21,2 % | 21,9 % | +0,7 pp |
| Persistencia diag. estacionario | 0,917 | 0,917 | 0 |
| **Concordancia día a día** | — | **98,99 %** | — |
| Días reclasificados HMM2→HMM3 | — | 212 de 21 081 | — |
| Δ % migración en verano (jun-ago) | — | **+1,2 pp** (peor) | — |

**Medias de vegetación por estado interno en HMM3**: mig veg_low 0,16, veg_high 0,33 / est veg_low 0,22, veg_high 0,33 — casi idénticas. La vegetación **no discrimina** entre estados.

**Conclusión del experimento HMM3**: añadir vegetación es prácticamente inocuo (99 % de acuerdo). El motor de separación es `step_length`; las features de vegetación tienen varianza ~0,1 frente a ~10⁴ km² de step. Sin embargo, introducen un sesgo leve que aumenta la etiqueta "migración" en verano (+1,2 pp), cuando las aves están en cría. Eso hace que HMM2 siga siendo la implementación recomendada: resultados equivalentes, sin el sesgo de hábitat.

### Decisiones de diseño clave (HMM2)

1. **`lengths=` por `trayectoria_id`** — imprescindible; sin él el HMM modela ~480 transiciones espurias entre aves distintas.
2. **`step_length` en bruto, sin `log` ni `StandardScaler`** — la distribución tiene una bimodalidad natural (pico 0–5 km / cola 50–200+ km) que es la señal más fuerte del problema. Escalar la diluye (demostrado en la v1 fallida, que se latchó al eje de `cos(turning_angle)` en su lugar).
3. **`cos(turning_angle)` en lugar de `turning_angle` raw** — resuelve la circularidad (−π y +π son el mismo ángulo) sin necesitar von Mises.
4. **`covariance_type='diag'`** — con varianzas tan dispares (step ~10⁴ km² vs cos ~0,5), full es inestable.
5. **Sin `veg_low/veg_high`** — son features de hábitat, no de comportamiento. La profesora pidió "velocidad y rumbo".

### Limitación conocida del modelo de 2 estados

El estado "migración" captura tanto **vuelos de larga distancia** (step > 100 km, 28 % de los días) como **commutes activos intra-residencia** (step 20–100 km, 72 % de los días). Con `n_components=2` (restricción de la profesora) es imposible separar ambos. Para distinguirlos haría falta un tercer estado. Esta limitación explica:
- Mediana en migración de 43 km (no 100+).
- `|turning_angle|` medio en migración de 1,52 rad (~87°), mayor de lo esperado para vuelo recto.
- ~14–21 % de días en "migración" incluso en verano (temporada de cría).
- 31 % de las rachas de migración son de 1 solo día.

---

## Estado actual y resultados (Random Forest)

### Evolución de los modelos ML

| Versión | Cambio principal | Migración Top-1 | Estacionario Top-1 | Global Top-1 |
|---------|-----------------|-----------------|-------------------|--------------|
| ML1 † | Baseline: mes + día semana + fase lunar | — | — | 80.3% |
| ML2 † | Quita fase lunar | — | — | 80.8% |
| ML3 | Solo semana del año | 47.9% | 87.2% | 81.1% |
| ML4 | Modelos separados por estado + lag-1 | 44.5% | 89.2% | 82.3% |
| **ML5** | +lag-2, racha, episodio, dist. latitudinal, delta_bearing | **43.4%** | **89.8%** | **82.9%** |
| ML6 | +viento ERA5 a 850 hPa (u, v, speed, tail, cross) | 41.6% | 88.4% | — |
| ML0 | Baseline limpio sobre `hmm2.csv`, anti-leakage (12 features) | pendiente | pendiente | pendiente |

† Notebook eliminado del repositorio el 2026-05-06 (ML3 los superó sin aportar señal). Los resultados se conservan como contexto histórico de la evolución; recuperables desde el historial de git.

**ML5 sigue siendo la mejor versión de la línea histórica**. ML6 empeora respecto a ML5 a pesar de añadir viento ERA5 — resultado importante: el viento a 850 hPa no captura el viento efectivo que experimenta el ave a baja altitud. El análisis de la evolución ML3→ML6 es parte central del trabajo. ML0 es la nueva línea paralela limpia sobre `hmm2.csv` (pendiente de ejecución completa).

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
- **El viento ERA5 a 850 hPa no mejora (ML6)**: añadir u_wind, v_wind, wind_speed, tail_wind, cross_wind empeora ambos estados (migración: 41.6% vs 43.4% en ML5; estacionario: 88.4% vs 89.8%). El viento a ese nivel de presión no captura bien el viento que experimenta el ave a baja altitud.
- **LightGBM no funciona** con la configuración actual (~8-41% según versión). Con +850 clases y `class_weight='balanced'` en la API sklearn de LightGBM 4.6 necesita tuning específico de hiperparámetros. No usarlo sin esa búsqueda.
- **XGBoost ≈ Random Forest** en resultados pero tarda 3-4× más en entrenar.

### Error geográfico (ML5, Random Forest)

| Estado | Mediana | ≤50 km | ≤100 km |
|--------|---------|--------|---------|
| Migración | 32 km | 62% | 78% |
| Estacionario | 5 km | 98.6% | 98.8% |

### Features de ML6 (23 en total = ML5 + viento ERA5)

```
[todas las de ML5]
Viento ERA5:    u_wind, v_wind, wind_speed, tail_wind, cross_wind
```

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
