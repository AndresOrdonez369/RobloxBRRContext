# PLAN — Sistema Base de Efectos de Cartas (Card Effect Framework)

> Place: **"Patio de recreo de terror de Brainrot"** (dev). Feature de cartas Fases 0–6 ya implementadas.
> Fuente de diseño: PDF `TKA-Sistema de Cartas Coleccionables-130726` — **§10 en adelante** (duplicados, efectos de condición, sinergias de arma, coop, coins, scope V1).
> Objetivo de ESTE plan: construir **solo el sistema base** que permita implementar cualquier efecto de carta (los del PDF y futuros) de forma uniforme, declarativa y escalable. **NO** implementa los efectos finales; deja el terreno para pedirlos uno a uno después.

---

## 1. Objetivo

Hoy los efectos de cartas son 5 arquetipos de stat plano (`EffectArchetypes`: DAMAGE / IRON_SKIN / RATE_OF_FIRE / SCATTER / LIFESTEAL) inyectados como boons `CARD_*`. El PDF §11 describe ~30 efectos que ese modelo no cubre: efectos condicionales (racha, % de vida, sin recibir daño), buffs temporales con stacks disparados por eventos, sinergias filtradas por arquetipo de arma, efectos de coop por proximidad/revive, efectos al ganar coins, y modificadores de stats de arma que hoy no existen (reload, recoil, spread, magazine, rayRadius, chance de no consumir munición, velocidad de movimiento, valor de coins).

Al terminar este plan, implementar un efecto nuevo = **escribir 1 módulo de spec declarativa** (+ un hook custom solo si el efecto es exótico), sin tocar el núcleo. Todos los efectos comparten: ciclo de vida, eventos, condiciones, modificadores, buffs, estado, teardown y feedback.

**Principio rector:** se conserva el carrier actual (carta equipada → boon `CARD_*` → BoonManager aplica en RunStarted vía CardLoadoutService). Muerte/retry/revive/HUD ya funcionan con ese carrier y NO se rediseñan. Lo que crece es la **capacidad** del efecto detrás del boon.

---

## 2. Análisis de los efectos del PDF (qué debe soportar el sistema)

### 2.1 Taxonomía — todo efecto del PDF se descompone en 4 piezas

| Pieza | Qué es | Ejemplos del PDF |
|---|---|---|
| **Modificador de stat** | Cambia un stat del player o del arma, de forma constante o condicionada | 6-7 Strength (+recoil, +speed, +fire rate); Rifle Marksman (−mag, −cadencia, ++daño); Last Standing (+speed si HP<35%) |
| **Trigger (evento)** | Reacciona a un evento del juego, opcionalmente con % de chance y condición | Detonation (X% al matar); Rush (al entrar a cuarto); Coin reload (X% al ganar coins) |
| **Acción** | Lo que hace el trigger: curar, daño en área, dar buff temporal, devolver munición, modificar valor de coins, spawn de objeto de mundo | Room Service (cura), Detonation (AoE), Recoil Control (buff con 5 stacks), Coin Excess (multiplica valor) |
| **Estado propio** | Contadores/flags que el efecto necesita recordar | Rifle-focused fire (hits consecutivos al mismo target, cap 15); Defensive Setup (timer sin recibir daño); Coin Combo (medidor de racha de coins) |

### 2.2 Inventario de requisitos concretos extraídos de §11

**Eventos que el juego debe publicar (vocabulario del bus):**

| Evento | Contexto necesario | Lo exigen |
|---|---|---|
| `EnemyKilled` | enemigo, arma + arquetipo, **distancia del kill**, racha actual | Detonation, Rifle Distance, Shotgun Vampirism/Precision/Rampage, SMG Rush/Quickreload, Recoil Control, Rifle Weaken |
| `EnemyHit` | daño, target (para ramp por objetivo), arma | Rifle-focused fire, lifesteal |
| `ShotFired` (server) | ammo antes/después, magazineSize, arquetipo, herramienta | Last Reserves (última mitad del cargador), SMG All In, Infinite Mags, Pistol Efficiency |
| `ReloadStarted/Finished` | arma, arquetipo | Can't Stop (resetea ramp de All In), Quickreload |
| `WeaponEquipped` | arquetipo | Pistol Draw |
| `PlayerDamaged` | daño, HP resultante | Defensive Setup (reset timer), Room Service (flag del cuarto) |
| `PlayerHealed` | cantidad, `atMaxHealth` | Brain Juice |
| `HealthThresholdCrossed` | pct, dirección | Last Standing (35%), Coin Savings |
| `RoomEntered` | roomId | Rush |
| `RoomCleared` | roomId, `tookDamageInRoom` | Room Service |
| `CoinsGained` | cantidad, fuente (kill/assist/dupe/misión) — **con pre-hook para modificar el valor** | Coin Savings/reload/Steady Hands/Excess/Trail/Combo |
| `TeammateProximityEnter/Exit` | otro player, distancia | Reload motivation, Coop Chain |
| `PlayerRevivedOther / PlayerRevived` | quién revivió a quién (DBNO del DownSystem) | Jolly Cooperation, Double Revive |
| `RunStarted / RunEnded` | — | ciclo de vida (ya existe RunStarted) |

**Stats modificables (superficie completa que el resolver debe cubrir):**

