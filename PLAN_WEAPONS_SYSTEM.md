# ADR-001: Sistema Modular y Escalable de Armas

**Status:** Proposed
**Date:** 2026-05-06
**Deciders:** Juanse (game lead), Claude (arquitectura)
**Supersedes:** WeaponConfig + WeaponService legacy (a deprecar)

---

## 1. Context

### 1.1 Estado actual auditado

El proyecto tiene **dos pistas paralelas** para armas, una funcional y otra legacy:

#### A) Engine Blaster — funcional, modular, pero monolítico-de-1-arma
Ubicación:
- `ReplicatedStorage.Blaster/` (compartido)
- `ServerScriptService.Blaster/` (servidor)
- `ServerStorage.Blaster` (Tool prefab)

Componentes existentes (yá modulares, separación limpia):
| Modulo | Función |
|---|---|
| `BlasterController` | Orquestador del Tool en cliente |
| `ViewModelController` | Renderiza FPS arms + animations |
| `AimAssistController` | Friction/Tracking/Centering |
| `GuiController` | HUD ammo + reticle + hitmarker |
| `TouchInputController` | Input móvil |
| `CameraRecoiler` | Camera shake |
| `CharacterAnimationController` | Animaciones third-person |
| `ShotReplication` | Replica shots a otros clientes |
| `ServerScriptService.Blaster.Scripts.Blaster` | Validación servidor + damage + Tagged/Eliminated events |

Engine es **100% data-driven via Tool attributes**:
```
damage, spread, raysPerShot, range, rayRadius,
ammo (_ammo runtime), magazineSize, reloadTime, fireMode (Semi|Auto),
recoilMin, recoilMax, viewModel (string), aimAssistRange,
aimAssistFov, aimAssistFrictionStrength, aimAssistTrackingStrength,
aimAssistCenteringStrength, unanchoredImpulseForce
```

**Punto clave:** El atributo `viewModel="Blaster"` ya hace lookup en `ReplicatedStorage.Blaster.ViewModels.<name>`. El sistema **ya soporta múltiples ViewModels conceptualmente** (existen `Blaster` y `AutoBlaster`), solo que **no hay logic que cambie de uno a otro**.

#### B) Sistema legacy (DEAD CODE)
- `ReplicatedStorage.Shared.Config.WeaponConfig` — solo `Placeholder_Melee` y `Placeholder_Ranged`
- `ServerScriptService.Services.WeaponService` — `giveWeapon` nunca se llama
- `StarterPlayer.StarterPlayerScripts.Controllers.WeaponController` — escucha `EQUIP_WEAPON` que nadie firea
- IDs aquí (`Placeholder_Melee`, etc.) **no coinciden** con los de `WeaponPacksConfig` (`iron_sword`, `shadow_blade`...)

### 1.2 Sistema de Packs y Rewards
- `WeaponPacksConfig` define packs (`starter`, `shadow`, `inferno`) con weapons (`Id`, `DisplayName`, `Icon`, `DPS`, `MagSize`)
- `WeaponShop` permite buy/select de packs (persiste en `_data` por sesión, falta hookear DataStore)
- `WeaponShop:GetActivePackWeapons(player)` devuelve la lista activa
- `RoomRewardsService.weapon.grant` — **PLACEHOLDER**: ignora el `weaponId` y siempre clona `ServerStorage.Blaster` Tool

### 1.3 Problema central
Hay un buen engine de FPS aislado, infraestructura de packs/shop/rewards funcionando, pero **falta el bridge entre weapon ID y comportamiento real**. Los `iron_sword`, `flame_sword`, etc. son strings decorativos — todo termina dando el mismo AK.

---

## 2. Decision

Construir un **sistema de armas en capas** con separación clara entre **definición**, **factoría**, **engine** y **assets**, anclado en 4 pilares:

### Pilar 1 — Weapon Catalog (single source of truth)
Un módulo central `WeaponCatalog` que responde "dame todo sobre el arma X". Reemplaza al `WeaponConfig` muerto y unifica con `WeaponPacksConfig`.

### Pilar 2 — Weapon Factory
Dada una weaponId, construye el **Tool configurado**: clona prefab + setea atributos + adjunta sounds/animations/viewmodel. Es la única forma de instanciar armas en runtime.

### Pilar 3 — Engine Taxonomy (un engine por familia)
Cada *familia* de armas comparte un engine. Distintos engines, mismas interfaces:
| Engine | Familia | Reusa |
|---|---|---|
| `RaycastFirearmEngine` | Pistolas, rifles, SMGs | El Blaster actual generalizado |
| `ShotgunEngine` | Escopetas | Hereda de Raycast (multi-ray + spread amplio) |
| `SniperEngine` | Snipers | Hereda de Raycast (ADS + zoom + 1-shot) |
| `ProjectileFirearmEngine` | RPG, lanzagranadas | Nuevo, projectiles físicos |
| `MeleeSwingEngine` | Espadas, daggers, hammers | Nuevo, hitbox + animation |
| `BowEngine` | Arcos, ballestas | Nuevo, charge + projectile |

