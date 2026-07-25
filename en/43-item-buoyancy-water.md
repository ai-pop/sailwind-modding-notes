# 43. Item Buoyancy and Water: Mechanics in Detail

Detailed technical analysis of how items interact with water: buoyancy, kinematics, water height. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 14 (boat buoyancy), 16 (ShipItem), 31 (ocean), 33 (spawning/pickup).

## Two GameObjects per Item

Reminder (note 16): every `ShipItem` has **two** objects:
- **visual** — the `ShipItem` itself (Renderer, trigger colliders, logic);
- **`itemRigidbody`** — separate runtime object (`ItemRigidbody`), created in `ShipItem.CreateRigidbody()`; holds `Rigidbody`, physics colliders and **`SimpleFloatingObject`** (buoyancy).

Buoyancy is applied to **`itemRigidbody`**, not visual. Visual follows `itemRigidbody` (or vice versa, depending on state).

## `ItemRigidbody`: Creation and Floater

In `Start()` (on `itemRigidbody` object):
```csharp
rigidbody = AddComponent<Rigidbody>();
rigidbody.drag = 1.2f;
rigidbody.angularDrag = item.mass * 0.1f;
rigidbody.isKinematic = true;                 // starts kinematic
UpdateMass();                                 // mass = item.mass + contents (crate/bottle/soup/salt/tea)
StartCoroutine(AddCollider());                // Box/Mesh/Capsule colliders (copy from visual)
floater = AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;    // float-up height = ShipItem.floaterHeight (default 1.6)
gameObject.layer = 2;
ResetPos();                                   // rigidbody.isKinematic = true (again)
world = FloatingOriginManager.instance.transform;
SetDynamicColTimer();                         // dynamicColTimer = 6
```

- `SimpleFloatingObject` (Crest, base class `FloatingObjectBase`) is created **for every item**.
- `_raiseObject` taken from `ShipItem.floaterHeight` — target height above water.
- `FloatingObjectBase.InWater` — "object in water" flag.
- Body **starts kinematic** (`isKinematic = true`).

## Kinematics: When Item Is Dynamic vs Not

In `FixedUpdate`, `flag2` is computed → `rigidbody.isKinematic = flag2`. Item is **kinematic** (`flag2 = true`) if **at least one** condition holds:

| Condition | Comment |
|---------|-------------|
| `item.held` | item in hand |
| `!item.sold` | not purchased (shop item) |
| `item.nailed` | nailed/secured |
| `attached` | attached (e.g. to wall/stove) |
| `currentInventorySlot` or `currentBox` | in inventory/crate |
| `outOfRange` | further than **600** units from `Camera.main` |
| `!GameState.playing || recovering || sleeping || inBed || currentShipyard` | utility states |
| `fixedFramesSinceSpawn < 6` | first 6 fixed frames after spawn |
| `debugForceKinematic` (item field) | forced (set by e.g. `WorldItemSpawner`) |
| `Debugger.kinematicItemsTimer > 0` / `debugForceKinematicBoat` | debug flags |
| `meshCol != null && rigidbody.IsSleeping() && dynamicColTimer <= 0` | mesh collider "sleeping" |

Item is **dynamic** (`isKinematic = false`) only when: purchased (`sold`), not in hand/inventory/crate, not nailed/attached, **within 600 units** of camera, game active, ≥ 6 fixed frames passed, no forced kinematics.

> Kinematic body **doesn't respond to forces** (including buoyancy force). So for buoyancy, body must be dynamic.

## Floater Management: `ToggleCollider` Peculiarity

```csharp
public void ToggleCollider(bool state) {
    ((Behaviour)floater).enabled = false;   // ← unconditional floater disable
    if (boxCol) boxCol.enabled = state;
    if (meshCol) meshCol.enabled = state;
    if (capsuleCol) capsuleCol.enabled = state;
}
```

- `ToggleCollider` **always** sets `floater.enabled = false` (regardless of `state`).
- In `FixedUpdate` for item not in inventory, `ToggleCollider(true)` is called (enable colliders) — this **also disables floater**.
- In game code, **there is no place** that sets `floater.enabled = true` back.

