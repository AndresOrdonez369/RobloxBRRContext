# GUÍA — Implementar un efecto de carta en el Card Effect Framework

> **Para quién:** cualquier modelo/agente (o humano) que deba implementar un efecto de carta nuevo en el place **"Patio de recreo de Brainrot"** (Studio). Esta guía es autocontenida: siguiéndola paso a paso, el efecto queda acoplado al sistema sin tocar el núcleo.
>
> **Estado del framework:** Fases E0–E3 completas y verificadas (bus de eventos, providers de estado, condiciones, modificadores, patcher de armas, motor de efectos, registry, carrier de boons, FX cliente). **Hay 14 efectos implementados y verificados en el place** — declarativos, con triggers/buffs y con hooks. Antes de escribir uno nuevo, busca en §9 el que más se parezca y cópialo.
>
> **Regla de oro:** un efecto nuevo = **1 ModuleScript en `Definitions/` + 1 línea en `CardEffectBindings`**. Si para implementar un efecto sientes que necesitas editar el motor, PARA y reporta: o el efecto necesita un hook (§7), o falta un seam que se decide con el orquestador. **Nunca** edites EffectRuntime/ModifierStore/GameEventBus/ActionLibrary/BoonManager para un efecto puntual.

---

## 0. Mapa del sistema (dónde vive cada cosa)

```
game.ServerScriptService.Services.Cards.Effects   ← el framework (server)
├─ EffectsConfig        knobs (racha, proximidad, clamps de armas, DEBUG_BUS)
├─ GameEventBus         eventos normalizados (publish/subscribe)
├─ RunStateProviders    racha de kills, flags de room, salud, proximidad
├─ ConditionRegistry    predicados declarativos (eval + watch)
├─ ModifierStore        modificadores de stats por player (resolve → mult, flat)
├─ WeaponStatPatcher    escribe los atributos del arma (compone boons × cartas)
├─ PlayerStatSync       aplica MoveSpeed compuesto al Humanoid
├─ CardStats            fachada neutra para consumidores externos
├─ ActionLibrary        acciones que ejecutan los triggers (heal, buff, AoE…)
├─ EffectRuntime        EL MOTOR: attach/detach, triggers, buffs, timer-wheel
├─ EffectRegistry       valida y cataloga las definiciones al boot
└─ Definitions/         ★ AQUÍ SE CREAN LOS EFECTOS ★  (_TEMPLATE = plantilla)

game.ReplicatedStorage.Shared.Config.CardEffectBindings  ← cardId → effectId
game.ReplicatedStorage.Shared.Config.CardCatalog          ← cartas (la rareza de la carta = tier del efecto)
game.ReplicatedStorage.Shared.Config.CardsConfig           ← ENABLED (kill-switch global)
game.ReplicatedStorage.CardEffectVFX                       ← templates visuales (editor-authored)
game.ServerScriptService.Services.Cards.CardEffectDefinitions ← carrier (NO tocar: puentea boon→motor solo)
game.StarterPlayer.StarterPlayerScripts.Controllers.CardEffectFXController ← FX cliente
```

**Flujo en runtime:** carta equipada → run start → `CardLoadoutService` aplica boon `CARD_<cardId>` → `OnApply` del boon llama `EffectRuntime.attach(player, effectId, tier, cardId)` → el motor registra modifiers y suscribe triggers → muerte/retry/lobby limpian vía el ciclo de boons (`OnRemove` → `detach`). Revive restaura por snapshot (re-OnApply; attach es idempotente). **Tú no tocas nada de este flujo.**

---

## 1. Paso a paso (checklist ejecutable)

1. **Leer el efecto en el PDF de diseño** (§11 de `TKA-Sistema de Cartas Coleccionables`). Extraer: qué dispara el efecto, qué hace, en qué rarezas existe ("Niveles de rareza que tiene"), y si depende de un arquetipo de arma.
2. **Descomponerlo** en las 4 piezas del framework (§2 de esta guía): modifiers / triggers+actions / condiciones / estado propio.
3. **Verificar cobertura** contra los catálogos de §3–§6. Si TODO existe → efecto 100 % declarativo. Si falta algo → §7 (hooks) o consultar al orquestador.
4. **Crear el ModuleScript** en `Services.Cards.Effects.Definitions` con nombre = effectId en MAYÚSCULAS (ej. `RUSH`). Copiar la estructura de `_TEMPLATE` o de un spec existente (`DAMAGE` para modifiers, `LIFESTEAL` para triggers).
5. **Escribir `params` SOLO para los tiers donde el efecto existe** según el PDF (claves válidas: `Common`, `Uncommon`, `Rare`, `Epic`, `Legendary`). Todos los números balanceables van ahí, NUNCA hardcodeados en modifiers/triggers (se referencian por nombre string).
6. **Escribir `describe(p)`** — string corto en español que verá el jugador en el álbum/HUD.
7. **Crear el SET de cartas del efecto (una carta por rareza) y bindearlas.**
   **Regla "1 efecto = N cartas" (invariante de contenido):** cuando el diseño da valores por rareza (Common, Uncommon, …), NO se bindea el efecto a una carta suelta que ya existe. Se crea la **misma carta repetida en cada una de las rarezas donde el efecto existe** (mismo `displayName` en todas; lo único que cambia entre versiones es la rareza, que selecciona el `params[tier]` del spec — normalmente un %, pero puede ser cualquier número balanceable). Ejemplo: efecto con `Common/Uncommon/Rare/Epic/Legendary` → 5 cartas nuevas con el mismo nombre, una por rareza.
   - En `CardCatalog` añadir una entrada por rareza: `card_0XX = { id="card_0XX", displayName="<Nombre>", rarity="<Tier>", textureId="rbxassetid://0", coinsPerSecond=<n> }`. El catálogo es data-driven (no hay tope fijo de cartas; el `_orderedIds` y la colección se autoexpanden). `coinsPerSecond` sigue las bandas por rareza del catálogo existente.
   - En `CardEffectBindings` bindear **todas** esas cartas al MISMO effectId: `card_0XX = "EFFECT_ID"`.
   - La rareza de cada carta **debe existir en `params`** del spec (si no, el carrier warnea y skipea esa carta — no revienta, pero el efecto no aplica). Por eso el set de cartas y las claves de `params` deben cubrir exactamente las mismas rarezas.
