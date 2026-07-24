# GUÍA — Subir texturas de cartas al álbum

> Place objetivo: **Carrera de habitaciones de Brainrot [BETA]**.
> Datos: `ReplicatedStorage.Shared.Config.CardCatalog` (campo `textureId` por carta).
> Estado actual: **las 60 cartas** tienen `textureId = "rbxassetid://0"` (placeholder).
> El álbum ya funciona con placeholder; subir texturas es un paso **incremental** — no bloquea nada.

---

## 0. Concepto clave: NO son 60 imágenes, son ~31

Muchas cartas comparten personaje y solo cambian de rareza (el marco lo pone la UI, no la imagen).
Ejemplo: `card_045..card_048` = "Ballerina Cappuccina" en 4 rarezas → **una sola imagen** reusada en las 4.

Personajes únicos a subir:

| Rango de cartas | Personaje | Imágenes |
|---|---|---|
| `card_001` … `card_018` | Set Hotel (18 distintos, uno c/u) | 18 |
| `card_023` | Lirili Larila | 1 |
| `card_026`–`028` | Tung Tung Tung Sahur | 1 |
| `card_029`–`033` | Trippi Troppi Troppa Trippa | 1 |
| `card_034`–`036` | Pi Pi Watermelon | 1 |
| `card_037`–`038` | Avocadorilla | 1 |
| `card_039`–`041` | La Vacca Saturno Saturnita | 1 |
| `card_042`–`044` | Cappuccino Assassino | 1 |
| `card_045`–`048` | Ballerina Cappuccina | 1 |
| `card_049`–`053` | Pot Hotspot | 1 |
| `card_054`–`057` | Boneca Ambalabu | 1 |
| `card_058`–`062` | Tralalero Tralala | 1 |
| `card_063`–`066` | Trenostruzzo Turbo 3000 | 1 |
| **Total** | | **~31** |

➡️ Subes **una imagen por personaje** y pegas el **mismo** `assetId` en todas las filas de rareza de ese personaje.

---

## 1. Preparar los archivos

- Formato: **PNG** (soporta transparencia; ideal si el arte tiene fondo recortado).
- Tamaño: **512×512** o **1024×1024** cuadrado (las celdas del grid son cuadradas).
- Nombre del archivo = id o personaje, ej. `card_045_ballerina.png`, `card_001_botones.png`.
  El nombre no afecta al juego, pero ayuda a no perderte al pegar los IDs.

---

## 2. Subir (Asset Manager en Studio — la vía escalable)

> Sube desde la **misma cuenta/grupo dueño de la experiencia**. Assets subidos por otra cuenta
> quedan **no autorizados** y **fallan en silencio** (la imagen no aparece, sin error). Es un
> problema conocido de este proyecto.

1. En Studio, con el place abierto: **View → Asset Manager**.
2. Panel Asset Manager → carpeta **Images**.
3. Click derecho → **Add Images…** (permite **selección múltiple** — sube las ~31 de una).
4. Roblox mete cada imagen a **moderación** (minutos a horas). Mientras esté pendiente no renderiza.
5. Cuando estén aprobadas: click derecho en cada imagen → **Copy Asset ID**.
   Da un número; el formato final es `rbxassetid://<ese_numero>`.

### Alternativa: Creator Dashboard (web)
`create.roblox.com` → **Creations** → tu experiencia → **Assets/Images** → **Upload**.
Mismo resultado; útil si prefieres el navegador. Misma regla de cuenta dueña.

---

## 3. Pegar los IDs en `CardCatalog`

Ruta: `ReplicatedStorage.Shared.Config.CardCatalog` → tabla `Cards`.
Cambia solo el campo `textureId` de cada fila. **No** toques `id`, `rarity`, `coinsPerSecond`.

Antes:
```lua
card_045 = { id = "card_045", displayName = "Ballerina Cappuccina", rarity = "Common",    textureId = "rbxassetid://0", coinsPerSecond = 2 },
card_046 = { id = "card_046", displayName = "Ballerina Cappuccina", rarity = "Uncommon",  textureId = "rbxassetid://0", coinsPerSecond = 4 },
card_047 = { id = "card_047", displayName = "Ballerina Cappuccina", rarity = "Rare",      textureId = "rbxassetid://0", coinsPerSecond = 8 },
card_048 = { id = "card_048", displayName = "Ballerina Cappuccina", rarity = "Epic",      textureId = "rbxassetid://0", coinsPerSecond = 16 },
```

Después (mismo assetId en las 4 filas del personaje):
```lua
card_045 = { id = "card_045", displayName = "Ballerina Cappuccina", rarity = "Common",    textureId = "rbxassetid://123456789", coinsPerSecond = 2 },
card_046 = { id = "card_046", displayName = "Ballerina Cappuccina", rarity = "Uncommon",  textureId = "rbxassetid://123456789", coinsPerSecond = 4 },
card_047 = { id = "card_047", displayName = "Ballerina Cappuccina", rarity = "Rare",      textureId = "rbxassetid://123456789", coinsPerSecond = 8 },
card_048 = { id = "card_048", displayName = "Ballerina Cappuccina", rarity = "Epic",      textureId = "rbxassetid://123456789", coinsPerSecond = 16 },
```

Reemplaza `123456789` por tu número real. Guarda el script.

---

## 4. Verificar

1. El álbum detecta `textureId ~= "rbxassetid://0"` y **automáticamente** cambia la cara placeholder
   por la imagen — **sin tocar código**. No hay lista de IDs hardcodeada.
2. Abre el álbum (`HUD003.CardsButton`) y confirma que la carta muestra el arte.
3. Si sale en blanco: (a) el asset sigue en moderación, o (b) se subió desde una cuenta que **no**
   es dueña de la experiencia → re-subir desde la cuenta dueña.

---

## 5. Escalabilidad (agregar cartas nuevas a futuro)

- Agregar carta = **una fila** en `CardCatalog.Cards`. El álbum la pinta sola (grid desde
  `listIds()`, footer desde `count()`, filtros desde `CardRarityConfig.Order`). Cero cambios de UI.
- Rareza nueva = agregarla a `CardRarityConfig.Order`; aparece un chip de filtro automático.
- Personaje nuevo con varias rarezas = subir 1 imagen, repetir el `assetId` en sus filas.
- `rbxassetid://0` siempre es el fallback seguro: cara neutra + nombre, nunca rompe.

---

## Checklist

- [ ] ~31 imágenes cuadradas PNG listas y nombradas.
- [ ] Subidas por Asset Manager **desde la cuenta/grupo dueño** de la experiencia.
- [ ] Moderación aprobada.
- [ ] `textureId` actualizado en `CardCatalog` (mismo ID en filas del mismo personaje).
- [ ] Álbum muestra el arte; ninguna carta en blanco.
