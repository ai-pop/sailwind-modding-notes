# 02. ShipItem Class Hierarchy

> Complete reference of all `ShipItem` subclasses, their C# fields, behaviors, and interactions.
> Extends note 16 (Item Framework).

---

## Inheritance Chain

```
MonoBehaviour
  └─ GoPointerButton          (interactive object — look, activate, alt-activate)
       └─ PickupableItem       (pickup/drop/hold mechanics)
            └─ ShipItem        (base item — mass, value, physics, save)
                 └─ ShipItemHangable  (hangable items — lamps, etc.)
                      └─ ShipItemLight  (light sources)
                 └─ 30+ other subclasses
```

---

## Base Class: `ShipItem`

### Serialized Fields (set in Unity Inspector per prefab)

| Field | Type | Default | Description |
|-------|------|:-------:|-------------|
| `wallAttachment` | bool | false | Can be nailed to walls via raycast |
| `delayLook` | bool | false | Delay look text update |
| `mass` | float | 1.0 | Item mass in kg (affects BoatMass) |
| `value` | int | — | Base price for economy |
| `name` | string | — | Display name |
| `category` | TransactionCategory | — | Transaction category |
| `inventoryScale` | float | 1.0 | Scale in inventory slot |
| `inventoryRotation` | float | 0 | Y rotation in inventory |
| `inventoryRotationX` | float | 0 | X rotation in inventory |
| `floaterHeight` | float | 1.6 | Buoyancy raise height above water |
| `itemRigidbody` | Transform | — | Physics twin (serialized ref) |

### Runtime State

| Field | Type | Description |
|-------|------|-------------|
| `sold` | bool | Purchased from shop (false=still on shelf) |
| `nailed` | bool | Nailed/secured in place |
| `health` | float | Durability / liquid volume / food bites |
| `amount` | float | Quantity / stack / fill level |
| `daysInStorage` | int | Days stored (affects spoilage) |
| `itemRigidbodyC` | ItemRigidbody | Cached physics twin component |
| `currentWalkCol` | Transform | Current boat walk collider |
| `currentActualBoat` | Transform | Current boat parent |

### Key Virtual Methods

| Method | When Called | Override Purpose |
|--------|-------------|------------------|
| `OnLoad()` | After Awake + 1 frame | Init item state from save data |
| `OnBuy()` | After purchase | Post-purchase setup |
| `OnPickup()` | Picked up by player | Clear attachments, inventory |
| `OnDrop()` | Dropped by player | Wall attach, return to shop |
| `OnEnterInventory()` | Placed in inventory slot | Disconnect joints, exit boat |
| `OnLeaveInventory()` | Removed from inventory | Re-enable physics |
| `UpdateLookText()` | Every frame when looked at | Set `lookText`/`description` |
| `OnAltActivate()` | Right-click / Alt button | Special action (sell, sleep, use) |
| `OnScroll(float)` | Mouse wheel | Rotate held item, zoom spyglass |
| `ExtraFixedUpdate()` | FixedUpdate | Per-item physics logic |
| `ExtraLateUpdate()` | LateUpdate | Per-item visual updates |
| `AllowOnItemClick()` | Check if other item can interact | Item→item interactions |
| `OnItemClick()` | Item clicked with held item | Item→item interaction |

---

## Subclass Reference

### Food & Drink

#### `ShipItemFood` — Edible Food
| Field | Type | Description |
|-------|------|-------------|
| `eatenMeshes[]` | Mesh[] | Progressive eaten meshes (3 states) |
| `energyPerBite` | float | Energy per bite (default 10) |
| `rawEnergyMult` | float | Raw food energy multiplier (default 0.25) |
| `protein` | float | Protein content |
| `vitamins` | float | Vitamin content |

**Behavior:** On alt-activate while held — eat one bite. Each bite reduces `health` by 1 (3 bites total for most foods). Cooked/raw/salted/smoked/dried states affect energy. Uses `FoodState` for spoilage/preservation. Can be salted with `ShipItemSalt` click.

