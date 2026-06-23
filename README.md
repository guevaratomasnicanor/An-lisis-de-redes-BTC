# Detección de Fraude en Bitcoin OTC Web of Trust
### Análisis de Redes + KPIs Conductuales con R e igraph

---

## Resumen

Este proyecto analiza más de **35.000 transacciones** de la plataforma Bitcoin OTC Web of Trust — una red histórica donde los usuarios se califican entre sí para operar cripto P2P — usando **Teoría de Grafos** y **análisis conductual** para detectar patrones de fraude organizado que los sistemas de reputación tradicionales no detectan.

La pregunta central: **¿la reputación acumulada es una garantía real de confianza en mercados descentralizados?**

La respuesta corta: no. Y los datos muestran exactamente por qué.

**Dataset:** [Stanford SNAP — Bitcoin OTC Web of Trust](https://snap.stanford.edu/data/soc-sign-bitcoinotc.html)  
**Herramientas:** R, igraph, ggplot2  
**Enfoque:** Análisis de redes (grafos dirigidos) + KPIs conductuales + etiquetado multicapa

---


## Metodología

### 1. Construcción del Grafo

Se construyó un grafo **dirigido y con pesos** donde:
- Cada **nodo** es un usuario de la plataforma
- Cada **arista** representa una calificación emitida (de -10 a +10)
- El **peso** de la arista es el valor del rating

```
Nodos:   5.881 usuarios
Aristas: 35.592 transacciones
Rango de ratings: -10 a +10
```

### 2. Métricas de Centralidad

Se calcularon cuatro métricas para cada nodo:

| Métrica | Qué mide |
|---|---|
| **In-degree** | Popularidad directa — cuánta gente calificó al usuario |
| **Betweenness** | Intermediación — qué tan "puente" es el usuario entre comunidades |
| **Eigenvector** | Influencia por asociación — importancia de sus conexiones |
| **PageRank** | Reputación y autoridad en la red dirigida |

### 3. Detección de Comunidades

Se aplicó el algoritmo de **Louvain** sobre la versión no dirigida del grafo, identificando 21 comunidades con tamaños que van desde 2 hasta 1.361 usuarios.

### 4. Sistema de Etiquetado Multicapa

En lugar de usar un único umbral de rating para definir "fraudulento" (lo que genera demasiados falsos positivos), se diseñó un sistema de **5 señales independientes**:

| Señal | Criterio | Usuarios detectados |
|---|---|---|
| S1 — Denuncias graves | 3+ ratings ≤ -8 recibidos | ~400 |
| S2 — Asimetría conductual | Promedio dado > 8 AND promedio recibido < 0 | 20 |
| S3 — Anillos de colusión | Pertenece a cluster cerrado de +10 puros (3-10 nodos) | ~150 |
| S4 — Bots / cuentas títere | Variabilidad de voto = 0 AND velocidad > 0.2 tx/día | variable |
| S5 — Pares de inflación mutua | En par de +10 bilateral con al menos 1 denunciado | variable |

**Clasificación final:**
- 0 señales → Normal
- 1 señal → Sospechoso
- 2+ señales → Fraudulento probable
- 3+ señales → Fraudulento confirmado

---

## Hallazgos — Análisis de Redes

### Hallazgo 1: Fragilidad Extrema de la Red

La red de confianza es estructuralmente frágil. Al eliminar únicamente los **5 usuarios con mayor betweenness centrality** (de un total de 5.881), la red se fragmenta de **4 componentes a 432 fragmentos aislados**.

Esto significa que el **0,08% de los usuarios sostiene la conectividad de toda la red**. Si esos intermediarios fallan, son comprometidos, o resultan ser actores maliciosos, la estructura de confianza colapsa.

```
Red original:          4 componentes
Sin los 5 críticos:  432 fragmentos aislados
Factor de fragmentación: 108x
```

Este patrón es consistente con redes scale-free donde unos pocos hubs concentran casi toda la intermediación, haciéndolas muy eficientes pero extremadamente vulnerables a ataques dirigidos.

### Hallazgo 2: Los Hubs son Imanes de Riesgo

El algoritmo de PageRank —equivalente al usado por Google para rankear páginas— identifica a los usuarios más "reputados" de la red. Sin embargo, al analizar el entorno inmediato de los 5 nodos más críticos (por betweenness), surge una paradoja:

| Usuario | Vecinos directos | Fraudulentos en su red |
|---|---|---|
| Usuario 1810 (PR #2) | 440 | 207 |
| Usuario 905 (PR #6) | 321 | 125 |
| Usuario 2642 (PR #1) | 439 | 79 |
| Usuario 1 (PR #8) | 265 | 59 |
| Usuario 35 (PR #1 Betweenness) | 796 | 58 |

**El usuario más reputado de toda la plataforma tiene 207 fraudulentos confirmados en su círculo de contactos directos.** El PageRank no mide honestidad — mide conectividad. Y en una red donde los estafadores se posicionan estratégicamente, estar cerca de un hub famoso es un vector de riesgo, no una garantía.

La correlación de Spearman entre PageRank y cantidad de denuncias es **ρ = 0.34** (p < 0.001) sin controlar por volumen, y **ρ = -0.53** (p < 0.001) controlando por volumen de actividad. Esto confirma que el efecto es un artefacto del volumen: los hubs no son más peligrosos per se, pero concentran más riesgo en su entorno.

### Hallazgo 3: Anillos de Colusión Estructurados

Al construir el subgrafo de solo ratings +10 y analizar sus componentes conexos, se identificaron **31 clusters cerrados de entre 3 y 10 usuarios** que se calificaban mutuamente con +10 exclusivo, recibiendo únicamente denuncias de usuarios externos.

El patrón conductual de estos nodos es consistente con un **ataque Sybil coordinado**: cuentas falsas o coordinadas que se inflan la reputación mutuamente para superar los filtros de la plataforma.

**Cruce de señales:**
- 50% de los usuarios asimétricos (dan +10, reciben negativo) pertenecen a estos anillos
- 10 usuarios acumulan simultáneamente las 3 señales: denuncias graves + asimetría + colusión

### Hallazgo 4: El Intercambio de +10 Mutuo como Firma de Fraude

Se detectaron **220 pares de usuarios** que se intercambiaban calificaciones perfectas (+10) mutuamente. Al cruzar con la lista de fraudulentos confirmados:

```
Pares con intercambio mutuo de +10:              220
Pares con al menos un fraudulento confirmado:     90
Porcentaje:                                      41%
```

**El 41% de los pares que se dan +10 mutuamente tiene al menos un fraudulento confirmado adentro.** El sistema de reputación no solo falla en detectar el comportamiento colusivo — lo premia activamente, ya que el intercambio bilateral de ratings positivos eleva el score de ambos participantes.

---

## Hallazgos — KPIs Conductuales

Se calcularon 9 KPIs para cada usuario y se compararon entre el grupo Fraudulento y el grupo Normal. Los resultados ordenados por diferencia porcentual:

| KPI | Fraudulento | Normal | Diferencia % | Dirección |
|---|---|---|---|---|
| Ratings positivos/día | 0.417 | 0.014 | **+2.879%** | Fraude mayor |
| Tasa de polarización | 0.639 | 0.043 | **+1.386%** | Fraude mayor |
| Reciprocidad selectiva | 0.213 | 0.020 | **+965%** | Fraude mayor |
| Ratio de destrucción | 0.473 | 0.049 | **+865%** | Fraude mayor |
| Promedio recibido | -2.913 | 0.931 | **-413%** | Normal mayor |
| Variabilidad del voto | 2.966 | 0.807 | **+268%** | Fraude mayor |
| Días entre votos | 2.223 | 18.079 | **-88%** | Normal mayor |
| Promedio emitido | 0.751 | 1.384 | **-46%** | Normal mayor |


### KPI 1: Velocidad de Construcción de Reputación (+2.879%)

Los fraudulentos acumulan ratings positivos **29 veces más rápido** que los usuarios normales. Un usuario legítimo tarda meses o años en construir reputación mediante interacciones orgánicas. Un estafador la fabrica en semanas mediante intercambios coordinados.

Esta métrica es el KPI más discriminante del análisis y no requiere información estructural de la red para calcularse — es detectable con SQL básico.

### KPI 2: Tasa de Polarización del Voto (+1.386%)

Los fraudulentos emiten votos extremos (+9, +10, -9, -10) en una proporción **14 veces mayor** que los usuarios normales. Los usuarios legítimos utilizan todo el rango de la escala (-10 a +10) con distribución relativamente uniforme. Los fraudulentos casi exclusivamente votan en los extremos.

Esto refleja dos comportamientos simultáneos: inflar la reputación de aliados con +10 sistemáticos y destruir la de competidores o víctimas que los denunciaron con -10 coordinados.

### KPI 3: Reciprocidad Selectiva (+965%)

Los fraudulentos devuelven +10 a quienes les dieron +10 a una tasa **10 veces mayor** que los usuarios normales. Este patrón de reciprocidad perfecta es la firma matemática de los anillos de colusión: el voto no refleja una evaluación genuina de la transacción sino una respuesta automática al estímulo recibido.

### KPI 4: Ratio de Destrucción (+865%)

La proporción de ratings recibidos que son -10 puro es **9 veces mayor** en fraudulentos que en normales. Esto confirma que las denuncias graves no son accidentales ni disputas puntuales — son el resultado de un patrón de comportamiento sistemático que múltiples usuarios independientes identificaron y reportaron.

### KPI 5: Variabilidad del Voto (+268%)

Paradójicamente, los fraudulentos tienen **mayor variabilidad** en sus votos emitidos que los normales, no menor. La explicación: votan +10 perfecto a su red de aliados y -10 puro a sus víctimas o competidores, generando alta varianza global pero baja varianza dentro de cada subgrupo. Los usuarios normales usan calificaciones intermedias con más frecuencia, generando menor varianza extrema.

### KPI 6: Velocidad de Votación (-88% en días entre votos)

Los fraudulentos votan cada **2.2 días** en promedio; los normales cada **18 días**. La actividad de votación en usuarios fraudulentos es 8 veces más frecuente, consistente con campañas coordinadas de inflación o ataque que ocurren en ráfagas temporales cortas (burstiness).

### KPI Descartado: Índice de Aislamiento Comunitario (+3.5%)

Contrariamente a la hipótesis inicial, el índice de aislamiento comunitario (proporción de transacciones dentro de la propia comunidad Louvain) no discrimina entre fraudulentos y normales. Ambos grupos tienen índices similares (~0.68), sugiriendo que los fraudulentos no operan exclusivamente dentro de comunidades cerradas sino que se integran activamente en la red general para maximizar su alcance.

---

## Limitaciones y Consideraciones

**Sobre la definición de fraude:** No existe una etiqueta ground-truth en el dataset. La clasificación multicapa es una aproximación basada en señales observables, no una verdad absoluta. Un usuario con 3+ denuncias graves puede ser víctima de una campaña de desprestigio coordinada.

**Sobre la correlación causal:** Los KPIs muestran diferencias estadísticamente significativas entre grupos, pero no implican causalidad. La velocidad de acumulación de reputación puede ser alta en usuarios legítimos muy activos.

**Sobre la temporalidad:** El dataset es histórico (2010-2016). Los patrones de fraude evolucionan y las técnicas detectadas aquí pueden no ser representativas de plataformas actuales.

**Sobre el índice de aislamiento:** La hipótesis de que los fraudulentos operan en comunidades cerradas no se confirmó con los datos. Esto es un hallazgo en sí mismo: los estafadores sofisticados se integran en la red general, no se aíslan.

---

## Conclusiones

Los sistemas de reputación basados en promedios de calificación son insuficientes para detectar fraude organizado por tres razones estructurales:

**1. No capturan la velocidad.** Un score de 9.8 sobre 10 no dice nada sobre si esa reputación se construyó en 3 años de interacciones orgánicas o en 3 semanas de intercambios coordinados.

**2. No capturan la estructura.** Un usuario puede tener reputación impecable y estar rodeado de 200 estafadores confirmados. El score individual no refleja el riesgo sistémico del entorno.

**3. No capturan el comportamiento.** La polarización del voto, la reciprocidad selectiva y la velocidad de votación son señales conductuales que los promedios ocultan pero que discriminan con diferencias de hasta 2.879% entre grupos.

El modelo de scoring híbrido desarrollado —combinando señales de red (betweenness, PageRank, clustering) con KPIs conductuales (polarización, reciprocidad, velocidad)— identificó **usuarios de riesgo que los filtros estándar de la plataforma ignoraban por completo** al tener promedios de calificación aparentemente positivos.

---

## Reproducibilidad

```r
# Instalar dependencias
install.packages(c("igraph", "ggplot2"))

# El script descarga el dataset automáticamente desde Stanford SNAP
source("bitcoin_otc_analisis.R")
```

El script completo es autocontenido: descarga los datos, ejecuta el análisis completo y genera las 6 visualizaciones en el directorio de trabajo.

---

## Contacto

Si tenés preguntas sobre la metodología o querés discutir aplicaciones de este enfoque a otros datasets, podés encontrarme en LinkedIn.
