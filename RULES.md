# Reglas y Convenciones - RPS Chaos

## Convenciones de Código Luau

### Nombrado
- **Funciones**: `camelCase` → `calculateWinner()`, `validateMove()`, `replicateResult()`
- **Constantes**: `UPPER_SNAKE_CASE` → `MAX_PLAYERS`, `ROUND_TIMEOUT`, `VALID_MOVES`
- **Módulos/Clases**: `PascalCase` → `RoundManager`, `CombatSystem`, `PlayerState`
- **Variables privadas**: `_leadingUnderscore` → `_internalState`, `_pendingMoves`
- **Booleanos**: prefijo `is`, `has`, `can` → `isRoundActive`, `hasWon`, `canMove()`
- **Signals/Events**: `on` + PascalCase → `onRoundStart`, `onPlayerJoined`
- **Enums/Tablas de config**: `UPPER_SNAKE_CASE` → `MOVES.ROCK`, `GAME_RULES`

### Estructura de Código
- **Máximo 250 líneas** por script (dividir si se excede)
- **Máximo 50 líneas** por función (refactorizar si es mayor)
- **Comentarios**: Español preferentemente, ser consistente en todo el proyecto
- **Indentación**: 4 espacios (NO tabs)
- **Espacios en blanco**: Separar lógica en bloques visuales

### Estándar de Funciones

```lua
-- Comentario descriptivo de QUÉ hace
-- Parámetros (si no son obvios)
-- Devuelve: qué retorna
local function myFunction(param1, param2)
    -- Validación de inputs
    if not param1 then return nil end
    
    -- Lógica principal
    local result = param1 + param2
    
    return result
end
```

### Comentarios

```lua
-- Comentario simple para una línea
--[[ Comentario multi-línea para secciones
     Se usa para documentar funciones complejas ]]

-- TODO: Algo pendiente de implementar
-- FIXME: Bug conocido que hay que arreglar
-- NOTE: Contexto importante para mantener el código
```

### Manejo de Errores

```lua
-- ✅ CORRECTO: Assert para lógica crítica
assert(player, "Player must exist")

-- ✅ CORRECTO: Early returns para validaciones
if not param1 then return nil end

-- ✅ CORRECTO: Pcall para operaciones peligrosas
local ok, err = pcall(function()
    return DataStoreService:GetDataStore("PlayerStats"):GetAsync(userId)
end)
if not ok then
    warn("[DataStore] Error: " .. err)
    return nil
end

-- ❌ INCORRECTO: Ignorar errores
DataStore:GetAsync(userId) -- ¿Qué si falla?
```

---

## Estructura del Proyecto (Roblox Studio)

```
ServerScriptService/
├── CombatSystem/
│   ├── RoundManager.server.lua        -- Orquestación de rondas
│   ├── MoveValidator.server.lua       -- Validación anti-cheat
│   ├── ResultCalculator.server.lua    -- Lógica RPS + ganancias
│   └── MatchmakingService.server.lua  -- Emparejamiento de jugadores
├── PlayerManagement/
│   ├── PlayerStateManager.server.lua  -- Estado + persistencia
│   └── StatsService.server.lua        -- Cálculo de elo/stats
├── Network/
│   └── RemoteHandler.server.lua       -- Centraliza RemoteEvents
└── Utilities/
    ├── Logger.server.lua              -- Sistema de logs debug
    └── Constants.lua                  -- Valores globales

ReplicatedStorage/
├── Modules/
│   ├── RPS_Constants.lua              -- VALID_MOVES, TIMEOUTS, etc
│   ├── RPS_Rules.lua                  -- Lógica determinística RPS
│   ├── Signal.lua                     -- Sistema de eventos customizado
│   └── Utilities.lua                  -- Funciones compartidas
└── Remotes/
    ├── SubmitMove                     -- RemoteEvent: cliente → servidor
    ├── RoundStart                     -- RemoteEvent: servidor → clientes
    ├── RoundEnd                       -- RemoteEvent: servidor → clientes
    └── UpdateStats                    -- RemoteEvent: stats updates

StarterPlayer/
└── StarterPlayerScripts/
    ├── MainController.client.lua      -- Orquestador cliente
    ├── InputHandler.client.lua        -- Input de movimientos
    ├── UIController.client.lua        -- Pantalla de selección
    ├── ResultDisplay.client.lua       -- Mostrar ganador
    └── StatsUI.client.lua             -- Leaderboard local
```

