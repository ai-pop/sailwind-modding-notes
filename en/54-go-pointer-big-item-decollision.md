# 54. GoPointer big-item decollision: full algorithm

Breakdown of the `GoPointer.LateUpdate` decollision algorithm for `big`-items — answer to request E4. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 52 (layers), 53 (EnterBoat/ExitBoat), 33 (pickup).

## Decollision fields in GoPointer

| Field | Type | Content |
|------|-----|-----------|
| `bigItemLocalPos` | `Vector3` | Target big-item position in **camera local coordinates** (forward × holdDistance). |
| `decolLocalPos` | `Vector3` | **Actual big-item position** in camera local coordinates — accounting for decollision (pushed away from walls). Initialized = `bigItemLocalPos` on PickUp. |
| `bigItemLocalRot` | `Quaternion` | Big-item rotation relative to camera (scroll-rotation). |
| `bigItemLocked` | `bool` | (not seen in LateUpdate — possibly legacy field) |
| `bigItemLookTimer` | `float` | Timer for decollision change delay |
| `currentBigItemLook` | `ShipItem` | Item being "looked at" as big |
| `lastPointerRot` | `Quaternion` | Previous GoPointer rotation — for rotating decolLocalPos on camera turn |

## Full big-item algorithm in `GoPointer.LateUpdate`

```csharp
// GoPointer.LateUpdate — big item branch (verbatim key lines)
// Item already heldItem.big == true

float num2 = Mathf.InverseLerp(throwDelay, 1f, currentThrowPower);

// 1. Set position/rotation from decolLocalPos:
Quaternion rotation = transform.rotation * bigItemLocalRot;
heldItem.transform.position = transform.TransformPoint(decolLocalPos);
heldItem.transform.rotation = rotation;

// 2. Push back when preparing throw:
heldItem.transform.Translate(transform.forward * (0f - num2) * 0.4f, Space.World);

// 3. Scroll rotation:
heldItem.transform.Rotate(transform.right, heldItem.heldRotationOffset, Space.World);

// 4. DECOLLISION — ComputePenetration:
Vector3 decollision = heldItem.colChecker.GetDecollision();
decollection = heldItem.transform.TransformVector(decollection); // world-space

// 5. Convert to camera local coordinates:
Vector3 val = transform.InverseTransformPoint(heldItem.transform.position + decollection);

// 6. Limit checks (if decollision pushes too far):
if (val.z < 0.6f)         { val = decolLocalPos; Debug.Log("Close decol limit"); }
else if (val.z < -1.4f)    { val = decolLocalPos; Debug.Log("Decol limit"); }
else if (Mathf.Abs(val.x) > 1.4f) { val = decolLocalPos; Debug.Log("Decol limit"); }
else if (Mathf.Abs(val.y) > 1.4f) { val = decolLocalPos; Debug.Log("Decol limit"); }

// 7. Smooth blending:
decolLocalPos = Vector3.Lerp(decolLocalPos, val, Time.deltaTime * 3f);

// 8. Rotation on camera turn:
float num3 = Quaternion.Angle(transform.rotation, lastPointerRot);
decolLocalPos = Vector3.Lerp(decolLocalPos, bigItemLocalPos, Time.deltaTime * num3 * 1.2f);
lastPointerRot = transform.rotation;

// 9. Red outline:
if (heldItem.colChecker.collisions > 0) { /* (empty block) */ }
```

**No `while` loops!** Decollision is single-pass: `GetDecollision()` → Lerp. **While-hang is impossible in this code.**

## `PickupableItemCollisionChecker` — details

```csharp
// PickupableItemCollisionChecker.cs — key methods

// OnTriggerEnter: if other not isTrigger, not own item GO, not Boat tag → collisions++, add to list
// OnTriggerExit:  if other not isTrigger, not own item GO, not Boat tag → collisions--, remove from list

// GetDecollision():
//   foreach collidedCol in collidedCols:
//     Physics.ComputePenetration(this.collider, this.pos, this.rot, collidedCol, col.pos, col.rot, ref dir, ref dist)
//     decollisionVector += dir * dist * 1.8f
//   return transform.InverseTransformVector(decollectionVector)

// Update():
//   if held && not in inventory → UpdateDecolDistance() + allowObstructedDropping = currentDecolDistance < 0.06f
//   else → collisions = 0
//   enableRedOutline = (collisions>0 && big && !allowObstructedDropping) || (collisions>0 && InputName8 && !allowObstructedDropping)
```

### Critical details for the mod

1. **`OnTriggerEnter/Exit`** — on **visual GO** of ShipItem (trigger colliders). These are `isTrigger=true` colliders on the visual.
2. **Filter `!other.CompareTag("Boat")`:** decollision **ignores the boat** (tag `Boat`). So vanilla decollision **doesn't count** collision with the boat → `collisions` not incremented → red outline not shown → decollision doesn't push away from boat.
3. **`ComputePenetration`** is called for **visual-collider** (isTrigger=true) vs collidedCols (non-trigger). `ComputePenetration` works **regardless** of isTrigger — it checks geometric intersection of two colliders. But: **collidedCols** are only non-trigger colliders (filter in OnTriggerEnter: `!other.isTrigger`). Trigger colliders of the boat **don't enter** collidedCols.

> **Key conclusion:** vanilla decollision **excludes the boat** (tag `Boat`) and **excludes trigger colliders** from collidedCols. Twin-colliders of the item (on layer 2, isTrigger=true in vanilla) — also trigger → don't enter collidedCols. Mod makes twin non-trigger → **twin collider enters OnTriggerEnter of other items** — but **Boat tag excludes** the boat.

## Does the item "see itself"?

`OnTriggerEnter`: condition `(Object)(object)((Component)other).gameObject != (Object)(object)((Component)item).gameObject` — **excludes own visual GO**. But twin GO — **different gameObject** (with name `itemName:ItemRigidbody`). Twin-collider (non-trigger) **is not excluded** from collidedCols.

> **If twin-collider is non-trigger and overlaps visual trigger-collider of the same item** → twin enters collidedCols → ComputePenetration visual vs twin → decollision pushes item away from itself. This is unlikely (visual and twin usually far apart when held), but with certain geometries — possible.

## Practical conclusions

1. **Decollision has no while-loops**: `GetDecollision()` is single-pass over collidedCols, Lerp in GoPointer.LateUpdate. **Hanging from decollision is impossible.**
2. **Decollision excludes the boat** (tag `Boat`) → vanilla held big-item **is not pushed away from the boat**.
3. **Mod with non-trigger twin:** twin-collider can enter OnTriggerEnter (visual-side) as non-trigger → collisions++ → decollision tries to push away from twin of the item itself → **potential self-interaction bug**.
4. **Crash is not from decollision, but from `OnCollisionEnter`** on twin (non-trigger) → `ItemRigidbody.OnCollisionEnter` → `ItemCollisionSoundPlayer.PlayWoodColSound` — doesn't crash (AudioSource pool), but **BoatImpactSoundsCollider.OnCollisionEnter → BoatImpactSounds.Impact → BoatDamage.Impact** — this is the potential crash path (see note 55).
5. Decollision limits: z < 0.6 / z < -1.4 / |x| > 1.4 / |y| > 1.4 → fallback to `decolLocalPos` (no push). When item is inside boat's CapsuleCollider — ComputePenetration can return a large vector → limit fallback → item "freezes" at last safe position.
