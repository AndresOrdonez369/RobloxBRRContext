# TODO — Próxima Feature después del Split Places

## Rehacer tienda de armas (WeaponShop)

**Estado actual (post split-places + tiendas-en-ambos-places):**

`WeaponShop` está completo y funcional en AMBOS places:

- **Hub place**: WeaponShop completo (UI ShopGui, ShopController, ShopParts, ShopService).
- **Game place**: WeaponShop completo también — se restauró todo el shop layer en Fase 7. `ShopService.init()` registra los 3 shops (CoinUpgrades, PowerUps, WeaponShop) en ambos places. RoomRewardsService lo consume vía `ShopRegistry.Get("WeaponShop")`.

**Pendiente real de WeaponShop:** persistencia. CoinUpgrades y PowerUps persisten vía write-through a `sessionData.shopLevels`. WeaponShop NO tiene persistencia implementada todavía — esa es la tarea de esta feature.

## Tarea concreta (cuando arranque esta feature)

1. Diseñar el modelo de persistencia de WeaponShop: ¿qué se guarda? ¿armas compradas? ¿packs desbloqueados? ¿loadout?
2. Implementar persistencia siguiendo el patrón write-through de CoinUpgradesShop/PowerUpsShop:
   - `_initPlayer` aliasa el estado a una key dentro de `sessionData` (e.g. `sessionData.weaponShop` o dentro de `shopLevels`).
   - Compras escriben directo a sessionData + `markDirty`.
   - `_savePlayer` solo `markDirty`.
3. Replicar el cambio en AMBOS places (hub + game) — son copias idénticas, deben quedar sincronizadas.
4. Si se agrega una key nueva a `sessionData`, actualizar `SessionService.DEFAULT_DATA` + `mergeWithDefaults` en ambos places.
5. Validar sync cross-place igual que CoinUpgrades/PowerUps: comprar en un place → teleport → reflejado en el otro.

## Referencias que tocar

- `ServerScriptService.Services.Shops.WeaponShop` (en ambos places — copias idénticas).
- `ServerScriptService.Services.SessionService` `DEFAULT_DATA` / `mergeWithDefaults` (ambos places) si se agrega key nueva.
- `Services.RoomRewardsService` consume `ShopRegistry.Get("WeaponShop")` para weapon rewards de room — no romper.

## Patrón de referencia

CoinUpgradesShop y PowerUpsShop ya tienen el write-through aplicado (Fase 7). Copiar su patrón `_initPlayer` / `_savePlayer` exactamente.
