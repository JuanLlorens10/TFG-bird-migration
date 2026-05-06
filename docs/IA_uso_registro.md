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

### O2 — Predicción estadística con Markov

#### Redacción de `docs/O2_markov.md` como referencia del notebook `markov1.ipynb`

- **Rol de la IA**: leer el notebook celda a celda, verificar las cifras ejecutando una réplica del pipeline (1 253 celdas únicas, 21 561 transiciones, % self-loops por mes), y redactar un documento que sustituya el placeholder previo (~20 líneas con campos "a completar") por una referencia operativa: input heredado de O1, pipeline en 3 pasos, cobertura mensual de las matrices, decisiones de diseño justificadas (resolución 0,5°, anclaje en `(lon_min, lat_min)`, formato disperso), limitaciones (self-loops dominantes del 64–90 %, cold start con 90 % de la rejilla nunca visitada, sin suavizado, matrices no persistidas) y próximos pasos sugeridos.
- **Aportación propia**: encargo de mantener el mismo formato que `O1_datos.md`; aprobación de la lectura crítica de los self-loops como punto clave para la memoria (el modelo Markov es una baseline cuyo valor real se mide en días de migración, no en accuracy global).
- **Validación**: ejecución programática del pipeline para confirmar transiciones, celdas únicas y porcentajes mensuales antes de incluirlos en el documento.
- **Prompt clave** (`conversation_log.md`): `[2026-05-06 17:01]` "redacta O2_markov.md igual".

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

#### HMM5 — experimento con vegetación + horas de luz

- **Rol de la IA**: diseñar una variante que añade tres features sobre HMM2 (`veg_low`, `veg_high`, `horas_luz`); implementar la función `daylight_hours` con la fórmula astronómica de Spencer 1971 y clipping para sol de medianoche / noche polar; construir un protocolo de comparación formal con BIC y AIC además de la concordancia día a día usada en HMM3/HMM4; añadir un gráfico de barras de Cohen's d por feature.
- **Aportación propia**: petición explícita del experimento tras razonar que el análisis previo de complementariedad de vegetación abría la pregunta sobre la fotoperiodía como feature ortogonal.
- **Validación**: 15 seeds convergen al mismo LL (`-171 351,2`); sanity checks de `daylight_hours` contra valores conocidos (ecuador 12 h, 60° N solsticio 18,5 h, 80° N solsticio 24/0 h) pasan; concordancia día-a-día con HMM2 (86,4 % sobre 20 836 días alineados) y matriz de confusión 2×2; test Mann-Whitney U sobre `horas_luz` entre estados.
- **Lectura inicial (parcialmente errónea)**: la primera lectura concluyó que HMM5 era "peor" que HMM2 por el ΔBIC = +116 684. La IA presentó esa conclusión y el alumno la cuestionó: "el patrón estacional parece tener más sentido en HMM5".
- **Caso de corrección crítica** (relevante para el anexo): el alumno señaló que el patrón estacional biológico era netamente mejor en HMM5 (verano 2,8 % vs 23,7 %; invierno 3,6 % vs 13,8 %) y que la tutora había recomendado las cuatro features. Esto obligó a la IA a reabrir la conclusión, profundizar en por qué el BIC penalizaba HMM5 (*misspecification* del Gaussiano sobre features no-gaussianas: vegetación cero-inflada, horas de luz bimodal estacional) y separar la noción de "ajuste estadístico" de "validez biológica". *Lección: la primera respuesta de la IA seguía un único criterio (BIC) y producía un veredicto incorrecto; el alumno aplicó conocimiento biológico y forzó una reevaluación más completa.*

#### HMM5 — Decisión final (versión canónica del TFG)

