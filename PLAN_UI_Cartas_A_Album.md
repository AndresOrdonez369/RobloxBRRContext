# PLAN A — UI de Cartas: Álbum (colección + equipar)

> **Uno de dos planes paralelos.** Este es el **Plan A (Álbum)**. Su gemelo es
> [`PLAN_UI_Cartas_B_HUD_Bolsillo.md`](PLAN_UI_Cartas_B_HUD_Bolsillo.md) (**Plan B**).
> Los dos se pueden trabajar **al mismo tiempo, en el mismo place, sin chocar**, porque
> cada uno es dueño de ScreenGuis, controllers y remotes distintos (ver §6).
>
> **Place objetivo:** `Carrera de habitaciones de Brainrot [BETA]` (producción / Hotel Havoc).
> **Estrategia elegida por el dueño (2026-07-23):** *UI nueva sobre backend viejo.* Se porta
> **solo la UI** del handoff `tvg_andy` (`CardSystem_Handoff.md`) como piel nueva del sistema de
> cartas que **YA existe y está completo en producción**. **No** se trae el backend de `tvg_andy`.

---

## 0. Contexto crítico (leer antes de tocar nada)

Producción **ya tiene un sistema de cartas completo**, distinto al del handoff, apagado tras
`ReplicatedStorage.Shared.Config.CardsConfig.ENABLED = false`:

- **Backend server (todo hecho):** `ServerScriptService.Services.Cards.*` — `CardService`,
  `CardDropService`, `WorldCardService`, `CardRunService`, `CardCashoutService`,
  `CardLoadoutService`, `CardShelfService`, `CardEffectDefinitions`, `EffectArchetypes`, y el
  framework `Effects/` con 18 definiciones.
- **Configs (todos):** `CardCatalog` (18 cartas), `CardRarityConfig` (5 tiers), `CardDropConfig`,
  `CardsConfig`, `CardDupeRewardConfig`, `LuckConfig`, `CardEffectBindings`.
- **UI actual:** `StarterGui.CardCollectionGui` + controller `CardCollectionController`.
- **Doc de verdad del backend:** `PLAN_Sistema_Cartas_Coleccionables.md` y
  `GUIA_Implementar_Efecto_De_Carta.md` (en este repo).

**Diferencia de modelo de datos que HAY que respetar** (el handoff `tvg_andy` asumía otra cosa):

| El handoff `tvg_andy` asumía… | Producción usa… |
|---|---|
| 15 cartas | **18 cartas** (`CardCatalog`) |
| Rareza **multi-nivel** por carta (1..5, "mejor nivel") | **count + rareza fija** por carta: `data.cards[cardId] = count`, la rareza vive en `CardCatalog[cardId].rarity` |
| `_G.CardService`, `CardSystem_RE/RF`, `OpenAlbum` BindableEvent | Remotes `Shared.Types.Remotes` (ver §2) + backend por módulos |
| Efectos "solo visuales" | Efectos **reales** vía framework de Boons (no los tocamos aquí) |

➡️ **Al portar la UI del álbum, se ELIMINA todo lo de "track de niveles / mejor nivel".**
La celda de carta muestra: arte + marco por rareza + badge de **cantidad** (`xN`) + estado
owned/locked. El panel de detalle muestra arte + rareza + descripción del efecto
(`CardEffectDefinitions[cardId].describe`) — **no** una barra de niveles.

---

## 1. Objetivo de este plan

Reemplazar la UI de colección actual (`CardCollectionGui`) por la **UI de álbum pulida** portada
de `tvg_andy` (`CardAlbumScreen`), cableada al **backend de producción**. Incluye:

1. **Grid de colección** de las 18 cartas por rareza (owned/locked, badge de cantidad, marco por
   rareza).
2. **Panel de detalle** (arte hero + rareza + descripción de efecto).
3. **Filtros** ALL / EQUIPPED / por rareza + búsqueda — **generados desde `CardRarityConfig.Order`**
   (no hardcodear los tiers).
4. **Sección Equipadas (3 slots)** con equip/unequip server-authoritative.
5. **Footer de colección** (progreso).
6. **Apertura** desde el botón que ya existe: `StarterGui.HUD003.CardsButton`.

Fuera de alcance de este plan (es del Plan B): bolsillo temporal, reveal al recoger, feedback de
cashout, HUD in-run.

---

## 2. Contrato con el backend (lo que este plan CONSUME, no modifica)

