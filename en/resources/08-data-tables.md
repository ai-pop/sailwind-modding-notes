# 08. Data Tables — Reference

> Quick-reference tables for all enum values, constants, and mappings used by the item/resource system.

---

## TransactionCategory (enum)

| Value | Name | Use |
|:-----:|------|-----|
| 0 | retailFood | Retail food (market stalls) |
| 1 | retailWater | Retail drinks |
| 2 | retailAlco | Retail alcohol |
| 3 | toolsAndSupplies | Tools, repair items |
| 4 | furniture | Beds, tables, chairs, shelves |
| 5 | otherItems | Miscellaneous items |
| 6 | bulkFood | Bulk food cargo |
| 7 | bulkWater | Bulk liquid cargo |
| 8 | bulkAlco | Bulk alcohol cargo |
| 9 | bulkGood | Bulk dry goods cargo |
| 10 | mission | Mission items |
| 11 | boat | Boat purchases |
| 12 | recovery | Recovery/insurance items |
| 13 | other | Uncategorized |
| 14 | currencyExchange | Currency exchange |

`ShipItem.IsBulk()` = true for categories 6, 7, 8, 9.

---

## LiquidType (enum)

| Value | Name | Display Name | Hydration | Energy | Alcohol |
|:-----:|------|-------------|:---------:|:------:|:-------:|
| 0 | none | *(empty)* | 0 | 0 | 0 |
| 1 | water | water | +10 | 0 | 0 |
| 2 | rum | rum | +4 | 0 | 18 |
| 3 | wine | wine | +6 | 0 | 12 |
| 4 | cocoWine | coconut wine | +6 | 0 | 12 |
| 5 | mead | mead | +8 | 0 | 6 |
| 6 | honeyBeer | honey beer | +8 | 0 | 6 |
| 7 | riceBeer | rice beer | +8 | 0 | 6 |
| 8 | cider | cider | +8 | 0 | 6 |
| 9 | seaWater | sea water | **-10** | 0 | 0 |
| 10 | coffee | coffee | +10 | +10 | 0 |
| 11 | blackTea | black tea | +10 | +4 | 0 |
| 12 | greenTea | green tea | +10 | +6 | 0 |
| 13 | whiteTea | white tea | +10 | **-4** | 0 |

### Alcohol Calculation
```
alcohol = (10 - hydration) × 3
```

| Hydration | Alcohol |
|:---------:|:-------:|
| 4 (rum) | 18 |
| 6 (wine, coco wine) | 12 |
| 8 (mead, beers, cider) | 6 |
| 10 (water, tea, coffee) | 0 |
| -10 (sea water) | 0 (special case) |

---

## Food Label Generation (FoodState)

Label order in `UpdateLookText()`:

| Condition | Label |
|-----------|-------|
| `spoiled > 0.9` | "rotten {name}" |
| `amount >= 1.5` (burnt) | "burnt {name}" |
| `salted >= 0.99 && smoked >= 0.99` | "salted smoked {name}" |
| `salted >= 0.99` | "salted {name}" |
| `smoked >= 0.99` | "smoked {name}" |
| `dried >= 0.99` | "dried {name}" |
| `amount >= 1.0` (cooked) | "cooked {name}" |
| `ShowRaw() && spoiled < 0.2` | "fresh raw {name}" |
| `ShowRaw()` | "raw {name}" |
| `spoiled < 0.2` | "fresh {name}" |
| *(default)* | "{name}" |

---

## Food Energy System

### ShipItemFood
- `energyPerBite` — base energy per bite (default 10)
- `rawEnergyMult` — multiplier for raw food (default 0.25, so raw = 25% energy)
- 3 bites per food item (`health` starts at 3, decreases by 1 per bite)
- 3 `eatenMeshes[]` for progressive visual

### ShipItemSoup
- `currentEnergy` — cooked energy available
- `currentUncookedEnergy` — uncooked energy remaining
- Both contribute to mass: `mass += water + energy/20 + uncooked/20`

---

## Container Size Classes

Based on `ShipItemBottle.capacity`:

| Capacity | Name | Example |
|:--------:|------|---------|
| < 5 | mug | mug wood (100), mug clay (101), mug metal (102) |
| 9 | bucket | bucket (70) |
| 10–29 | bottle | water bottle (55), wine, beer bottles |
| ≥ 30 | barrel | barrel water (10), barrel rum (11) |

---

## Save Prefab Parent Object Codes

| Code | Meaning |
|:----:|---------|
| `-1` | In world (no boat parent) |
| `-2` | Marked for destruction |
| `-3` | Lost (cargo loss during blackout), layer=IgnoreRaycast |
| `≥ 0` | Parent object scene index (boat, island, etc.) |

---

## Save Prefab Extra Values

`SavePrefabData` stores 5 `extraValue` floats plus `itemHealth` and `itemAmount` for subclass state:

| Subclass | itemHealth | itemAmount | extra0 | extra1 | extra2 | extra3 | extra4 |
|----------|:----------:|:----------:|:------:|:------:|:------:|:------:|:------:|
| FoodState | — | — | dried | smoked | salted | spoiled | — |
| ShipItemSoup | currentWater | currentEnergy | uncookedEnergy | spoiled | salted | vitamins | protein |
| ShipItemKettle | currentWater | teaType (cast) | teaAmount | cookedTeaAmount | — | — | — |
| ShipItemFoldable | — | — | — | — | — | — | chartData |

---

## Inventory Slot Conventions

| Slot Range | Location |
|:----------:|----------|
| 0–99 | Personal inventory (`GPButtonInventorySlot.inventorySlots[slotIndex]`) |
| 100+ | Cargo carrier (`portIndex + 100`) → `CargoCarrier.carriers[portIndex]` |

---

## Refs Constants

| Constant | Value |
|----------|:-----:|
| `Refs.goodCount` | 65 |
| `Refs.portCount` | 34 |

---

## Region Codes in Prefab Names

| Code | Region |
|:----:|--------|
| A | Aestrin |
| M | Medi |
| E | Emerald |
| L | Large / Lagoon |
| Am | Aestrin-modified |
| Em | Emerald-modified |

---

## ShipItem Field Defaults

| Field | Default | Notes |
|-------|:-------:|-------|
| `mass` | 1.0 | kg |
| `value` | — | Set per prefab in inspector |
| `floaterHeight` | 1.6 | Buoyancy height above water |
| `inventoryScale` | 1.0 | Scale in inventory slot |
| `inventoryRotation` | 0 | Y-axis rotation in inventory |
| `inventoryRotationX` | 0 | X-axis rotation in inventory |
| `wallAttachment` | false | Nail to wall ability |
| `delayLook` | false | Delay look text |
| `sold` | false | Purchased state |
| `nailed` | false | Secured state |
| `health` | — | Per-item meaning |
| `amount` | — | Per-item meaning |
| `daysInStorage` | 0 | Spoilage tracking |

---

## Scenes / Islands

From `Sailwind_Data` level files and strings:
- Island scenes: level0 through level42
- Named islands include references in `sharedassets*`:
  - Lagoon Fisherman (level31)
  - Various regional islands for Aestrin, Medi, Emerald

See note 19 for complete world/port/region reference.

---

*All data from Sailwind v0.38 decompilation + asset analysis.*
