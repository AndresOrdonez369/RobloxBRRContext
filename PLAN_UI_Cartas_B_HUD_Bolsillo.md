# PLAN B — UI de Cartas: HUD de Bolsillo in-run (temporales + reveal + cashout)

> **Uno de dos planes paralelos.** Este es el **Plan B (HUD in-run)**. Su gemelo es
> [`PLAN_UI_Cartas_A_Album.md`](PLAN_UI_Cartas_A_Album.md) (**Plan A**).
> Los dos se pueden trabajar **al mismo tiempo, en el mismo place, sin chocar**, porque
> cada uno es dueño de ScreenGuis, controllers y remotes distintos (ver §6).
>
> **Place objetivo:** `Carrera de habitaciones de Brainrot [BETA]` (producción / Hotel Havoc).
> **Estrategia elegida por el dueño (2026-07-23):** *UI nueva sobre backend viejo.* Se porta
> **solo la UI** del handoff `tvg_andy` (`CardSystem_Handoff.md`) como piel nueva del sistema de
> cartas que **YA existe y está completo en producción**. **No** se trae el backend de `tvg_andy`.

---

## 0. Contexto crítico (leer antes de tocar nada)

Producción ya tiene el sistema de cartas completo, apagado tras
`ReplicatedStorage.Shared.Config.CardsConfig.ENABLED = false`. Backend server relevante para este
plan (**todo hecho, no se toca**):

- `ServerScriptService.Services.Cards.CardRunService` — inventario **pendiente** por run (efímero).
  Al recoger una carta emite el sync de pendiente; al morir Retry lo limpia; revive/DBNO lo conserva.
- `ServerScriptService.Services.Cards.CardCashoutService` — estaciones (tag `CardCashoutStation`) +
  cashout al escapar. Al asegurar, emite el resumen de cashout.
- **Modelo de 2 fases** (ver `PLAN_Sistema_Cartas_Coleccionables.md` §3): recoger = **pendiente**
  (no persiste); solo el **cashout** (estación o escape) lo vuelve permanente; **morir sin revivir
  = se pierde el pendiente**.
- Ya existe un controller de pendiente: `StarterPlayer.StarterPlayerScripts.Controllers.CardPendingHUD`
  (hoy muestra el pendiente como fichas de rareza junto a `BoonHUD`). **Este plan lo reemplaza/expande
  con la UI pulida del bolsillo.**

**Diferencia de modelo que HAY que respetar** (el handoff `tvg_andy` asumía otra cosa):

| El handoff `tvg_andy` asumía… | Producción usa… |
|---|---|
| Rareza multi-nivel (`{id,isNew,level}`) | **count + rareza fija**; el pendiente es `{[cardId]=count}` con rareza de `CardCatalog` |
| `OpenTempBag` BindableEvent, `CardSystem_RE` | Remotes de `Shared.Types.Remotes` (ver §2) |
| "Vault" como zona con tag `CardVault` | Estaciones con tag `CardCashoutStation` + cashout al escapar (ya en el backend) |

➡️ **Al portar, se ELIMINA todo lo de "level".** Las fichas del bolsillo se colorean por **rareza**
(`CardRarityConfig`), y el contador es la **cantidad** de cartas pendientes.

---

## 1. Objetivo de este plan

Portar la UI in-run pulida de `tvg_andy` (`CardTempHUD` + `CardTempScreen`) como front-end del
**pendiente del run** de producción. Incluye:

1. **Icono de bolsillo** en esquina + **contador** con tensión (juice) — `CardTempHUD`.
2. **Juice de obtención**: al recoger, la carta "vuela" al bolsillo (zoom/anim).
3. **Reveal al recoger**: mostrar momentáneamente la carta real (o su rareza) al recogerla
   (decisión menor pendiente §10.9 del plan de backend — imagen completa vs solo rareza).
4. **Overlay modal de temporales** (`CardTempScreen`): grid de las cartas pendientes del run +
   badge NUEVA.
5. **Feedback de cashout**: al asegurar en estación/escape, animación de "asegurado" + reset del
   bolsillo a 0.
6. **Feedback de pérdida**: al morir sin revivir, vaciar el bolsillo con feedback de pérdida.

Fuera de alcance (es del Plan A): álbum, colección permanente, equipar, `HUD003.CardsButton`.

---

## 2. Contrato con el backend (lo que este plan CONSUME, no modifica)

Remotes en `ReplicatedStorage.Shared.Types.Remotes` (confirmar nombres exactos leyendo
`Shared.Types.Remotes` + `CardRunService` + `CardCashoutService`):

| Remote | Dirección | Uso en el HUD in-run |
|---|---|---|
| `CardPendingSync` (`"CardPendingSync"`) | server→client | Estado incremental del pendiente `{[cardId]=count}`. Actualiza contador del bolsillo, fichas por rareza, y el grid del overlay. |
| `CardCashoutDone` (`"CardCashoutDone"`) | server→client | Resumen del cashout (cartas nuevas, duplicados, coins, rarezas). Dispara la animación de "asegurado" + reset a 0. |