### Reglas de Organización
- **Server Scripts**: `ServerScriptService/` agrupados por funcionalidad
- **LocalScripts**: `StarterPlayer/` para lógica cliente
- **Modules compartidos**: `ReplicatedStorage/Modules/` para código reutilizable
- **Remotes**: `ReplicatedStorage/Remotes/` (carpeta dedicada)
- **Una responsabilidad por script**: No mezclar combat logic con UI

---

## UI Responsive (Scale-Only)

**REGLA CRÍTICA**: Toda `UDim2` para `Size` y `Position` de elementos UI debe usar **scale** (primer y tercer parámetro). El **offset DEBE SER 0** (segundo y cuarto parámetro).

### Por qué
Roblox renderea en pantallas de tamaños muy distintos (PC ultrawide, tablet, móvil, consola). Offset en píxeles fijos rompe el layout: en pantallas pequeñas los elementos se desbordan, en grandes quedan minúsculos. Scale (0-1) se adapta proporcional al contenedor padre, manteniendo proporciones consistentes.

### Forma correcta

```lua
-- ✅ CORRECTO: solo scale, offset = 0
frame.Size     = UDim2.fromScale(0.5, 0.2)            -- 50% ancho, 20% alto del padre
frame.Position = UDim2.fromScale(0.25, 0.1)           -- 25% desde izq, 10% desde arriba
frame.Size     = UDim2.new(0.5, 0, 0.2, 0)            -- equivalente explícito
btn.Size       = UDim2.new(1, 0, 0, 0)                -- ❌ NO, height offset
```

### Forma incorrecta

```lua
-- ❌ INCORRECTO: offset en pixeles rompe responsive
frame.Size     = UDim2.fromOffset(220, 160)
frame.Position = UDim2.new(0, 20, 0, 80)
frame.Size     = UDim2.new(1, -16, 0, 40)             -- offset -16 y 40 rompen
```

### Excepciones permitidas

1. **`UIPadding`**: padding en píxeles está OK (`UDim.new(0, 8)`) — es ajuste interno menor, no layout.
2. **`UICorner.CornerRadius`**: `UDim.new(0, 8)` está OK — radio visual, no layout.
3. **`UIStroke.Thickness`**: número absoluto, no aplica.
4. **`UIListLayout.Padding` / `UIGridLayout.CellPadding`**: gap entre items, OK en píxeles si es pequeño (< 10).
5. **`UIGridLayout.CellSize`**: usa scale en X, scale o offset Y (Y puede ser offset si la altura del item es fija intencional).
6. **`UISizeConstraint` MinSize/MaxSize**: en píxeles está OK, son guardas para extremos.

### Helpers para mantener proporciones

- **`UIAspectRatioConstraint`**: fuerza ratio fijo (ej. 16:9) cuando un elemento debe mantener proporción.
- **`UIScale`**: multiplica escala global del padre (útil para mobile).
- **`UISizeConstraint`**: cap min/max para evitar elementos minúsculos/gigantes.

### Convertir UI existente

Si encuentras UI con offsets, refactoriza así:
1. Calcula `scaleX = offsetX / parentWidth`, `scaleY = offsetY / parentHeight`.
2. Reemplaza `UDim2.new(0, x, 0, y)` por `UDim2.fromScale(scaleX, scaleY)`.
3. Para tamaños fijos de iconos pequeños, considera `UIAspectRatioConstraint` + scale.

### Checklist al crear UI

- [ ] `Size` usa solo scale, offset = 0
- [ ] `Position` usa solo scale, offset = 0
- [ ] `AnchorPoint` configurado correctamente para alinear desde el centro/borde
- [ ] `UIPadding` aplicado en píxeles si necesitas margen interno
- [ ] `UISizeConstraint` para evitar tamaños extremos
- [ ] Probado en al menos: 1920x1080 (PC), 1280x720 (laptop), 800x600 (mobile portrait)

