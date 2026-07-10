# Brainrot Room Rush — Contexto del Juego

> **Archivo de contexto maestro.** Léelo al inicio de cada sesión para entender qué es el juego,
> cómo está construido y qué está pendiente. Los detalles técnicos profundos viven en los
> documentos enlazados al final.

---

## Identidad

- **Nombre del juego:** Brainrot Room Rush
- **Repo/carpeta de trabajo:** `Roblox_HavocHotel`
- **Plataforma:** Roblox Studio (Luau)
- **Arquitectura de places (split places):**
  - **Hub place** — "Hotel Havoc": lobby, tiendas, locker, estantería de cartas.
  - **Game place** — "Patio de recreo de terror de Brainrot": el gameplay (cuartos, combate, NPCs).
  - Las tiendas existen en AMBOS places como copias idénticas; deben mantenerse sincronizadas.
- **Nota histórica:** `PROJECT.md` y `RULES.md` aún usan el nombre de trabajo "RPS Chaos" (prototipo
  original de Rock-Paper-Scissors). El juego evolucionó a un shooter/brawler de cuartos con NPCs
  brainrot. La documentación técnica de esos archivos sigue vigente donde aplica (room system,
  weapons, convenciones); la parte de RPS 1v1 es legacy.

---

## Core Loop

1. Jugador entra al **hub** (Hotel Havoc): compra/mejora en tiendas, equipa armas y cosméticos.
2. Teleport al **game place**: avanza por una secuencia lineal de cuartos (`Room0` → `Room4`).
3. Al entrar a un cuarto se **spawnean NPCs** y las puertas se cierran (adelante y atrás según config).
4. Matar todos los NPCs del cuarto → puerta se abre, **reward** (coins/armas), siguiente cuarto.
5. Monedas se gastan en tiendas (CoinUpgrades, PowerUps, WeaponShop) que escalan el poder del jugador.
6. Capa futura (en HOLD): NPCs sueltan **cartas coleccionables** que se vuelven permanentes al cashout.

---

## Principios No Negociables

- **Server-authoritative:** toda lógica crítica (daño, drops, compras, progresión) se valida en servidor.
- **Data-driven:** agregar contenido (arma, pack, carta, VFX) = editar UNA tabla de config, cero código nuevo.
- **Event-driven, sin polling:** signals y Touched events; nada de loops per-frame en servidor.
- **Visuales en el editor, no por código:** UI estática, luces, modelos y prompts se crean en Studio;
  el código solo los referencia por nombre/tag. `Instance.new` solo para contenido dinámico runtime.
- **Cambios de código Roblox se implementan vía el subagente `roblox-systems-scripter`** sobre el place activo.
- **UI responsive scale-only:** `UDim2` solo con scale, offset = 0 (ver RULES.md para excepciones).

---

## Sistemas Principales

### 1. Room System v2 (DoorActor) — ✅ implementado
Sistema event-driven de cuartos con puertas. Módulos en `ServerScriptService.RoomSystem/`:
`CollisionGroupsBootstrap`, `DoorActor`, `RoomScanner`, `RoomInstance`, `RoomManager`.
- Cada `Room` en `Workspace.Level` tiene un `Door` Model (FrontVolume, BackVolume, BackVisual,
  RaycastBlocker, BlockerAnchor, NPCBlocker) + Attributes configurables desde Studio
  (`RoomType`, `MaxAttackers`, `HealthMult`, `BlocksBackward`, etc.).
- Blockers físicos **per-player** vía CollisionGroups (`PlayerBlocker_<UserId>` ↔ `Crosser_<UserId>`).
- Asume avance lineal en +Z. Layouts que doblan/ramifican: NO implementado, roadmap en PROJECT.md.
- Detalle completo: `PROJECT.md` § "Sistema de Cuartos v2".

### 2. Weapons System — Phase 1 hecho, Phase 2 pendiente
- **Catálogo data-driven:** `ReplicatedStorage.Shared.Config.WeaponCatalog` + `WeaponDefs/<weaponId>.lua`.
- **Engine Blaster** (FPS raycast) reutilizado como engine base; Tool prefabs en
  `ServerStorage.WeaponPrefabs.<id>` (hoy solo `ak47`).
- **Daño centralizado:** `ReplicatedStorage.Weapons.Shared.DamageCalculator.applyDamage(humanoid, baseDamage, attacker)`
  — único punto que aplica multiplicadores de PowerUps. NUNCA duplicar el patrón viejo `getDamageMultiplier`.
- **Packs con niveles** (PLAN_WEAPONSHOP_POLISH): unlock + ~10 niveles por pack; subir nivel sube
  todas las armas del pack. Stats por nivel viven en `WeaponDef.levels`; economía en `WeaponPacksConfig`.