So for an item going through standard `ItemRigidbody`, `SimpleFloatingObject` is created enabled, but effectively **switched to `enabled = false`** and never re-enabled by the game. Whether Crest buoyancy still applies depends on internal `SimpleFloatingObject` implementation (class absent from export): if it relies on own `enabled` cycle — buoyancy doesn't apply; runtime check needed.

### Physics "Sleeping" Far from Player
- `outOfRange` = distance to `Camera.main` > **600** units (check every 5–8 s, `distanceCheckTimer`).
- At `outOfRange`, `FixedUpdate` does **early exit** (physics not managed), and for purchased item not on boat counts `framesUntilDestroy`; after > 10 frames — `DestroyItem()` (distant items **destroyed**).

## What Actually Holds Item on Water: 4 Mechanisms

| Mechanism | Class | How It Works | Used By |
|----------|-------|--------------|------------------|
| **Crest float** | `SimpleFloatingObject` (`FloatingObjectBase`) | Force/lift to surface, `_raiseObject`, `InWater` | `ItemRigidbody` (creates, but see `ToggleCollider`), **working** examples: fishing bobber (`FishingRodFish`/`ShipItemChipLog`), `ChipLogRopeEnd` |
| **Height snap** | `Boyant` | In `Update`: if `ocean.canCheckBuoyancyNow[0]==1`, sets `position.y = GetWaterHeightAtLocation2(x−choppy, z) + buoyancy` (direct height setting) | Defined, but **not called** by other classes (possible on scene prefabs) |
| **Blob buoyancy** | `Buoyancy` (note 14) | Point grid, force by submersion depth | Boats, large objects |
| **Ocean directly** | `Ocean` (Crest) | `GetWaterHeightAtLocation2`, `GetChoppyAtLocation2` | Data source for all above |

### Working Examples of Floating Items
- **Fishing bobber** (`FishingRodFish.bobber`, `ShipItemChipLog.bobberBody`): separate **dynamic** body (`!isKinematic`) with **its own enabled** `SimpleFloatingObject` (not through `ItemRigidbody.ToggleCollider`). Bobber guaranteed to float — entire fishing mechanic relies on this (`floater.InWater` checked constantly).
- **Chip log rope end** (`ChipLogRopeEnd`): `GetComponent<SimpleFloatingObject>()` from prefab, reads `InWater`.

Common trait of working cases: **dynamic body + enabled `SimpleFloatingObject` not managed through `ToggleCollider`**.

## Water Height: Correct Call

| API | What It Does |
|-----|-----------|
| `Ocean.Singleton` | Static access to ocean. |
| `ocean.GetWaterHeightAtLocation2(x, z)` | Water surface height at point. |
| `ocean.GetChoppyAtLocation2(x, z)` / `GetChoppyAtLocation(x, z)` | Horizontal displacement ("choppy"); take height at `x − choppy`. |
| `ocean.canCheckBuoyancyNow[0]` | Byte gate: `== 1` when Crest ready to compute height (set 0/1 inside `Ocean`). Height queries only meaningful at `== 1`. |
| `ocean.choppy_scale` (2f) | Choppy scale. |
| `OceanHeight.GetHeight(SampleHeightHelper, pos[, accuracy])` | Wrapper over Crest `SampleHeightHelper.Init/Sample` (notes 24/31). |
| `OceanHeight.IsUnderwater(helper, pos)` | `GetHeight(helper, pos) > pos.y`. |

**Floating origin** (note 11): query water height in **scene coordinates** (as object stands); for "real"/global calculations — `FloatingOriginManager.instance.ShiftingPosToRealPos / RealPosToShiftingPos`. `GetWaterHeightAtLocation2` works with current (scene) coordinates.

## "Item Floats in World" Ritual (Minimum Sequence)

After `Instantiate(prefab, scenePos, rot)` (visual created; `ShipItem` will create `itemRigidbody` via `CreateRigidbody`):