8. **Verificar en frío**: `pcall(require)` del spec en el probe + confirmar que `EffectRegistry` lo registra (con `ENABLED=true` el boot imprime `[EffectRegistry] N efectos registrados` — debe subir en 1). Un spec inválido se skipea con warn: leer el warn, corregir.
9. **Smoke test en play** (§8) — obligatorio antes de reportar terminado.
10. **Revertir `CardsConfig.ENABLED` al valor en que estaba** (hoy el estándar es `false` hasta el lanzamiento; solo se enciende para testear).

---

## 2. Cómo descomponer cualquier efecto

Todo efecto del PDF se expresa con estas piezas (combinables):

| Pieza | Cuándo usarla | Campo del spec |
|---|---|---|
| **Modifier pasivo** | "mientras la carta esté equipada, +X% de stat" (constante o condicionado) | `modifiers` |
| **Trigger + acciones** | "al pasar EVENTO, haz COSA" (con % de chance y/o condición opcionales) | `triggers` |
| **Buff temporal** | "…por X segundos (con N stacks)" — es una ACCIÓN (`grantBuff`) dentro de un trigger | `triggers[].actions` |
| **Condición** | "si HP<35%", "con racha ≥5", "cerca de un compañero"… — se cuelga de un modifier O de un trigger | `condition` |
| **Scope de arma** | "solo con rifles/escopetas/SMG/pistolas" | `weaponScope` |
| **Estado propio** | contadores/rampas que el efecto recuerda (hits consecutivos, etc.) | `hooks` + `instance.state` |

Ejemplos de mapeo (efectos reales del PDF):
- *Rush* ("al entrar a un cuarto, +X% velocidad y recarga por Xs") → trigger `RoomEntered` → acción `grantBuff` con modifiers MoveSpeed+ReloadSpeed.
- *Last Standing* ("bajo 35 % de vida, +X% velocidad") → modifier `MoveSpeed` con `condition = {type="hpBelowPct", pct=0.35}`. Sin triggers.
- *Detonation* ("X% de chance de que al matar, explote en área") → trigger `EnemyKilled` + `chance` + acción `aoeDamage`.
- *Last Reserves* ("la última mitad del cargador hace +X% daño") → modifier `ShotDamage` percent con `condition = {type="magazineBelowPct", pct=0.5}` (§5) — el BlasterServer resuelve `ShotDamage` por disparo pasando `ammoBefore/magazineSize` en el ctx, que es lo que el predicado necesita. **Ya implementado**: ver `Definitions/LAST_RESERVES`.
- *Rifle Marksman* ("−mag, −cadencia, ++daño, solo rifles") → 3 modifiers con `weaponScope="rifle"`: `MagazineSize` y `RateOfFire` en negativo, y el daño con **`ShotDamage`** (NO `Damage`: con `weaponScope` ese statKey no aplica nunca — ver §3).
- *Coin Savings* ("bajo 35 % HP, las monedas curan") → trigger `CoinsGained` + `condition hpBelowPct` + acción `heal`.
- *Coin Excess* ("monedas valen +X% con vida llena") → modifier `CoinValue` percent con `condition = {type="hpAtMax"}`.
- *Pistol Efficiency / Infinite Mags* ("X% de no consumir bala") → modifier `AmmoSaveChance` **flat** (con `weaponScope="pistol"` en el primero). El roll lo hace el servidor del Blaster solo — no necesitas trigger.

---

## 3. Catálogo de statKeys (modifiers)

`{ statKey, valueType = "percent"|"flat", value = <number|"paramRef">, weaponScope?, condition? }`
Percent son **aditivos** entre sí: dos +10 % = ×1.20. `value` casi siempre es un string que referencia `params[tier]`.

