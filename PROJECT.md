# RPS Chaos - Documentación de Proyecto

## Propósito
Prototipo de juego multiplayer en Roblox basado en mecánica Rock-Paper-Scissors con combate server-authoritative. 

**Objetivo:** Validar flujo de combate, sincronización de estado y anti-cheat como base para expansión futura a otras mecánicas en Roblox.

---

## Estado Actual
- **Última actualización**: 2026-07-21
- **Etapa**: Prototipo en desarrollo
- **Plataforma**: Roblox Studio (Luau)
- **Versión**: 0.1.0 (MVP inicial)

---

## Contexto Técnico

### Stack Tecnológico
- **Engine**: Roblox + Luau
- **Arquitectura**: Server-Authoritative (lógica crítica validada en servidor)
- **Comunicación**: RemoteEvents (cliente → servidor), ReplicatedStorage (estado compartido)
- **Servicios**: DataStoreService, RunService, Players, StarterPlayer
- **Patrón de Red**: Fire & Wait (cliente envía movimiento, servidor calcula y replica resultado)

### Principios de Diseño
- ✅ **Server-authoritative**: El servidor valida TODOS los movimientos (previene trampas)
- ✅ **Movimientos simultáneos**: Ambos jugadores envían movimiento, servidor calcula al mismo tiempo
- ✅ **Sin client prediction**: Simplicidad > latencia mínima en MVP
- ✅ **Determinístico**: Mismo par de movimientos siempre produce mismo resultado
- ✅ **Timeout global**: Si un jugador no envía movimiento en 10s, pierde automáticamente
- ✅ **Anti-cheat básico**: Servidor rechaza movimientos inválidos/tardíos

---

## Componentes Principales

### 1. Combat System
**Responsabilidad**: Orquestar rondas, validar movimientos y calcular resultados

- **RoundManager** (server)
  - Controla duración de rondas (10s)
  - Maneja timeouts y desconexiones
  - Inicia/cierra rondas automáticamente

- **MoveValidator** (server)
  - Valida que el movimiento sea válido (Rock/Paper/Scissors)
  - Rechaza movimientos tardíos
  - Previene exploits de timing

- **ResultCalculator** (server)
  - Lógica RPS determinística: Rock > Scissors > Paper > Rock
  - Calcula empate si ambos movimientos son iguales
  - Retorna ganador + stats

### 2. Player State Management
**Responsabilidad**: Persistencia de datos y estadísticas de jugador

- **PlayerStateManager** (server)
  - Crea perfil al join, destruye al leave
  - Sincroniza stats: wins, losses, elo/rating
  - Interfaz con DataStore

- **DataStore Integration**
  - Almacena stats entre sesiones (persistencia)
  - Load en join, save al dejar/perder conexión
  - Manejo de errores para fallos de guardar

### 3. Network Replication
**Responsabilidad**: Sincronización y comunicación en tiempo real

- **RemoteEvents**
  - Servidor escucha `PlayerMovement` desde cliente
  - Servidor reenvía `CombatResult` a ambos clientes
  - Servidor envía `TimerTick` cada segundo

- **ReplicatedStorage**
  - Constantes compartidas: timeouts, movimientos válidos, cooldowns
  - Módulos de utilidad (Logger, Constants)

### 4. UI System
**Responsabilidad**: Input y feedback visual

- **UIController** (client)
  - Escucha input del jugador
  - Muestra opciones (Rock/Paper/Scissors)
  - Envía movimiento al servidor

- **ResultDisplay** (client)
  - Muestra ganador/perdedor con animaciones
  - Muestra timer durante selección
  - Feedback visual de timeout/desconexión

### 5. Weapon Systems & Damage Scaling
**Responsabilidad**: Aplicar multiplicadores de daño desde PowerUpsShop a todos los sistemas de armas

#### Arquitectura de Dos Rutas de Daño

Actualmente hay dos caminos independientes de cálculo de daño:

1. **CombatService** (melee integrado)
   - Archivo: `game.ServerScriptService.Services.CombatService`
   - Función: `applyDamage(target, damage, attacker)`
   - Aplica multiplicador: ✅ Sí
   - Origen: `PowerUpsShop:GetDamageMultiplier(attacker)`