- *Arma (atributos del Tool, todos ya leídos en vivo por el cliente/servidor):* `damage`, `rateOfFire`, `reloadTime`, `magazineSize`, `spread`, `range`, `raysPerShot`, `rayRadius`, `recoilMin/recoilMax`. **Nuevo atributo:** `ammoSaveChance` (probabilidad de no consumir bala — Infinite Mags, Pistol Efficiency).
- *Player:* `MoveSpeed` (compone con PowerUpsShop), `DamageReduction` (ya existe flat), `Damage` global (ya existe percent), `CoinValue` (nuevo — Coin Excess).
- *Alcance (scope):* global o filtrado por `weaponArchetype` (rifle/pistol/smg/shotgun) — toda la sección §11.2.
- *Condición de activación:* siempre / mientras-predicado (HP<X%, racha≥N, cerca de teammate, HP al máximo, sin daño hace X seg).
- *Duración:* permanente-mientras-equipada / buff temporal con `duration` + `maxStacks` (Recoil Control ×5, SMG Rush ×5, Shotgun Precision ×5).

**Acciones de mundo (para los efectos "too much" y Detonation — el sistema base define la librería aunque los efectos lleguen después):** daño en área en posición, zona de debuff persistente (Rifle Weaken, Coin Trail), broadcast de VFX a clientes, rebote de daño en cadena (Shotgun Ricochet, SMG Swarm), succión/onda (Pistol Blackhole). Todas server-side sobre `CombatService.applyDamage` + VFX por remote.

### 2.3 Conclusión del análisis

El 85–90% de los efectos del PDF se expresa **100% declarativamente** con: `modifiers` (constantes o condicionales) + `triggers` (evento + chance + condición + acciones) + parámetros por rareza. El resto (SMG All In, Ricochet, Blackhole, Coop Chain, Double Revive) necesita lógica custom, pero puede vivir dentro del MISMO ciclo de vida vía `hooks` — misma instalación, mismo teardown, mismo estado, misma UI. Eso es exactamente lo que el sistema base debe garantizar: **un solo camino para todo efecto**.

---

## 3. Estado actual (verificado en el place, 2026-07-12)

**Lo que ya existe y se REUSA sin cambios de semántica:**

| Pieza | Dónde | Rol en el nuevo sistema |
|---|---|---|
| Carrier boon `CARD_*` | `ServerScriptService.Services.Cards.CardEffectDefinitions` (inyecta en `BoonDefinitions` al require, gateado por `CardsConfig.ENABLED`) | Sigue siendo el carrier. Sus callbacks `OnApply/OnExpire/OnRemove` pasan a delegar en el runtime nuevo |
| Aplicación en run-start | `CardLoadoutService.applyLoadout` (BindableEvent `RoomSystem.RunStarted`) | Sin cambios |
| Ciclo muerte/retry/revive | `BoonManager.init` + DeathService snapshot (revive re-aplica boons → re-OnApply) | Sin cambios; exige **OnApply idempotente** en el runtime |
| Trigger dispatch | `BoonManager.trigger(player, event, ctx)` — hoy solo `onHit/onKill/onDamaged` desde `Main` | Se conserva para boons legacy; los efectos de carta usan el bus nuevo |
| Stats existentes | `BoonManager.getStatMultiplier/getStatFlat` (Damage, DamageReduction) + `patchWeapon` (raysPerShot, rateOfFire con snapshot `_base*`) | El patrón `_base*` se generaliza; los getters legacy se mantienen |
| Daño | `ReplicatedStorage.Weapons.Shared.DamageCalculator.getMultiplier` = shopMult × boonMult | Punto de enganche para el mult de cartas |
| Pipeline de disparo server-authoritative | `ServerScriptService.Blaster.Scripts.Blaster` (`onShoot`: valida, **decrementa `_ammo`**, recastea rayos, aplica daño; `onReload`: `reloadTime` + refill) | Punto de enganche #1 del sistema (ShotPipeline) |
| Cliente lee atributos EN VIVO | `ReplicatedStorage.Blaster.Scripts.BlasterController` (`shoot()` lee spread/rays/range/rayRadius/recoil por disparo; `canReload` lee magazineSize) | Patch de atributos server-side surte efecto inmediato, sin tocar el cliente |
| Señales de juego | `CombatService.onEnemyHit/onEnemyKilled`; `PlayerService.onHealthChanged/heal/addCoins/setDamageMiddleware`; `RoomService.onRoomCleared`; BindableEvents `RoomSystem.RunStarted/PlayerRoomChanged/PlayerEscaped`; `DownSystemServer` (DBNO/revive, atributo `DownState`) | Adaptadores del bus |
| WeaponDefs con `archetype` | `ReplicatedStorage.Shared.Config.WeaponDefs.*` (rifle/pistol/smg/shotgun) + stats por nivel | Base del scope por arquetipo |
| Loadout máx 3 + álbum + shelf + HUD | Fases 4–5 | Sin cambios |

**Gaps que ESTE plan cierra:**

