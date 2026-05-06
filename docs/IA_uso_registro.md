# Registro estructurado de uso de IA

> Documento curado para preparar la **declaración de uso de IA** exigida por la UPM.
> A diferencia de `conversation_log.md` (transcripción cronológica completa), aquí se mantiene una visión estructurada por tarea, lista para volcar al anexo del TFG.
> **Mantener actualizado tras cada decisión asistida por IA**.

## 1. Herramientas y modelos

| Herramienta | Modelo | Identificador | Periodo de uso | Tareas principales |
|---|---|---|---|---|
| Claude Code (CLI Anthropic) | Claude Opus 4.7 | `claude-opus-4-7` | mayo 2026 – actualidad | Diseño de experimentos, análisis comparativo, refactorización |
| Claude Code (CLI Anthropic) | Claude Sonnet 4.6 | `claude-sonnet-4-6` | uso puntual | Tareas más cortas |

URL de la herramienta: `https://claude.com/claude-code`.

## 2. Inventario de tareas asistidas (por objetivo)

### O1 — Preparación de datos

#### Redacción de `docs/O1_datos.md` como referencia del pipeline de limpieza

- **Rol de la IA**: leer `notebooks/dataExploration1.ipynb` celda a celda, verificar el CSV producido (`data/processed/aves_procesado_markov.csv`) ejecutando una inspección programática, detectar dos errores en la versión previa del documento (afirmaba que `veg_low`/`veg_high` se derivaban en este notebook cuando en realidad vienen del raw, y listaba `grid_x`/`grid_y`/`cell_id` en el output cuando esas columnas las añade `markov1.ipynb`), y redactar un documento auto-contenido con el pipeline completo, los números reales de cada paso (89 867 → 23 585 → 22 375 → 22 041), la justificación de cada decisión de diseño y las limitaciones que conviene reflejar en la memoria.
- **Aportación propia**: encargo explícito de que el documento sirva tanto de referencia operativa para futuros cambios en el notebook como de fuente para la memoria; validación de que la información reflejada coincide con el código.
- **Validación**: contraste programático con `pandas.read_csv` del CSV final (8 columnas, 22 041 filas, 117 aves, 480 trayectorias, rango 2009-05-25 → 2015-08-23) y con la distribución horaria final (14h: 15 408 / 13h: 4 548 / 15h: 2 085).
- **Prompt clave** (`conversation_log.md`): `[2026-05-06 16:47]` "Redacta O1_datos.md correctamente en base a la información proporcionada por dataExploration1…".

### O3 — Detección de comportamiento (HMM)

#### HMM2 — corrección del bug de `lengths=` y rediseño

- **Rol de la IA**: identificar el bug crítico de HMM1 (no pasaba `lengths=` a `.fit()`, lo que producía ~480 transiciones espurias entre aves), proponer el rediseño con `cos(turning_angle)` en lugar de `turning_angle` raw, y diseñar el plan completo (`HMM2_plan.md`) con justificación de cada decisión frente a alternativas.
- **Aportación propia**: validar la lectura del bug ejecutando los notebooks y comparando resultados HMM1 vs HMM2. Decidir mantener `step_length` sin escalar tras ver el comportamiento de la v1 fallida (StandardScaler).
- **Validación**: 15 inicializaciones convergen al mismo log-likelihood (`-113 068,9`). Métricas comportamentales (paso medio, persistencia diagonal, patrón estacional) coherentes con literatura.
- **Prompt clave**: ver `conversation_log.md` entrada `[2026-05-03 17:04]` ("Ultrathink: O3 - Detección de comportamiento HMM…").

#### HMM3 — experimento con vegetación

- **Rol de la IA**: diseñar una variante experimental que cambia **una sola variable** respecto a HMM2 (añadir `veg_low` + `veg_high`) para aislar su efecto, y diseñar el análisis de concordancia día a día.
- **Aportación propia**: aprobar el diseño "una variable cambiada a la vez" como metodología comparativa estándar.
- **Validación**: concordancia 98,99 % vs HMM2; medias de vegetación por estado casi idénticas; sesgo +1,2 pp de migración en verano detectado.
- **Conclusión asumida**: la vegetación no discrimina entre estados → HMM2 sigue siendo la referencia.

#### HMM4 — experimento con 3 estados

- **Rol de la IA**: proponer y diseñar el experimento (única variable cambiada: `n_components=3`); ejecutar el notebook end-to-end; redactar la Fase 7 del plan; identificar el hallazgo inesperado tras los resultados (la frontera principal del tercer estado está entre quietud casi total y movimiento leve, no entre commute y migración como anticipaba la hipótesis original).
- **Aportación propia**: decidir saltar la restricción de `n_components=2` sabiendo que el experimento es exploratorio; interpretar críticamente que la hipótesis original del plan se confirmaba solo parcialmente.
- **Validación**: 15 seeds convergen al mismo LL (`-98.614,3`); tabla de contingencia HMM4×HMM2 con 21 081 días emparejados; verificación de los 4 síntomas listados en HMM2_plan.md.
- **Caso de corrección crítica** (relevante para el anexo): el plan original asumía que el commute de HMM4 caería en `mig(HMM2)`. La ejecución mostró lo contrario (84,4 % en `est(HMM2)`). La narrativa de la Fase 7 se reescribió tras los datos para reflejar el hallazgo real, no la hipótesis previa.

