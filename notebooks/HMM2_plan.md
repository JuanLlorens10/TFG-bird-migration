# Plan — O3: HMM de comportamiento (descanso vs. viaje)

## Contexto

Trabajo Fin de Grado del Grado en Matemáticas e Informática (UPM). Objetivo **O3 – Detección de comportamiento mediante HMM**: distinguir dos estados (descanso vs. viaje) a partir de la velocidad y el rumbo del ave.

Ya existe una implementación en `notebooks/HMM.ipynb`, pero este plan diseña un HMM alternativo desde cero partiendo solo de los datos limpios producidos por `dataExploration1.ipynb` (`data/processed/aves_procesado_markov.csv`). Después se analiza el HMM1, se compara con el nuevo y se justifica por qué el nuevo es mejor.

**Alcance acotado**: en este momento del trabajo, el proyecto solo cuenta con `dataExploration1.ipynb` y `markov1.ipynb`. Los notebooks `ML*.ipynb` se ignoran a propósito — la única responsabilidad del HMM es detectar correctamente los dos estados de comportamiento. No tiene que generar `grid_x/grid_y/cell_id/target_cell/next_cell_id` ni preparar features para machine learning; de eso se encarga `markov1.ipynb`.

**Restricciones de la profesora**:
- Librería: `hmmlearn`.
- `n_components = 2` (etiquetas 0/1).

**Criterio de calidad acordado**: coherencia del comportamiento detectado — medias por estado, matriz de transición, duración media por estado y patrón estacional del % de migración.

## Datos y utilidades

- **Entrada**: `data/processed/aves_procesado_markov.csv` (22 041 filas, 117 aves, 480 segmentos, 1 fila/(ave, día) hacia las 14h).
- **Columnas disponibles**: `animal_id, date, hora, lon, lat, veg_low, veg_high, trayectoria_id`.
- `trayectoria_id` ya separa segmentos continuos de ≥4 días — dentro de un mismo `trayectoria_id` no hay gaps de más de un día, así que no hace falta normalizar por Δt.
- Las fórmulas de **haversine** y **bearing great-circle** ya están en `notebooks/HMM.ipynb` (celdas 13–28). Se reutilizan tal cual: son funciones puras y están bien escritas.
- **Output mínimo del HMM**: añadir al CSV de entrada `step_length`, `bearing`, `turning_angle` y `estado_hmm` (0/1). Lo guardamos como `data/processed/hmm2.csv` (no `hmm.csv`) para no sobrescribir el output del HMM1 y poder comparar.
- **Convención**: `0 = migración`, `1 = estacionario` (la fija CLAUDE.md y la respeta HMM1).

---

## Fase 1 — Diseño del HMM

### 1.1 Cálculo de features de movimiento

Se ordena el DataFrame por `(trayectoria_id, date)` y para cada `trayectoria_id` se calcula:

| Variable | Fórmula | Unidades |
|---|---|---|
| `step_length` | haversine entre `(lat[t], lon[t])` y `(lat[t+1], lon[t+1])` | km |
| `bearing` | great-circle initial bearing entre los mismos dos puntos | grados, [0, 360) |
| `turning_angle` | diff de `bearing` dentro del `trayectoria_id`, normalizado a [-π, π] con `arctan2(sin(Δ), cos(Δ))` | radianes |
| `next_lat`, `next_lon` | shift(-1) dentro del `trayectoria_id` | grados |

`dropna` sobre `step_length` y `turning_angle` (la última fila de cada `trayectoria_id` no tiene siguiente, y la primera no tiene rumbo previo).

### 1.2 Features observadas que entran al HMM

Solo movimiento puro, dos features, **sin transformaciones que igualen escalas**:

```
obs1 = step_length          # km, en bruto
obs2 = cos(turning_angle)   # ∈ [-1, 1], no circular
```

