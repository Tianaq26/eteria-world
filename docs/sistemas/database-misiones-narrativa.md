# 🗃️ DataBase1, Misiones y Sistema Narrativo — Eteria World

> Documentación técnica del singleton central del juego, el sistema de misiones encadenadas con herencia de clases, el gestor de cinemáticas y el sistema de narrativa. Estos sistemas forman la columna vertebral arquitectural del proyecto.

**Scripts involucrados:** `DataBase1.cs` · `GDM.cs` · `Mision0.cs` · `Mision1-5.cs` · `Manejo_Timeline.cs` · `introduccion.cs` · `raycast-line.cs`

---

## Índice

1. [DataBase1 — Singleton central](#1-database1--singleton-central)
2. [Sistema de guardado](#2-sistema-de-guardado)
3. [Gestor de misiones — GDM](#3-gestor-de-misiones--gdm)
4. [Arquitectura de misiones — Mision0 + herencia](#4-arquitectura-de-misiones--mision0--herencia)
5. [Cadena narrativa completa — Misiones 1 a 5](#5-cadena-narrativa-completa--misiones-1-a-5)
6. [Sistema de cinemáticas — Manejo_Timeline + introduccion](#6-sistema-de-cinemáticas--manejo_timeline--introduccion)
7. [Sistema de waypoints A* — raycast-line](#7-sistema-de-waypoints-a--raycast-line)

---

## 1. DataBase1 — Singleton central

`DataBase1` es el punto de acceso global a todo el estado del juego. Ningún sistema guarda estado propio relevante — todo pasa por aquí. Esto garantiza que cualquier script puede leer o modificar el estado del juego sin referencias directas entre sí.

```mermaid
graph TD
    subgraph "Referencias de escena"
        PR[playerReference]
        CAM[cameraReference]
        ANIM[PlayerAnimator1]
        MT2[MT — ManejoTimeline]
        CDN2[clima_reference — CCL]
        FOG2[PivoteFog]
        GDM2[gestorDeMisiones]
    end

    subgraph "Estado del jugador"
        MA[MoveActive]
        FVA[freeViewActive]
        DA[dragonAc — montado]
        VOL[volar]
        PA[peleaActiva]
        INV[InventarioAbierto]
        COC[CocinaAbierta]
        TIE[tiendaAbierta]
    end

    subgraph "Inventario del jugador"
        PD[PlayerDragons\nList DragonData]
        PAR[playerArmas\nList Arma]
        PAA[PlayerActiveArma]
        PEZ[pez\nList PezData]
        OBJ[objetosConsumibles]
        OBI[objetosInertes]
        FOOD[foodP\nList foodsP]
    end

    subgraph "Estado del mundo"
        NIV[nivelDelJuego]
        HORA[hora — CDN]
        NMIS[MisionActual]
        NTASK[nivelMisionTarea]
        CARGA[carga]
        FT[FirstTime]
        PE[playerentro]
    end

    subgraph "Dragones del mundo"
        DAC[dragonactivo\nGameObject]
        DRAC[dragones\nList DragonBase]
        TEX[textura\nList Texture2D]
        PAT[patron1... Material]
    end

    DB[(DataBase1\nInstancia)] --> PR & CAM & ANIM & MT2 & CDN2 & FOG2 & GDM2
    DB --> MA & FVA & DA & VOL & PA & INV & COC & TIE
    DB --> PD & PAR & PAA & PEZ & OBJ & OBI & FOOD
    DB --> NIV & HORA & NMIS & NTASK & CARGA & FT & PE
    DB --> DAC & DRAC & TEX & PAT
```

**Por qué un singleton y no ScriptableObjects:**
El estado del juego es altamente dinámico y muchos sistemas necesitan leer y escribir el mismo dato en el mismo frame. Un singleton en memoria es la opción más directa y performante para este caso. Los ScriptableObjects son ideales para datos estáticos de diseño, no para estado en tiempo real.

**Patrón de acceso:**
```csharp
// Cualquier script accede así, sin referencias en Inspector:
DataBase1 db = DataBase1.Instancia;
db.MoveActive = false;
db.PlayerDragons[0].vida = 100;
```

---

## 2. Sistema de guardado

El juego guarda y carga el estado completo mediante `SaveSystem`, que serializa `DataBase1` a JSON en disco.

```mermaid
flowchart TD
    A([Evento de guardado]) --> B[SaveSystem.guardar_db]
    B --> C[Serializar DataBase1\na JSON]
    C --> D[Escribir en\nApplication.persistentDataPath]

    E([Inicio del juego]) --> F{¿Existe archivo\nde guardado?}
    F -->|No| G[db.FirstTime = true\nComenzar desde cero]
    F -->|Sí| H[SaveSystem.cargar_db]
    H --> I[Deserializar JSON\na DataBase1]
    I --> J[Restaurar:\n- MisionActual\n- PlayerDragons\n- Inventario\n- nivelDelJuego]
    J --> K[db.FirstTime = false\nCargar estado guardado]

    L([SaveSystem.restablecer_db]) --> M{¿Confirmación?}
    M -->|Sí| N[Borrar archivo JSON\nReiniciar DataBase1]
    M -->|No| O([Cancelar])
```

**Datos que se persisten:**

| Categoría | Datos guardados |
|-----------|----------------|
| Progreso | `MisionActual`, `nivelMisionTarea`, `nivelDelJuego` |
| Dragones | `PlayerDragons` — vida, stats, color, textura de cada dragón |
| Inventario | `playerArmas`, `pez`, `objetosConsumibles`, `objetosInertes`, `foodP` |
| Estado | `FirstTime`, `hierro_conseguido` |

---

## 3. Gestor de misiones — GDM

`GDM` orquesta la cadena de misiones de forma lineal. Espera a que cada misión termine antes de iniciar la siguiente, usando el flag `finalizada` de cada instancia.

```mermaid
flowchart TD
    A([Update GDM]) --> B{gestorDeMisionesIniciado?}
    B -->|No| C{db.playerentro?}
    C -->|No| D([Esperar])
    C -->|Sí| E[StartCoroutine\nmisiones misionActual .iniciarEventos]
    E --> F[gestorDeMisionesIniciado = true]

    B & F --> G{misionActual\n== misiones.Length}
    G -->|Sí| H([Todas las misiones\ncompletadas])
    G -->|No| I{misiones misionActual\n.finalizada?}
    I -->|No| D
    I -->|Sí| J[misionActual++]
    J --> K{misionActual\n< misiones.Length}
    K -->|Sí| L[StartCoroutine\nnueva misión.iniciarEventos]
    K -->|No| H
```

**Por qué corrutinas y no Update para las misiones:**
Cada misión tiene una secuencia de eventos con tiempos específicos entre diálogos, cinemáticas y condiciones. Implementarlo en Update requeriría máquinas de estado explícitas con muchos flags. Con corrutinas, el flujo se lee secuencialmente como un guion: `yield return dialogo` → `yield return cinematica` → `yield return condicion`, exactamente en el orden que ocurre en el juego.

---

## 4. Arquitectura de misiones — Mision0 + herencia

`Mision0` es la clase base que provee todos los servicios que una misión puede necesitar. Cada misión concreta hereda de ella y solo implementa su propia secuencia de eventos.

```mermaid
classDiagram
    class Mision0 {
        +bool finalizada
        +UIDocument uip
        +Animator anim
        +bool Dialogos_rapidos
        -Label texto
        -bool interactuo
        -bool adentro
        +Start()
        +OnTriggerEnter()
        +OnTriggerExit()
        +colocar_texto(dialogos, interactivo, duracion)*
        +RotarSoloEnY(jugador, objetivo, duracion)*
        +activar_marcado()
        +iniciarEventos()* virtual
    }

    class Mision1 {
        -Transform puerta
        +iniciarEventos() override
    }

    class Mision2 {
        -PlayableDirector herreria_cinematica
        +bool fin_cinematica
        +iniciarEventos() override
    }

    class Mision3 {
        -PlayableDirector cinematicaTienda
        -CCE referencia_tiendaScript
        +iniciarEventos() override
    }

    class mision4 {
        +iniciarEventos() override
    }

    class mision5 {
        -PlayableDirector cinematica
        -string[][] dialogos_cinematica
        +bool llego_al_santuario
        +int dialogoActual
        +iniciarEventos() override
    }

    Mision0 <|-- Mision1
    Mision0 <|-- Mision2
    Mision0 <|-- Mision3
    Mision0 <|-- mision4
    Mision0 <|-- mision5
```

**`colocar_texto` — sistema de diálogo:**
La función más usada en todo el sistema narrativo. Escribe el texto carácter por carácter (efecto typewriter), activa la animación de hablar del NPC y espera según el modo:

```mermaid
flowchart TD
    A([colocar_texto dialogos interactivo duracion]) --> B{interactivo?}
    B -->|Sí| C[Esperar E o Cross del jugador]
    B -->|No| D[Continuar directo]
    C & D --> E[Para cada diálogo en array]
    E --> F[texto.text = vacío]
    F --> G[anim.SetBool hablar true]
    G --> H[Para cada char en diálogo:\ntexto.text += char\nyield return null]
    H --> I{i == 30 y es espacio}
    I -->|Sí| J[Agregar salto de línea\nautomático]
    I & J --> K{interactivo?}
    K -->|Sí y Dialogos_rapidos| L[WaitForSeconds 3]
    K -->|Sí y !rapidos| M[Esperar Space o Click]
    K -->|No| N[WaitForSeconds duracion]
    L & M & N --> O[texto.text = vacío]
    O --> P{¿Más diálogos?}
    P -->|Sí| E
    P -->|No| Q[db.MoveActive = true]
```

---

## 5. Cadena narrativa completa — Misiones 1 a 5

```mermaid
sequenceDiagram
    participant J as Jugador
    participant GDM as GDM
    participant M1 as Mision1
    participant M2 as Mision2
    participant M3 as Mision3
    participant M4 as mision4
    participant M5 as mision5
    participant DB as DataBase1

    GDM->>M1: iniciarEventos
    M1->>J: Diálogo: Arturo despierta al jugador
    M1->>J: Diálogo: Lista de tareas del día
    M1->>DB: nivelMisionTarea = 1
    M1->>M1: Animar puerta abriéndose 110 frames
    M1-->>GDM: finalizada = true

    GDM->>M2: iniciarEventos
    M2->>J: Diálogo: Herrero saluda
    M2->>M2: herreria_cinematica.Play
    M2->>J: Diálogo: Necesito 4 hierros brutos
    M2->>DB: nivelMisionTarea = 2
    M2->>J: Esperar hierro_conseguido >= 4
    M2->>J: Diálogo: Perfecto, aquí tu espada
    M2->>DB: playerArmas.Add espada de hierro
    M2->>DB: nivelMisionTarea = 3
    M2-->>GDM: finalizada = true

    GDM->>M3: iniciarEventos
    M3->>M3: Esperar jugador entre a tienda
    M3->>M3: cinematicaTienda.Play
    M3->>J: Diálogo: Helen ofrece salmón
    M3->>M3: referencia_tiendaScript.abrirforzado
    M3->>J: Esperar pez[2].cantidad >= 1
    M3->>DB: nivelMisionTarea = 4
    M3-->>GDM: finalizada = true

    GDM->>M4: iniciarEventos
    M4->>J: Esperar 2 hongos recolectados
    M4->>DB: nivelMisionTarea = 5
    M4->>J: Esperar brocheta de hongos rauda en foodP
    M4->>DB: nivelMisionTarea = 6
    M4-->>GDM: finalizada = true

    GDM->>M5: iniciarEventos
    M5->>J: Diálogo: Arturo pregunta si está listo
    M5->>M5: cinematica.Play — viaje al santuario
    M5->>J: 10 diálogos narrativos durante el viaje
    M5->>M5: Activar Arturo en santuario
    M5->>DB: MoveActive = false temporalmente
    M5-->>GDM: finalizada = true
```

**Progreso de `nivelMisionTarea`:**

| Valor | Estado |
|-------|--------|
| 0 | Inicio — sin tareas |
| 1 | Ir a herrería |
| 2 | Recolectar 4 hierros |
| 3 | Comprar salmón |
| 4 | Recolectar hongos |
| 5 | Cocinar brocheta |
| 6 | Ir al santuario |

Este valor se guarda en `DataBase1` y se persiste en el archivo de guardado, permitiendo retomar el progreso exacto de misión al recargar el juego.

---

## 6. Sistema de cinemáticas — Manejo_Timeline + introduccion

### 6.1 Arquitectura de cinemáticas (`Manejo_Timeline.cs`)

```mermaid
classDiagram
    class ManejoTimeline {
        +int CinematicaActual
        +bool activarCinematica
        +GameObject[] cinematicas
        +iniciarCinInicial()
    }

    class cinematica {
        +bool inciar
        +DesactivePlayer()
        +ActivarPlayer()
    }

    class introduccion {
        -PlayableDirector timelineDirector
        -string[] dialogos
        +int dialogoActual
        +bool finalizo
        +LookCamera camaraPlayer
        +LateUpdate()
    }

    cinematica <|-- introduccion : hereda
    ManejoTimeline "1" --> "*" cinematica : gestiona array
```

**Por qué herencia en lugar de switch/if:**
Con herencia, `ManejoTimeline` solo llama `cinematicas[i].GetComponent<cinematica>().inciar = true` sin saber qué tipo de cinemática es. Agregar una nueva cinemática es crear una nueva clase que hereda de `cinematica` e implementar su propia lógica, sin modificar el gestor.

### 6.2 Flujo de la cinemática de introducción

```mermaid
flowchart TD
    A([db.FirstTime == true]) --> B[UIscript1.entraraljuego]
    B --> C[db.MT.activarCinematica = true\ndb.MT.iniciarCinInicial]
    C --> D[introduccion.inciar = true]

    D --> E[LateUpdate detecta inciar]
    E --> F[Mover player a origen 0,0,0]
    F --> G[ui1.activar_UI false\nOcultar HUD]
    G --> H[DesactivePlayer\ndb.MoveActive = false]
    H --> I[sonidoMusica.SetActive false]
    I --> J[timelineDirector.Play\nInicia Timeline Unity]

    J --> K[Timeline ejecuta secuencia]
    K --> L[4 escenarios con\nVirtual Cameras distintas]
    L --> M[Diálogos narrados\ntipo letra a letra]
    M --> N[Dragón Scarn cruza\nel encuadre como transición]

    N --> O{finalizo == true}
    O --> O2{db.MisionActual == 0}
    O2 -->|Sí| P[Mover player a posición\nde inicio del juego]
    P --> Q[ui1.activar_UI true\nMostrar HUD]
    Q --> R[ActivarPlayer\ndb.MoveActive = true]
    R --> S[camaraPlayer.ActivarPrimeraPersona true\nInicio en cama]
    S --> T[gdm.iniciarColdown\nEsperar antes de primera misión]
    T --> U[sonidoMusica.SetActive true]
    U --> V([Jugador en control\ncomienza Mision1])
```

**Sistema de diálogos sincronizados con Timeline:**
La cinemática de introducción tiene diálogos que deben aparecer en momentos específicos del Timeline. En lugar de usar eventos del Timeline directamente, `introduccion.cs` expone la propiedad pública `dialogoActual`. El Timeline modifica este valor en el momento exacto mediante `AnimationTrack`, y `Update` detecta el cambio comparando con `dialogoAnterior`, disparando el efecto typewriter.

```mermaid
flowchart LR
    TL[Unity Timeline] -->|Frame específico\nAnimationTrack| DA[dialogoActual = N]
    DA --> UP[Update detecta\ndialogActual != dialogoAnterior]
    UP --> CO[StartCoroutine\ncambiarDialogo dialogos N]
    CO --> TW[Efecto typewriter\ncarácter a carácter]
    TW --> WT[WaitForSeconds 2\nlimpiar texto]
```

**4 escenarios de la introducción:**

| Escenario | Frames | Virtual Camera | Descripción |
|-----------|--------|---------------|-------------|
| Cerro de Chirripó | 0 - 600 | VCam_Chirripó | Vista del mundo desde las alturas |
| Llanura de Ámbar | 600 - 1200 | VCam_Ambar | Dragón Scarn cruza la escena |
| Campo de entrenamiento | 1200 - 1800 | VCam_Campo | Pendiente de desarrollo |
| Llanura de Scarn | 1800 - 2400 | VCam_Scarn | Llegada al santuario |

---

## 7. Sistema de waypoints A* — raycast-line

`SmartWaypointPath` implementa el algoritmo A* sobre una red de waypoints para calcular la ruta más corta entre el jugador y el punto de misión activo. Actualmente en desarrollo, no activo en el juego.

```mermaid
flowchart TD
    A([Start]) --> B[Obtener hijos como Waypoints]
    B --> C[Conectar vecinos\npor distancia <= radioVecino]
    C --> D[Desactivar todos los waypoints]

    E([Update cada 0.25s]) --> F{playerReference\ny punto_mision != null}
    F -->|No| G([Esperar])
    F -->|Sí| H[FindClosestWaypoint\nal jugador → start]
    H --> I[FindClosestWaypoint\nal objetivo → end]
    I --> J[CalcularRutaAStar start end]

    J --> K[Cola abierta: start]
    K --> L[Loop mientras abierta no vacía]
    L --> M[Seleccionar nodo con\nmenor f = g + heurística]
    M --> N{¿Es el destino?}
    N -->|Sí| O[Reconstruir ruta\ndesde padres]
    N -->|No| P[Remover de abierta]
    P --> Q[Para cada vecino:\ntentativeG = g actual +\ndistancia al vecino]
    Q --> R{tentativeG\n< gScore vecino}
    R -->|Sí| S[Actualizar gScore\ngScore vecino = tentativeG\npadres vecino = current\nAñadir a abierta]
    R -->|No| L
    S --> L

    O --> T[Activar solo waypoints\nde la ruta calculada]
    T --> U[Debug.DrawLine\npara visualización]
```

**Heurística usada:** Distancia euclidiana directa al destino (`Vector3.Distance`), que es admisible (nunca sobreestima) en un espacio 3D sin obstáculos entre waypoints.

**Por qué no NavMesh directamente:**
El sistema de waypoints permite mostrar visualmente la ruta al jugador en el mundo como marcadores flotantes, algo que NavMesh no puede hacer de forma nativa sin código adicional. Al activar solo los waypoints de la ruta calculada, el jugador ve una línea de puntos que lo guía al objetivo.

---

> 📸 *Capturas sugeridas:*
> - `docs/assets/sistemas/cinematica-intro.gif` — secuencia de la cinemática de introducción
> - `docs/assets/sistemas/dialogo-typewriter.gif` — efecto typewriter en diálogos de misión
> - `docs/assets/sistemas/mision-progreso.png` — HUD mostrando tarea activa de misión
