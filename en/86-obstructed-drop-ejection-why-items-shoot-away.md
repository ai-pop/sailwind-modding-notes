# 86. Obstructed Drop Ejection (Red Outline): Why Items Shoot Away and How Vanilla Reshuffles Them

A technical investigation into one of the most common issues in item physics mods (including user feedback on SailwindItemPhysics): **why dropping an item while obstructed (red outline, key `T`/`F`) causes it to instantly shoot away at high velocity**, whereas vanilla gameplay smoothly re-shuffles items into tight stacks.

Related to [Note 54](54-go-pointer-big-item-decollision.md), [Note 59](59-velocity-zeroing-release-frame-pose.md), [Note 60](60-held-readers-throw-key-binding.md), and [Note 84](84-pickupable-item-collision-checker-and-decollision.md).

---

## 1. How Vanilla Item "Re-Shuffling" Works

When the player holds a cargo crate and attempts to squeeze it into a tight gap between other boxes (or force-drops it via `T`/`F` while the red outline is illuminated), vanilla Sailwind executes the following sequence:

```csharp
[ Player presses Drop/Throw (T / F) while obstructed ]
                        │
                        ▼
      [ PickupableItem.DropItem() ]
  item.held = null; visual.layer = 0 (Default)
                        │
                        ▼
   [ ItemRigidbody.FixedUpdate() (frames 1..10) ]
   twinCollider remains isTrigger / kinematic on release frame
   Physics.ComputePenetration → gentle geometric displacement
                        │
                        ▼
    [ Crate smoothly "re-shuffles" onto shelf / into stack ]
```

### 1.1. Why Vanilla Crates Never Eject
1. **No Solid Contact on the Release Frame:** In vanilla's `ItemRigidbody` ([Note 59](59-velocity-zeroing-release-frame-pose.md)), the held item's physical twin collider remains `isTrigger = true` and/or `isKinematic = true` during holding and on the immediate release step.
2. **Zero Initial Linear Velocity:** Upon `DropItem()`, vanilla explicitly zeroes out linear and angular momentum (`rigidbody.velocity = Vector3.zero`).
3. **Smooth `ComputePenetration` Recovery:** Depenetration occurs via geometric offset vector addition (`decollisionVector`, [Note 84](84-pickupable-item-collision-checker-and-decollision.md)), rather than through solid-body PhysX contact impulses. As a result, the crate simply glides into the nearest valid pocket.

---

## 2. Why Physics Mods Cause High-Velocity Ejection ("Shoot Away")

Item physics mods (such as SailwindItemPhysics v4.x/v5.x) introduce dynamic in-hand holding ("physics in hands") and momentum throw impulses (`MomentumThrow` / `ThrowImpulse`). When dropping an item in an obstructed space, this triggers an explosive solver collision:

```
[ Mod: Drop item while overlapping geometry (red outline) ]
                        │
                        ├──────────────────────────────┐
                        ▼                              ▼
            [ Twin collider is SOLID ]      [ ThrowImpulse is applied ]
            (col.isTrigger = false)        rb.velocity = handVelocity
                        │                              │
                        └──────────────┬───────────────┘
                                       ▼
                     [ PhysX: Penetration Depth Solver ]
                     Deep solid-body overlap + forced velocity
                                       │
                                       ▼
              [ HIGH-VELOCITY EJECTION ("SHOOT AWAY") ]
              Impulse spike > 6000 N → crate launches overboard
```

### 2.1. Two Sources of the Explosive Impulse
| Source | Technical Cause | PhysX Outcome |
|---|---|---|
| **Solid Twin Collider (`isTrigger = false`)** | The mod forces the twin collider solid so held items bump against walls (`HoldSolidColliders`). If dropped while already intersecting 5–10 cm into an adjacent crate, PhysX instantaneously applies depenetration force $F = \frac{\Delta x}{\Delta t^2} \cdot m$. | For a 30 kg crate with 0.1 m penetration, the restoring force exceeds $6000$ N, firing the crate like a cannonball. |
| **Throw Momentum Injection (`ThrowImpulse.Plant`)** | The mod tracks hand velocity (`handVelocity`) and forcibly re-asserts `rb.velocity = _vel` for 12 fixed frames post-release. | If the item is intersecting static geometry, asserting velocity prevents the PhysX solver from relaxing the contact, causing an infinite reaction force spike. |

---

## 3. Practical Solution for Item Physics Mods

To preserve realistic throwing in open space while restoring vanilla's smooth re-shuffling and eliminating ejection during obstructed drops, the mod must inspect penetration state on the release frame:

```csharp
public static void HandleSafeRelease(ShipItem item, Rigidbody rb, Vector3 handVelocity)
{
    // 1. Query vanilla penetration depth checker
    var checker = item.GetComponent<PickupableItemCollisionChecker>();
    bool isObstructed = false;
    
    if (checker != null && checker.collisions > 0)
    {
        // Check if obstructed dropping tolerance is violated
        if (!checker.allowObstructedDropping || item.big)
            isObstructed = true;
    }

    if (isObstructed)
    {
        // VANILLA RESHUFFLE MODE:
        // 1. Suppress ThrowImpulse (abort throw impulse when obstructed)
        // 2. Temporarily keep twin collider isTrigger=true for 10 fixed frames
        rb.velocity = Vector3.zero;
        rb.angularVelocity = Vector3.zero;
        item.StartCoroutine(SoftResolveCoroutine(rb, item));
    }
    else
    {
        // Open space — plant true momentum throw impulse
        ThrowImpulse.Plant(rb, handVelocity);
    }
}
```

### 3.1. Checklist for Fixing the "Shoot Away" Bug
1. **Suppress `ThrowImpulse` on Red Outline:** If an item is released while intersecting geometry (`collisions > 0` and `!allowObstructedDropping`), dropping must release the item at zero velocity without injecting hand momentum.
2. **Soft Collider Transition (`SoftResolve`):** When releasing an obstructed item, delay switching `col.isTrigger = false` by `0.15–0.25 s` (8–12 `FixedUpdate` steps). This gives vanilla's geometric decollision time to push the crate out before solid PhysX contacts engage.
3. **Cap Bounce Velocity:** Apply a temporary clamp `rb.maxLinearVelocity = 2.5f` during the first 15 physics steps following item release to prevent solver spikes.