### O4 — Predicción con Machine Learning

#### ML5 / ML6 — análisis comparativo

- **Rol de la IA**: identificar que ML6 empeora respecto a ML5 a pesar de añadir viento ERA5 a 850 hPa, y articular la razón (el viento a esa presión no captura el viento efectivo a baja altitud).
- **Aportación propia**: aceptar la conclusión negativa como resultado válido del TFG (ejemplo de "qué cambios empeoran y por qué" — el eje comparativo que pide la tutora).

#### ML0 — corrección de OOM en `RandomizedSearchCV`

- **Rol de la IA**: diagnosticar el cierre repetido del kernel de Jupyter al ejecutar `notebooks/ML0.ipynb`. Causa: los tres `RandomizedSearchCV` usaban `n_jobs=-1` sobre una máquina de 15 GB RAM × 8 cores; con ~1100-1300 clases (celdas) cada estimador multiclase (RF con `class_weight='balanced'`, LGBM con `num_leaves` altos, XGB `multi:softprob`) ocupa varios GB y 8 procesos paralelos disparaban el OOM-killer. Aplicación de la corrección: `n_jobs=1` + `pre_dispatch=1` en los tres searches, recorte de `n_estimators` (≤300 RF, ≤400 XGB/LGBM), `min_samples_leaf` ≥ 5, `num_leaves` ≤ 63, y eliminación de `class_weight='balanced'` (multiplica memoria por n_classes con poco beneficio en problemas tan multiclase).
- **Aportación propia**: detectar y reportar el síntoma ("se me cierra la sesión") con la información operativa necesaria; decisión de no rebajar `N_ITER_RANDOM`/`CV_SPLITS` (ya están en 3/3 prototipo) sino atacar la causa real (paralelismo + tamaño del modelo).
- **Validación**: pendiente — ejecutar `tfg_env/bin/jupyter nbconvert --to notebook --execute --inplace notebooks/ML0.ipynb` y verificar que termina sin OOM antes de subir a presupuesto publicable.
- **Prompt clave** (`conversation_log.md`):
  - `[2026-05-04 20:43]` "No puedo ejecutar ML0 sin que se me cierre la sesión. Corrígelo".
- **Lección**: replicar el patrón ya documentado en ML1 (`n_jobs=1, "USAR SOLO 1 NÚCLEO AQUÍ TAMBIÉN"`) — la línea ML existente había aprendido este límite y ML0 lo había olvidado al rediseñarse.

### Estructura del proyecto

#### Reorganización de la documentación (CLAUDE.md → docs/)

- **Rol de la IA**: proponer y ejecutar la reestructuración a documento general + 5 sub-documentos por objetivo, asegurando que ninguna información se pierde (267 → 104 líneas en CLAUDE.md, contenido íntegro extraído a `docs/Ox_*.md`).
- **Aportación propia**: definir la estructura general (un documento por objetivo, mantener instrucciones de log y git en CLAUDE.md, separar documentación de código en `docs/`).

#### Plan modesto: gestión del proyecto

- **Rol de la IA**: mantener el `conversation_log.md` con resúmenes de cada respuesta; añadir entradas a `docs/MEMORIA_redaccion.md` cuando se toman decisiones de redacción; preservar la trazabilidad notebook → memoria.
- **Aportación propia**: directrices explícitas sobre estilo de trabajo (énfasis comparativo, explicación tras editar código, mantener log).

#### Creación de la skill `/tfg-code` (`.claude/skills/tfg-code/SKILL.md`)

- **Rol de la IA**: consolidación de convenciones técnicas dispersas (HMM, ML, sanity checks, anti-patrones) en un único documento operativo invocable; diseño de los 4 flujos procedimentales (primera versión sin referencia, variante con cambio único, documentar resultado, commit + push); formulación de la regla "un único `notebooks/Ox_plan.md` por subapartado, acumulativo". Doble pasada IA: borrador local + refinamiento por **Ultraplan** (sesión remota de Claude Code on the web) que corrigió el formato de la skill (directorio `tfg-code/SKILL.md`, no archivo único) y aportó la verificación detallada.
- **Aportación propia**: decisión de qué convenciones consolidar y bajo qué granularidad (UNA skill general en lugar de varias focalizadas); ubicación a nivel de proyecto (`.claude/skills/`) versionada en git en lugar de a nivel de usuario; refinaciones explícitas tras la primera propuesta — distinción **Flujo 1 (primera versión sin referencia)** vs **Flujo 2 (variante con cambio único)**, y la regla **`notebooks/Ox_plan.md` por subapartado** que separa proceso (acumulativo) de resultados (`docs/Ox_*.md`, reescribibles).
- **Validación**: edición manual del SKILL.md tras revisar la salida de Ultraplan; verificación de cobertura (las 8 convenciones críticas + los 8 anti-patrones + los 4 flujos están presentes).
- **Prompts clave** (`conversation_log.md`):
  - `[2026-05-04 18:11]` "Ahora quiero que en base a nuestra conversación crees una skill de cómo programar el código para este proyecto."
  - `[2026-05-04 18:30]` "no siempre tenemos un primer archivo de referencia… cuando no lo tengamos… debemos de crear una primera versión del código concienzuda, creando un plan detallado…" (origen del Flujo 1).
  - `[2026-05-04 18:40]` "cada vez que hagamos un plan para el desarrollo de una parte en específica este se debe guardar en un markdown específico para ese subapartado del trabajo" (origen de la regla `Ox_plan.md`).
