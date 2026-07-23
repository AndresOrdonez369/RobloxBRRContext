# Sistema de Cartas Coleccionables — Handoff

> Implementado en **Pisso Workshop**. Prototipo funcional y verificado en playtest. Efectos pasivos **solo visuales** (no tocan gameplay real todavía). Listo para portar a producción (Hotel Havoc).

---

## 1. Qué es

Sistema de cartas coleccionables con **loop de riesgo/recompensa**:

1. Los enemigos sueltan cartas en el mundo → van a un **bolsillo temporal**.
2. El jugador debe llegar a una **zona Vault** para **asegurarlas** al **álbum permanente**.
3. Si **muere** antes del Vault, pierde todo lo no asegurado.
4. En el álbum, **equipa** cartas en slots (los bonos hoy se muestran, no se aplican aún).

**Modelo multi-nivel:** cada carta cae en varios **niveles de rareza** (1=Common … 5=Mythic). El álbum guarda por carta la **cantidad** y tu **mejor nivel** obtenido; la rareza/color que ves = tu mejor nivel.

---

## 2. Dónde está todo (rutas exactas)

### Datos compartidos — `ReplicatedStorage.CardSystem/`
| Ruta | Tipo | Rol |
|---|---|---|
| `ReplicatedStorage.CardSystem.Config` | ModuleScript | Tunables (slots, drops, vault tag, muerte, debug, `ApplyRealEffects`) |
| `ReplicatedStorage.CardSystem.Theme` | ModuleScript | Tokens del DS (colores de rareza, superficies, FredokaOne) |
| `ReplicatedStorage.CardSystem.RarityConfig` | ModuleScript | Orden/rank de rarezas, pesos de drop, `FromLevel(n)` |
| `ReplicatedStorage.CardSystem.CardDatabase` | ModuleScript | Catálogo REAL (15 cartas) |
| `ReplicatedStorage.CardSystem.CardTypes` | ModuleScript | Tipos Luau |
| `ReplicatedStorage.CardSystem.CardSystem_RE` | RemoteEvent | Canal server↔cliente |
| `ReplicatedStorage.CardSystem.CardSystem_RF` | RemoteFunction | Snapshot inicial de estado |
| `ReplicatedStorage.CardSystem.OpenAlbum` | BindableEvent | API: abrir álbum |
| `ReplicatedStorage.CardSystem.OpenTempBag` | BindableEvent | API: abrir overlay del bolsillo |

### Servidor — `ServerScriptService/`
| Ruta | Rol |
|---|---|
| `ServerScriptService.CardService` | Autoritativo: estado por jugador, loop temporal→álbum, equip, muerte, `randomDrop`. Expone `_G.CardService`. |
| `ServerScriptService.CardWorldServer` | Recolección en mundo: drops, pickups con imán (un solo Heartbeat), Vault, showcase. |

### Cliente — `StarterGui/`
| Ruta | Rol |
|---|---|
| `StarterGui.CardAlbumScreen` | Pantalla del álbum. Chrome = **instancias editables** (Panel/Content/FilterBar/Grid/Detail/Footer/TitleImage/CloseBtn). `AlbumController` (LocalScript) referencia + llena. `CloseBtnTemplate`. |
| `StarterGui.CardTempHUD` | Icono de bolsillo (esquina) + contador + FX del juice. `TempHUDController`. |
| `StarterGui.CardTempScreen` | Overlay modal de cartas temporales. `TempScreenController` + `CloseBtnTemplate`. |
| `StarterPlayer.StarterPlayerScripts.Controllers.CardScreenGuard` | Fuerza los modales de cartas a cerrado al iniciar (evita bloquear el movimiento). |

### Mundo — `Workspace.CardSystemDemo/` (DEMO, quitar en prod)
| Ruta | Rol |
|---|---|
| `Workspace.CardSystemDemo.CardEnemyDemo` | Enemigo de prueba (suelta carta al prompt o al tocarlo). |
| `Workspace.CardSystemDemo.CardVaultZone` | Zona de guardado. **Tag: `CardVault`**. |
| `Workspace.CardSystemDemo.Showcase` / `.Pickups` | Se crean en runtime (showcase por rareza + pickups activos). |

### Tag CollectionService
- **`CardVault`** — cualquier `Part` con este tag es zona de guardado.

### Global
- **`_G.CardService`** — API de integración server (ver §3).

---

## 3. Funcionalidad esperada

**Loop:** enemigo suelta carta (a un nivel) → pickup con **imán** que persigue al jugador → **bolsillo temporal** (juice: zoom de la carta que vuela al bolsillo) → llegar al **Vault** la asegura al **álbum** (animación de guardado) → **morir** vacía el bolsillo (feedback de pérdida).

