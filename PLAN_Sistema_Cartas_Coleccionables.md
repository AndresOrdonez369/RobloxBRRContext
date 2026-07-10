# Plan — Sistema de Cartas Coleccionables (Havoc Hotel)

> **Autor del diseño original:** Camilo Medina · **Plan técnico:** sesión Claude 2026-07-07
> **Place objetivo:** Studio `Patio de recreo de terror de Brainrot` (place de trabajo del sistema de cartas).
> **Estado del feature:** parcialmente implementado y en HOLD detrás de `CardsConfig.ENABLED = false`.
> **Este documento vive en el repo del proyecto para ser accesible en futuras sesiones de Claude.**

---

## 0. Cómo usar este documento

Este `.md` es la **fuente de verdad** del feature de cartas. Contiene:
1. Qué ya existe en el place (inventario exacto de archivos).
2. El delta entre lo construido y el diseño nuevo (el punto crítico).
3. Las decisiones de diseño ya confirmadas por el dueño del juego.
4. La arquitectura objetivo, módulo por módulo.
5. Reglas de rendimiento (no-lag), balanceo (para diseño) y escalabilidad.
6. Fases de implementación testeables.
7. Trampas conocidas del codebase que aplican a este feature.
8. Preguntas abiertas pendientes de definir.

**Regla de oro de este proyecto:** todos los cambios de código Roblox se implementan vía el subagente `roblox-systems-scripter` sobre el place activo. Este documento es SOLO plan; no toca código.

**Regla de oro #2 — VISUALES EN EL EDITOR, NO POR CÓDIGO:** todo lo que se ve y debería poder ajustarse a mano (prompts, texto/BillboardGui, luces, Highlights, modelos, layouts de UI estáticos, posiciones) se **crea y edita en el editor de Studio** como instancias persistentes. El código **solo las referencia** (las busca por nombre/tag y les conecta comportamiento), **nunca las genera con `Instance.new`**. Generar visuales por código hace imposible ajustarlos sin tocar Luau. Excepción única: feedback **dinámico/data-driven** que depende del estado del run (ej. los chips de "cartas depositadas" del cashout, el reveal, las fichas de rareza del HUD) — eso sí se construye por código porque su contenido cambia en runtime. Los placeholders generados por código solo son aceptables como fallback temporal de testeo, nunca como la solución final.

---

## 1. Objetivo y principios no-negociables

Crear una **capa de colección roguelike** encima del loop de combate existente: matar enemigos suelta cartas, el jugador las recoge por colisión, quedan **pendientes** durante el run, y solo se vuelven **permanentes** al hacer cashout (estación cada N cuartos o al escapar). Las cartas guardadas se ven en un álbum, se pueden **equipar 3** como pseudo-powerups, y aparecen en una estantería del lobby.

Cuatro principios que **todo** en este plan debe respetar:

| Principio | Qué significa en la práctica |
|---|---|
| **Modular** | Cada responsabilidad en su propio ModuleScript/Controller. Nada de lógica de cartas mezclada en combate, salas o UI existente. El feature entero se puede apagar con un flag. |
| **Escalable** | Agregar una carta, un efecto, una estación o una misión = editar UNA tabla de config data-driven. Cero código nuevo para contenido nuevo. |
| **Óptimo (sin lag)** | Cero polling per-frame en el servidor para lógica de cartas. Todo event-driven. Las animaciones de cartas usan UN solo loop compartido (ya existe). Sync incremental, no full-dumps. |
| **Balanceable por diseño** | Todos los números (drop chance, pesos por rareza, curva de Luck, magnitud de efectos, cada cuántos cuartos hay estación) viven en configs planas y comentadas. El área de diseño no debe leer lógica para tunear. |

---

## 2. Estado actual — lo que YA existe (y sirve)

Todo esto ya está construido en el place, funcional, y **apagado** por `CardsConfig.ENABLED = false`. **No borrar nada.** Se reutiliza casi todo.

### 2.1 Configs (ReplicatedStorage.Shared.Config)

| Archivo | Qué hace | ¿Se reusa? |
|---|---|---|
| `CardsConfig` | Kill-switch global (`ENABLED`). Apaga UI + drops + cartas de mundo. | **Sí.** Se mantiene como master flag del feature. |
| `CardCatalog` | Catálogo data-driven de 18 cartas: `{ id, displayName, rarity, textureId, coinsPerSecond }`. Helpers `get/listIds/getByRarity/count`. `coinsPerSecond` está almacenado pero NO usado (seam para ingreso pasivo futuro). Texturas placeholder `rbxassetid://0`. | **Sí.** Se extiende con campo de efecto (ver §5.4). |
| `CardRarityConfig` | 5 tiers: Common, Uncommon, Rare, Epic, Legendary. Color por tier (marco UI + glow del drop) + displayName + order. | **Sí.** Base para el weighting de Luck. |
| `CardDropConfig` | Espejo desacoplado de LootConfig. Por `enemyConfigId`: `dropChance` (0..1) + `pool` de `{cardId, weight}`. Elige 1 carta con weightedRandom. | **Sí.** Se le agrega la modulación por Luck. |

### 2.2 Servicios servidor (ServerScriptService.Services.Cards)

| Archivo | Qué hace | ¿Se reusa? |
|---|---|---|
| `CardService` | **Dueño único** de la colección + persistencia write-through vía `SessionService.getData(player).cards` (`{[cardId]=count}`) + remotes de sync + `onCardCollected` Signal (seam para misiones). Server-authoritative: `collect(player, cardId)` suma 1 y persiste. | **Sí, con cambio de rol:** `collect()` pasa a ser el paso de **COMMIT permanente** que solo dispara el cashout, no el touch. |
| `CardDropService` | Escucha `CombatService.onEnemyKilled`, rollea dropChance, elige carta con weightedRandom, construye el prefab visual (Part vertical + SurfaceGui/ImageLabel + SelectionBox + PointLight por rareza), lo spawnea con `OwnerUserId`/`CardId`, pickup por `Touched` **server-side** que hoy llama `CardService.collect` directo. Limpieza por timeout (20s). | **Sí, con cambio:** el pickup ya NO llama `collect` (permanente); llama `CardRunService.addPending` (ver §5.1). |
| `WorldCardService` | Cartas escondidas **compartidas** (visibles para todos), montadas sobre anchors tagged `WorldCard`. Claim 1 vez por jugador por run, re-arma al volver al lobby (`LobbyZone.Touched`), animación bobbing+spin **server-side** (un Heartbeat para todas). No se destruyen al reclamar. | **Sí, con cambio:** al reclamar llama `addPending` en vez de `collect` (decisión confirmada: mundo también pasa por cashout). |