2. **Blaster** (arma externa/disparos)
   - Archivo: `game.ServerScriptService.Blaster.Scripts.Blaster`
   - Función: `applyDamage(target, damage, attacker)`
   - Aplica multiplicador: ✅ Sí
   - Origen: `PowerUpsShop:GetDamageMultiplier(attacker)`

#### Patrón para Nuevas Armas Escalables

Cuando implementes una nueva arma o sistema de daño, sigue este patrón:

```lua
-- En el script del arma (weapon script o parte de un sistema)
local function getDamageMultiplier(player)
    local ok, ShopRegistry = pcall(require, game.ServerScriptService.Services.Shops.ShopRegistry)
    if not ok then return 1 end
    
    local shop = ShopRegistry.Get("PowerUps")
    return shop and shop:GetDamageMultiplier(player) or 1
end

-- En la función applyDamage del arma:
local function applyDamage(target, baseDamage, attacker)
    if not target or not attacker then return end
    
    -- Aplicar multiplicador de PowerUps
    local multiplier = getDamageMultiplier(attacker)
    local finalDamage = math.floor(baseDamage * multiplier)
    
    -- Resto de lógica de daño...
    target:TakeDamage(finalDamage)
end
```

#### Notas Técnicas

- **Lazy require**: Usamos `pcall(require, ...)` para evitar circular dependencies
- **Fallback**: Si ShopRegistry no existe o PowerUps no está disponible, retorna multiplicador 1.0 (sin bonus)
- **Escalabilidad**: Mismo patrón se reutiliza en múltiples armas sin duplicación

#### Refactor Futuro (3+ Armas)

Cuando existan 3 o más sistemas de armas, REFACTORIZAR a módulo centralizado:

```lua
-- Crear: game.ServerScriptService.Services.DamageCalculator
local DamageCalculator = {}

function DamageCalculator.applyWithMultiplier(target, baseDamage, attacker)
    local multiplier = getDamageMultiplier(attacker)
    local finalDamage = math.floor(baseDamage * multiplier)
    target:TakeDamage(finalDamage)
end

return DamageCalculator
```

Luego, todas las armas usan: `DamageCalculator.applyWithMultiplier(...)`

**Beneficio**: Evita duplicación del helper, centraliza lógica de multiplicadores, facilita cambios globales.

---

## Flujo de Juego Completo (1v1)

1. **Lobby**: Jugador entra y espera matchmaking
2. **Emparejamiento**: Sistema agrupa 2 jugadores
3. **Inicio de ronda**: Timer de 10s comienza, UI muestra opciones
4. **Selección**: Cliente envía `PlayerMovement` (Rock/Paper/Scissors)
5. **Validación servidor**: MoveValidator rechaza si es tardío/inválido
6. **Cálculo**: Servidor espera ambos movimientos o timeout → ResultCalculator
7. **Replicación**: Servidor envía `CombatResult` a ambos clientes
8. **Visualización**: UI muestra ganador + animación en ambos clientes
9. **Siguiente**: Ronda siguiente automática o regresa al lobby si no hay revancha

### Caso especial - Timeout
- Si jugador no envía movimiento en 10s → cuenta como pérdida automática
- ResultCalculator calcula al instante (no espera al jugador timeout)
- Otro jugador recibe victoria inmediata

---

## Decisiones Arquitectónicas Importantes

| Decisión | Razón | Trade-offs |
|----------|-------|-----------|
| **Server-authoritative** | Previene trampas, una única fuente de verdad | Latencia mínima de validación |
| **Movimientos simultáneos** | Más justo, evita metagaming de timing | Complejidad de sincronización |
| **Timeout en servidor** | Detecta desconexiones, previene AFK | Falsos positivos con lag |
| **Sin client prediction** | MVP simple, fácil de debuggear | Latencia visible al usuario |
| **DataStore para stats** | Persistencia real | Overhead de I/O |
| **Dos rutas de daño → Patrón escalable** | Flexibilidad para múltiples armas | Requiere coordinación de multiplicadores |