- **Pendiente:** subir 4-5 modelos reales (pistol, rifle, smg, shotgun, sniper).
- Detalle: `PLAN_WEAPONS_SYSTEM.md` (ADR-001), `PLAN_WEAPONSHOP_POLISH.md`, RULES.md § "Sistema de Armas".

### 3. Tiendas (Shops) — funcionales, WeaponShop sin persistencia
- Registro central: `ServerScriptService.Services.Shops.ShopRegistry` (`ShopRegistry.Get("PowerUps")`, etc.).
- `ShopService.init()` registra 3 shops en ambos places: **CoinUpgrades**, **PowerUps**, **WeaponShop**.
- CoinUpgrades y PowerUps persisten con **write-through** a `sessionData.shopLevels`.
- **WeaponShop NO persiste todavía** — es la próxima feature (ver `TODO_NEXT_FEATURE.md`):
  copiar el patrón `_initPlayer`/`_savePlayer` de las otras dos, en AMBOS places, actualizando
  `SessionService.DEFAULT_DATA` + `mergeWithDefaults` si se agrega key nueva a sessionData.

### 4. Persistencia — SessionService
- `SessionService` es el dueño de los datos: write-through + `markDirty` + autosave periódico staggered.
- Sync cross-place: comprar en un place → teleport → reflejado en el otro.
- `RoomRewardsService` consume `ShopRegistry.Get("WeaponShop")` para rewards de cuarto — no romper.

### 5. Bullet VFX Packs — ✅ hecho (Fases 1-5, 2026-06-04)
- 11 packs de VFX de balas (slots: Bullet, Muzzle, Impact, Dead) vendibles en Premium Store,
  activables desde Locker. Config `BulletVFXPacks`, registry `BulletVFXRegistry`.
- **Pendiente del owner:** 2 asset IDs rotos (Bubble-Bullet 116948755598996, Pixel-Muzzle 77039140000534),
  precios/DevProducts reales (hoy `developerProductId=0`), borrar `DebugGrantBulletVFX` antes de publicar.
- Detalle: `PLAN_BULLETVFX_PACKS.md`.

### 6. Cartas Coleccionables — ⏸️ HOLD (`CardsConfig.ENABLED = false`)
- Capa roguelike: kills sueltan cartas → pendientes durante el run → permanentes al **cashout**.
- Álbum, equipar 3 como pseudo-powerups, estantería en lobby. Diseño original: Camilo Medina.
- Parcialmente implementado (CardService, CardDropService, WorldCardService, controllers). No borrar nada.
- Detalle y plan de reactivación: `PLAN_Sistema_Cartas_Coleccionables.md`.

---

## Estado Actual y Pendientes (2026-07-10)

| Área | Estado |
|---|---|
| Room System v2 | ✅ Funcional (lineal) |
| Weapons Phase 0-1 (catálogo + factory + ak47) | ✅ Hecho |
| Weapons Phase 2 (modelos reales) | ⬜ Pendiente |
| Shops en ambos places | ✅ Funcional |
| Persistencia WeaponShop | ⬜ **Próxima feature** (`TODO_NEXT_FEATURE.md`) |
| Bullet VFX packs | ✅ Hecho (pendientes IDs/precios del owner) |
| Cartas coleccionables | ⏸️ HOLD tras flag |
| Coin rewards con tiers | ✅ Hecho (`PLAN_COIN_REWARDS_REWORK.md`) |

---

## Documentos del Repo

| Archivo | Qué contiene |
|---|---|
| [PROJECT.md](PROJECT.md) | Arquitectura general, room system v2, decisiones, historial (nombre legacy "RPS Chaos") |
| [RULES.md](RULES.md) | Convenciones Luau, estructura de carpetas, UI scale-only, patrones ✅/❌, DataStore |
| [TODO_NEXT_FEATURE.md](TODO_NEXT_FEATURE.md) | Próxima feature: persistencia WeaponShop |
| [PLAN_WEAPONS_SYSTEM.md](PLAN_WEAPONS_SYSTEM.md) | ADR-001: arquitectura modular de armas |
| [PLAN_WEAPONSHOP_POLISH.md](PLAN_WEAPONSHOP_POLISH.md) | UI dos paneles + niveles por pack |
| [PLAN_BULLETVFX_PACKS.md](PLAN_BULLETVFX_PACKS.md) | Packs de VFX de balas (hecho) |
| [PLAN_COIN_REWARDS_REWORK.md](PLAN_COIN_REWARDS_REWORK.md) | Tiers escalables de rewards (hecho) |
| [PLAN_Sistema_Cartas_Coleccionables.md](PLAN_Sistema_Cartas_Coleccionables.md) | Feature de cartas (HOLD) |

## Mantenimiento de este archivo

- Actualiza la tabla de estado y la fecha cuando una feature cambie de estado.
- Cambios de arquitectura → `PROJECT.md`. Convenciones de código → `RULES.md`. Este archivo solo
  resume y enlaza; no dupliques detalle técnico aquí.
