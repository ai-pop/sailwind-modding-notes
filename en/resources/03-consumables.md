# 03. Consumables: Food, Drinks, Elixirs, Tobacco

> Complete catalog of all consumable items — food, drinks, elixirs, tobacco, and related cooking/preservation items.
> Cross-reference: notes 23 (Fishing/Food), 26 (Cooking), 25 (Sleep).

---

## Food Items (Raw/Cookable)

### Fish (Raw)

| Prefab Index | Name | Region | Slice Index | Notes |
|:-----------:|------|:------:|:-----------:|-------|
| 33 | salmon | Emerald | 351? | Common Emerald fish |
| 34 | eel | Emerald | 354 | Emerald eel |
| 35 | shimmertail | Emerald | 355 | Emerald shimmertail |
| 38 | blackfin hunter | Medi | 358 | Medi blackfin |
| 140 | gold albacore | — | 348 | Gold albacore |
| 141 | swamp snapper | Swamp | 353 | Swamp fish 1 |
| 142 | swamp bubbler | Swamp | 362 | Swamp fish 2 |
| 148 | swamp fish 3 | Swamp | — | Third swamp fish |
| — | templefish | Aestrin | 351 | Aestrin templefish |
| — | sunspot fish | Aestrin | 352 | Aestrin sunspot |
| — | trout | Medi | 356 | Medi trout |
| — | north fish | Medi | 357 | Medi north fish |
| — | tuna | Aestrin | 366 | Aestrin tuna |
| — | firefish | — | 365 | Firefish |

### Meat

| Prefab Index | Name | Slice Index |
|:-----------:|------|:-----------:|
| 39 | pork | 359 |
| 54 | lamb chop | 360 |
| 154 | chicken | 343 |
| 15 | sausages (crate) | 361 |

### Vegetables & Fruits

| Prefab Index | Name | Slice Index |
|:-----------:|------|:-----------:|
| 147 | orange | 349 |
| 149 | apple | 350 |
| 150 | carrot | 346 |
| 151 | cucumber | 347 |
| 152 | potato | 345 |
| 153 | sweet potato | 344 |
| 17 | bananas (crate) | 364 |

### Mushrooms

| Prefab Index | Name | Slice? |
|:-----------:|------|:------:|
| 144 | forest mushroom | — |
| 145 | field mushroom | — |
| 146 | cave mushroom | — |

### Bread & Grains

| Prefab Index | Name | Slice Index |
|:-----------:|------|:-----------:|
| 155 | baguette | 342 |
| 143 | rice cake | — |
| — | bread (generic) | 363 |
| — | bun | 367 |

### Cheese & Dairy

| Prefab Index | Name | Slice Index |
|:-----------:|------|:-----------:|
| 7 | cheese (crate) | 368 |
| 8 | goat cheese (crate) | 369 |

---

## Sliced Food Variants (342–369)

When a whole food item is sliced with a knife, it generates sliced variants. Each `FoodState` has `slicePrefabIndex` pointing to the corresponding slice.

| Prefab Index | Slice Name | From |
|:-----------:|------------|------|
| 342 | baguette slice | baguette |
| 343 | chicken slice | chicken |
| 344 | sweet potato slice | sweet potato |
| 345 | potato slice | potato |
| 346 | carrot slice | carrot |
| 347 | cucumber slice | cucumber |
| 348 | gold albacore slice | gold albacore |
| 349 | orange slice | orange |
| 350 | apple slice | apple |
| 351 | templefish slice | templefish |
| 352 | sunspot slice | sunspot fish |
| 353 | swamp snapper slice | swamp snapper |
| 354 | eel slice | eel |
| 355 | shimmertail slice | shimmertail |
| 356 | trout slice | trout |
| 357 | north fish slice | north fish |
| 358 | blackfin slice | blackfin hunter |
| 359 | pork slice | pork |
| 360 | lamb slice | lamb chop |
| 361 | sausage slice | sausage |
| 362 | bubbler slice | swamp bubbler |
| 363 | bread slice | bread |
| 364 | banana slice | banana |
| 365 | firefish slice | firefish |
| 366 | tuna slice | tuna |
| 367 | bun slice | bun |
| 368 | cheese slice | cheese |
| 369 | goat cheese slice | goat cheese |

---

## Food Preservation System

### FoodState Component

Each food item has `FoodState` with these preservation mechanics:

| Property | Range | Description |
|----------|:-----:|-------------|
| `dried` | 0–1 | Increases over time (×2 on drying rack). At 1.0 → "dried", prevents spoilage |
| `smoked` | 0–1 | Set by smoking rack. At 1.0 → "smoked", prevents spoilage |
| `salted` | 0–1 | Set by Salt item. At 1.0 → "salted", prevents spoilage |
| `spoiled` | 0–1 | Increases at rate `1/spoilDuration`, slowed by dried/smoked/salted. At >0.9 → "rotten" |

### Spoilage Formula
```
spoilage_rate = (Δt × timescale / spoilDuration) 
              × (1 - dried) 
              × (1 - smoked) 
              × InverseLerp(1, 0, currentHeat)
```

### Drying Racks

| Prefab Index | Name |
|:-----------:|------|
| 374 | drying rack A (Aestrin) |
| 375 | drying rack E (Emerald) |
| 376 | drying rack M (Medi) |