**Álbum:** grid de las 15 cartas por rareza, panel de detalle con arte hero + track de niveles + efecto + descripción, filtros (ALL/EQUIPPED/RARITY + búsqueda), barra de colección, slots equipables.

**Cómo se abre/activa (API mínima):**
- Álbum: `ReplicatedStorage.CardSystem.OpenAlbum:Fire()` (lo dispara `HUD_Lobby.StoreRail.StoreButton_Cards` y `HUD_Hub.HubController` Btn_Cards).
- Bolsillo: `ReplicatedStorage.CardSystem.OpenTempBag:Fire()` (lo dispara el icono del bolsillo).

**API de integración server — `_G.CardService`:**
```lua
_G.CardService.GiveTempCard(plr, cardId, level) -- al derrotar enemigo que suelta carta
_G.CardService.CommitVault(plr)                 -- al entrar a un cuarto Vault
_G.CardService.OnPlayerDead(plr)                -- hook de muerte real
_G.CardService.GetState(plr)                    -- lectura del estado
_G.CardService.RandomDrop()                     -- -> cardId, level (drop ponderado)
```

**Protocolo `CardSystem_RE`:**
- Cliente→Server: `"Equip"`/`"Unequip"` (cardId); `"DebugDrop"`/`"DebugVault"`/`"DebugDeath"`/`"DebugWorldDrop"`/`"DebugShowcase"` (solo con `Config.Debug.Enabled`).
- Server→Cliente: `"State"` (estado completo), `"TempGained"` `{id,isNew,level}`, `"Vaulted"` `{count,ids}`, `"TempLost"` `{count,ids}`, `"Toast"` `{kind,text}`.

**Estado por jugador (autoritativo):**
```
album/tempBag = { [cardId] = { c = cantidad, lv = mejor nivel } }
equipped = { cardId, ... };  slotsUnlocked = n
```

---

## 4. Wired vs stub/mock

| Elemento | Estado |
|---|---|
| Loop completo (drop→temporal→vault→álbum, pérdida al morir) | **Wired** (probado) |
| Estado server-autoritativo (album/tempBag/equipped, mejor nivel) | **Wired** |
| Pickups con imán + un solo Heartbeat + texturas + haz de rareza | **Wired** |
| Álbum (grid, detalle con arte/niveles/efecto, filtros, footer, equip) | **Wired** |
| Bolsillo (icono, contador con tensión, juice de obtención, banking, pérdida) | **Wired** |
| Overlay de cartas temporales (grid + badges NUEVA) | **Wired** |
| Waypoint al Vault al cargar cartas (reusa `WaypointTracker`) | **Wired** |
| Datos reales de 15 cartas + texturas | **Wired** (falta ID de textura de Trulimero) |
| Enemigo / Vault / Showcase de prueba (`CardSystemDemo`) | **Stub/demo** (reemplazar por combate/mapa reales) |
| Efectos pasivos (daño/monedas/luck) | **Stub visual** (`Config.ApplyRealEffects=false`; solo se muestran) |
| Chips coin/luck/damage del footer del álbum | **Placeholder** (`+0%`; el modelo de efectos es nombrado, repensar) |
| Persistencia (DataStore) | **No implementado** |
| Audio (SFX por rareza) | **No implementado** (equipo de audio) |

---

## 5. Pendientes de backend (server-autoritativo)

1. **Persistencia**: cargar/guardar `album` (y `equipped`) por jugador con el sistema de datos de Hotel Havoc (ProfileService/DataStore). Hoy el estado vive solo en sesión (`CardService.state`).
2. **Hooks reales** (reemplazar el demo):
   - Al **matar** un enemigo con drop → `_G.CardService.GiveTempCard(plr, cardId, level)` (usar `RandomDrop()` o la lógica de drop real).
   - Cuarto de guardado real → taggear su `Part` con `CardVault` (o llamar `CommitVault(plr)`).
   - Muerte real → ya se auto-conecta a `Humanoid.Died`; validar en producción.
3. **Efectos reales**: al poner `Config.ApplyRealEffects=true`, conectar los bonos por nivel de cada efecto (Detonation, Infinite Mags, etc.) a los sistemas de arma/combate. Definir los valores por nivel.
4. **Anti-cheat**: el server ya es autoritativo; validar que `Equip`/`Unequip` respeten posesión y slots (ya lo hace) y que ningún drop venga del cliente.

---

## 6. No-colisión + integración

**Nombres verificados sin choque** (escaneados en el place; todos namespaced bajo `CardSystem` o prefijo `Card*`):

