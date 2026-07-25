# 66. Item Rotation Freeze — All Mechanisms

> Every code path that locks, freezes, or suppresses item rotation.
> Critical for physics mods — these are silent killers. Complements note 64 (Physics Model).

---

## 1. MeshCollider Sleep → Full Kinematic Freeze

**Location:** `ItemRigidbody.FixedUpdate()` line ~521

```csharp
if (meshCol && rigidbody.IsSleeping() && dynamicColTimer <= 0f)
    flag2 = true;   // → isKinematic = true → ROTATION + POSITION FROZEN
```

**Trigger chain:**
1. Item with MeshCollider enters water
2. Buoyancy dampens → Rigidbody.Sleep() activates
3. `dynamicColTimer` counts down from 6s (set on release)
4. After 6s of sleep: `isKinematic = true`
5. Item becomes a statue — angular velocity zeroed by Unity

**Affected:** All items with complex collider geometry. Box/Capsule collider items NOT affected.

---

## 2. HangableItem.LateUpdate — Axis Lock

```csharp
if (currentHook != null)
{
    Vector3 ea = transform.eulerAngles;
    if (lockX) ea.x = rotX;    // ← FREEZES X
    if (lockZ) ea.z = rotZ;    // ← FREEZES Z
    transform.eulerAngles = ea;
}
```

Defaults: `lockX=true, lockZ=true`. Only Y-rotation (swinging) is free.

---

## 3. attached=true → Kinematic

Paths that set `attached=true`:
| Path | Location |
|------|----------|
| `wallAttachment` prefab init | `ShipItem.LoadAfterDelay()` |
| `CrateInventory.InsertItem()` | line ~64 |
| `HangableItem.ConnectJoint()` | line ~68 |

---

## 4. GameState.sleeping → Global Kinematic

```csharp
if (!GameState.playing || GameState.sleeping || GameState.recovering
    || GameState.inBed || GameState.currentShipyard)
    flag2 = true;   // ALL items kinematic during sleep
```

Post-wake: items become non-kinematic but angular velocity was zeroed.

---

## 5. angularDrag = mass × 0.1

| Mass | angularDrag |
|:----:|:-----------:|
| 0.8 kg | 0.08 |
| 5.0 kg | 0.5 |
| 14 kg | 1.4 |

Below 1.0 = nearly undamped rotation. Opposite of freeze but looks like freeze if no initial spin.

---

## 6. ResetPos() — Velocity Zeroed, Angular Preserved

```csharp
rigidbody.velocity = Vector3.zero;
// angularVelocity NOT zeroed
```

Item may keep spinning after being placed down.

---

## 7. CrateInventory.LateUpdate — Rotation Lock

```csharp
item.transform.rotation = transform.rotation;  // Every frame, every contained item
```

---

## 8. Crest _dragInWaterRotational = 0.02

Vanilla floater provides near-zero rotational drag in water.

---

## Summary

| # | Mechanism | Effect | Delay |
|:--|-----------|:------:|:-----:|
| 1 | MeshCollider sleep → kinematic | **Full freeze** | ~6s |
| 2 | HangableItem LateUpdate | **Lock X+Z** | Instant |
| 3 | attached=true | **Full freeze** | Instant |
| 4 | GameState.sleeping | **Full freeze** | Instant |
| 5 | angularDrag=mass×0.1 | ~No damping | Instant |
| 6 | ResetPos preserves angularVel | Spin survives | On drop |
| 7 | CrateInventory LateUpdate | **Lock to crate** | Every frame |
| 8 | Crest _dragInWaterRotational | ~No damping | Instant |

---

*Extracted from Assembly-CSharp.dll (Sailwind v0.38).*
