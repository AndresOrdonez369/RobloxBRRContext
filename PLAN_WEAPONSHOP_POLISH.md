# PLAN: Polish WeaponShop + Sistema de Armas por Niveles

**Status:** Approved
**Date:** 2026-05-21
**Deciders:** Juanse (game lead), Claude (arquitectura)
**Relacionado:** `PLAN_WEAPONS_SYSTEM.md` (ADR-001), `TODO_NEXT_FEATURE.md`

---

## 1. Objetivo

Polish de la tienda de armas a una UI de dos paneles a pantalla completa, y
agregar al sistema de armas progresión por niveles.

### Mecánica

- La tienda muestra packs de armas. Cada pack es un botón.
- Primera compra de un pack = **Unlock**. De ahí en adelante se compran
  **niveles** (≈10 niveles por pack, `MaxLevel` configurable).
- Subir un pack 1 nivel sube **todas las armas del pack 1 nivel a la vez**
  (uniforme: mismo entero de nivel para todas).
- Cada arma define sus propios stats por nivel — diseño puede tunear cada
  arma individualmente.
- Unlock y Upgrade se pagan con **Coins o Robux** (botones separados).

---

## 2. Decisión de arquitectura: dónde viven los valores por nivel

**Stats por nivel = por arma. Costo/nivel/unlock = por pack.**

| Dato | Vive en | Quién tunea |
|---|---|---|
| Stats de cada arma por nivel (damage, rateOfFire, reloadTime, mag, rarity...) | `WeaponDef.levels` | Diseño, 1 archivo por arma |
| `UnlockCost`, `UpgradeCost` por nivel, `MaxLevel`, product ids Robux | `WeaponPacksConfig.Packs[id]` | Diseño, 1 entrada por pack |

Razón: el leveo es por pack, pero las armas dentro de un pack son distintas
entre sí. Tabla explícita por arma (10 filas/arma) es manejable y da control
total a diseño. El pack solo lleva el estado de leveo y la economía.

---

## 3. Modelo de datos

### WeaponDef — tabla `levels` (NUEVO, opcional)

```lua
-- ReplicatedStorage.Shared.Config.WeaponDefs.ak47
stats = { damage = 35, rateOfFire = 600, magazineSize = 10, reloadTime = 1.0, ... }, -- base / fallback

levels = {
    -- Cada fila override CUALQUIER stat. Lo no listado cae a `stats` base.
    [1]  = { damage = 35, magazineSize = 10, rarity = "Common" },
    [2]  = { damage = 41, magazineSize = 12, rarity = "Common" },
    -- ...
    [10] = { damage = 98, rateOfFire = 720, reloadTime = 0.8, magazineSize = 30, rarity = "Epic" },
},
```

- `levels` es opcional. Arma sin `levels` resuelve siempre a `stats` (legacy intacto).
- `WeaponCatalog` validación no se rompe: solo chequea `REQUIRED_BASE`.

### WeaponPacksConfig — campos por pack (NUEVO)

```lua
starter = {
    DisplayName = "All Rounder",
    Weapons     = { { Id = "ak47" }, { Id = "cobra_pistol" } },
    MaxLevel    = 10,
    UnlockCost  = 0,            -- coins
    UnlockProductId  = 0,       -- Developer Product Robux (unlock)
    UpgradeProductId = 0,       -- Developer Product Robux (upgrade; reusado en cada nivel)
    Levels = {
        [2]  = { UpgradeCost = 300 },   -- costo en coins para ir de L1 a L2
        [3]  = { UpgradeCost = 650 },
        -- ... hasta [MaxLevel]
    },
}
```

- Robux price NO va en config: lo fija el Developer Product en el dashboard;
  la UI lo lee con `MarketplaceService:GetProductInfo`.
- `DPS`/`MagSize` duplicados en las entradas `Weapons` se eliminan en una fase
  posterior (cuando se actualicen sus lectores). Fase 1 NO los toca.

---

## 4. Cambios de código por área

### WeaponCatalog
- `getStatsAtLevel(weaponId, level) -> stats, rarity` — único punto de
  resolución (merge de `levels[L]` sobre `stats` base).
- `getMaxLevel(weaponId)` opcional.

### WeaponFactory
- `build(weaponId, level)` — param `level` opcional (default 1, legacy intacto).
  `SETTERS.RaycastFirearm` lee stats resueltos en vez de `def.stats` directo.
- `giveToPlayer` resuelve el nivel server-side vía `WeaponShop:GetPackLevel`.
  Cliente nunca manda nivel.