1. El Tool **no** tiene atributo `archetype` (WeaponFactory setea weaponId/baseDef/engineType/family pero no archetype).
2. No hay racha de kills server-side (KillStreak es LocalScript de UI, `StarterGui.KillStreak`).
3. No hay flag "recibió daño en este cuarto", ni tracker de proximidad entre players, ni evento de threshold de vida.
4. No hay capa de modificadores generalizada (solo Damage/DamageReduction/RaysPerShot/RateOfFire); no hay modificadores condicionales ni buffs temporales hijos con stacks.
5. `MoveSpeed`: `PowerUpsShop/PowerUpsManager` escriben `hum.WalkSpeed = GetMovementSpeed(player)` directo — sin punto de composición.
6. `CoinValue`: `Progression.Coins():GrantEliminationReward` y `PlayerService.addCoins` no publican evento ni admiten multiplicador.
7. Ammo: si el server NO decrementa (`ammoSave` proc), el contador local del cliente (`self.ammo` en BlasterController) se desincroniza → HUD y `canShoot` mienten.
8. `CombatService.onEnemyKilled:Fire(enemy, killer)` no lleva contexto (distancia, arma).
9. `BoonManager` solo soporta **un** `triggerEvent` por def; efectos como Room Service necesitan escuchar 2+ eventos.

---

## 4. Arquitectura del sistema base

### 4.1 Diagrama de módulos (nuevos = ★, modificados = ✎)

```
ServerScriptService.Services.Cards.Effects (★ Folder nuevo)
│
├─ ★ GameEventBus          — vocabulario único de eventos enriquecidos + adaptadores
├─ ★ RunStateProviders     — KillStreakTracker, RoomCombatFlags, ProximityMonitor,
│                            HealthState (inputs de condiciones, event-driven)
├─ ★ ConditionRegistry     — predicados nombrados y parametrizables
├─ ★ ModifierStore         — registro por-player de modificadores de stat + resolve()
│                            + señal onModifiersChanged(player, statKey)
├─ ★ WeaponStatPatcher     — generaliza patchWeapon: snapshot _base* de TODOS los
│                            atributos, recompute = base × resolve(stat, scope arma)
├─ ★ ActionLibrary         — heal / aoeDamage / grantBuff / refillAmmo / setCoinMult /
│                            debuffZone / broadcastVFX (server-side, componibles)
├─ ★ EffectRuntime         — motor: attach/detach de instancias (player, effectId, tier);
│                            interpreta EffectSpec; timers compartidos; estado por instancia
├─ ★ EffectRegistry        — carga specs desde subfolder Definitions/ y valida schema
└─ ★ Definitions/          — 1 ModuleScript por efecto (aquí vivirán los efectos futuros)

ReplicatedStorage.Shared.Config
└─ ✎ CardCatalog / ★ CardEffectBindings — cardId → { effectId } (tier = rareza de la carta)

Modificados (integraciones quirúrgicas):
✎ CardEffectDefinitions   — el boon CARD_* se vuelve carrier delgado → EffectRuntime
✎ BoonManager             — soporta triggerEvents (lista) además de triggerEvent; nada más
✎ Main                    — wiring de adaptadores del bus (kills/hits con contexto)
✎ Blaster server (onShoot/onReload) — hooks de ShotPipeline
✎ DamageCalculator        — término de cartas en getMultiplier + mult por-disparo
✎ PlayerService           — evento CoinsGained con pre-hook de valor; healed ctx
✎ PowerUpsShop/PowerUpsManager — composición de MoveSpeed
✎ WeaponFactory           — atributo archetype + patch inicial vía WeaponStatPatcher
✎ BlasterController (cliente) — resync de self.ammo con GetAttributeChangedSignal("_ammo")
```

### 4.2 Contratos de cada módulo

#### ★ GameEventBus (server)
- API: `Bus.publish(eventName, player, ctx)` / `Bus.subscribe(eventName, fn) -> conn`.
- **Adaptadores** (viven en el propio módulo o en Main, conectan señales existentes → bus):
  - `CombatService.onEnemyKilled/onEnemyHit` → `EnemyKilled/EnemyHit` **enriquecidos**: `ctx = { enemy, tool, archetype, distance, damage, streak }`. La distancia se calcula en el server de Blaster (posición de impacto del rayResult vs origin) y viaja en el Fire — ver §5 Integración I-4.
  - `PlayerService`: middleware de daño → `PlayerDamaged`; `heal` → `PlayerHealed { amount, atMaxHealth }`; `onHealthChanged` → `HealthThresholdCrossed` (el provider decide cuándo cruza).
  - `RoomSystem.PlayerRoomChanged` → `RoomEntered`; `RoomService.onRoomCleared` + membership de RoomManager → `RoomCleared { tookDamageInRoom }` **por player presente**.
  - Blaster server → `ShotFired { ammoBefore, ammoAfter, magazineSize, archetype, tool }`, `ReloadStarted/Finished`.
  - `DownSystemServer` → `PlayerRevivedOther { target } / PlayerRevived { rescuer }`.
  - Grants de coins → `CoinsGained { amount, source }` (post-modificación).
  - `RunStarted` (ya existe) y `PlayerEscaped` → `RunEnded`.
- Regla: **el bus NO reemplaza** las señales existentes ni `BoonManager.trigger`; es capa aditiva. Los boons legacy siguen igual.
- Todas las suscripciones de efectos pasan por EffectRuntime (que garantiza disconnect en teardown). pcall alrededor de cada handler (un efecto roto no tumba el bus).