**El cliente carga el engine correcto leyendo `Tool:GetAttribute("engineType")`** — un solo dispatcher en lugar de N LocalScripts.

### Pilar 4 — Convención de assets para designers
Reglas estrictas de carpetas + nombres + naming patterns que permiten al diseñador agregar un arma nueva subiendo modelos a las carpetas correctas y agregando una entrada en el catálogo. Cero código si el arma cabe en una familia existente.

---

## 3. Options Considered

### Option A: Extender Blaster a todo
Forzar todo (melee, projectile) a usar atributos de Blaster + parches.

| Dimensión | Asesoría |
|---|---|
| Complexity | Baja al inicio, **explota** en mes 2 |
| Cost | Bajo en código, alto en deuda técnica |
| Scalability | Pésima — atributos del Blaster no modelan melee/projectile |
| Reuso | Total al engine actual |

**Rechazado:** Acopla animations, aim assist, ammo a melee donde no aplica.

### Option B: Engines separados con interfaces compartidas ✅ ELEGIDA
Cada familia (firearm, melee, projectile, bow) tiene su propio engine. Comparten utilities (raycasting, validación, network) en `ReplicatedStorage.Weapons.Shared/`.

| Dimensión | Asesoría |
|---|---|
| Complexity | Media — 1-2 engines nuevos hechos bien |
| Cost | 2-3 semanas para framework, luego ~2 días/arma |
| Scalability | Alta — cada engine evoluciona independiente |
| Reuso | Alto — el Blaster existente queda intacto, solo se renombra |
| Designer friendliness | Alta una vez setup |

**Por qué:** Separa concerns sin sobre-arquitecturar. Mantiene la inversión en el Blaster engine.

### Option C: Component-based ECS-style
Cada arma se compone de N components (`HasReload`, `HasADS`, `HasProjectile`, `HasMeleeSwing`...).

| Dimensión | Asesoría |
|---|---|
| Complexity | Alta — diseño de framework genérico |
| Cost | 4-6 semanas iniciales |
| Scalability | Máxima en teoría |
| Reuso | Bajo del Blaster actual (rewrite) |
| Designer friendliness | Baja — muchas decisiones por arma |

**Rechazado:** Overkill para el scope (esperamos 10-30 armas, no 200). Tira a la basura el engine actual.

---

## 4. Trade-off Analysis

**B vs A:** B paga ~1 semana extra de framework para ahorrar deuda técnica recurrente cada arma nueva. Si vamos a tener melee, vale la pena.

**B vs C:** C es más flexible pero significa rewrite del Blaster. B aprovecha que el Blaster YA es genérico via attributes.

**Riesgo principal de B:** Si en el futuro queremos un arma híbrida (ej. magic staff que es melee + proyectil), tenemos que decidir engine principal. Mitigación: el engine elegido puede invocar utilities de otros engines (ej. MeleeEngine puede usar `castProjectile()` shared).

---

## 5. Architecture Detail

### 5.1 Layer diagram

```
┌────────────────────────────────────────────────────────────┐
│ DESIGNER LAYER (no-code)                                   │
│ • Weapon prefabs en ServerStorage/WeaponPrefabs/<id>       │
│ • ViewModels en ReplicatedStorage/Weapons/ViewModels/<id>  │
│ • Stats en ReplicatedStorage/Shared/Config/WeaponDefs/<id> │
└──────────────────────────┬─────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────┐
│ CATALOG LAYER                                              │
│ WeaponCatalog (registry) — agrega todos los WeaponDefs     │
│   :get(id) -> def                                          │
│   :listByFamily(family)                                    │
│   :listByPack(packId)                                      │
└──────────────────────────┬─────────────────────────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────┐
│ FACTORY LAYER                                              │
│ WeaponFactory                                              │
│   :build(weaponId, owner) -> Tool   (sets all attributes)  │
│   :destroy(tool)                                           │
└──────────────────────────┬─────────────────────────────────┘
                           │
            ┌──────────────┼──────────────┬──────────────┐
            │              │              │              │
┌───────────▼──┐  ┌────────▼─────┐  ┌────▼─────┐  ┌─────▼─────┐
│ Raycast      │  │ Projectile   │  │ Melee    │  │ Bow       │
│ Firearm      │  │ Firearm      │  │ Swing    │  │ Engine    │
│ Engine       │  │ Engine       │  │ Engine   │  │           │
│ (hoy=Blaster)│  │ (nuevo)      │  │ (nuevo)  │  │ (nuevo)   │
└──────────────┘  └──────────────┘  └──────────┘  └───────────┘
            │              │              │              │
            └──────────────┴──────────────┴──────────────┘
                           │
┌──────────────────────────▼─────────────────────────────────┐
│ SHARED UTILITIES (ya existen, mantener)                    │
│ castRays, getRayDirections, drawRayResults, hitscan,       │
│ AimAssist, ViewModel, CameraRecoil, sounds, animations,    │
│ damage multipliers (PowerUps), Tagged/Eliminated events    │
└────────────────────────────────────────────────────────────┘
```

