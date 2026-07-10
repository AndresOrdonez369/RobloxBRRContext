# PLAN: Dynamic Tier-Based Reward System para CoinUpgrades

**Objetivo:** Rework backend de CoinUpgrades para manejar rewards basados en **tiers de enemigos** independientes del tipo. UI mantiene misma estructura visual (3 categorías), pero backend calcula rewards dinámicamente según el tier del NPC que se elimine.

---

## FASE 1: Sistema de Tiers Independientes

### STEP 1.1: Asignar `npcTier` a EnemyConfig
- **Qué:** Cada enemigo en `EnemyConfig` recibe un campo `npcTier` (1, 2, 3, etc.)
- **Por qué escalable:** Los tiers son **independientes del tipo** (Goblin, Arquero, Metralleta)
- **Ejemplo:**
  - Room 1: `Melee_Basic.npcTier = 1`, `Ranged_Basic.npcTier = 1`, `Gunner_Basic.npcTier = 1`
  - Room 3: `Melee_Basic.npcTier = 2`, `Ranged_Basic.npcTier = 2`, `Gunner_Basic.npcTier = 2`
  - Mismo enemigo, diferente tier según la room
- **Cambios:** +1 línea por enemigo en EnemyConfig
- **Beneficio:** Permite reutilizar modelos de NPCs con diferentes rewards sin duplicar

```lua
["Melee_Basic"] = {
    id = "Melee_Basic",
    displayName = "Goblin",
    npcTier = 1,  -- ← NUEVO: cambiar según room/dificultad
    health = 50,
    -- resto igual
},
```

---

## FASE 2: Configuración con Values en Workspace

### STEP 2.1: Crear Config Visual en Workspace
- **Qué:** Carpeta `workspace/RewardConfig` con Values para cada stat
- **Por qué:** Equipo puede cambiar valores sin editar código Lua
- **Estructura:**
```
workspace/
  RewardConfig (Folder)
    Elimination (Folder)
      tier_1_base (IntValue) = 10
      tier_1_linear (NumberValue) = 1.5
      tier_1_quadratic (NumberValue) = 0.3
      tier_2_base (IntValue) = 13
      tier_2_linear (NumberValue) = 1.95
      tier_2_quadratic (NumberValue) = 0.39
      tier_3_base (IntValue) = 17
      tier_3_linear (NumberValue) = 2.55
      tier_3_quadratic (NumberValue) = 0.51
    Assist (Folder)
      tier_1_base (IntValue) = 5
      tier_1_linear (NumberValue) = 0.8
      -- ... etc
    CoinReward (Folder)  -- sin tiers
      base (IntValue) = 15
      linear (NumberValue) = 2
      quadratic (NumberValue) = 0.5
```

- **Beneficio:** Designers/balance team cambian valores en Studio sin recargar scripts
- **API:** `CoinUpgradesShop:GetConfigValue(category, tier, param)` lee desde workspace

---

## FASE 2.2: Estructura de Datos en Config Module

### STEP 2.2: Reestructurar CoinUpgradesConfig
- **Keep:** Schema de formulas (reward, cost)
- **New:** Definición de cuántos tiers hay y sus multiplicadores base
- **Runtime:** Leer valores de workspace, aplicar formulas

```lua
local Config = {
    MaxLevel = 100,
    CategoryOrder = { "CoinReward", "Elimination", "Assist" },
    
    -- Define cuántos tiers existen y su escalado
    TierConfig = {
        tier_1 = { multiplier = 1.0 },   -- baseline
        tier_2 = { multiplier = 1.3 },   -- +30%
        tier_3 = { multiplier = 1.7 },   -- +70%
        -- Agregar más tiers sin tocar backend
    },
    
    Categories = {
        CoinReward = {
            DisplayName = "Coin Reward",
            Description = "Base coins for non-combat rewards.",
            Icon = "...",
            -- Formulas de referencia (valores activos vienen de workspace)
            rewardBase      = 15,
            rewardLinear    = 2,
            rewardQuadratic = 0.5,
            rewardExp       = 2,
            costBase        = 50,
            costLinear      = 15,
            costQuadratic   = 3,
            costExp         = 2,
        },
        -- Elimination y Assist: valores se leen por tier desde workspace
    },
}
```

---