#### `ShipItemSoup` — Soup/Pot Food
| Field | Type | Description |
|-------|------|-------------|
| `capacity` | float | Max water capacity (default 20) |
| `currentWater` | float | Current water volume |
| `currentEnergy` | float | Cooked energy (nutrition) |
| `currentUncookedEnergy` | float | Uncooked energy remaining |
| `currentSpoiled` | float | Spoilage amount |
| `currentVitamins` | float | Vitamin content |
| `currentProtein` | float | Protein content |
| `currentSalted` | float | Salt level |

**Save data:** Uses `extraValue0–4` for state beyond health/amount.

#### `ShipItemKettle` — Tea Kettle
| Field | Type | Description |
|-------|------|-------------|
| `capacity` | float | Max water (default 10) |
| `brewSpeed` | float | Tea brewing speed |
| `currentWater` | float | Water volume |
| `currentTeaAmount` | float | Tea leaves added |
| `currentCookedTeaAmount` | float | Brewed tea strength |
| `currentTeaType` | LiquidType | Type (white/black/green) |

**Behavior:** Accepts `ShipItemTea` clicks to add tea. Cooking brews tea. Drinking restores hydration + energy based on tea type.

#### `ShipItemBottle` — Liquid Container
| Field | Type | Description |
|-------|------|-------------|
| `capacity` | float | Max volume (9=bucket, <5=mug, <30=bottle, ≥30=barrel) |

**Behavior:** `health` = fill level. `amount` = liquid type index (see `LiquidType`). Right-click to drink. Can be poured between bottles/soup/kettle. Container name: `capacity==9` → "bucket", `<5` → "mug", `<30` → "bottle", `≥30` → "barrel".

#### `ShipItemTea` — Tea Leaves
| Field | Type | Description |
|-------|------|-------------|
| `teaType` | LiquidType | White/black/green tea type |

**Behavior:** Click on kettle to add tea. `amount * 0.1` added to mass.

#### `ShipItemSalt` — Salt
| Field | Type | Description |
|-------|------|-------------|
| `maxSalt` | float | Max salt capacity (default 100) |

**Behavior:** Click on food to salt it. `amount / maxSalt * 100` = salt %. Prevents spoilage.

---

### Cooking

#### `ShipItemStove` — Cooking Stove
Extends stove interaction. Holds fuel (`ShipItemStoveFuel`). Has cooking positions (`StoveCookTrigger`).

#### `ShipItemStoveFuel` — Stove Fuel
Fuel for stove. Burn duration based on `health`. Can be firewood or other fuel types.

#### `ShipItemKnife` — Slicing Knife
Slices food into sliced variants. `KnifeCollider` handles cutting.

---

### Tools

#### `ShipItemHammer` — Repair Hammer
Used for hull repair (nailing planks) and general construction.

#### `ShipItemBroom` — Cleaning Broom
| Field | Type | Description |
|-------|------|-------------|
| `cleaner` | Cleaner | Reference to cleaner component |

**Behavior:** Alt-activate sweeps. `cleaner.activated = true`. Disables red outline while sweeping.

#### `ShipItemOakum` — Caulking Oakum
| Field | Type | Description |
|-------|------|-------------|
| `maxAmount` | float | Max oakum amount |

**Behavior:** Alt-activate near damaged boat to apply oakum to hull. `amount -= num`, `boatDamage.oakum += num`.

#### `ShipItemOar` — Oar
Manual propulsion. No special fields in decompilation (likely Unity-driven).

---

### Navigation

#### `ShipItemCompass` — Compass (including Sun Compass)
| Field | Type | Description |
|-------|------|-------------|
| `sunCompassSundial` | Transform | Sundial part (sun compass variant) |
| `chronoLatitude` | ChronometerLatitude | Latitude chronometer attachment |
| `lockX/Y/Z` | bool | Lock rotation axes when held |
| `sharpenShadow` | bool | Sun compass sharp shadow mode |

**Behavior:** Sun compass variant sets `GameState.holdingSunCompass`. Scroll adjusts chronometer latitude.

#### `ShipItemClock` — Chronometer/Clock
| Field | Type | Description |
|-------|------|-------------|
| `minuteHand` | Transform | Minute hand transform |
| `hourHand` | Transform | Hour hand transform |
| `lid` | Transform | Optional lid for opening |

