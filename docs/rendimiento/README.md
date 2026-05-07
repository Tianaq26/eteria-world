# ⚙️ Optimización de Rendimiento — Eteria World

> Este documento detalla las técnicas de optimización aplicadas en Eteria World para mantener **30 FPS estables en hardware gama media-baja** con un mundo abierto de 5 regiones, más de 100 modelos 3D, vegetación densa, IA activa y shaders dinámicos.

---

## Resultado general

| Métrica | Antes | Después |
|---------|-------|---------|
| Draw calls | ~5.000 | ~1.000 |
| FPS (gama baja, calidad baja) | Inestable | 20 FPS estables |
| FPS (gama media-baja, calidad media) | Inestable | 30 FPS estables |
| Triángulos en aldea principal | ~10.000.000 | ~1.000.000 |

La optimización no fue un paso final sino una práctica continua durante todo el desarrollo. Cada sistema fue diseñado considerando su costo de renderizado y lógica desde el inicio.

---

## Herramientas de análisis utilizadas

### Frame Debugger
Usado para inspeccionar el flujo exacto de dibujado de la GPU en cada frame, hilo por hilo. Permitió identificar qué objetos generaban draw calls innecesarios, qué shaders se ejecutaban más veces de lo esperado y en qué orden se procesaban las capas de renderizado.

> 📸 *[Insertar captura del Frame Debugger mostrando el flujo de dibujado]*

### Unity Profiler
Usado para analizar el consumo de CPU y GPU por frame, identificando loops costosos en scripts de IA, actualizaciones innecesarias en el hilo principal y picos de memoria. Fue clave para descubrir que el editor de Unity en sí consumía una cantidad significativa de recursos, lo que distorsionaba las métricas reales del juego.

> 📸 *[Insertar captura del Profiler con distribución de tiempo por sistema]*

### Stats Window
Monitoreo en tiempo real de triángulos, draw calls, vértices y uso de memoria directamente desde la ventana de juego. Usado como referencia rápida durante el desarrollo para detectar picos de rendimiento al agregar nuevos elementos al mundo.

> 📸 *[Insertar captura de Stats Window con métricas en escena abierta]*

---

## Técnicas aplicadas

### 1. Reducción de Draw Calls — de 5.000 a 1.000

Los draw calls son instrucciones individuales que la CPU envía a la GPU para dibujar geometría. Cada objeto con su propio material genera al menos un draw call. Con un mundo de esta escala, esto escala rápidamente.

Se aplicaron dos métodos combinados:

**Mesh Combine (combinación de mallas)**
Las cabañas, decoración y elementos estáticos de cada aldea fueron combinados en una sola malla por zona. En lugar de que cada silla, mesa o pared genere su propio draw call, todo el conjunto se envía a la GPU en una sola instrucción.

```
Antes: 1 cabaña con 8 elementos = 8 draw calls
Después: zona completa de aldea combinada = 1–3 draw calls
```

**GPU Instancing en materiales**
Para elementos que se repiten muchas veces (rocas, árboles, hongos, postes), se activó GPU Instancing en los materiales. Esto permite que la GPU dibuje cientos de copias del mismo mesh en una sola pasada, siempre que compartan el mismo material. Los shaders propios del juego fueron escritos con compatibilidad con instancing desde el inicio.

```
Sin instancing: 300 rocas = 300 draw calls
Con instancing: 300 rocas = 1–2 draw calls
```

> 📸 *[Insertar comparativa de Stats Window antes y después de aplicar estas técnicas]*

---

### 2. LOD — Level of Detail

El sistema LOD reduce automáticamente la complejidad de los modelos según su distancia a la cámara. En Eteria World se implementó en dos niveles: visual y lógico.

**LOD visual (modelos 3D)**
Cada modelo tiene entre 2 y 3 versiones con diferente cantidad de polígonos. Unity selecciona automáticamente cuál renderizar según la distancia al jugador. Se aplicó a:
- Todos los modelos de decoración (rocas, muebles, estructuras)
- Vegetación (árboles, hongos)
- Puentes y estructuras grandes
- NPCs

**LOD lógico (IA de NPCs)**
Más allá del modelo visual, la lógica de IA también se desactiva por distancia. Un NPC fuera del rango de visión del jugador no necesita ejecutar pathfinding, detección ni actualizaciones de estado. Esto libera CPU significativamente en escenas con muchos NPCs activos.

```
Distancia corta  → modelo HD + IA completa (patrullaje, detección, animación)
Distancia media  → modelo MD + IA reducida (solo animación idle)
Distancia larga  → modelo LD + IA desactivada (objeto estático sin animacion ni movimiento)
```

> 📸 *[Insertar captura mostrando LOD Groups configurados en Inspector]*

---

### 3. Occlusion Culling