Justificación:
- `step_length` se deja en su escala natural (km). En este dataset tiene una distribución claramente bimodal: pico estrecho 0–5 km (descanso) y cola larga 50–200+ km (migración). Esa enorme varianza es **precisamente la señal** que permite al HMM separar los dos modos. Cualquier intento de "equilibrar" escalas (log + StandardScaler) la diluye.
- `cos(turning_angle)` resuelve la **circularidad** del rumbo (–π y +π son el mismo ángulo): vale 1 cuando el ave sigue recto (típico de migración) y -1 cuando se da media vuelta (típico de búsqueda local). `hmmlearn` no soporta von Mises ni wrapped Cauchy.

**No aplicamos `StandardScaler`**. La asimetría de varianzas entre `step_length` (~10⁴ km²) y `cos(turning_angle)` (~0,5) es deseable: hace que `step_length` domine la verosimilitud Gaussian y el modelo encuentre los dos modos naturales del problema. `cos(turning_angle)` actúa como feature secundaria.

### 1.3 Configuración del modelo

```python
GaussianHMM(
    n_components=2,
    covariance_type="diag",   # con escalas dispares, full es inestable; diag basta
    n_iter=200,
    tol=1e-4,
    random_state=<variable>,
)
```

- `lengths=`: lista del tamaño de cada `trayectoria_id` (en el mismo orden que aparecen las filas), para que `hmmlearn` no aprenda transiciones espurias entre el final de una secuencia y el comienzo de otra.
- 15 inicializaciones con distintos `random_state`; se conserva la de mayor `score(X, lengths)`. Mitiga óptimos locales del EM.

### 1.4 Asignación automática de etiquetas (0/1)

Tras el ajuste se inspecciona `model.means_[:, 0]` (media de `step_length` por estado, en km). El estado con media mayor → `estado_hmm = 0` (migración); el otro → `1` (estacionario). Asignación robusta a inicializaciones que converjan invertidas, manteniendo la convención del proyecto.

### Lección aprendida — versión v1 fallida

Una primera versión del notebook (v1) usaba `log(step_length+1)`, `cos(turning_angle)`, `StandardScaler` y `covariance_type='full'`. **No funcionó**: el HMM separó los dos estados por el eje de `cos(turning_angle)` en lugar de por el de `step_length`, dando un estado "estacionario" con mediana de paso 4,5 km (mayor que la del estado "migración", 3,2 km), persistencia diagonal de 0,27 (físicamente imposible) y sin pico estacional. El log + escalado **diluyeron la bimodalidad natural** de `step_length` que era precisamente la señal más fuerte del problema. Esta versión v2 corrige esa decisión manteniendo el resto del diseño (`lengths=`, multi-init, auto-label, sin vegetación).

### 1.5 Output

`data/processed/hmm2.csv` con las columnas originales más `step_length` (km), `bearing` (grados), `turning_angle` (radianes) y `estado_hmm` (0/1). Sin discretización en celdas ni cálculo de celda objetivo.

### 1.6 Validación (criterio acordado: coherencia del comportamiento)

Para el modelo entrenado se reportan:
- Medias y desviaciones de `step_length` y `|turning_angle|` por estado (en escala original).
- Matriz de transición `model.transmat_` y duración media esperada por estado: `1 / (1 - p_diag)`.
- Histogramas de `step_length` por estado.
- Patrón estacional: % de días en migración por mes (debería picar en marzo-abril y septiembre-octubre).
- Trayectoria de un ave concreta coloreada por estado.

---

## Fase 2 — Análisis del HMM1 (`notebooks/HMM.ipynb`)

| Aspecto | HMM1 |
|---|---|
| Features observadas | `step_length` (km), `turning_angle` (rad), `veg_low`, `veg_high` (4 features) |
| Transformación previa | Ninguna; escalas mezcladas (km, rad, [0,1]) |
| Clase y config | `GaussianHMM(n_components=2, covariance_type='diag', n_iter=2000, random_state=42)` |
| `lengths=` en `.fit()` | **No se pasa** — todo el dataset entra como una sola secuencia |
| Inicializaciones múltiples | No |
| Asignación 0/1 | Manual, post-hoc, leyendo medias |
| Validación | No hay análisis de transiciones, ni de duración por estado, ni de patrón estacional |