### 2.3 Controllers cliente (StarterPlayer.StarterPlayerScripts.Controllers)

| Archivo | Qué hace | ¿Se reusa? |
|---|---|---|
| `CardCollectionController` | Abre/cierra `CardCollectionGui` (botón HUD `CardsButton`), grid paginado de TODO el catálogo (owned/locked), cache local vía `CardCollectionSync`, toast "¡Nueva carta!" vía `CardCollected`. | **Sí, se expande** con la sección Equipado + filtros por rareza + búsqueda (mockup). |
| `CardVisibilityController` | Cartas-drop (`workspace._Cards`, tag `DropCard`) con `OwnerUserId`: oculta con `LocalTransparencyModifier=1` las que no son tuyas + anima bobbing/spin local de las tuyas. | **Sí, tal cual.** Ya resuelve la visibilidad per-player pedida. |
| `BoonCardStyler` | Estiliza frames de carta por rareza (reusado por el HUD de boons). | **Sí,** reutilizable para estilizar celdas de carta por rareza en el álbum. |

### 2.4 Datos y remoting existentes

- **Persistencia:** `SessionService` (write-through + autosave periódico staggered). La colección vive en `data.cards`. Ya persiste. Ver memoria `[[autosave-periodic-session]]` y `[[launch-data-wipe-store-versioning]]`.
- **Remotes (Shared.Types.Remotes):** `CARD_COLLECTION_SYNC = "CardCollectionSync"`, `CARD_COLLECTED = "CardCollected"`. Patrón: `Remotes.fireClient` / `Remotes.onClient`.
- **Signal:** `CardService.onCardCollected:Fire(player, cardId, newCount)` — seam listo para misiones.

### 2.5 Sistemas vecinos que se REUSAN (no son de cartas pero encajan)

| Sistema | Path | Uso en cartas |
|---|---|---|
| **BoonManager** | `Services.BoonManager` | Motor de efectos data-driven del run: apply/remove/trigger, `getStatMultiplier`/`getStatFlat` (ya **leídos por combate**), stackeo, exclusiones, cleanup al morir. **Las cartas equipadas se aplican como boons.** |
| **BoonDefinitions** | `Services.BoonDefinitions` | Schema de efecto de referencia (statKey, valueType, value, triggerEvent, OnApply/OnTrigger/OnExpire). El schema de efecto de carta lo copia. |
| **BaseShopManager** | `Services.Shops.BaseShopManager` | Patrón para bindear un `ProximityPrompt` a una Part del lobby y disparar un callback. **La estación de cashout usa este patrón** (`BindLobbyPart`). |
| **MissionTracker** | `ServerScriptService.MissionTracker` | Sistema de misiones data-driven (`ALL_QUESTS`, tipos Kill/Damage/Clear/etc, rewards Gold/Boon/Title). **Las misiones de cartas se agregan aquí** con un nuevo `Type`. |
| **RoomManager** | `ServerScriptService.RoomSystem.RoomManager` | Detecta escape del run: al tocar `RoomEscape.LobbyTeleportZone` incrementa `Escapes` y dispara `EscapeVictory`. **Hook del cashout-al-escapar.** También expone índice de cuarto por jugador (para "cada 5 cuartos"). |
| **Tienda de power-ups** | Shop system (`ShopService` + shops concretos tipo `CoinUpgradesShop`) | Hogar de la nueva stat **Luck** (§5.5). |

---

## 3. EL DELTA CRÍTICO — modelo de 2 fases

Esta es la diferencia central entre lo construido y el diseño nuevo, y el corazón del refactor.

```
IMPLEMENTACIÓN ACTUAL (1 fase):
  tocar carta ──► CardService.collect() ──► PERMANENTE + persistido    ← sin riesgo, sin tensión

DISEÑO NUEVO (2 fases):
  tocar carta ──► PENDIENTE (HUD, en memoria, NO persiste)
                     │
                     ├─ estación de cashout (cada N cuartos)  ──┐
                     ├─ escapar del run                        ──┼─► COMMIT: CardService.collect() ──► PERMANENTE
                     │                                          ─┘
                     └─ MORIR sin revivir ──► SE PIERDEN (decisión confirmada: roguelike puro)
```

**Traducción a código:** se introduce una capa intermedia `CardRunService` que mantiene el **inventario pendiente por jugador por run** (tabla efímera en servidor, nunca a DataStore). Los tres orígenes de carta (drop de enemigo, carta de mundo, y cualquier futuro) llaman `CardRunService.addPending(player, cardId)` en vez de `collect`. El `collect` permanente **solo** lo dispara el cashout (estación o escape), que hace flush del pendiente a `CardService`.

Esto respeta la sección 4 del diseño ("una carta se considera recolectada cuando el jugador la deposita en un terminal") de forma uniforme para todos los orígenes.

---

## 4. Decisiones de diseño confirmadas (dueño del juego)

Estas cuatro respuestas **fijan** la arquitectura. No re-litigar sin autorización.

1. **Muerte y cartas pendientes (regla completa, PDF §9).** El pendiente vive/muere según el resultado real de la muerte:
   - **Muere y vuelve al lobby (Retry) → pierde TODAS las cartas pendientes.** Roguelike puro, máxima tensión.
   - **Muere y REVIVE → recupera las cartas pendientes**, para darle chance de llegar a una estación y hacer cashout.
   - **DBNO (down-but-not-out) y se recupera → mantiene las cartas pendientes.**
   Implementación: `CardRunService` limpia el pendiente en el MISMO punto donde `BoonManager` hace cleanup al morir, respetando la excepción de estado revivible de `DeathService` (`getPendingState == "Dying"`): no limpiar mientras pueda revivir/levantarse; limpiar solo cuando el resultado es Retry→lobby. Espeja exactamente la lógica de boons (que también se restauran al revivir vía `restoreSnapshot`).