Remotes en `ReplicatedStorage.Shared.Types.Remotes` (patrón `Remotes.fireClient`/`Remotes.onClient`;
confirmar los nombres exactos leyendo `Shared.Types.Remotes` y `CardService`/`CardLoadoutService`):

| Remote | Dirección | Uso en el álbum |
|---|---|---|
| `CardCollectionSync` (`"CardCollectionSync"`) | server→client | Estado de la colección permanente `{[cardId]=count}`. Push inicial + incrementales. Pinta owned/locked + badge `xN`. |
| `CardCollected` (`"CardCollected"`) | server→client | Toast "¡Nueva carta!" cuando una carta se asegura por primera vez. |
| `CardLoadoutSync` (`"CardLoadoutSync"`) | server→client | Estado de los 3 slots equipados. Pinta la sección Equipadas. |
| `CardEquipRequest` (`"CardEquipRequest"`) | client→server | Pedir equip/unequip (el server valida posesión y slots). |

- **Fuente de datos de catálogo/rareza (read-only):** `ReplicatedStorage.Shared.Config.CardCatalog`
  (`get/listIds/getByRarity/count`), `CardRarityConfig` (`Order`, color por tier, displayName).
- **Descripción de efecto para el detalle:** el texto lo da el spec del efecto
  (`describe(params)`); el mapeo carta→efecto está en `CardEffectBindings`. Si una carta no tiene
  binding, mostrar su descripción de catálogo o un placeholder — **no** romper.
- **Estilo por rareza (reuso, read-only):** `StarterPlayer.StarterPlayerScripts.Controllers.BoonCardStyler`.

> Si al implementar falta un remote o el state shape no calza, **NO** modificar el server por tu
> cuenta: anotarlo y coordinar (probablemente ya existe con otro nombre). El backend está completo.

---

## 3. Archivos de los que este plan es DUEÑO (puede escribir)

| Ruta | Acción |
|---|---|
| `StarterGui.CardAlbumScreen` | **NUEVO** — portar de `tvg_andy` por Ctrl+C/Ctrl+V nativo. Chrome como instancias editables (Panel/Content/FilterBar/Grid/Detail/Footer/TitleImage/CloseBtn) + `CloseBtnTemplate`. |
| `StarterPlayer.StarterPlayerScripts.Controllers.CardCollectionController` | **MOD (reescribir vista)** — apuntar al nuevo `CardAlbumScreen` en vez de `CardCollectionGui`, manteniendo su cableado a los remotes de colección + agregando loadout/equip. Este controller es **exclusivo de este plan**. |
| `StarterGui.CardCollectionGui` | **DEPRECAR** — una vez migrada la vista, dejar `Enabled=false` y marcar para borrar (no borrar hasta QA). |
| `StarterGui.HUD003.CardsButton` | **WIRE (solo este botón)** — que abra el álbum nuevo. Es el único punto de `HUD003` que toca este plan (el Plan B no toca `HUD003`). |

**Assets del álbum (del handoff `CardSystem_Handoff.md` §7):**
| Uso | Asset |
|---|---|
| Fondo del álbum (libro) | `rbxassetid://135331975348156` |
| Título del álbum (banner) | `rbxassetid://114376698013456` |
| Icono moneda (footer) | `rbxassetid://81917086216085` |
| Icono suerte (footer) | `rbxassetid://110166557754419` |
| Icono daño (footer) | `rbxassetid://93320968714938` |

> **Footer de efectos:** en `tvg_andy` los chips coin/luck/damage eran placeholder `+0%`. En
> producción los efectos son reales pero **por carta** (no un agregado global simple). Para V1 del
> footer: mostrar progreso de colección (`X/18`) y, si se quiere, un resumen de las 3 equipadas.
> **No** inventar porcentajes agregados. Marcar como decisión menor con diseño.

---

## 4. Adaptaciones obligatorias al portar (no es copy-paste)

1. **Quitar el modelo multi-nivel.** Eliminar del controller/UI toda referencia a "level",
   "best level", "track de niveles". Sustituir por `count` (badge `xN`) + `rarity` de `CardCatalog`.
2. **18 cartas, no 15.** El grid se llena iterando `CardCatalog.listIds()`, no una lista fija.
   Nada de índices hardcodeados a 15.
