# 05. Furniture & Decor

> All furniture, decorations, paintings, shelves, beds, tables, chairs, flower pots, wind chimes, and model ships.
> Complements notes 22 (Shipyard), 14 (Boat Customization).

---

## Beds

| Prefab Index | Name | Notes |
|:-----------:|------|-------|
| 60 | bed A | Aestrin style |
| 61 | bed E | Emerald style |
| 62 | bed M | Medi style |
| 63 | bed M 2 | Medi style variant |

**Behavior:** `ShipItemBed` with `sleepPos`/`wakePos` transforms. Alt-activate → `Sleep.instance.EnterBed(transform)`. Must be `sold` (purchased) and not already sleeping/swimming/in menu.

---

## Tables

| Prefab Index | Name | Style |
|:-----------:|------|:-----:|
| 64 | table M small | Medi small |
| 65 | table M large | Medi large |
| 66 | table A small | Aestrin small |
| 67 | table A large | Aestrin large |
| 68 | table E small | Emerald small |
| 69 | table E large | Emerald large |
| 72 | table generic | Generic |
| 73 | table generic 2 | Generic variant |

---

## Chairs

| Prefab Index | Name | Style |
|:-----------:|------|:-----:|
| 74 | chair M small | Medi small |
| 75 | chair M large 1 | Medi large |
| 76 | chair M large 2 | Medi large variant |
| 77 | chair E | Emerald |
| 78 | chair A | Aestrin |

---

## Shelves

| Prefab Index | Name | Type |
|:-----------:|------|------|
| 180 | shelves A | Standing shelves (Aestrin) |
| 185 | wall shelf A1N | Wall-mounted (Aestrin) |
| 186 | wall shelf E1N | Wall-mounted (Emerald) |
| 187 | wall shelf M1N | Wall-mounted (Medi) |

Wall shelves likely have `wallAttachment = true`.

---

## Baskets

| Prefab Index | Name |
|:-----------:|------|
| 109 | basket A |

---

## Flower Pots

| Prefab Index | Name | Size |
|:-----------:|------|:----:|
| 190 | flower pot | Standard |
| 191 | flower pot 1 | Standard |
| 192 | flower pot 2 | Standard |
| 193 | flower pot 3 | Standard |
| 194 | flower pot 4 | Small |
| 195 | flower pot 5 | Small |
| 196 | flower pot 6 | Big |

---

## Wind Chimes

| Prefab Index | Name | Material |
|:-----------:|------|----------|
| 135 | wind chimes metal | Metal |
| 136 | wind chimes bamboo | Bamboo |

**Behavior:** `Windchimes` component — plays sound based on wind. `WindCloth` or `WindClothSimple` for visual sway.

---

## Model Ships

| Prefab Index | Name | Size |
|:-----------:|------|:----:|
| 137 | model ship junk (big) | Large |
| 138 | model ship junk (small) | Small |

Decorative junk ship models for cabin.

---

## Paintings (Indices 120–263)

### Aestrin Paintings

| Index | Name |
|:-----:|------|
| 120 | painting 0 A |
| 121 | painting 1 A |
| 124 | painting 4 A |
| 240 | painting A |
| 244 | painting A |
| 246 | painting A |
| 250 | painting A |
| 252 | painting A |
| 254 | painting A |
| 256 | painting A |

### Emerald Paintings

| Index | Name |
|:-----:|------|
| 241 | painting E |
| 242 | painting E |
| 247 | painting E |
| 253 | painting E |
| 260 | painting E (no mesh) |
| 263 | painting E |

### Medi Paintings

| Index | Name |
|:-----:|------|
| 125 | painting 5 M |
| 127 | painting 7 M |
| 128 | painting 8 M |
| 243 | painting M |
| 245 | painting M |
| 249 | painting M |
| 251 | painting M |
| 255 | painting M (chronos) |
| 257 | painting M |
| 258 | painting M (chronos) |
| 259 | painting M |
| 262 | painting M (Porthos) |

### Chronos Paintings

| Index | Name |
|:-----:|------|
| 122 | painting 2 (chronos) |
| 123 | painting 3 (chronos) |
| 126 | painting 6 chronos maybe |
| 129 | painting 9 (chronos) |
| 248 | painting M (chronos) |
| 255 | painting M (chronos) |
| 258 | painting M (chronos) |

### Special Paintings

| Index | Name |
|:-----:|------|
| 261 | painting SAILWIND |

---

## Totems

| Prefab Index | Name | Effect |
|:-----------:|------|--------|
| 163 | totem sun | Weather — sun/clear |
| 164 | totem rain | Weather — rain/storm |

**Behavior:** `ShipItemTotem` with `stormAttraction` float. Alt-held to cast. Particle effects + audio. After cast: `health = -1` (depleted). Rune and totem materials change during cast.

---

## Elixir Tribal Disco

`ElixirTribalDisco` class — likely a special decorative/ritual item related to elixirs. Prefab index not confirmed.

---

## Furniture Properties

All furniture items typically have:
- `big = true` (large items, two-handed carry)
- `wallAttachment = true` for shelves (nail to wall)
- `category = TransactionCategory.furniture`
- Higher `mass` than small items (chairs ~5–10kg, beds ~15–25kg)

### Wall Attachment
Items with `wallAttachment = true` use raycast forward from `itemRigidbody` at 1.3m distance. On valid hit:
- Calculates `attachPos`/`attachRot` from hit point and normal
- On drop: snaps to wall position, `ItemRigidbody.attached = true`
- Normal check: if `normal.y > 0.8` → use `transform.up` for rotation (ceiling/floor)

---

*Extracted from Sailwind v0.38 asset analysis.*