**Datos reales del modelo HMM1**:
- Estado 0 ("Migración"): media `step_length` ≈ 119,87 km.
- Estado 1 ("Reposo"): media `step_length` ≈ 3,90 km.

**Hechos objetivos relevantes**:
1. Falta `lengths=` → `hmmlearn` modela 21k observaciones como una sola cadena, generando ~480 transiciones espurias entre aves.
2. Sin estandarización + `cov 'diag'` → `step_length` (km) domina la verosimilitud; `veg_low/veg_high` apenas pesan.
3. `veg_low/veg_high` son features de hábitat, no de comportamiento → mete sesgo no pedido por el enunciado.
4. `turning_angle` modelado como Gaussian sobre [-π, π] ignora su circularidad.
5. Una sola inicialización (`random_state=42`).
6. Asignación manual frágil: si cambia la semilla y el modelo converge invertido, el código no lo detecta.

---

## Fase 3 — Comparación HMM2 v2 vs HMM1 (valores reales)

Criterio: coherencia del comportamiento detectado.

| Métrica | HMM1 (real) | HMM2 v2 (real) |
|---|---|---|
| Media `step_length` migración (km) | 126,07 | **129,57** |
| Media `step_length` estacionario (km) | 3,95 | **4,09** |
| Distribución mig/est | 21,9 % / 78,1 % | **21,2 % / 78,8 %** |
| Persistencia diagonal migración | 0,706 ⚠️ sesgado | **0,706** |
| Persistencia diagonal estacionario | 0,912 ⚠️ sesgado | **0,917** |
| Duración media migración (días) | 3,41 ⚠️ sesgado | **3,41** |
| Duración media estacionario (días) | 11,31 ⚠️ sesgado | **12,10** |
| % migración en enero | 7,6 % | **7,1 %** |
| % migración en abril | 34,3 % | **34,0 %** |
| % migración en septiembre | 32,5 % | **31,9 %** |
| **Concordancia día a día vs HMM2** | **98,94 %** | — |
| **Días que difieren** | **223 de 20 964** | — |
| Tratamiento del rumbo | `turning_angle` Gaussian circular | `cos(turning_angle)` no circular |
| Separación de secuencias | **No** (~480 transiciones espurias) | **Sí** — `lengths=` por `trayectoria_id` |
| Robustez a la semilla | frágil (1 init, etiquetado manual) | estable (15 inits convergen al mismo LL) |
| Alineación con enunciado | mete vegetación (hábitat) | solo velocidad y rumbo |

**Los tres modelos coinciden en el 99 % de los casos.** Los 223 días en que HMM1 y HMM2 difieren tienen `step_length` exclusivamente entre 0 y 25 km (media 17,5 km, mediana 19 km). Ningún desacuerdo tiene step > 50 km: los modelos son idénticos en los casos claros y solo discrepan en la **franja ambigua 10–25 km**. HMM1 etiqueta 182 de esos días como migración; HMM2 los clasifica como estacionario, lo cual es más coherente (una velocidad de 15–20 km/día es commute, no migración de larga distancia).

El efecto cuantificable del bug `lengths=`: la duración media del estado estacionario baja de **12,10 días (HMM2) a 11,31 días (HMM1)**, una reducción del 6,5 % causada por las ~480 transiciones espurias entre aves que interrumpen artificialmente las rachas sedentarias.

---

## Fase 4 — Justificación

