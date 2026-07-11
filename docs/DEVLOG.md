# CoolRefugeEquity — Bitácora técnica

Registro de decisiones de diseño, metodología e incidencias resueltas durante el desarrollo. Sirve como base para el README final y el notebook reproducible.

**Estado actual del proyecto:** Componentes 1-5 cerrados para la comparación **Curicó – Valdivia**. Pendiente: carrusel LinkedIn.

---

## 1. Objetivo del proyecto

Medir equidad de acceso a "zonas de refresco" (alivio térmico real, no solo cobertura verde cosmética) en dos ciudades chilenas, comparando 2016 vs 2026. Pregunta central: ¿qué proporción de la población vive en las zonas urbanas térmicamente más desfavorecidas de su propia ciudad, y cómo cambió esa relación entre 2016 y 2026?

## 2. Métrica central

**Índice de Refresco** por hexágono, por ciudad-año:

```
Índice de Refresco = z(NDVI) − z(LST)
```

Ambos componentes estandarizados (z-score) **dentro de cada ciudad-año por separado**. Decisión deliberada: las dos ciudades comparadas tienen climas de base distintos, así que comparar valores absolutos entre ciudades sería injusto. Estandarizar intra-ciudad-año permite comparar *posición relativa* de cada hexágono respecto a su propio contexto urbano.

## 3. Unidad espacial

Grilla hexagonal H3, resolución 9 (~174 m de lado, ~0.105 km² área de referencia global). Sin relación de escala con los hexágonos de 10.000 ha de la tesis doctoral (bosque boreal, MILP) — mismo concepto de grilla, escala completamente distinta.

---

## 4. Componente 1 — Grilla H3 sobre la ciudad

**Input:** GHS-UCDB R2024A (delineación fija epoch 2025), descarga regional "Latin America and the Caribbean", capa `GHSL_UCDB_THEME_GENERAL_CHARACTERISTICS_GLOBE_R2024A`.

**Selección de ciudad:** por contención geométrica (punto centroide conocido dentro del polígono UCDB), no por coincidencia de texto en campos de nombre.

**Resultado final:**

| Ciudad | Hexágonos | Área media (km²) | Std |
|---|---|---|---|
| Curicó | 251 | 0.0886 | 0.000022 |
| Valdivia | 314 | 0.0916 | 0.000032 |

**Nota metodológica:** H3 no es estrictamente equiárea — el área real de una celda varía con la latitud. Validado cruzando área en EPSG:32718 contra `h3.cell_area()` nativo.

**Decisión de diseño:** hexágonos de borde se conservan completos, sin recorte geométrico al polígono UCDB.

## 5. Componente 2 — NDVI + LST por ciudad-año

**Fuente:** Landsat 8/9 Collection 2 Level 2, banda `ST_B10` para LST, ventana estacional enero-marzo. L8 solo para 2016; L8+L9 para 2026.

**AOI:** bounding box de la malla H3 por ciudad + buffer 500 m.

**Máscara de nubes:** bits estándar de `QA_PIXEL`, composite de mediana.

### Incidencia resuelta: nodata no propagado en la exportación

Primera exportación no definía `nodata` explícito → `rasterstats` contaba píxeles enmascarados por nubes como válidos. **Fix:** `composite.unmask(NODATA_VALUE)` antes de exportar (`-9999.0`), y `nodata=NODATA_VALUE` explícito en `zonal_stats`.

## 6. Componente 3 — Agregación a hexágonos + Índice de Refresco

**Agregación zonal:** media de NDVI y LST por hexágono, umbral de cobertura válida 50%, aplicado por banda de forma independiente.

### Incidencia resuelta: umbral de cobertura calculado sobre la banda equivocada

El código original calculaba `valid_fraction` usando solo el conteo de píxeles NDVI y lo aplicaba también como criterio de validez para LST. **Fix:** `valid_fraction_ndvi` y `valid_fraction_lst` calculados y aplicados por separado.

**Estandarización:** `z = (x − media_ciudad_año) / std_ciudad_año`, agrupado por (`city`, `year`). Verificado: media ~0, std = 1.0 exacto en las 8 combinaciones (2 ciudades × 2 años × 2 variables).

**Cobertura final:** 0 hexágonos sin NDVI válido en ambas ciudades. 32 hexágonos sin LST válido — enteramente en Valdivia (16 en 2016, 16 en 2026; ~5% del total de esa ciudad), consistente con nubes puntuales normales de un composite de mediana trimestral. Curicó: 0 hexágonos sin LST válido en ambos años, coherente con el diagnóstico de cobertura ST que motivó elegirla (ver sección 9).

## 7. Componente 4 — Población (WorldPop)

**Fuente:** `ee.ImageCollection("projects/sat-io/open-datasets/WORLDPOP/pop")`, catálogo comunitario `sat-io` (Samapriya Roy), cobertura 2015-2030 anual, modelo **constrained**.

**Incidencia evitada:** la propiedad `year` de la colección es **string**, no entero — verificado antes de escribir el filtro.

**Agregación:** suma ponderada por área (`ee.Reducer.sum()`, no media — población es aditiva).

**Validación de orden de magnitud** (WorldPop constrained vs UCDB `GC_POP_TOT_2025` unconstrained):

