# Plan: Sistema de Packs de Bullet VFX (escalable por IDs)

> Place objetivo: **Lugar de juego** (game place). Hotel Havoc (lobby) no requiere render.
> Estado: **TODO HECHO — Fases 1-5** (2026-06-04). **Falta guardar el place en Studio (Ctrl+S)**.
> - F1: Config `BulletVFXPacks` + 11 packs horneados en `Blaster.Objects.BulletVFX`.
> - F2: `Effects.particlePackEffect` + `BulletVFXRegistry.getSlot` + `drawRayResults` (Bullet viajero / Impact) + `ShotReplication` (Muzzle reemplaza flash arma). Fallback legacy intacto.
> - F3 Dead VFX: hook en `Weapons.Shared.DamageCalculator.applyDamage` (choke point) → si `killed` y attacker=Player y target=NPC, `FireAllClients` por RemoteEvent nuevo `Blaster.Remotes.EnemyDiedVFX(styleId, cframe)`. Cliente en `ShotReplication` reproduce slot `Dead`.
> - F4 Tienda: `BulletVFXCategory` reescrito — 11 products con `id`==nombre de pack (lo valida `Registry.exists`). Fix bug `onSelect` (`:`→`.`).
> - F5 Debug: `ServerScriptService.PremiumStore.DebugGrantBulletVFX` (Studio-only, gate `IsStudio`) concede los 11 packs al entrar vía `SessionService.onPlayerAdded` → owned en Locker.
>
> Pendiente del owner: 2 IDs malos (Bubble-Bullet, Pixel-Muzzle), precios/nombres reales de tienda, DevProducts Robux (developerProductId=0 ahora). Borrar/desactivar DebugGrant antes de publicar.
>
> IDs inaccesibles ("User is not authorized to access Asset") — revisar typo/privacidad/ownership:
> - **Bubble / Bullet** = 116948755598996
> - **Pixel / Muzzle** = 77039140000534
> Slots sin ID (fallback a Default): General-Dead, Fire-Bullet, Dark(Bullet/Muzzle/Impact), Ligth-Dead,
> + los 2 de arriba hasta que se arreglen.

## Objetivo
Vender packs de VFX de balas en la Premium Store y dejar que el jugador active el que
tenga comprado desde el Locker. Cada pack = 1 tema con 4 slots de VFX:
**Bullet** (proyectil/tracer), **Muzzle** (flash de cañón), **Impact** (impacto),
**Dead** (muerte de enemigo). Debe ser escalable: agregar un pack = agregar una fila de IDs.

## Hallazgos (research con MCP Roblox)

### Sistema base que YA existe (`ReplicatedStorage.Blaster`)
- `Effects.BulletVFXRegistry` — resuelve estilo por atributo `BulletVFXStyle` del Player, fallback `Default`.
- `ServerScriptService.Services.BulletVFXService.setPlayerStyle(player, id)` — único punto de integración.
- Render: `Utility.drawRayResults` → `laserBeamEffect` (Beam INSTANTÁNEO) + `impactEffect`.
- Estilo `Default` (Model) = 3 partes: `Beam`, `CharacterImpact`, `EnvironmentImpact` + attrs (VisualSpeed=200, ImpactLifetime=0.5, EmitSpark=10, EmitCircle=2).
- Ya conectado a Locker/Store: `ReplicatedStorage.PremiumStore.Config.Categories.BulletVFXCategory.onSelect`.

### Estructura real de tus assets (inspeccionados vía LoadAsset)
Cada ID = `Model → Part` con ParticleEmitters:
- **Bullet**: Part con emitters + **Trail** (Beam66) → diseñado para VIAJAR.
- **Muzzle**: Part con emitters de flash.
- **Impact**: Part con emitters de burst.
- **Dead**: mismo patrón (burst en muerte).

Es más completo que el sistema base (que solo cubre beam + impact). Hay que ampliar a 4 slots
y añadir 2 hooks nuevos: Muzzle per-estilo y Dead.

### Bugs encontrados (a corregir en este plan)
1. `BulletVFXCategory.onSelect` llama `BulletVFXService:setPlayerStyle` con `:` pero está
   definido con `.` → desplaza args (player→self). Equipar romperá.
2. `setPlayerStyle` valida `Registry.exists(id)` → el `productId` de la tienda DEBE coincidir
   con el nombre del Model de estilo. Hay que alinear nombres.
3. `BulletVFXCategory.products` hoy tiene placeholders (`bvfx_neon_red`...) — reemplazar por packs reales.

## Decisiones tomadas
- **Carga: pre-hornear en el place.** Importo cada ID una vez a `BulletVFX.<Pack>` como Model.
  Sin LoadAsset en runtime = más rápido, replica normal, offline-safe. Re-importo al agregar IDs.