| statKey | valueType | Efecto real | Consumidor (ya cableado) |
|---|---|---|---|
| `Damage` | percent | daño global de TODOS los disparos. **⚠️ NO admite `weaponScope`** (ver aviso abajo) | DamageCalculator |
| `ShotDamage` | percent | daño POR DISPARO, evaluado con ctx del tiro (`ammoBefore`, `magazineSize`, `archetype`, `tool`) — para efectos condicionales de cargador **y para todo daño acotado por arquetipo** | BlasterServer.onShoot |
| `DamageReduction` | flat | resta plana al daño recibido | PlayerService.takeDamage |
| `MoveSpeed` | percent | velocidad de movimiento (compone con tienda y eventos — ver aviso abajo). **⚠️ NO admite `weaponScope`** | PlayerStatSync + PowerUps |
| `RateOfFire` | percent | cadencia (compone con boons legacy) | WeaponStatPatcher |
| `ReloadSpeed` | percent | mult>1 = recarga MÁS rápida (`reloadTime = base/mult`) | WeaponStatPatcher |
| `MagazineSize` | percent/flat | tamaño de cargador (clampa ammo, nunca regala balas) | WeaponStatPatcher |
| `Spread` | percent | dispersión (mult<1 = más preciso) | WeaponStatPatcher |
| `Range` | percent | alcance | WeaponStatPatcher |
| `RayRadius` | percent | grosor del proyectil (área de las pistolas del PDF) | WeaponStatPatcher |
| `RaysPerShot` | percent | nº de perdigones (round, mín 1) | WeaponStatPatcher |
| `Recoil` | percent | escala recoilMin/Max (mult<1 = menos recoil) | WeaponStatPatcher |
| `AmmoSaveChance` | **flat only** | prob. 0..1 de NO consumir bala (clamp 0.95) | BlasterServer roll |
| `CoinValue` | percent | valor de las monedas ganadas (ctx.source disponible) | PlayerService.addCoins |

**⚠️ `MoveSpeed`: la velocidad del jugador NO es fija — nunca la hardcodees.** Un efecto de velocidad **solo declara su porcentaje**; nunca escribe `WalkSpeed` ni asume un valor base. La composición la hace el sistema:

```
WalkSpeed = velocidadBase(nivel de tienda)  ×  multDeCartas  ×  multDeEventos(Speed Rush)
```

- La **base sube con la tienda** (`PowerUps:GetMovementSpeed`, curva por nivel). Se lee **en vivo en cada aplicación**, así que un "+10%" se aplica siempre sobre la velocidad mejorada real del jugador, sin tocar nada. En el place de prueba la base estaba en 16.52, no en 16.
- Los **eventos/boosters** (Speed Rush, ×1.20 temporal) son un tercer factor multiplicativo, aparte de la tienda y de las cartas.
- Los percent de cartas son **aditivos entre sí** (`mult = 1 + Σ value×stacks`) y luego multiplican a los otros dos factores.

**Hay TRES escritores de `WalkSpeed` y los tres deben usar la MISMA fórmula:** `PowerUpsShop:_applyMovementSpeed` (compra), `PowerUpsManager:ApplyToCharacter` (spawn / grant de booster) y `PlayerStatSync.applyMoveSpeed` (cambios de modifier en runtime). Si tocas la composición de velocidad, actualiza los tres o quedan inconsistentes.

> 🐞 **Bug ya corregido (histórico, útil como advertencia):** `PlayerStatSync` escribía solo `base × cartas` y **perdía el factor de eventos**. Como corre en cada cambio de modifier de MoveSpeed, cualquier carta de velocidad borraba el bonus del evento activo. Medido con SMG_RUSH: con Speed Rush (×1.20) activo, ganar un stack de +10% **bajaba** WalkSpeed de 19.82 a 18.17 en vez de subirla a 21.81 — el jugador se volvía MÁS LENTO al recibir un buff de velocidad. Corregido y verificado (los tres escritores ya componen igual). **Al implementar un efecto de MoveSpeed, sigue verificando la composición con un booster activo en el smoke test** (`BoosterService.grant(player, "SpeedRush")`): es la clase de regresión que no da warn ni error.

Clamps de sanidad en `EffectsConfig.CLAMPS` (reload mín 0.15 s, RoF máx 1200, etc.) — el patcher los aplica solo.

**⚠️ `Damage` vs `ShotDamage`:** global → `Damage`; condicional-por-disparo (depende del cargador/tiro) → `ShotDamage`. Usar los dos para lo mismo = doble multiplicación.

**⚠️⚠️ Qué statKeys admiten `weaponScope` y cuáles NO.** El scope por arquetipo solo funciona si el CONSUMIDOR del stat resuelve pasando `ctx.archetype`. Si no lo pasa, `ModifierStore.resolve` compara `"rifle" ~= nil` y deja el record **inactivo para siempre, sin warn ni error**.