## FASE 3: Cambios en CoinUpgradesShop

### STEP 3.1: Actualizar Almacenamiento de Niveles
- **Antes:** `_levels[userId] = { CoinReward=5, Elimination=10, Assist=8 }`
- **Después:** `_levels[userId] = { CoinReward=5, Elimination_tier_1=12, Elimination_tier_2=8, Elimination_tier_3=3, Assist_tier_1=10, Assist_tier_2=6, Assist_tier_3=1 }`
- **Lógica:** Cada tier de Elimination/Assist tiene su propio nivel

### STEP 3.2: Nuevas APIs Públicas

```lua
-- Devuelve coins por eliminar enemigo de X tier
-- Busca: Elimination_tier_X nivel del player
-- Calcula: formula(base, linear, quad, level) * tier_multiplier
CoinUpgradesShop:GetEliminationReward(player, npcTier) -> number

-- Igual para asistencia
CoinUpgradesShop:GetAssistReward(player, npcTier) -> number

-- Sin cambios
CoinUpgradesShop:GetCoinReward(player) -> number

-- Para debugging/admin
CoinUpgradesShop:GetLevel(player, category, tier) -> number
```

**Ejemplo:**
```lua
shop:GetEliminationReward(player, 2)
-- Si player está en L5 de Elimination_tier_2
-- Lee tier_2_base=13, tier_2_linear=1.95, tier_2_quad=0.39 desde workspace
-- Calcula: 13 + 1.95*5 + 0.39*5^2 = ~33
-- Aplica multiplier tier_2: 33 * 1.3 = ~43 coins
-- Devuelve: 43
```

### STEP 3.3: GetShopState — UI Simplificada
- **Problema:** Mostrar 10 tiers en UI es caótico
- **Solución:** Mostrar solo la categoría "padre" con estado general, sin expandir todos los tiers
- **Ejemplo en UI:**
  - `Elimination` → `"Niveles en tiers: 1-12 | 2-8 | 3-3"` (comprimido)
  - O solo: `"Elimination (Mixed levels)"` con botón para ver detalles si quieren
  - Cuando clickean para upgradear, UI muestra dialog para elegir qué tier upgradear

```lua
function CoinUpgradesShop:GetShopState(player)
    return {
        Categories = {
            {
                Key = "CoinReward",
                DisplayName = "Coin Reward",
                Level = 5,  -- nivel actual
                NextReward = 100,  -- próximo reward
                -- ... std fields
            },
            {
                Key = "Elimination",
                DisplayName = "Elimination",
                Level = nil,  -- nil significa "multiple tiers"
                TierSummary = { tier_1=12, tier_2=8, tier_3=3 },  -- compact
                Description = "Rewards by NPC tier (see dropdown)",
                -- ... std fields
            },
            -- ... Assist similar
        }
    }
end
```

### STEP 3.4: Actualizar HandlePurchase
```lua
function CoinUpgradesShop:HandlePurchase(player, payload)
    -- payload: { Category="Elimination", Tier=2 }
    local category = payload.Category
    local tier = payload.Tier  -- NEW
    
    if category == "CoinReward" then
        tier = nil  -- CoinReward no tiene tiers
    end
    
    local levelKey = category .. (tier and "_tier_" .. tier or "")
    local level = self:GetLevel(player, category, tier)
    
    -- Proceder igual que antes
    if level >= Config.MaxLevel then return false, "max_level" end
    local cost = self:GetCostForTier(category, tier, level)
    if not self._ps.spendCoins(player, cost) then return false, "not_enough_coins" end
    
    self._levels[player.UserId][levelKey] = level + 1
    return true, "ok"
end
```

### STEP 3.5: Helper: Lectura de Workspace
```lua
local function GetRewardFormula(category, tier)
    local rewardConfig = workspace:FindFirstChild("RewardConfig")
    if not rewardConfig then
        return Config.Categories[category]  -- fallback a módulo
    end
    
    local categoryFolder = rewardConfig:FindFirstChild(category)
    if not categoryFolder then return Config.Categories[category] end
    
    local prefix = tier and ("tier_" .. tier .. "_") or ""
    local base = categoryFolder:FindFirstChild(prefix .. "base")
    local linear = categoryFolder:FindFirstChild(prefix .. "linear")
    local quad = categoryFolder:FindFirstChild(prefix .. "quadratic")
    
    return {
        rewardBase = base and base.Value or 0,
        rewardLinear = linear and linear.Value or 0,
        rewardQuadratic = quad and quad.Value or 0,
        rewardExp = 2,  -- invariante
    }
end
```