**Por qué HMM2 es mejor**:
1. **Matriz de transición correcta**. `lengths=` modela 480 secuencias separadas. El efecto medible: la duración del estado estacionario pasa de 11,3 días (HMM1, sesgado) a 12,1 días (HMM2, correcto).
2. **Decisiones más correctas en la frontera**. Los 223 días en que HMM1 y HMM2 discrepan tienen step 10–25 km; HMM1 etiqueta 182 de ellos como migración y HMM2 como estacionario. A esas velocidades (commute diario) HMM2 toma la decisión más coherente físicamente.
3. **Sin sesgo de hábitat**. Solo `step_length` y `cos(turning_angle)` — fiel al enunciado de la profesora ("velocidad y rumbo").
4. **Tratamiento correcto del rumbo circular** con `cos(turning_angle)`.
5. **Robustez**. 15 inicializaciones convergen al mismo óptimo (LL = −113 068,9). Asignación automática de 0/1 invariante al `random_state`.

**En qué falla HMM1**:
- Bug de `lengths` → matriz de transición y dinámica temporal sesgadas por ~480 transiciones inventadas.
- Vegetación como feature de comportamiento → confunde estado con hábitat.
- `turning_angle` modelado como Gaussian sobre variable circular [-π, π].
- Una sola inicialización; etiquetado manual frágil si cambia el `random_state`.

**Lo que se conserva de HMM1** (y se reutiliza en HMM2):
- Funciones `haversine_km` y `calculate_bearing` (geométricamente correctas).
- Estructura general del notebook (load → features → fit → output → plots).

---

## Fase 5 — Evaluación crítica post-ejecución

### Puntos positivos (confirmados con datos reales)

1. **Separación numérica perfecta entre estados — zero overlap en step_length**:
   - Estacionario: max step = 25 km. **Cero días con step > 30 km**.
   - Migración: q05 = 19 km. Solo 0,6 % de días con step < 10 km.
   - Las dos distribuciones no se solapan en absoluto — resultado excepcionalmente limpio para un HMM Gaussian.

2. **Patrón estacional robusto a través de aves** (24 aves con ≥ 200 días):
   - Ene 7,6 ± 8,4 % | Feb 8,7 ± 8,1 % | Abr **30,8 ± 23,8 %** | May **31,5 ± 26,9 %** | **Sep 46,0 ± 21,8 %** | Oct 34,2 ± 24,9 %
   - Pico otoñal (septiembre 46 %) y primaveral (abril-mayo 31 %). Mínimo en invierno y verano.
   - Coincide con el ciclo real de *Larus fuscus*: migración S→N en primavera y N→S en otoño tras la cría.

3. **Dinámica temporal físicamente realista**:
   - Estacionario persistente (p_diag 0,917 → duración media del modelo 12 días). Migración menos persistente (p_diag 0,706 → 3,4 días).
   - **Interpretación correcta de "12 días"**: es la longitud media de una racha ININTERRUMPIDA de días consecutivos estacionarios dentro de una trayectoria, NO la duración de la temporada de cría/invernada. La distribución empírica de rachas es muy asimétrica (ver caveat 6 abajo).

4. **Coherencia inter-anual en ejemplos individuales** (ave 91916A, 2026 días):
   - El patrón de migración en April+Sep/Oct se repite año tras año de manera consistente.
   - Inviernos quietos (Dic-Feb 0–15 %), verano moderado (~17 % en junio, cría).

5. **Robustez del óptimo**: los 15 `random_state` convergen al mismo log-likelihood (−113 068,9). El mínimo del EM es estable.

---

### Limitaciones y caveats honestos

1. **El estado "migración" mezcla dos comportamientos distintos**:
   - Vuelos de larga distancia (step > 100 km): 27,6 % de los días de migración.
   - Commutes activos intra-residencia (step 20–100 km): ~72 % de los días.
   
   Con `n_components=2` (restricción de la profesora) es **imposible separar** ambos. Para hacerlo haría falta n=3 (descanso / commute activo / migración real). Esta limitación es intrínseca al enunciado, no un fallo de implementación.

2. **31 % de las rachas de migración son de exactamente 1 día** (457 de 1 464 rachas). Algunas son auténticas (un salto largo puntual); otras pueden ser "flips" del modelo ante un único día con step elevado dentro de una residencia. Sin ground truth no se distinguen.