### WeaponShop
- Estado por pack: `{ owned = bool, level = int }` + `selected`.
- `HandlePurchase` acciones: `unlock`, `upgrade`, `activate`, `deactivate`.
  Todas validadas server-side, con el debounce de CoinUpgrades/PowerUps.
- `GetShopState` nuevo: lista de packs + `Details` (stats `current`/`next` por
  arma, calculados server-side).
- `GrantFromRobux(player, packId, action)` — patrón de PowerUpsShop.

### Persistencia (SessionService)
Nueva key en `DEFAULT_DATA` (los packs no son un map plano de ints):

```lua
weaponPacks = {
    owned    = { starter = { owned = true, level = 1 } },
    selected = "starter",
}
```

- Write-through: el cache de WeaponShop ES `sessionData.weaponPacks`.
- `mergeWithDefaults` con merge anidado sobre `weaponPacks.owned` + backfill
  defensivo (`{owned=true}` sin `level` -> `level=1`).

### Robux (PurchaseService)
- `buildProductLookup()` también escanea `WeaponPacksConfig`:
  `lookup[pid] = { shopId="WeaponShop", packId=id, action="unlock"|"upgrade" }`.
- Cada pack = 2 Developer Products (unlock + upgrade).

### UI (ShopGui / ShopController)
- `Backdrop` a pantalla completa (`IgnoreGuiInset`, size `{1,0},{1,0}`).
  Se quita el `UIAspectRatioConstraint` fijo del `Panel`; `UISizeConstraint`
  min `320x240` / max `1600x900`.
- `Panel` con 2 sub-frames alternados por `ShopType`:
  - `GridView` — grilla actual (CoinUpgrades, PowerUps), sin cambios.
  - `WeaponView` — dos paneles nuevos.
- `WeaponView`: izquierda lista de packs (botón toggle, **color de relleno** =
  seleccionado, botones Coins azul + Robux verde por pack); derecha "Pack
  Details" con fila por arma (`icono actual -> siguiente`, `DPS X > Y`,
  `Mag X > Y`, rareza como pill de color). Abajo-derecha `Activate`/`Deactivate`.
- Mobile portrait angosto (`AbsoluteSize.X < 600`): paneles apilados vertical.
- Plantillas nuevas: `PackRow`, `WeaponCompareRow`, `RarityTag`.

---

## 5. Comportamiento mid-run

Subir un pack a mitad de run **no re-estatea las armas ya equipadas**. Las
armas que se otorguen después del upgrade salen con el nivel nuevo (vía
`RoomRewardsService` -> `WeaponFactory.giveToPlayer`, que resuelve el nivel
actual al hacer grant).

---

## 6. Fases (incremental, el juego corre entre cada una)

| Fase | Alcance | Riesgo |
|---|---|---|
| **1** | `WeaponDefs`: agregar tabla `levels` a las 6 armas. `WeaponPacksConfig`: agregar `MaxLevel`, `UnlockCost`, `Levels`, product ids. **Puramente aditivo.** | Cero — nadie lee los campos nuevos aún |
| **2** | `WeaponCatalog.getStatsAtLevel` + `WeaponFactory.build(weaponId, level)` | Bajo — default legacy intacto |
| **3** | `SessionService`: `weaponPacks` en `DEFAULT_DATA` + `mergeWithDefaults` (ambos places) | Bajo — sin lector aún |
| **4** | `WeaponShop`: estado nuevo, acciones `unlock/upgrade/activate/deactivate`, write-through, `GetShopState` nuevo | Medio |
| **5** | Robux: `PurchaseService` + `GrantFromRobux` + registrar Developer Products | Medio |
| **6** | UI: `WeaponView` + plantillas; `ShopController` ramifica por `ShopType`. Eliminar `DPS/MagSize` duplicados | Medio |

CoinUpgrades y PowerUps no se rompen: son clases separadas; comparten solo la
grilla, que no se toca.

---

## 7. Notas

- **Doble place**: `SessionService` y los módulos de
  `ReplicatedStorage.Shared.Config` se cambian **idénticos en hub y game**.
  Cualquier drift desincroniza la propiedad de packs en el teleport.
- **Pruebas manuales tras cada fase** (Studio): armas legacy disparan bien;
  `unlock`/`upgrade` cobran coins correcto; Robux unlock vs upgrade rutean a la
  acción correcta vía `ProcessReceipt`; nivel del pack sobrevive teleport
  hub<->game; upgrade mid-run no re-estatea el arma equipada pero sí afecta el
  próximo reward.
