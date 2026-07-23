# 16. Item Framework: ShipItem and Family

Analysis of the item system (everything you can pick up, buy, put in inventory/cargo hold). Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Foundation for inventory mods, new items, trading.

## Class Hierarchy

```
GoPointerButton            (interactive object, see note 10)
  └─ PickupableItem        (everything pickupable)
       └─ ShipItem         (base "ship" item, [RequireComponent Rigidbody+Collider+Renderer])
            └─ 35 subclasses (food, tools, furniture, drinks…)
```

### `PickupableItem : GoPointerButton`
| Field | Default | Purpose |
|------|:--:|-----------|
| `big` | — | Large (two-handed) item. |
| `holdDistance` | 1.15 | Holding distance from camera. |
| `furniturePlaceHeight` | 0.15 | Furniture placement height. |
| `holdHeight` | — | Vertical offset in hand. |
| `held` | — | `GoPointer` that holds the item. |
| `heldRotationOffset` | — | Rotation of held item; `OnScroll(input)` rotates by `input * 3` (mouse wheel). |
| `colChecker` | — | `PickupableItemCollisionChecker`. |

Virtual hooks: `OnPickup()`, `OnDrop()`, `OnScroll()`.

## `ShipItem` — Base Item

### Main Fields
| Field | Default | Content |
|------|:--:|-----------|
| `mass` | 1 | Mass (affects `BoatMass`, see note 14). |
| `value` | — | **Base price** of good (used by market, see note 13). |
| `name` | — | Item name (used in `lookText`, missions). |
| `category` | — | `TransactionCategory` — transaction category. |
| `inventoryScale` / `inventoryRotation(X)` | 1 / 0 | How item appears in inventory slot. |
| `floaterHeight` | 1.6 | Float height (buoyancy in water). |
| `sold` | false | Whether purchased. In shop — `false`; purchase → `Sell()` → `true`. |
| `nailed` | false | Nailed/secured (doesn't move). |
| `health` | — | Durability/condition (for food — freshness, for liquids — volume). |
| `amount` | — | Quantity (stacks, volume). |
| `daysInStorage` | — | Days in storage (spoils). |
| `wallAttachment` | false | Can hang on wall (raycast attachment). |

### Dual-Object Physics Architecture
**Each item is two GameObjects:**
1. **Main** (`ShipItem`) — visuals (`Renderer`), trigger colliders (`isTrigger = true`), LOD.
2. **`itemRigidbody`** — separate runtime object with real physics body (`ItemRigidbody`), created in `CreateRigidbody()` as child of the "world" (`FloatingOriginManager`).

Why: the physics body lives in world coordinate space and is reused when entering/exiting the boat, going into inventory, etc. `Rigidbody.collisionDetectionMode = 3` (ContinuousDynamic). In `Awake()`, all colliders on the main object are converted to triggers.

`ItemRigidbody` additionally:
- holds `SimpleFloatingObject` (`floater`) — items **float** in water;
- states `onBoat`, `inStove`;
- `idleTimer` / `dynamicColTimer` — optimization: "sleeping" physics of unused items.

### `ItemRigidbody` and Placement
- **Inventory:** `GPButtonInventorySlot` slot (`slotIndex`).
- **Cargo carrier** (`CargoCarrier`): `GetCurrentInventorySlot()` returns `portIndex + 100` (see note 11 on `inventorySlot >= 100`).
- **Wall attachment** (`wallAttachment`): forward raycast at 1.3 m; on hit — `attachPos`/`attachRot` calculated, item hung (`HangableItem`).

### Entering/Exiting Boat
Item detects trigger with tag `EmbarkCol` (`BoatEmbarkCollider`). After `frameCounter > threshold`:
- **EnterBoat:** item reparents to boat, `saveable.SetParentObject(boat's sceneIndex)`, `ItemRigidbody.EnterBoat()`.
- **ExitBoat:** reparent to world, `SetParentObject(-1)`, `HangableItem.DisconnectJoint()`.

### Special Parent Object Codes (`saveable.GetParentObject()`)
`ProcessSaveable()` handles magic `parentObject` values:

| Code | Meaning |
|:--:|----------|
| `-1` | In the world (not on boat). |
| `-2` | **Destroy item** (`DestroyItem()`). |
| `-3` | **Lost** (cargo-loss during blackout): layer = 2 (`IgnoreRaycast`), destroyed when `GameState.recovering && eyesFullyClosed`. This implements "cargo loss" from `Recovery.GetCargoLossChance` (note 12). |

### Buying/Selling
- `AddToShop(shop)`: remembers `shopPos`/`shopRot`, adds to `shop.itemsForSale`.
- `Sell()`: `sold = true`, reparent to world, `RegisterToSave()`, picked up by player, `OnBuy()`.
- `ReturnToShopPos()`: if item is unsold and dropped — smoothly (lerp 2/s) returns to shop shelf.
- `OnAltActivate()`: right-click on unsold good → `shopkeeper.TryToSellItem(this)` (attempt to sell to shop).

### `DestroyItem()`
Removes item from boat, `SaveablePrefab.Unregister()` (removes from save), `Destroy(gameObject)`. Calling without `SaveablePrefab` is logged as `CRITICAL ERROR`.

## ShipItem Family (35 Subclasses)

| Category | Classes |
|-----------|--------|
| **Food/Drink** | `ShipItemFood`, `ShipItemSoup`, `ShipItemTea`, `ShipItemKettle`, `ShipItemBottle`, `ShipItemSalt` |
| **Cooking** | `ShipItemStove`, `ShipItemStoveFuel`, `ShipItemKnife` |
| **Tools** | `ShipItemHammer`, `ShipItemBroom`, `ShipItemOakum` (caulking), `ShipItemOar` (oar) |
| **Navigation** | `ShipItemCompass`, `ShipItemClock`, `ShipItemQuadrant`, `ShipItemSpyglass`, `ShipItemChipLog` |
| **Lighting** | `ShipItemLight`, `ShipItemLanternFuel`, `ShipItemLampHook` |
| **Rest/Habits** | `ShipItemBed`, `ShipItemPipe` (pipe), `ShipItemTobacco`, `ShipItemElixir`, `ShipItemRandomElixir` |
| **Storage/Cargo** | `ShipItemCrate` (box), `ShipItemFoldable` (foldable, maps) |
| **Fishing** | `ShipItemFishingRod`, `ShipItemFishingHook` |
| **Misc** | `ShipItemHangable`, `ShipItemInkSet` (ink), `ShipItemScroll` (scroll), `ShipItemTotem` |

Subclasses override `OnLoad()`, `OnBuy()`, `OnEnterInventory()`, `UpdateLookText()` and store special state (soup has `currentWater/currentEnergy/...`, kettle has `currentTeaType/Amount`, etc.) serialized via `extraValue0..4` (see note 11).

## `TransactionCategory` (enum)

Transaction category — used in economy/income logs (`DayLogs`, `TransactionCategory`):

`retailFood, retailWater, retailAlco, toolsAndSupplies, furniture, otherItems, bulkFood, bulkWater, bulkAlco, bulkGood, mission, boat, recovery, other, currencyExchange`

`ShipItem.IsBulk()` = true for `bulkAlco/bulkFood/bulkWater/bulkGood` (bulk/cargo-hold goods).

## Practical Takeaways for Modding

1. **New item** = GameObject with `ShipItem`(+subclass), `Rigidbody`, `Collider`, `Renderer`, `SaveablePrefab` (with unique `prefabIndex` in `PrefabsDirectory.directory`). Without `SaveablePrefab`, the item doesn't save and breaks `DestroyItem`.
2. **Physics is separated** from visuals (`itemRigidbody`) — account for this when spawning/teleporting items.
3. **`value`** — base price for market; **`mass`** — contribution to boat mass.
4. **`sold`** distinguishes shop item from purchased; **`nailed`** — secured.
5. **`parentObject` codes** `-1/-2/-3` control world/destroy/cargo-loss — don't use these values blindly.
6. **Inventory:** slot `< 100`; cargo carrier = `portIndex + 100`.
7. Mission item tooltip — `"{name}\nto {port}\ndue: {text}"` (hardcoded, EN).