---

## Próximos Hitos

### MVP Core (Actual)
- [x] Estructura base de componentes
- [x] Rework CoinUpgrades con tiers escalables
- [x] Conectar PowerUps damage multiplier a CombatService y Blaster
- [x] **Weapons Phase 0**: Foundation (WeaponCatalog, DamageCalculator, cleanup legacy)
- [x] **Weapons Phase 1**: WeaponFactory + Tool migration (ServerStorage.WeaponPrefabs.ak47)
- [ ] **Weapons Phase 2**: Subir 4-5 modelos reales (pistol, rifle, smg, shotgun, sniper)
- [ ] MVP de combate 1v1 funcional (2 jugadores)
- [ ] Testing básico en Studio
- [ ] Anti-cheat básico (validación servidor)

### Fase 2 - Multiplayer
- [ ] Testing con 4+ jugadores simultáneos
- [ ] Matchmaking simple (colas de espera)
- [ ] Persistencia de stats (DataStore)

### Fase 3 - Engagement
- [ ] Leaderboards (top 10, por amigo, etc.)
- [ ] Cosmetics/Skins (opcional post-MVP)
- [ ] Recompensas por racha de victorias

### Fase 4 - Optimización & Escalabilidad
- [ ] Anti-cheat mejorado (detección de patrones)
- [ ] Optimización de latencia (client prediction)
- [ ] Estadísticas avanzadas (elo ranking)
- [ ] Refactor DamageCalculator centralizado (cuando existan 3+ armas)

---

## Dependencias y Conexiones

### Servicios Roblox Usados
- **DataStoreService**: Persistencia de stats entre sesiones
- **RemoteEvents**: Comunicación real-time cliente-servidor
- **RunService**: Loops de timeout, heartbeats
- **Players**: Detección join/leave/disconnect
- **StarterPlayer**: Scripts de entrada de jugador

### Módulos Propios
- `RPS_Constants` (compartido): Enums de movimientos, timeouts
- `Utilities` (compartido): Logger, helper functions
- `CombatSystem` (servidor): Toda lógica de pelea
- `ShopRegistry` (servidor): Lookup de shops (CoinUpgrades, PowerUps)
- `RewardFormulas` (servidor): Lectura dinámica de workspace RewardConfig

---

## ⚠️ INSTRUCCIONES DE MANTENIMIENTO

**Cuando actualices este archivo o RULES.md:**

### 1. ¿Qué información va en PROJECT.md?
- Cambios de arquitectura / decisiones importantes
- Nuevos componentes o refactoring de existentes
- Estado del proyecto (hitos completados, etapa actual)
- Cambios en flujo de juego
- Patrones para nuevos sistemas (ej: armas escalables)

### 2. ¿Qué información va en RULES.md?
- Convenciones de código (naming, estructura)
- Patrones recomendados / prohibidos
- Estándares de testing y debugging
- Formato de comentarios y documentación
- Estructura de carpetas esperada

### 3. Cómo actualizar
- Después de cada cambio significativo, edita el archivo correspondiente inmediatamente
- Usa timestamps (YYYY-MM-DD HH:MM) cuando documentes cambios críticos
- Mantén ambos archivos en sincronía (no dejes TODOs sin reflejar)
- Actualiza "Última actualización" en sección "Estado Actual"

---

## Historial de Cambios

