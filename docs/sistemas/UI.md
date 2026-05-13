# 🖥️ Sistema de UI, Inventario, Tiendas y Cocina — Eteria World

> Documentación técnica del sistema de interfaz de usuario, gestión de inventario, tiendas y sistema de cocina. Toda la UI del juego está construida con **Unity UI Toolkit** (UIElements) en lugar del sistema Canvas tradicional, lo que permite mayor flexibilidad y rendimiento.

**Scripts involucrados:** `UIscript1.cs` · `uiscript2.cs` · `uiscript3.cs` · `UIscript8.cs` · `MapController.cs` · `CCE.cs` · `CTM.cs` · `CF.cs` · `CFM.cs` · `FadeBackground.cs` · `Easing.cs`

---

## Índice

1. [Arquitectura general de UI](#1-arquitectura-general-de-ui)
2. [Sistema de pantallas — UIscript1](#2-sistema-de-pantallas--uiscript1)
3. [Sistema de inventario — uiscript2](#3-sistema-de-inventario--uiscript2)
4. [UI del jugador — uiscript3](#4-ui-del-jugador--uiscript3)
5. [Sistema de tiendas — CCE + CTM](#5-sistema-de-tiendas--cce--ctm)
6. [Sistema de cocina — CF + CFM](#6-sistema-de-cocina--cf--cfm)
7. [Mapa interactivo — MapController](#7-mapa-interactivo--mapcontroller)
8. [Sistema de ajustes — UIscript8](#8-sistema-de-ajustes--uiscript8)
9. [Efectos de transición — FadeBackground + Easing](#9-efectos-de-transición--fadebackground--easing)
10. [Canvas mixto 3D y UI](#10-canvas-mixto-3d-y-ui)

---

## 1. Arquitectura general de UI

El sistema de UI usa múltiples `UIDocument` independientes en lugar de un Canvas único. Cada pantalla es un documento separado que se activa o desactiva según el estado del juego. `UIscript1` actúa como controlador central que coordina todos los documentos.

```mermaid
graph TD
    UI1[UIscript1\nControlador central]

    UI1 --> UII[UIi\nInventario]
    UI1 --> UIP[UIp\nHUD del jugador]
    UI1 --> UIT[UIT1\nTiendas]
    UI1 --> UIF[UIF\nFogata / Cocina]
    UI1 --> UIDVD[UIDVD\nCombate DVD]
    UI1 --> UIM[UIM\nMenú principal]
    UI1 --> UIA[UIA\nAjustes]
    UI1 --> LOAD[LoadScreen\nPantalla de carga]
    UI1 --> START[StartLoadScreen\nPantalla de inicio]

    UII --> US2[uiscript2\nInventario]
    UIP --> US3[uiscript3\nHUD base]
    UIT --> CCE[CCE\nTienda individual]
    UIT --> CTM[CTM\nGestor de tiendas]
    UIF --> CFM[CFM\nGestor de fogatas]
    UIA --> US8[UIscript8\nAjustes]
```

**Principio de funcionamiento:**
Ninguna pantalla se destruye ni se crea en runtime. Todas existen siempre en memoria y se muestran u ocultan cambiando `DisplayStyle.Flex / None`. Esto elimina el costo de instanciado y garantiza transiciones instantáneas.

---

## 2. Sistema de pantallas — UIscript1

`UIscript1` es el director de orquesta de toda la UI. Gestiona qué pantalla es visible en cada momento y coordina la entrada al juego.

```mermaid
flowchart TD
    A([Inicio del juego]) --> B[OnEnable:\nObtener referencias\na todos los UIDocuments]
    B --> C[Ocultar todas las\npantallas excepto UIM]
    C --> D[Suscribir eventos\na botones]
    D --> E([Menú principal visible])

    E --> F{Jugador presiona\nJugar}
    F --> G[entraraljuego]
    G --> H[FadeBackground\noscurecer 2s]
    H --> I{¿Primera vez?}
    I -->|Sí| J[Activar cinemática\nde introducción]
    I -->|No| K[Cargar estado\nguardado de misiones]
    J & K --> L[db.playerentro = true]
    L --> M([HUD del jugador visible])

    M --> N{Evento de UI}
    N -->|Abrir inventario| O[OnClicked:\nUIP → UII\nTimeScale = 0]
    N -->|Abrir tienda| P[Tiendadisplay2:\nUIP → UIT]
    N -->|Abrir cocina| Q[Tiendadisplay1:\nUIP → UIF]
    N -->|Abrir combate| R[DisplayAtaque:\nUIP → UIDVD]
    N -->|Cerrar cualquier cosa| S[OnClicked:\nVolver a UIP]
```

**Función `OnClicked` — el núcleo del sistema:**
```mermaid
flowchart LR
    A([OnClicked llamado]) --> B[Ocultar root2\nDisplayStyle.None]
    B --> C[Mostrar root1\nDisplayStyle.Flex]
    C --> D[BringToFront]
    D --> E{pausa?}
    E -->|Sí| F[TimeScale = 0]
    E -->|No| G[TimeScale = 1]
    F & G --> H{cameras?}
    H -->|Sí| I[Activar cámaras\nde dragones]
    H -->|No| J[Desactivar cámaras]
    I & J --> K{inventarioA?}
    K -->|Sí| L[db.InventarioAbierto = abrirI]
```

**Sistema de escalado por dispositivo (`metaquerist`):**
Al iniciar, detecta el DPI y tamaño de pantalla para escalar todos los textos de la UI proporcionalmente. Distingue entre móvil, laptop y monitor grande, ajustando `fontSize` de cada elemento mediante su `resolvedStyle`.

---

## 3. Sistema de inventario — uiscript2

El inventario muestra armas, armaduras, consumibles, dragones capturados y el mapa. Usa `ScrollView` y elementos generados dinámicamente desde las listas de `DataBase1`.

```mermaid
flowchart TD
    A([Abrir inventario]) --> B[UIscript1.OnClicked\nUIP → UII\nTimeScale = 0]
    B --> C[uiscript2.recargar]
    C --> D[Limpiar elementos\nanteriores del ScrollView]
    D --> E{Pestaña activa}

    E -->|Armas| F[Iterar db.playerArmas\nCrear botón por arma\ncon imagen y nombre]
    E -->|Dragones| G[Iterar db.PlayerDragons\nCrear botón por dragón\ncon miniatura de color]
    E -->|Consumibles| H[Iterar db.objetosConsumibles\nMostrar cantidad]
    E -->|Mapa| I[Activar MapController\nCargar SVG del mundo]

    F & G & H --> J[Asignar callbacks\na cada botón]
    J --> K{Tipo de ítem}
    K -->|Arma| L[Equipar arma\nactualizar PlayerActiveArma]
    K -->|Dragón| M[CMD.Dragonini\nCambiar dragón activo]
    K -->|Consumible| N[Aplicar efecto\nal jugador]
```

**Miniaturas dinámicas de dragones:**
Cada dragón capturado tiene su color único guardado en `DataBase1.PlayerDragons`. Al mostrar el inventario, el ícono del dragón se genera con ese color aplicado como tinte sobre la textura base, manteniendo coherencia visual con el dragón que el jugador vio en el mundo.

---

## 4. UI del jugador — uiscript3

`uiscript3` gestiona el HUD permanente del jugador: barra de vida, stamina, dinero, botón de acción contextual y notificaciones de objetos obtenidos.

```mermaid
flowchart LR
    A[uiscript3] --> B[Barra de vida\nactualizada por daño]
    A --> C[Barra de stamina\nactualizada por sprint/vuelo]
    A --> D[Contador de dinero\ndb.objetosInertes 3 .cantidad]
    A --> E[actionB1\nBotón contextual]
    A --> F[mostrarobj\nNotificación de ítem]
    A --> G[FPS counter\nactivable desde ajustes]

    E --> E1{Contexto activo}
    E1 -->|Cerca de dragón| H[Texto: Montar]
    E1 -->|Cerca de tienda| I[Texto: Hablar]
    E1 -->|Cerca de fogata| J[Texto: Cocinar]
    E1 -->|Cerca de NPC misión| K[Texto: Interactuar]

    F --> F1[Mostrar imagen + cantidad\ndel objeto obtenido]
    F1 --> F2[Animación de entrada\nEasing.AbrirSuave]
    F2 --> F3[Auto-ocultar tras 3s]
```

---

## 5. Sistema de tiendas — CCE + CTM

El sistema de tiendas usa dos capas: `CCE` (tienda individual en el mundo) y `CTM` (gestor central que maneja la UI compartida entre todas las tiendas).

```mermaid
sequenceDiagram
    participant J as Jugador
    participant CCE as CCE (tienda física)
    participant CTM as CTM (gestor UI)
    participant DB as DataBase1
    participant CAM as Cinemachine

    J->>CCE: Entra en trigger
    CCE->>CCE: Mostrar actionB1\ntexto "Hablar"
    CCE->>CCE: StartCoroutine EsperarInteraccion

    J->>CCE: Presiona E
    CCE->>CTM: enviarDatos()\nRango, índice tienda,\ncámaras, referencia helen
    CCE->>CTM: cambiarObjeto(true)
    CTM->>CAM: cameras[0].Priority = 10\nresto Priority = 0
    CTM->>CTM: cargarInfo() según indice_tienda
    CTM-->>J: UI de tienda visible\ncon primer objeto

    loop Navegación de productos
        J->>CTM: cambiarObj0_btn / cambiarObj1_btn
        CTM->>CTM: IndiceActivoObj ++/--
        CTM->>CAM: Cambiar cámara activa
        CTM->>CTM: cargarInfo() → ActualizarUi
    end

    loop Ajustar cantidad
        J->>CTM: AumentarCantidad / DisminuirCantidad
        CTM->>CTM: CantidadPorComprar ++/--
        CTM->>CTM: Actualizar CostoTotal label
    end

    J->>CTM: compprar_btn
    CTM->>DB: Verificar dinero suficiente
    CTM->>DB: objetosInertes[3].cantidad -= costoTotal
    CTM->>DB: Añadir ítem comprado a lista correspondiente
    CTM-->>J: Reset cantidad a 0
```

**Tipos de tienda por `indice_tienda`:**

| Índice | Contenido | Lista en DataBase1 |
|--------|-----------|-------------------|
| 0 | Peces (comida dragones) | `db.pez` |
| 2 | Frutas y consumibles | `db.objetosConsumibles[10-12]` |
| 6 | Hongos | `db.objetosConsumibles[5-9]` |
| 7 | Vegetales | `db.objetosConsumibles[13-16]` |
| 8 | Carne | `db.objetosConsumibles[3]` |

**Cámaras por Cinemachine:**
Cada producto de la tienda tiene su propia `CinemachineVirtualCamera`. Al cambiar de producto, CTM baja la prioridad de todas las cámaras a 0 y sube a 10 solo la cámara del producto activo. La Main Camera sigue automáticamente la de mayor prioridad sin código adicional.

> 📸 *[Insertar GIF navegando entre productos en la tienda con cambio de cámara]*

---

## 6. Sistema de cocina — CF + CFM

El sistema de cocina usa el patrón **Singleton Manager + Instancia individual**: `CFM` es el gestor global (singleton) y cada fogata en el mundo es una instancia de `CF` que se registra en él.

```mermaid
flowchart TD
    A([CF.Start]) --> B[CFM.instancia.Registrar this]
    B --> C([Fogata registrada en lista global])

    D([Jugador entra en trigger CF]) --> E[Mostrar actionB1\ntexto Cocinar]
    E --> F{Jugador presiona E}
    F --> G[CFM.IntentarAbrir this]
    G --> H{¿Otra fogata ya activa?}
    H -->|Sí| I([Rechazar — return false])
    H -->|No| J[fogataActual = this]
    J --> K[db.MoveActive = false\ndb.freeViewActive = false\ndb.CocinaAbierta = true]
    K --> L[Mostrar UI de cocina UIF]
    L --> M[CargarIngredientes\ndesde db.objetosConsumibles]

    M --> N([UI de fogata visible])

    N --> O{Jugador añade ingrediente}
    O --> P{cantidadIngredientes <= 3}
    P -->|No| Q([Máximo 4 ingredientes])
    P -->|Sí| R[elementosids registrar índice]
    R --> S[Instanciar mesh 3D\ndel ingrediente en fogata]
    S --> T[db.objetosConsumibles cantidad--]
    T --> U[Recargar UI ingredientes]

    U --> V{Jugador presiona Cocinar}
    V --> W[Analizar tipos de ingredientes]
    W --> X[Calcular tipoMayor por frecuencia]
    X --> Y{¿Ingrediente único?}
    Y -->|Sí| Z[nombreFinal = nombre + asada]
    Y -->|No| AA[nombreFinal = brocheta de + tipo]
    Z & AA --> AB[cronoA = true\nhumitoCocinado.Play\ndestellitos.Play]

    AB --> AC[Timer 5 segundos\nIngredientes rebotan con Rigidbody]
    AC --> AD[AnimacionFinal]
    AD --> AE[Destruir meshes 3D]
    AE --> AF{¿Ya existe esta receta?}
    AF -->|Sí| AG[db.foodP cantidad++]
    AF -->|No| AH[db.foodP.Add nueva receta]
    AG & AH --> AI[Mostrar resultado en UI 3s]
    AI --> AJ[LimpiarIngredientes]
```

**Nomenclatura de recetas:**
```
1 ingrediente solo    → "[nombre] asada"        ej: "Hongo asado"
2-4 ingredientes      → "brocheta de [tipo]"    ej: "brocheta de hongos rauda"
El tipo se determina por cuál tipo de ingrediente aparece más veces
```

**Efectos visuales durante cocción:**
Los meshes 3D de los ingredientes tienen `Rigidbody`. Durante los 5 segundos de cocción, cada 0.2 segundos se aplica un `AddForce` con valores aleatorios pequeños, haciendo que los ingredientes reboten dentro de la fogata de forma orgánica.

> 📸 *[Insertar GIF del proceso de cocción mostrando los ingredientes 3D y el efecto de partículas]*

---

## 7. Mapa interactivo — MapController

`MapController` implementa un mapa SVG interactivo con zoom, pan por mouse y navegación completa por gamepad, sin depender de ninguna librería externa.

```mermaid
flowchart TD
    A([MapController constructor]) --> B[Registrar callbacks\nde mouse y geometry]
    B --> C[GeometryChangedEvent\n→ guardar tamaño base]

    D([Mouse en el mapa]) --> E{Tipo de evento}
    E -->|MouseDown| F[_isDragging = true\nguardar dragStart + offsetActual]
    E -->|MouseMove| G{_isDragging?}
    G -->|Sí| H[delta = posActual - dragStart\n_offset = offsetInicial + delta\nClampOffset → ApplyTransform]
    E -->|WheelEvent| I[factor zoom in/out\nnewScale = Clamp scale×factor\nZoom centrado en cursor\nClampOffset → ApplyTransform]

    J([Gamepad conectado]) --> K{_gamepadNavActive?}
    K -->|No| L{Click en mapa}
    L --> M[EnterGamepadNav\n_gamepadNavActive = true]
    K -->|Sí| N{Botón East Circle}
    N -->|Presionado| O[ExitGamepadNav]
    K -->|Sí| P[Left stick → PAN\n600px/s a deflexión máxima\ndeadzone 0.15]
    K -->|Sí| Q[R2 → Zoom in\nL2 → Zoom out\ncentrado en contenedor]

    H & I & P & Q --> R[ApplyTransform:\nmapSvg.style.width = baseWidth × scale\nmapSvg.style.marginLeft = offset.x]
```

**ClampOffset — límites del mapa:**
Impide que el usuario mueva el mapa más allá de sus bordes. Calcula el rango válido de offset comparando el tamaño del mapa escalado contra el tamaño del contenedor, asegurando que siempre haya contenido visible.

```
offsetX mínimo = contW - mapW    (borde derecho del mapa alineado con borde derecho del contenedor)
offsetX máximo = 0               (borde izquierdo del mapa alineado con borde izquierdo del contenedor)
```

> 📸 *[Insertar GIF del mapa SVG con zoom y pan mostrando las regiones de Eteria]*

---

## 8. Sistema de ajustes — UIscript8

Menú de ajustes con 4 secciones: General, Sonido, Controles y Gráficos. Cada sección es un `VisualElement` que se muestra u oculta al cambiar de pestaña.

```mermaid
flowchart LR
    A[UIscript8] --> B[Menú 0\nGeneral]
    A --> C[Menú 1\nSonido]
    A --> D[Menú 2\nControles]
    A --> E[Menú 3\nGráficos]

    C --> C1[Slider volumen principal\ndb.PLOS]
    C --> C2[Slider volumen ambiental\ndb.ALOS]
    C --> C3[Slider música\ndb.MLOS]
    C --> C4[Slider efectos\ndb.ELOS]

    D --> D1[Slider sensibilidad X\nLookCamera.sensibilidadX]
    D --> D2[Slider sensibilidad Y\nLookCamera.sensibilidadY]
    D --> D3[Toggle invertir eje Y\nLookCamera.invertirY]
    D --> D4[Toggle mostrar FPS\nuiscript3.ActivateFPS]

    E --> E1[Slider calidad gráfica\ncameraReference.farClipPlane\n50 a 4050 unidades]

    A --> F[Botón restablecer\nSaveSystem.restablecer_db]
```

Los valores se leen cada segundo desde una corrutina en lugar de en cada frame, reduciendo el costo de actualización de parámetros de audio y cámara.

---

## 9. Efectos de transición — FadeBackground + Easing

### FadeBackground
Fade a negro y de vuelta usando una `Image` UGUI con alpha animado. Se usa en: entrada al juego, inicio de combate, carga de escena.

```mermaid
flowchart LR
    A([Oscurecimiento oscurecer duracion]) --> B[alphaInicial = color.a actual]
    B --> C{oscurecer?}
    C -->|Sí| D[alphaFinal = 1]
    C -->|No| E[alphaFinal = 0]
    D & E --> F[Loop durante duracion segundos]
    F --> G[t = tiempo/duracion\nalpha = Lerp alphaInicial alphaFinal t\nbackgroundFade.color = RGBA 0,0,0,alpha]
    G --> H([Color final aplicado])
```

### Easing
Librería estática de funciones de interpolación usada en animaciones de UI y transiciones de cámara.

| Función | Uso en el juego |
|---------|----------------|
| `InOutSine` | Rotación del dragón enemigo al iniciar combate |
| `BackOut` | Rotación de cámara al iniciar combate (llega rápido, asienta suave) |
| `AbrirSuave` | Aparición de pantallas UI con animación de entrada |

---

## 10. Canvas mixto 3D y UI

Eteria World usa simultáneamente dos sistemas de renderizado de UI:

```mermaid
graph TD
    subgraph "Espacio de pantalla (Screen Space)"
        UITk[UI Toolkit\nUIDocument\nInventario, HUD, Tiendas,\nCocina, Ajustes, Mapa]
    end

    subgraph "Espacio de mundo (World Space)"
        TXT[TextMeshPro\ntextoNivel en dragones\nnombres flotantes]
        PART[Sistemas de partículas\nVFX Graph\nfogata, antorchas, niebla]
        MESH[Meshes 3D de ingredientes\ninstanciados en fogata\ncon Rigidbody]
    end

    subgraph "Cámaras Cinemachine"
        VCAM[Virtual Cameras\npor producto de tienda\npor cinemática]
        MCAM[Main Camera\nsigue mayor prioridad\nautomáticamente]
        VCAM --> MCAM
    end

    UITk -->|Superpuesto sobre| MCAM
    MCAM -->|Renderiza| TXT
    MCAM -->|Renderiza| PART
    MCAM -->|Renderiza| MESH
```

**Por qué UI Toolkit en lugar de Canvas:**
Unity Canvas genera draw calls por cada elemento gráfico visible. UI Toolkit agrupa los elementos del mismo documento en una sola pasada de renderizado. Con la cantidad de elementos UI del juego (HUD, inventario, tiendas, ajustes), esto representa una reducción significativa de draw calls solo en la capa de interfaz.

Los elementos 3D que coexisten con la UI (ingredientes en fogata, niveles de dragones, partículas) se renderizan en espacio de mundo y se ven "a través" de la cámara principal, creando la ilusión de UI integrada con el mundo sin costo adicional de Canvas en World Space.

> 📸 *[Insertar captura mostrando el HUD sobre el mundo 3D con partículas de fogata visibles al mismo tiempo]*

---

> 📸 *Capturas sugeridas:*
> - `docs/assets/sistemas/inventario-dragones.png` — inventario con miniaturas de color
> - `docs/assets/sistemas/tienda-camaras.gif` — cambio de cámara al navegar productos
> - `docs/assets/sistemas/cocina-proceso.gif` — ingredientes rebotando en fogata
> - `docs/assets/sistemas/mapa-svg.gif` — zoom y pan del mapa interactivo
