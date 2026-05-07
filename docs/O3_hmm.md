# O3 — Detección de comportamiento con HMM

> Referencia específica para el objetivo O3 del TFG.
> Actualizar tras cada sesión de trabajo en este objetivo.
> Para el detalle completo de cada fase (diseño, experimentos, resultados paso a paso) ver `notebooks/HMM2_plan.md`. Notebook canónico del TFG: `notebooks/HMM5.ipynb`.

## Objetivo

Identificar automáticamente si el ave está en reposo o en migración activa mediante un Hidden Markov Model gaussiano. La versión canónica (HMM5) usa cuatro features recomendadas por la tutora: `step_length`, `turning_angle`, cobertura vegetal (`veg_low`, `veg_high`) y horas de luz diarias (`horas_luz`).

## Notebooks

| Notebook | Output | Descripción |
|---|---|---|
| `HMM.ipynb` | `hmm.csv` | HMM1: implementación original (referencia, tiene bug de `lengths`) |
| `HMM2.ipynb` | `hmm2.csv` | HMM2: referencia simplificada (2 features: step + cos(turn)) |
| `HMM3.ipynb` | `hmm3.csv` | HMM3: experimento — HMM2 + vegetación |
| `HMM4.ipynb` | `hmm4.csv` | HMM4: experimento — HMM2 con 3 estados |
| `HMM5.ipynb` | `hmm5.csv` | **HMM5: CANÓNICA del TFG** — 4 features recomendadas por la tutora |

## Cinco implementaciones comparadas

| Aspecto | HMM1 | HMM2 | HMM3 | HMM4 | HMM5 |
|---|---|---|---|---|---|
| Features | `step`, `turn`, `veg_low`, `veg_high` | `step`, `cos(turn)` | `step`, `cos(turn)`, `veg_low`, `veg_high` | `step`, `cos(turn)` | `step`, `cos(turn)`, `veg_low`, `veg_high`, `horas_luz` |
| `n_components` | 2 | 2 | 2 | **3** | 2 |
| `covariance_type` | `'diag'` | `'diag'` | `'diag'` | `'diag'` | `'diag'` |
| `lengths=` en `.fit()` | **No** — bug | **Sí** | **Sí** | **Sí** | **Sí** |
| Inicializaciones | 1 | 15 seeds | 15 seeds | 15 seeds | 15 seeds |
| Asignación etiquetas | Manual | Auto por `means_` | Auto por `means_` | Auto por `means_` | Auto por `means_` |
| Validación | Sin dinámica | Completa | + concordancia vs HMM2 | + contingencia vs HMM2 | + BIC/AIC + concordancia vs HMM2 |

**HMM5 es la versión canónica del TFG** (sigue las 4 features recomendadas por la tutora: `step_length`, `turning_angle`, cobertura vegetal, horas de luz, con `n_components=2`). HMM2 se conserva como **referencia simplificada de 2 features** para la comparación. HMM3 y HMM4 son experimentos exploratorios intermedios. La justificación de elegir HMM5 sobre HMM2 (mejor patrón estacional, mayor pureza en zona ambigua, mayor utilidad para ML downstream, a pesar del peor BIC por *misspecification* del Gaussiano sobre features no-gaussianas) está en la sección "Decisión final HMM2 vs HMM5" más abajo.

---

## Resultados de HMM2 (referencia simplificada)

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

**Conclusión HMM3**: inocuo (99 % de acuerdo) pero introduce sesgo leve de +1,2 pp en verano. HMM2 es más limpio que HMM3 como referencia simplificada. La versión canónica del TFG es HMM5, que combina vegetación con horas de luz y obtiene un patrón estacional biológicamente más correcto.

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

## Experimento HMM5: ¿añadir vegetación + horas de luz mejora?

**Hipótesis**: la fotoperiodía es un disparador conocido del comportamiento migratorio; en latitudes altas el sol de medianoche permite vuelos prolongados y en invierno boreal los días cortos limitan la actividad. Si la fotoperiodía discrimina estados, debería aparecer una diferencia clara en la media de `horas_luz` entre migración y estacionario.

`horas_luz` se calcula con la fórmula astronómica de Spencer (1971) a partir de `lat` y `day_of_year`, con clipping para sol de medianoche (24 h) y noche polar (0 h). Larus fuscus alcanza ~80° N en verano, donde el clipping es relevante.

### Resultados HMM5 vs HMM2 vs HMM3 (n = 21 081 días)