### 5.2 WeaponDef schema (un módulo por arma)

```lua
-- ReplicatedStorage.Shared.Config.WeaponDefs.flame_sword
return {
    id = "flame_sword",
    displayName = "Flame Sword",
    icon = "rbxassetid://...",

    -- Family + engine selection
    family = "MELEE",                 -- MELEE | FIREARM | BOW
    engineType = "MeleeSwing",        -- mapea al engine concreto
    archetype = "sword",              -- subtype para animaciones default

    -- Asset references
    prefabPath = "WeaponPrefabs.flame_sword",     -- en ServerStorage
    viewModelId = "flame_sword",                  -- en ReplicatedStorage.Weapons.ViewModels
    animSetId = "sword_default",                  -- conjunto compartido sword

    -- Combat stats
    stats = {
        damage = 45,
        range = 8,
        attackRate = 1.5,             -- swings/seg
        knockback = 25,
        comboCount = 3,
    },

    -- Family-specific
    melee = {
        hitboxShape = "cone",         -- cone | sphere | box
        hitboxArc = 90,               -- degrees
        hitboxDuration = 0.15,        -- seg
        canHitMultiple = true,
    },

    -- VFX/SFX
    sfx = { swing = "...", hit = "...", equip = "..." },
    vfx = { trail = "FireTrail", impact = "FlameImpact" },

    -- Metadata
    cost = 0,                         -- comprado via pack, no individual
    dpsDisplay = 67,                  -- para UI shop
    rarity = "Uncommon",
}
```

Para firearms el schema cambia:
```lua
return {
    id = "tactical_pistol",
    family = "FIREARM",
    engineType = "RaycastFirearm",
    archetype = "pistol",

    prefabPath = "WeaponPrefabs.tactical_pistol",
    viewModelId = "tactical_pistol",
    animSetId = "pistol_default",

    stats = {
        damage = 18,
        rateOfFire = 400,         -- RPM
        magazineSize = 12,
        reloadTime = 1.2,
        range = 600,
        spread = 1.5,
        raysPerShot = 1,
        rayRadius = 0.3,
    },

    fireMode = "Semi",
    recoil = { min = Vector2.new(2,1), max = Vector2.new(4,2) },

    aimAssist = {
        range = 80, fov = 18,
        friction = 0.4, tracking = 0.4, centering = 0.0,
    },

    sfx = { ... },
    vfx = { muzzleFlash = "...", tracer = "..." },

    cost = 0,
    dpsDisplay = 120,
    rarity = "Common",
}
```

### 5.3 WeaponCatalog API

```lua
local WeaponCatalog = {}

function WeaponCatalog.get(id)
    -- Devuelve la def completa o nil
end

function WeaponCatalog.listAll()
function WeaponCatalog.listByFamily(family)   -- "MELEE" | "FIREARM" | "BOW"
function WeaponCatalog.listByEngine(engineType)
function WeaponCatalog.exists(id) -> bool
function WeaponCatalog.getDefaultStarter()    -- arma garantizada al spawn
```

Carga: el módulo recorre `WeaponDefs/` con `:GetChildren()` y registra cada uno por id. Validación: schema check al cargar; warns si falta prefab/viewmodel/animSet.

### 5.4 WeaponFactory

```lua
-- ServerScriptService.Services.WeaponFactory
local WeaponFactory = {}

-- Crea un Tool listo para meter al Backpack del player
function WeaponFactory.build(weaponId)
    local def = WeaponCatalog.get(weaponId)
    assert(def, "WeaponDef not found: " .. weaponId)

    -- 1) Resolver prefab Tool
    local prefab = resolvePrefab(def.prefabPath)
    local tool = prefab:Clone()
    tool.Name = def.id   -- canon: el Tool se llama como el id

    -- 2) Setear atributos según engine
    setEngineAttributes(tool, def)

    -- 3) Validar viewmodel + animSet existen
    validateAssets(def)

    -- 4) Set common metadata attributes
    tool:SetAttribute("weaponId", def.id)
    tool:SetAttribute("engineType", def.engineType)
    tool:SetAttribute("family", def.family)

    return tool
end

-- Reemplaza arma actual del jugador (al elegir reward)
function WeaponFactory.giveToPlayer(player, weaponId, options)
    options = options or {}
    local backpack = player:WaitForChild("Backpack")
    local char = player.Character

    if options.replaceExisting then
        clearWeaponsFromPlayer(player)
    end

    local tool = WeaponFactory.build(weaponId)
    tool.Parent = (options.equipImmediately and char) or backpack
    return tool
end
```

