# 26. Cooking: Stove, Fuel, Boiling, and Smoking

Analysis of food preparation subsystem. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Food spoilage/conservation (drying/salting) described in note 23 (`FoodState`); here — thermal processing.

## Cooking Chain

```
StoveFuel (fuel, health=200)
   └─ burning: health -= dt*timescale*100; stove.AddHeat(burnRate)
        └─ ShipItemStove.AddHeat → each slot: CookableFood.Cook(heat*cookEfficiency, smoker)
             └─ item.amount (cooking) and currentHeat grow; smoker → foodState.AddSmoked
```

## Stove (`ShipItemStove : ShipItem`)

| Field | Default | Content |
|------|:--:|-----------|
| `cookEfficiency` | 0.4 | Stove efficiency (share of heat going to cooking). |
| `smoker` | false | Smoking mode (adds smoking, doesn't cook soup). |
| `slots` (`StoveCookTrigger[]`) | — | Food slots (several dishes simultaneously). |
| `currentHeat` | 0 | Current stove heat. |
| `fuelTrigger` | — | Fuel receptacle. |

### Interaction (`OnItemClick` on stove)
- If holding **fuel** (`StoveFuel`) → refuel in `fuelTrigger`.
- If holding **food** (`CookableFood`) → place in free slot (`GetFreeSlot`). **Smoker doesn't accept soup** (`CookableFoodSoup`).

### Heating (`AddHeat`)
For each occupied slot: `currentFood.Cook(heat * cookEfficiency, smoker)`. Sizzle sound (`audio.volume`) grows with number of cooking dishes (up to `0.777 * count`).

## Fuel (`StoveFuel`)

| Field | Default | Content |
|------|:--:|-----------|
| `initialFuel` | 200 | Fuel reserve → `item.health`. |
| `lit` | false | Whether burning. |
| `inserted` | — | Whether inserted into stove. |

- **Insertion:** in `StoveFuelTrigger` (by trigger or click).
- **Burning** (`LightUp` → `lit = true`): while `health > 0`:
  - `health -= deltaTime * Sun.sun.timescale * 100` (fuel consumption, depends on time acceleration);
  - `cookTrigger.stove.AddHeat(same amount)` — heat transfer to stove.
- When fuel runs out (`health <= 0`) → `UnregisterBurntFuel()` + `DestroyItem()` (fuel disappears).

> Fuel is a consumable item: 200 health units burn at rate `100 * timescale` per second.

## Food Cooking (`CookableFood`)

| Field | Content |
|------|-----------|
| `currentHeat` (0..2) | Dish's own heat. |
| `cookRate` (1) | Cooling rate. |
| Materials | `rawMaterial`, `cookedMaterial`, `burntMaterial`, `spoiledMaterial`, `driedMaterial`. |

### `Cook(addedHeat, smoking)`
```
energyPerBite = itemFood.GetEnergyPerBite()
num = addedHeat * 0.066 / energyPerBite
item.amount  += num * currentHeat * 0.5     // "cooking" (cap 2.2)
currentHeat  += num * 10                     // heating (cap 2)
if smoking → foodState.AddSmoked(num * currentHeat * 0.5)
```
Dense/caloric food (high `energyPerBite`) cooks slower.

### `item.amount` = Cooking Degree (visual and status)
| `amount` | State | Material |
|:--:|-----------|----------|
| `< 0.75` | raw (`raw`) | `rawMaterial` |
| `0.75–1.0` | raw→cooked transition | lerp raw→cooked |
| `1.0–1.5` | **cooked** (`cooked`) | `cookedMaterial` |
| `1.5–1.75` | cooked→burnt transition | lerp cooked→burnt |
| `>= 1.75` | **burnt** (`burnt`) | `burntMaterial` |
| `dried >= 0.99` | dried | `driedMaterial` |
| `spoiled >= 0.9` | spoiled | lerp → `spoiledMaterial` |

This aligns with `FoodState` prefixes (note 23): `"cooked"` at `amount>=1`, `"burnt"` at `amount>=1.5`.

### Cooling and Particles
- Outside stove: `currentHeat -= 50 * dt * cookRate * 5e-5` (slow cooling).
- **Steam** at `currentHeat >= 0.5`: emission lerp 12→24, wind displaces particles (`Wind.currentWind * 0.015`).
- **Green rot particles** at `spoiled > 0.9` (color RGB ≈ 0.27/0.42/0.27).

### Shop Cooking
- `CookInShop()`: `currentHeat = 1.25`, `amount = 1.2` (instantly cooked).
- `SmokeInShop()`: `amount = 1.01` (smoked).

## Triggers
- **`StoveCookTrigger`** (food slot): `smoking`, `stove`, `currentFood`. `OnTriggerEnter` → if slot is free and stove not in hand → `CookableFood.InsertIntoCookTrigger`.
- **`StoveFuelTrigger`** (fuel receptacle): `InsertFuel(item)`, `UnregisterBurntFuel()`.
- `InsertIntoCookTrigger`: food attaches to slot (`attached`, `inStove`, `disableCol`), position fixed on stove (`UpdatePosition`). `TakeOutOfCooker` — removal.

## Practical Takeaways for Modding

1. **Cooking** = fuel (`StoveFuel.health`) → stove heat (`AddHeat`) → `CookableFood.Cook`. Stove efficiency `cookEfficiency = 0.4`.
2. **Cooking degree** stored in `ShipItem.amount`: `<0.75` raw, `1–1.5` cooked, `>1.75` burnt. Changing `amount` can instantly "cook"/"burn" food.
3. **Smoking** — via `smoker` stove (`AddSmoked` in `FoodState`); smoker doesn't cook soup.
4. **Fuel consumption** at rate `100 * timescale`/s from 200 units; time acceleration speeds consumption too.
5. **Food visuals** (raw/cooked/burnt/dried/spoiled materials) and particles (steam/rot) controlled by `amount`/`currentHeat`/`spoiled`.
6. Cooking state (`amount`, `currentHeat` via `health`/`amount`) saved via `SavePrefabData` (note 11); spoilage/drying/smoking — via `extraValue0..3`.
