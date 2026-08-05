# 89. Worked Example: Safe Obstructed Drop and Throw Resolver for Physics Mods

A practical modding guide and reference C# implementation (recipe for BepInEx mods such as SailwindItemPhysics) for eliminating high-velocity ejection during obstructed item drops ("shoot away", red outline) while preserving accurate ballistic throwing in open space.

Derived from research findings in [Note 84](84-pickupable-item-collision-checker-and-decollision.md), [Note 86](86-obstructed-drop-ejection-why-items-shoot-away.md), and [Note 88](88-anchor-physics-exclusion-why-anchors-require-separate-handling.md).

---

## 1. The Challenge: Why a Unified Resolver Is Required

Item physics mods frequently intercept item release events (`OnDrop` / `DropItem`) to inject player hand velocity (`ThrowImpulse`) into the dynamic twin body. Without checking for deck or stack overlaps, this creates two major bugs:
1. Crates dropped into crowded stacks shoot overboard due to solid-body PhysX penetration spikes.
2. Attempting to throw an anchor (`Anchor`) snaps the `ConfigurableJoint` rope limit.

---

## 2. Complete Reference Resolver (`SafeDropAndThrowResolver.cs`)

```csharp
using System.Collections;
using UnityEngine;

namespace SailwindModdingRecipes
{
    /// <summary>
    /// Unified release handler for item physics mods. Ensures vanilla-like smooth
    /// geometric re-shuffling when dropping into tight spaces and accurate
    /// ballistic throwing in open airspace.
    /// </summary>
    public static class SafeDropAndThrowResolver
    {
        // Safety penetration threshold for detecting obstructed drops (in meters)
        private const float ObstructedThreshold = 0.045f;

        /// <summary>
        /// Call immediately after vanilla executes DropItem() / release cleanup.
        /// </summary>
        public static void ResolveRelease(ShipItem item, Rigidbody rb, Vector3 handVelocity)
        {
            if (item == null || rb == null)
                return;

            // 1. Dedicated filter for anchors (Anchor : PickupableItem)
            if (item is Anchor anchor)
            {
                ResolveAnchorThrowSafe(anchor, rb, handVelocity);
                return;
            }

            // 2. Query whether the item is in an obstructed / overlapping state
            bool isObstructed = IsItemObstructed(item);

            if (isObstructed)
            {
                // VANILLA RESHUFFLE MODE:
                // Abort throw momentum injection and clamp velocities so ComputePenetration can work
                rb.velocity = Vector3.zero;
                rb.angularVelocity = Vector3.zero;
                item.StartCoroutine(SoftReshuffleCoroutine(item, rb));
            }
            else
            {
                // OPEN AIRSPACE:
                // Apply player hand velocity (true momentum throw impulse)
                ApplyThrowMomentum(rb, handVelocity);
            }
        }

        private static bool IsItemObstructed(ShipItem item)
        {
            var checker = item.GetComponent<PickupableItemCollisionChecker>();
            if (checker == null || checker.collisions == 0)
                return false;

            // Obstructed if red outline is lit (!allowObstructedDropping or multiple collisions)
            return !checker.allowObstructedDropping || checker.collisions > 1;
        }

        /// <summary>
        /// Temporarily holds the twin collider in trigger mode for 10 PhysX steps,
        /// allowing geometric decollision to push the crate out smoothly without solid-contact explosions.
        /// </summary>
        private static IEnumerator SoftReshuffleCoroutine(ShipItem item, Rigidbody rb)
        {
            Collider col = rb.GetComponent<Collider>();
            if (col == null)
                yield break;

            bool wasTrigger = col.isTrigger;
            col.isTrigger = true; // Suspend solid PhysX contact solvers

            for (int i = 0; i < 10; i++)
            {
                rb.velocity = Vector3.zero; // Damp out spurious impulses
                yield return new WaitForFixedUpdate();
            }

            // Restore solid collider state after geometric displacement completes
            col.isTrigger = wasTrigger;
            rb.WakeUp();
        }

        private static void ApplyThrowMomentum(Rigidbody rb, Vector3 handVelocity)
        {
            if (handVelocity.magnitude < 0.8f)
                return; // Movement too slow — standard vertical drop at feet

            rb.velocity = handVelocity;
            rb.useGravity = true;
        }

        private static void ResolveAnchorThrowSafe(Anchor anchor, Rigidbody rb, Vector3 handVelocity)
        {
            ConfigurableJoint joint = anchor.GetComponent<ConfigurableJoint>();
            if (anchor.IsSet() || joint == null || joint.linearLimit.limit < 2f)
                return;

            if (rb.mass < 10f)
                rb.mass = 100f; // Restore nominal anchor mass

            float maxSafeSpeed = Mathf.Min(6f, joint.linearLimit.limit * 0.7f);
            Vector3 safeLaunch = Vector3.ClampMagnitude(handVelocity, maxSafeSpeed);

            rb.AddForce(safeLaunch, ForceMode.VelocityChange);
        }
    }
}
```

---

## 3. Integrating the Resolver Into a Harmony Patch

Invoke `SafeDropAndThrowResolver.ResolveRelease` within your item drop postfix patch (or within your state monitor):

```csharp
[HarmonyPatch(typeof(PickupableItem), nameof(PickupableItem.DropItem))]
public static class DropItem_Patch
{
    public static void Postfix(PickupableItem __instance)
    {
        ShipItem item = __instance as ShipItem;
        if (item == null)
            return;

        Rigidbody twinRb = GetItemRigidbody(item);
        Vector3 handVel = GetMeasuredHandVelocity(item);

        SafeDropAndThrowResolver.ResolveRelease(item, twinRb, handVel);
    }
}
```

### 3.1. Architectural Benefits
1. **Eliminates "Shoot Away":** Obstructed crates smoothly re-shuffle into tight stacks thanks to the 10-frame `SoftReshuffleCoroutine` in `isTrigger = true` mode.
2. **Safe Anchor Heaving:** Anchor throwing is capped by available rope scope and applied via `ForceMode.VelocityChange` without breaking the `ConfigurableJoint`.
3. **Preserves Ballistic Throwing:** In open airspace, items fly naturally along the player's hand velocity trajectory.