`setEngineAttributes` es un dispatch table:
```lua
local SETTERS = {
    RaycastFirearm = function(tool, def)
        tool:SetAttribute("damage", def.stats.damage)
        tool:SetAttribute("rateOfFire", def.stats.rateOfFire)
        tool:SetAttribute("magazineSize", def.stats.magazineSize)
        tool:SetAttribute("_ammo", def.stats.magazineSize)
        tool:SetAttribute("reloadTime", def.stats.reloadTime)
        tool:SetAttribute("range", def.stats.range)
        tool:SetAttribute("spread", def.stats.spread)
        tool:SetAttribute("raysPerShot", def.stats.raysPerShot or 1)
        tool:SetAttribute("rayRadius", def.stats.rayRadius)
        tool:SetAttribute("fireMode", def.fireMode)
        tool:SetAttribute("recoilMin", def.recoil.min)
        tool:SetAttribute("recoilMax", def.recoil.max)
        tool:SetAttribute("viewModel", def.viewModelId)
        -- aim assist
        tool:SetAttribute("aimAssistRange", def.aimAssist.range)
        tool:SetAttribute("aimAssistFov", def.aimAssist.fov)
        tool:SetAttribute("aimAssistFrictionStrength", def.aimAssist.friction)
        tool:SetAttribute("aimAssistTrackingStrength", def.aimAssist.tracking)
        tool:SetAttribute("aimAssistCenteringStrength", def.aimAssist.centering)
    end,
    Shotgun = function(tool, def) ... end,
    Sniper = function(tool, def) ... end,
    MeleeSwing = function(tool, def)
        tool:SetAttribute("damage", def.stats.damage)
        tool:SetAttribute("range", def.stats.range)
        tool:SetAttribute("attackRate", def.stats.attackRate)
        tool:SetAttribute("knockback", def.stats.knockback)
        tool:SetAttribute("hitboxShape", def.melee.hitboxShape)
        tool:SetAttribute("hitboxArc", def.melee.hitboxArc)
        tool:SetAttribute("hitboxDuration", def.melee.hitboxDuration)
        tool:SetAttribute("comboCount", def.stats.comboCount)
        tool:SetAttribute("viewModel", def.viewModelId)
    end,
    -- ...
}
```

### 5.5 Engine selector (cliente)

Reemplaza el LocalScript que hoy carga directo `BlasterController`:

```lua
-- ReplicatedStorage.Weapons.Client.WeaponBootstrapper (LocalScript dentro del Tool)
local engineType = tool:GetAttribute("engineType")
local engineModule = resolveEngine(engineType)
local controller = engineModule.new(tool)
```

`resolveEngine` mapea string → ModuleScript. Los engines viven en `ReplicatedStorage.Weapons.Engines.<engineType>Controller`.

Esto significa: **el mismo bootstrapper es el único LocalScript dentro del Tool**, sin importar tipo de arma. Cero duplicación de boot code.

### 5.6 Server engine selector

`ServerScriptService.Blaster.Scripts.Blaster` actualmente solo escucha el remote del Blaster. Lo generalizamos:

```
ServerScriptService.Weapons/
├── WeaponServer.server.lua              (entry point)
├── EngineHandlers/
│   ├── RaycastFirearmHandler.lua        (lo de hoy en Blaster.Scripts.Blaster)
│   ├── ProjectileFirearmHandler.lua
│   ├── MeleeSwingHandler.lua
│   └── BowHandler.lua
└── Shared/
    ├── DamageCalculator.lua             (con multiplier de PowerUps)
    └── ValidationUtils.lua
```

Cada handler escucha **su propio remote** (`Shoot`, `Reload`, `MeleeAttack`, `BowRelease`). El Tool sabe qué engine usar via attribute → cliente firea remote correcto → handler correcto valida y aplica daño.

---

## 6. Convenciones de Assets (DESIGNER PLAYBOOK)

### 6.1 Estructura de carpetas