- **Bullet: proyectil que viaja.** Clono el Bullet en el cañón y lo muevo al impacto con su Trail.
  El daño sigue siendo hitscan instantáneo (el VFX es puramente cosmético).

## Arquitectura propuesta

### 1. Config de IDs — `ReplicatedStorage.Blaster.Config.BulletVFXPacks` (ModuleScript)
Single source of truth. El usuario edita esto para agregar packs/IDs:
```lua
return {
  General = { Bullet=83831631218794, Muzzle=116718076789053, Impact=133501434355806, Dead=0 },
  Fire    = { Bullet=0,              Muzzle=80329351489751,  Impact=112223301171670, Dead=89127424440086 },
  Lighting= { Bullet=97280073154341, Muzzle=83200827395762,  Impact=96319885447806,  Dead=81762665029014 },
  -- ...resto de packs...
}
```
`0`/`nil` en un slot → fallback al slot del `Default`. Agregar pack = agregar fila.

### 2. Importer build-time (script vía MCP, NO corre en runtime)
Lee `BulletVFXPacks`, `LoadAsset` cada id, extrae el Part interno, ensambla Models de estilo
bajo `ReplicatedStorage.Blaster.Objects.BulletVFX.<Pack>` con hijos `Bullet/Muzzle/Impact/Dead`
(+ copia `Beam/CharacterImpact/EnvironmentImpact` del Default para fallback). Re-ejecutable.
Lo corro yo vía MCP cuando me pases IDs.

### 3. Extender `BulletVFXRegistry`
- Añadir `getSlotForPlayer(player, slotName)` → devuelve el Part del slot, con fallback per-slot a Default.
- Mantener API actual intacta.

### 4. Render — nuevo `ReplicatedStorage.Blaster.Effects.particlePackEffect` + editar pipeline
- `playMuzzle(part, muzzleCFrame)` — clona, Emit, Debris.
- `playBullet(part, fromPos, toPos, speed)` — clona, tween con Trail, destruye al llegar.
- `playImpact(part, pos, normal)` — clona orientado a normal, Emit, Debris.
- `playDead(part, cframe)` — clona en la muerte, Emit, Debris.

Editar:
- `Scripts.ShotReplication` — Muzzle del pack (reemplaza el FlashEmitter del arma).
- `Utility.drawRayResults` — Bullet viajero + Impact del pack (fallback a beam/impact si el pack no trae slot).

### 5. Dead VFX hook (nuevo server→client)
- Killer: leer creator tag del Humanoid en la muerte (patrón estándar). **Verificar** que el
  daño del Blaster taggea al humanoide; si no, agregar el tag.
- Nuevo RemoteEvent `EnemyDiedVFX` (server→clients): `{ styleId, cframe }`.
- Fire desde `ServerScriptService.Services.EnemyAI:_onDied` (o `RoomSystem.RoomInstance:_onNPCDied`).
- Cliente escucha y llama `playDead` con el estilo del killer.

### 6. Fix integración tienda
- `BulletVFXService.setPlayerStyle`: pasar a método `:` o corregir la llamada en `BulletVFXCategory` a `.`.
- Alinear `productId` de products con nombres de pack (`General`, `Fire`, ...).
- Poblar `products` con los packs reales (necesito displayName + precio Robux + thumbnails).

### 7. Scope de places
- Render e importer: solo **Lugar de juego**.
- Selección persiste por atributo; Locker/Store ya viven en ambos places.

## Fases de implementación
1. `BulletVFXPacks` + importer + importar General/Fire/Lighting de prueba.
2. Registry slots + `particlePackEffect` + Muzzle/Bullet/Impact en pipeline. Probar 1 pack en Studio.
3. Dead VFX hook (verificar killer attribution).
4. Fix bugs tienda + poblar products + probar flujo compra→equipar→render.
5. Importar packs restantes con la lista completa de IDs.

## Lo que necesito del usuario
- **Lista completa de IDs** (4 slots × packs). En la imagen faltan varios (Bullet de Fire/Dark,
  Impact de Dark, etc.). Dame los que tengas; slots vacíos = fallback automático.
- **Precio Robux + nombre display** por pack para la tienda.
- Confirmar: ¿el Muzzle del pack **reemplaza** el flash actual del arma o se **suma**? (recomiendo reemplaza).

## Testing manual (Studio / Lugar de juego)
- Tras Fase 2: equipar estilo vía atributo y disparar → ver Muzzle + Bullet viajero + Impact del pack.
- Tras Fase 3: matar enemigo → ver Dead VFX del estilo equipado en el cadáver.
- Tras Fase 4: comprar en Premium Store → activar en Locker → confirmar render del pack correcto.
- Multiplayer: con 2 clientes, cada quien ve el estilo del SHOOTER (no el propio).