#### ★ RunStateProviders (server)
- **KillStreakTracker**: cuenta kills por player con ventana de decay (config, alineada al KillStreak UI: reset por tiempo sin kill / daño recibido / muerte / cambio de cuarto — knobs en config). API: `getStreak(player)`, publica `StreakChanged`. Es la fuente para Can't Stop y el ctx de `EnemyKilled`.
- **RoomCombatFlags**: `tookDamageInRoom[player]` — set en `PlayerDamaged`, reset en `RoomEntered`. Lo consume el ctx de `RoomCleared`.
- **ProximityMonitor**: poll 1 Hz (task.wait loop, ≤6 pares con 4 players) sobre HRP; histéresis (enter < R, exit > R+2); publica `TeammateProximityEnter/Exit` y expone `isNearTeammate(player)`. Radio default en config.
- **HealthState**: envoltorio de `PlayerService.onHealthChanged`; expone `getHealthPct(player)`, `isAtMaxHealth(player)` y publica `HealthThresholdCrossed` con umbrales REGISTRADOS por los efectos activos (no lista fija — el runtime registra el 35% cuando Last Standing se instala).
- Cero Heartbeat por efecto; solo ProximityMonitor tiene loop propio (1 Hz) y el timer-wheel del runtime otro (§4.2 EffectRuntime).

#### ★ ConditionRegistry (server)
- Predicados nombrados: `hpBelowPct { pct }`, `hpAtMax`, `streakAtLeast { n }`, `nearTeammate`, `noDamageForSeconds { s }`, `weaponArchetypeIs { archetype }`, `inCombatRoom`… extensible.
- Cada predicado declara **de qué provider depende** y qué evento lo invalida → los modificadores condicionales se re-evalúan SOLO cuando su input cambia (event-driven).
- API: `Conditions.eval(player, condSpec) -> boolean` + `Conditions.watch(player, condSpec, fn(active))` (devuelve conn).

#### ★ ModifierStore (server) — corazón del sistema
- Registro por player: `add(player, record) -> handle`, `remove(handle)`, `clear(player)`.
- `record = { sourceId, statKey, valueType = "percent"|"flat", value, weaponScope = archetype?|nil, condition = condSpec?, stacks = 1 }`.
- `resolve(player, statKey, ctx?) -> (mult, flat)`: suma aditiva de percents (mismo criterio que BoonManager: `1 + Σ value×stacks`) + suma de flats; ignora records cuya `condition` esté inactiva o cuyo `weaponScope` no matchee `ctx.archetype`.
- Señal `onModifiersChanged(player, statKey)` → dispara re-patch de armas / re-apply de WalkSpeed.
- Los **buffs temporales** son records con expiración manejada por EffectRuntime (el store no conoce timers; recibe remove).
- **Convivencia con BoonManager:** los getters legacy (`getStatMultiplier/getStatFlat`) NO se tocan. Los consumidores que hoy suman boons ganan un término más de cartas (ver integraciones). Un solo lugar nuevo de verdad para efectos de carta; cero riesgo de regresión en boons.

#### ★ WeaponStatPatcher (server)
- Generaliza el patrón `_base*` de `BoonManager.patchWeapon` a TODA la superficie: `damage, rateOfFire, reloadTime, magazineSize, spread, range, raysPerShot, rayRadius, recoilMin, recoilMax, ammoSaveChance`.
- `patchTool(player, tool)`: snapshot `_base<Stat>` la primera vez → `SetAttribute(stat, base × mult + flat)` con el scope del arquetipo del tool. Redondeos: `raysPerShot`/`magazineSize` → `math.max(1, math.round())`; `reloadTime` se DIVIDE por el mult de "ReloadSpeed" (mult>1 = recarga más rápida); recoil escala el Vector2 completo.
- `patchAll(player)`: sobre character + backpack (mismo `findPlayerTools` de BoonManager).
- Se dispara por: `onModifiersChanged` (stats de arma), give/equip de arma (WeaponFactory ya llama patch de boons — se añade éste), run-start.
- **Clamps de sanidad** en config (p.ej. reloadTime ≥ 0.15s, rateOfFire ≤ 1200, magazineSize ≥ 1) para que stacking de efectos futuros no rompa el juego.
- Nota `magazineSize`: al bajar el mag (Rifle Marksman) clampear `_ammo` al nuevo tamaño; al expirar el modificador restaurar base sin regalar balas (`_ammo = math.min(_ammo, mag)`).

#### ★ ActionLibrary (server)
- Acciones componibles que usan servicios existentes (nunca reimplementan):
  - `heal(player, { amount | pctOfDamage | pctOfMax })` → `PlayerService.heal`.
  - `aoeDamage(player, { position, radius, damage, falloff? })` → itera enemigos (CombatService.applyDamage / DamageCalculator para humanoids) + `broadcastVFX`.
  - `grantBuff(player, { buffId, duration, maxStacks, stackBehavior, modifiers = {records} })` → registra records en ModifierStore, agenda expiración en el timer-wheel del runtime, re-otorgar = +1 stack o refresh. **Este es el bloque más usado del PDF** (Rush, Recoil Control, Pistol Draw, Brain Juice, Shotgun Precision/Rampage, SMG Rush/Quickreload, Coin Steady Hands, Jolly Cooperation, Reload motivation…).
  - `grantBuffToNearbyTeammates(...)` — variante para coop (Reload motivation afecta a AMBOS).
  - `refillAmmo(player, { pctOfMagazine | amount })` → `SetAttribute(_ammo, …)` clampeado al mag (Coin reload).
  - `modifyCoinGain(ctx, mult)` — solo válido dentro del pre-hook de CoinsGained (Coin Excess).
  - `broadcastVFX(vfxId, position/target)` → remote a clientes; **Regla #2 (editor-authored)**: los templates de VFX viven en `ReplicatedStorage.CardEffectVFX.<vfxId>` creados en el editor; el cliente los clona/emite. El sistema base define el remote + el controller cliente que resuelve por nombre; los templates concretos se autoran cuando lleguen los efectos.
  - `spawnDebuffZone(...)` — para Rifle Weaken/Coin Trail (V-futura, la firma queda definida; implementación mínima: parte anclada + Touched/poll + registro de slow en EnemyAI). Marcada como stub documentado en el base system.