| Consumidor | ¿pasa `ctx.archetype`? | statKeys | `weaponScope` |
|---|---|---|---|
| `WeaponStatPatcher.patchTool` | Sí (`{archetype, tool}`) | `RateOfFire`, `ReloadSpeed`, `MagazineSize`, `Spread`, `Range`, `RayRadius`, `RaysPerShot`, `Recoil`, `AmmoSaveChance` | ✅ funciona |
| `BlasterServer.onShoot` | Sí (`{archetype, tool, ammoBefore, magazineSize}`) | `ShotDamage` | ✅ funciona |
| `DamageCalculator.getMultiplier` | **No** (sin ctx) | `Damage` | ❌ **nunca aplica** |
| `PlayerStatSync.applyMoveSpeed` | **No** (sin ctx) | `MoveSpeed` | ❌ **nunca aplica** |
| `PlayerService.takeDamage` | **No** (sin ctx) | `DamageReduction` | ❌ **nunca aplica** |
| `PlayerService.addCoins` | Solo `{source}`, sin archetype | `CoinValue` | ❌ **nunca aplica** |

Regla corta: **stats de ARMA aceptan `weaponScope`; stats de JUGADOR y economía no.** Para acotar por arquetipo un efecto de stat de jugador (ej. "matar con SMG da velocidad"), el filtro va en el **trigger** (`spec.weaponScope` filtra por `ctx.archetype` del evento), NO en el modifier. Ver `Definitions/SMG_RUSH`.

**Caso especial `Damage`:** Si el efecto es "+X% de daño **solo con rifles/pistolas/…**", el modifier DEBE ser `ShotDamage`, nunca `Damage`.
- Por qué: `DamageCalculator.getMultiplier` resuelve el daño global llamando `CardStats.mult(attacker, "Damage")` **sin ctx**. Sin ctx no hay `ctx.archetype`, y `ModifierStore.resolve` marca INACTIVO todo record cuyo `weaponScope ~= nil` (compara `"rifle" ~= nil` → nunca activo). El modifier queda registrado pero **jamás aplica, y no hay warn ni error**: el efecto simplemente no hace nada.
- `ShotDamage` sí funciona con scope porque `BlasterServer.onShoot` lo resuelve pasando `ctx = { archetype, tool, ammoBefore, magazineSize }`.
- Regla corta: **daño acotado por arma → `ShotDamage`. Daño global (sin `weaponScope`) → `Damage`.**
- Cómo detectarlo en el smoke test: comparar `CardStats.mult(p, "Damage")` (sin ctx) contra `CardStats.mult(p, "ShotDamage", {archetype="rifle"})`. Si el primero da `1.00` con el efecto attacheado, el modifier está muerto.
- Los demás statKeys con `weaponScope` (`RateOfFire`, `MagazineSize`, `Spread`, `Range`, `RayRadius`, `RaysPerShot`, `Recoil`, `AmmoSaveChance`) SÍ funcionan: los resuelve `WeaponStatPatcher`, que siempre pasa `ctx.archetype` leído de la Tool.

---

## 4. Catálogo de eventos (triggers)

`{ event, chance?, condition?, actions = {...} }` — el motor ya filtra por dueño del evento y por `weaponScope`.

| event | ctx que llega | Notas |
|---|---|---|
| `EnemyKilled` | `enemy, tool, archetype, distance, streak` | distance en studs desde el origen del disparo |
| `EnemyHit` | `enemy, damage, tool, archetype, distance` | damage = daño final aplicado |
| `ShotFired` | `tool, archetype, ammoBefore, ammoAfter, magazineSize, saved` | server-side, post-roll de ammoSave |
| `ReloadStarted` / `ReloadFinished` | `tool, archetype` | cancelar recarga NO dispara Finished |
| `WeaponEquipped` | `tool, archetype` | al equipar cualquier Tool |
| `PlayerDamaged` | `amount, health, maxHealth` | derivado del delta de vida |
| `PlayerHealed` | `amount, health, maxHealth, atMaxHealth` | ⚠️ curar a vida YA llena no dispara (delta 0). Si un efecto necesita "heal attempt at max" (Brain Juice), pedir el seam I-6 al orquestador |
| `HealthThresholdCrossed` | `pct, below` | solo para thresholds registrados vía provider |
| `RoomEntered` | `roomIndex, prevIndex, isLobby` | roomIndex 0 = lobby |
| `RoomCleared` | `roomIndex, tookDamageInRoom` | por cada player presente en la room |
| `CoinsGained` | `amount, total, source` | source: "kill"/"assist"/"coinReward"/"cardDupe"/"mission"/"generic" |
| `TeammateProximityEnter` / `Exit` | `teammate, distance` / `{}` | histéresis 20/24 studs (EffectsConfig) |
| `PlayerDowned` | `downCount` | DBNO |
| `PlayerRevived` | `rescuer` | revivido por compañero |
| `PlayerRevivedOther` | `target` | TÚ reviviste a alguien (Jolly Cooperation) |
| `RunStarted` / `RunEnded` | `{}` / `reason` | ciclo de vida del run |
| `StreakChanged` | `streak` | racha de kills (server-side, ventana 6 s) |

---

## 5. Catálogo de condiciones

`condition = { type = "...", <args> }` — usables en modifiers Y en triggers. Se re-evalúan solas (event-driven, con timers para expiraciones silenciosas).

