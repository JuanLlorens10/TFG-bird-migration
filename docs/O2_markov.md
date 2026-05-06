# O2 — Predicción estadística con Markov

> Referencia específica para el objetivo O2 del TFG.
> Documento de referencia para futuros cambios en `markov1.ipynb` y para la redacción de la memoria.

## Objetivo

Construir tablas empíricas de probabilidad de transición entre celdas de una malla espacial, **segmentadas por mes del año**, a partir de los movimientos diarios observados de las gaviotas. La predicción estadística para un día consiste en, dada la celda actual y el mes, devolver la celda destino más probable según el histórico.

## Notebook

`notebooks/markov1.ipynb` → lee `data/processed/aves_procesado_markov.csv`, produce 12 tablas de transición mensuales en memoria (no se serializan a disco).

## Datos de entrada

`data/processed/aves_procesado_markov.csv` — salida de O1.

| Atributo | Valor |
|---|---|
| Filas | 22 041 |
| Aves | 117 |
| Trayectorias | 480 |
| Rango temporal | 2009-05-25 → 2015-08-23 |
| Rango geográfico | lon 7,72° → 52,50° (span 44,8°) · lat −2,41° → 65,17° (span 67,6°) |

Columnas usadas: `trayectoria_id`, `date`, `lon`, `lat`. Las demás (`hora`, `veg_low`, `veg_high`, `animal_id`) se ignoran en este objetivo.

## Pipeline

### Paso 1 — Construcción de la malla espacial

Se define una rejilla regular de resolución **0,5° × 0,5°** anclada en la esquina suroeste del conjunto de datos:

```python
res = 0.5
lon_min = df['lon'].min()   # 7,72°
lat_min = df['lat'].min()   # −2,41°
df['grid_x'] = ((df['lon'] - lon_min) / res).astype(int)
df['grid_y'] = ((df['lat'] - lat_min) / res).astype(int)
df['cell_id'] = df['grid_x'].astype(str) + "_" + df['grid_y'].astype(str)
```

| Métrica | Valor |
|---|---|
| Resolución | 0,5° × 0,5° (~55 km en latitud; ~25–55 km en longitud según la latitud, por el coseno) |
| Extensión teórica de la rejilla | 90 columnas × 136 filas = 12 240 celdas posibles |
| **Celdas efectivamente visitadas** | **1 253** (~10 % de la rejilla) |
| Formato de `cell_id` | `"{grid_x}_{grid_y}"` (string) |

### Paso 2 — Generación de transiciones día a día

Por cada trayectoria continua se calcula la celda del día siguiente con `groupby('trayectoria_id').shift(-1)`. El último día de cada trayectoria queda con `next_cell_id = NaN` y se descarta.

| Métrica | Valor |
|---|---|
| Filas con destino válido | 21 561 |
| Filas eliminadas (último día de cada trayectoria) | 480 (= número de trayectorias) |

> El uso de `trayectoria_id` aquí es crítico: si se hiciera `shift(-1)` sobre `animal_id`, se generarían transiciones espurias entre días no consecutivos cuando un ave tiene huecos GPS.

### Paso 3 — 12 matrices mensuales

Para cada mes `m ∈ {1,…,12}` se filtra el dataset de transiciones, se agrupa por `(cell_id → next_cell_id)`, se cuenta la frecuencia y se normaliza por la suma de salidas de cada celda origen:

```
T[m][i, j] = N(i → j en mes m) / Σ_k N(i → k en mes m)
```

Se almacenan en formato disperso (DataFrame `cell_id, next_cell_id, frecuencia, probabilidad`) en el diccionario `matrices_mensuales`.

#### Cobertura mensual de las matrices

| Mes | Transiciones | Transiciones únicas (origen→destino) | % de self-loops (i → i) |
|---|---|---|---|
| Enero | 1 487 | 102 | 89,0 % |
| Febrero | 1 330 | 98 | 89,9 % |
| Marzo | 1 385 | 171 | 83,0 % |
| Abril | 1 192 | 370 | 65,4 % |
| Mayo | 1 134 | 213 | 72,7 % |
| Junio | 1 528 | 105 | 78,7 % |
| Julio | 1 751 | 119 | 76,1 % |
| Agosto | 2 934 | 490 | 75,9 % |
| **Septiembre** | **2 972** | **995** | **64,5 %** |
| Octubre | 2 298 | 693 | 68,2 % |
| Noviembre | 1 851 | 347 | 75,4 % |
| Diciembre | 1 699 | 183 | 84,7 % |

Septiembre concentra la mayor diversidad espacial (995 transiciones únicas), coherente con el paso migratorio Norte→Sur.

## Decisiones de diseño clave

