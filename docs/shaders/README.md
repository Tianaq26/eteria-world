# 🌊 Shaders — Eteria World

> Todos los shaders del juego fueron construidos en **Unity Shader Graph** (URP) sin código HLSL externo. Cada uno resuelve un problema visual específico con parámetros configurables desde el Inspector, permitiendo reutilizar el mismo shader en múltiples materiales con resultados distintos.

---

## Índice

1. [Shader Reactivo](#1-shader-reactivo)
2. [Dragones](#2-dragones)
3. [Rio Water](#3-rio-water)
4. [Catarata](#4-catarata)
5. [Water Shader (Mar)](#5-water-shader-mar)
6. [Rayo VFX](#6-rayo-vfx)
7. [ShaderNPC](#7-shadernpc)

---

## 1. Shader Reactivo

**Archivo:** `shader_reactivo.shadergraph`
**Tipo:** Lit · Opaque · Metallic Workflow

### Propósito
Shader principal del proyecto. Diseñado para ser el material base de la mayoría de objetos del mundo (terreno, rocas, cabañas, modelos de entorno) sin necesidad de crear un shader distinto por cada variación visual. Permite mezclar texturas, colores y efectos desde parámetros del material.

### Parámetros expuestos

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `base` | Texture2D | Textura principal del objeto |
| `scala` | Float | Escala de tiling de la textura base |
| `normal map` | Texture2D | Normal map para iluminación con detalle |
| `brillo` | Float | Control de smoothness (reflectividad) |
| `ambient` | Float | Nivel de luz ambiental base |
| `color` | Color | Tinte de color aplicado sobre la textura |
| `color / textura` | Texture2D | Segunda textura opcional para blend |
| `mix_color` | Float | Nivel de mezcla entre color sólido y textura secundaria |
| `fusion_mixcolor` | Float | Control fino de fusión entre capas de color |
| `Mix_texture_activo` | Boolean | Activa o desactiva la segunda textura |
| `mix_texture_input` | Float | Opacidad de la textura secundaria (usa canal alfa) |
| `scala separada` | Float | Escala independiente para la textura secundaria |
| `separar_scala` | Boolean | Permite usar escala distinta para cada textura |
| `normal activo` | Boolean | Activa o desactiva el normal map |
| `ambient oclusion active` | Boolean | Activa ambient occlusion |
| `ambient oclusion level` | Float | Intensidad del ambient occlusion |
| `nivel de brillo` | Float | Ajuste fino de brillo independiente del smoothness |

### Cómo funciona

El grafo toma la textura base y le aplica el tinte de color. Si `Mix_texture_activo` está activado, mezcla una segunda textura usando el canal alfa como máscara de control, permitiendo efectos como desgaste, variación de suelo o manchas de humedad. El normal map se aplica de forma opcional para no penalizar rendimiento en objetos distantes. El ambient occlusion agrega profundidad en esquinas y cavidades sin costo de raytracing.

```
Textura base + escala
    → Tinte de color
        → [Opcional] Blend con textura secundaria por canal alfa
            → Normal map [Opcional]
                → Ambient Occlusion [Opcional]
                    → Salida Lit (Albedo, Normal, Smoothness, AO)
```

### Por qué es importante para el rendimiento
Al tener un solo shader que cubre la mayoría de variaciones visuales, se reduce drásticamente la cantidad de shaders únicos compilados y el tiempo de cambio de estado de la GPU entre draw calls.

<img src="https://raw.githubusercontent.com/Tianaq26/eteria-world/main/docs/assets/shaders/ShaderReactivo.png" alt="descripcion" width="80%"/>

---

## 2. Dragones

**Archivo:** `dragones.shadergraph`
**Tipo:** Lit · Opaque · Metallic Workflow

### Propósito
Shader para todos los modelos de dragones del juego. Permite que cada instancia de dragón tenga una variación de color única sin duplicar el material, usando GPU Instancing. La textura base define los detalles visuales del modelo y el color se fusiona encima con una intensidad configurable.

### Parámetros expuestos

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `base` | Texture2D | Textura pintada a mano del dragón |
| `Color` | Color | Color de variación por instancia |
| `Fusion` | Float | Intensidad de fusión del color sobre la textura (0 = solo textura, 1 = color sólido) |

### Cómo funciona

El grafo samplea la textura base con su tiling y coordenadas UV. Luego mezcla el resultado con el color de instancia usando el parámetro `Fusion` como control de blend. Al tener GPU Instancing activado en el material, Unity puede pasar valores de `Color` y `Fusion` distintos a cada instancia del dragón sin romper el batching.

```
Textura base (UV del modelo)
    → Blend con Color de instancia controlado por Fusion
        → Normal map del modelo
            → Salida Lit (Albedo, Normal, Smoothness)
```

### Resultado visual
Cada dragón capturado en el juego conserva su color único generado al momento de spawn. Ese color se guarda en la base de datos y se aplica como parámetro de instancia al cargar el dragón, manteniendo coherencia visual entre sesiones.

<img src="https://raw.githubusercontent.com/Tianaq26/eteria-world/main/docs/assets/shaders/Dragones.png" alt="descripcion" width="80%"/>
---

## 3. Rio y lagos

**Archivo:** `rio_water.shadergraph`
**Tipo:** Lit · Transparent · Alpha Blend

### Propósito
Shader para ríos del mundo. Simula agua en movimiento con ondas animadas, espuma en orillas y transparencia configurable. Tiene dos sistemas independientes en el grafo: el cuerpo del agua (parte superior) y el sistema de espuma (parte inferior).

### Parámetros expuestos

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `Color` | Color | Color base del agua |
| `velocidad ondas` | Float | Velocidad de animación de las ondas |
| `color ondas` | Color | Color de las crestas de onda |
| `escala de ondas` | Float | Tamaño del patrón de onda |
| `disolucion de onda` | Float | Qué tan definidas o suaves son las ondas |
| `gloss` | Float | Reflectividad de la superficie |
| `metalicidad` | Float | Nivel metálico del agua |
| `normalStrength` | Float | Intensidad del normal map de olas |
| `Normal speed` | Vector | Dirección y velocidad de animación del normal map |
| `Foam offset` | Float | Distancia desde la orilla donde aparece espuma |
| `transparencia` | Float | Nivel de transparencia del agua |
| `foamcolor` | Color | Color de la espuma |
| `alfa` | Float | Control fino de la transparencia general |

### Cómo funciona

**Cuerpo del agua:** Un normal map de olas se mueve usando un nodo Time multiplicado por `Normal speed`, generando la ilusión de flujo. Dos capas de normal maps con velocidades levemente distintas se combinan para evitar patrones repetitivos. El color varía entre `Color` base y `color ondas` según la intensidad del normal, simulando crestas brillantes.

**Espuma:** Un sistema separado en la parte inferior del grafo calcula la distancia al borde de la geometría usando depth buffer. Donde la profundidad es baja (orilla), se activa el color de espuma con su propia opacidad, creando el efecto de agua rompiendo en la costa.

```
Normal map animado (capa 1 + capa 2 con velocidades distintas)
    → Color base mezclado con color de onda por intensidad normal
        → Transparencia por alfa
            → [Paralelo] Sistema de espuma por profundidad
                → Salida Lit Transparent (Albedo, Normal, Alpha)
```
<img src="https://raw.githubusercontent.com/Tianaq26/eteria-world/main/docs/assets/shaders/Lago.png" alt="descripcion" width="80%"/>

> *EL shader esta en desarrollo el efecto de la espuma no funciona correctamente*
---

## 4. Catarata

**Archivo:** `catarata.shadergraph`
**Tipo:** Unlit · Transparent · Alpha Blend

### Propósito
Shader para cascadas y caídas de agua. Usa material **Unlit** (sin iluminación de escena) para que el agua siempre se vea luminosa y definida independientemente de las condiciones de luz del entorno. El movimiento se logra con dos sistemas: animación de UVs y desplazamiento de vértices.

### Parámetros expuestos

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `riples color` | Color | Color del patrón de ondas/rizos |
| `ruples speed` | Float | Velocidad de animación de los rizos |
| `ripies amount` | Float | Cantidad o densidad de rizos |
| `voronoi speed` | Float | Velocidad del patrón Voronoi de fondo |
| `voronoi scale` | Float | Escala del patrón Voronoi |
| `bottom power` | Float | Intensidad del efecto en la base de la cascada |
| `bottom brillo` | Float | Brillo en la zona de impacto inferior |
| `vertexOffsetAmount` | Float | Cantidad de desplazamiento de vértices |

### Cómo funciona

El grafo usa un **patrón Voronoi** animado como base visual del agua cayendo, controlado por `voronoi speed` y `voronoi scale`. Encima se agrega una capa de rizos (riples) con su propia velocidad y densidad para dar detalle fino. Las UVs de ambas capas se desplazan hacia abajo con el nodo Time para simular el flujo descendente del agua.

El **vertex displacement** mueve la geometría del plano de la cascada en el eje de la normal, creando ondulación tridimensional en la superficie del agua en lugar de ser un plano perfectamente plano.

La zona inferior (`bottom power`, `bottom brillo`) recibe un tratamiento especial para simular el impacto del agua al chocar con la superficie, con mayor brillo y espuma concentrada en esa área.

```
Voronoi animado (velocidad + escala)
    → Blend con capa de rizos animados
        → UVs desplazadas hacia abajo (flujo de caída)
            → Vertex displacement en normal del plano
                → Efecto de impacto en zona inferior
                    → Salida Unlit Transparent (Emission, Alpha)
```

<img src="https://raw.githubusercontent.com/Tianaq26/eteria-world/main/docs/assets/shaders/Catara.gif" alt="Catara" width="80%"/>

>  *El shader de Catarata se usa en un sistema de particulas*

---

## 5. Water Shader (Mar)

**Archivo:** `water_shader.shadergraph`
**Tipo:** Lit · Transparent · Alpha Blend

### Propósito
Shader del océano entre las islas. Es el más complejo del proyecto. Simula profundidad del agua por color, movimiento de olas con dos texturas animadas, normal maps dobles para detalles de superficie y displacement de vértices para geometría de ola real. El cambio de color entre zonas someras y profundas es el efecto visual más distintivo.

### Parámetros expuestos

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `Depth` | Float | Distancia de profundidad para el gradiente de color |
| `fuerza` | Float | Intensidad general del efecto de olas |
| `depthColor` | Color | Color del agua en zonas profundas (azul oscuro) |
| `shallowColor` | Color | Color del agua en zonas poco profundas (azul claro/verde) |
| `MainTexture` | Texture2D | Textura principal de la superficie del agua |
| `Second Texture` | Texture2D | Segunda textura de superficie para mezcla |
| `fuerza normal` | Float | Intensidad de los normal maps |
| `smoth` | Float | Smoothness/reflectividad de la superficie |
| `displasment` | Float | Cantidad de desplazamiento de vértices (altura de olas) |
| `anim strength` | Float | Intensidad de la animación de las texturas |

### Cómo funciona

**Sistema de profundidad:** Compara la posición en Y del fragmento contra el depth buffer de la escena. Donde hay poca distancia al fondo (zona somera), interpola hacia `shallowColor`. Donde hay mucha distancia (zona profunda), usa `depthColor`. Este gradiente funciona en tiempo real sin necesidad de pintar zonas manualmente.

**Animación de superficie:** Dos texturas (`MainTexture` y `Second Texture`) se mueven en direcciones levemente opuestas usando el nodo Time multiplicado por `anim strength`. Al combinarlas, el patrón de intersección genera una apariencia de olas más compleja y menos repetitiva que una sola textura.

**Normal maps dobles:** Dos normal maps se mueven a velocidades distintas y se combinan antes de enviarse a la salida. Esto genera un detalle de superficie más rico y natural que un solo normal map.

**Vertex displacement:** Las UVs del plano del mar se usan para calcular un desplazamiento vertical de los vértices basado en `displasment`, creando ondulación geométrica real en la malla del océano, no solo ilusión visual.

```
Depth buffer → gradiente depthColor / shallowColor
    ↓
MainTexture animada + Second Texture animada (direcciones opuestas)
    → Blend de texturas
        → Normal map 1 + Normal map 2 (velocidades distintas) → Combined Normal
            → Vertex displacement por UVs
                → Salida Lit Transparent (Albedo, Normal, Smoothness, Alpha)
```

### Por qué Transparent y no Opaque
El mar necesita mostrar el fondo en zonas someras (arena, rocas sumergidas) y oscurecerse progresivamente en profundidad. Usar Transparent con Alpha Blend permite que el color del fondo influya en el resultado final, haciendo que el efecto de profundidad sea físicamente coherente con la geometría real del terreno submarino.

<img src="https://raw.githubusercontent.com/Tianaq26/eteria-world/main/docs/assets/shaders/Mar.png" alt="Catara" width="80%"/>

---

## 6. Rayo VFX

**Archivo:** `RayoShader2.shadergraph`
**Tipo:** Lit · Transparent · **Additive Blend**

### Propósito
Shader para los efectos visuales de ataques de tipo rayo(rayo de ambar,rayo de veneno, ect) de los dragones. El blending **Additive** es la característica más importante: en lugar de reemplazar el color del fondo, lo suma, haciendo que el efecto siempre brille sobre cualquier superficie como si emitiera luz propia. Es el método estándar para efectos de energía y magia en videojuegos.

### Parámetros expuestos

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `Color` | Color | Color base del efecto (ámbar, azul, verde según tipo de dragón) |
| `mask` | Texture2D | Máscara de ruido que controla dónde aparece el efecto |
| `mainTex` | Texture2D | Textura principal con forma del rayo |
| `MainTexspeed` | Vector | Dirección y velocidad de animación de la textura principal |
| `AlphaClip` | Float | Umbral de corte de transparencia (define bordes del efecto) |
| `DistortionSpeed` | Vector | Dirección y velocidad de la distorsión UV |
| `distortionAmount` | Float | Intensidad de la distorsión (qué tan irregular se ve el rayo) |
| `DistortionScale` | Float | Escala del patrón de distorsión |

### Cómo funciona

El grafo combina dos sistemas: la textura principal del rayo (`mainTex`) animada con `MainTexspeed`, y una máscara de ruido (`mask`) que se usa para distorsionar las UVs de la textura principal antes de samplearla. Esto hace que el rayo no se vea como una imagen estática moviéndose, sino como una forma que se deforma y ondula orgánicamente en tiempo real.

El `AlphaClip` recorta los píxeles más transparentes, definiendo el contorno del efecto de forma no uniforme, lo que da la apariencia de bordes irregulares propios de un rayo eléctrico o llamarada.

Al ser Additive, múltiples capas del mismo shader pueden superponerse sumando brillo, lo que se usa en el sistema VFX para crear el efecto de rayo con una capa base más oscura y una capa interior más brillante.

```
mask (ruido) → distorsión de UVs (DistortionSpeed + distortionAmount)
    → mainTex sampleada con UVs distorsionadas + animadas (MainTexspeed)
        → AlphaClip para bordes irregulares
            → Color multiplicado
                → Salida Additive (se suma al fondo → efecto de brillo)
```

### Por qué Additive y no Alpha Blend
Con Alpha Blend el efecto reemplazaría parcialmente el color del fondo, perdiendo luminosidad sobre fondos oscuros. Con Additive, el efecto siempre se ve brillante independientemente del entorno: sobre cielo oscuro brilla más, sobre terreno claro se mezcla naturalmente. Es el comportamiento físicamente correcto para energía luminosa ademas añade un mejor efecto con el postprosesado(bloom, ect).

<img src="https://raw.githubusercontent.com/Tianaq26/eteria-world/main/docs/assets/shaders/Rayo.gif" alt="Rayo" width="80%"/>

---

## 7. ShaderNPC

**Archivo:** `ShaderNPC.shadergraph`
**Tipo:** Lit · Opaque · Metallic Workflow

### Propósito
Shader para todos los NPCs del juego. Resuelve un problema de optimización concreto: los modelos de NPCs tienen tres zonas visuales distintas (pelo, cara, ropa) que normalmente requerirían tres materiales separados y tres draw calls. Este shader las unifica en un solo material usando una textura de máscara RGB para determinar qué textura se aplica en cada zona del modelo.

### Parámetros expuestos

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `Pelo` | Texture2D | Textura del cabello del NPC |
| `Cara` | Texture2D | Textura de la cara (incluye rasgos, pecas, detalles faciales) |
| `Ropa` | Texture2D | Textura de la vestimenta |
| `BlendTexture` | Texture2D | Máscara RGB que define qué zona recibe qué textura |

### Cómo funciona

La `BlendTexture` es una textura de máscara donde cada canal de color (R, G, B) representa una zona distinta del modelo:
- **Canal R (rojo)** → zona de ropa
- **Canal G (verde)** → zona de cara
- **Canal B (azul)** → zona de pelo

El grafo samplea las tres texturas en paralelo y usa los valores de cada canal de la máscara como peso de mezcla (Lerp) para seleccionar qué textura domina en cada píxel. El resultado es un solo material que renderiza las tres zonas correctamente sin necesidad de subdividir la malla ni usar múltiples materiales.

```
BlendTexture → separar canales R, G, B
    ↓              ↓              ↓
  Ropa          Cara           Pelo
    └──────────── Lerp por canal ────────────┘
                        ↓
              Normal map combinado
                        ↓
              Salida Lit (Albedo, Normal, Smoothness)
```

### Impacto en rendimiento
Cada NPC del juego usa un solo material en lugar de tres. Con docenas de NPCs distribuidos por el mundo, esto se traduce directamente en menos draw calls y menos cambios de estado de la GPU por frame. Además, facilita crear variantes visuales de NPCs: basta con cambiar la textura de `Ropa` o `Cara` para obtener un NPC visualmente distinto sin duplicar el shader.

<img src="https://raw.githubusercontent.com/Tianaq26/eteria-world/main/docs/assets/shaders/npcs.png" alt="npcs" width="80%"/>

---

## Comparativa general

| Shader | Tipo | Transparent | Unlit | Blending | Vertex Displacement | Normal Map | GPU Instancing |
|--------|------|-------------|-------|----------|---------------------|------------|----------------|
| Shader Reactivo | Lit | ❌ | ❌ | Opaque | ❌ | ✅ opcional | ✅ |
| Dragones | Lit | ❌ | ❌ | Opaque | ❌ | ✅ | ✅ |
| Rio y lago | Lit | ✅ | ❌ | Alpha | ❌ | ✅ doble | ❌ |
| Catarata | Unlit | ✅ | ✅ | Alpha | ✅ | ❌ | ❌ |
| Water Shader | Lit | ✅ | ❌ | Alpha | ✅ | ✅ doble | ❌ |
| Rayo VFX | Lit | ✅ | ❌ | **Additive** | ❌ | ❌ | ❌ |
| ShaderNPC | Lit | ❌ | ❌ | Opaque | ❌ | ❌ | ✅ |

---

> 📸 *Todas las capturas referenciadas deben colocarse en `docs/assets/shaders/` con nombres descriptivos (ej: `shader-reactivo-comparativa.png`, `mar-profundidad.png`) y actualizar las rutas en este documento.*
