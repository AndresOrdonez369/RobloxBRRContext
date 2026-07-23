# HUD Hub (Lobby) — Handoff

> Generado con el flujo `/feature-handoff`. Feature construido en el place dev activo **Pisso Workshop** (el DataModel reporta `game.Name = "IslandUIv2"` — mismo place, renombrado/guardado; tenerlo en cuenta al identificarlo). ENG lo traslada a producción (Hotel Havoc).

## 1. Qué es
Overhaul del **HUD de lobby/hub**: menú de acciones rápidas mobile-first con identidad de marca Hotel Havoc (placas cuadradas biseladas + remaches, paleta morado/verde/cian/dorado). Incluye menú de 8 acciones, 3 features de esquina, buffs premium color-codeados, CTA premium con carrusel, botón JUGAR, chip de monedas y barra de nivel, más 3 pantallas modales propias (Settings, Leaderboard, Community). NO incluye el HUD de combate in-run (`HUD004`).

## 2. Dónde está todo
**Raíz:** `StarterGui.HUD_Hub` (ScreenGui always-on, `Enabled=true`, sin `Overlay` → no modal).
- `HUD_Hub.LeftCluster` (UIGridLayout 2×4): `Btn_ShopPowerup/ShopWeapon/ShopCoin/ShopQuest/Daily/Cards/Invite/Locker`
- `HUD_Hub.CornerCluster`: `Btn_Settings/Btn_Leaderboards/Btn_Community`
- `HUD_Hub.RightCluster`: `Btn_PremiumCTA` (con `CarouselClip`>`Slide1..3`, `TiendaTag`, `PremiumCarousel` LocalScript, `PremiumPulse` LocalScript) + `MicroRow`>`Btn_Micro1..3` (buffs) con `BuffRings` LocalScript
- `HUD_Hub.BottomBar.Btn_Run` (JUGAR)
- `HUD_Hub.CurrencyChip`, `HUD_Hub.LevelBar`
- Scripts: `HUD_Hub.HubController`, `HUD_Hub.HubIntro`, `HUD_Hub.HubBadges` (ModuleScript), `HUD_Hub.BadgeDemo` (DEMO)
- Cada botón: `IconHolder`(`IconBg`+`Shadow`+`IconImage`+`Glyph`+`Rivet1..4`+`CountdownRing` en buffs) + `Label` + `AlertBadge`.

**Pantallas modales (ScreenGui propios, `Enabled=false` al inicio):**
- `StarterGui.SettingsScreen` — `SettingsController`, BindableEvent `OpenSettings`, `Sheet` (side-sheet), `Overlay` (scrim modal).
- `StarterGui.LeaderboardScreen` — `LeaderboardController`, BindableEvent `OpenLeaderboard`, `Card`, `Overlay`.
- `StarterGui.CommunityScreen` — `CommunityController`, BindableEvent `OpenCommunity`, `Card`, `Overlay`.

## 3. Funcionalidad esperada
Hub del lobby, siempre visible. Entrada-firma "reparto de cartas" (`HubIntro`) al spawnear. Botones (disparo por `.Activated`, cableado en `HubController` por nombre):
- Shops (Powerup/Weapon/Coin/Quest) → `Remotes.StoreTeleport_RE:FireServer("<id>")`
- JUGAR → `Remotes.GameStart_RE:FireServer()`
- Cards → `CardSystem.OpenAlbum:Fire()`
- Premium CTA → `StoreTeleport_RE:FireServer("Premium")` (el server abre PremiumShopScreen vía `UITest_RE` "SHOP")
- Invite → `SocialService:PromptGameInvite(player)`
- Ajustes/Ranking/Comunidad → `SettingsScreen.OpenSettings:Fire()` / `LeaderboardScreen.OpenLeaderboard:Fire()` / `CommunityScreen.OpenCommunity:Fire()`

**API mínima de integración:** cada pantalla se abre disparando su BindableEvent (`OpenSettings`/`OpenLeaderboard`/`OpenCommunity`). Badges: `HUD_Hub.HubBadges` módulo → `HubBadges.Set(key, valor|"dot"|"new", {pulse=})` / `HubBadges.Clear(key)`. Anillos de buff: `BuffRings.Start(nombre, duracion)` (al activar buff). Todo motion respeta `player:GetAttribute("Setting_ReduceMotion")`.