- **Casos de corrección crítica**:
  - La primera versión local del plan asumía `.claude/skills/tfg-code.md` (archivo único). Ultraplan corrigió a directorio `.claude/skills/tfg-code/SKILL.md` (formato real de Claude Code). Ejemplo de **validación cruzada IA ↔ IA** que evitó un error que habría hecho la skill inservible.

## 3. Prompts curados (selección representativa)

Selección de prompts del `conversation_log.md` que ilustran momentos clave de decisión. Para el anexo final, refinar a 5–10 según el espacio disponible.

| Fecha | Prompt (extracto) | Por qué es relevante |
|---|---|---|
| 2026-05-03 16:42 | "Según mi tutora… es importante ver qué soluciones funcionan frente a otras y explicar el por qué…" | Define el eje metodológico del TFG (análisis comparativo) y se aplicó coherentemente en HMM2/3/4 y ML1–6. |
| 2026-05-03 17:04 | "Ultrathink: O3 — Detección de comportamiento (HMM)… 1) plan, 2) analizar HMM1, 3) comparar y ver cuál es mejor" | Origen de la cadena HMM1 → HMM2 — pidió a la IA un plan independiente antes de ver el código existente, para evitar sesgo de confirmación. |
| 2026-05-04 10:05 | "Quiero que hagamos una versión HMM4 que sea similar a HMM2 pero que añada un tercer estado para solucionar los problemas encontrados" | Decisión de saltarse la restricción de la profesora como experimento exploratorio. |
| 2026-05-04 17:27 | "Tienes que hacer ese análisis asumiendo que no existe ninguno de los archivos ML, y decirme si te parece más lógico construirlos en base a HMM2 o HMM4" | Pidió un razonamiento independiente para validar la elección de HMM2 → HMM4 sin restricciones externas. La IA recomendó HMM2 por motivos puramente ML (clase commute impredecible). |
| 2026-05-04 17:40 | "He pensado en la siguiente estrategia de estructurar mi archivo claude.md…" | Decisión arquitectónica del proyecto tomada por el alumno; la IA solo ejecutó la reestructuración. |

## 4. Validación y verificación crítica

Patrón general de validación seguido en este TFG:

1. **Ejecución end-to-end del notebook tras cada cambio**: `tfg_env/bin/jupyter nbconvert --to notebook --execute --inplace ...`. La salida se inspecciona antes de reportar el resultado como completo.
2. **Sanity checks numéricos** explícitos en los notebooks (gaps entre medias, dispersión de log-likelihoods entre seeds, % población por estado). Cualquier valor sospechoso emite warning visible.
3. **Comparación día a día con la implementación de referencia** (HMM2 frente a HMM3 y HMM4): tablas de contingencia sobre 21 081 días emparejados.
4. **Comparación con literatura / sentido biológico**: % de migración por mes coherente con cría/migración real; vuelos rectos en migración real; ave en residencia con step ≪ 1 km.

## 5. Casos de corrección crítica (la IA se equivocó)

Estos son los más útiles para demostrar que el alumno valida y no se limita a aceptar la salida.

- **HMM4 — hipótesis del plan original incorrecta**: el plan inicial de HMM4 (escrito por la IA) asumía que el commute caería en `mig(HMM2)`. La ejecución mostró lo contrario (84,4 % en `est(HMM2)`). La narrativa de la Fase 7 se reescribió tras los datos. *Lección: el plan se contrasta con la realidad y se actualiza.*
- *(Pendiente añadir más casos a medida que ocurran.)*

## 6. Checklist final antes de redactar el anexo

- [ ] Confirmar lista completa de modelos usados (revisar el log al final).
- [ ] Curar la lista de prompts a 5–10 (sección 3).
- [ ] Cerrar la lista de tareas asistidas por capítulo (sección 2).
- [ ] Revisar que los `docs/Ox_*.md` reflejan **decisiones propias justificadas**, no solo descripciones generadas por IA.
- [ ] Añadir todos los casos de corrección crítica detectados (sección 5).
- [ ] Decidir formato exacto de la cita (numérica `[1]` vs nota al pie).