**Behavior:** Hands rotate based on `Sun.sun.globalTime`. Alt-activate toggles lid open/close.

#### `ShipItemQuadrant` — Quadrant
| Field | Type | Description |
|-------|------|-------------|
| `dial` | Transform | Angle dial |
| `rotatingParent` | Transform | Rotating part |
| `lockX/Y/Z` | bool | Lock axes |

**Behavior:** Alt-activate toggles inspection mode. Rotates for angle measurement.

#### `ShipItemSpyglass` — Spyglass/Telescope
| Field | Type | Description |
|-------|------|-------------|
| `currentZoom` | float | Current zoom level |
| `movingParts[]` | Transform[] | Extending tube parts |
| `cam` | Camera | Zoom camera |
| `minZoom` / `maxZoom` | float | Zoom range |

**Behavior:** Scroll to zoom in/out. Camera enables on hold.

#### `ShipItemChipLog` — Chip Log (Speed Measurement)
| Field | Type | Description |
|-------|------|-------------|
| `bobberJoint` | ConfigurableJoint | Bobber connection |
| `maxLength` | float | Max rope length (default 40) |
| `bobberForceMult` | float | Bobber force multiplier |

**Behavior:** Throwing mechanic with rope. Measures speed via bobber drag.

---

### Lighting

#### `ShipItemLight : ShipItemHangable` — Lantern/Light
| Field | Type | Description |
|-------|------|-------------|
| `on` | bool | Light state |
| `usesOil` | bool | Uses oil fuel (vs candle) |
| `fuelConsumptionRate` | float | Fuel burn rate |
| `light` | Light | Unity Light component |
| `paperRenderer` | Renderer | Paper/lantern material |
| `paperOffMat` | Material | Off-state material |
| `particles` | ParticleSystem | Flame particles |

**Behavior:** `amount >= 1` → light on. Fuel from `ShipItemLanternFuel`. Hangable on `ShipItemLampHook`.

#### `ShipItemLanternFuel` — Lantern Fuel
| Field | Type | Description |
|-------|------|-------------|
| `oilBottle` | bool | Is oil bottle (vs candle) |
| `initialHealth` | float | Initial fuel amount |

**Behavior:** Click on lantern to refuel. `RequestOil(amount)` transfers fuel.

#### `ShipItemLampHook` — Lamp Hook
| Field | Type | Description |
|-------|------|-------------|
| `occupied` | bool | Has hanging lamp |

**Behavior:** Click with `ShipItemHangable` to hang. Creates `ConfigurableJoint`.

#### `ShipItemHangable` — Hangable Base
Base class for items that can hang on hooks. Disconnects joint on pickup/enter inventory.

---

### Rest & Habits

#### `ShipItemBed` — Bed
| Field | Type | Description |
|-------|------|-------------|
| `sleepPos` | Transform | Sleep position |
| `wakePos` | Transform | Wake position |

**Behavior:** Alt-activate → `Sleep.instance.EnterBed(transform)`.

#### `ShipItemPipe` — Smoking Pipe
Used with `ShipItemTobacco`. Has `PipeSmokingTrigger` and `PipeExhaleEffect`.

#### `ShipItemTobacco` — Tobacco
Various colors (white/green/black/brown/blue). Consumed via pipe.

#### `ShipItemElixir` — Elixir (Specific Effect)
| Field | Type | Description |
|-------|------|-------------|
| `addedSleep` | float | Sleep restoration |
| `addedWater` | float | Hydration restoration |
| `addedFood` | float | Hunger restoration |

#### `ShipItemRandomElixir` — Random Effect Elixir
Random effects on consumption.

---

### Storage & Cargo

#### `ShipItemCrate` — Crate/Box
| Field | Type | Description |
|-------|------|-------------|
| `containedPrefab` | GameObject | Prefab of contents |
| `smokedFood` | bool | Contents are pre-smoked |

**Behavior:** `big = true`. Unseal spawns `amount` × `containedPrefab`. Mass includes contents. `CrateInventory` manages sealed state.