2. **Drop chance = stat "Luck", en la tienda de power-ups.** No es solo drop chance: **Luck sube tanto la probabilidad de soltar carta como el sesgo hacia rarezas altas.** "A mayor suerte, más probabilidad de cartas más raras." Luck es una stat del jugador (persistida), comprada/subida con oro en la tienda de power-ups. Es un multiplicador global sobre `CardDropConfig`.

3. **Efectos de cartas equipadas → REUSAR el pipeline de Boons.** Cada carta con efecto mapea a una definición estilo-boon. Al iniciar el run se aplican las 3 cartas equipadas como boons permanentes-del-run vía `BoonManager.apply`. Combate ya lee esos multiplicadores. Cero re-cableado, cero riesgo de romper balance existente.

4. **Cartas escondidas del mundo → también pendiente + cashout.** Consistencia total: todo origen de carta pasa por la capa de pendiente. Tocar carta de mundo = agregar a pendiente, no guardar directo.

5. **Estaciones de cashout = anchors manuales.** Diseño coloca el tag `CardCashoutStation` donde quiera. Guía default: **una cada 5 cuartos**, pero movible/editable a mano en cualquier momento por diseño. Los anchors son la fuente de verdad; el "5" es solo referencia de colocación, no una regla rígida en código.

6. **Duplicados → +1 al contador de unidades del álbum Y coins por rareza (PDF §10.1).** Un duplicado se recoge y pasa por pendiente igual que una carta nueva. Al hacer cashout de un duplicado: **(a)** suma **+1 al indicador de unidades coleccionadas** de esa carta dentro del Álbum (el `count` existente `data.cards[cardId]`, badge "xN") — la carta NO desaparece de la colección, se acumulan unidades; **(b)** además otorga **coins** cuyo monto depende **solo de la rareza** (ej. Common 100, Rare 200, Epic …), **fácilmente modificable por rareza**. → No hay niveles ni escalado de poder por duplicados; el count es solo un tally de colección + fuente de coins.

7. **TODAS las cartas dan powerup, incluidas las comunes.** Lo que cambia entre rarezas es la **magnitud** del efecto a favor del jugador. Ej: un efecto lifesteal Common devuelve 3% del daño como vida; el mismo efecto en una rareza superior devuelve 20%, 40%, etc. → efectos data-driven con valor que **escala por rareza**.

8. **Rarezas 100% data-driven.** V1 usa 5 rarezas, pero el sistema debe permitir **agregar o quitar rarezas sin tocar lógica**. Todo lo que depende de rareza (filtros del álbum, coins por duplicado, sesgo de Luck, colores) se deriva de `CardRarityConfig.Order`, nunca hardcodeado. Esto también resuelve la discrepancia del mockup (4 filtros) → los filtros se **generan desde config**.

9. **Luck tiene tope, y en el tope la mejor rareza = 100%.** Existe `LUCK_MAX`. Si el Luck del jugador == `LUCK_MAX`, la probabilidad de que salga la **mejor rareza** es 100%. **La tienda de oro NUNCA debe permitir llegar al tope**: el upgrade con oro se capa por debajo (ej. si `LUCK_MAX = x50`, la tienda sube hasta ~x30–x40). El tramo restante (ej. el x10 final) queda **reservado para venta en tienda premium** — *esto es V-futura, fuera de este plan, solo se deja el seam y la nota*.

10. **Drops no recogidos se despawnean al avanzar de cuarto o escapar.** Si el jugador pasa al siguiente cuarto (o escapa) sin recoger una carta-drop suya, esa carta se **despawnea y libera** de inmediato (no espera el timeout). Esto mantiene el mundo limpio y acota memoria en runs largos.

11. **Carta incógnita en el mundo (PDF §3.3).** El jugador NO sabe qué carta cayó antes de recogerla. **Todas** las cartas en el mundo (drops de enemigo y cartas escondidas) se renderizan con un **visual genérico de "incógnita"** (misma carta misteriosa para todas), no la textura real. La identidad se **revela solo al recogerla**. Esto cambia el render de los prefabs actuales (que hoy muestran `textureId`). Ver §5.12.

12. **Reveal al recoger + pendiente por rareza junto a los boons (PDF §4).** Al recoger, la carta se **revela** un instante en pantalla (con imagen completa, o solo su rareza — **TBD**, ver §10.9). Tras el reveal, la carta pendiente queda representada de forma compacta **solo por su rareza, junto a los boons** en el HUD. Ver §5.3.

---

## 5. Arquitectura objetivo — componentes

Convención: **[NUEVO]** = crear, **[MOD]** = modificar existente, **[REUSO]** = usar tal cual.

### 5.1 [NUEVO] `CardRunService` — inventario pendiente por run

`ServerScriptService.Services.Cards.CardRunService`