| Ciudad | WorldPop 2016 | WorldPop 2026 | UCDB (referencia 2025) | Diferencia |
|---|---|---|---|---|
| Curicó | 87.331 | 95.418 | 103.447 | ~-16% |
| Valdivia | 120.024 | 131.270 | 148.826 | ~-16% |

Diferencia sistemática, mismo signo y magnitud similar en ambas ciudades — consistente con la explicación metodológica (constrained excluye zonas no edificadas que unconstrained sí cuenta), no con un error de escala o filtro.

### Limitaciones a declarar en el README

1. **2026 es proyección**, no censo real.
2. **Fuente comunitaria, no oficial de Google**.
3. **Modelo constrained** — totales sistemáticamente más bajos que fuentes unconstrained. No afecta el análisis de equidad (usa distribución relativa, no totales absolutos).

## 8. Componente 5 — Métrica de equidad (resultado final)

**Definición:** cuartil de `indice_refresco` intra-ciudad-año (`pd.qcut`, 4 grupos). Peor cuartil = 25% de hexágonos con `indice_refresco` más bajo. Métrica de equidad = % de población de la ciudad-año que vive en ese peor cuartil.

**Referencia de interpretación:** si la población estuviera distribuida de forma pareja entre cuartiles térmicos, el peor cuartil concentraría ~25% de la población por construcción. Desviaciones indican concentración (>25%) o dispersión protectora (<25%).

**Resultado final:**

| Ciudad | 2016 | 2026 | Δ 2016→2026 |
|---|---|---|---|
| Curicó | 26.2% | 28.2% | +1.9 |
| Valdivia | 42.4% | 42.0% | -0.5 |

**Validación visual (QGIS, ambas ciudades, ambos años):** confirmado que el peor cuartil no está siendo arrastrado por hexágonos aislados de alta población en ninguna de las dos ciudades. En Valdivia, el cuartil 1 forma una mancha contigua que coincide con el casco urbano denso. En Curicó, el resultado cercano al neutral (26-28%) se confirmó visualmente como consistente, no como ruido estadístico disfrazado.

**Lectura del hallazgo:** Curicó está apenas por encima del valor neutral, con leve tendencia al alza (empeora levemente 2016→2026). Valdivia se mantiene muy por encima del neutral en ambos años, de forma estable — casi el doble de lo esperable por azar. La comparación entre ambas ciudades es el contraste central del proyecto: una ciudad con exposición térmica repartida de forma razonablemente pareja, frente a otra donde casi la mitad de la población carga con el peso del peor cuarto del territorio urbano, de forma sostenida en el tiempo.

**Matiz a no omitir:** el hallazgo no compara temperatura absoluta entre ciudades (la estandarización intra-ciudad-año lo impide por diseño) — compara qué tan desigual es la distribución de exposición térmica *dentro* de cada ciudad respecto a sí misma.

---

## 9. Incidencia mayor: cobertura de datos ST insuficiente en Concepción — pivote a Curicó

### Síntoma

Al aplicar el fix del Componente 3 (umbral por banda separado), apareció que ~50% de los hexágonos de Concepción quedaban sin `lst_mean` válido.

### Diagnóstico

1. **Descartado — máscara de nubes:** 4 variantes de `QA_PIXEL` probadas (todos los bits, sin cloud shadow, solo cloud, sin máscara). Las 4 dieron el mismo resultado (0.613 de cobertura).
2. **Descartado — falta de pasadas satelitales:** 12 imágenes, 6 fechas, 2 combinaciones PATH/ROW, footprint = 100% del AOI.
3. **Descartado — ventana estacional corta:** ampliar de enero-marzo a diciembre-abril (+67% de escenas, 12→20) no cambió la cobertura (0.613 → 0.614).
4. **Conclusión:** pérdida intrínseca al algoritmo de recuperación de `ST_B10` para esa zona geográfica — requiere datos auxiliares (reanálisis atmosférico, emisividad) por píxel que fallan de forma sistemática ahí, independiente de nubes o cantidad de pasadas. Probable relación con la topografía compleja de Concepción (cerros, quebradas, estuario), sin confirmar la causa exacta a nivel de algoritmo.

### Decisión

Chequeo rápido de cobertura ST en Curicó (mismo diagnóstico, sin construir pipeline completo): 100% de cobertura válida con 5 escenas. Se reemplazó Concepción por Curicó, manteniendo Valdivia sin cambios.

**Lección metodológica para el README:** antes de comprometerse a una ciudad en un proyecto de LST con Landsat, correr el diagnóstico de cobertura ST como paso de validación previa — buena cobertura óptica (NDVI, RGB) no garantiza buena cobertura térmica.

**Concepción queda documentada como explorada y descartada** — candidato a retomar en un proyecto futuro con otra fuente térmica (MODIS LST, ECOSTRESS).

---

## 10. Pendiente

- Carrusel LinkedIn (216×270mm, 300dpi, CartoDB Positron No Labels, export PNG→PDF vía ImageMagick).
- `git config` local: autor de commits configurado automáticamente por el sistema. No bloqueante mientras el trabajo sea local; corregir antes de conectar el repo a GitHub.
- Notebook reproducible final: consolidar todas las celdas en un único `.ipynb` limpio, con las celdas de exploración fallida (Concepción) movidas a un apéndice o notebook separado.
- README.md del repositorio (estructura completa: problema, metodología, datos, instalación, uso, resultados, limitaciones, mejoras futuras) — esta bitácora es el insumo base.
