# 🐉 Sistema de Dragones y Combate — Eteria World

> Documentación técnica de los sistemas de dragones: instanciado, montura, comportamiento de dragones salvajes y combate elemental por turnos. Basada directamente en el código fuente.

**Scripts involucrados:** `CMD.cs` · `CD.cs` · `spawn.cs` · `CDS.cs` · `DVD.cs` · `StatsUt.cs`

---

## Índice

1. [Arquitectura general](#1-arquitectura-general)
2. [Sistema de instanciado — CMD](#2-sistema-de-instanciado--cmd)
3. [Máquina de estados del dragón activo — CD](#3-máquina-de-estados-del-dragón-activo--cd)
4. [Sistema de dragones salvajes — spawn + CDS](#4-sistema-de-dragones-salvajes--spawn--cds)
5. [Sistema de combate elemental — DVD + StatsUt](#5-sistema-de-combate-elemental--dvd--statsut)
6. [Fórmula de daño](#6-fórmula-de-daño)

---

## 1. Arquitectura general

Relación entre los scripts del sistema de dragones y cómo se comunican a través del singleton `DataBase1`.

```mermaid
graph TD
    DB[(DataBase1\nSingleton)] 
    CMD[CMD\nInstanciado de dragón]
    CD[CD\nDragón activo del jugador]
    SPAWN[spawn\nSpawneo de salvajes]
    CDS[CDS\nComportamiento salvaje]
    DVD[DVD\nCombate por turnos]
    STATS[StatsUt\nCálculos de stats]
    M2[movimiento2\nControl de vuelo]

    DB -->|PlayerDragons| CMD
    DB -->|dragonactivo| CD
    DB -->|textura, dragones| SPAWN
    DB -->|nivelDelJuego| SPAWN
    CMD -->|Instancia + configura| CD
    CMD -->|Asigna anim1| M2
    CD -->|Detecta colisión| CDS
    CDS -->|iniciarPelea| DVD
    DVD -->|CalculoDaño| STATS
    DVD -->|peleaActiva| DB
    SPAWN -->|Instancia + stats| CDS
```

---

## 2. Sistema de instanciado — CMD

`CMD.cs` construye el dragón del jugador en runtime, ensamblando todos sus componentes dinámicamente en lugar de usar prefabs preconstruidos.

```mermaid
flowchart TD
    A([Llamada a Dragonini]) --> B{¿Vida del dragón > 0?}
    B -->|No| Z([Abortar])
    B -->|Sí| C[Destruir dragón activo anterior]
    C --> D[Instanciar mesh del dragón]
    D --> E[Instanciar silla de montar]
    E --> F{¿Tipo de dragón?}
    F -->|typoD == 0| G[Renderer principal\nmaterials index 1]
    F -->|typoD == 1| H[Renderer principal\nmaterials index 0]
    F -->|typoD == 2| I[Segundo renderer\nmaterials index 0]
    G & H & I --> J[Asignar material\npor instancia\nMaterialPropertyBlock]
    J --> K[Configurar silla\nen posición local según tipo]
    K --> L[Añadir BoxCollider\nisTrigger = true]
    L --> M[Añadir Animator\n+ RuntimeAnimatorController]
    M --> N[Añadir CharacterController\ncon dimensiones por tipo]
    N --> O[Añadir componente CD\nvida + dragonIndex]
    O --> P[Añadir EventoAnimacion]
    P --> Q[Registrar en db.dragonactivo]
    Q --> R[Buscar flytrail VFX\nRecursivo en jerarquía]
    R --> S[Asignar VFX alas\na movimiento2]
    S --> T([Dragón listo])
```

**Puntos técnicos destacados:**
- El material se asigna con `MaterialPropertyBlock` — no rompe GPU Instancing
- Todos los componentes (`CD`, `CharacterController`, `Animator`) se añaden en runtime, el prefab solo contiene el mesh
- La posición de la silla varía por array `sillaP[typoD]` según el tipo de dragón
- Los VFX de alas se buscan recursivamente en la jerarquía con `Utilidades.BuscarPorNombreRecursivo`

---

## 3. Máquina de estados del dragón activo — CD

`CD.cs` gestiona todos los estados posibles del dragón que acompaña al jugador: seguimiento, montura, vuelo y desmonte.

```mermaid
stateDiagram-v2
    [*] --> Siguiendo : Inicio

    Siguiendo --> Montado : Jugador en trigger\n+ presiona E\n+ puedeMontar

    Montado --> Volando : movimiento2 activa vuelo\n(altura > suelo)

    Volando --> Montado : Dragón toca suelo

    Montado --> Desmontando : Presiona E\n+ !db.volar\n+ puedeBajar

    Desmontando --> Siguiendo : cooldown 1s\ntiempoDesmontado = Time.time

    Siguiendo --> Idle : distancia < 15\ny en suelo

    note right of Montado
        transform.parent = player
        player.position = silla1.position
        CharacterController.height = ccheight
        db.dragonAc = true
        LookCamera.distancia = 30
    end note

    note right of Desmontando
        controller.enabled = true
        db.dragonAc = false
        transform.SetParent(null)
        LookCamera.distancia = 12
        CharacterController.height = 4.2
    end note

    note right of Siguiendo
        Si no hay suelo Y lejos del jugador:
        → vuelo directo hacia jugador
        Si hay suelo Y lejos (>15u):
        → caminar con gravedad
        Si cerca: Idle
    end note
```

**Lógica de seguimiento (SeguirJugador):**

```mermaid
flowchart TD
    A([Update SeguirJugador]) --> B{¿peleaActiva?}
    B -->|Sí| Z([Return])
    B -->|No| C{¿dragonActivo en DB?}
    C -->|No| D[Reset crono → Return]
    C -->|Sí| E{¿crono < 2s?}
    E -->|Sí| F[Esperar → crono++]
    E -->|No| G{¿Hay suelo debajo?\ny cerca del jugador?}
    G -->|No suelo\nY lejos| H[Volar directo\nhacia jugador\nanim fly=true]
    G -->|Suelo\nY distancia > 15| I[Caminar hacia jugador\nSlerp rotación\nanim walk=true]
    G -->|Cerca| J[Idle\nanim walk=false\nfly=false]
    I --> K{¿isGrounded?}
    K -->|No| L[Aplicar gravedad\nvelocidadVertical.y -= 9.81]
    K -->|Sí| M[velocidadVertical.y = -1]
```

---

## 4. Sistema de dragones salvajes — spawn + CDS

### 4.1 Spawneo (`spawn.cs`)

```mermaid
flowchart TD
    A([Start]) --> B[Esperar db.carga > 9]
    B --> C[SpawnDragon]
    C --> D[Destruir dragón anterior si existe]
    D --> E[Instantiate DragonesBase]
    E --> F[Obtener Renderer hijo]
    F --> G[MaterialPropertyBlock:\n- textura base por dragonBase\n- Color aleatorio RGB\n- Fusion aleatorio 0-0.3]
    G --> H[Asignar Stats:\n- nivel = nivelDelJuego + Random\n- vida = vidaMax + AumentoVida\n- daño, armadura del DB]
    H --> I[textoNivel.text = lvl.N]
    I --> J([Dragón salvaje activo])

    K([Update]) --> L{¿vivo?}
    L -->|Sí| M([Nada])
    L -->|No| N{contador >= 500s}
    N -->|No| O[contador++]
    N -->|Sí| P[SpawnDragon\nvivo = true\ncontador = 0]
```

### 4.2 Comportamiento salvaje (`CDS.cs`)

```mermaid
stateDiagram-v2
    [*] --> Patrullando : Start → BuscarNuevoDestino

    Patrullando --> Esperando : remainingDistance <=\nstoppingDistance

    Esperando --> Patrullando : contadorEspera >=\ntiempoEspera aleatorio

    Patrullando --> IniciarPelea : ddc.entro == true\n(jugador en rango)

    IniciarPelea --> CombateActivo : fadeBackground +\ndvdscript.empezarDVD

    note right of Patrullando
        NavMesh.SamplePosition
        30 intentos máximo
        Verifica PathComplete
        antes de asignar destino
    end note

    note right of IniciarPelea
        1. agent.isStopped
        2. Rotar cámara con easing BackOut
        3. Rotar dragón enemigo con InOutSine
        4. Fade a negro
        5. Reposicionar dragones enfrentados
        6. Fade de vuelta
        7. dvdscript.empezarDVD
    end note
```

**Optimización de detección (CDS):**
```
En lugar de verificar distancia en Update (60 veces/segundo):
→ timerCheck acumula Time.deltaTime
→ ComprobarDistancia() se llama cada 0.2s (5 veces/segundo)
→ Usa sqrMagnitude en lugar de Distance (evita sqrt)
```

---

## 5. Sistema de combate elemental — DVD + StatsUt

El combate es por turnos entre el dragón del jugador y un dragón salvaje. `DVD.cs` gestiona la UI, los turnos y las animaciones. `StatsUt.cs` provee todos los cálculos matemáticos.

```mermaid
sequenceDiagram
    participant J as Jugador
    participant DVD as DVD
    participant STATS as StatsUt
    participant DB as DataBase1
    participant CDS as CDS (enemigo)

    CDS->>DVD: empezarDVD(dragonJugador, DragonBaseID, enemigo)
    DVD->>DVD: Cargar stats jugador y enemigo
    DVD->>DVD: Mostrar UI de combate
    DVD->>DVD: turnoJugador = true

    loop Turno del jugador
        J->>DVD: Selecciona ataque (1-4)
        DVD->>STATS: CalculoDaño(defensor, atacante, dañoAtaque)
        STATS-->>DVD: dañoFinal
        DVD->>DVD: vida enemigo -= dañoFinal
        DVD->>DVD: Animación de ataque
        DVD->>DVD: turnoJugador = false
    end

    loop Turno del enemigo
        DVD->>DVD: IA selecciona ataque aleatorio
        DVD->>STATS: CalculoDaño(defensor, atacante, dañoAtaque)
        STATS-->>DVD: dañoFinal
        DVD->>DVD: vida jugador -= dañoFinal
        DVD->>DVD: Animación de ataque
        DVD->>DVD: turnoJugador = true
    end

    alt Enemigo sin vida
        DVD->>DB: Agregar dragón capturado a PlayerDragons
        DVD->>DB: peleaActiva = false
        DVD->>DVD: Mostrar pantalla victoria
    else Jugador sin vida
        DVD->>DB: peleaActiva = false
        DVD->>DVD: Mostrar pantalla derrota
    end
```

**Tipos elementales y ataques:**

```mermaid
graph LR
    T0[Tipo Ámbar] --> A0[Ataque 1\nRayo de Ámbar]
    T0 --> A1[Ataque 2]
    T0 --> A2[Ataque 3]
    T0 --> A3[Ataque 4]

    T1[Tipo Veneno] --> B0[Ataque 1]
    T1 --> B1[Ataque 2]
    T1 --> B2[Ataque 3]
    T1 --> B3[Ataque 4]

    T2[Tipo Arena] --> C0[Ataque 1]
    T2 --> C1[Ataque 2]
    T2 --> C2[Ataque 3]
    T2 --> C3[Ataque 4]

    A0 & B0 & C0 --> VFX[Grafo VFX\nEspecífico por tipo]
    VFX --> DMG[StatsUt.CalculoDaño]
```

> 📸 *[Insertar GIF del combate por turnos mostrando la transición de cámara y los ataques]*

---

## 6. Fórmula de daño

`StatsUt.CalculoDaño` es la función central del sistema de combate. Tiene en cuenta nivel, stats individuales y armadura del defensor.

```
Bonus de ataque  = (niveles[2] × 0.025 + 1) × daño × (nivel × 0.02 + 1)
Daño base        = dañoAtaque × bonusAtaque
Defensa %        = (niveles[4] × 0.025 + 1) × armadura × (nivel × 0.02 + 1)
Defensa %        = Clamp(defensa%, 0, 60)        ← máximo 60% de reducción
Daño final       = dañoBase × (1 - defensa%/100)
Daño final       = Max(1, dañoFinal)             ← mínimo 1 de daño siempre
Daño final       = Round(dañoFinal)              ← sin decimales
```

**Sistema de captura (`StatsUt.intentarCaptura`):**

```
Dificultad = (vida/vidaMaxima × 100) + nivel×2 - nivelPescado
Probabilidad = 1 - (dificultad/100)
Probabilidad = Clamp(probabilidad, 0, 1)
Captura exitosa si Random.value < probabilidad
```

Esto significa que:
- Un dragón con poca vida y nivel bajo es fácil de capturar
- El nivel del pescado usado reduce la dificultad directamente
- Siempre hay al menos una probabilidad mínima de captura

---

> 📸 *Capturas sugeridas para este documento:*
> - `docs/assets/sistemas/dragon-vuelo.gif` — transición suelo/vuelo
> - `docs/assets/sistemas/combate-turnos.gif` — secuencia completa de combate
> - `docs/assets/sistemas/captura-dragon.gif` — animación de captura exitosa