```
ServerStorage/
└── WeaponPrefabs/                  ← Tools, uno por arma
    ├── ak47/                       ← (Tool con todo dentro)
    │   ├── Handle (Part)
    │   ├── Sounds/                 ← AudioPlayers (Shoot, Reload, Equip)
    │   ├── Animations/             ← Animations (las del Tool, no del ViewModel)
    │   ├── Haptics/
    │   └── Scripts/
    │       └── WeaponBootstrapper (LocalScript) ← MISMO en todas las armas
    ├── tactical_pistol/
    ├── flame_sword/
    └── ...

ReplicatedStorage/
├── Weapons/
│   ├── ViewModels/                 ← Models FPS
│   │   ├── ak47/
│   │   │   ├── Model
│   │   │   ├── Animations/         ← Idle, Reload, Shoot, Equip
│   │   │   ├── AnimationController/
│   │   │   ├── Root/
│   │   │   ├── LeftArm/
│   │   │   └── RightArm/
│   │   ├── tactical_pistol/
│   │   ├── flame_sword/
│   │   └── ...
│   ├── AnimSets/                   ← Sets compartidos opcionales
│   │   ├── pistol_default/
│   │   ├── rifle_default/
│   │   └── sword_default/
│   ├── Engines/                    ← Controllers (cliente)
│   │   ├── RaycastFirearmController.lua
│   │   ├── ProjectileFirearmController.lua
│   │   ├── MeleeSwingController.lua
│   │   └── BowController.lua
│   ├── Shared/
│   │   ├── castRays.lua            (ya existe)
│   │   ├── getRayDirections.lua    (ya existe)
│   │   └── ...
│   └── Effects/                    ← VFX prefabs reutilizables

└── Shared/
    └── Config/
        ├── WeaponCatalog.lua       ← Carga todos los Defs, expone API
        ├── WeaponDefs/
        │   ├── ak47.lua
        │   ├── tactical_pistol.lua
        │   ├── flame_sword.lua
        │   └── ...
        └── WeaponPacksConfig.lua   ← (solo metadata visual de packs, ya existe)
```

### 6.2 Cómo agregar un arma nueva (ej: nueva pistola)

**Paso 1 — Modelar/importar la pistola en Studio**
- Importar mesh principal como Tool prefab → guardar en `ServerStorage/WeaponPrefabs/`
- Crear ViewModel (rig FPS con LeftArm, RightArm, Root, MuzzleAttachment) → `ReplicatedStorage/Weapons/ViewModels/`
- Cargar/asignar Animations: `Idle`, `Reload`, `Shoot`, `Equip` (nombres exactos)

**Paso 2 — Crear `WeaponDefs/cobra_pistol.lua`**
```lua
return {
    id = "cobra_pistol",
    displayName = "Cobra Pistol",
    family = "FIREARM",
    engineType = "RaycastFirearm",
    archetype = "pistol",
    prefabPath = "WeaponPrefabs.cobra_pistol",
    viewModelId = "cobra_pistol",
    stats = {
        damage = 22, rateOfFire = 350, magazineSize = 10,
        reloadTime = 1.0, range = 400, spread = 2.0,
        raysPerShot = 1, rayRadius = 0.3,
    },
    fireMode = "Semi",
    recoil = { min = Vector2.new(2,1), max = Vector2.new(3,2) },
    aimAssist = { range=70, fov=15, friction=0.4, tracking=0.3, centering=0 },
    cost = 0, dpsDisplay = 95, rarity = "Common",
}
```

**Paso 3 — Sumarla a un Pack en `WeaponPacksConfig.Packs.<pack>.Weapons`**
```lua
{ Id = "cobra_pistol", DisplayName = "Cobra Pistol", DPS = 95, MagSize = 10, Icon = "..." }
```

**Paso 4 — Done.** El reward screen ya la mostrará y la entregará. El engine `RaycastFirearm` la maneja automáticamente.

### 6.3 Cómo agregar una FAMILIA nueva (ej: shotgun)

**Solo si shotgun no encaja en `RaycastFirearm`** (subtype). Si necesita lógica diferente:

1. Crear `Engines/ShotgunController.lua` (cliente) heredando de `RaycastFirearmController` o desde cero
2. Crear `EngineHandlers/ShotgunHandler.lua` (servidor)
3. Agregar `Shotgun` al `WeaponFactory.SETTERS`
4. Registrar `engineType = "Shotgun"` en defs correspondientes

> **Pro tip**: 99% de las armas (pistol, rifle, smg, shotgun, sniper) caben en `RaycastFirearm` con stats distintos. Sniper se puede hacer con `raysPerShot=1`, `damage=200`, `magazineSize=4`, `rateOfFire=60`. Shotgun con `raysPerShot=8`, `spread=12`, `damage=8`-per-pellet. **No hagas engines nuevos hasta que el comportamiento sea fundamentalmente distinto.**

### 6.4 Naming conventions

| Tipo | Convención | Ejemplo |
|---|---|---|
| weapon id | snake_case, descriptivo | `cobra_pistol`, `flame_sword` |
| Tool prefab name | == weapon id | `cobra_pistol` |
| ViewModel folder name | == weapon id | `cobra_pistol` |
| Animation slots | PascalCase fijos | `Idle`, `Shoot`, `Reload`, `Equip`, `Swing1`, `Swing2`, `Swing3` |
| MuzzleAttachment | exacto | `MuzzleAttachment` |
| WeaponDef file | == weapon id + `.lua` | `cobra_pistol.lua` |
| AnimSet shared | `<archetype>_<variant>` | `pistol_default`, `sword_aggressive` |

