# 33. Item Spawning and Pickup in the World

How the game creates items in the world and how the player picks them up. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Item framework — in note 16, inventory — in note 32.

## World Item Spawning (`WorldItemSpawner` + `ItemSpawners`)

### `ItemSpawners` (singleton, cooldown registry)
- `cooldowns[]` (`float`) — one cooldown per spawner. Ticks in `Update`: `cooldowns[i] -= deltaTime * Sun.sun.timescale` (depends on time acceleration).
- `RegisterNewSpawner()` — finds free slot (`cooldown < -2`), sets `0`, returns index.
- Cooldowns saved to save (`itemSpawnerCooldowns`, note 11) — respawn survives loading.

### `WorldItemSpawner` (single item spawn point)
Placed in world as GameObject. Fields: `itemSpawnerIndex` (slot in `ItemSpawners`), `respawnTime`, `itemPrefab`, `item` (currently spawned item).

**Logic (`Update`):**
1. **Item picked up** (`item.held != null`): spawner sets respawn cooldown — `respawnTime * Random(0.75, 1.25)` (or `-999` if `respawnTime <= 0` = no respawn), registers item in save (`RegisterToSave`), releases reference (`item = null`).
2. **No item and cooldown expired** (`0 >= cooldown > -10`):
   - if player **further than 100 units** → defer (`cooldown = 0.1`, try later) — don't spawn far away;
   - otherwise → `SpawnItem()`.

**`SpawnItem()`** — the "ritual" of item creation:
```csharp
item = Instantiate(itemPrefab, transform.position, transform.rotation).GetComponent<ShipItem>();
item.transform.parent = transform.parent;
item.sold = true;                       // "world-owned", picked up
item.GetComponent<Good>()?.RegisterAsMissionless();   // not tied to mission
StartCoroutine(FreezeItem());           // kinematic = true next frame (sits motionless)
```

> **Important for modders:** spawned item is **kinematic** (`debugForceKinematic = true`) — it doesn't fall or sink until touched. Buoyancy activates when item is taken off "freeze".

## Item Buoyancy (`ItemRigidbody` + `SimpleFloatingObject`)

Every item **floats in water**. In `ItemRigidbody` (separate item physics object, note 16):
```csharp
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;   // "float-up" height (from ShipItem.floaterHeight, default 1.6)
```
- `SimpleFloatingObject` (class absent from repo, note 24) holds item on surface: `_raiseObject` — target height above water, `_dragInWaterRotational` — rotational drag.
- `floater.enabled = false` — when item shouldn't float (in inventory, in hand, on stove).

## Item Pickup (`GoPointer`)

Pickup goes through raycast pointer `GoPointer` (note 10/03).

### Main Methods
| Method | Action |
|-------|----------|
| `PickUpItem(item)` | `heldItem = item; item.held = this;` reset rotation. Starts smooth pickup animation (`timerAfterPickup`). |
| `DropItem()` | Item layer → 0, `item.held = null`, `heldItem = null`. |
| `GetHeldItem()` | Current item in hand. |

### Holding (`Update`, when `heldItem != null`)
- **Position:** item held at camera — `camera.position + forward * holdDistance + up * holdHeight`, smoothly brought by lerp (`timerAfterPickup`).
- **Mouse wheel rotation:** `OnScroll` rotates `heldRotationOffset` (or direct rotation for large items).
- **Furniture placement** (small, `allowPlacingItems`): `camera + forward*lookDistance + up*furniturePlaceHeight`.
- **Large items** account for decollision (pushing out of walls).

### Collisions and Decollision (`PickupableItemCollisionChecker`)
On item's physics object. Counts collisions (`collisions`) and computes push-out vector via `Physics.ComputePenetration`:
- `GetDecollision()` — vector pushing item out of walls (only if `Settings.enableDecol`).
- `allowObstructedDropping` — can drop even in collision, if penetration `< 0.06`.
- **Red outline** (`enableRedOutline`), if item is in collision and can't be placed here (large, or action key `InputName 8` held).

## Crates (`ShipItemCrate`) — Loot Containers

`ShipItemCrate : ShipItem` — sealed crate with `amount` copies of `containedPrefab`.

