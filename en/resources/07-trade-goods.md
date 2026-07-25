# 07. Trade Goods & Cargo

> Complete catalog of all trade goods (Good components), cargo items, and their regional origins.
> Complements notes 13 (Economy), 15 (Missions), 45 (Crate/Cargo).

---

## Good System Overview

`Good` is a component attached to tradeable `ShipItem` prefabs. It adds:

| Field | Type | Description |
|-------|------|-------------|
| `nativeRegion` | PortRegion | Region where this good originates |
| `requiredRepLevel` | int | Minimum reputation to purchase |
| `sizeDescription` | string | Size class description |
| `missionIndex` | int | Assigned mission (-1 if none) |
| `dueDay` | int | Mission due day |

**Total goods:** 65 (`Refs.goodCount = 65`)

### Index Mapping
```
Good indices 0–30  → Item indices 0–30  (shared indices)
Good indices 31–64 → Item indices 201–234 (offset +170)
```

---

## Goods 0–30: Consumer Trade Goods

### Food & Fish Crates

| Good | Item | Name | Notes |
|:----:|:----:|------|-------|
| 0 | 0 | salmon (E) | Raw fish — Emerald |
| 1 | 1 | crate salmon (E) | Crated Emerald salmon |
| 2 | 2 | crate dates (good)(N | Dried dates |
| 3 | 3 | crate coconuts (good) | Coconuts |
| 4 | 4 | crate lamb (good) | Lamb meat |
| 7 | 7 | crate cheese (good) | Cheese |
| 8 | 8 | crate goat cheese (good) | Goat cheese |
| 9 | 9 | crate sunspot fish (A)(N | Aestrin sunspot fish |
| 14 | 14 | crate north fish (M) | Medi north fish |
| 15 | 15 | crate sausages | Sausages |
| 16 | 16 | crate pork | Pork |
| 17 | 17 | crate bananas(N | Bananas |
| 18 | 18 | crate trout (M) | Medi trout |
| 19 | 19 | crate eel (E)(N | Emerald eel |
| 27 | 27 | crate seafood(N | Mixed seafood |
| 30 | 30 | crate books | Books (not food, included here as index) |

### Liquid Barrels

| Good | Item | Name | Contents |
|:----:|:----:|------|----------|
| 10 | 10 | barrel water | Water (LiquidType 1) |
| 11 | 11 | barrel rum | Rum (LiquidType 2) |
| 12 | 12 | barrel beer | Beer (LiquidType 6/7/8) |
| 13 | 13 | barrel wine | Wine (LiquidType 3) |
| 24 | 24 | barrel spices(N | Spices |

### Mineral/Material Crates

| Good | Item | Name | Notes |
|:----:|:----:|------|-------|
| 20 | 20 | crate gems | Gems |
| 21 | 21 | crate iron | Iron |
| 22 | 22 | crate gold | Gold |
| 23 | 23 | crate copper | Copper |
| 25 | 25 | crate grain | Grain |
| 26 | 26 | crate medicine | Medicine |
| 28 | 28 | crate silk | Silk |
| 29 | 29 | crate generic goods | Generic/unspecified goods |

---

## Goods 31–64: Bulk Trade Goods

*(Item indices = goodIndex + 170)*

| Good | Item | Name | Category |
|:----:|:----:|------|----------|
| 31 | 201 | crate venison (good) | bulkFood |
| 32 | 202 | crate truffles (good) | bulkFood |
| 33 | 203 | tools | bulkGood |
| 34 | 204 | crate sculptures | bulkGood |
| 35 | 205 | logs | bulkGood |
| 36 | 206 | barrel mead(N | bulkAlco |
| 37 | 207 | tobacco white (big)(N | bulkGood |
| 38 | 208 | tobacco green (big)(N | bulkGood |
| 39 | 209 | tobacco black (big)(N | bulkGood |
| 40 | 210 | tobacco brown (big)(N | bulkGood |
| 41 | 211 | tobacco blue (big) | bulkGood |
| 42 | 212 | crate rice | bulkFood |
| 43 | 213 | crate oranges | bulkFood |
| 44 | 214 | crate forest mushrooms | bulkFood |
| 46 | 216 | crate cave mushrooms | bulkFood |
| 47 | 217 | lumber | bulkGood |
| 48 | 218 | nails | bulkGood |
| 49 | 219 | leather(N | bulkGood |
| 50 | 220 | rabbit furs(N | bulkGood |
| 51 | 221 | mail | mission |
| 52 | 222 | wool | bulkGood |
| 53 | 223 | olive oil | bulkFood |
| 54 | 224 | apples | bulkFood |
| 55 | 225 | marble | bulkGood |
| 56 | 226 | silver | bulkGood |
| 57 | 227 | sulfur | bulkGood |
| 58 | 228 | barrel cider | bulkAlco |
| 59 | 229 | hemp | bulkGood |
| 60 | 230 | dyes | bulkGood |
| 61 | 231 | rubber | bulkGood |
| 62 | 232 | coffee | bulkFood |
| 63 | 233 | salt | bulkGood |
| 64 | 234 | saffron(N | bulkGood |

---

## Cargo Items (Non-Trade)

These are items that function as containers/cargo but aren't necessarily in the Good system.

| Prefab Index | Name | Type |
|:-----------:|------|------|
| 40 | empty bottle | Container |
| 104 | crate of fishing hooks | Cargo |
| 108 | crate of firewood | Cargo |
| 131 | lantern candle crate(N | Cargo |
| 311 | crate of tobacco white | Cargo |
| 313 | crate of tobacco green | Cargo |
| 315 | crate of tobacco black | Cargo |
| 317 | crate of tobacco brown | Cargo |
| 319 | crate of tobacco blue | Cargo |
| 385 | salt barrel | Container |
| 386 | coffee barrel | Container |

---

## Crate System

### ShipItemCrate

Crates are sealed containers that spawn contents when unsealed.

| Property | Description |
|----------|-------------|
| `containedPrefab` | GameObject prefab spawned on unseal |
| `smokedFood` | Contents pre-smoked (for food crates) |
| `amount` | Number of items inside |
| `mass` | Base crate mass + `containedPrefab.mass × amount` |

### Unsealing Process
```
UnsealCrate():
  for i in 0..amount:
    Instantiate(containedPrefab, position + up*0.5, rotation)
    RegisterToSave()
    if CookableFood: optionally smoke
  DestroyItem()
```

### CrateInventory

Manages sealed crate UI. `CrateSealUI` handles the seal interaction.

---

## Bulk vs Retail

`ShipItem.IsBulk()` returns true for these categories:
- `bulkAlco`
- `bulkFood`
- `bulkWater`
- `bulkGood`

Bulk items typically go in cargo hold (cargo carrier) rather than personal inventory.

---

## Mission Goods

When a `Good` is registered to a mission via `RegisterToMission(assignedMission, missionDueDay)`:
- `missionIndex` is set
- `SaveablePrefab.RegisterToSave()` called
- `ShipItem.RegisterMissionGood()` → sets `sold = true`, enables outline
- `UpdateLookText()` shows: `"{name}\nto {port}\ndue: {dueText}"`

On delivery: `Deliver()` → untag, notify mission, `DestroyItem()`.

Mail (good 51 / item 221) is a special mission-only trade good.

---

## Regional Origins

Goods have `nativeRegion` of type `PortRegion`:
- Aestrin goods: sunspot fish, templefish, tuna
- Emerald goods: salmon, eel, shimmertail, bananas
- Medi goods: trout, north fish, lamb, cheese, goat cheese
- Swamp goods: snapper, bubbler
- Some goods may be universal (tools, generic goods)

---

## Economy Integration

`IslandEconomy` manages per-island supply/demand. Prices fluctuate based on:
- Distance from native region
- Island market conditions
- Player reputation (`requiredRepLevel`)

See note 13 for detailed economy mechanics.

---

*Extracted from Sailwind v0.38 decompilation.*
