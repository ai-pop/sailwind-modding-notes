# 64. Item Physics Model

> Dry facts about ALL components involved in item physics: buoyancy, inertia, collisions, mass.
> Based on Assembly-CSharp.dll decompilation (Sailwind v0.38).

---

## Architecture: Dual-GameObject Model

Every item exists as **TWO** GameObjects simultaneously:

```
ShipItem (visual)                       ItemRigidbody (physics)
├─ Collider: isTrigger = true           ├─ Rigidbody
├─ Renderer + LODGroup                   ├─ SimpleFloatingObject (Crest)
├─ Stores position/rotation              ├─ BoxCollider / MeshCollider / CapsuleCollider
└─ Layer: dynamic (2/5/26)               ├─ Subcolliders (ItemSubcollider)
                                         └─ Layer: 2 (IgnoreRaycast)
```

### Position Flow

| State | Who follows whom |
|-------|------------------|
| Held (`held != null`) | `itemRigidbody` → `ShipItem` position |
| On shelf (`sold == false`) | `itemRigidbody` → `ShipItem` position |
| Free (`sold && !held`) | `ShipItem` → `itemRigidbody` position |
| On boat, not held | `ShipItem` → `itemRigidbody` via walkCol |
| In inventory | Both → inventory slot position |

### Creation in `ItemRigidbody.Start()`

```csharp
rigidbody.drag = 1.2f;
rigidbody.angularDrag = item.mass * 0.1f;
rigidbody.isKinematic = true;
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;  // default 1.6
```

Colliders copied from `ShipItem` via `AddCollider()` with 3 FixedUpdate delay. If `MeshCollider` present → `collisionDetectionMode = 2` (Continuous), else `3` (ContinuousDynamic).

---

## SimpleFloatingObject (Crest Ocean System)

**Source code NOT in decompilation** — embedded in compiled Assembly-CSharp.dll as part of Crest asset.

### Known Parameters (set in ItemRigidbody.Start)

| Parameter | Value | Source |
|-----------|:-----:|--------|
| `_raiseObject` | `item.floaterHeight` (≈1.6) | `ShipItem.floaterHeight` |
| `_dragInWaterRotational` | `0.02` | Hardcoded |

### Inferred Behavior

1. Checks water height via Crest `Ocean` API
2. If below water — applies upward force
3. `_raiseObject` added to target height above water
4. At `_raiseObject = 1.6`, object centers ~1.6 units above water surface

---

## Alternative Buoyancy Systems

### Buoyancy (used by boats and large objects)

Grid-based sampling ("blobs"), Archimedes force per sample point.

| Field | Default | Description |
|--------|:------:|-------------|
| `SlicesX` | 2 | Sample points along X |
| `SlicesZ` | 2 | Sample points along Z (total 2×2=4) |
| `magnitude` | 2.0 | Buoyancy force multiplier |
| `dampCoeff` | 0.1 | Vertical velocity damping |
| `CenterOfMassOffset` | −1.0 | Center of mass offset downward |
| `interpolation` | 3 | Frame smoothing |
| `ChoppynessAffectsPosition` | false | Currents displace object |
| `WindAffectsPosition` | false | Wind pushes object |
| `xAngleAddsSliding` | false | Tilt creates sliding |
| `sink` | false | Sinking mode with extra force |
| `moreAccurate` | false | Bilinear water height interpolation |

**Per-blob logic:**
```
waterHeight = ocean.GetWaterHeightAtLocation2(x - choppy, z)
delta = magnitude × (blobWorldY − waterHeight)
// delta > 0: blob ABOVE water → force DOWN
// delta < 0: blob BELOW water → force UP
verticalVel = rigidbody.GetPointVelocity(blobWorldPos).y
force = −Vector3.up × (delta + dampCoeff × verticalVel)
rigidbody.AddForceAtPosition(force, blobWorldPos)
```

When `moreAccurate = true`: uses `GetWaterHeightAtLocation2` (bilinear). Otherwise: `GetWaterHeightAtLocation` (nearest neighbor) + lerp smoothing 0.5.

**Tilt sliding:** if `xAngleAddsSliding`:
- Tilt 5°–90° → forward acceleration
- Tilt 270°–355° → backward acceleration
- Max ±20 force units, 0.05/tick decay

### Boyant (simplest)

Direct Y position set without physics:
```
height = ocean.GetWaterHeightAtLocation2(x - choppy, z) + buoyancy
transform.position.y = height
```

---

## isKinematic Conditions (ItemRigidbody.FixedUpdate)

Rigidbody becomes kinematic (excluded from physics simulation) on ANY of:

| # | Condition | Code |
|:--|-----------|------|
| 1 | Held | `item.held != null` |
| 2 | Not purchased (on shelf) | `!item.sold` |
| 3 | Nailed | `item.nailed` |
| 4 | Wall-attached | `attached` |
| 5 | Sleeping / recovering / shipyard | `GameState.sleeping \|\| recovering \|\| currentShipyard != null` |
| 6 | In box or inventory | `currentBox != null \|\| currentInventorySlot != null` |
| 7 | Out of range (600m) | `outOfRange` |
| 8 | First 6 FixedUpdates | `fixedFramesSinceSpawn < 6` |
| 9 | Debug flags | `Debugger.kinematicItemsTimer > 0 \|\| debugForceKinematicBoat` |
| 10 | Force flag | `debugForceKinematic` |
| 11 | MeshCollider sleeping | `meshCol != null && sleeping && dynamicColTimer <= 0` |

