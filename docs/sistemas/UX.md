# 🌍 Sistema de Mundo, NPCs, Animales y Recolección — Eteria World

> Documentación técnica de los sistemas que dan vida al mundo abierto de Eteria: comportamiento de NPCs, animales, spawneo, recolección de recursos, clima, ciclo día/noche, niebla volumétrica y antorchas. Todos estos sistemas priorizan el rendimiento mediante LOD lógico y activación por distancia.

**Scripts involucrados:** `NPC_behavior.cs` · `SpawnNpc.cs` · `COV.cs` (Animal base) · `SAC.cs` · `rockD.cs` · `CCL.cs` · `CDN.cs` · `fogPoint.cs` · `Vfx_antorcha.cs`

---

## Índice

1. [Arquitectura general del mundo](#1-arquitectura-general-del-mundo)
2. [Sistema de NPCs — NPC_behavior + SpawnNpc](#2-sistema-de-npcs--npc_behavior--spawnnpc)
3. [Sistema de animales — Animal + COV + SAC](#3-sistema-de-animales--animal--cov--sac)
4. [Sistema de recolección — rockD](#4-sistema-de-recolección--rockd)
5. [Sistema de clima — CCL](#5-sistema-de-clima--ccl)
6. [Ciclo día/noche — CDN](#6-ciclo-díanoche--cdn)
7. [Niebla volumétrica — fogPoint](#7-niebla-volumétrica--fogpoint)
8. [Antorchas con LOD — Vfx_antorcha](#8-antorchas-con-lod--vfx_antorcha)
9. [Integración entre sistemas](#9-integración-entre-sistemas)

---

## 1. Arquitectura general del mundo

Todos los sistemas del mundo se comunican a través de `DataBase1` y reaccionan al estado global del juego sin acoplamiento directo entre sí.

```mermaid
graph TD
    DB[(DataBase1\nSingleton)]

    CCL[CCL\nClima] -->|evento_activo| CDN
    CDN[CDN\nCiclo día/noche] -->|hora| DB
    CDN -->|skyboxBlended| RS[RenderSettings]

    DB -->|playerReference| NPC[NPC_behavior]
    DB -->|playerReference| FOG[fogPoint]
    DB -->|playerReference| ANT[Vfx_antorcha]
    DB -->|playerReference| CDS2[CDS\nDragones salvajes]
    DB -->|carga| SAC2[SAC\nSpawn animales]
    DB -->|carga| SPAWN2[spawn\nSpawn dragones]

    NPC -->|NavMeshAgent| NM[NavMesh]
    SAC2 -->|Instancia| COV2[COV\nOvejas]
    COV2 -->|hereda de| AN[Animal\nClase base]

    FOG -->|pivoteCamara1| DB
    CDN -->|hora| FOG
```

---

## 2. Sistema de NPCs — NPC_behavior + SpawnNpc

### 2.1 Spawneo de NPCs (`SpawnNpc.cs`)

El sistema genera NPCs con apariencia única combinando texturas de pelo, cara y ropa sin duplicar materiales.

```mermaid
flowchart TD
    A([Start]) --> B[GenerarCombinaciones\npelo × cara × ropa]
    B --> C[MezclarLista\nFisher-Yates shuffle]
    C --> D[Loop cantidad de NPCs]
    D --> E[Seleccionar prefab aleatorio\nhombre o mujer]
    E --> F[Random dentro de radioSpawn]
    F --> G[NavMesh.SamplePosition\nbuscar punto válido]
    G --> H{¿Punto encontrado?}
    H -->|No| I([Saltar este NPC])
    H -->|Sí| J[Instantiate en NavMesh]
    J --> K[GetComponentInChildren\nSkinnedMeshRenderer]
    K --> L[MaterialPropertyBlock:\n_Pelo = pelo de combinación\n_Cara = cara de combinación\n_Ropa = ropa de combinación\n_BlendTexture = máscara RGB]
    L --> M[rend.SetPropertyBlock mpb]
    M --> N[AddComponent NPC_behavior]
    N --> O([NPC activo en mundo])
```

**Por qué `MaterialPropertyBlock` y no materiales únicos:**
Con 10 NPCs, 3 opciones de pelo, 3 de cara y 3 de ropa hay 27 combinaciones posibles. Crear un material por combinación generaría 27 materiales únicos, rompiendo el batching. Con `MaterialPropertyBlock`, todos los NPCs comparten el mismo material base y la GPU recibe los parámetros de textura por instancia, manteniendo el batching activo.

### 2.2 Comportamiento de NPCs (`NPC_behavior.cs`)

Sistema de deambulación autónoma con LOD lógico completo: cuando el jugador está lejos, el NPC se congela completamente — sin Update, sin NavMesh, sin animación.

```mermaid
stateDiagram-v2
    [*] --> Inicializando : Start

    Inicializando --> Caminando : ColocarEnNavMesh\nElegirNuevoDestino

    Caminando --> Llegó : remainingDistance\n<= distanciaLlegada

    Llegó --> Pausado : StartCoroutine PausarNPC\n(distanciaRecorrida >= distanciaParaPausa)

    Llegó --> Caminando : ElegirNuevoDestino\nnuevo destino aleatorio

    Pausado --> Caminando : duracion aleatoria\n2s - 5s completada

    Caminando --> Bloqueado : velocity < 0.05\ny remainingDistance > 0.5\ndurante 2s

    Bloqueado --> Caminando : ElegirNuevoDestino\n(anti-bloqueo forzado)

    Caminando --> Congelado : CheckDistance:\ndistancia > distanciaActivacion

    Congelado --> Caminando : CheckDistance:\ndistancia <= distanciaActivacion\nagent.enabled = true

    note right of Congelado
        agent.enabled = false
        anim.SetFloat speed 0
        No ejecuta Update
        Corrutina CheckDistance
        cada 1 segundo
    end note

    note right of Pausado
        agent.ResetPath
        anim speed = 0
        Duración aleatoria
        posicionAnterior reset
    end note
```

**Sistema de pausa inteligente:**
En lugar de que los NPCs caminen indefinidamente, acumulan distancia recorrida. Al superar `distanciaParaPausa` (8 unidades por defecto), se detienen durante un tiempo aleatorio (2-5 segundos) simulando comportamiento natural como mirar alrededor o descansar.

```mermaid
flowchart LR
    A([Cada frame activo]) --> B[delta = distancia\nposActual - posAnterior]
    B --> C[distanciaRecorrida += delta]
    C --> D{distanciaRecorrida\n>= distanciaParaPausa}
    D -->|No| E[Continuar caminando]
    D -->|Sí| F[distanciaRecorrida = 0\nStartCoroutine PausarNPC]
    F --> G[agent.ResetPath\nanim speed = 0]
    G --> H[WaitForSeconds\nRandom 2-5s]
    H --> I[ElegirNuevoDestino]
```

**ElegirNuevoDestino — selección robusta:**
Hace hasta 8 intentos para encontrar un punto válido en el NavMesh. Verifica que el path calculado tenga estado `PathComplete` antes de asignarlo, descartando puntos en islas de NavMesh separadas donde el NPC quedaría atrapado.

**Configuración del NavMeshAgent para multitud:**
```
speed:              Random 1.8 - 2.8 m/s    (velocidad variable evita sincronización)
radius:             0.22                      (radio pequeño → pueden pasar cerca)
avoidancePriority:  Random 20 - 80           (prioridad variable → no todos ceden a la vez)
obstacleAvoidance:  LowQualityObstacleAvoidance (suficiente para NPCs de fondo)
```

> 📸 *[Insertar GIF de la aldea con múltiples NPCs deambulando con comportamiento natural]*

---

## 3. Sistema de animales — Animal + COV + SAC

### 3.1 Spawneo de animales (`SAC.cs`)

```mermaid
flowchart TD
    A([Start]) --> B[StartCoroutine crear]
    B --> C{db.carga > 24}
    C -->|No| D[yield return null\nesperar]
    C -->|Sí| E[Loop cantidad aleatoria\ncantMin - cantMax]
    E --> F[instantiate1]
    F --> G[Random dentro de radioSpawn]
    G --> H[Raycast hacia abajo\ndesde +10 altura]
    H --> I{¿Golpea terreno?}
    I -->|Sí| J[spawnPos = hit.point\nsobre el suelo exacto]
    I -->|No| K[Usar posición sin ajuste]
    J & K --> L[Instantiate animal]
    L --> M[db.INI_ovejas]
    M --> N([Animales activos en mundo])
```

### 3.2 Clase base Animal y comportamiento (`COV.cs`)

La clase `Animal` es una clase base que define el sistema de vida, recompensas y muerte. `COV` (oveja) hereda de ella y configura sus parámetros específicos.

```mermaid
classDiagram
    class Animal {
        +string tipo
        +float vidaMax
        +float vidaActual
        +List~Recompensa~ posiblesRecompensas
        +NavMeshAgent agent
        +Animator anim
        +Start()
        +Update()
        +Morir()*
        +RecibirDaño(float daño)
        +DropRecompensas()
    }

    class Recompensa {
        +ObjetoEnDB objetoEnDB
        +float probabilidad
        +int cantidadMin
        +int cantidadMax
    }

    class COV {
        +tipo = Oveja
        +vidaMax = 50
        +posiblesRecompensas : Lana 60% + Carne 100%
        +Start()
        +Morir()
    }

    Animal <|-- COV : hereda
    Animal "1" --> "*" Recompensa : tiene
```

**Sistema de recompensas al morir (COV):**

| Recompensa | Probabilidad | Cantidad |
|-----------|-------------|---------|
| Lana (`objetosInertes[0]`) | 60% | 1 - 6 |
| Carne (`objetosConsumibles[0]`) | 100% | 1 - 7 |

La probabilidad se evalúa con `Random.value < probabilidad` por cada recompensa independientemente, por lo que una oveja puede soltar ambas, solo una, o solo la garantizada (carne).

---

## 4. Sistema de recolección — rockD

Las rocas interactuables requieren un arma de minería específica para romperse. El sistema detecta el arma activa del jugador dentro de un trigger y verifica si puede hacer daño en el frame actual.

```mermaid
flowchart TD
    A([Jugador entra en trigger]) --> B{¿roto?}
    B -->|Sí| Z([Ignorar])
    B -->|No| C[StartCoroutine\nChequearJugadorDentro]

    C --> D{Cada frame:\n¿db.PlayerActiveArma\nes pico o mazo?}
    D -->|ID 9, 10 u 11| E{¿CA2.puedeHacerDaño?}
    D -->|Otra arma| F[yield null\nsiguiente frame]
    E -->|No| F
    E -->|Sí| G[RocaRomper]

    G --> H[Animator.SetBool roto true]
    H --> I[Rigidbody.isKinematic = false\nen todos los hijos]
    I --> J[WaitForSeconds 1s]
    J --> K[Random cantidad\ncantMin - cantMax]
    K --> L[db.objetosInertes 0 .cantidad += cant\nLana / Piedra]
    L --> M{¿mineral != 0?}
    M -->|Sí| N[Random 1-4\ndb.objetosInertes 2 += cant\nHierro]
    N --> O[db.hierro_conseguido += cant]
    M & O --> P[WaitForSeconds 4s]
    P --> Q[Destroy gameObject]

    R([Jugador sale del trigger]) --> S[StopCoroutine\nChequearJugadorDentro]
```

**Por qué corrutina en lugar de Update:**
Verificar el arma activa y el estado de daño en cada frame (60 veces/segundo) para cada roca en la escena es costoso. La corrutina solo existe mientras el jugador está dentro del trigger, y se detiene automáticamente al salir, eliminando todo costo cuando el jugador no está interactuando con la roca.

---

## 5. Sistema de clima — CCL

El clima cambia automáticamente cada cierto intervalo con probabilidad configurable. Puede ser forzado por eventos de misión.

```mermaid
stateDiagram-v2
    [*] --> Soleado : Start\nAplicarClima inicial

    Soleado --> Lluvia : CicloClima cada 50s\nRandom.value < 0.4\ny !evento_activo

    Lluvia --> Soleado : CicloClima cada 50s\nRandom.value >= 0.4\ny !evento_activo

    Soleado --> ForzadoLluvia : ForzarClima Lluvia\ndesde misión o evento

    ForzadoLluvia --> Soleado : ForzarClima Soleado

    note right of Lluvia
        lluviaFX.SetActive true
        temperatura = Random 5-15°C
    end note

    note right of Soleado
        lluviaFX.SetActive false
        temperatura = Random 25-35°C
    end note
```

**`evento_activo`:** Cuando una misión o cinemática requiere un clima específico, activa este flag para bloquear los cambios automáticos. Así el clima se mantiene fijo durante el evento y vuelve al ciclo normal al terminar.

---

## 6. Ciclo día/noche — CDN

El ciclo día/noche blenda entre 5 texturas de skybox panorámico según la hora actual, con velocidad acelerada durante la noche para que no sea aburrida.

```mermaid
flowchart TD
    A([Update CDN_activo]) --> B{hora >= 20\no hora < 6}
    B -->|Sí noche| C[velocidad = 1.5×]
    B -->|No día| D[velocidad = 1×]
    C & D --> E[hora += deltaTime × velocidad\n× 24 / 60 × duracionDia]
    E --> F{hora >= 24}
    F -->|Sí| G[hora = 0]
    F & G --> H[RS — Rotación del sol]

    H --> I[solx = 15 × hora\nsol.localEulerAngles.x = solx]
    I --> J{hora 6-18}
    J -->|Sí| K[luzSol.intensity = 1]
    J -->|No| L[luzSol.intensity = 0]
    K & L --> M[Seleccionar par de skyboxes]

    M --> N{Rango de hora}
    N -->|4-6h| O[Noche → Amanecer\nblend = InverseLerp 4 6 hora]
    N -->|6-8h| P[Amanecer → Día/Nublado\nblend = InverseLerp 6 8 hora]
    N -->|8-18h| Q[Día/Nublado → Tarde\nblend = InverseLerp 8 18 hora]
    N -->|18-20h| R[Tarde → Noche\nblend = InverseLerp 18 20 hora]
    N -->|20-4h| S[Noche → Noche\nblend = 0]

    O & P & Q & R & S --> T[skyboxBlended.SetTexture _Tex1 tex1\nskyboxBlended.SetTexture _Tex2 tex2\nskyboxBlended.SetFloat _Blend blend]
    T --> U[RenderSettings.skybox = skyboxBlended]
```

**Integración con clima:**
En el rango 6-8h (amanecer → día), si `CCL.evento_activo` está activado, se usa `skyboxNublado` en lugar de `skyboxDia`. Esto hace que el skybox responda automáticamente al estado del clima sin código adicional en CCL.

**5 texturas de skybox:**

| Textura | Rango horario |
|---------|--------------|
| `skyboxNoche` | 20h - 6h |
| `skyboxAmanecer` | 4h - 8h (transición) |
| `skyboxDia` | 6h - 18h |
| `skyboxNublado` | 6h - 18h (si llueve) |
| `skyboxTarde` | 8h - 20h (transición) |

> 📸 *[Insertar GIF del ciclo día/noche mostrando las 5 transiciones de skybox]*

---

## 7. Niebla volumétrica — fogPoint

Cada punto de niebla en el mundo es un VFX Graph que se activa y ajusta su densidad según la distancia al jugador. El sistema tiene zona de aparición gradual y zona de desvanecimiento suave más allá del radio máximo.

```mermaid
flowchart TD
    A([Start]) --> B[Obtener posicionBase]
    B --> C[Offsets aleatorios X y Z\npara movimiento de Perlin]
    C --> D[StartCoroutine ControlNiebla\ncada 0.2 segundos]

    D --> E[Calcular distancia\nposicionBase → pivoteCamara1]
    E --> F{Rango de distancia}

    F -->|d <= distanciaMax\n1000 unidades| G[colorM = InverseLerp\ndistMin distMax d × 1.2\naparece gradualmente]

    F -->|distMax < d\n<= distMax + fadeZona\n1200 unidades| H[fadeT = InverseLerp\ndistMax distMax+200 d\ncolorM = Lerp 1.2 0 fadeT\ndesvanece suavemente]

    F -->|d > distMax + fadeZona| I[colorM ≈ 0\nVFX invisible]

    G & H & I --> J[vfx.SetFloat colorM colorM+0.05]
    J --> K[Movimiento orgánico:\nx = Sin tiempo×vel+offsetX × amplitud\nz = Cos tiempo×vel+offsetZ × amplitud\ntransform.position = base + offset]

    L([Cada 10 iteraciones\n= cada ~10s]) --> M{hora >= 20\no hora < 6}
    M -->|Noche| N[distanciaMin = 300\nniebla más densa de noche]
    M -->|Día| O[distanciaMin = 100\nniebla más tenue de día]
```

**Por qué 0.2s y no Update:**
La niebla no necesita actualizarse 60 veces por segundo. Los cambios de densidad y posición son imperceptibles a esa frecuencia. Actualizar a 5Hz (cada 0.2s) reduce el costo de CPU de todos los puntos de niebla en un 92% respecto a Update, sin diferencia visual.

---

## 8. Antorchas con LOD — Vfx_antorcha

Cada antorcha del mundo tiene un VFX Graph de llama y una luz puntual. Ambos se desactivan automáticamente cuando el jugador está lejos, usando `InvokeRepeating` en lugar de Update.

```mermaid
flowchart TD
    A([Start]) --> B[luzAntorcha.intensity = 0\nvfxLlama.enabled = false]
    B --> C[InvokeRepeating ActualizarLOD\noffset aleatorio 0-2s\ncada 2 segundos]

    C --> D[ActualizarLOD]
    D --> E[dist = SqrMagnitude\nposición - playerReference]
    E --> F{dist < distanciaFade²\n40 unidades}
    F -->|Cerca| G[vfxLlama.enabled = true\nintensidadObjetivo = intensidadMax]
    F -->|Lejos| H[vfxLlama.enabled = false\nintensidadObjetivo = 0]

    G & H --> I([Update — fade de luz])
    I --> J{intensity != intensidadObjetivo}
    J -->|Sí| K[MoveTowards intensity\nobjetivo\nvelocidadFade × deltaTime × intensidadMax]
    J -->|No| L([Nada])
```

**Offset aleatorio en InvokeRepeating:**
Todas las antorchas no se actualizan en el mismo frame. El offset inicial aleatorio (0-2 segundos) distribuye las actualizaciones a lo largo del tiempo, evitando picos de CPU cuando muchas antorchas están visibles simultáneamente.

**SqrMagnitude en lugar de Distance:**
`Vector3.Distance` calcula una raíz cuadrada, que es costosa. `SqrMagnitude` devuelve el cuadrado de la distancia sin raíz cuadrada. Comparar con `distanciaFade²` es matemáticamente equivalente y más eficiente, especialmente cuando hay decenas de antorchas en escena.

---

## 9. Integración entre sistemas

Todos los sistemas del mundo reaccionan a los mismos datos centrales sin comunicarse directamente entre sí.

```mermaid
graph LR
    CDN -->|hora| fogPoint
    CCL -->|evento_activo| CDN
    DB -->|pivoteCamara1| fogPoint
    DB -->|playerReference| NPC_behavior
    DB -->|playerReference| Vfx_antorcha
    DB -->|carga > 24| SAC
    DB -->|carga > 9| spawn

    fogPoint -->|distanciaMin\nsegún hora| fogPoint
    CDN -->|skybox| RenderSettings
    CCL -->|lluviaFX| Particulas

    subgraph "LOD por distancia"
        NPC_behavior -->|CheckDistance 1Hz| NavMeshAgent
        Vfx_antorcha -->|InvokeRepeating 2s| VFXGraph
        fogPoint -->|Corrutina 0.2s| VFXGraph2
    end

    subgraph "Spawn controlado por carga"
        SAC -->|carga > 24| Ovejas
        spawn -->|carga > 9| DragonesSalvajes
    end
```

**Sistema de carga (`db.carga`):**
Los sistemas de spawn no se activan inmediatamente al inicio. Esperan a que `db.carga` supere un umbral, evitando que todos los sistemas inicialicen simultáneamente en el primer frame y causando picos de CPU en la carga del juego. Los dragones salvajes esperan `carga > 9` y los animales esperan `carga > 24`, escalonando la inicialización.

---

> 📸 *Capturas sugeridas:*
> - `docs/assets/sistemas/npcs-aldea.gif` — NPCs deambulando con pausas naturales
> - `docs/assets/sistemas/ciclo-dia-noche.gif` — transición completa de skybox
> - `docs/assets/sistemas/niebla-volumetrica.png` — efecto de niebla en zona boscosa
> - `docs/assets/sistemas/roca-recoleccion.gif` — animación de roca rompiéndose
> - `docs/assets/sistemas/antorcha-lod.gif` — antorcha activándose al acercarse