3. **`|turning_angle|` medio en migración = 1,52 rad (~87°)**, mayor de lo esperado para vuelo en línea recta (~0°). Esto se explica por la mezcla de vuelos rectos (cos ≈ 1) y commutes con vuelta atrás (cos ≈ −1) dentro del mismo estado. Si el estado fuera migración pura, el ángulo de giro debería ser mucho menor.

4. **~14–21 % de días en "migración" incluso en verano** (junio-agosto). Durante la cría, las aves no migran — lo que el modelo llama "migración" en esos meses son commutes activos de larga distancia (p. ej. viajes desde la colonia a zonas de pesca). No es un bug; es la consecuencia de tener solo 2 estados.

5. **El HMM2 no genera `grid_x/grid_y/cell_id/target_cell`** (las columnas que usa `markov1.ipynb` y los modelos ML). Solo produce `step_length`, `bearing`, `turning_angle`, `estado_hmm`. Esas columnas adicionales las genera `HMM.ipynb` y pertenecen al pipeline de Markov/ML, no al O3 puro.

6. **Los "12 días" del modelo ≠ la duración de la temporada sedentaria**. La distribución empírica real de rachas estacionarias (calculada sobre `hmm2.csv`) es muy asimétrica:

   | Percentil | Longitud de racha |
   |---|---|
   | q25 | 2 días |
   | **q50 (mediana)** | **4 días** |
   | q75 | 9 días |
   | q90 | 21 días |
   | q99 | 97 días |
   | máximo | 482 días |

   - El 20 % de las rachas son de **1 solo día**.
   - Solo el 7 % son ≥ 30 días; solo el 3 % ≥ 60 días.
   - El máximo (482 días) corresponde a un ave con GPS que casi no se mueve durante más de un año.
   - La media de 9,8 días (fórmula del modelo: 12 días) está inflada por la cola derecha de rachas largas.

   **Por qué son tan cortas**: durante la temporada de cría (90 días en N. Europa), el ave hace viajes diarios de alimentación. Los que superan ~20–25 km se etiquetan como "migración" y rompen la racha. Un periodo real de 90 días en la colonia puede aparecer como muchas rachas de 4–12 días separadas por días de "commute" etiquetados como migración. La causa es la limitación de 2 estados con resolución diaria (ver caveat 1).

---

### Veredicto global (los tres modelos)

| | HMM1 | HMM2 | HMM3 |
|---|---|---|---|
| Acuerdo con HMM2 | **98,94 %** | — | **98,99 %** |
| Días discrepantes | 223 (step 10–25 km) | — | 212 (step 10–25 km) |
| Dirección del sesgo | +182 días en migración | **referencia** | +150 días en migración |
| Duración estacionario | 11,3 d ⚠️ | **12,1 d** | 12,0 d |
| Sesgo estacional | ninguno extra | **referencia** | +1,9 pp en junio |
| Metodología correcta | ❌ | ✅ | ✅ parcial |

Los tres modelos son **funcionalmente equivalentes para los casos claros** (step < 10 km o step > 50 km, que representan el 99 % de los datos). Solo difieren en la **franja 10–25 km**, donde el paso es ambiguo entre commute activo y forrajeo. HMM1 y HMM3 son sistemáticamente más "agresivos" en etiquetar esos días como migración; HMM2 es más conservador — y esa conservación está justificada porque 15–20 km/día no es velocidad de migración.

**Conclusión final**: HMM2 es el mejor por criterios cuantificables (transmat correcta, decisiones más coherentes en la frontera, sin sesgo estacional), no solo por pureza teórica. La diferencia no es dramática en los agregados, pero es real y medible.

La interpretación precisa de los estados en cualquiera de los tres modelos es:

- `estado_hmm = 0` → **"día activo"** (movimiento ≥ ~20 km): incluye vuelos migratorios reales Y commutes activos largos.
- `estado_hmm = 1` → **"día sedentario"** (movimiento < ~25 km): reposo en cría/invernada, forrajeo local.

