# 44. `ItemRigidbody`: Field Map and Twin Object Contract

Exact API contract of the item's physics twin object — to avoid guessing with reflection. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. "Two GameObject" architecture — in note 16; buoyancy — in note 43.

## Two Objects and How to Get Them

Every `ShipItem` has **visual** (the `ShipItem` itself) and **twin** (separate GameObject with physics).

### Fields on `ShipItem`
| Field | Modifier | Type | What It Is |
|------|-------------|-----|---------|
| `itemRigidbody` | `protected` | `Transform` | Twin object transform. **Not public** — can't access directly from outside. |
| `itemRigidbodyC` | `public` | `ItemRigidbody` | `ItemRigidbody` component on twin. **Public**. |

### Canonical Accessors (on `ShipItem`)
```csharp
public ItemRigidbody GetItemRigidbody()  // → itemRigidbody.GetComponent<ItemRigidbody>()
public void ResetRigidbody()             // → GetItemRigidbody().ResetPos()
public Rigidbody /* via twin */        // shipItem.GetItemRigidbody().GetBody()
```
- **Twin's Rigidbody** reliably accessed via: `shipItem.GetItemRigidbody().GetBody()` (`GetBody()` returns twin's private `rigidbody`).
- `ItemRigidbody` itself: `shipItem.itemRigidbodyC` (public) or `shipItem.GetItemRigidbody()`.

## ⚠ Twin Creation Timing (Critical)

Twin is created **in a coroutine started from `Awake`**, not synchronously:

```
ShipItem.Awake()
  └─ StartCoroutine(LoadAfterDelay())
        ├─ (before first yield, same frame, but AFTER Awake)
        │    CreateRigidbody()      ← creates twin-GameObject + ItemRigidbody
        │    AddCollisionChecker()
        │    AddLODGroup()
        │    visual.GetComponent<Rigidbody>().isKinematic = true
        ├─ yield WaitForEndOfFrame
        ├─ OnLoad()                 ← virtual hook for subclass
        ├─ yield WaitForEndOfFrame
        └─ (currentCrateId processing — nesting in crate)
```