1. `var item = go.GetComponent<ShipItem>(); item.sold = true;` — otherwise `!sold` → kinematic (note 33).
2. `go.GetComponent<Good>()?.RegisterAsMissionless();`
3. `go.GetComponent<SaveablePrefab>().RegisterToSave();` — saving.
4. **Don't set** `itemRigidbody.debugForceKinematic = true` (this freezes body; as `WorldItemSpawner` does).
5. Keep item within 600 units of camera (otherwise early physics exit + destruction).
6. **Buoyancy** — standard `SimpleFloatingObject` on item via `ItemRigidbody` is effectively disabled (`ToggleCollider`, see above). For item to actually float, need **one of**:
   - **(a)** enable and maintain `floater.enabled = true` (patch `ItemRigidbody.ToggleCollider` to not touch floater, or re-enable in `LateUpdate` after `ItemRigidbody`); body must be dynamic;
   - **(b)** add own float modeled on `Boyant` — snap `position.y` to `GetWaterHeightAtLocation2` (simple, doesn't conflict with `ItemRigidbody`);
   - **(c)** hybrid: leave `ItemRigidbody` for mass/collisions, drive Y yourself.

### Code Skeleton (BepInEx, variant b — custom snap float)
```csharp
// own component on item visual object
class KeepOnSurface : MonoBehaviour {
    Rigidbody _body; Ocean _ocean;
    public float buoyancy = 0.3f;
    void Start() { _ocean = Ocean.Singleton; _body = GetComponentInChildren<Rigidbody>(); }
    void LateUpdate() {
        if (_ocean == null || _ocean.canCheckBuoyancyNow[0] != 1) return;
        var p = transform.position;
        float chop = _ocean.GetChoppyAtLocation2(p.x, p.z);
        float y = _ocean.GetWaterHeightAtLocation2(p.x - chop, p.z) + buoyancy;
        // softly pull toward surface (or set directly, as Boyant does)
        p.y = Mathf.Lerp(p.y, y, Time.deltaTime * 3f);
        transform.position = p;
    }
}
// spawn:
var go = Object.Instantiate(itemPrefab, scenePos, Quaternion.identity);
var item = go.GetComponent<ShipItem>(); item.sold = true;
go.GetComponent<Good>()?.RegisterAsMissionless();
go.GetComponent<SaveablePrefab>().RegisterToSave();
go.AddComponent<KeepOnSurface>();   // own float
// IMPORTANT: don't set debugForceKinematic; keep within 600 of camera
```

## Summary: What Determines Whether an Item Floats

1. **Body dynamic?** (`sold`, not held/nailed/attached, not in inventory/crate, within 600, game active, ≥ 6 frames, no `debugForceKinematic`). Kinematic → no forces → doesn't float.
2. **Buoyancy enabled?** Standard `SimpleFloatingObject` on `ItemRigidbody` disabled by `ToggleCollider` and never re-enabled by game → for real buoyancy need enabled floater (patch) OR own float (`Boyant`-snap).
3. **Water height** taken from `Ocean.GetWaterHeightAtLocation2(x−choppy, z)` when `canCheckBuoyancyNow[0]==1`, in scene coordinates.
4. **Working reference** — fishing bobber: dynamic body + own enabled `SimpleFloatingObject`.

## Truth Table: Item Type → Does It Float?

(by v0.38 code logic; for items through `ItemRigidbody` consistent with runtime observation `floater=False` → sinks)

| Item Type | floater created? | floater `enabled`? | Kinematic by default | Floats in vanilla? |
|--------------|:--:|:--:|--------------------------|:--:|
| Standard `ShipItem` (free, `sold`) | yes | **NO** (disabled by `ToggleCollider`) | starts kinematic → dynamic when free/in zone/playing | effectively **no** |
| Shop item (`!sold`) | yes | **NO** | **kinematic** (`!sold` → `flag2`) | no |
| Mission good (`Good`) | yes | **NO** | same as free `sold` | no |
| World-spawned (`WorldItemSpawner`) | yes | **NO** | **kinematic** (`debugForceKinematic=true`) | no (frozen in place) |
| Fishing bobber (`FishingRodFish`/`ShipItemChipLog`) | own | **YES** | dynamic | **YES** |
| Chip log rope end (`ChipLogRopeEnd`) | own | **YES** | dynamic | **YES** |
| Boat | `Buoyancy` (blob), not `SimpleFloatingObject` | — | — | **YES** (different system, note 14) |

Conclusion: for items going through standard `ItemRigidbody`, floater is created but **disabled** and body is often managed by kinematics → no vanilla buoyancy. Guaranteed floating objects have **own enabled** `SimpleFloatingObject` on dynamic body (bobber, chip log) and boats (blob-`Buoyancy`).

## Sequence: spawn → twin → floater → pickup → drop → float

```
Object.Instantiate(prefab, scenePos, rot)            // VISUAL created (ShipItem)
  └─ ShipItem.Awake()
       └─ StartCoroutine(LoadAfterDelay())
            ├─ CreateRigidbody()                     // TWIN: new GO + ItemRigidbody
            │     ├─ rigidbody (isKinematic = true)
            │     └─ floater = AddComponent<SimpleFloatingObject>()  // ENABLED, _raiseObject=floaterHeight
            ├─ AddCollisionChecker() / AddLODGroup()
            ├─ [WAIT WaitForEndOfFrame]              // ← twin appears NOT synchronously with Instantiate
            └─ OnLoad()
  ── (mod) sold = true; Good?.RegisterAsMissionless(); SaveablePrefab.RegisterToSave()
  ── ItemRigidbody.FixedUpdate() [every fixed frame]
       ├─ outOfRange (>600 from camera)? → early exit (+ culling)
       ├─ ToggleCollider(true)  →  floater.enabled = false      // ← floater DISABLED
       └─ rigidbody.isKinematic = flag2                          // dynamic if sold/free/in zone/playing
  ── player: GoPointer raycast on visual trigger → click → PickUpItem()   // into hand (twin follows visual)
  ── drop → item in world, TWIN = position master (visual follows twin)
  ── buoyancy: standard floater off → for floating need OWN float (Boyant-snap) or patch (below)
```

## Canonical Water Height Snippet (+ Floating Origin)

One block instead of scattered notes 11/31/34:

```csharp
// Instant water surface height at CURRENT object position.
// Pattern from Boyant/Buoyancy (v0.38). Input — SCENE coordinates of object.
public static bool TryGetWaterY(Vector3 scenePos, out float y)
{
    var o = Ocean.Singleton;
    y = scenePos.y;                                   // fallback: keep current Y
    if (o == null || o.canCheckBuoyancyNow == null || o.canCheckBuoyancyNow[0] != 1)
        return false;                                 // Crest not ready → DO NOT sample
    float chop = o.GetChoppyAtLocation2(scenePos.x, scenePos.z);
    y = o.GetWaterHeightAtLocation2(scenePos.x - chop, scenePos.z);   // subtract choppy!
    return true;
}
```

- **Input = scene xz** (current object position). `GetWaterHeightAtLocation2` **internally tiles** wave grid (`sizeQX/sizeQZ` + fractional part), so for instant lookup manual real↔scene conversion **not needed**.
- **Garbage sample (~0.4)** occurs when: (a) sample when `canCheckBuoyancyNow[0] != 1` (Crest data not ready), or (b) `chop` not subtracted (`GetChoppyAtLocation2`). Gate + choppy subtraction fix the issue.
- **Floating origin** matters for **storage/comparison** of positions over time (spawn points, despawn distance): store in **real** (`ShiftingPosToRealPos`), apply in **scene** (`RealPosToShiftingPos`) — note 11. For instant height at current position — scene directly.
- **Fallback**, if `Ocean`/Crest unavailable: no height source — keep current `y` or fixed level; snap without height impossible.

## End-to-End Checklist: "Item Floats in World and Is Clickable"

1. `Object.Instantiate(prefab, scenePos, rot)` (visual; `ShipItem` creates twin itself).
2. `sold = true`; `GetComponent<Good>()?.RegisterAsMissionless()`; `GetComponent<SaveablePrefab>().RegisterToSave()`.
3. **Wait for twin**: `while (itemRigidbodyC == null) yield return null;` (or 1–2 `WaitForEndOfFrame`).
4. **Buoyancy** — own float on **twin** (Boyant-snap via `TryGetWaterY`) OR patch `ToggleCollider` + keep `floater.enabled = true`. **Not on visual.**
5. **Not kinematic**: `GetItemRigidbody().debugForceKinematic = false`; item `sold`, not held/attached/nailed, within 600, `playing`.
6. **Pickup/unseal work**: don't add anything to visual (visual trigger colliders = pickup raycast); crate unsealed normally (`CrateSealUI` → `CrateInventory`).
7. **Save/load**: item itself saves via `RegisterToSave` (`savedPrefabs`); spawner state (timer, instanceId list) — in `GameState.modData`.

## Consolidated `GameState` Gates for Ticking/Spawning Mod

| Flag | When | What to Gate |
|------|-------|-------------|
| `GameState.playing` | active gameplay | entire mod tick |
| `GameState.sleeping` | sleep | don't spawn/move |
| `GameState.recovering` | blackout/recovery | don't spawn |
| `GameState.currentShipyard != null` | at shipyard | don't spawn |
| `GameState.currentlyLoading` | scene loading | don't spawn, wait |
| `Refs.observerMirror == null` | no player position | don't spawn |
| Start menu / loading animation | main menu | don't tick (covered by `playing == false`) |

Unified gate:
```csharp
bool CanTick() =>
    GameState.playing && !GameState.sleeping && !GameState.recovering &&
    GameState.currentShipyard == null && !GameState.currentlyLoading &&
    Refs.observerMirror != null;
```

## Antipatterns (What Breaks)

- ❌ **`AddComponent<SimpleFloatingObject>`/`Rigidbody` on VISUAL** → breaks pickup: for free item master = twin (overwrites visual), and visual trigger colliders used for raycast. **Physics/float — only on twin.**
- ❌ **Store spawn position in scene coordinates** and apply after world shift → drifts away. Store **real**, apply **scene** (note 11).
- ❌ **Sample `GetWaterHeightAtLocation2` when `canCheckBuoyancyNow[0] != 1`** → garbage (~0.4).
- ❌ **Don't subtract `chop`** (`GetChoppyAtLocation2`) → height offset.
- ❌ **`timer = 0` after empty `modData`** → zero/infinite spawn interval. When key missing — initialize with default.
- ❌ **`debugForceKinematic = true`** when floating needed (this freezes; as `WorldItemSpawner` does).
- ❌ **`shipItem.debugForceKinematic`** → CS1061: field is on **`ItemRigidbody`** (`GetItemRigidbody().debugForceKinematic`).
- ❌ **`UISoundPlayer.instance` on early boot** without guard → null. Check `instance != null` / try.
- ❌ **`GetComponent<ShipItemCrate>()` as "cargo crate" filter** → catches `tobacco big`, `apples` (note 45).
- ❌ **Access twin immediately after `Instantiate`** → `itemRigidbodyC == null` (twin created in coroutine from `Awake`).

## Practical Takeaways

- `SimpleFloatingObject` is created on every item (`_raiseObject = floaterHeight`), but standard `ItemRigidbody.ToggleCollider` path switches it to `enabled = false` with no re-enable in game code.
- Kinematics gated by long list of conditions (table above); far away (>600) physics disabled, and purchased items off boat destroyed.
- Guaranteed floating objects have **dynamic body and enabled `SimpleFloatingObject` outside `ToggleCollider` management** (bobber, chip log).
- `Boyant` — ready simple snap-float (`GetWaterHeightAtLocation2 + buoyancy`), convenient model for own solution.
- Water height queries — only at `canCheckBuoyancyNow[0]==1`, in scene coordinates, accounting for choppy displacement.