- Toda acción es pcall-safe y respeta `DeathService.isInvulnerable`.

#### ★ EffectRuntime (server) — motor genérico
- `attach(player, effectId, tier, carrierInstance) -> instanceHandle` / `detach(player, effectId)`.
  - **Idempotente**: attach repetido del mismo (player, effectId) = no-op (crítico para revive-snapshot que re-aplica el boon).
  - Al attach: resuelve params del tier (`spec.params[tier]`), registra `modifiers` estáticos/condicionales en ModifierStore, suscribe `triggers` al bus (con `Conditions.watch` para los condicionales), inicializa `state = {}` propio de la instancia, llama `spec.hooks.onAttach` si existe.
  - Al detach (boon OnRemove/OnExpire, muerte real, salida del juego): desconecta TODO, remueve modifiers y buffs hijos vivos, llama `hooks.onDetach`. Un solo camino de teardown.
- **Evaluación de trigger** (orden fijo): evento llega → ¿condición pasa? → ¿roll de chance pasa? (`Random` compartido server) → ejecutar `actions` en orden → actualizar `state` (helpers de ramp/stacks del kit).
- **Timer-wheel único**: un solo loop (reusa el patrón Heartbeat de BoonManager o `task.delay` agendado) para expiración de buffs y timers de efectos (Defensive Setup). Nada de un loop por efecto.
- **Helpers de estado** parametrizables desde la spec: `rampPerTarget` (Rifle-focused fire: contador por enemigo, reset al cambiar de target/reload), `sinceLastDamage` (Defensive Setup), `counter` genérico (Coin Combo).
- Expone snapshot para debug: `getActiveInstances(player)`.

#### ★ EffectRegistry + Definitions/
- `Definitions/` = un ModuleScript por efecto. El registry los require a todos al boot, **valida el schema** (warn+skip si falta un param o el tier no existe — mismo criterio defensivo que CardEffectDefinitions hoy) y expone `get(effectId)`.
- Con `CardsConfig.ENABLED=false` el registry no registra nada (mismo gate actual).

### 4.3 Schema de EffectSpec (el contrato para TODO efecto futuro)

```lua
-- Definitions/RUSH.lua  (ejemplo ilustrativo, NO se implementa en este plan)
return {
    id          = "RUSH",
    displayName = "Rush",
    -- Tiers en los que existe el efecto (PDF: "Niveles de rareza que tiene").
    -- Clave = rareza de la CARTA que lo porta. Solo params, cero lógica.
    params = {
        Common    = { speedPct = 0.06, reloadPct = 0.06, duration = 4 },
        Uncommon  = { speedPct = 0.10, reloadPct = 0.10, duration = 5 },
        Rare      = { speedPct = 0.15, reloadPct = 0.15, duration = 6 },
        Epic      = { speedPct = 0.22, reloadPct = 0.22, duration = 7 },
    },
    describe = function(p) return ("Al entrar a un cuarto: +%d%% velocidad y recarga por %ds")
        :format(p.speedPct*100, p.duration) end,

    -- weaponScope = "rifle",           -- (§11.2) filtra modifiers/triggers al arquetipo
    -- modifiers = { ... },             -- records constantes o con condition (6-7 Strength, Last Standing)
    triggers = {
        {
            event  = "RoomEntered",
            -- chance = 0.25,           -- roll opcional (Detonation, Infinite Mags)
            -- condition = { type="hpBelowPct", pct=0.35 },  -- opcional (Coin Savings)
            actions = {
                { type = "grantBuff", buffId = "RUSH_BUFF",
                  duration = "duration", maxStacks = 1, stackBehavior = "refresh",
                  modifiers = {
                      { statKey = "MoveSpeed",   valueType = "percent", value = "speedPct" },
                      { statKey = "ReloadSpeed", valueType = "percent", value = "reloadPct" },
                  } },
                -- { type = "vfx", vfxId = "RushProc" },
            },
        },
    },
    -- hooks = { onAttach=fn, onDetach=fn, onEvent=fn },  -- escape hatch p/ efectos exóticos
    -- state = { ... },                                   -- declaración de helpers (rampPerTarget…)
}
```

- Los strings en `value/duration` referencian claves de `params[tier]` (resueltos al attach) — un solo lugar para tunear balance por rareza.
- `hooks.onEvent(instance, eventName, ctx)` recibe el MISMO ciclo de vida y estado que lo declarativo: SMG All In o Shotgun Ricochet se escriben como hook sin salirse del sistema.

### 4.4 Carrier: CardEffectDefinitions v2 (✎)

