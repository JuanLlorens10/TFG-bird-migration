# O5 — Módulo de visualización

> Referencia específica para el objetivo O5 del TFG.
> Actualizar tras cada sesión de trabajo en este objetivo.

## Objetivo

Crear un módulo de visualización con mapas interactivos que comparen la ruta real del ave con la predicción de los modelos (Markov y ML), y analicen el error en km.

## Estado actual

**Implementado en `notebooks/VIZ.ipynb` el 2026-05-09**, tomando como referencia el modelo Random Forest de `ML3.ipynb` sobre `hmm5.csv` (versión canónica HMM del TFG).

### Pipeline (`VIZ.ipynb`)

1. **Reentrenamiento de ML3** sobre `hmm5.csv` (mismo split por animal 80/20, mismas 9 features). Obtiene **Top-1 = 81,15 %**, idéntico al `ML3.ipynb` ejecutado el 2026-05-09 → garantiza que la visualización refleja el modelo canónico.
2. **Reconstrucción del centroide geográfico** de cada celda predicha invirtiendo el grid 0,5°: `lat = lat_min + (gy + 0.5) · 0.5`, `lon = lon_min + (gx + 0.5) · 0.5`.
3. **Cálculo del error** vía haversine entre el centroide predicho y la posición real del día siguiente (`next_lat`, `next_lon`).
4. **Resúmenes** global, por estado HMM y por mes.
5. **Mapas Folium** por animal y heatmap global; **gráficos Plotly** de distribución del error.

### Resultados — error geográfico (Random Forest, ML3 sobre hmm5.csv)

| Subconjunto | n | Top-1 | Mediana | Media | P90 | ≤50 km | ≤100 km |
|---|---:|---:|---:|---:|---:|---:|---:|
| **Global** | 3 782 | 81,15 % | 23,9 km | 48,5 km | 40,5 km | 91,8 % | 94,7 % |
| Estacionario | 3 448 | 86,43 % | 23,3 km | 39,4 km | 34,2 km | 96,3 % | 97,3 % |
| Migración | 334 | 26,65 % | 56,2 km | 142,8 km | 340,7 km | 44,9 % | 67,7 % |

> Notas de lectura:
> - El **error mínimo posible** del esquema (acertar la celda) es del orden del semilado de la celda 0,5° — entre ~25 km (cerca del ecuador) y ~17 km (lat 60°). Los 23 km de mediana del estacionario están en ese suelo: el modelo no puede afinar más sin cambiar la rejilla.
> - Que `p90 < media` en el global y el estacionario refleja la cola larga y delgada de fallos migratorios: ~9 % de los puntos suben mucho la media, pero el percentil 90 sigue dominado por la masa estacionaria.
> - La métrica de migración (mediana 56 km, P90 340 km) confirma cuantitativamente la conclusión de O4: sin viento, los saltos largos no se predicen — el ave puede aterrizar en cualquiera de ~50 celdas alcanzables.

### Análisis mensual del error (medianas)

Errores estables (~20–28 km) entre enero y diciembre. Los meses con peor accuracy Top-1 son **abril** (0,63) y la franja **agosto–noviembre** (0,75–0,77), coincidiendo con la **migración prenupcial y postnupcial**. La mediana en km cambia poco porque está dominada por días estacionarios; lo que sube es la cola: P90 dispara en septiembre (145 km) y noviembre (80 km).

### Visualizaciones generadas (`img/o5/`)

| Archivo | Descripción |
|---|---|
| `mapa_91916A.html` | Animal con más test (n=379, 35 días de migración). Top-1 90,5 %, mediana 22,8 km. |
| `mapa_91732A.html` | Animal con migración intensa (n=78, 27 mig). Top-1 56,4 %. |
| `mapa_91803A.html` | Caso extremo: 22/22 días de migración en test. Top-1 36,4 % — ilustra el techo del modelo. |
| `mapa_global_error.html` | Heatmap global del error (cap 300 km) + 800 marcadores aciertos/fallos coloreados por estado. |
| `error_histograma.html` | Distribución del error por estado HMM (Plotly). |
| `error_cdf.html` | CDF del error por estado HMM. |
| `error_mensual.html` | Boxplot mensual del error por estado HMM. |

Cada mapa Folium incluye:
- **Ruta real completa** (gris fino, contexto train + test).
- **Tramo test real** coloreado por estado HMM (rojo migración, azul estacionario).
- **Trayectoria predicha** (verde, línea discontinua).
- **Marcadores diarios** con tooltip detallado (fecha, estado, celda real vs. predicha, error en km, ✓/✗).
- **Líneas de error** (gris) que conectan posición real y predicha cada día.
- **Control de capas** para activar/desactivar cada componente.

## Diseño previsto (estado original)

- Mapas interactivos con Plotly/Folium: trayectoria real vs. predicha sobre mapa geográfico. ✓
- Coloreado por estado HMM (migración / estacionario) para correlacionar comportamiento con precisión de la predicción. ✓
- Análisis del error en km por día, estado y mes. ✓

## Conclusiones

- **Estado estacionario**: el modelo predice la celda exacta el 86 % de los días; cuando falla, el centroide vecino queda a ~23 km de la posición real. La línea verde (predicha) y la negra (real) se solapan visualmente en los mapas → el módulo de visualización **valida** que el modelo es útil para la fase de invernada/cría.
- **Estado de migración**: el mapa muestra divergencias claras entre real y predicha en los corredores migratorios. Es la confirmación visual del techo identificado en O4: el modelo "predice" que el ave seguirá donde está y la línea verde se queda anclada mientras la negra avanza.
- **Geografía del error**: el heatmap global concentra los fallos grandes en los corredores migratorios (golfo de Vizcaya, costa atlántica de Marruecos, Sahel) más que en las zonas de invernada — coherente con la naturaleza del problema.
- El módulo permite **inspeccionar caso a caso** mediante los tooltips de cada día → herramienta diagnóstica útil para la memoria del TFG y para identificar visualmente patrones de error sistemático.

## Pendiente / posibles extensiones

- Añadir capa con la **predicción del modelo Markov** (`markov1.ipynb`) para comparar Markov vs. RF en el mismo mapa.
- Versión que use el **modelo separado por estado de ML5** y la mejor configuración cuando ML5 se reejecute sobre `hmm5.csv`.
- Exportar métricas a `img/o5/resumen.csv` para enlazar directamente desde la memoria.