---

## Sistema de Armas (Weapons)

> **Plan completo**: ver `PLAN_WEAPONS_SYSTEM.md`. Este es el resumen de naming + ubicaciones.

### Estructura de carpetas (Phase 0+)

```
ReplicatedStorage/
├── Weapons/
│   ├── Engines/              -- Cliente controllers por engine type
│   ├── Shared/               -- Utilities compartidas (DamageCalculator, etc.)
│   ├── ViewModels/           -- Models FPS por arma
│   ├── Effects/              -- VFX prefabs reutilizables
│   └── AnimSets/             -- Animation sets compartidas
└── Shared/Config/
    ├── WeaponCatalog.lua     -- Registry central
    └── WeaponDefs/
        └── <weaponId>.lua    -- Una def por arma

ServerScriptService/
└── Weapons/
    └── EngineHandlers/       -- Server handlers por engine type

ServerStorage/
└── WeaponPrefabs/            -- Tools (uno por arma) [Phase 1+]
    └── <weaponId>/
```

### Naming conventions

| Tipo | Convención | Ejemplo |
|---|---|---|
| Weapon ID | `snake_case` descriptivo | `ak47`, `cobra_pistol`, `flame_sword` |
| Tool prefab name | == weapon id | `cobra_pistol` |
| ViewModel folder name | == weapon id | `cobra_pistol` |
| Animation slots | PascalCase fijos | `Idle`, `Shoot`, `Reload`, `Equip`, `Swing1`, `Swing2`, `Swing3` |
| MuzzleAttachment | exacto | `MuzzleAttachment` |
| WeaponDef file | == `<weaponId>.lua` | `cobra_pistol.lua` |
| Engine type | PascalCase | `RaycastFirearm`, `MeleeSwing`, `Bow` |
| Family | UPPERCASE | `FIREARM`, `MELEE`, `BOW` |

### Schema de WeaponDef (mínimo)

```lua
-- ReplicatedStorage.Shared.Config.WeaponDefs.<weaponId>
return {
    id          = "<weaponId>",       -- == nombre del módulo
    displayName = "Nombre visible",
    family      = "FIREARM",          -- FIREARM | MELEE | BOW
    engineType  = "RaycastFirearm",   -- mapea al controller correcto
    archetype   = "rifle",            -- rifle | pistol | sword | bow ...
    viewModelId = "<viewModelName>",  -- folder name en ReplicatedStorage.Weapons.ViewModels
    stats       = { ... },            -- engine-specific
    -- Campos extra según engineType (ver PLAN_WEAPONS_SYSTEM.md)
}
```

### Damage scaling pattern (Phase 0 ✅ implementado)

**Único punto de aplicación de daño con multiplicadores**: `ReplicatedStorage.Weapons.Shared.DamageCalculator`.

```lua
local DamageCalculator = require(ReplicatedStorage.Weapons.Shared.DamageCalculator)

-- En cualquier engine handler servidor:
local finalDamage, prevHp, killed = DamageCalculator.applyDamage(humanoid, baseDamage, attacker)
-- Aplica multiplicador de PowerUps automáticamente.

-- Solo calcular sin aplicar:
local final = DamageCalculator.calculate(baseDamage, attacker)

-- Solo obtener el multiplicador:
local mult = DamageCalculator.getMultiplier(attacker)
```

**NUNCA dupliques** el patrón `getDamageMultiplier` viejo. Siempre usa DamageCalculator.

### Reglas para agregar una arma nueva

1. Subir modelo (Tool prefab) → `ServerStorage.WeaponPrefabs.<id>` *(Phase 1+ habilita esto)*
2. Subir ViewModel → `ReplicatedStorage.Weapons.ViewModels.<id>`
3. Crear def → `ReplicatedStorage.Shared.Config.WeaponDefs.<id>.lua`
4. Agregar a un pack en `WeaponPacksConfig.Packs.<pack>.Weapons` *(si aplica)*
5. Listo. WeaponCatalog la carga automáticamente.