El Occlusion Culling evita que la GPU renderice objetos que están fuera del campo de visión de la cámara o bloqueados por geometría. Se configuró en prácticamente todo el mundo: estructuras, vegetación, decoración, NPCs y terreno.

El proceso requiere un bake previo donde Unity calcula qué objetos se bloquean entre sí desde distintos ángulos. Después de cada modificación grande del terreno fue necesario recalcular este bake para mantenerlo vigente.

**Impacto especialmente notable en:**
- Zonas de aldea con edificios densos (cada edificio oculta lo que hay detrás)
- Zonas de montaña donde el terreno oculta valles completos
- Interiores de estructuras

> 📸 *[Insertar captura de la ventana de Occlusion Culling con zonas configuradas]*

---

### 4. Bake de Iluminación

La luz direccional principal (sol/luna del ciclo día-noche) tiene su contribución a superficies estáticas precalculada en un lightmap. Esto elimina el costo de calcular iluminación en tiempo real para todos los objetos estáticos del mundo.

Solo los objetos dinámicos (jugador, dragones, NPCs, partículas) reciben iluminación en tiempo real. El resto del mundo usa los lightmaps precalculados, que son simplemente texturas que Unity aplica sin costo de cómputo adicional.

```
Objetos estáticos → lightmap precalculado (costo: 0 en tiempo real)
Objetos dinámicos → iluminación en tiempo real (costo normal)
```

> ⚠️ Cada vez que se modificó el terreno o se movieron estructuras estáticas fue necesario hacer un nuevo bake. En un mundo de este tamaño, ese proceso puede tomar varios minutos.

---

### 5. Árboles renderizados desde GPU

Los árboles del mundo no son GameObjects convencionales. Se renderizan directamente desde datos procesados en la GPU mediante el sistema de Terrain Tools de Unity, similar al sistema de césped. Esto permite tener miles de árboles distribuidos por el mundo con un impacto mínimo en rendimiento, ya que no generan draw calls individuales ni tienen componentes de Unity activos ademas de sistema LOD especial con Bilboards que son el ultimo nivel LOD este es una imagen en vez de un modelo a una gran distancia no hay diferencia visual pero si de rendimiento.

```
Árboled → renderizado GPU, sin GameObject, costo mínimo
```

---

### 6. Shader Reactivo Multi-textura

En lugar de crear un material diferente por cada variación visual de un objeto, se desarrolló un shader propio que permite mezclar múltiples texturas desde el mismo material usando parámetros configurables (blend, opacidad, canal alfa).

Esto reduce drásticamente la cantidad de materiales únicos en escena, lo que directamente reduce draw calls y el tiempo de cambio de estado de la GPU entre objetos.

```
Sin shader reactivo: 1 objeto con 3 variaciones = 3 materiales = 3+ draw calls
Con shader reactivo: 1 objeto con 3 variaciones = 1 material = 1 draw call
```

---

### 7. Limpieza de Assets y Shaders

Durante el análisis con el Frame Debugger se identificó que muchos shaders y assets de versiones anteriores del proyecto seguían cargados en memoria sin ser usados. Se realizó una limpieza completa eliminando:
- Shaders obsoletos de etapas tempranas del desarrollo
- Materiales duplicados sin referencia activa
- Prefabs de versiones descartadas de modelos

Esta limpieza redujo el tiempo de carga y el uso de memoria sin cambiar ninguna funcionalidad visible.

---

### 8. Calidad de Gráficos Configurable

Se implementó un sistema de calidad de gráficos con múltiples niveles configurables desde el menú del juego. Cada nivel ajusta resolución de sombras, distancia de renderizado, densidad de vegetación, distancia de los sistemas LOD y calidad de shaders.

| Nivel | FPS aproximados |
|-------|----------------|
| Bajo | ~60 FPS (gama media) |
| Medio | ~50 FPS (gama media) |
| Alto | ~40 FPS (game media) |

---

## Lecciones aprendidas

**El editor de Unity consume recursos propios.** Los FPS medidos desde el editor son significativamente menores que en una build real. Las métricas finales deben medirse siempre en una build de desarrollo, no desde el editor.

**La optimización de lógica importa tanto como la visual.** Reducir polígonos ayuda, pero desactivar scripts de IA innecesarios a distancia tuvo un impacto igual o mayor en la estabilidad de los FPS.

**Los draw calls escalan más rápido de lo esperado.** Un mundo que se ve "simple" puede tener miles de draw calls si cada elemento decorativo usa su propio material. Planificar el sistema de materiales desde el inicio habría ahorrado tiempo de refactorización.

**El bake de iluminación requiere disciplina.** En un mundo que cambia constantemente, mantener los lightmaps actualizados es un costo operativo real que hay que planificar.

---

> 📸 *Todas las capturas referenciadas en este documento deben colocarse en `docs/assets/rendimiento/` y actualizar las rutas correspondientes.*