| Fecha | Versión | Cambio | Impacto |
|-------|---------|--------|--------|
| 2026-07-21 | 0.3.2 | Fix jugador atascado en puerta: blocker anti-retroceso pasa a CLIENT-SIDE, se eliminan los CollisionGroups per-player (`Crosser_*`/`PlayerBlocker_*`) | Se acaba el soft-lock desde el jugador ~14; grupos pasan de crecer sin límite a 5/32 constantes; escala a 60+ jugadores |
| 2026-05-14 | 0.3.1 | Doc: sección "Soporte de Layouts No-Lineales" - qué cambiar para cuartos que doblen/ramifiquen | Roadmap claro para layouts no-lineales |
| 2026-05-14 | 0.3.1 | Fix puerta cerrada atravesable: RaycastBlocker/BackVisual 4 studs antes de BackVolume + nudge anti-stuck empuja ATRAS + `_onBackTouched` ignora si puerta cerrada | Imposible avanzar de cuarto con puerta cerrada |
| 2026-05-13 | 0.3.0 | Sistema de Cuartos v2 (DoorActor) - Event-driven, blockers per-player | Eliminar polling Z y agregar blocker visual per-jugador |
| 2026-05-06 | 0.2.0 | Agregado patrón de armas escalables y docum. DamageCalculator | Escalabilidad para futuros sistemas de armas |
| 2026-05-05 | 0.1.0 | Unificación de PROJECT.md (dos chats) | Consolidación de contexto |
| 2026-05-05 | 0.1.0 | Creación inicial de documentación | Base del proyecto |

---

## Sistema de Cuartos v2 (DoorActor)

### Arquitectura
Reemplaza el polling Z del v1 por un sistema event-driven basado en **DoorActor**:

```
Workspace.Level/
  Room0..Room4/                 (Folders, Attributes de configuracion)
    Door/                       (Model, anidado por Room)
      FrontVolume               (Part CanTouch, trigger de entrada)
      BackVolume                (Part CanTouch, trigger de salida)
      BackVisual                (Part visible, niebla/penumbra; 4 studs ANTES de BackVolume)
      BlockerAnchor             (Part invisible, posicion donde spawnea blocker per-player)
      RaycastBlocker            (Part solido cuando cerrada; coplanar con BackVisual,
                                 4 studs ANTES de BackVolume para que el muro solido se
                                 toque PRIMERO y el trigger BackVolume quede inalcanzable
                                 mientras la puerta este cerrada)
      NPCBlocker                (Part CollisionGroup="NPCBlocker", confina NPCs; en BackVolume)
    References/                 (Folder de ObjectValues que apuntan a las Parts del Door)
    NPCSpawnPoints/             (existente)
    (legacy ZMin/ZMax/DoorOutName REMOVIDOS)
```

### Modulos Server-side (`ServerScriptService.RoomSystem/`)
- **CollisionGroupsBootstrap** - Registra SOLO grupos estaticos: `NPCs`, `NPCBlocker`. Cantidad constante, no crece con jugadores. (Ver "Blocker anti-retroceso" abajo.)
- **DoorActor** - Encapsula una puerta. API: `setOpen`, `spawnBlockerFor(player)`, `removeBlockerFor(player)`, `removeAllBlockers`. Signals: `PlayerEntered`, `PlayerExited`, `PlayerLeftBack`.
- **RoomScanner** - Lee Attributes + Door/References de cada Room y devuelve `roomDesc`.
- **RoomInstance** - Solo NPC lifecycle. Callbacks: `onSpawned`, `onCleared`, `onReset`, `onNpcCountChanged`. (Sin logica de puertas).
- **RoomManager** - Orquesta event-driven. Mantiene `playerRoom` y `playersInRoom`.

### Flujo
1. Jugador cruza `FrontVolume` de Room X -> DoorActor dispara `PlayerEntered`.
2. RoomManager: si Room X esta IDLE -> `room:spawnNPCs()` (transicion IDLE -> ACTIVE).
3. `onSpawned` -> `doorActor[X]:setOpen(false)` + cerrar tambien Room X+1 si nadie en ella.
4. Si `roomDesc.blocksBackward=true` y Room X ACTIVE -> `doorActor[X]:spawnBlockerFor(player)` (blocker fisico solo solido para ese crosser).
5. NPCs mueren -> `onCleared` -> `doorActor[X]:setOpen(true)` + remover blockers + reward UI + spawn Room X+1.
6. Si player retrocede via `PlayerLeftBack` (solo posible en rooms con `BlocksBackward=false`, es decir Room0 y Room1) -> RoomManager checkDespawn rooms adelante.

### Blocker anti-retroceso (CLIENT-SIDE)