- **Estado:** `_pending[userId] = { [cardId] = count }` en memoria. **Nunca** a DataStore.
- **API:**
  - `addPending(player, cardId)` → valida contra catálogo, suma 1, `fireClient(CARD_PENDING_SYNC, player, {[cardId]=count})` (incremental). Retorna nuevo count pendiente.
  - `getPending(player)` → tabla viva.
  - `flush(player)` → recorre el pendiente. Por cada `(cardId, n)` aplica la **regla de duplicados** (decisión #6):
    - Consulta si el jugador **ya posee** la carta (`CardService.getCollection`).
    - **Todos los ejemplares suman al contador de unidades** del álbum: `data.cards[cardId] += n` (el badge "xN" existente). Un `CardService.collectMany(player, {[cardId]=n})` batch hace el incremento + sync (§7.8).
    - **Coins por duplicado (PDF §10.1):** de esos `n`, los que sean **duplicados** (todos si ya poseías la carta antes de este run; `n-1` si es la primera vez que la obtienes) generan **coins según la rareza** vía `PlayerService.addCoins` usando `CardDupeRewardConfig` (ver §5.11). El "primer ejemplar" de una carta nunca-antes-poseída no da coins (es la carta en sí).
    - Al terminar: limpia el pendiente, dispara `CARD_CASHOUT_DONE` con resumen (cartas nuevas, duplicados, coins ganados, rarezas) para el feedback. Batchea el commit (un solo `markDirty`/sync). Retorna el resumen.
  - `clearPending(player)` → descarta sin guardar (muerte).
- **Cleanup:** se engancha a `PlayerService.onDied` con la MISMA lógica de excepción que `BoonManager` (chequear `DeathService.getPendingState == "Dying"`). Al salir del juego: descartar.
- **Optimización:** todo tabla-en-memoria + eventos. Cero loops. El sync al cliente es incremental (solo la carta que cambió), igual que el patrón de `CardService`.

### 5.2 [NUEVO] `CardCashoutService` — estaciones + escape

`ServerScriptService.Services.Cards.CardCashoutService`

- **Estaciones = anchors manuales** (decisión #5). Diseño coloca el tag `CardCashoutStation` donde quiera; guía default: una cada 5 cuartos, pero 100% movible a mano. Al init, `CardCashoutService` recorre `CollectionService:GetTagged("CardCashoutStation")` (+ `GetInstanceAddedSignal`) y bindea cada uno con el patrón `BaseShopManager:BindLobbyPart(part, onTriggered, "Guardar Cartas")` → `ProximityPrompt` con `HoldDuration` **corto** (config, ej. 0.6s; el diseño pide "hold muy corto"). Al `Triggered` → `CardRunService.flush(player)` + feedback.
- **Placeholder de estación (decisión #6-arte):** si un anchor tagged no tiene modelo, `CardCashoutService` genera un **cubo placeholder** llamativo (Part con color/glow + BillboardGui "Guardar Cartas") para poder testear. El dueño reemplazará después por la máquina real (tamaño de kiosko de lobby); basta con re-taguear el modelo final. No hardcodear el visual: si el anchor ya trae modelo, usarlo tal cual.
- **Cashout al escapar:** se engancha al punto de escape de `RoomManager` (touch de `RoomEscape.LobbyTeleportZone` / signal de run completado). Al escapar → `flush` automático antes de/junto con el teleport al lobby. Así "escapar = asegurar cartas" sin interacción.
- **`CASHOUT_EVERY_N_ROOMS` = 5** queda como constante documental/guía de colocación (no gobierna spawns; los anchors mandan). Se deja para que diseño tenga la referencia.
- **Feedback (PDF §6):** al completar cashout, `CARD_CASHOUT_DONE` → doble feedback:
  1. **Encima de la estación:** un `BillboardGui` sobre la Part de la estación que muestra **cuáles cartas fueron depositadas** (miniaturas/rarezas de lo asegurado). Requisito explícito del PDF: "el feedback debe indicar arriba del cashout station cuáles cartas fueron depositadas". Visible para el jugador que depositó (y opcionalmente para los demás, decisión menor).
  2. **En pantalla del jugador:** animación llamativa (partículas + sonido + resumen: cartas nuevas, unidades, duplicados → coins ganados).
- La estación real debe ser **llamativa, tamaño de kiosko de lobby** (modelo hecho por diseño/arte; V1 usa el cubo placeholder).

### 5.3 [NUEVO] HUD de cartas pendientes

`StarterPlayer.StarterPlayerScripts.Controllers.CardPendingHUD` (o dentro del HUD existente)

- **Reveal al recoger (PDF §4):** al recoger una carta, mostrar un **reveal momentáneo** en pantalla (imagen completa de la carta, o solo su rareza — **TBD §10.9**), con animación breve. Es el momento "sorpresa" que paga la incógnita del mundo (§5.12).
- **Pendiente por rareza junto a los boons (PDF §4):** tras el reveal, la carta pendiente se representa de forma **compacta, solo por rareza, al lado de los boons activos** del HUD. Reutilizar/extender el área de `BoonHUD` (`StarterGui.BoonHUD`) — las cartas pendientes son "fichas" de rareza junto a los íconos de boon. NO mostrar imagen completa aquí, solo un pip/ficha coloreada por tier (usa color de `CardRarityConfig`).
- Escucha `CARD_PENDING_SYNC` (incremental) y `CARD_CASHOUT_DONE` (reset a 0 + animación de asegurado).
- Al morir con resultado Retry→lobby: limpiar visualmente (se perdieron). Al revivir/DBNO-recuperado: se conservan (escucha la señal de resultado de muerte, coherente con `CardRunService`).
- **Optimización:** event-driven, sin polling. Reusa `BoonCardStyler` para el color por rareza.

### 5.4 [MOD/NUEVO] Efectos de cartas equipadas (reuso de Boons)

- **[NUEVO] `CardEffectDefinitions`** (server-side, tiene callbacks como `BoonDefinitions`): mapea `cardId → { effectType, statKey, valueType, value, triggerEvent, OnApply/OnTrigger, ... }` con el **mismo schema que `BoonDefinitions`**. **TODAS las cartas dan powerup, incluidas las comunes** (decisión #7). Lo que cambia entre rarezas es la **magnitud** a favor del jugador.
  - **Escala por rareza (recomendado, escalable):** definir un `EffectArchetypes` reutilizable keyed por `effectType` (ej. `LIFESTEAL`, `DAMAGE`, `IRON_SKIN`) con una tabla de **valor por rareza**: `{ Common=0.03, Uncommon=0.08, Rare=0.20, Epic=0.40, Legendary=0.60 }`. Cada carta solo declara su `effectType`; el `value` se resuelve por el tier de la carta. Así el diseño crea una carta nueva sin re-escribir el efecto, y sube toda una rareza tocando UN número. Coincide con el ejemplo del dueño (lifesteal 3% Common → 20/40% en rarezas altas).
  - Los valores por rareza se leen de una tabla derivada de `CardRarityConfig.Order`, así **agregar/quitar una rareza** solo requiere extender la tabla (decisión #8).
- **[NUEVO] `CardLoadoutService`** (`ServerScriptService.Services.Cards.CardLoadoutService`):
  - Estado persistido: `data.equippedCards = { cardId1, cardId2, cardId3 }` (máx 3, en `SessionService`).
  - API: `equip(player, cardId, slot)`, `unequip(player, slot)`, `getEquipped(player)`. Valida que la carta esté **owned** (`CardService.getCollection`).
  - **Al iniciar run:** por cada carta equipada con efecto, `BoonManager.apply(player, derivedBoonId)`. Los boons son run-scoped y `BoonManager` ya los limpia al morir/volver al lobby → se re-aplican en el siguiente run. Combate ya lee los multiplicadores.
  - Remotes: `CARD_LOADOUT_SYNC` (estado de slots), request `CARD_EQUIP_REQUEST` (server-authoritative).
- **Sin escalado de poder por duplicados** (decisión #6 / PDF §10.1): los duplicados suman al **contador de unidades** del álbum (`data.cards[cardId]`, badge "xN") y dan **coins por rareza** al cashout — pero NO suben la magnitud del efecto. Para el powerup **solo importa poseer la carta** (count ≥ 1) y su **rareza**; tener 5 copias no la hace más fuerte. El "número + dots" del mockup es indicador visual de rareza/decoración, no un nivel de poder.
- **Optimización:** aplicar efectos solo en run-start (evento), no por-frame. `BoonManager.getStatMultiplier` ya es O(boons activos).

### 5.5 [NUEVO/MOD] Stat "Luck" — drop chance + sesgo de rareza

- **[MOD] Tienda de power-ups:** agregar Luck como upgrade comprable con oro (patrón `CoinUpgradesShop`). Persistir `data.luck` (o nivel de luck) en `SessionService`.
- **[NUEVO] `LuckConfig`** (`ReplicatedStorage.Shared.Config.LuckConfig`): curva de Luck → dos efectos, ambos data-driven y comentados para diseño:
  1. **Drop chance:** `finalDropChance = baseDropChance * (1 + luck * DROP_CHANCE_PER_LUCK)` (clamp a 1.0).
  2. **Sesgo de rareza:** en `weightedRandom`, el peso efectivo de cada carta se modula por su tier: `effWeight = baseWeight * (1 + luck * RARITY_BIAS[tier])`, donde `RARITY_BIAS` es 0 (o negativo) para Common y crece con la rareza. Resultado: más Luck → las cartas raras "pesan" más en el pool.
- **Tope de Luck y garantía de mejor rareza (decisión #9):**
  - `LUCK_MAX` = tope absoluto de la stat (ej. `50`). **En `luck == LUCK_MAX`, la probabilidad de la MEJOR rareza (última de `CardRarityConfig.Order`) es 100%.** Implementación: a medida que `luck → LUCK_MAX`, el sesgo colapsa el pool hacia el top tier; en el tope, los pesos de todos los tiers inferiores se anulan (solo cae la mejor rareza). La curva se define en `LuckConfig` de forma que la interpolación llegue exactamente a 100% en el máximo.
  - `SHOP_LUCK_MAX` = tope comprable con **oro** en la tienda (ej. `30`–`40`), **estrictamente menor que `LUCK_MAX`**. La tienda de oro **jamás** debe llevar al jugador al 100% garantizado.
  - **Reserva premium (V-FUTURA, fuera de este plan):** el tramo `SHOP_LUCK_MAX → LUCK_MAX` (ej. el x10 final) se reserva para venta en la **tienda premium** (Robux/gamepass). No implementar ahora; solo dejar el seam: la stat ya soporta el rango completo, solo la tienda de oro está capada. Documentado aquí para la versión futura.
- **[MOD] `CardDropService`:** al rollear, leer `luck` del jugador (clamp a `LUCK_MAX`) y pasar por `LuckConfig` antes del `weightedRandom`.
- **Optimización:** matemática pura en el momento del drop (que ya es un evento poco frecuente). Cero costo continuo.

### 5.6 [MOD] Álbum / menú de Cartas (mockup adjunto)

Expandir `CardCollectionController` + `CardCollectionGui` para igualar el mockup:

- **Sección EQUIPADAS (3/3):** 3 slots arriba. Click en carta de la colección = equipar en primer slot libre; click en slot equipado = desequipar. Server-authoritative vía `CardLoadoutService`.
- **Colección:** grid con badge de conteo/nivel + marco por rareza (ya existe base) + estado owned/locked (ya existe).
- **Filtros por rareza:** botón "Todas" + un botón por cada tier, **generados dinámicamente desde `CardRarityConfig.Order`** (decisión #8), no hardcodeados. Así V1 muestra los 5 tiers (Común, Poco Común, Rara, Épica, Legendaria) y agregar/quitar una rareza actualiza los filtros solo. Esto resuelve la discrepancia del mockup (mostraba 4). Búsqueda por nombre incluida.
- **Gamepad:** hints del mockup (A equipar, X ver carta, Y filtros, B atrás). Reusa `GamepadNav` (ya usado por el controller actual).
- **Acceso:** desde Hub y Lobby (el diseño lo pide en ambos). Botón HUD `CardsButton` ya existe.
- **Optimización:** celdas se construyen UNA vez y se reusan (ya lo hace); filtrar = toggle `Visible`, no recrear.

### 5.7 [NUEVO] Estantería del lobby (shelf)

`ServerScriptService.Services.Cards.CardShelfService` + controller/visual.

- Solo en el Lobby (V1). Modelo con **4 secciones** (1 por jugador de la instancia), **3 slots por sección** = las 3 cartas equipadas de ese jugador.
- Asignación de sección por jugador al entrar; slot vacío si no tiene carta equipada.
- Se actualiza **por evento** (cuando alguien equipa/desequipa o entra/sale), NO por polling. Usa `CardLoadoutService` como fuente de verdad + un remote broadcast `CARD_SHELF_SYNC`.
- **Optimización:** actualización event-driven, un solo modelo compartido. Las texturas de carta ya existen (`textureId`).

### 5.8 [MOD] Misiones de cartas

En `MissionTracker` (`ALL_QUESTS`):
- Nuevo `Type = "CardCollect"` (y opcionalmente `"CardCashout"`, `"CardRarity"`). Se incrementa enganchándose a `CardService.onCardCollected` (que dispara en el COMMIT del cashout, no en el pickup → cuenta solo cartas realmente aseguradas, correcto per diseño §4).
- Rewards ya soportan Gold/Boon/Title; el diseño menciona recompensas cosméticas → **fuera de V1** (excluidas explícitamente), pero el seam de rewards cosméticos se puede añadir después sin tocar cartas.
- **Optimización:** ya es event-driven e incremental con toasts.

### 5.11 [NUEVO] Coins por duplicado — `CardDupeRewardConfig`

`ReplicatedStorage.Shared.Config.CardDupeRewardConfig` (o campo `dupeCoins` dentro de `CardRarityConfig.Tiers`).

- Tabla `{ [rarity] = coinsPorDuplicado }`, derivada de `CardRarityConfig.Order` para respetar decisión #8 (rarezas dinámicas). Ej: `Common=100, Uncommon=150, Rare=200, Epic=400, Legendary=800`.
- **Fácilmente modificable por rareza** (requisito explícito). Un solo número por tier.
- Consumido por `CardRunService.flush` (decisión #6 / PDF §10.1): por cada **ejemplar duplicado** otorga `CardDupeRewardConfig[rarity]` coins vía `PlayerService.addCoins`. "Duplicado" = todo ejemplar de una carta que **ya poseías antes de este run**, o el 2º+ ejemplar del mismo cardId dentro del mismo run. El **primer** ejemplar de una carta nunca-antes-poseída no paga coins (es la carta nueva).
- **Escala por número de duplicados:** si en un mismo flush hay 3 ejemplares de una carta ya poseída, se pagan 3× los coins de esa rareza (y +3 al contador de unidades del álbum).

### 5.12 [MOD] Carta incógnita en el mundo (mystery visual)

Requisito PDF §3.3 / decisión #11: en el mundo, **todas** las cartas se ven iguales (incógnita); la identidad se revela al recoger.

- **[MOD] `CardDropService.buildCardPart` y `WorldCardService.buildCardPart`:** en vez de renderizar `def.textureId`, renderizar un **visual genérico de incógnita** (una textura "?" / carta misteriosa común a todas). El color/marco por rareza es **decisión de diseño**: o bien también se oculta la rareza (incógnita total), o se muestra el marco de rareza pero no la cara (pista de rareza). **Recomendado:** ocultar todo (incógnita total) para máxima sorpresa; el reveal al recoger paga el efecto. Marcar como TBD menor (§10.9).
- El `cardId` real ya viaja en el atributo `CardId` de la Part (server-authoritative); el cliente nunca necesita la textura real hasta el reveal, así que **no se filtra** qué carta es (bueno para la sorpresa y para no replicar texturas de más).
- **[MOD] Reveal:** al recoger (server confirma `addPending`), el cliente hace el **reveal** (§5.3) mostrando la carta real por primera vez.
- **Optimización:** una sola textura de incógnita compartida por TODOS los drops → menos assets únicos replicados en el mundo, más barato que hoy (que carga la textura real de cada drop).

### 5.9 Remotes nuevos a registrar (Shared.Types.Remotes)

```
CARD_PENDING_SYNC   = "CardPendingSync"     -- server→client, incremental, pendiente del run
CARD_CASHOUT_DONE   = "CardCashoutDone"     -- server→client, resumen + feedback
CARD_LOADOUT_SYNC   = "CardLoadoutSync"     -- server→client, estado de los 3 slots
CARD_EQUIP_REQUEST  = "CardEquipRequest"    -- client→server, request equip/unequip (authoritative)
CARD_SHELF_SYNC     = "CardShelfSync"       -- server→clients (lobby), estado de la estantería
```
(Ya existen: `CARD_COLLECTION_SYNC`, `CARD_COLLECTED`.)

### 5.10 Schema de datos (SessionService `data`)

```
data.cards         = { [cardId] = count }         -- YA EXISTE. Colección permanente.
data.equippedCards = { cardId1, cardId2, cardId3 } -- NUEVO. Máx 3. Persistido.
data.luck          = <number>                      -- NUEVO. Stat Luck (o nivel). Persistido.
-- PENDIENTE del run: NO va aquí. Vive en CardRunService en memoria. Efímero.
```

---

## 6. Diagrama de flujo (data flow)

```
                                 ┌─────────────────────────────────────────────┐
  CombatService.onEnemyKilled ──►│ CardDropService  (roll dropChance×Luck,      │
                                 │  pick por weight×Luck, spawn prefab per-owner)│
                                 └───────────────┬─────────────────────────────┘
  CollectionService WorldCard  ──► WorldCardService (shared, claim x run)        │
                                                  │ touch (server-side)          │
                                                  ▼                              ▼
                                        CardRunService.addPending(player, cardId)
                                                  │  CARD_PENDING_SYNC → HUD pendiente
                                                  ▼
                        ┌──────────── cashout (estación / escape) ──────────────┐
                        │  CardCashoutService.flush → CardRunService.flush       │
                        └───────────────────────┬───────────────────────────────┘
                                                ▼
                                   CardService.collect (×n)  ──► data.cards + persist
                                                │  CARD_COLLECTION_SYNC → álbum
                                                │  CardService.onCardCollected ──► MissionTracker
                                                ▼
                                       CARD_CASHOUT_DONE → feedback llamativo

  Álbum: equip 3 ──► CardLoadoutService (data.equippedCards) ──► run-start: BoonManager.apply
                                                            └──► CARD_SHELF_SYNC → estantería lobby
  Muerte sin revivir ──► CardRunService.clearPending (se pierden)  [respeta estado revivible]
```

---

## 7. Rendimiento — presupuesto y reglas (crítico: sin lag)

Reglas duras que la implementación debe cumplir. Havoc Hotel ya tuvo problemas de rendimiento y físicas; no repetir. Ver memorias `[[enemy-rigs-physics-no-humanoid]]`, `[[enemy-collision-groups]]`.

1. **Cero polling servidor para cartas.** Todo por Signal/evento (drop, pickup, cashout, equip, shelf). El único loop permitido es el Heartbeat **ya existente** de `WorldCardService` que anima TODAS las cartas de mundo en un solo pase (y el `RenderStepped` client-side de drops, también un solo pase). No agregar loops nuevos.
2. **Prefabs de carta baratos:** Anchored (sin física), `CanCollide=false`, `CastShadow=false`, `SmoothPlastic`. Ya es así. La animación mueve CFrame, no simula física. **No** dar Humanoid ni colliders pesados.
3. **Despawn al avanzar de cuarto / escapar (decisión #10):** las cartas-drop de enemigo de un jugador que NO fueron recogidas se **despawnean y liberan** en cuanto ese jugador pasa al siguiente cuarto o escapa — no esperan el timeout. `CardDropService` trackea los drops vivos por owner y los limpia enganchándose al cambio de cuarto (`RoomService`/`RoomManager`) y al escape. El cleanup por timeout (20s, ya existe) queda como red de seguridad. Esto acota memoria en runs largos y hace innecesario un cap duro. (Las cartas de mundo compartidas de `WorldCardService` son otra cosa: persisten dentro del run y re-arman por run, no aplican a esta regla.)
4. **Sync incremental siempre:** solo se envía la carta que cambió (`{[cardId]=count}`), nunca el dump completo salvo el push inicial al entrar. Ya es el patrón de `CardService`.
5. **PointLight con cuidado:** cada drop tiene un `PointLight` (Range 8). Muchas luces dinámicas = costo de render en móvil. **Recomendación:** limitar rango/brillo o usar solo en rarezas altas; evaluar en móvil. Tunear en config.
6. **UI se construye una vez, se togglea:** celdas del álbum persistentes; filtrar/paginar = `Visible`/`LayoutOrder`, no re-instanciar. Ya es el patrón.
7. **Estantería event-driven:** actualizar solo al cambiar equip o roster de la instancia. Nunca por frame.
8. **Cashout en batch:** `flush` debe agrupar el commit (un `markDirty` + un sync), no N escrituras. Añadir `CardService.collectMany(player, {[cardId]=n})` para evitar N remotes/N dirty.

---

## 8. Balanceo y escalabilidad — para el área de diseño

Dónde toca diseño para tunear/crear contenido, **sin leer lógica**:

| Quiero… | Edito SOLO… |
|---|---|
| Agregar una carta | Una entrada en `CardCatalog.Cards` (+ textura). |
| Cambiar qué enemigo suelta qué | `CardDropConfig.Tables[enemyConfigId]` (dropChance + pool de pesos). |
| Tunear rareza (color, orden) | `CardRarityConfig.Tiers` / `.Order`. |
| Agregar o quitar una rareza | Extender `CardRarityConfig.Order` + tablas derivadas (dupe coins, effect archetypes, luck bias). Todo se deriva de ahí. |
| Cambiar coins que da un duplicado | `CardDupeRewardConfig` (un número por rareza). |
| Cambiar cómo Luck afecta drop/rareza | `LuckConfig` (curvas + `LUCK_MAX` + `SHOP_LUCK_MAX`). |
| Dar/cambiar efecto de una carta | `CardEffectDefinitions[cardId]` (effectType) + `EffectArchetypes` (valor por rareza). |
| Colocar/mover estación de cashout | Tag `CardCashoutStation` en la Part que quieras (anchors mandan). |
| Cambiar duración del hold de cashout | Config del prompt en `CardCashoutService`. |
| Agregar misión de cartas | Una entrada en `ALL_QUESTS` con `Type="CardCollect"`. |
| Colocar carta escondida en el mundo | Tag `WorldCard` en una Part/Model + atributo `CardId` o `RandomRarity`. |

**Todos los números de balance en un solo lugar por dominio, comentados.** Nada hardcodeado en lógica.

---

## 9. Fases de implementación (milestones testeables)

Cada fase deja el juego jugable y se puede probar aislada. Todo detrás de `CardsConfig.ENABLED` hasta la fase final.

- **Fase 0 — Refactor de 2 fases + incógnita.** Crear `CardRunService`. Redirigir `CardDropService` y `WorldCardService` a `addPending`. `CardService.collect` = solo-commit. Visual **incógnita** en los prefabs (§5.12). Despawn de drops al avanzar de cuarto/escapar (decisión #10). Reglas de muerte: revive recupera / DBNO mantiene / Retry pierde (decisión #1). Test: matar enemigo → drop incógnito → recoger → pendiente (log); avanzar de cuarto → drop no recogido desaparece; revivir → conserva; Retry → pierde.
- **Fase 1 — Cashout + duplicados + feedback en estación.** `CardCashoutService` (anchor tagged + cubo placeholder + prompt corto) + cashout-al-escapar. `CardService.collectMany` batch. Duplicados → +1 unidad álbum + coins por rareza (`CardDupeRewardConfig`). `BillboardGui` **encima de la estación** con cartas depositadas (PDF §6). Test: carta nueva → estación → permanente; duplicado → +1 badge + coins; escapar → guarda; morir Retry → pierde.
- **Fase 2 — HUD pendiente + reveal.** `CardPendingHUD`: reveal momentáneo al recoger (imagen o rareza, TBD §10.9) + fichas de rareza pendientes junto a `BoonHUD`. Feedback de cashout en pantalla. Test visual del loop completo.
- **Fase 3 — Luck.** `LuckConfig` (con `LUCK_MAX` = 100% mejor rareza, `SHOP_LUCK_MAX` capado) + stat en tienda de power-ups + modulación en `CardDropService`. Test: subir Luck → más drops y más raros; verificar que la tienda de oro NO llega al 100%. (Tramo premium reservado = V-futura, no se implementa.)
- **Fase 4 — Equip + efectos.** `CardEffectDefinitions` + `EffectArchetypes` (valor por rareza) + `CardLoadoutService` + apply como boons en run-start + sección Equipado en el álbum. Test: equipar 3 (incl. una común), iniciar run, verificar efecto en combate y que la magnitud sube con rareza.
- **Fase 5 — Estantería lobby.** `CardShelfService` + visual. Test multi-jugador.
- **Fase 6 — Misiones de cartas.** Nuevo `Type` en `MissionTracker`. Test: cashout incrementa misión.
- **Fase 7 — Activación.** Flip `CardsConfig.ENABLED = true`. QA integral. Reemplazar texturas placeholder (`rbxassetid://0`) por arte final.

**Nota de arte/assets:** las texturas de carta son placeholder y hay que subirlas y autorizarlas a la experiencia. Ver memoria `[[audio-asset-permissions-gating]]` — assets no autorizados fallan silenciosamente; verificar autorización.

---

## 10. Preguntas abiertas — RESUELTAS (2026-07-07)

Todas las dudas originales fueron definidas por el dueño. Se conservan aquí como registro:

1. **Estación cada N cuartos — colocación:** ✅ **Anchors manuales.** Tag `CardCashoutStation`, guía default cada 5 cuartos, movible a mano por diseño en cualquier momento. Ver §5.2.
2. **Escalado por duplicados / niveles de carta:** ✅ **No hay niveles.** Los duplicados dan **coins por rareza** al cashout (Common 100, Rare 200, etc., editable por rareza). Ver §5.11 y §5.1.
3. **Filtro Uncommon en el álbum:** ✅ **Filtros generados desde `CardRarityConfig.Order`** (5 tiers en V1). Nada hardcodeado. Ver §5.6.
4. **Alcance del efecto por rareza:** ✅ **TODAS las cartas dan powerup**, incluidas comunes. La **magnitud escala por rareza** (ej. lifesteal 3% Common → 20/40% en rarezas altas) vía `EffectArchetypes`. Ver §5.4.
5. **Luck — techo y curva:** ✅ **`LUCK_MAX` con mejor rareza al 100%** en el tope. `SHOP_LUCK_MAX` (oro) capado por debajo; el tramo restante **reservado para tienda premium (V-futura)**. Curva/números finos y costo en oro por nivel se afinan en `LuckConfig` durante Fase 3. Ver §5.5.
6. **Estación de cashout — modelo/arte:** ✅ **Cubo placeholder** generado por código; el dueño coloca la máquina real después (basta re-taguear). Ver §5.2.
7. **Cap de cartas pendientes en el mundo:** ✅ **Resuelto por despawn al avanzar de cuarto/escapar** (decisión #10), no hace falta cap duro. Ver §7.3.
8. **Recompensas cosméticas de misiones:** ⏳ **PENDIENTE** — el dueño aún no tiene info. Fuera de V1. Dejar solo el seam de rewards en `MissionTracker` (ya soporta tipos extensibles) para no re-trabajar.

9. **Reveal e incógnita — detalles visuales (TBD del PDF):** ⏳ **PENDIENTE (menor).** (a) El reveal al recoger, ¿muestra la **imagen completa** de la carta o **solo la rareza**? (PDF dice TBD). (b) La carta incógnita en el mundo, ¿oculta también la rareza (incógnita total, recomendado) o muestra el marco de rareza como pista? No bloquea implementación; se define en Fase 2 al construir el reveal/HUD.

**Dudas cerradas por el PDF nuevo (2026-07-07):** reglas de muerte (revive recupera / DBNO mantiene / Retry pierde), duplicados (+1 unidad al álbum + coins por rareza), carta incógnita en el mundo, feedback encima de la estación, pendiente por rareza junto a los boons.

---

## 11. Trampas conocidas del codebase (aplican aquí)

De la memoria del proyecto — leer antes de implementar:

- **Fallo silencioso de refs de sala / contrato del sistema de salas:** `[[room-system-contract]]`. Si el cashout automático se engancha a salas, respetar el contrato o falla mudo.
- **Estado revivible en muerte:** `BoonManager` NO limpia si `DeathService.getPendingState == "Dying"`. `CardRunService.clearPending` debe replicar esta excepción, o se pierden pendientes de alguien que iba a revivir. Crítico para la decisión #1.
- **Colisiones/físicas de NPCs y grupos:** `[[enemy-collision-groups]]`, `[[enemy-rigs-physics-no-humanoid]]`. Los prefabs de carta deben quedar fuera de grupos de colisión problemáticos (ya Anchored + CanCollide=false).
- **VM separada del MCP:** `[[mcp-execute-luau-separate-vm]]` — `require()` de prueba ≠ instancia real del módulo; instrumentar con prints, no asumir estado compartido al testear.
- **Autosave / persistencia:** `[[autosave-periodic-session]]`, `[[launch-data-wipe-store-versioning]]` — `equippedCards`/`luck` entran al mismo `data` de `SessionService`; el autosave staggered ya los cubre. Ojo con el versionado de stores si hay wipe.
- **Race de spawn de personaje:** `[[character-spawn-race-blue-screen]]` — el push inicial de colección/loadout usa `task.delay(3, ...)` como `MissionTracker`/`CardService` ya hacen; mantener ese patrón para que el cliente tenga controllers conectados.

---

## 12. Anexo — mapa de archivos del feature

```
ReplicatedStorage.Shared.Config
  ├─ CardsConfig            [REUSO]  master flag
  ├─ CardCatalog            [MOD]    + seam de efecto/nivel
  ├─ CardRarityConfig       [REUSO]
  ├─ CardDropConfig         [MOD]    modulado por Luck
  ├─ CardEffectDefinitions  [NUEVO]  efecto por carta (effectType)
  ├─ EffectArchetypes       [NUEVO]  valor de efecto por rareza (escala magnitud)
  ├─ CardDupeRewardConfig   [NUEVO]  coins por duplicado, por rareza
  └─ LuckConfig             [NUEVO]  curvas drop + rareza + LUCK_MAX/SHOP_LUCK_MAX

ReplicatedStorage.Shared.Types
  └─ Remotes                [MOD]    + 5 remotes nuevos (§5.9)

ServerScriptService.Services.Cards
  ├─ CardService            [MOD]    collect = solo commit + collectMany batch
  ├─ CardDropService        [MOD]    → addPending, + Luck, visual incógnita, despawn al avanzar
  ├─ WorldCardService       [MOD]    → addPending, visual incógnita
  ├─ CardRunService         [NUEVO]  pendiente por run (efímero)
  ├─ CardCashoutService     [NUEVO]  estaciones + escape
  ├─ CardLoadoutService     [NUEVO]  equip 3 + apply boons run-start
  └─ CardShelfService       [NUEVO]  estantería lobby

ServerScriptService
  ├─ BoonManager            [REUSO]  motor de efectos
  ├─ MissionTracker         [MOD]    + Type CardCollect
  └─ RoomSystem.RoomManager [REUSO]  hook de escape

StarterPlayer.StarterPlayerScripts.Controllers
  ├─ CardCollectionController [MOD]   + Equipado + filtros + búsqueda
  ├─ CardVisibilityController [REUSO]
  ├─ CardPendingHUD           [NUEVO] reveal al recoger + pendiente por rareza junto a BoonHUD
  ├─ BoonHUD                  [MOD]   área compartida para fichas de rareza de cartas pendientes
  ├─ CardShelfController      [NUEVO] visual estantería
  └─ BoonCardStyler           [REUSO] estilo por rareza

Tienda de power-ups (ShopService / shop concreto)  [MOD]  + Luck

SessionService data: + equippedCards, + luck   [MOD schema]

Assets: reemplazar texturas placeholder rbxassetid://0 + textura "incógnita" del mundo  [PENDIENTE arte]
```

---

*Fin del plan. Actualizar este documento conforme se cierren las preguntas abiertas de §10 y se avance por las fases de §9.*