1. **Resolución 0,5°** — compromiso entre granularidad y soporte estadístico. A 0,1° habría >25× más celdas y la inmensa mayoría tendría 0 o 1 transición, haciendo la predicción `argmax` poco informativa. A 1° o más, dos posiciones a 60–80 km caen en la misma celda y se borra la señal de movimiento corto.
2. **Anclaje en `(lon_min, lat_min)` y no en (0, 0)** — todas las celdas tienen `grid_x, grid_y ≥ 0` y los `cell_id` son cadenas legibles. No afecta a las probabilidades (es un cambio de origen), pero simplifica inspección y mapeado.
3. **12 matrices mensuales en lugar de 4 estacionales** — la migración tiene picos muy marcados en abril/mayo (Sur→Norte) y septiembre/octubre (Norte→Sur) que se diluirían al promediar trimestres. La granularidad mensual mantiene esta señal.
4. **Una transición por día** — el dato es ya 1 registro/día/ave (decisión de O1); la matriz hereda esa cadencia.
5. **Formato disperso (long)** — con 1 253 celdas, una matriz densa por mes ocuparía 1 253² ≈ 1,57 M celdas, prácticamente todas a 0. El formato `(cell_id, next_cell_id, prob)` es ~10³–10⁴× más compacto y se filtra trivialmente.
6. **Segmentación por `trayectoria_id`** — heredada de O1, evita transiciones entre días no consecutivos.

## Limitaciones y notas para la memoria

### Dominancia de self-loops
Entre el **64 % y el 90 %** de las transiciones cada mes son `i → i` (el ave permanece en la misma celda al día siguiente). Esto fija un baseline trivial muy alto: predecir siempre "se queda" acierta ~75 % de los días en promedio anual. Cualquier modelo (Markov, ML) debe superar este baseline para ser útil; la métrica importante para la memoria no es la accuracy global sino **la accuracy en días de migración** (transiciones `i → j`, `i ≠ j`).

### Sparsidad por (origen, mes)
Aunque el total de transiciones por mes parece grande, se reparten entre cientos o miles de orígenes. En enero, p. ej., 102 transiciones únicas reparten 1 487 observaciones → muchas celdas tienen una única transición observada con probabilidad 1,0 que es estadísticamente débil. La predicción `argmax` para celdas con poco soporte es ruido.

### Cold start espacial
El 90 % de la rejilla teórica (12 240 − 1 253 = 10 987 celdas) **nunca se visita** en los datos. Si el modelo Markov ve un `cell_id` no presente en el train (ave en zona nueva), no puede predecir nada. Para ML5/ML6 esto se mitiga porque las features incluyen `step_length`, `bearing`, etc.; para Markov puro no hay alternativa.

### Predicciones determinísticas vs. estocásticas
El notebook actual ordena las transiciones por probabilidad descendente pero no implementa una función de predicción explícita. El uso natural es:
- **Top-1 determinístico**: `argmax_j T[m][i, j]`.
- **Muestreo**: `j ~ Categorical(T[m][i, :])` para simular trayectorias.

Cualquiera de los dos cae en self-loop en el 64–90 % de los casos por la distribución empírica.

### Ausencia de regularización
No hay suavizado (Laplace, back-off mensual, etc.). Una transición observada 1 vez tiene la misma probabilidad numérica que una observada 100 veces si vienen del mismo origen y son las únicas. Para la memoria conviene mencionar esto como limitación y posible extensión.

### Las matrices no se persisten
`matrices_mensuales` vive solo en memoria del kernel. Si se quiere usar Markov como input para otro notebook (p. ej. para combinar con HMM o ML), hay que re-ejecutar `markov1.ipynb` o serializarlo (`pickle`/`parquet`). Es una pequeña deuda técnica del pipeline.

## Output (estado actual)

| Estructura | Tipo | Contenido |
|---|---|---|
| `matrices_mensuales` | `dict[int, pd.DataFrame]` | 12 entradas (claves 1–12). Cada DataFrame tiene columnas `cell_id`, `next_cell_id`, `frecuencia`, `probabilidad`, ordenado por `(cell_id, probabilidad desc)`. |

No se escribe ningún archivo CSV/parquet en disco.

## Próximos pasos sugeridos (no implementados)

- Persistir `matrices_mensuales` como `data/processed/markov_matrices.pkl` o `.parquet` para evitar reejecutar el notebook.
- Añadir una función de evaluación: split 80/20 por trayectoria, calcular Top-1 / Top-3 / error en km vs. ML5 como baseline puramente estadístico.
- Reportar accuracy desagregada por estado HMM (migración vs. estacionario) — el verdadero valor del modelo de Markov se mide en días de movimiento, no en días de reposo donde el self-loop trivializa la predicción.
- Comparar resolución 0,25° vs 0,5° vs 1° en el mismo pipeline para justificar la elección actual con datos.