**Consequence:** items on boat not held → kinematic. Position set via parent transforms (`MoveItemToWalkColRigidbody`), not physics.

---

## Collider Management

### dynamicColTimer

After releasing item — `collisionDetectionMode = 2` (Continuous) for 6 seconds. After — `3` (ContinuousDynamic) if no MeshCollider.

### isTrigger

Colliders become triggers when held, in box, in inventory, or attached.

### outOfRange

If distance to camera > 600m and not on boat:
- Counts frames (up to 10)
- Destroys item (`DestroyItem()`)
- Checks every 5–8 seconds

---

## Mass System

### Base Rules

- `ShipItem.mass` (kg) — set in prefab
- `Rigidbody.mass` updated via `UpdateMass()`
- `angularDrag = mass × 0.1`

### Modifiers

| Subclass | Mass Addition |
|----------|---------------|
| `ShipItemCrate` | `containedPrefab.mass × amount` |
| `ShipItemBottle` | `item.health` (fill level = liquid mass) |
| `ShipItemTea` | `amount × 0.1` |
| `ShipItemSalt` | `amount × 0.1` |
| `ShipItemSoup` | `currentWater + currentEnergy/20 + currentUncookedEnergy/20` |

### BoatMass.UpdateMass()

```csharp
totalMass = selfMass + partsMass
for each item on boat where inventorySlot == null:
  if item.held && player on this boat:
    totalMass += item.mass
    comShift += playerLocalPos × (item.mass / selfMass) × leverageMult
  else:
    totalMass += item.mass
    comShift += itemLocalPos × (item.mass / selfMass) × leverageMult
totalMass += 160  // player
comShift += playerLocalPos × (160 / selfMass) × leverageMult
for each mast sail:
  totalMass += GetSailMass(sail)
body.mass = totalMass
body.centerOfMass = keel.centerOfMass + comShift
```

### GetSailMass

```
junk/gaff: sailPower × 20 + sailPower × 20  (= sailPower × 40)
staysail:  sailPower × 20 + 0                (= sailPower × 20)
others:    sailPower × 20 + sailPower × 10   (= sailPower × 30)
```

---

## Decollision (PickupableItemCollisionChecker)

Component on `itemRigidbody`.

| Constant | Value |
|----------|:-----:|
| Push-out multiplier | 1.8 |
| Allowed drop threshold | 0.06 (penetration depth) |

**Algorithm:**
```
decollisionVector = Σ(normal × penetration × 1.8) across all collisions
return in itemRigidbody local space
```

When held: if `penetration >= 0.06` → red outline, drop disallowed (for big items or with Throw held).

---

## RigidbodyDirectionalDrag

Per-frame velocity scaling in local axes:

```csharp
localVel = transform.InverseTransformDirection(rb.velocity)
localVel.x *= x
localVel.y *= y
localVel.z *= (localVel.z > 0) ? zForward : zBack
rb.velocity = transform.TransformDirection(localVel)
```

Values `x, y, zForward, zBack` set in inspector, must be ≤ 1.0.

---

## HullDrag

```csharp
addedDrag = boat._dragInWaterForward
addedSideDrag = boat._dragInWaterRight
speed = rb.velocity.magnitude
boat.addedHullDrag = speed² × addedDrag + boat._dragInWaterForward × 2
boat.addedSideDrag = speed × addedSideDrag + boat._dragInWaterRight × baseSideDragMult
```

---

## ShiftingRigidbody (FloatingOrigin)

On FloatingOrigin shift:
1. `PrepareForShifting()`: saves velocity/angularVelocity, sets isKinematic
2. `RestoreMomentum()`: restores via coroutine with frame delay

---

## Constants Reference

| Constant | Value | Source |
|----------|:-----:|--------|
| drag (linear) | 1.2 | `ItemRigidbody.Start` |
| angularDrag | `mass × 0.1` | `ItemRigidbody.Start` |
| `_dragInWaterRotational` | 0.02 | `ItemRigidbody.Start` |
| `_raiseObject` | `floaterHeight` (≈1.6) | `ItemRigidbody.Start` |
| `floaterHeight` default | 1.6 | `ShipItem` |
| Buoyancy.magnitude default | 2.0 | `Buoyancy` |
| Buoyancy.dampCoeff default | 0.1 | `Buoyancy` |
| Decollision multiplier | 1.8 | `PickupableItemCollisionChecker` |
| Decollision threshold | 0.06 | `PickupableItemCollisionChecker` |
| Out-of-range distance | 600 m | `ItemRigidbody.FixedUpdate` |
| dynamicColTimer | 6 sec | `ItemRigidbody.SetDynamicColTimer` |
| Player mass in BoatMass | 160 | `BoatMass.UpdateMass` |
| Gravity | Unity default (−9.81) | `Physics.gravity` |
| collisionDetectionMode (mesh) | 2 (Continuous) | `ItemRigidbody.AddCollider` |
| collisionDetectionMode (primitive) | 3 (ContinuousDynamic) | `ItemRigidbody.AddCollider` |

---

*Extracted from Assembly-CSharp.dll decompilation (Sailwind v0.38).*
*SimpleFloatingObject — Crest Ocean System component; source code unavailable.*