> **Si el arma encaja en un engine existente** (firearm raycast, melee swing, bow), no se necesita código nuevo — solo def + assets.

---

## Patrones Recomendados

### ✅ 1. Inicialización de Módulos

```lua
local MyModule = {}

function MyModule.init()
    -- Setup al iniciarse
end

function MyModule.doSomething(param)
    -- Lógica pública
end

return MyModule
```

### ✅ 2. Signals/Events (comunicación entre scripts)

```lua
local Signal = require(game.ReplicatedStorage.Modules.Signal)
local mySignal = Signal.new()

-- Disparar evento
mySignal:Fire(data1, data2)

-- Escuchar evento
mySignal:Connect(function(data1, data2)
    print("Evento recibido:", data1)
end)
```

### ✅ 3. RemoteEvent Pattern (Client → Server)

**Client (InputHandler.client.lua):**
```lua
local submitMoveRemote = game.ReplicatedStorage.Remotes.SubmitMove

local function sendMove(move)
    submitMoveRemote:FireServer(move)
end
```

**Server (RemoteHandler.server.lua):**
```lua
local submitMoveRemote = game.ReplicatedStorage.Remotes.SubmitMove

submitMoveRemote.OnServerEvent:Connect(function(player, move)
    -- Validar en servidor (SIEMPRE)
    if not isValidMove(move) then return end
    
    -- Procesar resultado
    local result = calculateWinner(move)
    
    -- Replicar a clientes
    submitMoveRemote:FireAllClients(result)
end)
```

### ✅ 4. Validación de Movimientos (Server-Authoritative)

```lua
local VALID_MOVES = { "rock", "paper", "scissors" }

local function validateMove(player, move)
    -- Verificación 1: Movimiento válido
    if not table.find(VALID_MOVES, move) then
        warn(player.Name .. " intentó movimiento inválido: " .. move)
        return false
    end
    
    -- Verificación 2: Player existe
    if not player or not player.Parent then
        return false
    end
    
    -- Verificación 3: No está en cooldown
    if isPlayerInCooldown(player) then
        return false
    end
    
    -- Verificación 4: No es movimiento tardío
    if (tick() - roundStartTime) > ROUND_TIMEOUT then
        return false
    end
    
    return true
end
```

### ✅ 5. Sincronización de Estado (Source of Truth en Servidor)

```lua
-- ✅ CORRECTO: Source of truth en servidor
-- Servidor:
local winnerValue = Instance.new("StringValue")
winnerValue.Value = winner
resultEvent:FireAllClients(winner, move1, move2)

-- Cliente: solo recibe y muestra
resultEvent.OnClientEvent:Connect(function(winner, m1, m2)
    showResult(winner)
end)
```

### ✅ 6. Documentación de Funciones Públicas

```lua
--[[ 
    calculateWinner(move1, move2)
    
    Calcula el ganador de una ronda RPS determinísticamente.
    
    @param move1 (string) - Movimiento jugador 1 (rock, paper, scissors)
    @param move2 (string) - Movimiento jugador 2 (rock, paper, scissors)
    @return (string) - "player1", "player2", o "draw"
    @throws error si movimientos no son válidos
    
    Ejemplo:
    local winner = ResultCalculator.calculateWinner("rock", "scissors")
    -- winner = "player1"
]]
function ResultCalculator.calculateWinner(move1, move2)
    assert(move1, "move1 no puede ser nil")
    assert(move2, "move2 no puede ser nil")
    
    -- lógica...
end
```

### ✅ 7. Multiplicadores de Daño Escalables (Weapon Systems)

Para aplicar multiplicadores de upgrades (PowerUps, etc.) a sistemas de armas sin duplicación:

```lua
-- En cada script de arma (CombatService, Blaster, nueva arma, etc):
local function getDamageMultiplier(player)
    -- Lazy require para evitar circular dependencies
    local ok, ShopRegistry = pcall(require, game.ServerScriptService.Services.Shops.ShopRegistry)
    if not ok then return 1 end
    
    local shop = ShopRegistry.Get("PowerUps")
    return shop and shop:GetDamageMultiplier(player) or 1
end

-- En applyDamage o similar:
local function applyDamage(target, baseDamage, attacker)
    if not target or not attacker then return end
    
    -- Aplicar multiplicador de PowerUps
    local multiplier = getDamageMultiplier(attacker)
    local finalDamage = math.floor(baseDamage * multiplier)
    
    -- Resto de lógica de daño...
    target:TakeDamage(finalDamage)
end
```

**Ventajas**:
- ✅ Reutilizable en todas las armas sin duplicación
- ✅ Lazy require evita circular dependencies
- ✅ Fallback a 1.0 si PowerUps no está disponible
- ✅ Cuando existan 3+ armas, refactorizar a `DamageCalculator` centralizado

**Refactor futuro (3+ armas)**:
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

Luego todas las armas usan: `DamageCalculator.applyWithMultiplier(...)`

---

## Patrones Prohibidos ❌

### ❌ Client-Authoritative
```lua
-- NO HACER: Calcular ganador en cliente y confiar en resultado
local winner = calculateWinner(move1, move2)
resultEvent:FireServer(winner) -- ¡Cualquiera puede mentir!

-- HACER: Enviar movimiento, servidor calcula
submitMoveRemote:FireServer(move) -- Servidor valida y calcula
```

### ❌ Hardcoding Mágico
```lua
-- NO HACER:
if math.random() > 0.33 then -- ¿Qué es 0.33?

-- HACER:
local WIN_PROBABILITY = 0.33
if math.random() > WIN_PROBABILITY then
```

### ❌ Sin Timeout
```lua
-- NO HACER: Esperar indefinidamente
while not moveReceived do end -- Infinito si falla conexión

-- HACER: Con timeout
local startTime = tick()
while not moveReceived and (tick() - startTime) < ROUND_TIMEOUT do
    task.wait(0.1)
end
```

### ❌ Variables Globales
```lua
-- NO HACER:
_G.playerStats = {}

-- HACER:
local playerStats = {}
-- retornar en módulo
return { playerStats = playerStats }
```

### ❌ Funciones sin Documentación
```lua
-- NO HACER:
function doThing(x, y) return x + y end

-- HACER:
-- Suma dos números y retorna el resultado
function add(x, y) return x + y end
```

### ❌ Magic Numbers
```lua
-- NO HACER:
if player.Health < 10 then -- ¿Por qué 10?

-- HACER:
local CRITICAL_HEALTH_THRESHOLD = 10
if player.Health < CRITICAL_HEALTH_THRESHOLD then
```

---

## Testing y Validación

### Unit Testing Pattern

```lua
local function testCalculateWinner()
    local result = ResultCalculator.calculateWinner("rock", "scissors")
    assert(result == "player1", "Rock should beat scissors")
    
    result = ResultCalculator.calculateWinner("paper", "rock")
    assert(result == "player1", "Paper should beat rock")
    
    result = ResultCalculator.calculateWinner("scissors", "paper")
    assert(result == "player1", "Scissors should beat paper")
    
    result = ResultCalculator.calculateWinner("rock", "rock")
    assert(result == "draw", "Same move should tie")
    
    print("✅ All calculateWinner tests passed")
end

testCalculateWinner()
```

### Testing Manual (Checklist por ronda)
- [ ] Ambos jugadores envían movimiento simultáneamente
- [ ] Resultado es correcto (Rock > Scissors > Paper)
- [ ] Cliente recibe resultado en <500ms
- [ ] Timeout funciona (10s sin respuesta = pérdida)
- [ ] Stats se actualizan correctamente en DataStore
- [ ] Desconexión no rompe la ronda para el otro jugador

### Testing de Seguridad (Anti-cheat)
- [ ] Movimiento inválido ("Dinamita") es rechazado
- [ ] Movimiento tardío (después de 10s) es rechazado
- [ ] Cambio de movimiento a mitad de ronda es ignorado
- [ ] Replicación no muestra movimiento antes de terminar ronda
- [ ] Cliente no puede calcular ganador (siempre viene del servidor)

---

## Convenciones DataStore