El blocker lo construye el **cliente dueño**, no el servidor.

- `DoorActor:spawnBlockerFor(player)` solo hace `FireClient(player, "add", roomId, anchorCFrame)`.
  El servidor NO crea ninguna Part; `self._blockers[uid]` es un flag booleano.
- LocalScript `StarterPlayer.StarterPlayerScripts.BlockerVisibility` clona
  `ReplicatedStorage.Shared.BlockerTemplate`, lo pivotea al CFrame recibido y lo parenta a
  `workspace` **localmente**. Una part local solo existe para ese cliente ⇒ aislamiento
  per-player gratis, **cero CollisionGroups**.
- Protocolo: `("add", roomId, cframe)` | `("remove", roomId)` | `("clear")`. Se indexa por
  `roomId` porque un jugador puede tener blockers en varias puertas.

**⚠️ POR QUÉ (bug de producción, 2026-07-21):** Roblox tiene un límite **duro de 32
CollisionGroups por lugar**. La versión anterior registraba 2 grupos por jugador
(`Crosser_<UserId>` + `PlayerBlocker_<UserId>`) y **nunca los liberaba al salir**. Con 5 grupos
base el presupuesto real era de **~13 jugadores por vida del servidor** (acumulados, no
simultáneos) contra `MaxPlayers=60`. Modo de fallo, todo silencioso:

1. `RegisterCollisionGroup` falla (`Cannot register more than 32 collision groups`) pero el
   módulo marcaba el grupo como registrado igual → nunca reintentaba.
2. `CollisionGroupSetCollidable` falla (`Both collision groups must be registered`) → ese
   jugador queda **sin ninguna entrada en la matrix**.
3. `part.CollisionGroup = "Crosser_<uid>"` **NO lanza error**: asigna el nombre igual y deja
   la part en un grupo **fantasma** que colisiona con TODO y al que no se le puede configurar
   ninguna excepción.
4. Resultado: ese jugador chocaba contra el `NPCBlocker` (invisible, `CanCollide=true`
   siempre, dentro del `BackVolume`) y **quedaba atascado en la puerta** mientras el resto
   pasaba normal. Irreproducible en Studio (nunca hay 14 joiners únicos en solo).

`Crosser_<UserId>` era además **redundante**: todas sus reglas eran idénticas a las de
`Default`. Los personajes ahora quedan en `Default`, con comportamiento idéntico.

**Seguridad:** un exploiter puede borrar su blocker local pero no gana nada — el servidor no
avanza el tracking con la puerta cerrada (`_onBackTouched` hace early-return si `not _open`) y
el `RaycastBlocker` sigue siendo sólido server-side. El blocker solo impide el RETROCESO, que
no otorga ventaja. La autoridad no cambió respecto de antes.

**Invariante a sostener:** el número de CollisionGroups debe ser **constante**. Si algún
sistema futuro necesita aislamiento per-player, resolverlo con parts locales del cliente, no
con grupos. Presupuesto actual: 5/32 (`Default`, `NPCs`, `NPCBlocker`, `Enemies`, `Decorations`).

### Attributes de Room (configurables desde Studio)
- `RoomType` ("COMBAT"|"CORRIDOR"|"FINAL"), `DisplayName`, `MaxAttackers`, `HealthMult`, `DamageMult`, `Armor`, **`BlocksBackward`** (bool, nuevo).

---

## Soporte de Layouts No-Lineales (Futuro - NO implementado)

El sistema actual asume **avance lineal en +Z**: todas las puertas miran hacia +Z y los
cuartos van Room0→Room1→...→Room4 en línea recta. Para cuartos que doblen
(izquierda/derecha/atrás) hay que cambiar cosas concretas. Dos niveles según ambición:

### Nivel 1 — Camino lineal que DOBLA (corredor con curvas)
La secuencia sigue siendo Room0→...→Room4, pero la geometría gira. **Solo cambia `DoorActor`.**
RoomManager (idx/idx+1), RoomScanner y el resto NO se tocan.