---

## 7. Edge Cases & Reglas

### 7.1 Inventario de armas
- Player puede portar **N armas a la vez** en backpack (Roblox standard)
- Reward screen actualmente da una por elección — mantenemos
- `equipImmediately` flag en `WeaponFactory.giveToPlayer` controla si va a Character o Backpack
- Reaparecer (respawn): WeaponService re-equipa la última arma seleccionada (TODO en `_initPlayer`)

### 7.2 Persistencia
- `WeaponShop._data` debe persistir owned packs en SessionService (TODO ya marcado en código)
- Armas individuales NO se persisten (son rewards de la corrida)
- Pack seleccionado SÍ se persiste

### 7.3 Diferencias de aim/punto de mira por familia
| Familia | Punto de mira | Aim Assist | ADS |
|---|---|---|---|
| Pistol | Reticle pequeño centrado | Sí, friction 0.4 | Opcional fase 3 |
| Rifle | Reticle medio | Sí, friction 0.4 | Opcional fase 3 |
| SMG | Reticle grande (spread alto) | Sí, friction 0.5 | No |
| Shotgun | Reticle wide | Sí, centering 0.3 | No |
| Sniper | Crosshair fino + zoom on RMB | Mínimo (friction 0.1) | Sí, mandatorio |
| Bow | Charge bar + reticle | Sí, tracking alto | Sí (drawback) |
| Melee | Sin reticle, indicador de hitbox | N/A | N/A |

`GuiController` debe ramificar por `family`. Refactor: `GuiController.new(blaster, family)` o subclassing.

### 7.4 Efectos visuales y replicación
- **VFX por arma** se referencian en def (`vfx.muzzleFlash`, `vfx.tracer`, `vfx.impact`)
- Cargados desde `ReplicatedStorage/Weapons/Effects/`
- `ShotReplication` (network) ya replica para Blaster — generalizarlo a un `WeaponShotReplication` que despache por engineType
- Melee no tiene shot replication, pero sí **hit feedback replication** (mostrar impact a otros clientes)

### 7.5 Sonidos
- Cada Tool prefab tiene su `Sounds/` folder
- `bindSoundsToAnimationEvents` (ya existe) sigue funcionando
- AudioCompressor target opcional (ya manejado en ViewModelController)

### 7.6 Damage multiplier
- Mantener pattern `getDamageMultiplier()` (ya en CombatService y Blaster)
- **Mover a módulo compartido** `ReplicatedStorage.Weapons.Shared.DamageCalculator` para que los nuevos engines lo importen
- Refactor planeado en `RULES.md` (sección 7) ya documentado

### 7.7 Animations: per-weapon vs per-archetype
- **Default**: cada weapon tiene sus propias animations en su ViewModel
- **Optimización opcional fase posterior**: armas mismo archetype comparten AnimSet (ej. todas las pistolas usan `pistol_default`); solo el modelo cambia
- AnimSet system se introduce solo si tenemos 5+ armas con animations idénticas

### 7.8 Validación al boot
`WeaponCatalog.init()` debe correr al inicio del servidor y:
- Verificar que cada def tenga su prefab existente
- Verificar que cada def tenga su viewmodel existente
- Verificar que el engineType esté registrado
- Verificar campos obligatorios del schema según family
- `error()` con mensaje claro si falla — fail fast en desarrollo

### 7.9 Compatibilidad con ServerStorage.Blaster legacy
Durante migración (Phase 1):
- `RoomRewardsService.weapon.grant` chequea: si existe def, usa Factory; si no, fallback al Blaster Tool
- Una vez Phase 1 termina, eliminar fallback

### 7.10 Edge cases combat
- **NPC dispara armas?** → ahora EnemyAI usa otro stack (Gunner_Basic). Los engines de armas son **player-only**. NPCs siguen su pipeline (`HitscanHandler` ya existe).
- **Self-damage de explosivos?** → para Phase 4 (projectile), considerar friendly-fire flag
- **Reload interrumpido por unequip** → ya manejado en BlasterServer
- **Armas con ammo infinito** → `magazineSize = math.huge` y skip reload
- **Melee combo break si te golpean** → comboCount + window timer, reset en damage taken (Phase 3)

---

## 8. Phased Implementation Plan

### Phase 0 — Foundation (3-4 días)
**Objetivo:** Catalogar y empezar refactor sin romper nada.