- **Rol de la IA**: tras la corrección del alumno, recomponer la evaluación final con razonamiento profundo (ultrathink) cubriendo cuatro dimensiones: estadística (BIC y *misspecification penalty*), biológica (patrón estacional, pureza de bins, % migración global), metodológica (recomendación de la tutora, objetivo predictivo del TFG) y defensiva (defendibilidad ante el tribunal). Escribir la justificación en `docs/O3_hmm.md` con citas a Vuong (1989) y Lv & Liu (2014) sobre selección de modelos bajo *misspecification*.
- **Aportación propia**: aclarar la framing real del enunciado de la tutora (cuatro features recomendadas, decisión final al alumno), interpretar críticamente los resultados, y tomar la decisión final de adoptar HMM5 como canónica del TFG.
- **Validación**: cuatro criterios biológicos independientes a favor de HMM5 (patrón estacional, pureza zona ambigua 10-50 km, % migración global, Cohen's d sobre `step_length`); HMM5 no pierde migraciones reales (100 % de step > 100 km siguen siendo migración).
- **Conclusión asumida**: HMM5 es la versión canónica; HMM2 se conserva como referencia simplificada. El peor BIC se acepta como coste consciente, justificándolo en la memoria como *misspecification penalty* — el Gaussiano no es el contenedor probabilístico óptimo para vegetación cero-inflada y horas de luz bimodal, pero las features sí tienen contenido biológico genuino (Cohen's d significativo, p < 1e-22).

### O4 — Predicción con Machine Learning

#### Migración de ML3/ML4/ML5 a hmm5.csv

- **Rol de la IA**: identificar que ML3–ML5 leían `hmm.csv` (etiquetas HMM1), que `hmm5.csv` no incluye las columnas derivadas `grid_x`, `grid_y`, `cell_id`, `next_lat`, `next_lon`, y actualizar la celda de carga de cada notebook añadiendo el cálculo del grid (resolución 0,5° igual que `markov1.ipynb`) y `next_lat`/`next_lon` para la evaluación de error geográfico.
- **Aportación propia**: decisión de usar `hmm5.csv` como fuente canónica en todos los notebooks ML para que el `estado_hmm` refleje las etiquetas biológicamente más correctas de HMM5.
- **Validación**: `grep read_csv notebooks/ML3.ipynb notebooks/ML4.ipynb notebooks/ML5.ipynb` confirma que los tres apuntan a `hmm5.csv`; ML6 se mantiene sobre `hmm_wind.csv` (experimento de viento ERA5 no migrable sin reconstruir el dataset). Los notebooks no se han re-ejecutado — resultados históricos de la tabla de evolución en `docs/O4_ml.md` están marcados con ‡ como pendientes de actualización.
- **Nota**: cambiar la fuente a `hmm5.csv` actualiza `estado_hmm` pero **no** elimina el leakage en `step_length`/`bearing` de ML3–ML5 (salto t→t+1). Solo ML0 lo corrige recalculando como t-1→t.

#### ML5 / ML6 — análisis comparativo

- **Rol de la IA**: identificar que ML6 empeora respecto a ML5 a pesar de añadir viento ERA5 a 850 hPa, y articular la razón (el viento a esa presión no captura el viento efectivo a baja altitud).
- **Aportación propia**: aceptar la conclusión negativa como resultado válido del TFG (ejemplo de "qué cambios empeoran y por qué" — el eje comparativo que pide la tutora).

#### Curación de la línea ML — eliminación de ML1 y ML2

- **Rol de la IA**: inventariar los 7 notebooks (ML0–ML6) y cruzar el contenido real con `docs/O4_ml.md` y `notebooks/O4_plan.md`; detectar dos hechos no reflejados en la documentación ("`ML0.ipynb` no aparece en `O4_ml.md` aunque ya existe el código" y "`ML1`/`ML2` quedan dominados por `ML3` porque `semana_num` única absorbe la señal de mes + día semana + fase lunar"); proponer en plan-mode tres niveles de poda (mínimo / conservador / agresivo) para que el alumno decida; ejecutar la opción mínima elegida (borrar `ML1`/`ML2`, conservar `ML0` + `ML3`–`ML6`); actualizar la documentación coherentemente (`docs/O4_ml.md`, `CLAUDE.md`) marcando con † las filas históricas y dejando los resultados de `ML1`/`ML2` como contexto.
- **Aportación propia**: decisión final del alcance de la limpieza (rechazo deliberado de la opción agresiva para preservar la narrativa de evolución de features que pide la tutora — el TFG exige justificar qué cambios mejoran o empeoran y por qué); aprobación del borrado pese a haber modificaciones locales sin commitear en `ML1` (verificadas como ruido de re-ejecución, no como trabajo no salvado); validación de que la fila `ML0` se añade como "pendiente" sin inventar métricas.
- **Validación**: `git diff notebooks/ML1.ipynb` antes de borrar para confirmar que las modificaciones locales eran solo timings + un accuracy estocástico de LightGBM (5 líneas cambiadas, ningún cambio de contenido); `ls notebooks/ML*.ipynb` tras el borrado muestra exactamente {ML0, ML3, ML4, ML5, ML6}; `grep` en `docs/O4_ml.md` y `CLAUDE.md` confirma que `ML1`/`ML2` solo aparecen marcados con † en la tabla histórica.
- **Prompt clave** (`conversation_log.md`): `[2026-05-06 17:06]` "Creo que tengo demasiadas versiones de ML. Lee el documento O4_ml.md y dame qué candidatos conservar y cuáles puedo borrar.".
- **Lección**: la documentación de evolución (`O4_ml.md`) puede sobrevivir al código que la generó. Para el TFG es preferible consolidar resultados en la tabla y eliminar notebooks redundantes (recuperables desde git) que mantener fuentes paralelas que se desincronizan.

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
