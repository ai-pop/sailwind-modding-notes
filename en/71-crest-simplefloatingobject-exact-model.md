# 71. Crest `SimpleFloatingObject`: exact buoyancy model from `Crest.dll`

This note resolves the previous uncertainty in notes 43/63: Sailwind v0.38's `Crest.dll` was decompiled, so `Crest.SimpleFloatingObject` behavior is now known exactly.

## Class and lifecycle

```text
Crest.FloatingObjectBase : MonoBehaviour
  └─ Crest.SimpleFloatingObject
```

`SimpleFloatingObject.Start()` obtains the Rigidbody on its own GameObject:

```csharp
_rb = GetComponent<Rigidbody>();
if (OceanRenderer.Instance == null)
    enabled = false;
```

With a live `OceanRenderer.Instance`, the component runs in `FixedUpdate`; setting `enabled = false` genuinely stops its physical forces.

## Parameters

| Field | Default | Meaning |
|---|---:|---|
| `_raiseObject` | 1 | vertical surface offset; `ItemRigidbody.Start()` assigns `ShipItem.floaterHeight` (usually **1.6**) |
| `_buoyancyCoeff` | 3 | cubic lift-acceleration coefficient |
| `_boyancyTorque` | 8 | torque aligning body to water normal |
| `_objectWidth` | 3 | `SampleHeightHelper` minimum wavelength/filter width |
| `_forceHeightOffset` | -0.3 | Y offset of the water-drag application point |
| `_dragInWaterUp/Right/Forward` | 3 / 2 / 1 | directional drag coefficients |
| `_dragInWaterRotational` | 0.2 | rotational drag (`ItemRigidbody` overwrites it to **0.02**) |

## Exact `FixedUpdate`

When the Rigidbody is dynamic and `OceanRenderer.Instance` is live, Crest:

1. samples displacement, normal, and surface velocity:
   ```csharp
   _sampleHeightHelper.Init(transform.position, _objectWidth,
                            allowMultipleCallsPerFrame: true);
   _sampleHeightHelper.Sample(out displacement, out normal, out surfaceVelocity);
   ```
2. adds flow from `SampleFlowHelper`;
3. calculates `relativeVelocity = rb.velocity - surfaceVelocity`;
4. calculates depth:
   ```csharp
   depth = displacement.y + OceanRenderer.Instance.SeaLevel
           - transform.position.y + _raiseObject;
   InWater = depth > 0;
   ```
5. if `InWater`, applies:
   ```csharp
   force = -Physics.gravity.normalized * _buoyancyCoeff * depth * depth * depth;
   rb.AddForce(force, ForceMode.Acceleration);
   ```

### Key property: cubic, mass-independent lift

Crest uses `ForceMode.Acceleration`, not `ForceMode.Force`:

```text
acceleration ∝ depth³
```

Consequences:

- Rigidbody mass **does not participate** in the response;
- a small item that has penetrated deeply receives rapidly increasing acceleration;
- `_raiseObject = 1.6` means a vanilla item can be considered in water 1.6 Unity units above the physical surface;
- enabling this component together with a mod's Archimedes model creates two independent lift systems and can cause water-trampoline/ejection behavior.

## Crest drag and torque

While `InWater`, Crest applies `ForceMode.Acceleration` directional drag at:

```csharp
position = rb.position + _forceHeightOffset * Vector3.up;
rb.AddForceAtPosition(up      * Dot(up,      -relativeVelocity) * _dragInWaterUp, position, Acceleration);
rb.AddForceAtPosition(right   * Dot(right,   -relativeVelocity) * _dragInWaterRight, position, Acceleration);
rb.AddForceAtPosition(forward * Dot(forward, -relativeVelocity) * _dragInWaterForward, position, Acceleration);
```

Then it uses:

```csharp
rb.AddTorque(Cross(transform.up, waterNormal) * _boyancyTorque,
             ForceMode.Acceleration);
rb.AddTorque(-_dragInWaterRotational * rb.angularVelocity);
```

This is not mass-aware water dynamics either.

## Why `ItemRigidbody` is special

In `ItemRigidbody.Start()`, vanilla creates and configures the component:

```csharp
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;
```

But `ItemRigidbody.ToggleCollider()` unconditionally executes:

```csharp
floater.enabled = false;
```

The floater lifecycle therefore depends on `Start`/`FixedUpdate`/mod-patch order. A mod replacing buoyancy must not merely calculate its own force; it must reliably keep the vanilla floater disabled. An `enabled` check alone is insufficient against a re-enable race: the robust ownership boundary is a Harmony prefix on `SimpleFloatingObject.FixedUpdate` for a managed twin.

## `SampleHeightHelper` and Crest readiness

`SampleHeightHelper.Sample()` calls:

```csharp
OceanRenderer.Instance.CollisionProvider.Query(...)
```

When there is no provider or the query has not succeeded, it returns `false` and sea level. The existence of the class/helper instance **does not prove** that the query pipeline is ready. Mod code must handle readiness/fallback separately and avoid unbounded queries in a hot physics path.

## Practical implications

1. Vanilla `SimpleFloatingObject` is not Archimedes: it uses cubic depth acceleration independent of mass.
2. `floaterHeight=1.6` directly becomes `_raiseObject`; it is a substantial lift offset, not item draft.
3. Never run the Crest floater and custom `ForceMode.Force` buoyancy on the same Rigidbody together.
4. For a managed item twin, robustly suppress Crest `FixedUpdate`, not only `Behaviour.enabled`.
5. Mass-aware behavior needs its own mass, displaced-volume, water-entry damping, and relative-velocity forces.

## Related notes

- [43 — item buoyancy](../43-item-buoyancy-water.md)
- [48 — ocean-height lifecycle](48-ocean-height-helper-lifecycle.md)
- [63 — buoyancy and floating cargo](63-vanilla-buoyancy-floating-cargo.md)
- [70 — water splash visuals](70-water-splash-particle-systems.md)