#### `ShipItemFoldable` — Foldable (Maps)
| Field | Type | Description |
|-------|------|-------------|
| `allowCharting` | bool | Can draw charts on it |
| `foldedMesh` / `unfoldedMesh` | Mesh | Fold/unfold meshes |
| `unfoldedDetails[]` | GameObject[] | Details shown when unfolded |
| `foldedColSize` | Vector3 | Collider size when folded |
| `mapChart` | MapChart | Chart drawing component |

---

### Fishing

#### `ShipItemFishingRod` — Fishing Rod
Holds hook. Casting mechanic. `amount` = has hook attached.

#### `ShipItemFishingHook` — Fishing Hook
Click on rod to attach. Used as bait.

---

### Misc

#### `ShipItemScroll` — Scroll/Document
| Field | Type | Description |
|-------|------|-------------|
| `directory` | ScrollDirectory | Scroll content directory |
| `page` | Renderer | Page renderer |
| `openMesh` / `closedMesh` | Mesh | Open/close states |

**Behavior:** `amount` = scroll type index (tutorial ≤2, other >2). Tutorial scrolls: value=30, other: value=120.

#### `ShipItemTotem` — Weather Totem
| Field | Type | Description |
|-------|------|-------------|
| `castParticles` | ParticleSystem | Casting effect |
| `stormAttraction` | float | Storm attraction power |
| `castAudio` | AudioSource | Casting sound |

**Behavior:** Alt-held to cast. After cast: `health = -1`. Sun totem (index 163) and Rain totem (index 164).

#### `ShipItemInkSet` — Ink Set
Click on `ShipItemFoldable` (with `allowCharting`) to draw on maps.

---

## Complete Subclass List (35+ classes)

| # | Class | Base | Category |
|:--|-------|------|----------|
| 1 | `ShipItem` | PickupableItem | base |
| 2 | `ShipItemBed` | ShipItem | furniture |
| 3 | `ShipItemBottle` | ShipItem | drink |
| 4 | `ShipItemBroom` | ShipItem | tool |
| 5 | `ShipItemChipLog` | ShipItem | navigation |
| 6 | `ShipItemClock` | ShipItem | navigation |
| 7 | `ShipItemCompass` | ShipItem | navigation |
| 8 | `ShipItemCrate` | ShipItem | cargo |
| 9 | `ShipItemElixir` | ShipItem | consumable |
| 10 | `ShipItemFishingHook` | ShipItem | fishing |
| 11 | `ShipItemFishingRod` | ShipItem | fishing |
| 12 | `ShipItemFoldable` | ShipItem | misc |
| 13 | `ShipItemFood` | ShipItem | food |
| 14 | `ShipItemHammer` | ShipItem | tool |
| 15 | `ShipItemHangable` | ShipItem | base-hangable |
| 16 | `ShipItemInkSet` | ShipItem | tool |
| 17 | `ShipItemKettle` | ShipItem | cooking |
| 18 | `ShipItemKnife` | ShipItem | tool |
| 19 | `ShipItemLampHook` | ShipItem | lighting |
| 20 | `ShipItemLanternFuel` | ShipItem | fuel |
| 21 | `ShipItemLight` | ShipItemHangable | lighting |
| 22 | `ShipItemOakum` | ShipItem | tool |
| 23 | `ShipItemOar` | ShipItem | tool |
| 24 | `ShipItemPipe` | ShipItem | consumable |
| 25 | `ShipItemQuadrant` | ShipItem | navigation |
| 26 | `ShipItemRandomElixir` | ShipItem | consumable |
| 27 | `ShipItemSalt` | ShipItem | cooking |
| 28 | `ShipItemScroll` | ShipItem | misc |
| 29 | `ShipItemSoup` | ShipItem | food |
| 30 | `ShipItemSpyglass` | ShipItem | navigation |
| 31 | `ShipItemStove` | ShipItem | cooking |
| 32 | `ShipItemStoveFuel` | ShipItem | fuel |
| 33 | `ShipItemTea` | ShipItem | drink |
| 34 | `ShipItemTobacco` | ShipItem | consumable |
| 35 | `ShipItemTotem` | ShipItem | magic |

---

*Extracted from Assembly-CSharp.dll decompilation (Sailwind v0.38).*