| Nombre | Verificado |
|---|---|
| `CardSystem_RE`, `CardSystem_RF` | Solo card system |
| `OpenAlbum`, `OpenTempBag` (BindableEvents) | Solo card system + integraciones del lobby |
| Tag `CardVault` | Solo card system |
| `_G.CardService` | Solo card system |
| Módulos `Config/Theme/RarityConfig/CardDatabase/CardTypes` | Hijos de `CardSystem` (namespaced, sin colisión global) |
| ScreenGuis `CardAlbumScreen/CardTempHUD/CardTempScreen` | Prefijo `Card*`, únicos |
| Scripts `CardService/CardWorldServer/CardScreenGuard` | Prefijo `Card*`, únicos |
| `Workspace.CardSystemDemo` | Único |

**Reusa (aditivo, no duplica):** `WaypointTracker`, `ButtonEffects`, `ResponsiveManager` + `ScreenPolicy` (responsive por convención de nombres), y el `HUD_Lobby.StoreRail` (se le añadió `StoreButton_Cards`). No renombra ni borra nada existente.

**Pasos para integrar en producción:**
1. Copiar entre ventanas de Studio con **Ctrl+C/Ctrl+V nativo** (no relay por código): `ReplicatedStorage.CardSystem`, `ServerScriptService.CardService` + `CardWorldServer`, `StarterGui.CardAlbumScreen` + `CardTempHUD` + `CardTempScreen`, `StarterPlayer.StarterPlayerScripts.Controllers.CardScreenGuard`.
2. Cablear los hooks reales de §5.
3. Botón de apertura: replicar `StoreButton_Cards` (o su llamada a `OpenAlbum`) en el HUD que corresponda.
4. Responsive: el álbum usa la convención `ScreenPolicy` (nombres `TitleImage`/`*Container`/`Background`); ya es compatible con el `ResponsiveManager` de producción.

---

## 7. Asset IDs

**UI del álbum / bolsillo**
| Uso | Asset |
|---|---|
| Fondo del álbum (libro) | `rbxassetid://135331975348156` |
| Título del álbum (banner) | `rbxassetid://114376698013456` |
| Icono/arte del bolsillo + juice | `rbxassetid://83104017518458` |
| Icono moneda (footer) | `rbxassetid://81917086216085` |
| Icono suerte (footer) | `rbxassetid://110166557754419` |
| Icono daño (footer) | `rbxassetid://93320968714938` |
| Botón "Card Album" del riel | `rbxassetid://94250320241190` |

**Texturas de las 15 cartas** (en `CardDatabase`)
| Código | Carta | Efecto | Textura |
|---|---|---|---|
| T_Card_01 | Boneca Ambalabu | Shotgun Vampirism | `98844501501226` |
| T_Card_02 | Cappuccino Assassino | Rifle-focused Fire | `126728197559752` |
| T_Card_03 | Lirilì Larilà | Infinite Mags | `97745121916044` |
| T_Card_04 | Pi Pi Watermelon | Pistol Efficiency | `80005700104499` |
| T_Card_05 | Pot Hotspot | Last Reserves | `97611043274426` |
| T_Card_06 | Tralalero Tralala | SMG Rush | `130843052546096` |
| T_Card_07 | Trenostruzzo Turbo 3000 | Can't Stop | `113497161282547` |
| T_Card_08 | Trippi Troppi Troppa Trippa | SMG All In | `70483673457984` |
| T_Card_09 | Tung Tung Tung Sahur | Detonation | `86522131996883` |
| T_Card_10 | La Vacca Saturno Saturnita | Rifle Marksman | `135327874973625` |
| T_Card_11 | Karkerkar Kurkur | *(por definir)* | `125463819667956` |
| T_Card_12 | Trulimero Trulicina | *(por definir)* | **sin ID** |
| T_Card_13 | Piccione Macchina | *(por definir)* | `124342722017624` |
| T_Card_14 | Ballerina Cappuccina | Shotgun Precision | `103798945465386` |
| T_Card_15 | Avocadorilla | World's Smallest Gun | `129093303193890` |

---

## Checklist antes de publicar

- [ ] `Config.Debug.Enabled = false` (desactiva teclas de prueba y el showcase).
- [ ] Quitar/gate `Workspace.CardSystemDemo` (enemigo, Vault y showcase demo).
- [ ] Modales cerrados: `CardAlbumScreen.Enabled=false`, `CardTempScreen.Enabled=false` (el `CardScreenGuard` lo asegura, pero déjalos así en el editor).
- [ ] `CardTempHUD.Enabled` según decisión (HUD de combate: lo enciende el HUD de acción).
- [ ] Conectar persistencia (DataStore) del `album`.
- [ ] Conectar hooks reales (`GiveTempCard` on kill, `CommitVault` en cuarto real).
- [ ] Reemplazar el ID de textura faltante de **Trulimero (T_Card_12)**.
- [ ] Cuando se definan los efectos por nivel, activar `ApplyRealEffects` y actualizar el footer del álbum.