**Archivo: `ServerScriptService.RoomSystem.DoorActor`**
- `_currentZSide(volume, player)` — **ROMPE.** Hardcodea `hrp.Position.Z - volume.Position.Z`.
  Fix: usar el vector forward de la puerta y proyección (dot product):
  ```
  forward = (backVolume.Position - frontVolume.Position).Unit
  rel = (hrp.Position - volume.Position):Dot(forward)
  rel > 0 -> lado "adelante"  |  rel < 0 -> lado "atras"
  ```
  Renombrar los labels `"higherZ"/"lowerZ"` -> `"forward"/"behind"`.
- `_onFrontTouched` / `_onBackTouched` — consumen los labels de `_currentZSide`. Ajustar a
  los nombres nuevos. La lógica de PlayerEntered/Exited/LeftBack es idéntica.
- `_onRaycastBlockerTouched` — ya usa `forward` + dot product (axis-agnostic). PERO el grosor
  está hardcodeado como `rb.Size.Z`. Fix: grosor a lo largo de forward =
  `abs(forward.X)*Size.X + abs(forward.Y)*Size.Y + abs(forward.Z)*Size.Z`.

**Geometría en Workspace (por cada cuarto que doble):**
- El `Door` Model debe estar físicamente rotado según la dirección de SU corredor.
  `FrontVolume`/`BackVolume` posicionados en la entrada/salida reales de ese tramo.
- `BackVisual` + `RaycastBlocker` siguen 4 studs ANTES de `BackVolume`, pero "antes" ahora
  es a lo largo del forward de esa puerta, no -Z global.
- `BlockerAnchor.CFrame` orientado para que el eje delgado del blocker atraviese el corredor.
- Sizes de los volumes: el "ancho" del corredor pasa a ser otro eje. Ajustar Size por puerta.

**Lo que NO cambia:** RoomManager, RoomScanner, CollisionGroupsBootstrap, RoomInstance,
blockers per-player (`PivotTo` usa CFrame completo, ya hereda rotación).

### Nivel 2 — Layout RAMIFICADO (un cuarto lleva a 2+ cuartos)
Ya no es secuencia lineal. Necesita refactor de `RoomManager` y `RoomScanner` a **grafo**.

**Archivo: `RoomManager`**
- `idx` / `idx+1` / `idx-1` por todos lados — ROMPE. Reemplazar por grafo: cada cuarto con
  `nextRooms` (lista) y `prevRoom`.
- `playerRoom[player] = idx` y `playersInRoom[idx]` — cambiar a tracking por room id.
- `onPlayerEnteredRoom`, `onPlayerExited`, `onPlayerRetreated`, `checkDespawn`,
  `checkDespawnFrom` — reescribir para navegar el grafo en vez de aritmética de idx.
- `onSpawned` cierra `doorActors[idx]` y `doorActors[idx+1]` — cambiar a "cerrar las puertas
  de los cuartos vecinos".

**Archivo: `RoomScanner`**
- `ROOM_ORDER` (array lineal) — ROMPE. Agregar metadata de conexiones: Attributes tipo
  `NextRooms` (CSV) / `PrevRoom`, o un `Exits` folder con ObjectValues por cuarto.
- `scan()` devuelve grafo en vez de lista ordenada.

**Lo que NO cambia (si ya hiciste Nivel 1):** DoorActor, CollisionGroupsBootstrap,
RoomInstance, geometría.

### Resumen rápido
| Quiero... | Archivos a cambiar | Esfuerzo |
|-----------|--------------------|----------|
| Corredor que dobla (sigue lineal) | `DoorActor` (`_currentZSide` + grosor) + rotar Door Models | Bajo |
| Cuartos que ramifican (árbol) | Nivel 1 + `RoomManager` (idx→grafo) + `RoomScanner` (lista→grafo) | Alto |

---

## Documentos Relacionados
- **[RULES.md]** - Convenciones, patrones y estándares de código
- **[PLAN_COIN_REWARDS_REWORK.md]** - Plan completado: sistema de tiers escalables para rewards
- **Desarrollo**: [Link a repositorio cloud o Studio]
- **Testing**: [Links a servidores de test si aplica]