## 4. Wired vs stub/mock
| Elemento | Estado |
|---|---|
| Shops, JUGAR, Cards, Premium CTA, Invite, Ajustes/Ranking/Comunidad | **Cableado** (usa remotes/pantallas reales) |
| Daily, Locker, Buffs (Micro1-3) | **Stub** (print; falta sistema/compra) |
| CurrencyChip, LevelBar | **Mock** (valores hardcodeados) |
| LeaderboardScreen | **Mock** (datos falsos) |
| SettingsScreen (8 toggles) | Funcional; guarda `player:SetAttribute("Setting_<key>")`; falta apply real + persistencia |
| CommunityScreen | Funcional; handles **placeholder** (discord.gg/____, @tucanal) |
| AlertBadges, BuffRings, PremiumCarousel | Sistemas listos; falta que server/compra los dispare |

## 5. Pendientes de backend (server-autoritativo)
1. **Currency + Level:** server empuja monto/nivel reales a `CurrencyChip.Amount` y `LevelBar`.
2. **Buffs (microtx):** `MarketplaceService:PromptProductPurchase`; al confirmar → activar buff + `BuffRings.Start(nombre, dur)`.
3. **Badges:** server empuja estado (`HubBadges.Set(...)`).
4. **Leaderboard:** `OrderedDataStore` (score al terminar run, key por semana) + `GetUserThumbnailAsync`. Reemplazar `MOCK` en `LeaderboardController`.
5. **Settings:** hooks reales (SoundService, etc.) + persistencia server.
6. **Daily / Locker:** sistemas por construir.
7. **Community:** handles reales + íconos (Roblox no abre links externos; se muestran para copiar).

## 6. No-colisión + integración
**Nombres NUEVOS a verificar contra producción antes de integrar** (deben estar libres):
`HUD_Hub`, `SettingsScreen`, `LeaderboardScreen`, `CommunityScreen` (ScreenGuis); controllers `HubController/HubIntro/HubBadges/BadgeDemo/PremiumCarousel/PremiumPulse/BuffRings/SettingsController/LeaderboardController/CommunityController`; BindableEvents `OpenSettings/OpenLeaderboard/OpenCommunity`, `HUD_Hub.RightCluster.MicroRow` etc. → **recomendación ENG: prefijar si producción ya usa alguno.**

**Dependencias que DEBEN existir en producción** (el feature las reusa, no las duplica): `ReplicatedStorage.Remotes.StoreTeleport_RE`, `GameStart_RE`, `Weapon_RE`, `Quest_RE`; `ReplicatedStorage.UITest_RE`; `ReplicatedStorage.CardSystem` (con `OpenAlbum`); `ReplicatedStorage.ScreenPolicy`; `StarterGui.ResponsiveManager`, `ButtonEffects`, `ModalInputGuard`. (Verificado presentes en el place dev.)

**⚠️ Modificaciones a sistemas EXISTENTES (NO aditivo — ENG debe portar estos edits o replicarlos):**
- `StarterGui.PremiumShopScreen.ShopController` L43: texto del tab activo → blanco (legibilidad).
- `StarterGui.CardTempHUD.TempHUDController` L25: auto-heal `gui.Enabled=true`→`false` (el bolsillo es de combate, no lobby; el HUD in-run debe encenderlo).
- `StarterGui.HUD_Lobby.Enabled=false` (lobby viejo apagado; el nuevo hub lo reemplaza).

**Pasos de integración:** copiar por Ctrl+C/Ctrl+V nativo: `HUD_Hub`, `SettingsScreen`, `LeaderboardScreen`, `CommunityScreen`. Aplicar los 3 edits a sistemas existentes de arriba. Confirmar dependencias presentes. Apagar `HUD_Lobby` viejo.

**Antes de publicar:** quitar `BadgeDemo` (tecla B); `BuffRings.DEMO=false` (ya); verificar todos los modales `Enabled=false`; dejar de usar mock en currency/level/leaderboard; borrar `HUD003old` (prototipo deprecado).

## 7. Asset IDs
**Iconos menú:** Powerup 136945440097521 · Coins 72740637637102 · Quest 90252728774391 · Weapon 105453835761655 · Album 134807568784253 · Daily 76559523531180 · Invite 81699509063596 · Play 126417442548902 · Locker 88085627323218
**Esquina:** Settings 105727285373917 · Leaderboards 76053633206884 · Community 108093117836953
**Premium (carrusel):** Gold 100998589621740 · VFX Eliminations 88092732523832 · VFX Bullets 91013626618069
**Buffs:** Health 99080845134522 · Damage 110652198972192 · Speed 75056607664027
