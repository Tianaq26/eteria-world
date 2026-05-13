# 🎮 Sistema de Personaje y Cámara — Eteria World

> Documentación técnica del control del personaje, sistema de movimiento, vuelo en dragón y cámara en tercera/primera persona con colisión. Todos los sistemas soportan teclado, mouse, gamepad (DualSense) y controles táctiles móviles desde el mismo código.

**Scripts involucrados:** `movimiento2.cs` · `LookCamera.cs`

---

## Índice

1. [Arquitectura general](#1-arquitectura-general)
2. [Sistema de movimiento — movimiento2](#2-sistema-de-movimiento--movimiento2)
3. [Máquina de estados de movimiento](#3-máquina-de-estados-de-movimiento)
4. [Sistema de vuelo en dragón](#4-sistema-de-vuelo-en-dragón)
5. [Sistema de cámara — LookCamera](#5-sistema-de-cámara--lookcamera)
6. [Modo primera persona](#6-modo-primera-persona)
7. [Colisión de cámara](#7-colisión-de-cámara)
8. [Soporte multi-plataforma](#8-soporte-multi-plataforma)

---

## 1. Arquitectura general

```mermaid
graph TD
    INPUT[Input\nTeclado / Mouse\nGamepad / Joystick táctil]

    INPUT --> M2[movimiento2\nControl del personaje]
    INPUT --> LC[LookCamera\nControl de cámara]

    M2 -->|anim1| ANIM[Animator\nPersonaje o Dragón]
    M2 -->|CharacterController| CC[CharacterController\nFísica de movimiento]
    M2 -->|moveD| CD2[CD\nDragón activo]

    LC -->|player Transform| M2
    LC -->|SphereCast| COL[Colisión de cámara]
    LC -->|primeraPersona| FP[Modo primera persona]

    DB[(DataBase1)] -->|MoveActive| M2
    DB -->|freeViewActive| LC
    DB -->|dragonAc| M2
    DB -->|volar| M2
```

---

## 2. Sistema de movimiento — movimiento2

`movimiento2` maneja todo el movimiento del personaje: caminar, correr, saltar, agacharse y volar en dragón. Un solo script cubre todos los estados posibles del jugador.

```mermaid
flowchart TD
    A([Update movimiento2]) --> B{db.MoveActive?}
    B -->|No| Z([Aplicar gravedad\nsin input])
    B -->|Sí| C{db.Celular?}
    C -->|Sí| D[Input desde\nJoystick táctil]
    C -->|No| E{Gamepad conectado?}
    E -->|Sí| F[Left stick\ndel DualSense]
    E -->|No| G[WASD / Flechas]

    D & F & G --> H[Calcular dirección\nrelativa a cámara]
    H --> I{moveD — montado\nen dragón?}

    I -->|Sí| J[ControlarDragon]
    I -->|No| K{¿Hay input?}

    K -->|No| L[Idle\nanim walk=false\nanim run=false]
    K -->|Sí| M{Shift / L1\npresionado?}
    M -->|Sí| N[Correr\nvelocidad × 2\nanim run=true]
    M -->|No| O[Caminar\nvelocidad base\nanim walk=true]

    N & O --> P[CharacterController.Move\ndirección × velocidad × deltaTime]
    P --> Q{isGrounded?}
    Q -->|Sí| R[velocidadY = -1\npuedeSaltar = true]
    Q -->|No| S[velocidadY -= gravedad\n× deltaTime]

    R --> T{Espacio / Cruz\npresionado?}
    T -->|Sí| U[velocidadY = fuerzaSalto\nanim jump=true]
```

**Dirección relativa a cámara:**
El movimiento no es absoluto (eje Z del mundo) sino relativo a la orientación de la cámara. Si la cámara mira al norte, W mueve al norte. Si la cámara mira al este, W mueve al este.

```
forward = Vector3(cameraForward.x, 0, cameraForward.z).normalized
right   = Vector3(cameraRight.x, 0, cameraRight.z).normalized
dirección = forward × inputV + right × inputH
```

---

## 3. Máquina de estados de movimiento

```mermaid
stateDiagram-v2
    [*] --> Idle : Inicio

    Idle --> Caminando : Input WASD\ny MoveActive

    Caminando --> Corriendo : Shift / L1\nmantener presionado

    Corriendo --> Caminando : Soltar Shift / L1

    Caminando --> Saltando : Espacio / Cruz\nisGrounded

    Saltando --> Cayendo : velocidadY < 0

    Cayendo --> Idle : isGrounded

    Idle --> MontandoDragón : CD detecta E\nmoveD = true

    MontandoDragón --> VolandoDragón : altura > umbral suelo\ndb.volar = true

    VolandoDragón --> MontandoDragón : altura <= umbral suelo\ndb.volar = false

    MontandoDragón --> Idle : E para desmontar\nmoveD = false

    Idle --> Bloqueado : db.MoveActive = false\ncinemática o diálogo

    Bloqueado --> Idle : db.MoveActive = true

    note right of VolandoDragón
        Input vertical:
        W/S → ascender/descender
        A/D → rotar en Y
        Velocidad base del dragón
        desde db.PlayerDragons stats
    end note

    note right of MontandoDragón
        CharacterController.height = ccheight
        anim1 = Animator del dragón
        transform.parent = dragón
    end note
```

---

## 4. Sistema de vuelo en dragón

Cuando el jugador monta un dragón, `movimiento2` toma control del dragón mediante la referencia `anim1` que apunta al `Animator` del dragón (asignada por `CMD`). El personaje del jugador pasa a ser pasajero.

```mermaid
flowchart TD
    A([ControlarDragon llamado]) --> B[Leer input\nvertical y horizontal]
    B --> C[Calcular dirección\nrelativa a cámara]
    C --> D[Mover dragón:\nCharacterController.Move\ndirección × velocidadDragonVueloBase]

    D --> E[Rotar dragón\nhacia dirección de movimiento\nSlerp suave]

    E --> F{Raycast hacia abajo\nhasta suelo}
    F -->|Distancia > umbral| G[db.volar = true\nanim fly=true\nanim walk=false]
    F -->|Distancia <= umbral| H[db.volar = false\nanim fly=false]

    H --> I{¿Hay input\nhorizontal?}
    I -->|Sí| J[anim walk=true]
    I -->|No| K[anim walk=false\nIdle del dragón]

    G --> L{Input ascender\nW o stick up}
    L -->|Sí| M[velocidadY += fuerzaAscenso]
    G --> N{Input descender\nS o stick down}
    N -->|Sí| O[velocidadY -= fuerzaDescenso]

    M & O & K --> P[Aplicar velocidadY\nal CharacterController]

    Q([ReiniciarVuelo]) --> R[velocidadY = 0\nEstado limpio al montar]
```

**Velocidad del dragón desde stats:**
```csharp
m1.velocidadDragonVueloBase = db.PlayerDragons[dragonIndex].stat.velocidadMovimiento;
```
Cada dragón tiene su propia velocidad de vuelo calculada por `StatsUt.calculoVelocidadMovimiento`, que suma velocidad base + nivel + stat de velocidad. Un dragón de mayor nivel es notablemente más rápido.

**VFX de alas:**
Los efectos de partículas `flytrail` en las alas del dragón se activan solo durante el vuelo. Se buscan recursivamente en la jerarquía del dragón por nombre con `Utilidades.BuscarPorNombreRecursivo` al montar, y se activan/desactivan según `db.volar`.

---

## 5. Sistema de cámara — LookCamera

`LookCamera` implementa una cámara en tercera persona con rotación libre, límites verticales, suavizado de distancia y colisión física con el entorno.

```mermaid
flowchart TD
    A([LateUpdate LookCamera]) --> B{TimeScale == 0\no player == null\no !freeViewActive}
    B -->|Sí| Z([Return])
    B -->|No| C{Fuente de input}

    C -->|Gamepad right stick| D[mouseX = stick.x × sensibilidad\nmouseY = stick.y × sensibilidad]
    C -->|Móvil joystick| E[mouseX = joystick.Horizontal\nmouseY = joystick.Vertical]
    C -->|Mouse| F[Input.GetAxis Mouse X/Y\n× sensibilidad]

    D & E & F --> G[rotX += mouseX\nrotY += invertirY ? mouseY : -mouseY]

    G --> H{primeraPersona?}
    H -->|No| I[rotY = Clamp rotY -30 60\nLímites tercera persona]
    H -->|Sí| J[rotY = Clamp rotY -60 60\nLímites primera persona\nControl horizontal limitado]

    I --> K[Quaternion rotacion =\nEuler rotY rotX 0]
    K --> L[Calcular posición deseada\ndetrás del jugador]
    L --> M[SphereCast hacia posición deseada]
    M --> N{¿Colisión?}
    N -->|Sí| O[distanciaObstaculo =\nMax distanciaMin\nhit.distance - offsetColision]
    N -->|No| P[distanciaObstaculo = distancia]

    O & P --> Q{distanciaObstaculo\n< distanciaActual}
    Q -->|Sí acercar| R[Lerp rápido\nsuavizado × 10]
    Q -->|No alejar| S[Lerp lento\nsuavizado normal]

    R & S --> T[transform.position =\npuntoOrigen + rotacion × 0,0,-distanciaActual]
    T --> U[transform.LookAt\nplayer.position + up × 1.5]
```

---

## 6. Modo primera persona

La cámara tiene un modo primera persona usado al inicio del juego (jugador en cama) y al entrar en interiores. Cuando se activa, la distancia de cámara va a 0 y la cámara se posiciona justo delante de los ojos del personaje.

```mermaid
flowchart TD
    A([ActivarPrimeraPersona estado]) --> B{estado?}
    B -->|true| C[primeraPersona = true\ncuerpoY = player.eulerAngles.y\nrotX = cuerpoY\ndistanciaActual = 0]
    B -->|false| D[primeraPersona = false\ndistanciaActual = distancia]

    C --> E[En LateUpdate primera persona:]
    E --> F[puntoOrigen = player.pos\n+ up × alturaPrimeraPersona]
    F --> G[offsetAdelante = rotacion\n× 0,0,offsetPrimeraPersona]
    G --> H[transform.position = origen + offset]
    H --> I[transform.rotation = rotacion\nCámara mira hacia adelante]

    J[Control horizontal limitado\nprimera persona] --> K[diferencia = DeltaAngle\ncuerpoY rotX]
    K --> L[diferencia = Clamp -80 80]
    L --> M{Abs diferencia\n>= 79?}
    M -->|Sí cerca del límite| N[Rotar cuerpo del jugador\nhacia la cámara\nLerpAngle suave]
    M -->|No| O[Solo rotar cámara\nsin mover cuerpo]
```

**Límites horizontales en primera persona:**
En tercera persona la cámara puede rotar 360° horizontalmente. En primera persona se limita a ±80° respecto a la orientación del cuerpo. Cuando el jugador llega cerca del límite, el cuerpo del personaje rota gradualmente para seguir la cámara, permitiendo girar más sin romper la ilusión de primera persona.

---

## 7. Colisión de cámara

La colisión evita que la cámara atraviese paredes o geometría. Usa `SphereCast` en lugar de `Raycast` para detectar obstáculos con un margen de radio, evitando que la cámara quede justo rozando la pared.

```mermaid
flowchart LR
    A[puntoOrigen\ncabeza del jugador] -->|SphereCast\nradio 0.4| B{¿Hit?}
    B -->|No| C[distanciaObstaculo\n= distancia máxima]
    B -->|Sí| D{¿Es trigger?\no está en ignorarObjetos?}
    D -->|Sí ignorar| C
    D -->|No es obstáculo| E[distanciaObstaculo\n= Max distanciaMin\nhit.distance - offsetColision]

    C & E --> F{distanciaObstaculo\n< distanciaActual}
    F -->|Acercar| G[Lerp velocidad × 10\nRápido — evita clip instantáneo]
    F -->|Alejar| H[Lerp velocidad normal\nSuave — no salta bruscamente]
```

**`ignorarObjetos`:**
Array de Transforms que el SphereCast ignora. Al montar un dragón se añaden tanto el jugador como el dragón a esta lista, evitando que la cámara (ahora alejada 30 unidades) colisione con el propio cuerpo del dragón.

```csharp
// Al montar:
l1.ignorarObjetos = new Transform[] { player.transform.root, transform.root };
// Al desmontar:
l1.ignorarObjetos = new Transform[] { player.transform.root };
```

**Velocidad asimétrica de suavizado:**
Acercarse es instantáneo (×10) para evitar que la cámara atraviese paredes aunque sea un frame. Alejarse es suave para que la cámara no "salte" bruscamente al salir de un espacio estrecho.

---

## 8. Soporte multi-plataforma

Todo el sistema de input está encapsulado en condiciones que detectan la plataforma activa en runtime, sin compilaciones separadas.

```mermaid
graph TD
    A{Plataforma detectada} -->|db.Celular == true| B[Joystick táctil\nJoystick.Horizontal/Vertical]
    A -->|Gamepad.current != null\ny stick.magnitude > 0.1| C[DualSense / Xbox\nleftStick + rightStick]
    A -->|Default| D[Teclado + Mouse\nInput.GetAxis]

    B & C & D --> E[Misma lógica de\nmovimiento y cámara]

    F[Acciones binarias\nsaltar, montar, interactuar] --> G{Plataforma}
    G -->|PC| H[KeyCode.E\nKeyCode.Space]
    G -->|Gamepad| I[buttonEast Cross\nbuttonSouth]
    G -->|Móvil| J[db.btnpresed\ndb.btnpresed2\ndesde botones UI]
```

**`db.btnpresed` — botones táctiles:**
En móvil no hay teclado, por lo que los botones de acción de la UI táctil escriben en flags de `DataBase1` (`btnpresed`, `btnpresed2`). Los scripts de gameplay leen estos flags junto con el input de teclado/gamepad en el mismo `if`, unificando la lógica sin duplicar código.

**Sensibilidad configurable:**
`LookCamera.sensibilidadX` y `sensibilidadY` son variables estáticas. `UIscript8` las modifica directamente desde los sliders de ajustes sin necesidad de referencias entre objetos:
```csharp
LookCamera.sensibilidadX = sensibilidades[0].value;
LookCamera.sensibilidadY = sensibilidades[1].value;
LookCamera.invertirY = ejeY.value;
```

---

> 📸 *Capturas sugeridas:*
> - `docs/assets/sistemas/tercera-persona.gif` — movimiento en tercera persona con colisión de cámara
> - `docs/assets/sistemas/primera-persona.gif` — transición a primera persona al entrar en interior
> - `docs/assets/sistemas/vuelo-dragon.gif` — control de vuelo con detección de altura
> - `docs/assets/sistemas/colision-camara.gif` — cámara esquivando paredes en espacio estrecho
