📊 Resumen del Proyecto
Este proyecto analiza la red de confianza de Bitcoin OTC, una plataforma P2P donde los usuarios se califican entre sí para mitigar el riesgo de contraparte. Utilizando Teoría de Grafos y Análisis Estadístico en R, el objetivo fue identificar estructuras de colusión, usuarios de alto riesgo y la resiliencia de la confianza en un entorno descentralizado.

🛠️ El Stack Tecnológico
Lenguaje: R (Base)

Librerías principales: igraph para procesamiento de redes y gzcon para ingesta eficiente de datos comprimidos.

Dataset: SNAP Bitcoin OTC (~35,000 transacciones).

📈 Proceso de Análisis
Ingesta y Limpieza: Procesamiento de flujos de datos binarios y conversión de tipos (Unix Timestamps a POSIXct).

Modelado de Red: Construcción de un grafo dirigido pesado donde los nodos son usuarios y las aristas son calificaciones (-10 a +10).

Cálculo de Centralidad: Implementación de PageRank, Betweenness y Eigenvector Centrality para identificar nodos críticos.

Detección de Anomalías: Identificación de patrones de asimetría de ratings y anillos de colusión.

Simulación de Ataque: Evaluación de la conectividad de la red ante la remoción de nodos estratégicos.

💡 Insights Principales (Hallazgos Clave)
1. La Paradoja de la Autoridad (PageRank vs. Fraude)
Se descubrió que la popularidad no es sinónimo de honestidad.

Hallazgo: Usuarios con el Top 1% de PageRank (máxima autoridad) acumulaban más de 30 denuncias por fraude.

Conclusión: Los algoritmos de autoridad pueden ser manipulados por redes coordinadas; el análisis de riesgo debe ser multimodal.

2. Fragilidad Estructural: "Los Guardianes"
La red es extremadamente dependiente de un puñado de nodos.

Evidencia: Al eliminar solo a los 5 usuarios con mayor Betweenness, la red se fragmentó de 4 componentes a 432 grupos aislados (un incremento del 10,700% en la desconexión).

Impacto: El ecosistema de confianza es vulnerable a "puntos de falla únicos".

3. Validación Cruzada de Riesgo (Convergencia del 50%)
Utilicé dos modelos independientes para detectar actores maliciosos:

Modelo A (Comportamiento): Usuarios con asimetría crítica (dan ratings +10 pero reciben -10).

Modelo B (Estructura): Usuarios en anillos de colusión (grupos cerrados de 3-10 personas).

Resultado: El 50% de los usuarios asimétricos pertenecían a anillos de colusión, validando que el fraude en esta red es una actividad coordinada.