**Consequence:** immediately after `Object.Instantiate(prefab)`, `itemRigidbody` / `itemRigidbodyC` fields may still be **null** (coroutine `LoadAfterDelay` hasn't reached `CreateRigidbody`). Before working with twin, **wait**:
```csharp
// variant 1: wait frame(s)
yield return new WaitForEndOfFrame();   // usually 1–2 sufficient

// variant 2: poll
while (shipItem.itemRigidbodyC == null) yield return null;
var twin = shipItem.GetItemRigidbody();
```
`CreateRigidbody()`:
```csharp
itemRigidbody = new GameObject().transform;   // empty GO
itemRigidbody.parent = world;                  // world = FloatingOriginManager.instance.transform
var ir = itemRigidbody.gameObject.AddComponent<ItemRigidbody>();
ir.RegisterItem(this);
itemRigidbodyC = ir;
```
Twin name set in `ItemRigidbody.Start()`: `gameObject.name = item.name + ":ItemRigidbody"`.

## `ItemRigidbody` Field Map

| Field | Modifier | Type | Purpose |
|------|-------------|-----|-----------|
| `debugForceKinematic` | `public` | `bool` | **Forced kinematics** (freeze). Set by `WorldItemSpawner.FreezeItem()`. **Field on `ItemRigidbody`, NOT on `ShipItem`** (access: `shipItem.GetItemRigidbody().debugForceKinematic`). |
| `debugLog` | `public` | `bool` | verbose logging. |
| `attached` | `public` | `bool` | attached (to wall/stove) → kinematic. |
| `disableCol` | `public` | `bool` | disable colliders (+ `ToggleCollider(false)`). |
| `inStove` | `public` | `bool` | item on stove. |
| `rigidbody` | `private` | `Rigidbody` | physics body; externally via `GetBody()`. |
| `floater` | `private` | `SimpleFloatingObject` | float (Crest); created in `Start()`. |
| `item` | `[SerializeField] private` | `ShipItem` | back-reference to visual. |
| `boxCol` / `meshCol` / `capsuleCol` | `private` | colliders | copy of visual colliders (created in `AddCollider()`). |
| `subcolliders` | `private` | `List<Collider>` | sub-colliders (tag `ItemSubcollider`). |
| `onBoat` | `private` | `bool` | on boat. |
| `currentBox` / `currentInventorySlot` | `private` | `Transform` | in crate / in inventory slot. |
| `outOfRange` | `[SerializeField] private` | `bool` | further than 600 units from camera. |
| `idleTimer` | `private` | `float` | **dead field** — declared but unused. |
| `dynamicColTimer` | `private` | `float` | precise collision mode timer (=6 on start/pickup). |
| `fixedFramesSinceSpawn` | `private` | `float` | fixed frame counter (first 6 — kinematic). |
| `distanceCheckTimer` | `private` | `float` | distance check period (5–8 s). |
| `framesUntilDestroy` | `private` | `int` | destroy counter when out of zone (>10 → destroy). |

### `ItemRigidbody` Public Methods
| Method | Action |
|-------|----------|
| `GetBody()` | → twin's `Rigidbody`. |
| `GetShipItem()` | → `ShipItem` (visual). |
| `RegisterItem(ShipItem)` | link to visual. |
| `ResetPos()` | `isKinematic = true`, position/rotation = visual, `velocity = 0`. |
| `UpdateMass()` | recalculate mass (item.mass + crate/bottle/tea/salt/soup contents). |
| `EnterInventorySlot(Transform)` / `ExitInventorySlot()` | into/out of inventory (layer 5 / 2, colliders). |
| `EnterBox(...)` / `ExitBox()` / `GetCurrentBox()` | into/out of crate. |
| `GetCurrentInventorySlot()` | current slot (or null). |
| `EnterBoat()` / `ExitBoat()` | onto/off boat (reparent, `BoatMass.Add/RemoveItem`). |
| `ToggleCollider(bool state)` | colliders on/off; **always** `floater.enabled = false` (see note 43). |
| `ForceRigidbodyToWalkCol()` | press against boat walk collider. |

## Freeze / Unfreeze (How to Correctly Unfreeze Item)

Twin kinematics controlled by `flag2` in `FixedUpdate` (full table — note 43). Forced freeze — `debugForceKinematic`.

**Unfreeze item for floating** (without breaking pickup):
```csharp
var ir = shipItem.GetItemRigidbody();
ir.debugForceKinematic = false;     // remove forced kinematics
// DON'T touch visual, DON'T add components to visual
// body becomes dynamic automatically when: sold && !held && !attached && in zone && playing
```
- `debugForceKinematic` — **on `ItemRigidbody`**, not on `ShipItem`. Attempting `shipItem.debugForceKinematic` → **CS1061** (no such member).
- Don't set `debugForceKinematic = true` if floating needed (this freezes).
- Item **must be `sold = true`** (otherwise `!sold` → kinematic, note 43).

## Who Is Position Master (visual ↔ physics)

In `ItemRigidbody.FixedUpdate`, synchronization depends on state:

| State | Master | Code |
|-----------|--------|-----|
| `held` **or** `!sold` (in hand / shop) | **visual** → twin follows | `twin.position = visual.position` |
| `sold` && free in world (not held) | **twin (physics)** → visual follows | `visual.position = twin.position` |
| in inventory/crate | slot/crate | lerp to slot / `box.TransformPoint` |
| on boat (`onBoat`) | boat walk collider | `MoveRigidbodyToWalkCol` / `MoveItemToWalkColRigidbody` |

**Critical for modder:** for free purchased item ("floats in world" case) **master is physics twin**. Move/float **twin** (`GetBody()`/twin.transform), **never visual**. If you move visual (or add float/`Rigidbody` to visual) — `FixedUpdate` overwrites visual with twin position → conflict, and this **breaks pickup** (visual has trigger colliders `isTrigger` used for raycast pickup).

## Physics "Sleep" and Range

- **`idleTimer` is unused** (dead field) — don't look for logic in it.
- **Unity body sleep:** `rigidbody.IsSleeping()` accounted only for mesh-collider items (`meshCol != null && IsSleeping() && dynamicColTimer <= 0` → kinematic).
- **Range:** `outOfRange` = distance to `Camera.main` > **600** units. At `outOfRange`: `FixedUpdate` early exit (physics not managed); purchased item not on boat after >10 frames **destroyed** (`DestroyItem`).
- **First 6 fixed frames** after spawn — kinematic (`fixedFramesSinceSpawn < 6`).

## Access Cheat Sheet

```csharp
ShipItem item = go.GetComponent<ShipItem>();
yield return WaitUntilItemRigidbody(item);        // wait for twin (see above)

ItemRigidbody ir   = item.GetItemRigidbody();      // or item.itemRigidbodyC
Rigidbody     body = ir.GetBody();                 // twin's physics body
Transform     twin = ir.transform;                 // twin's transform
SimpleFloatingObject fl = /* private floater — only via patch/Harmony */;

// unfreeze:
ir.debugForceKinematic = false;
// position master for free sold item = twin (move twin, not visual)
```

> `floater` — private field of `ItemRigidbody`; can't get externally. Managed by game through `ToggleCollider` (disabled). For own buoyancy — own component on **twin** or patch (note 43).

## Practical Takeaways

1. Twin access: `shipItem.GetItemRigidbody()` (or `itemRigidbodyC`, public); `Rigidbody` — via `GetBody()`. Field `itemRigidbody` (Transform) — protected.
2. **Twin created in coroutine from Awake** → after `Instantiate` wait a frame / poll `itemRigidbodyC != null`.
3. `debugForceKinematic` — **on `ItemRigidbody`** (public), not on `ShipItem` (else CS1061). Unfreeze = `false` + `sold=true` + not held/not attached.
4. For free item **master = physics twin**; move/float twin, **never visual** (otherwise breaks pickup).
5. `idleTimer` — dead field; "sleep" = Unity `IsSleeping()` (mesh items) + culling beyond 600 units.
