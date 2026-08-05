# 88. Anchor Physics Exclusion: Why Anchors Cannot Be Thrown Like Normal Cargo

A technical analysis of user feedback on item physics mods (request on SailwindItemPhysics: *"it doesnt affect the anchor, that would be nice to throw around"*). This note explains why an anchor (`Anchor : PickupableItem`) differs architecturally from standard twin-model items (`ShipItem` / `ItemRigidbody`), and why applying general item physics patches to an anchor breaks mooring and ship stability.

Related to [Note 29](29-anchor-mooring-ropes.md), [Note 44](44-itemrigidbody-field-map-contract.md), and [Note 85](85-anchor-joint-physics-seabed-setting-and-stowed-mass-reduction.md).

---

## 1. Architectural Differences Between Anchors and Normal Cargo

In Sailwind v0.38 (`Assembly-CSharp.dll`), an anchor is not a standard portable cargo item:

| Attribute | Normal Cargo (`ShipItem`) | Anchor (`Anchor : PickupableItem`) |
|---|---|---|
| **Physics Model** | Twin model: Visual GO (`kinematic`) + Twin GO (`ItemRigidbody`, dynamic) | **Single Body:** `Anchor` is itself a dynamic `Rigidbody` with no visual/twin separation |
| **Ship Coupling** | Free body, retained within the deck zone via `BoatLocalItems` | **Hard Coupling:** Permanently coupled to the boat via a `ConfigurableJoint` (`joint.connectedBody`) |
| **Buoyancy** | Evaluated via `SimpleFloatingObject` (Crest) | **Always Sinks:** Lacks any buoyancy evaluation component |
| **Stowed Mass** | Constant ([Note 61](61-item-catalog-mass-table-units.md)) | **Dynamic:** Decays down to **0.2 kg** when stowed at the bow (`limit < 1f`, [Note 85](85-anchor-joint-physics-seabed-setting-and-stowed-mass-reduction.md)) |

---

## 2. Why General Physics Patches Break Anchors

If an item physics mod attempts to process `Anchor` under the same rules as crates or barrels, collisions occur simultaneously across three subsystems:

```
[ Mod attempts to apply ThrowImpulse / Twin Override to Anchor ]
                        │
                        ├──────────────────────────────────────┐
                        ▼                                      ▼
        [ Conflict with ConfigurableJoint ]        [ Conflict with 0.2 kg Mass ]
        Throw velocity exceeds joint.linearLimit   At throw instant, mass = 0.2 kg;
        (rope scope), snapping the rope taut       impulse launches anchor into space,
                        │                          then rope yanks the ship's bow
                        ▼
          [ MOORING BREAKDOWN & HULL CAPSIZE ]
          Winch locks up; anchor fails to set on seabed
```

### 2.1. Conflict With Rope Scope Limit (`joint.linearLimit`)
The anchor rope restricts the anchor's maximum distance from the ship (`joint.linearLimit.limit`). If a mod applies a throw impulse over 12 frames (`ThrowImpulse.Plant`, [Note 57](57-vanilla-drop-throw-mechanics.md)), the anchor travels until it exhausts rope scope, at which point the `ConfigurableJoint` snaps taut. This jerk applies an enormous reaction impulse to `connectedBody`—pulling the ship's bow underwater.

### 2.2. Conflict With Stowed Mass (`0.2 kg`)
When stowed at the bow hawsepipe, an anchor weighs only **0.2 kg** ([Note 85](85-anchor-joint-physics-seabed-setting-and-stowed-mass-reduction.md)). If a mod evaluates throw force or momentum using the current `rb.mass = 0.2f`, the anchor launches at unnatural speeds.

### 2.3. Breaking Seabed Setting Criteria (`SetAnchor`)
To set in the seabed (`set = true`), an anchor must lie horizontally (`Angle > 60°`) and come to rest (`!audio.isPlaying`). Forcing a velocity override from a mod prevents the anchor from resting on the bottom.

---

## 3. Practical Solution for Safe Anchor Throwing

To allow players to heave anchors overboard without upsetting vessel physics, a mod must implement a **dedicated filter for `Anchor`**:

```csharp
public static void TryThrowAnchorSafe(Anchor anchor, Vector3 handVelocity)
{
    Rigidbody rb = anchor.GetComponent<Rigidbody>();
    ConfigurableJoint joint = anchor.GetComponent<ConfigurableJoint>();

    // 1. Ensure anchor is not set and rope scope is at least 2 meters
    if (anchor.IsSet() || joint.linearLimit.limit < 2f)
        return;

    // 2. Restore nominal mass before throwing (otherwise 0.2 kg launches too fast)
    if (rb.mass < 10f)
        rb.mass = 100f; // Restore nominal anchor mass

    // 3. Clamp throw speed so the anchor cannot snap the rope taut
    float maxSafeSpeed = Mathf.Min(8f, joint.linearLimit.limit * 0.8f);
    Vector3 launch = Vector3.ClampMagnitude(handVelocity * 1.2f, maxSafeSpeed);

    // 4. Apply a single-frame impulse (ForceMode.VelocityChange), NEVER a 12-frame ThrowImpulse
    rb.AddForce(launch, ForceMode.VelocityChange);
}
```

### 3.1. Safety Checklist for Modders
1. **Exclude `Anchor` From General Twin Physics:** Check `item is Anchor` or check for a `ConfigurableJoint` before applying twin-body logic. An anchor does not require `BoatZoneGhost`, `ShiftingRigidbody`, or modded buoyancy.
2. **Use Single-Frame `ForceMode.VelocityChange`:** Never overwrite an anchor's `rb.velocity` over multiple frames; allow the `ConfigurableJoint` to naturally arrest velocity as the rope pays out.
3. **Check Available Scope:** Prevent anchor throwing if available rope scope (`GetRopeLength()`) is less than 2 meters.