---

## FASE 4: Persistencia

### STEP 4.1: Actualizar SessionService
- **Qué:** Guardar/cargar los 6+ nuevos campos de subcategorías
- **Dónde:** `_initPlayer` (load) y `PlayerRemoving` (save)
- **Schema:**
```lua
shopLevels = {
    CoinReward = 5,
    Elimination_tier_1 = 12,
    Elimination_tier_2 = 8,
    Elimination_tier_3 = 3,
    Assist_tier_1 = 10,
    Assist_tier_2 = 6,
    Assist_tier_3 = 1,
}
```

---

## FASE 5: Conectar CombatService → Rewards

### STEP 5.1: Listener en CombatService
```lua
CombatService.onEnemyKilled:Connect(function(enemy, attacker)
    -- Obtener npcTier del enemigo
    local enemyId = enemy.Name  -- o field que identifique tipo
    local enemyConfig = EnemyConfig.get(enemyId)
    if not enemyConfig then return end
    
    local npcTier = enemyConfig.npcTier
    local shop = ShopRegistry.Get("CoinUpgrades")
    
    -- Reward para killer
    local elimReward = shop:GetEliminationReward(attacker, npcTier)
    PlayerService.addCoins(attacker, elimReward)
    
    -- Rewards para otros (assists)
    for _, player in ipairs(game:GetService("Players"):GetPlayers()) do
        if player ~= attacker and player.Character and player.Character.Humanoid.Health > 0 then
            local assistReward = shop:GetAssistReward(player, npcTier)
            PlayerService.addCoins(player, assistReward)
        end
    end
end)
```

---

## Resumen de Tareas

| # | Task | Archivo | Estimado |
|---|------|---------|----------|
| 1.1 | Asignar npcTier a NPCs | EnemyConfig | 15 min |
| 2.1 | Crear RewardConfig en Workspace | (manual Studio) | 10 min |
| 2.2 | Actualizar CoinUpgradesConfig schema | CoinUpgradesConfig | 20 min |
| 3.1 | Cambiar almacenamiento _levels | CoinUpgradesShop | 30 min |
| 3.2 | Nuevas APIs (GetEliminationReward, GetAssistReward) | CoinUpgradesShop | 45 min |
| 3.3 | GetShopState simplificado | CoinUpgradesShop | 30 min |
| 3.4 | HandlePurchase con Tier | CoinUpgradesShop | 30 min |
| 3.5 | Helper GetRewardFormula (leer Workspace) | CoinUpgradesShop | 30 min |
| 4.1 | SessionService persistencia | SessionService | 45 min |
| 5.1 | CombatService listener | CombatService | 30 min |
| **Total** | | | **~4 horas** |

---

## Orden de Ejecución

1. STEP 1.1 (15 min)
2. STEP 2.1 (10 min) — criar carpeta y Values en Studio
3. STEP 2.2 (20 min)
4. STEP 3.1-3.5 (2.5h) — Rewrite CoinUpgradesShop
5. STEP 4.1 (45 min)
6. STEP 5.1 (30 min) — Hook final
7. Testing + Balance tuning (30 min)

---

## Beneficios del Design

✅ **Escalable:** Agregar tier_4 solo requiere agregar 3 Values en workspace  
✅ **No-code balancing:** Equipo cambia rewards sin tocar Lua  
✅ **Simple UI:** Evita clutter mostrando todos los tiers  
✅ **Flexible:** Mismo NPC puede tener diferente tier en diferentes rooms  
✅ **Mantenible:** Backend separado de visuals

---

## Decisiones de Diseño

- **Tiers independientes de tipos:** Un Goblin puede ser tier 1 en Room 1 y tier 2 en Room 5
- **Valores en Workspace:** Permite cambios sin recompiar scripts
- **UI comprimida:** Muestra resumen en lugar de todas las subcategorías
- **Multiplicadores por tier:** Escalan linealmente (1.0x, 1.3x, 1.7x, etc.)