| Mechanic | Details |
|----------|--------|
| `containedPrefab` | Item prefab inside (e.g., cheese in cheese crate). |
| `amount` | Number of items inside; `> 0` = **sealed**, `0` = unsealed. |
| `UnsealCrate()` | Instantiates `amount` copies of contents **off-screen** (at `+100.5` Y, to hide spawning) and next frame **transfers them into crate inventory** (`CrateInventory.InsertItem`); `amount → 0`; sound `crateSealBreak`; opens crate UI. **Crate doesn't disappear** — remains as container. |
| `OnAltActivate()` | RMB: sealed (`amount>0`, not mission) → `CrateSealUI` (unseal confirmation); unsealed (`amount<=0`) → `OpenCrate()` (open crate inventory). |
| `OnLoad()` | `value = containedValue * amount + 20` (× 1.2 if `smokedFood`). |
| `lookText` | `"name (amount)"` (sealed) or `"crate"` (empty); for mission items — mission text. |
| `crates` (static) | `Dictionary<int, CrateInventory>` — crate registry. |

> **How it feels in-game:** crate is a good/mission cargo (sealed, `amount>0`). **Unsealing it** — contents don't drop into the world, but **transfer into crate inventory**, and the crate then serves as a **reusable container** (open/close). Unsealed crate cannot be transported by cargo carrier (note 32: `"Cannot transport unsealed crates"`).

## 👁 Modder's View: "I want to spawn pickupable items in the world"

**Minimum path (using existing systems):**
1. Create item prefab: GameObject with `ShipItem` (+ subclass), `Rigidbody`, `Collider`, `Renderer`, `SaveablePrefab` (with unique `prefabIndex` in `PrefabsDirectory.directory`). Buoyancy added automatically (`SimpleFloatingObject`).
2. Place `WorldItemSpawner` in world, set `itemPrefab`, `respawnTime`, register index in `ItemSpawners.instance`. Done — item respawns on player approach.

**Custom spawner (full control, e.g. "random sea loot"):**
```csharp
var go = Object.Instantiate(myItemPrefab, worldPos, Quaternion.identity);
var item = go.GetComponent<ShipItem>();
item.sold = true;                              // otherwise returns to "shop"
go.GetComponent<SaveablePrefab>().RegisterToSave();   // saving
// Y position from water surface:
//   OceanHeight.GetHeight(helper, worldPos)  (note 31)
// account for floating origin:
//   worldPos = FloatingOriginManager.instance.RealPosToShiftingPos(realPos)  (note 11)
```
- **Pickup already works**: player aims (GoPointer raycast) and clicks → `PickUpItem`. Nothing else needed.
- **Floating is NOT "out of the box"**: `ItemRigidbody` creates `SimpleFloatingObject`, but **disables it** (`ToggleCollider`, every frame) and doesn't re-enable; body is often kinematic. For real floating of spawned items you need **your own float** (Boyant-snap) or patch — **detailed in note 43**.
- **Saving**: `RegisterToSave()` → item goes into `savedPrefabs` in save (note 11) and survives loading.
- **Don't spawn far** from player (as `WorldItemSpawner` does, threshold 100 units) — and account for world shift (`outCurrentOffset`), otherwise item will "drift away".
- **Crate** (`ShipItemCrate`) is picked up **in hand/on deck, not into inventory slot**; unsealing → contents into `CrateInventory` (notes 32, 45).

## Practical Takeaways

1. **Spawning** = `WorldItemSpawner` (respawn by cooldown, only within 100 units, **`debugForceKinematic=true`** = frozen) + "ritual" `Instantiate → sold=true → RegisterAsMissionless → RegisterToSave`.
2. **Spawner cooldowns** tick by game time and are saved to save.
3. **Spawned item buoyancy doesn't work by default** (floater created but disabled by `ToggleCollider`; see **note 43** — truth table, canonical water height, custom float/patch).
4. **Pickup** fully handled by `GoPointer` (`PickUpItem`/`DropItem`); holding at camera, wheel rotation, decollision, red outline. Don't add anything to visual (note 44).
5. **Crate** (`ShipItemCrate`) — ready "loot container": sealed (`amount>0`); unsealing **transfers contents into crate inventory** (crate remains as reusable container, doesn't disappear). `GetComponent<ShipItemCrate>()` catches not just cargo crates — filter in note 45.
6. For custom loot: prefab with `ShipItem`+`SaveablePrefab` → `Instantiate` + `sold=true` + `RegisterToSave` → **pickup works out of the box, floating doesn't** (see note 43).