### Smoking Racks

| Prefab Index | Name |
|:-----------:|------|
| 377 | smoking rack A (Aestrin) |
| 378 | smoking rack E (Emerald) |
| 379 | smoking rack M (Medi) |

### Brining Jar

| Prefab Index | Name |
|:-----------:|------|
| 381 | brining jar small |

---

## Drinks & Liquids

### Liquid Types

| Index | Name | Hydration | Energy | Alcohol | Notes |
|:-----:|------|:---------:|:------:|:-------:|-------|
| 1 | water | +10 | 0 | 0 | Basic hydration |
| 2 | rum | +4 | 0 | 18 | Strong alcohol |
| 3 | wine | +6 | 0 | 12 | Moderate alcohol |
| 4 | coconut wine | +6 | 0 | 12 | Tropical alcohol |
| 5 | mead | +8 | 0 | 6 | Light alcohol |
| 6 | honey beer | +8 | 0 | 6 | Light alcohol |
| 7 | rice beer | +8 | 0 | 6 | Light alcohol |
| 8 | cider | +8 | 0 | 6 | Light alcohol |
| 9 | sea water | -10 | 0 | 0 | DEHYDRATES! |
| 10 | coffee | +10 | +10 | 0 | Energy boost |
| 11 | black tea | +10 | +4 | 0 | Mild energy |
| 12 | green tea | +10 | +6 | 0 | Moderate energy |
| 13 | white tea | +10 | -4 | 0 | Energy drain |

### Alcohol Formula
```
alcohol = (10 - hydration) × 3
```
So rum (hydration=4) → alcohol=18, wine (hydration=6) → alcohol=12, etc.

### Tea Types

| Prefab Index | Name | Tea Type | Notes |
|:-----------:|------|:--------:|-------|
| 387 | tea box white | white tea (13) | Energy: -4 |
| 388 | tea box black | black tea (11) | Energy: +4 |
| 389 | tea box green | green tea (12) | Energy: +6 |

### Coffee

| Prefab Index | Name |
|:-----------:|------|
| 373 | coffee box |
| 386 | coffee barrel |

### Liquid Containers

Container naming based on `ShipItemBottle.capacity`:
- `capacity == 9` → **bucket**
- `capacity < 5` → **mug**
- `capacity < 30` → **bottle**
- `capacity >= 30` → **barrel**

| Prefab Index | Container Type | Notes |
|:-----------:|----------------|-------|
| 40 | empty bottle | Generic |
| 55 | water bottle | Pre-filled |
| 56 | coco wine | Coconut wine |
| 57 | honey beer | |
| 58 | rice beer | |
| 70 | bucket | capacity=9 |
| 100 | mug wood | capacity<5 |
| 101 | mug clay | |
| 102 | mug metal | |
| 103 | mug metal gold | |

---

## Elixirs

| Prefab Index | Name | Sleep | Water | Food | Notes |
|:-----------:|------|:-----:|:-----:|:----:|-------|
| 96 | elixir of energy | ? | ? | +? | Restores food |
| 97 | elixir of sleep | +? | ? | ? | Restores sleep |
| 98 | elixir of random | random | random | random | Random effects |

---

## Tobacco & Pipes

### Pipes

| Prefab Index | Name | Region |
|:-----------:|------|:------:|
| 300 | pipe A | Aestrin |
| 301 | pipe E | Emerald |
| 302 | pipe M | Medi |

### Tobacco Varieties

| Prefab Index | Name | Crate Version |
|:-----------:|------|:------------:|
| 310 | tobacco white | 311 |
| 312 | tobacco green | 313 |
| 314 | tobacco black | 315 |
| 316 | tobacco brown | 317 |
| 318 | tobacco blue | 319 |

### Bulk Tobacco (Large format, Good indices 37–41)

| Good Index | Item Index | Name |
|:----------:|:----------:|------|
| 37 | 207 | tobacco white (big)(N |
| 38 | 208 | tobacco green (big)(N |
| 39 | 209 | tobacco black (big)(N |
| 40 | 210 | tobacco brown (big)(N |
| 41 | 211 | tobacco blue (big) |

---

## Cooking Equipment

| Prefab Index | Name | Type |
|:-----------:|------|------|
| 105 | stove_small_M | Small stove (Medi) |
| 106 | stove_small_E | Small stove (Emerald) |
| 107 | stove_small_A | Small stove (Aestrin) |
| 108 | crate of firewood | Stove fuel |
| 157 | pot big | Large cooking pot |
| 382 | kettle A | Kettle (Aestrin) |
| 383 | kettle E | Kettle (Emerald) |
| 384 | kettle M | Kettle (Medi) |
| 385 | salt barrel | Salt storage |
| 233 (good 63) | salt | Bulk salt trade good |

---

## Scrolls

| Prefab Index | Name | Value | Notes |
|:-----------:|------|:-----:|-------|
| 91 | scroll (generic) | ? | Generic scroll |
| 380 | scroll nutrition | ? | Food nutrition info |

**Value logic:** `amount <= 2` (tutorial scrolls) → value=30, `amount > 2` → value=120.

---

*All data from Sailwind v0.38 decompilation + asset extraction.*