| Métrica | HMM2 | HMM3 | **HMM5** |
|---|---|---|---|
| LL (train) | −113 068,9 | −121 642,2 | **−171 351,2** |
| nº parámetros | 11 | 19 | 23 |
| **BIC** | **226 247** ✅ | 243 474 | 342 931 |
| **AIC** | **226 160** ✅ | 243 322 | 342 748 |
| % días migración | 26,1 % | 21,9 % | 14,4 % |
| Media step migración (km) | 129,6 | 125,8 | 176,2 |
| Media step estacionario (km) | 4,1 | 4,0 | 6,2 |
| Persistencia est diag | 0,917 | 0,917 | 0,960 |
| Duración media est (días) | 12,1 | 12,0 | 24,7 |
| Concordancia con HMM2 | 100 % | 94,5 % | **86,4 %** |
| Media `horas_luz` migración | — | — | 12,43 h |
| Media `horas_luz` estacionario | — | — | 13,26 h |
| **Δ `horas_luz` mig − est** | — | — | **−0,83 h** |

**ΔBIC(HMM5 − HMM2) = +116 684**. El BIC penaliza fuertemente a HMM5.

### Interpretación inicial (superada tras análisis más profundo)

El análisis inicial concluyó que HMM5 era peor porque:

1. **El LL empeora** aunque HMM5 tiene más parámetros. Motivo: el Gaussian diagonal no captura bien la forma de las nuevas features (`veg_low/veg_high` con masa concentrada en 0; `horas_luz` con patrón estacional bimodal). El coste en log-verosimilitud por dimensión supera cualquier ganancia de discriminación.

2. **La fotoperiodía no discrimina estados por la media**: Δ medias = 0,83 h, mediana prácticamente idéntica (12,03 vs 12,06 h).

3. **HMM5 reasigna 2 826 días** (86,4 % concordancia), sobre todo mig→est en la franja 17–29 km.

Sin embargo, el análisis posterior reveló que el ΔBIC es un *misspecification penalty* (el modelo Gaussiano no es el contenedor óptimo para esas distribuciones) y que cuatro criterios biológicos independientes favorecen a HMM5. **Ver sección "Decisión final" más abajo.**

---

## Decisiones de diseño clave (compartidas por HMM2 y HMM5)

1. **`lengths=` por `trayectoria_id`** — imprescindible; sin él el HMM modela ~480 transiciones espurias entre aves distintas.
2. **`step_length` en bruto, sin `log` ni `StandardScaler`** — la distribución bimodal (pico 0–5 km / cola 50–200+ km) es la señal más fuerte. Escalar la diluye (demostrado en la v1 fallida de HMM2).
3. **`cos(turning_angle)` en lugar de `turning_angle` raw** — resuelve la circularidad (−π y +π son el mismo ángulo) sin necesitar von Mises.
4. **`covariance_type='diag'`** — con varianzas tan dispares (step ~10⁴ km² vs cos ~0,5), full es inestable.
5. **HMM5 añade `veg_low`, `veg_high`, `horas_luz`** siguiendo la recomendación de la tutora. El BIC penaliza estas features por *misspecification* del Gaussiano (no porque sean irrelevantes). HMM2 las omite y funciona como referencia simplificada.

## Limitación conocida del modelo de 2 estados

Con `n_components=2` el estado "migración" mezcla vuelos largos (step > 100 km) con commutes intra-residencia (step 20–100 km). HMM5 mitiga parcialmente esta mezcla usando la señal estacional de `horas_luz`.

**En HMM2** (referencia simplificada, 2 features):
- Mediana de migración = 43 km (no 100+).
- `|turning_angle|` medio en migración = 1,52 rad (~87°).
- 14–21 % de días en "migración" incluso en verano (cría) — falsos positivos de commute.
- 31 % de las rachas de migración duran 1 solo día.

**En HMM5** (canónica, 5 features):
- Mediana de migración = 73,9 km.
- `|turning_angle|` medio en migración = 1,225 rad (~70°) — más cercano al vuelo dirigido.
- Solo 2,8 % de días etiquetados como migración en junio–julio (verano/cría).
- Cohen's d sobre `step_length` = 1,00 (vs 0,88 en HMM2) — mayor separación entre estados.

**Justamente por esa limitación, HMM5 (que añade vegetación + horas de luz) corrige varios de estos síntomas** sin necesitar un tercer estado, manteniendo la restricción de `n_components=2`. Ver siguiente sección.

---

## Decisión final: HMM5 como versión canónica

La tutora recomendó cuatro features (`step_length`, `turning_angle`, cobertura vegetal, horas de luz) y dejó la decisión final al alumno. Tras evaluar HMM2 (2 features) frente a HMM5 (5 features) con métricas estadísticas y biológicas, **se elige HMM5 como versión canónica del TFG**.

### Argumentos a favor de HMM5

1. **Patrón estacional biológicamente correcto** (el argumento más fuerte):