| type | args | true cuando |
|---|---|---|
| `hpBelowPct` | `pct` (0..1) | vida actual < pct |
| `hpAtMax` | — | vida al máximo |
| `streakAtLeast` | `n` | racha de kills ≥ n |
| `nearTeammate` | — | hay compañero a ≤20 studs |
| `noDamageForSeconds` | `s` | lleva ≥s segundos sin recibir daño |
| `weaponArchetypeIs` | `archetype` | arma del ctx (o equipada) es rifle/pistol/smg/shotgun |
| `magazineBelowPct` | `pct` (0..1) | el disparo cae en el ÚLTIMO `pct` del cargador (`ammoBefore / magazineSize <= pct`). Para efectos tipo *Last Reserves* |

`weaponScope` del spec ya filtra por arquetipo en modifiers y triggers — usa `weaponArchetypeIs` solo para casos cruzados. Si el efecto necesita un predicado que no existe, se añade en `ConditionRegistry` (ese SÍ es un cambio al framework: consultarlo, no improvisarlo).

**⚠️ Condiciones que dependen del ctx del disparo → el modifier DEBE ser `ShotDamage`.** `magazineBelowPct` (y cualquier predicado futuro que lea `ammoBefore`/`magazineSize`/`archetype` del tiro) solo tiene su semántica plena cuando `eval` recibe el ctx. El único statKey de daño que se resuelve CON ctx es `ShotDamage` (lo pasa `BlasterServer.onShoot`); `Damage` se resuelve sin ctx, así que la condición caería a su fallback y además leería el ammo **ya decrementado** — se equivocaría justo en la bala del umbral. Esto aplica **aunque el efecto no tenga `weaponScope`** (es un motivo distinto al de §3, mismo desenlace).

*Nota de implementación de `magazineBelowPct`:* el umbral es inclusivo (`<=`), así que con `pct = 0.5` y cargador de 30, la bala nº15 YA cuenta como "última mitad". Escala con el arma (no hardcodea un número de balas): con cargador de 11 el corte cae en 5.5.

---

## 6. Catálogo de acciones (dentro de `triggers[].actions`)

| type | campos | hace |
|---|---|---|
| `heal` | `amount \| pctOfMax \| pctOfDamage`, `roundMode?` ("floor" = truncar) | cura vía PlayerService |
| `grantBuff` | `buffId, duration, maxStacks?, stackBehavior? ("refresh"\|"stack"\|"stackRefresh"), modifiers={...}` | buff temporal; los modifiers usan el MISMO formato de §3, **pero NO heredan el `weaponScope` del spec** (repetirlo en cada uno — ver abajo) |
| `grantBuffToNearbyTeammates` | igual + `radius?` | buff al player Y a compañeros cercanos (Reload Motivation) |
| `refillAmmo` | `pctOfMagazine \| amount` | repone balas sin exceder cargador |
| `aoeDamage` | `radius, damage \| damagePct, vfxId?` | daño en área a enemigos (Detonation); damagePct = % del daño del arma equipada |
| `vfx` | `vfxId, position?, data?` | solo feedback visual |

Todos los valores numéricos aceptan string-ref a `params[tier]`. Las acciones corren en orden y son pcall-safe.

**Curaciones — cómo conviven varios efectos que dan vida:**

Las curaciones **no chocan entre sí**, y eso es por diseño: `heal` es una ACCIÓN (no un modifier), así que no hay estado compartido donde colisionar. Cada efecto llama a `PlayerService.heal` por su cuenta y los montos **se suman**. Verificado: con *Shotgun Vampirism* (+12 HP por kill) y *Lifesteal* (+20 HP por golpe) equipados a la vez, un disparo letal cura 32 HP — ninguno pisa al otro. No hace falta ninguna convención de nombres (a diferencia de `buffId`, que sí es identidad compartida).

Lo que **sí** hay que respetar al escribir un efecto que cure:

1. ⚠️⚠️ **Un efecto que cura NUNCA debe dispararse con `PlayerHealed`.** `PlayerService.heal` emite `onHealthChanged`, de donde el bus deriva `PlayerHealed`: curar dentro de ese trigger se realimenta en **bucle infinito**. Si diseño pide algo tipo "al curarte, cúrate más", hay que pedir un seam con guarda al orquestador, no improvisarlo. (Mismo cuidado con cualquier acción que dispare el evento al que el propio trigger escucha.)
2. **El overheal ya está clampeado** (`math.min(maxHealth, ...)`): curar con la vida llena es un no-op seguro, no hay que chequearlo en el efecto. Ojo con la otra cara: curar a vida ya llena **no** dispara `PlayerHealed` (delta 0) — ver §4.
3. **`heal` tiene mínimo 1 HP** (`math.max(1, ...)` en ActionLibrary) siempre que el monto resuelto sea > 0. Con porcentajes chicos sobre vida baja, varias cartas de curación redondean cada una a 1 HP y suman más de lo que sugiere la aritmética. Es un detalle de balance a tener en cuenta al elegir los `params`, no un bug.
4. **`pctOfMax` escala solo con la vida máxima del jugador — esto es correcto y deseado.** `heal` lee `PlayerService.getState(player).maxHealth` **en vivo, en cada curación** (no un valor cacheado al equipar), y las mejoras de tienda actualizan ese mismo campo vía `setMaxHealth`. O sea: una carta de "+12% de vida" cura 12 HP con 100 de vida máxima y 18 con 150, sin tocar nada. Lo único a tener en cuenta es al **fijar los `params`**: no hagas la cuenta mental asumiendo 100 de vida máxima ni leas los resultados de un test como si fueran HP fijos (en el place de prueba la vida máxima estaba en 102, así que un 3% daba 3.06 → 3 HP).