- [ ] Crear `WeaponCatalog` con módulo placeholder (1 def: `ak47` migrado del actual Blaster)
- [ ] Crear estructura de carpetas `ReplicatedStorage/Weapons/`
- [ ] Mover utilities (`castRays`, `getRayDirections`, `drawRayResults`) a `Weapons/Shared/`
- [ ] Crear `DamageCalculator` shared con `applyDamageWithMultiplier(target, dmg, attacker)` → migrar Blaster + CombatService a usarlo
- [ ] Documentar naming conventions (este doc + RULES.md)
- [ ] Eliminar `WeaponConfig` legacy + `WeaponService.giveWeapon` muerto + `WeaponController` LocalScript

🧪 **Test manual**: Blaster sigue funcionando idéntico al actual. Equipar, disparar, recargar, matar enemigo, ver multiplier de PowerUps aplicado.

---

### Phase 1 — Generalize Blaster engine (4-5 días)
**Objetivo:** Convertir el Blaster en `RaycastFirearmEngine` capaz de manejar N armas.

- [ ] Renombrar `ReplicatedStorage.Blaster` → `ReplicatedStorage.Weapons.RaycastFirearm` (preservar referencias)
- [ ] Mover `BlasterController` → `RaycastFirearmController`
- [ ] Renombrar Tool prefab `Blaster` → `ak47` (id canónico)
- [ ] Crear `WeaponBootstrapper` LocalScript genérico (reemplaza el `Blaster` LocalScript dentro del Tool)
- [ ] Crear `WeaponFactory.build(weaponId)` que setea atributos desde def
- [ ] Refactor `RoomRewardsService.weapon.grant` para usar Factory
- [ ] Migrar todos los IDs en packs (`iron_sword`, `flame_sword`, etc.) — por ahora apuntan a un alias `ak47` mientras Phase 2 los hace reales

🧪 **Test manual**: Reward screen entrega armas con nombre correcto pero todas son AK por dentro (esperado en esta fase). Comprobar que el equipar/disparar funciona idéntico.

---

### Phase 2 — Multiple firearms (1 semana, depende de assets)
**Objetivo:** 3-4 armas reales nuevas usando el engine actual.

- [ ] **Pistola** (cobra_pistol): nuevo modelo + viewmodel + animations, def, en pack starter
- [ ] **Rifle de asalto** (military_rifle): nuevo modelo, def
- [ ] **SMG** (vandal_smg): nuevo modelo, def
- [ ] **Shotgun** (combat_shotgun): nuevo modelo, ajustar `raysPerShot=8` `spread=12` `damage=10` → comportamiento de escopeta sin engine nuevo
- [ ] **Sniper** (precision_rifle): nuevo modelo, ajustar `damage=200` `magazineSize=5` `rateOfFire=60` `range=2000`

🧪 **Test manual**: Comprar pack starter, dropear cada arma en reward screen, verificar que se siente distinta cada una (rate of fire, damage, ammo).

---

### Phase 3 — Melee engine (1-1.5 semanas)
**Objetivo:** Familia completamente nueva.

- [ ] Crear `MeleeSwingController` (cliente): hitbox calc, swing animation, combo manager
- [ ] Crear `MeleeSwingHandler` (servidor): validar hitbox, aplicar daño, knockback
- [ ] Crear `MeleeRemote` para attacks
- [ ] Diseñar AnimSet `sword_default` con Swing1/Swing2/Swing3
- [ ] Implementar **3 espadas**: `iron_sword`, `shadow_blade`, `flame_sword`
- [ ] Update `RoomRewardsService` UI para mostrar archetype icon (sword vs gun)
- [ ] Update `GuiController` para no mostrar ammo HUD si family == MELEE

🧪 **Test manual**: Equipar espada, atacar con M1, ver hitbox dañar enemigos cercanos, combo de 3 hits, sin ammo HUD.

---

### Phase 4 — Projectiles + Bow (1.5-2 semanas)
**Objetivo:** Lógica de proyectiles físicos.

- [ ] Crear `ProjectileEngine` con simulación servidor (recomendado: client-prediction + server validation)
- [ ] Implementar **arco** (`basic_bow`, `void_bow`) con charge mechanic
- [ ] Implementar **lanzagranadas** opcional (`lava_launcher`) con AOE damage
- [ ] AOE damage helper en `DamageCalculator`

🧪 **Test manual**: Cargar arco con M1, soltar para disparar flecha, ver flecha viajar y impactar, daño aplicado.

---

### Phase 5 — Polish & Designer kit (3-5 días)
**Objetivo:** Documentar y agregar quality of life.

- [ ] Crear template Tool prefab + ViewModel rig que el designer puede duplicar
- [ ] Script de validación: `WeaponCatalog.validateAll()` que reporta inconsistencias
- [ ] Comando admin in-game `/give <weaponId>` para QA
- [ ] Update `PROJECT.md` con sección "Weapon System Architecture" final
- [ ] Update `RULES.md` con conventions de naming weapon
- [ ] Crear `WEAPON_TEMPLATE.md` paso-a-paso para diseñadores

