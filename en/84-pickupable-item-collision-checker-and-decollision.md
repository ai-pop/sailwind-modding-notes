# 84. Item Collision Checking and Decollision System (`PickupableItemCollisionChecker`)

A technical breakdown of held item collision checking (`PickupableItemCollisionChecker`) and trigger overriding (`ItemTriggerOverride`) in Sailwind v0.38 (`Assembly-CSharp.dll`). This note is directly applicable to **modding item physics, cargo placement, and deck obstacle avoidance**.

Related to [Note 47](47-item-holding-pickup-flow.md), [Note 54](54-go-pointer-big-item-decollision.md), and [Note 58](58-clickability-layers-manual-held-break.md).

---

## 1. Held Item Collision Checking (`PickupableItemCollisionChecker`)

When the player holds an item (`item.held != null`), the visual GameObject follows the camera/cursor. To prevent dropped items from clipping inside walls or decks, `PickupableItemCollisionChecker` tracks intersecting colliders via `collidedCols`.

### 1.1. Decollision Vector Evaluation (`GetDecollision`)

If decollision is enabled (`Settings.enableDecol == true`), `GetDecollision()` evaluates an escape vector out of intersecting geometry:

```csharp
public Vector3 GetDecollision()
{
    if (!Settings.enableDecol)
        return Vector3.zero;

    decollisionVector = Vector3.zero;
    Vector3 dir = default(Vector3);
    float dist = default(float);

    foreach (Collider collidedCol in collidedCols)
    {
        Physics.ComputePenetration(
            GetComponent<Collider>(), transform.position, transform.rotation,
            collidedCol, collidedCol.transform.position, collidedCol.transform.rotation,
            ref dir, ref dist
        );
        decollisionVector += dir * dist * 1.8f;
    }
    return transform.InverseTransformVector(decollisionVector);
}
```

#### The 1.8× Over-Relaxation Multiplier
Notice the `1.8f` scaling coefficient in `dir * dist * 1.8f`:
- Rather than merely displacing the item by the exact penetration depth `dist`, the engine applies a **1.8× over-relaxation multiplier**, pushing the item 80% further out of the obstacle.
- This ensures that on the frame following release, the item's physical twin collider is guaranteed to reside completely outside deck or wall geometry, preventing explosive PhysX contact spikes.

---

## 2. Obstructed Drop Tolerance (`allowObstructedDropping`)

Every frame in `Update()`, the system evaluates the maximum penetration depth (`currentDecolDistance`) across all colliders in `collidedCols`:

```csharp
public void Update()
{
    if (item.held == null || item.GetCurrentInventorySlot() > -1)
    {
        collisions = 0;
        return;
    }

    UpdateDecolDistance();
    allowObstructedDropping = currentDecolDistance < 0.06f;

    bool enableRedOutline = false;
    if (collisions > 0)
    {
        if (item.big && !allowObstructedDropping)
            enableRedOutline = true;
    }
    ...
}
```

### 2.1. The 6-Centimeter Rule (`0.06f`)
| Penetration Depth | Flag State | Drop / Placement Behavior |
|---|:--:|---|
| `< 0.06 m` (6 cm) | `allowObstructedDropping = true` | Item **can be dropped or placed** even if its edge slightly intersects a deck, table, or wall. Red placement outline does not illuminate. |
| `≥ 0.06 m` | `allowObstructedDropping = false` | For large items (`item.big == true`), a red outline illuminates, blocking placement and dropping. |

> **Why?** Sailwind ship decks feature curved surfaces and wooden planking. Without a 6-centimeter tolerance threshold, players would be unable to stack crates cleanly on uneven decks.

---

## 3. Delayed Trigger Copying (`ItemTriggerOverride`)

To optimize interaction triggers, `ItemTriggerOverride` copies physical `BoxCollider` dimensions into a trigger collider after a 3-frame delay:

```csharp
private IEnumerator LoadAfterDelay()
{
    yield return new WaitForEndOfFrame();
    yield return new WaitForEndOfFrame();
    yield return new WaitForEndOfFrame();

    BoxCollider targetBox = item.GetComponent<BoxCollider>();
    targetBox.center = col.center;
    targetBox.size = col.size;
    col.enabled = false;
    ...
}
```

**Reason for 3-Frame Delay:** During prefab instantiation (`Awake`), scaling scripts or custom deck modifiers may resize `item`. Waiting 3 `WaitForEndOfFrame` steps guarantees that the trigger captures final, post-initialization item dimensions.

---

## 4. Practical Modding Conclusions

1. **Adhering to the `0.06f` Threshold:** If your mod implements grid snapping or magnetic cargo placement, ensure final deck intersection depths never exceed **0.05 m (5 cm)**. Otherwise, `PickupableItemCollisionChecker` will trigger red outline rejection.
2. **Accounting for 1.8× Over-Relaxation:** When patching item release or drop algorithms, keep in mind that vanilla decollision pushes items almost twice as far as their actual penetration depth. This can displace crates off narrow shelves.
3. **3-Frame Geometry Initialization Delay:** When spawning items programmatically, never read `ItemTriggerOverride` trigger dimensions on the frame of `Instantiate()`. Always yield for at least 3 `WaitForEndOfFrame()` steps.