| Período | HMM2 % mig | HMM5 % mig | Esperado biología |
|---|---|---|---|
| Jun + Jul (cría) | 23,7 % | **2,8 %** | muy bajo |
| Ene + Feb (invernada) | 13,8 % | **3,6 %** | bajo |
| Abr + Sep (paso migratorio) | 37,2 % | 27,6 % | pico |

   En verano las aves están en colonia incubando en latitudes altas (55-70°N): prácticamente ningún día debería ser migración. HMM2 etiqueta el 23,7 % de los días de junio-julio como migración (el conocido falso positivo "commute mezclado con migración"); HMM5 lo baja al 2,8 %, alineado con la biología real.

2. **Pureza en la zona ambigua 10-50 km** (n = 4 935 días): HMM2 etiqueta el 73,2 % como migración; HMM5 solo el 19,2 %. Un movimiento diario de 15-30 km en una gaviota es típicamente forrajeo o commute desde colonia, no migración de larga distancia.

3. **% migración global más cercano a la biología**: HMM2 26,1 % vs HMM5 14,4 % vs rango biológico esperado ~17-25 % (Larus fuscus migra ~2-3 meses/año). Ambos son plausibles; HMM5 es más conservador.

4. **Mayor separación de estados**: Cohen's d sobre `step_length` = 0,88 (HMM2) vs **1,00 (HMM5)**. La media de step en migración pasa de 79 km (HMM2) a 176 km (HMM5), reflejando un estado "migración" más puro (vuelos largos reales en lugar de commutes mezclados).

5. **No pierde migraciones reales**: HMM5 mantiene el 100 % de los días con step > 100 km como migración (idéntico a HMM2) y el 98,3 % de los 50-100 km. La reducción del % migración total ocurre exclusivamente en la zona ambigua 10-50 km.

6. **Útil para ML downstream**: el TFG predice posición diaria; un `estado_hmm` que separe migración real de commute permite al modelo de ML aprender estados con dinámicas espaciales distintas. HMM5 ofrece esa segregación, HMM2 los mezcla.

### El BIC favorece a HMM2 — por qué se acepta esta penalización

| | HMM2 | HMM5 |
|---|---|---|
| Log-likelihood | −113 069 | −171 351 |
| BIC | **226 247** | 342 931 |
| ΔBIC | — | **+116 684** |

El ΔBIC = +116 684 a favor de HMM2 es enorme y no debe ignorarse. Pero la mayor parte de esa penalización viene de **misspecification del modelo Gaussiano**, no de irrelevancia de las nuevas features:

- `veg_low` y `veg_high` son **fuertemente cero-infladas** (mucha masa en valor exacto 0): el Gaussian asigna densidad ridículamente baja a un 0 cuando la media del estado no es 0, lo que penaliza el log-likelihood sin que esto refleje un peor estado.
- `horas_luz` es **bimodal a nivel población** (joroba invierno ~11 h, joroba verano ~17-21 h): un único Gaussian por estado no la captura bien.

La literatura de selección de modelos bajo *misspecification* (Vuong 1989; Lv & Liu 2014) advierte que el BIC pierde su garantía de consistencia cuando la familia del modelo es incorrecta — que es exactamente el caso aquí. **El BIC mide cómo de bien el Gaussian HMM comprime los datos, no cómo de correctamente identifica los estados conductuales.** Si el experimento se repitiera con un modelo Beta para vegetación o una mezcla de Gaussianos para horas de luz, la penalización desaparecería.