**Prefijo de keys**: `RPS_Player_{userId}_{stat}`

**Estructura de datos**:
```lua
local playerData = {
    wins = 0,
    losses = 0,
    ties = 0,
    totalPoints = 0,
    elo = 1200,
    lastPlayedAt = os.time(),
    lastUpdatedAt = os.time(),
}
```

**Patrón de lectura**:
```lua
local function loadPlayerData(player)
    local userId = player.UserId
    local dataStore = DataStoreService:GetDataStore("PlayerStats")
    
    local success, data = pcall(function()
        return dataStore:GetAsync("RPS_Player_" .. userId)
    end)
    
    if success then
        return data or createDefaultData()
    else
        warn("[DataStore] Error loading " .. userId)
        return createDefaultData()
    end
end
```

---

## Convenciones de Remotes

- **Nombres descriptivos**: `SubmitMove`, `RoundStart`, `RoundEnd`, `UpdateStats`
- **Ubicación**: `ReplicatedStorage/Remotes/`
- **Patrón Client**: `Remote:FireServer(...)` 
- **Patrón Server**: `Remote.OnServerEvent:Connect(...)`
- **Centralizar handlers**: Usar `RemoteHandler.server.lua`

---

## Logs y Debug

### Usar Logger Centralizado

```lua
local Logger = require(game.ServerScriptService.Utilities.Logger)

-- En scripts:
Logger.info("[RoundManager] Ronda iniciada: " .. player1.Name .. " vs " .. player2.Name)
Logger.warn("[MoveValidator] Movimiento inválido de " .. player.Name)
Logger.error("[CombatSystem] Error crítico en cálculo de ganador")
```

### Debug en LocalScript

```lua
-- NO spammear print(), ser selectivo
local DEBUG = false
if DEBUG then
    print("[Client] Movimiento enviado:", move)
end
```

---

## Performance

### Límites de Tolerancia
- **Network latency**: <500ms desde input hasta resultado (aceptable)
- **Memory**: <50MB por instancia de juego (verificar con profiler)
- **CPU**: <2ms por cálculo de ronda (no bloquear 60fps)

### Optimizaciones
- Reutilizar RemoteEvents (centralizar en RemoteHandler)
- Poolear ResultDisplay (no crear nuevas UI cada ronda)
- Evitar GetChildren() en loops (cachear referencias)
- Usar ConnectionPool para listeners frecuentes

---

## Checklist para Nueva Funcionalidad

Antes de agregar un feature nuevo:
- [ ] ¿Es server-authoritative o necesita validación servidor?
- [ ] ¿Qué RemoteEvents/RemoteFunctions necesita?
- [ ] ¿Cómo se persiste en DataStore (si aplica)?
- [ ] ¿Cuál es el test manual para verificar que funciona?
- [ ] ¿Qué puede salir mal en multiplayer con latencia?
- [ ] ¿Está documentado?
- [ ] ¿Respeta convenciones de naming?
- [ ] ¿Está en la carpeta correcta?

## Checklist de Calidad (Antes de Mergear)

- [ ] Código pasa linting (sin errores de sintaxis)
- [ ] Funciones públicas tienen documentación
- [ ] Variables usan convención de naming
- [ ] No hay magic numbers (todo en constantes)
- [ ] Server-side validation implementada
- [ ] Scripts están en carpeta correcta
- [ ] Máximo 250 líneas por archivo
- [ ] Máximo 50 líneas por función
- [ ] Error handling implementado (pcall donde corresponda)
- [ ] No hay variables globales (_G)

---

## Historial de Cambios

| Fecha | Versión | Cambio | Impacto |
|-------|---------|--------|---------|
| 2026-05-06 | 1.1 | Agregado patrón ✅ 7 para multiplicadores de daño escalables | Documentación de weapon systems pattern |
| 2026-05-05 | 1.0 | Unificación de RULES.md (dos chats) | Consolidación de estándares |
| 2026-05-05 | 1.0 | Creación inicial de reglas | Base del proyecto |

---

**Última revisión**: 2026-05-06  
**Autor**: Consolidated from team chats