- `EFFECT_BY_CARD` (mapa plano cardId→archetype) se reemplaza por `CardEffectBindings` (ModuleScript en `Shared.Config`): `cardId -> effectId`. El tier = rareza de la carta en `CardCatalog` (validado contra `spec.params`).
- El boon inyectado queda **delgado**:
  ```lua
  BoonDefinitions[boonId] = {
      id=boonId, displayName=carta, rarity=carta, durationType="permanent",
      tags={"card"},                          -- ⚠ CONSERVAR: RoomRewards excluye por este tag
      description = spec.describe(params),
      OnApply  = function(plr) EffectRuntime.attach(plr, effectId, tier, ...) end,
      OnRemove = function(plr) EffectRuntime.detach(plr, effectId) end,
      OnExpire = idem OnRemove,
      triggerEvent = "none",                  -- los triggers viven en el runtime, no en BoonManager
  }
  ```
- Los 5 arquetipos actuales (DAMAge/IRON_SKIN/RATE_OF_FIRE/SCATTER/LIFESTEAL) se **migran a Definitions/** como specs (prueba de paridad del sistema): DAMAGE/IRON_SKIN = `modifiers` puros; LIFESTEAL = trigger `EnemyHit` + acción `heal pctOfDamage`. Comportamiento idéntico al actual = criterio de aceptación.
- `EffectArchetypes` queda deprecado tras la migración (se elimina cuando la paridad esté verificada).

---

## 5. Integraciones quirúrgicas (archivo por archivo)

> Cada una es un chunk pequeño, verificable y con kill-switch (`CardsConfig.ENABLED` o guard `if not CardEffects then`). Ninguna cambia comportamiento con el feature apagado.

**I-1. `WeaponFactory` (✎)**
- `tool:SetAttribute("archetype", def.archetype)` en `build()` (1 línea).
- En `giveToPlayer`, tras `bm.patchWeapon`, llamar `WeaponStatPatcher.patchTool(player, tool)` (lazy-require con pcall, mismo patrón que `getBoonManager`).

**I-2. `ServerScriptService.Blaster.Scripts.Blaster` — onShoot (✎)**
- Antes de decrementar ammo: `local saveChance = blaster:GetAttribute("ammoSaveChance") or 0` → roll server-side; si proca, NO decrementar (Infinite Mags / Pistol Efficiency ya quedan soportados por atributo, sin código de efecto).
- Publicar `ShotFired` al bus con `{ tool, archetype, ammoBefore, ammoAfter, magazineSize }`.
- Damage por disparo: `damage = damage * ModifierStore.resolveShot(player, blaster, shotCtx)` — cubre Last Reserves (condición sobre `ammoBefore/magazineSize`) y cualquier mult condicional por disparo. Un solo punto.
- En `applyDamage`/bloque de rayResults: calcular `distance = (hitPos - origin.Position).Magnitude` y pasarla al Fire de `onEnemyHit/onEnemyKilled` (I-4).
- En `onReload`: publicar `ReloadStarted` (y `ReloadFinished` en el task.delay).

**I-3. `BlasterController` (cliente, ✎ pequeño)**
- `blaster:GetAttributeChangedSignal("_ammo")` → `self.ammo = attr; guiController:setAmmo(...)`. Corrige el desync cuando el server no decrementa (ammoSave) o refilla (Coin reload). Conexión dentro del set de `connections` existente (teardown ya resuelto).

**I-4. `Main` + `CombatService` (✎)**
- `onEnemyHit/onEnemyKilled` ganan 4º argumento `ctx` (tabla opcional: `{ tool, archetype, distance }`) — aditivo, consumidores actuales ignoran el extra.
- Main conecta los adaptadores bus: kills/hits (añadiendo `streak` del tracker), y mantiene INTACTO el wiring legacy de `BoonManager.trigger`.

**I-5. `DamageCalculator` (✎)**
- `getMultiplier` añade término: `× CardEffects.resolve(attacker, "Damage")` (lazy pcall-require como hace con BoonManager). Con 0 efectos activos = ×1.

**I-6. `PlayerService` (✎)**
- `addCoins(player, amount, source?)`: pre-hook `amount = CardEffects.applyCoinGain(player, amount, source)` + publicar `CoinsGained` post-grant. `source` default `"generic"`; los call-sites de kill/assist (`CoinUpgradesManager/GrantEliminationReward`) pasan `"kill"/"assist"` (2 líneas c/u) — el resto queda genérico.
- `heal`: incluir `atMaxHealth` en el publish de `PlayerHealed` (el bus puede computarlo del estado, sin tocar la firma si es más simple).
- El middleware de daño existente ya alimenta `PlayerDamaged` (no se toca su contrato con LAST_STAND).

**I-7. `PowerUpsShop` + `PowerUpsManager` (✎)**
- Donde hoy hacen `hum.WalkSpeed = self:GetMovementSpeed(player)` → `hum.WalkSpeed = GetMovementSpeed(player) * CardEffects.resolve(player, "MoveSpeed")`.
- `ModifierStore.onModifiersChanged(player, "MoveSpeed")` re-aplica WalkSpeed (subscripción en el propio framework, no en las shops). Cuidado con DownSystem (WalkSpeed 6 downed / 16 revived): el re-apply respeta `DownState ~= "Downed"`.

**I-8. `PlayerService.takeDamage` / DamageReduction**
- La reducción flat de cartas entra por el MISMO camino que IRON_SKIN legacy: los records de carta con `statKey="DamageReduction"` se suman en el punto donde hoy se consulta `getStatFlat` (término adicional `+ CardEffects.resolveFlat(player, "DamageReduction")`). Defensive Setup (próxima instancia reducida) se modela como buff de 1 uso: record flat + remove al consumirse (trigger `PlayerDamaged`).

**I-9. `DownSystemServer` (✎)**
- En el punto de revive exitoso (donde restaura WalkSpeed=16 / limpia `DownState`), publicar `PlayerRevivedOther/PlayerRevived` al bus (BindableEvent o require directo del bus con pcall).

**I-10. `BoonManager` (✎ mínimo)**
- Soportar `def.triggerEvents = { "a", "b" }` además de `triggerEvent` (compat total). Solo por robustez del carrier legacy; los efectos de carta reales usan el bus. Nada más se toca aquí.

**I-11. Remote nuevo `CARD_EFFECT_FX` (Types.Remotes)**
- server→client: `{ kind = "proc"|"buff"|"expire", effectId, vfxId?, data }`. El controller cliente base (LocalScript nuevo `CardEffectFXController`, kill-switch por ENABLED) resuelve VFX de `ReplicatedStorage.CardEffectVFX` y puede alimentar pips de buffs en el HUD (integración visual fina = fase posterior, el remote y el controller esqueleto son parte del base).

---

## 6. Fases de implementación

> Proceso acordado: yo leo/decido/reviso-diff/pruebo + edits triviales; **roblox-systems-scripter (SIEMPRE model:"opus")** escribe el Luau en chunks discretos; SIEMPRE verificar el place tras cada chunk (trampa conocida: reportes falsos de subagente). Todo detrás de `CardsConfig.ENABLED` — en este place dev está `true` (testing), así que cada fase es testeable de inmediato, pero cada chunk debe ser no-op para el resto del juego.

### Fase E0 — Fundaciones de eventos y estado (sin gameplay)
1. **Chunk A:** `GameEventBus` + adaptadores de señales ya existentes (kills/hits sin contexto extra aún, salud, rooms, coins post-grant, run start/end). Log de debug togglable.
2. **Chunk B:** `RunStateProviders` (KillStreakTracker, RoomCombatFlags, ProximityMonitor, HealthState) + config de knobs (ventana de racha, radio proximidad).
3. **Chunk C:** contexto enriquecido: I-1 (atributo archetype), I-2 parcial (publicar ShotFired/Reload + distancia en hits), I-4 (ctx en CombatService + streak), I-9 (revive DBNO).
- ✅ Verificación: prints del bus en play-test — matar enemigo loguea `EnemyKilled {archetype, distance, streak}`; entrar a cuarto resetea flag; 2 players cerca disparan proximity.

### Fase E1 — Capa de modificadores
4. **Chunk D:** `ModifierStore` + `ConditionRegistry` (predicados: hpBelowPct, hpAtMax, streakAtLeast, nearTeammate, noDamageForSeconds, weaponArchetypeIs).
5. **Chunk E:** `WeaponStatPatcher` (superficie completa + clamps + re-patch por señal) + I-1 (patch en give) + I-7 (MoveSpeed) + I-5 (Damage) + I-8 (DamageReduction).
6. **Chunk F:** ShotPipeline completo: I-2 (ammoSaveChance roll + resolveShot damage mult) + I-3 (resync ammo cliente) + I-6 (CoinsGained pre-hook + source).
- ✅ Verificación: con `execute_luau` registrar un modifier manual (+50% ReloadSpeed, scope shotgun) → atributo `reloadTime` de la escopeta cambia y el rifle NO; remover → restaura `_base`. Modifier condicional hpBelowPct → togglea al recibir daño/curarse. `ammoSaveChance=1` manual → HUD cliente no baja balas.

### Fase E2 — Motor de efectos
7. **Chunk G:** `EffectRuntime` (attach/detach idempotente, triggers+chance+condición, timer-wheel, helpers de estado ramp/counter) + `EffectRegistry` + schema-validator + `ActionLibrary` núcleo (heal, grantBuff, grantBuffToNearbyTeammates, refillAmmo, modifyCoinGain, broadcastVFX stub server; aoeDamage sobre CombatService; debuffZone = firma + stub warn).
8. **Chunk H:** I-11 remote `CARD_EFFECT_FX` + `CardEffectFXController` esqueleto cliente + folder `ReplicatedStorage.CardEffectVFX` (vacío, editor-authored a futuro — Regla #2).
- ✅ Verificación: attach manual de una spec de prueba (en `Definitions/_TEST` temporal) con trigger RoomEntered→grantBuff → buff aparece, stackea, expira, re-patchea arma; detach limpia todo (inspección `getActiveInstances`).

### Fase E3 — Carrier v2 + migración de paridad
9. **Chunk I:** `CardEffectBindings` + `CardEffectDefinitions` v2 (boon delgado→runtime; conservar tag `"card"`, gate ENABLED, mismos boonIds `CARD_*` — HUD/álbum/shelf no notan el cambio) + migrar los 5 arquetipos actuales a `Definitions/` (DAMAGE, IRON_SKIN, RATE_OF_FIRE, SCATTER, LIFESTEAL) con los MISMOS valores por rareza de `EffectArchetypes.valueByRarity`.
10. **Chunk J:** limpieza — deprecar `EffectArchetypes` (require-free), smoke test integral del ciclo: equipar 3 cartas → run start aplica → morir+retry limpia → revive restaura (snapshot re-attach idempotente) → escapar limpia → re-equip en lobby.
- ✅ Verificación de PARIDAD: mismos números de daño/cadencia/reducción/lifesteal que antes de la migración (comparar con valores actuales en play-test); RoomRewards sigue sin ofrecer boons `CARD_*`; `ENABLED=false` → 0 registros, 0 remotes, juego idéntico.

### Fase E4 — Pilotos de validación del sistema (3 efectos, 1 por familia)
> Único punto donde se implementan efectos reales — mínimos y elegidos para EJERCITAR el sistema, no por diseño final. Sirven de plantilla/cookbook viviente.
11. **Chunk K:** `RUSH` (§11.1: trigger RoomEntered + buff timed multi-stat) — valida bus+buffs+patcher.
12. **Chunk L:** `LAST_RESERVES` (§11.1: mult condicional por disparo sobre ammo/mag) — valida ShotPipeline.
13. **Chunk M:** `COIN_SAVINGS` (§11.3: CoinsGained + condición hpBelowPct + heal) — valida coins+condiciones.
- ✅ Verificación: bindear cada piloto a una carta de prueba del catálogo, run completo con las 3 equipadas; medir que no hay drift de stats tras 10 procs (los `_base*` no se contaminan).

**Fuera de alcance de este plan** (queda listo el seam): efectos restantes del PDF (se piden uno a uno como specs), UI fina de buffs/procs en HUD, `spawnDebuffZone` real, VFX autorados, engine-hooks profundos de proyectil (Ricochet/Swarm/Blackhole — usarán `hooks` + ActionLibrary cuando se pidan), tramo premium de Luck.

---

## 7. Cookbook — cómo se pedirá un efecto cuando el sistema esté listo

1. Crear `Definitions/<EFFECT_ID>.lua` con `params` por rareza + `modifiers`/`triggers` (y `hooks` solo si es exótico).
2. Añadir `cardId -> effectId` en `CardEffectBindings` (o crear la carta en `CardCatalog` con la rareza que define el tier).
3. (Opcional) Autorar template VFX en `ReplicatedStorage.CardEffectVFX.<vfxId>` en el editor.
4. Nada más: describe() alimenta álbum/HUD; ciclo de vida, muerte/revive, stacking, teardown y anti-cheat ya están resueltos por el framework.

Checklist de review para cada efecto nuevo: ¿params completos para todos sus tiers? ¿usa ActionLibrary en vez de tocar servicios directo? ¿estado solo en `instance.state`? ¿ninguna suscripción fuera del runtime?

---

## 8. Riesgos y trampas conocidas

| Riesgo | Mitigación |
|---|---|
| Subagente reporta éxito sin editar (pasó en Fase 0) o muere a mitad (Fase 2) | Verificar SIEMPRE el place tras cada chunk (`script_read`/grep), nunca confiar en el reporte |
| `multi_edit` falla en bloques largos con divisores `──` | Chunks pequeños; fallback a `script.Source` + `UpdateSourceAsync` (patrón ya probado) |
| Probe con `execute_luau` require() = VM separada (estado no compartido) | Instrumentar con prints en el código real, no probar estado por require |
| Doble decremento/refill de ammo si BlasterServer y efectos escriben `_ammo` a la vez | ÚNICO escritor de `_ammo` en gameplay = BlasterServer + ActionLibrary.refillAmmo (ambos server, secuenciales); cliente jamás escribe |
| Drift de `_base*` (patch sobre patch) | El patcher SIEMPRE parte de `_base<Stat>`; los pilotos E4 incluyen test anti-drift |
| Revive re-aplica boons → doble attach | `EffectRuntime.attach` idempotente por (player, effectId) — criterio de aceptación E3 |
| `RoomRewards` ofreciendo cartas como boon (bug ya arreglado) | Carrier v2 CONSERVA `tags={"card"}` y el gate ENABLED en la inyección |
| MoveSpeed peleando con DownSystem (WalkSpeed 6/16) y con MovementFreezeGuard | Re-apply de WalkSpeed respeta `DownState`; no tocar CAS/ControlModule |
| Recoil es cliente-first (CameraRecoiler) | recoilMin/Max son atributos leídos por disparo → patch server replica; el "recoil real" es cosmético+spread, aceptable para V1 |
| Perf | Sin Heartbeat por efecto: 1 timer-wheel + proximidad 1 Hz; re-patch solo en cambios; bus con pcall por handler |
| `CardsConfig.ENABLED=true` en este place dev | OK para testear fases; recordar plan Fase 7: revertir a false antes de publicar en el place real |

---

## 9. Criterios de aceptación globales

1. **Paridad:** tras E3, las 18 cartas actuales se comportan EXACTO igual que hoy (mismos valores por rareza).
2. **Kill-switch:** `ENABLED=false` → cero módulos activos, cero remotes, cero atributos nuevos escritos, juego idéntico al actual.
3. **Uniformidad:** los 3 pilotos de E4 se implementan SOLO con spec + bindings (cero ediciones al núcleo) — si un piloto exige tocar el núcleo, el framework se corrige antes de cerrar el plan.
4. **Teardown limpio:** morir/retry/escapar/salir del juego con cualquier combinación de efectos deja: 0 modifiers residuales, 0 conexiones vivas, atributos de arma = `_base*`.
5. **Server-authoritative:** ningún trigger/roll/valor decidido por el cliente; el cliente solo recibe atributos y FX.
6. **Presupuesto:** sin trabajo por-frame nuevo aparte del timer-wheel y proximidad 1 Hz; cero polling adicional.