Por eso se acepta el peor BIC como coste consciente: las features tienen contenido biológico genuino (Cohen's d significativo, p-value Mann-Whitney < 1e-22 para horas_luz), pero el modelo Gaussian no es el contenedor probabilístico óptimo. La validación biológica multidimensional (cuatro criterios independientes) prevalece sobre un único criterio estadístico afectado por *misspecification*.

### Caveats honestos sobre HMM5

- **Duración media estacionario = 24,7 días** (vs 12,1 en HMM2). Plausible porque cría dura ~2 meses e invernada varios meses, pero está en el extremo alto. Verificar con histograma de duraciones.
- **Concordancia 86,4 % con HMM2**: 2 826 días reasignados, casi todos en step 17-29 km y casi todos en sentido mig→est. Defendible si esos días son commutes (que es la interpretación biológica), no defendible si la tutora considera commute como migración.

### Convención canónica del TFG

A partir de esta decisión:

- `data/processed/hmm5.csv` es la **fuente canónica del estado conductual** para el resto del TFG.
- El pipeline ML downstream (ML0) debe usar `hmm5.csv` como entrada.
- HMM2 se mantiene como referencia simplificada para la sección comparativa de la memoria.

---

## Por qué `horas_luz` cambia el modelo y `veg_low/veg_high` casi no — análisis del impacto por feature

Una observación importante para la memoria: el salto HMM2 → HMM3 añade **dos features** (`veg_low`, `veg_high`) y casi no modifica el modelo (99 % de concordancia, 212 días reclasificados); el salto HMM3 → HMM5 añade **una sola feature** (`horas_luz`) y produce ~1 776 reclasificaciones (8× más que las dos variables de vegetación juntas). Y sin embargo, el gráfico Cohen's d en HMM5 marca `veg_high` como aparentemente más influyente que `horas_luz`. Esta aparente paradoja tiene una explicación directa.

### Cohen's d en el gráfico ≠ influencia causal sobre la clasificación

El gráfico de Cohen's d se calcula **sobre la clasificación final de HMM5**: dado el etiquetado resultante, mide qué tan distintas son las medias de cada feature entre estados. Es una métrica del *resultado*, no de la *causa*.

La prueba directa es comparar Cohen's d de `veg_high` antes y después de añadir `horas_luz`:

| Modelo | veg_high mig | veg_high est | Cohen's d veg_high |
|---|---|---|---|
| **HMM3** (sin horas_luz) | 0,33 | 0,33 | **≈ 0** |
| **HMM5** (con horas_luz) | 0,22 | 0,35 | **−0,41** |

En HMM3, `veg_high` no discriminaba absolutamente nada entre estados — las medias eran idénticas. Pasa de Cohen's d ≈ 0 a −0,41 en HMM5 sin que se haya tocado `veg_high`: lo único que cambió fue añadir `horas_luz`. El "poder discriminativo" aparente de `veg_high` en HMM5 es un **artefacto inducido**: cuando `horas_luz` reclasifica los ~1 776 días dudosos (mayoritariamente commutes de verano en colonia), esos días resultan tener distribuciones distintas de vegetación (los commutes en colonia tienen `veg_high` alto; los vuelos de migración real, más bajo). La vegetación se limita a *correlacionar* con la nueva clasificación, no la causa.

Lo simétrico ocurre con `veg_low`: en HMM3 tenía Cohen's d ≈ −0,24 (pequeño pero real); en HMM5 cae a −0,09 (mínimo). Su poder discriminativo se *diluye* porque la clasificación nueva no se alinea bien con `veg_low`.

### Tres razones del impacto real de `horas_luz`

1. **Estructura calendárica que la vegetación no tiene**. `horas_luz` es esencialmente función de `lat × día_del_año`: 17–21 h en verano boreal alto, 9–12 h en invierno, 11–14 h en latitudes medias en primavera/otoño. Esa estructura está directamente alineada con el comportamiento real (cría en verano = estacionario, migración en primavera/otoño, invernada = estacionario). `veg_low/veg_high` son features espaciales sin estructura temporal sistemática — un día con alta vegetación puede ser commute, migración o reposo en hábitats variados.

2. **Alineación con la zona de confusión de `step_length`**. HMM2 confunde sistemáticamente días con `step_length` de 15-30 km (73 % los etiqueta migración cuando la mayoría son commute). Resulta que **muchos de esos días confundidos son commutes de verano desde colonia** (lat 60-70°N, `horas_luz` = 17-21 h): exactamente donde `horas_luz` es más informativa. La vegetación no se concentra en esa zona — un valor alto de `veg_high` ocurre tanto en bosques de paso migratorio como en colonias de cría —, así que no aporta poder de desambiguación.

3. **Varianza absoluta y *leverage* en el Gaussiano**. Aunque Cohen's d ya normaliza por σ, los valores extremos producen penalizaciones cuadráticas:

| Feature | σ | Rango típico | *Leverage* máximo (Δ/σ)² |
|---|---|---|---|
| `veg_high` | 0,36 | [0, 1] | (0,7/0,36)² ≈ 3,8 |
| `horas_luz` | 2,55 | [8, 21] h | (10/2,55)² ≈ 15,4 |

Los extremos de `horas_luz` (sol de medianoche en cría boreal, días cortos en invernada subtropical) generan penalizaciones de Mahalanobis hasta ~4× más fuertes que los extremos de vegetación. Esos extremos anclan los días correspondientes al estado estacionario con mucha autoridad. La vegetación, con rango más estrecho y sin valores extremos sistemáticos, no tiene este efecto de anclaje.

### Lectura para la memoria

Lo que el gráfico de Cohen's d en HMM5 muestra **no es** "veg_high contribuyó más a la clasificación que horas_luz". Lo que muestra es "tras la clasificación final, las medias marginales de `veg_high` están ligeramente más separadas entre estados que las de `horas_luz`". Son métricas distintas:

- **Cohen's d post-hoc** = qué tan distintas son las distribuciones marginales **dado** el etiquetado final.
- **Influencia causal** = qué feature movió la frontera de decisión.

La feature que **causó** el reordenamiento de etiquetas fue `horas_luz`; la vegetación se limitó a correlacionar con esa nueva clasificación. Esto explica por qué dos features de vegetación apenas modifican el modelo (HMM2 → HMM3, 99 % concordancia) mientras una sola feature de fotoperiodía reclasifica el 8 % de los días (HMM3 → HMM5).