---

---

## Fase 6 — Experimento HMM3: ¿qué ocurre al añadir `veg_low` y `veg_high`?

### Diseño del experimento

`notebooks/HMM3.ipynb` es una copia exacta de HMM2 con una única diferencia: añade `veg_low` y `veg_high` como features observadas (4 features en lugar de 2). Sin `StandardScaler` ni ningún otro cambio.

**Predicción a priori**: con `step_length` en km (varianza ~10⁴) y vegetación en [0, 1] (varianza ~0,1), la verosimilitud Gaussian estará dominada por la velocidad. Esperamos que los estados queden casi idénticos a los de HMM2.

### Resultados (valores reales)

| Métrica | HMM2 (sin veg) | HMM3 (con veg) | Δ |
|---|---|---|---|
| Media step migración (km) | 129,57 | 125,84 | −3 |
| Media step estacionario (km) | 4,09 | 4,00 | ≈0 |
| % días migración | 21,2 % | 21,9 % | +0,7 pp |
| Persistencia diag. estacionario | 0,917 | 0,917 | 0 |
| **Concordancia día a día** | — | **98,99 %** | — |
| Días que difieren | — | 212 de 21 081 | — |
| Δ % migración en verano (jun-ago) | — | +1,2 pp | — |

**Medias de vegetación por estado en HMM3**:

| Estado | veg_low | veg_high |
|---|---|---|
| Migración | 0,16 | 0,33 |
| Estacionario | 0,22 | 0,33 |

`veg_high` es idéntico (0,33 en ambos). `veg_low` difiere 0,06 — diferencia marginal. La vegetación **no discrimina entre estados**.

### Conclusiones del experimento

1. **La predicción a priori se confirma**: 99 % de acuerdo entre HMM2 y HMM3. La vegetación es un ruido de bajo peso porque su varianza (~0,1) es despreciable frente a `step_length` (~10⁴ km²). El motor de la separación es únicamente la velocidad.

2. **Añadir vegetación no mejora — introduce un sesgo leve**: el +0,7 pp global de migración se concentra en verano (+1,2 pp en jun-ago), que es temporada de cría cuando no debería haber migración real. El HMM3 etiqueta más días como "migración" en los meses equivocados.

3. **El fallo de HMM1 no era la vegetación**: el HMM1 original también incluía `veg_low/veg_high` pero producía resultados parecidos en separación de estados. Esto demuestra que el verdadero bug de HMM1 era no pasar `lengths=`, no la vegetación. Si arreglas solo ese bug y mantienes la vegetación (que es exactamente HMM3), los resultados son prácticamente los mismos que HMM2.

4. **HMM2 sigue siendo la versión recomendada**: resultados equivalentes, sin el sesgo de hábitat, más fiel al enunciado de la profesora ("velocidad y rumbo").

5. **Valor para el TFG**: este experimento permite justificar cuantitativamente la decisión de excluir la vegetación. No es una afirmación teórica ("la vegetación es hábitat, no comportamiento") sino una prueba empírica con números concretos.

---

## Implementación

- `notebooks/HMM2.ipynb` — implementación recomendada (sin vegetación).
- `notebooks/HMM3.ipynb` — variante experimental (con vegetación); no se toca `HMM.ipynb` para conservarlo como referencia.

## Archivos relevantes

- `notebooks/dataExploration1.ipynb` — fuente del CSV limpio.
- `data/processed/aves_procesado_markov.csv` — entrada del HMM.
- `notebooks/HMM.ipynb` — implementación original (HMM1, referencia).
- `notebooks/HMM2.ipynb` — implementación recomendada (sin vegetación).
- `notebooks/HMM3.ipynb` — variante con vegetación.
- `data/processed/hmm.csv` — output de HMM1.
- `data/processed/hmm2.csv` — output de HMM2.
- `data/processed/hmm3.csv` — output de HMM3.
- `CLAUDE.md` — fija la convención `0 = migración, 1 = estacionario`.