---

## 9. Consequences

### 9.1 Lo que se vuelve más fácil
- ✅ Agregar arma nueva del mismo type: 1-2 horas (modelo + def + entry en pack)
- ✅ Cambiar balance: editar 1 archivo (su def)
- ✅ Designer puede agregar armas sin tocar código (si encajan en engine existente)
- ✅ Hot-swap de armas en reward es trivial
- ✅ Testing de balance se hace por def, no por scripts

### 9.2 Lo que se vuelve más difícil
- ⚠️ Setup inicial de Phase 0-1 toma 1-2 semanas
- ⚠️ Cada engine nuevo requiere boilerplate (cliente + servidor + remote + handler)
- ⚠️ Debugging cross-engine: un bug puede vivir en shared utilities y afectar todas

### 9.3 Lo que hay que revisar después
- 📌 Si superamos 30 armas, considerar mover defs a DataStore para hot-config
- 📌 Si los archetypes empiezan a repetir mucho config, introducir AnimSets compartidos
- 📌 Si proyectiles + melee se vuelven core gameplay, fusionar engines en framework genérico (Option C del ADR)
- 📌 Network: si tenemos lag con muchos shots, mover a UnreliableRemoteEvent para shots no-críticos

---

## 10. Action Items (resumen ejecutable)

### Critical path (Phase 0-1, NO romper Blaster actual)
1. [ ] Crear `ReplicatedStorage/Weapons/` folder structure
2. [ ] Crear `WeaponCatalog` module con placeholder def `ak47`
3. [ ] Crear `WeaponFactory` con `build()` y `giveToPlayer()`
4. [ ] Crear `DamageCalculator` shared
5. [ ] Migrar `RoomRewardsService.weapon.grant` a Factory
6. [ ] Eliminar legacy: `WeaponConfig`, `WeaponService.giveWeapon`, `WeaponController`
7. [ ] Test: Blaster sigue funcionando 100% igual

### Phase 2 dependencies
8. [ ] **Designer**: subir 4 modelos de armas (pistol, rifle, smg, shotgun)
9. [ ] **Designer**: crear ViewModels para cada uno (rig + animations)
10. [ ] **Code**: 4 weapon defs nuevos
11. [ ] Update WeaponPacksConfig packs con IDs reales

### Phase 3+
12. [ ] Diseñar architecture detallada de `MeleeSwingEngine` (sub-ADR si es complejo)
13. [ ] Decisión: client-prediction vs server-only para projectiles (sub-ADR)

---

## 11. Quick Reference Tables

### 11.1 Mapping engineType → módulos

| engineType | Cliente Controller | Servidor Handler | Remote |
|---|---|---|---|
| `RaycastFirearm` | `Engines/RaycastFirearmController` | `EngineHandlers/RaycastFirearmHandler` | `Shoot`, `Reload` |
| `ProjectileFirearm` | `Engines/ProjectileFirearmController` | `EngineHandlers/ProjectileFirearmHandler` | `FireProjectile`, `Reload` |
| `MeleeSwing` | `Engines/MeleeSwingController` | `EngineHandlers/MeleeSwingHandler` | `MeleeAttack` |
| `Bow` | `Engines/BowController` | `EngineHandlers/BowHandler` | `BowRelease` |

### 11.2 Atributos requeridos por engineType

| Atributo | RaycastFirearm | MeleeSwing | Bow |
|---|---|---|---|
| `damage` | ✅ | ✅ | ✅ |
| `range` | ✅ | ✅ | ✅ |
| `viewModel` | ✅ | ✅ | ✅ |
| `magazineSize` | ✅ | ❌ | ✅ |
| `_ammo` | ✅ | ❌ | ✅ |
| `reloadTime` | ✅ | ❌ | ❌ |
| `rateOfFire` | ✅ | ❌ | ❌ |
| `attackRate` | ❌ | ✅ | ❌ |
| `chargeTime` | ❌ | ❌ | ✅ |
| `hitboxShape` | ❌ | ✅ | ❌ |
| `comboCount` | ❌ | ✅ | ❌ |

---

## 12. Documentos Relacionados
- **PROJECT.md** — sección Weapon Systems & Damage Scaling (existente)
- **RULES.md** — pattern ✅ 7 Multiplicadores de Daño (existente)
- **PLAN_COIN_REWARDS_REWORK.md** — sistema de tiers (completado)

---

**Próximo paso recomendado:** Aprobar Phase 0 (riesgo cero, todo es prep + cleanup) y empezar por allí. Phase 1 es el momento donde se rompe el código existente — review extra antes de mergear.
