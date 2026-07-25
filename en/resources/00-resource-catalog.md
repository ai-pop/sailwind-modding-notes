# Resource Catalog: Items, Prefabs, and Game Objects

> **Sailwind Modding Notes — Resources Section**
> Extracted from decompiled `Assembly-CSharp.dll` (v0.38) and `Sailwind_Data` asset files.
> Complements notes 16 (Item Framework), 45 (Crate/Cargo), 61 (Mass Table), 23 (Fishing/Food), 26 (Cooking).

---

## Overview

This is the master index for all game resources documented in the `resources/` section. Each sub-document covers a specific category of items, prefabs, or game objects found in Sailwind.

### Key Systems

| System | Description | See |
|--------|-------------|-----|
| `PrefabsDirectory` | Singleton catalog of ALL prefabs. `directory[]` — GameObject array by index. `shipItems[]` — parallel ShipItem component array. | [01-prefab-registry](01-prefab-registry.md) |
| `ShipItem` | Base class for all items. 35+ subclasses with specialized behavior. | [02-item-classes](02-item-classes.md) |
| `Good` | Component attached to tradeable items. Links to `PortRegion`, missions. 65 goods total (`Refs.goodCount = 65`). | [07-trade-goods](07-trade-goods.md) |
| `SaveablePrefab` | Persistence component on every item. `prefabIndex`, `instanceId`, parent/position/rotation save data. | Note 11 |
| `ItemRigidbody` | Physics twin for every ShipItem. Handles buoyancy, inventory, boat transitions. | Note 44 |

### Prefab Index Mapping

```
Good index → Item index:
  goods 0–30  → items 0–30
  goods 31–64 → items 201–234 (offset +170)

PrefabsDirectory.GoodToItemIndex(goodIndex)
PrefabsDirectory.ItemToGoodIndex(itemIndex)
```

### Total Counts

| Category | Count |
|----------|:----:|
| Total prefabs in directory | ~400 |
| ShipItem subclasses | 35+ |
| Trade goods (Good components) | 65 |
| Sail prefabs | ~100 |
| Liquid types | 14 |
| Transaction categories | 15 |
| Paintings | ~30 |
| Food/fish types | ~25 |

---

## Document Index

| # | Document | Contents |
|:--|----------|----------|
| 01 | [Prefab Registry](01-prefab-registry.md) | Complete prefab index → name mapping |
| 02 | [Item Classes](02-item-classes.md) | ShipItem subclass hierarchy, properties, behaviors |
| 03 | [Consumables](03-consumables.md) | Food, drinks, elixirs, tobacco, piping |
| 04 | [Equipment & Tools](04-equipment.md) | Navigation, tools, fishing, lighting, lamps |
| 05 | [Furniture & Decor](05-furniture-decor.md) | Beds, tables, chairs, shelves, paintings, flower pots |
| 06 | [Sails](06-sails.md) | All sail prefabs by region (Aestrin/Medi/Emerald/Large) |
| 07 | [Trade Goods](07-trade-goods.md) | Bulk cargo, crates, barrels, regional goods |
| 08 | [Data Tables](08-data-tables.md) | Liquids, categories, food states, fish list |

---

## Quick Reference

### Transaction Categories (`TransactionCategory`)

| Enum | Description |
|------|-------------|
| `retailFood` | Retail food items |
| `retailWater` | Retail water/drinks |
| `retailAlco` | Retail alcohol |
| `toolsAndSupplies` | Tools (hammer, oakum, etc.) |
| `furniture` | Furniture |
| `otherItems` | Miscellaneous |
| `bulkFood` | Bulk food cargo |
| `bulkWater` | Bulk water/liquid cargo |
| `bulkAlco` | Bulk alcohol cargo |
| `bulkGood` | Bulk dry goods cargo |
| `mission` | Mission items |
| `boat` | Boat purchases |
| `recovery` | Recovery items |
| `other` | Uncategorized |
| `currencyExchange` | Currency exchange |

### Liquid Types (`LiquidType`)

| Index | Name | Hydration | Energy | Alcohol |
|:-----:|------|:---------:|:------:|:-------:|
| 0 | none | 0 | 0 | 0 |
| 1 | water | 10 | 0 | 0 |
| 2 | rum | 4 | 0 | 18 |
| 3 | wine | 6 | 0 | 12 |
| 4 | coconut wine | 6 | 0 | 12 |
| 5 | mead | 8 | 0 | 6 |
| 6 | honey beer | 8 | 0 | 6 |
| 7 | rice beer | 8 | 0 | 6 |
| 8 | cider | 8 | 0 | 6 |
| 9 | sea water | -10 | 0 | 0 |
| 10 | coffee | 10 | 10 | 0 |
| 11 | black tea | 10 | 4 | 0 |
| 12 | green tea | 10 | 6 | 0 |
| 13 | white tea | 10 | -4 | 0 |

### Food States

Food items use `FoodState` component with preservation mechanics:
- **dried** (0–1): Increases over time, faster on drying rack. Slows spoilage.
- **smoked** (0–1): From smoking rack. Prevents spoilage.
- **salted** (0–1): From Salt item interaction. Prevents spoilage.
- **spoiled** (0–1): Increases proportional to `1/spoilDuration`, slowed by dried/smoked/salted.

Food labels: `"rotten"`, `"burnt"`, `"salted smoked"`, `"salted"`, `"smoked"`, `"dried"`, `"cooked"`, `"fresh raw"`, `"raw"`, `"fresh"`.

---

## Architecture Notes

### PrefabsDirectory Flow
```
Start()
  ├── instance = this
  ├── PopulateShipItems()  → shipItems[i] = directory[i].GetComponent<ShipItem>()
  └── Validate: each prefab must have SaveablePrefab with matching prefabIndex
```

### Item Lifecycle
```
Spawn → RegisterToSave() → (sold/purchased) → EnterBoat/ExitBoat → 
  → SaveLoadManager tracks → DestroyItem() on -2 parent or cargo loss (-3)
```

### Container Rules (Bottle sizes)
Based on `ShipItemBottle.UpdateLookText()`:
- `capacity == 9` → "bucket"
- `capacity < 5` → "mug"  
- `capacity < 30` → "bottle"
- `capacity >= 30` → "barrel"

---

*Generated from Sailwind v0.38 decompilation + asset analysis. Some prefab names are approximate (extracted from binary assets).*