- **Resultado de muerte (pendiente perdido vs conservado):** el backend ya limpia/conserva el
  pendiente correctamente (revive/DBNO conserva, Retry pierde). Para el **feedback visual** de
  pérdida, engancharse a las señales de muerte que ya existen en producción:
  `ReplicatedStorage.DownSystem.OnPlayerDead` / `OnPlayerRevived` / `OnPlayerDowned` (BindableEvents),
  o la señal que consuma el `CardPendingHUD` actual. **Leer cómo el `CardPendingHUD` actual ya
  distingue Retry vs revive** y reusar ese mismo criterio (coherente con `CardRunService`).
- **Estilo por rareza (reuso, read-only):** `BoonCardStyler` + `CardRarityConfig` (color por tier).
- **Datos de carta para el reveal/overlay (read-only):** `CardCatalog` (arte, displayName, rareza).
  El `cardId` real llega en el sync del pendiente; recién ahí se conoce la carta (en el mundo es
  incógnita — ver §5.12 del plan de backend; el reveal es el momento sorpresa).

> Si falta un remote o señal, **NO** modificar el server: anotar y coordinar. El backend está hecho.

---

## 3. Archivos de los que este plan es DUEÑO (puede escribir)

| Ruta | Acción |
|---|---|
| `StarterGui.CardTempHUD` | **NUEVO** — portar de `tvg_andy` (Ctrl+C/Ctrl+V nativo). Icono de bolsillo + contador + FX del juice. |
| `StarterGui.CardTempScreen` | **NUEVO** — portar de `tvg_andy`. Overlay modal de cartas temporales (grid + badges NUEVA) + `CloseBtnTemplate`. |
| `StarterPlayer.StarterPlayerScripts.Controllers.CardPendingHUD` | **MOD (reescribir vista)** — pasar de "fichas junto a BoonHUD" a manejar `CardTempHUD`/`CardTempScreen`, manteniendo su cableado a los remotes de pendiente y a la señal de resultado de muerte. Controller **exclusivo de este plan**. |
| Controllers de temp portados (`TempHUDController`/`TempScreenController` si se traen como scripts propios) | **NUEVO** — nombres únicos, sin chocar con nada. Alternativa: fusionar su lógica dentro de `CardPendingHUD`. **Decidir 1 de 2** (ver §4.6). |

**Assets del bolsillo (del handoff `CardSystem_Handoff.md` §7):**
| Uso | Asset |
|---|---|
| Icono/arte del bolsillo + juice | `rbxassetid://83104017518458` |

---

## 4. Adaptaciones obligatorias al portar (no es copy-paste)

1. **Quitar el modelo multi-nivel.** Nada de `level` en el payload ni en las fichas. Ficha =
   color por **rareza** (`CardRarityConfig`); contador = **cantidad** pendiente.
2. **Fuente del pendiente = `CardPendingSync`** (incremental), no el `TempGained` del handoff.
   Mapear el juice de obtención a cada incremento del sync.
3. **Cashout, no "Vault".** El reset del bolsillo lo dispara `CardCashoutDone`, no un touch de
   `CardVault`. El backend maneja estación/escape; este plan solo reacciona al resumen.
4. **Pérdida al morir con criterio correcto.** Vaciar el bolsillo visualmente **solo** cuando el
   resultado sea Retry→lobby. En revive/DBNO **conservar** (espejar exactamente lo que hace hoy
   `CardPendingHUD` / `CardRunService`). No inventar la regla: reusar la señal existente.
5. **Reveal.** Al llegar un incremento de pendiente, hacer el reveal momentáneo (imagen o rareza).
   Como en el mundo la carta es incógnita, el reveal es la **primera** vez que el jugador ve la
   carta — cuidar timing y que no tape el gameplay.
6. **Dónde vive la lógica.** Producción tiene los controllers en
   `StarterPlayerScripts.Controllers` (no dentro del ScreenGui como en `tvg_andy`). **Recomendado:**
   consolidar toda la lógica en `CardPendingHUD` y dejar los ScreenGuis como vista pura, para ser
   consistente con producción. Si se traen `TempHUDController`/`TempScreenController` embebidos,
   verificar que sus nombres no choquen y que no dupliquen conexiones a los remotes.
7. **Responsive.** Producción **no** tiene `ResponsiveManager`/`ScreenPolicy` (eran de `tvg_andy`).
   Usar el patrón de producción (`UIAspectRatioConstraint`/`Overlay`, ver `WeaponScreen`,
   `QuestScreen`). No portar dependencias de responsive de `tvg_andy`.
8. **Modales cerrados al inicio.** `CardTempScreen.Enabled=false`. `CardTempHUD.Enabled` según
   decisión: es HUD de **combate/in-run**, no de lobby → lo enciende el flujo in-run, no siempre.

---

## 5. Pasos

