# O3 — Detección de comportamiento con HMM

> Referencia específica para el objetivo O3 del TFG.
> Actualizar tras cada sesión de trabajo en este objetivo.
> Para el detalle completo de cada fase (diseño, experimentos, resultados paso a paso) ver `notebooks/HMM2_plan.md`.

## Objetivo

Identificar automáticamente si el ave está en reposo o en migración activa analizando su velocidad (`step_length`) y rumbo (`turning_angle`) mediante un Hidden Markov Model gaussiano.

## Notebooks

| Notebook | Output | Descripción |
|---|---|---|
| `HMM.ipynb` | `hmm.csv` | HMM1: implementación original (referencia, tiene bug de `lengths`) |
| `HMM2.ipynb` | `hmm2.csv` | HMM2: versión recomendada (sin bug, 15 seeds, `cos(turn)`) |
| `HMM3.ipynb` | `hmm3.csv` | HMM3: experimento — HMM2 + vegetación |
| `HMM4.ipynb` | `hmm4.csv` | HMM4: experimento — HMM2 con 3 estados |

## Cuatro implementaciones comparadas

| Aspecto | HMM1 (`HMM.ipynb`) | HMM2 (`HMM2.ipynb`) | HMM3 (`HMM3.ipynb`) | HMM4 (`HMM4.ipynb`) |
|---|---|---|---|---|
| Features | `step_length`, `turning_angle`, `veg_low`, `veg_high` | `step_length`, `cos(turn)` | `step_length`, `cos(turn)`, `veg_low`, `veg_high` | `step_length`, `cos(turn)` |
| `n_components` | 2 | 2 | 2 | **3** |
| Transformación | Ninguna | Ninguna | Ninguna | Ninguna |
| `covariance_type` | `'diag'` | `'diag'` | `'diag'` | `'diag'` |
| `lengths=` en `.fit()` | **No** — bug crítico | **Sí** | **Sí** | **Sí** |
| Inicializaciones | 1 (`random_state=42`) | 15 seeds | 15 seeds | 15 seeds |
| Asignación etiquetas | Manual post-hoc | Auto por `means_` (0/1) | Auto por `means_` (0/1) | Auto por `means_` (0/1/2) |
| Validación | Sin análisis de dinámica | Completa | Completa + concordancia vs HMM2 | Completa + contingencia vs HMM2 |
| Convención | 0=mig, 1=est | 0=mig, 1=est | 0=mig, 1=est | 0=mig real, 1=est, 2=commute |

**HMM2 es la versión recomendada para la defensa** (respeta `n_components=2`). HMM3 y HMM4 son experimentos exploratorios con un único cambio respecto a HMM2.

---

## Resultados de HMM2 (versión recomendada)

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

---

## Experimento HMM3: ¿añadir vegetación mejora?

| Métrica | HMM2 (sin veg) | HMM3 (con veg) | Δ |
|---|---|---|---|
| Media step migración (km) | 129,57 | 125,84 | −3 |
| Media step estacionario (km) | 4,09 | 4,00 | ≈0 |
| % días migración | 21,2 % | 21,9 % | +0,7 pp |
| Persistencia diag. estacionario | 0,917 | 0,917 | 0 |
| **Concordancia día a día** | — | **98,99 %** | — |
| Días reclasificados HMM2→HMM3 | — | 212 de 21 081 | — |
| Δ % migración en verano (jun-ago) | — | **+1,2 pp** (peor) | — |

**Medias de vegetación por estado en HMM3**: mig veg_low 0,16, veg_high 0,33 / est veg_low 0,22, veg_high 0,33 — casi idénticas. La vegetación **no discrimina** entre estados. Motor de separación = `step_length`.

**Conclusión HMM3**: inocuo (99 % de acuerdo) pero introduce sesgo leve de +1,2 pp en verano. HMM2 sigue siendo la versión recomendada.

---

## Experimento HMM4: ¿un tercer estado resuelve las limitaciones?

| Estado HMM4 | n días | Media step (km) | `\|turn\|` (rad) | Duración media (días) |
|---|---:|---:|---:|---:|
| Migración real (0) | 2 559 (12,1 %) | **205,0** | **1,131** (~65°) | 3,5 |
| Estacionario (1) | 6 095 (28,9 %) | 0,13 | 1,782 | 2,8 |
| Commute activo (2) | 12 427 (58,9 %) | 9,87 | 2,044 | 4,9 |

Las 15 seeds convergen al mismo LL (`-98.614,3`, dispersión 0,0).

**Tabla de contingencia HMM4 × HMM2**:

| | est HMM2 | mig HMM2 |
|---|---:|---:|
| migración real (0) HMM4 | 28 | **2 531 (98,9 %)** |
| estacionario (1) HMM4 | **6 086 (99,9 %)** | 9 |
| commute activo (2) HMM4 | **10 491 (84,4 %)** | 1 936 (15,6 %) |

**Verificación de los 4 síntomas de HMM2**:

| Síntoma | HMM2 | HMM4 | ¿Mejora? |
|---|---|---|---|
| `\|turning_angle\|` en migración | 1,52 rad (~87°) | **1,13 rad (~65°)** | ✅ |
| % migración en verano (jun-ago) | 17,4 % | **6,3 %** | ✅ |
| Step medio de migración | 129,6 km | **205,0 km** | ✅ |
| Mediana run-length estacionario | 4,0 días | 1,0 día | ❌ |

**Conclusión HMM4**: 3 de 4 síntomas mejoran. Hallazgo inesperado: el commute cae al 84,4 % en `est(HMM2)`, no en `mig(HMM2)`. La frontera principal que añade el tercer estado está entre quietud casi total (≪ 1 km) y movimiento leve (~10 km). Detalles completos en `notebooks/HMM2_plan.md` (Fase 7).

---

## Decisiones de diseño clave (HMM2)

1. **`lengths=` por `trayectoria_id`** — imprescindible; sin él el HMM modela ~480 transiciones espurias entre aves distintas.
2. **`step_length` en bruto, sin `log` ni `StandardScaler`** — la distribución bimodal (pico 0–5 km / cola 50–200+ km) es la señal más fuerte. Escalar la diluye (demostrado en la v1 fallida).
3. **`cos(turning_angle)` en lugar de `turning_angle` raw** — resuelve la circularidad (−π y +π son el mismo ángulo) sin necesitar von Mises.
4. **`covariance_type='diag'`** — con varianzas tan dispares (step ~10⁴ km² vs cos ~0,5), full es inestable.
5. **Sin `veg_low/veg_high`** — son features de hábitat, no de comportamiento. La profesora pidió "velocidad y rumbo".

## Limitación conocida del modelo de 2 estados

El estado "migración" mezcla vuelos largos (step > 100 km, 28 %) con commutes intra-residencia (step 20–100 km, 72 %). Con `n_components=2` es imposible separar ambos. Consecuencias:
- Mediana de migración = 43 km (no 100+).
- `|turning_angle|` medio en migración = 1,52 rad (~87°).
- 14–21 % de días en "migración" incluso en verano (cría).
- 31 % de las rachas de migración duran 1 solo día.