**Buffs — 3 cosas importantes:**
1. `buffId` es único por player: dos cartas que otorguen el MISMO `buffId` comparten el buff (a veces deseable; si no, prefija con el effectId: `"RUSH_BUFF"`).
2. ⚠️⚠️ **Los modifiers de un `grantBuff` NO heredan el `weaponScope` del spec — hay que repetirlo en CADA uno.** Si el efecto es "solo escopetas/rifles/…" y el buff omite `weaponScope`, el buff aplica a **TODAS las armas** (falla en silencio, sin warn).
   - Por qué: `EffectRuntime.attach()` registra los modifiers pasivos con `weaponScope = m.weaponScope or spec.weaponScope` (sí hereda), pero `EffectRuntime.grantBuff()` los registra con `weaponScope = m.weaponScope` **a secas**. Es estructural: `grantBuff(player, buffSpec, params)` nunca recibe el spec, así que no puede conocer su scope.
   - Correcto:
     ```lua
     { type = "grantBuff", buffId = "X_BUFF", duration = "duration",
       modifiers = {
           { statKey = "Spread", valueType = "percent", value = "spreadPct", weaponScope = "shotgun" },
           { statKey = "Range",  valueType = "percent", value = "rangePct",  weaponScope = "shotgun" },
       } },  -- ↑ weaponScope repetido en CADA modifier, aunque el spec ya lo declare
     ```
   - Cómo detectarlo en el smoke test: tras otorgar el buff, resolver el stat con un `ctx.archetype` de OTRO arquetipo. Si no da `1.00`, el scope se perdió.
   - Es la misma familia de fallo que `Damage` + `weaponScope` (§3): el scope se pierde sin avisar.
3. ⚠️ **Pendiente conocido:** los buffs NO se limpian en muerte (solo expiran por duración o en detachAll/salida). Para buffs cortos (≤10 s) es irrelevante. Si implementas el PRIMER efecto con buffs largos, avisa al orquestador para cablear el hook muerte→limpieza (está anotado como deuda de E3).