3. **Filtros desde config.** Generar los botones de filtro por rareza desde `CardRarityConfig.Order`
   (decisión #8 del plan de backend). ALL + EQUIPPED + un botón por tier + búsqueda por nombre.
4. **Equip = server-authoritative.** Click en carta owned → `CardEquipRequest` (equip en primer
   slot libre). Click en slot equipado → `CardEquipRequest` (unequip). Nunca mutar el estado local
   como fuente de verdad; esperar el `CardLoadoutSync` de vuelta.
5. **Celdas persistentes.** Construir las celdas una vez y filtrar por `Visible`/`LayoutOrder`
   (no recrear). Ya es el patrón del `CardCollectionController` actual — mantenerlo.
6. **Responsive.** Producción **no** tiene `ResponsiveManager`/`ScreenPolicy` (eran del place
   `tvg_andy`). Usar el patrón responsive que use el resto de pantallas de producción
   (`WeaponScreen`, `QuestScreen`, etc. — mirar cómo escalan: `UIAspectRatioConstraint`,
   `Overlay`, `FullBackground`). **No** portar dependencias de responsive de `tvg_andy`.
7. **Modal cerrado al inicio.** `CardAlbumScreen.Enabled=false`; abrir/cerrar por el controller.

---

## 5. Pasos

1. **Leer** `CardCollectionController` actual + `CardCollectionGui` + `Shared.Types.Remotes` +
   `CardLoadoutService` (para el shape exacto de loadout) + `CardCatalog`/`CardRarityConfig`.
2. **Portar** `CardAlbumScreen` de `tvg_andy` a `StarterGui` (Ctrl+C/Ctrl+V nativo).
3. **Adaptar** el controller (§4): grid desde catálogo, badge de cantidad, filtros desde config,
   detalle con descripción de efecto, sección equipadas cableada a loadout.
4. **Cablear equip/unequip** por `CardEquipRequest` + refrescar con `CardLoadoutSync`.
5. **Wire** `HUD003.CardsButton` → abrir álbum.
6. **Deprecar** `CardCollectionGui` (`Enabled=false`).
7. **Smoke test** con `CardsConfig.ENABLED=true` **local** (revertir a `false` antes de commitear —
   ver §6): abrir álbum, ver 18 cartas, filtrar por rareza, buscar, equipar 3, desequipar,
   confirmar que el server responde y el shelf/efectos reaccionan.

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
- 📖 **Configs de cartas** (`CardCatalog`, `CardRarityConfig`, `CardEffectBindings`, etc.): **read-only**
  para ambos planes. Si hay que cambiar datos (ej. textura faltante), lo hace **el owner de datos**
  (este Plan A, ver §7) y se avisa.
- 🚫 **Backend server** (`Services.Cards.*`, framework de efectos, `BoonManager`): **ninguno de los
  dos planes lo edita.** Es UI sobre backend hecho. Si aparece un gap real, se coordina, no se
  parcha por libre.
- ♻️ **`BoonCardStyler`**: reuso read-only por ambos (estilo por rareza).
- 🧩 **Contrato entre superficies**: la única relación A↔B es que ambos reflejan cartas de la misma
  colección/run, pero por remotes distintos. No comparten estado local.

---

## 7. Tarea de datos que posee este plan (compartida con B, la ejecuta A)

Porque el álbum es la superficie que muestra **todas** las cartas, este plan es dueño de alinear los
datos de catálogo (Plan B los consume read-only):

- Verificar que `CardCatalog` tenga las 18 cartas con `textureId` real (no `rbxassetid://0`).
- **Textura faltante conocida:** Trulimero (`T_Card_12` en el handoff / la carta equivalente en
  `CardCatalog`) no tiene ID. Conseguir/subir el asset.
- Confirmar autorización de assets a la experiencia (assets no autorizados fallan en silencio —
  ver memoria de audio/asset permissions del proyecto).

---

## 8. Checklist de cierre

- [ ] `CardAlbumScreen` portado; `CardCollectionGui` en `Enabled=false` (deprecado).
- [ ] Grid pinta las 18 cartas desde `CardCatalog` (sin listas hardcodeadas).
- [ ] Cero rastro del modelo multi-nivel; badge = `count`.
- [ ] Filtros generados desde `CardRarityConfig.Order` + búsqueda funcionando.
- [ ] Equip/unequip por `CardEquipRequest`, refresco por `CardLoadoutSync`, 3 slots.
- [ ] Detalle muestra descripción real del efecto (`describe`) o fallback limpio.
- [ ] `HUD003.CardsButton` abre el álbum; modal cerrado al inicio.
- [ ] Responsive con el patrón de producción (no dep. de `tvg_andy`).
- [ ] `CardsConfig.ENABLED` revertido a `false` en el commit.
- [ ] No se tocó ningún archivo del Plan B ni el backend server.
```