1. **Leer** `CardPendingHUD` actual (cómo consume el pendiente y la señal de muerte) +
   `Shared.Types.Remotes` + `CardRunService`/`CardCashoutService` (shape del sync y del resumen) +
   `CardCatalog`/`CardRarityConfig` + `BoonCardStyler`.
2. **Portar** `CardTempHUD` y `CardTempScreen` a `StarterGui` (Ctrl+C/Ctrl+V nativo).
3. **Reescribir** `CardPendingHUD` para manejar las nuevas vistas (§4): contador + fichas por
   rareza + juice, overlay con grid de pendientes, reveal.
4. **Cablear** `CardPendingSync` (incremental) y `CardCashoutDone` (reset + anim asegurado).
5. **Cablear** feedback de pérdida a la señal de muerte (criterio Retry vs revive/DBNO reusado).
6. **Smoke test** con `CardsConfig.ENABLED=true` **local** (revertir a `false` antes de commitear):
   matar enemigo → recoger → reveal + bolsillo sube; abrir overlay temporales; llegar a estación
   `CardCashoutStation` / escapar → asegurado + reset; morir Retry → bolsillo se vacía con feedback;
   revivir → se conserva.

---

## 6. Frontera y reglas de no-colisión (idénticas en Plan A y Plan B)

**Por qué no chocan:** cada plan es dueño de ScreenGuis y controllers **distintos** y consume
remotes **distintos**. En Team Create nunca editan la misma instancia.

| Recurso | Plan A (Álbum) | Plan B (HUD bolsillo) |
|---|---|---|
| ScreenGuis | `CardAlbumScreen`, (deprecar `CardCollectionGui`) | `CardTempHUD`, `CardTempScreen` |
| Controllers | `CardCollectionController` | `CardPendingHUD` (+ controllers de temp) |
| Remotes | `CardCollectionSync`, `CardCollected`, `CardLoadoutSync`, `CardEquipRequest` | `CardPendingSync`, `CardCashoutDone` |
| Punto en `HUD003` | `CardsButton` (abrir álbum) — **solo A** | **no toca `HUD003`** |
| Superficie | Menú / álbum | In-run / bolsillo |

**Reglas compartidas (ambos las cumplen):**

- 🔒 **`CardsConfig.ENABLED`**: para testear hay que ponerlo en `true`. **Nadie commitea con
  `ENABLED=true`.** Cada quien lo pone local, prueba, y lo revierte a `false`. El flip final a
  `true` es un paso de **integración** posterior, con el dueño, cuando ambas UIs estén listas.
- 📖 **Configs de cartas** (`CardCatalog`, `CardRarityConfig`, etc.): **read-only** para ambos. Si
  hay que cambiar datos (ej. textura faltante), lo hace el **owner de datos** (Plan A §7) y avisa.
  Plan B **no** edita `CardCatalog`.
- 🚫 **Backend server** (`Services.Cards.*`, framework de efectos, `BoonManager`): **ninguno de los
  dos planes lo edita.**
- ♻️ **`BoonCardStyler`**: reuso read-only por ambos.
- ⚠️ **`BoonHUD`**: el `CardPendingHUD` actual comparte área con `BoonHUD`. Al mover el pendiente al
  bolsillo (`CardTempHUD`), **no romper `BoonHUD`** (los boons siguen mostrándose). Si se deja de
  usar el área compartida, quitar solo la parte de cartas, no la de boons. `BoonHUD` no es de este
  plan; tocarlo mínimamente y solo para desacoplar las fichas de carta.

---

## 7. Riesgos específicos de este plan

- **Timing del reveal vs gameplay in-run:** el reveal no debe tapar la mira ni bloquear input en
  combate. Preferir una animación corta en esquina/centro-inferior.
- **Coherencia muerte/revive:** el bug clásico es vaciar el bolsillo cuando el jugador iba a
  revivir. **Reusar** el criterio existente (`DeathService.getPendingState == "Dying"` /
  la señal que ya usa `CardPendingHUD`), no reimplementarlo.
- **Incógnita en el mundo:** el cliente no debe filtrar la textura real antes del reveal; el
  `cardId` llega en el sync del pendiente (post-recogida), que es cuando se revela. No pedir la
  textura antes.

---

## 8. Checklist de cierre

- [ ] `CardTempHUD` + `CardTempScreen` portados; modales cerrados al inicio.
- [ ] Cero rastro del modelo multi-nivel; fichas por rareza, contador = cantidad pendiente.
- [ ] Pendiente cableado a `CardPendingSync` (incremental) con juice de obtención + reveal.
- [ ] Cashout cableado a `CardCashoutDone` (anim asegurado + reset a 0).
- [ ] Pérdida al morir Retry con feedback; conservado en revive/DBNO (criterio reusado).
- [ ] `BoonHUD` intacto (boons siguen visibles).
- [ ] Responsive con el patrón de producción (no dep. de `tvg_andy`).
- [ ] `CardsConfig.ENABLED` revertido a `false` en el commit.
- [ ] No se tocó ningún archivo del Plan A (`CardAlbumScreen`/`CardCollectionController`/`CardsButton`) ni el backend server.
```
