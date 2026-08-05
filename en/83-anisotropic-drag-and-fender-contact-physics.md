# 83. Anisotropic Drag (`RigidbodyDirectionalDrag`) and Fender Contact Physics (`Fender`)

A technical breakdown of the anisotropic directional drag component (`RigidbodyDirectionalDrag`) and ship fender collision absorbers (`Fender`) in Sailwind v0.38 (`Assembly-CSharp.dll`). This note is valuable for modders working with ship physics, leeway/drift mitigation, and dock collision handling.

---

## 1. Anisotropic Drag (`RigidbodyDirectionalDrag`)

Unity's standard `Rigidbody.drag` property applies isotropic linear damping equally across all three axes. However, a ship's hull resists lateral sideways motion (leeway/drift) orders of magnitude more than forward progress. Sailwind models this behavior using `RigidbodyDirectionalDrag`.

### 1.1. Component Parameters

| Field | Type | Description |
|---|---|---|
| `x` | `float` | Velocity retention multiplier along the transverse (lateral drift) axis. |
| `y` | `float` | Velocity retention multiplier along the vertical (heaving) axis. |
| `zForward` | `float` | Velocity retention multiplier for forward longitudinal motion. |
| `zBack` | `float` | Velocity retention multiplier for stern-way longitudinal motion. |

> **Important:** All values must lie within `(0.0, 1.0]`, as they represent **per-tick velocity retention multipliers**, not additive forces.

### 1.2. Damping Algorithm in `FixedUpdate()`

Every physics step, the component transforms the `Rigidbody` velocity vector into local space and multiplies its axes directly:

```csharp
private void FixedUpdate()
{
    Vector3 localVel = transform.InverseTransformDirection(rb.velocity);
    localVel.x *= x;
    localVel.y *= y;
    if (localVel.z > 0f)
        localVel.z *= zForward;
    else
        localVel.z *= zBack;

    rb.velocity = transform.TransformDirection(localVel);
}
```

### 1.3. Physical Significance
1. **Force-Free Damping:** Velocity reduction does not occur via `AddForce`, but through **direct scaling of the velocity vector** (`rb.velocity = ...`).
2. **Eliminating Leeway:** With `x = 0.95f`, a ship loses 5% of its lateral drift velocity every PhysX step (~45.5 Hz), instantly damping sideways slipping while leaving forward motion unimpeded (`zForward = 0.999f`).

---

## 2. Fender Contact Physics (`Fender`)

Boat fenders (`Fender`) hanging off hull sides are not merely decorative assets. They are active trigger-based shock absorbers that cushion hard impacts between rigid boat hulls and wooden dock structures.

### 2.1. Dock Repulsion Algorithm (`FixedUpdate`)

When any non-trigger dock collider enters a fender's trigger (`OnTriggerEnter`), it is added to `stayedColliders`. Every physics step, the fender applies a repulsive counter-force to the ship's hull:

```csharp
private void FixedUpdate()
{
    if (stayedColliders.Count > 0)
    {
        shipBody.AddForceAtPosition(transform.forward * forceMult * shipBody.mass, transform.position);
        GetComponent<Renderer>().material.color = Color.red;
    }
    else
    {
        GetComponent<Renderer>().material.color = Color.white;
    }

    if (resetFrameCount <= 0)
    {
        ResetCol();
        resetFrameCount = 12;
    }
    resetFrameCount--;
}
```

- The fender's `transform.forward` vector points from the dock structure inward toward the boat hull.
- Repulsive force scales proportionally with vessel mass (`forceMult * shipBody.mass`).

### 2.2. Why `ResetCol` Toggles the Collider Every 12 Frames
Notice the `ResetCol()` method invoked every 12 physics steps (~0.26 seconds, randomized initial offset `Random.Range(0, 12)`):

```csharp
private void ResetCol()
{
    col.enabled = false;
    stayedColliders.Clear();
    col.enabled = true;
}
```

**Reason:** In Unity's PhysX engine, `OnTriggerExit` callbacks frequently fail to fire when colliders slide smoothly against each other (e.g., when a hull slides along a dock during mooring). Without periodically flushing `stayedColliders`, a fender would continue pushing a vessel away from the dock even after physical separation. Toggling `col.enabled` forces Unity to re-evaluate trigger overlaps from scratch.

---

## 3. Practical Modding Conclusions

1. **Tuning Drift for Custom Ships:** When designing new boat hulls, do not increase standard `Rigidbody.drag` to suppress lateral drifting (this slows forward speed). Attach `RigidbodyDirectionalDrag` and tune `x` between `0.92 .. 0.97`.
2. **Direct `rb.velocity` Modification Order:** If your item physics or autopilot mod also manipulates `rb.velocity` in `FixedUpdate()`, manage your Unity Script Execution Order: `RigidbodyDirectionalDrag` should execute after sail thrust calculations but before kinematic origin shifts.
3. **Fender Compatibility With Solid Items:** The `Fender` trigger detects any collider where `isTrigger == false`. If an item physics mod converts item twin colliders to non-trigger (`non-trigger`), a dropped cargo crate contacting a fender will push the entire ship sideways with force `forceMult * shipBody.mass`!