**VFX (Regla #2 del proyecto — editor-authored):** el código NUNCA construye visuales con Instance.new. Si el efecto lleva VFX: usar la acción `vfx`/`vfxId` y pedir al usuario que autore el template en `ReplicatedStorage.CardEffectVFX.<vfxId>` (Model/Part con ParticleEmitters; atributos opcionales `EmitCount`, `Lifetime`). Si el template no existe, el cliente warnea una vez y degrada en silencio — el efecto de gameplay funciona igual.

---

## 7. Efectos exóticos: hooks (escape hatch)

Para lógica que lo declarativo no cubre (rampas por objetivo, máquinas de estado tipo SMG All In, rebotes tipo Ricochet):

```lua
hooks = {
    onAttach = function(instance) end,               -- setup; instance.state = tu memoria libre
    onDetach = function(instance) end,               -- teardown de lo TUYO (timers propios, etc.)
    onEvent  = function(instance, eventName, ctx)    -- corre DESPUÉS de las actions declarativas
        -- instance.player / .params / .state / .runtime disponibles
    end,
}
```

Reglas de los hooks:
- El estado SOLO en `instance.state` (nunca upvalues de módulo — se comparten entre players).
- Helpers listos: `instance.runtime.helpers.rampPerTarget(instance, key, target, cap)` (Rifle-Focused Fire) y `.counter(instance, key, delta)` (Coin Combo).
- Para stats: registra modifiers vía `instance.runtime.grantBuff(...)` o —si es imprescindible— `ModifierStore.add` con `sourceId = "EFFECT_"..instance.instanceId` (así el detach automático los limpia).
- Acciones de mundo: reusar `ActionLibrary` (require de `script.Parent.Parent.ActionLibrary` desde la definición) — NUNCA llamar servicios de gameplay directo desde un hook.
- Sigue siendo el mismo ciclo de vida: attach/detach/muerte/revive ya resueltos.

---

## 8. Verificación (obligatoria — no reportar "terminado" sin esto)

**Preparación:** `CardsConfig.ENABLED = true` temporal. Opcional `EffectsConfig.DEBUG_BUS = true` (imprime cada evento del bus).

**⚠️ Trampas de tooling conocidas (te van a morder si las ignoras):**
1. `execute_luau` (probe MCP) corre en **VM separado**: `require()` desde el probe da OTRA instancia del módulo (estado no compartido). Para testear estado del juego, crear un **Script temporal en ServerScriptService** (corre en el VM real) con prints, y borrarlo al final. Las INSTANCIAS (BindableEvents, atributos, Tools) sí se comparten con el probe.
2. `script_grep` puede devolver **números de línea stale** tras edits — matchear por string exacto, no por línea.
3. Reiniciar Studio **recarga el place guardado**: edits de sesión no guardados se pierden (incluido el flip de ENABLED). Verificar en el boot `[EffectRuntime] Ready` vs `Cards OFF` antes de confiar en cualquier test.
4. Si delegaste código a un subagente: **verificar SIEMPRE el place con script_read** — ha habido reportes falsos con 0 ediciones y muertes a mitad de tarea.

**Test mínimo (patrón del Script temporal):**
```lua
-- _EfectoXSmokeTest (Script en ServerScriptService, BORRAR al final)
task.wait(9)
local p = game.Players:GetPlayers()[1] or game.Players.PlayerAdded:Wait()
local Runtime = require(game.ServerScriptService.Services.Cards.Effects.EffectRuntime)
local Bus     = require(game.ServerScriptService.Services.Cards.Effects.GameEventBus)
-- attach directo (sin pasar por cartas) en el tier a probar:
print(Runtime.attach(p, "MI_EFECTO", "Rare", "test_instance"))
-- disparar el trigger manualmente por el bus con un ctx representativo:
Bus.publish("RoomEntered", p, { roomIndex = 2, isLobby = false })
task.wait(1)
-- imprimir lo observable: ModifierStore.resolve / atributos de la Tool /
-- WalkSpeed / vida / getActiveInstances(p) según lo que haga el efecto...
Runtime.detachAll(p)
-- confirmar que TODO volvió a base (anti-drift).
```

**Checklist de aceptación por efecto:**
- [ ] Boot sin errores; `[EffectRegistry]` cuenta el efecto nuevo.
- [ ] attach aplica lo esperado en los números del tier probado (verificar al menos 2 tiers).
- [ ] El trigger dispara con su chance/condición y NO dispara cuando la condición es falsa.
- [ ] `weaponScope`: probar que un arma de OTRO arquetipo no recibe el efecto.
- [ ] Buffs: stackean hasta `maxStacks`, expiran a tiempo, y el stat vuelve a base exacta.
- [ ] `detachAll` deja cero residuo (stats en base, `getActiveInstances` vacío).
- [ ] Flujo real de cartas: bindear a una carta, equiparla, disparar `RoomSystem.RunStarted:Fire(player)` y confirmar que el pipeline completo aplica (el carrier ya funciona — esto valida TU binding y params del tier de esa carta).
- [ ] `ENABLED=false` → el efecto no existe (registry vacío, cero warns nuevos).
- [ ] Revertir flags (ENABLED al estado previo, DEBUG_BUS=false) y borrar el Script de test.

---

## 9. Ejemplos completos de referencia

**Ya en el place** — 14 efectos. Busca el que más se parezca al tuyo y cópialo; casi todo patrón nuevo ya tiene precedente. `_TEMPLATE` es el schema completo comentado.

*Declarativos — solo modifiers pasivos:*
- `DAMAGE` — modifier pasivo mínimo (percent, global). **El más simple: empieza acá.**
- `IRON_SKIN` — `flat` en vez de `percent`.
- `RATE_OF_FIRE`, `SCATTER` — stats de arma vía WeaponStatPatcher.
- `INFINITE_MAGS` — `AmmoSaveChance` flat (el roll lo hace el Blaster solo, sin trigger).
- `PISTOL_EFFICIENCY` — igual que el anterior + `weaponScope`. **Ejemplo de efecto que NO existe en las 5 rarezas** (solo Rare/Epic/Legendary → su set tiene 3 cartas).
- `WORLDS_SMALLEST_GUN` — varios modifiers con trade-off (+`RayRadius` / −`MagazineSize`), scoped a pistol.
- `RIFLE_MARKSMAN` — 3 modifiers con trade-off scoped a rifle. **Referencia de por qué el daño acotado por arma va en `ShotDamage` y no en `Damage`** (§3).
- `LAST_RESERVES` — modifier con **condición sobre el ctx del disparo** (`magazineBelowPct`). Referencia de condición + `ShotDamage` (§5).
- `CANT_STOP` — modifier con **condición de estado del run** (`streakAtLeast`) + `weaponScope`. **Referencia de condición que se re-evalúa sola**: el `watch` re-parchea el atributo del arma en cada flanco, incluso cuando la racha expira por timeout sin ningún evento de por medio. Cero timers propios en el efecto.

*Con triggers y acciones:*
- `LIFESTEAL` — trigger + acción `heal` con `roundMode`.
- `DETONATION` — trigger `EnemyKilled` + `chance` + acción `aoeDamage` con `vfxId`. **Referencia de trigger probabilístico.**
- `SHOTGUN_PRECISION` — trigger + acción `grantBuff` (buff temporal con 2 modifiers). **Referencia de buffs**, y del gotcha de repetir `weaponScope` dentro del buff (§6).
- `SMG_RUSH` — `grantBuff` con **stacks** (`maxStacks` + `stackBehavior="stackRefresh"`) sobre `MoveSpeed`. **Referencia del caso inverso al anterior**: el modifier va SIN `weaponScope` (MoveSpeed no lo admite, §3) y el filtro por arma vive en el trigger.

*Con hooks (estado propio, §7):*
- `SHOTGUN_VAMPIRISM` — hook que filtra por `ctx.distance` y cura vía `ActionLibrary.run`. **Referencia de "el catálogo de condiciones no tiene el predicado que necesito, pero un hook lo resuelve sin tocar el framework"** — y de cómo llamar una acción desde un hook.
- `RIFLE_FOCUSED_FIRE` — rampa por objetivo con el helper `rampPerTarget` + `setStacks`. **El hook más corto si necesitas estado persistente.**
- `SMG_ALL_IN` — máquina de estado completa (rampa temporal por ráfaga sostenida, reset por recarga y por soltar el gatillo, sin loops propios).

*Patrón compartido por los dos hooks:* un solo modifier registrado en `onAttach` con `value` = el máximo y `stacks` = la fracción/nivel actual (`stacks` acepta fraccionarios), actualizado con `ModifierStore.setStacks` desde `onEvent`. `sourceId = "EFFECT_"..instance.instanceId` para que el detach del motor lo limpie solo. Los triggers pueden ir **sin `actions`**: sirven solo para que `onEvent` reciba ese evento, ya filtrado por `weaponScope`.

**Ejemplo nuevo típico (Rush, PDF §11.1):**
```lua
-- Definitions/RUSH.lua
return {
    id = "RUSH",
    displayName = "Rush",
    -- PDF: existe en rarezas 1,2,3,4 → Common..Epic (números de balance = placeholder
    -- razonable si diseño no los fijó; TODOS tuneables aquí sin tocar lógica).
    params = {
        Common   = { speedPct = 0.06, reloadPct = 0.06, duration = 4 },
        Uncommon = { speedPct = 0.10, reloadPct = 0.10, duration = 5 },
        Rare     = { speedPct = 0.15, reloadPct = 0.15, duration = 6 },
        Epic     = { speedPct = 0.22, reloadPct = 0.22, duration = 7 },
    },
    describe = function(p)
        return string.format("Al entrar a un cuarto: +%d%% velocidad y recarga por %ds",
            math.floor(p.speedPct * 100 + 0.5), p.duration)
    end,
    triggers = {
        {
            event = "RoomEntered",
            condition = nil, -- opcional: p.ej. no dispara en lobby → hook o condición futura; el ctx trae isLobby
            actions = {
                { type = "grantBuff", buffId = "RUSH_BUFF",
                  duration = "duration", maxStacks = 1, stackBehavior = "refresh",
                  modifiers = {
                      { statKey = "MoveSpeed",   valueType = "percent", value = "speedPct" },
                      { statKey = "ReloadSpeed", valueType = "percent", value = "reloadPct" },
                  } },
            },
        },
    },
}
```
Más `card_0XX = "RUSH"` en `CardEffectBindings`. Nada más.

*(Nota fina para RUSH real: `RoomEntered` también dispara al volver al lobby — si diseño no lo quiere, filtrarlo requiere condición sobre ctx.isLobby; hoy eso se hace con un `hooks.onEvent` que chequea `ctx.isLobby` y otorga el buff vía `instance.runtime.grantBuff`, o pidiendo un predicado `ctxFlag` al orquestador.)*

---

## 10. Qué NO hacer (resumen de invariantes)

- ❌ Editar el núcleo del framework, BoonManager, BlasterServer, PlayerService o Main para un efecto puntual.
- ❌ Lógica de efecto en el CLIENTE o confiar en datos del cliente (todo server-authoritative; el cliente solo recibe atributos replicados y FX).
- ❌ `Instance.new` de visuales en código (Regla #2: templates en el editor).
- ❌ Números de balance hardcodeados fuera de `params`.
- ❌ Loops propios (Heartbeat/while) en un efecto — usa buffs con duración, condiciones con timers integrados, o consulta si de verdad hace falta.
- ❌ Tocar `_ammo`/atributos de arma directo desde un efecto — para eso están `refillAmmo` y los statKeys del patcher.
- ❌ Dejar `ENABLED` o `DEBUG_BUS` flipeados, o Scripts de test sin borrar.
- ❌ Reportar "terminado" sin el checklist de §8.
- ❌ Curar dentro de un trigger `PlayerHealed` (bucle infinito de realimentación — §6). En general: que una acción dispare el mismo evento al que su trigger escucha.
- ❌ Omitir `weaponScope` en los modifiers de un `grantBuff` creyendo que lo hereda del spec. No lo hereda (§6): el buff acabaría aplicando a todas las armas, en silencio.
- ❌ Combinar el statKey `Damage` con `weaponScope` (del spec o del propio modifier). No aplica NUNCA y falla en silencio — para daño acotado por arquetipo va `ShotDamage` (§3).
- ❌ Bindear un efecto con valores por rareza a UNA sola carta suelta. Si el diseño da valores por rareza, se crea el **set completo**: la misma carta (mismo `displayName`) repetida en cada rareza donde el efecto existe, todas bindeadas al mismo effectId (§1 paso 7).
