# Private: crate unseal physics bug - root cause analysis

## Chain of causation

1. UnsealCrate() spawns items at +100.5 Y above crate
2. Item Awake -> ItemRigidbody.Start -> if wallAttachment=true in prefab -> attached=true
3. Item falls, mod registers it via BodyRegistry
4. PrepareForWorldFloat: sold=true, nailed=false, but attached stays true (BUG in mod)
5. SurfaceBodyDriver: IsAttached=true -> Suspended -> item frozen mid-air
6. Even if buoyancy worked: _densityRatio ~0.38 for 0.8kg cheese with FloatHullDensity=200

## Root cause 1: PrepareForWorldFloat missing attached=false

```csharp
// ItemAccess.PrepareForWorldFloat:
item.sold = true;
item.nailed = false;
// MISSING: item twin attached = false
```

If attached was set during init (wallAttachment=true in prefab, or race condition),
it stays true forever. SurfaceBodyDriver suspends the item permanently.

Fix: add `SetAttached(twin, false)` in PrepareForWorldFloat.

## Root cause 2: FloatHullDensity=200 cannot work for all sizes

NeutralBuoyancyMass = realColliderBounds * FloatHullDensity

- Crate (bounds ~0.8m3): NBM ~160 -> ratio ~0.1 -> settles ok
- Cheese (bounds ~0.01m3): NBM ~2 -> ratio ~0.4 -> floats too high

The density parameter needs per-item scaling or a lookup table based on mass.

## Root cause 3: Freeboard vs Archimedes model conflict

_dynFreeboard = Freeboard + HalfExtents.y * 2 * (1 - _densityRatio)
Submerged depth from _densityRatio

These two models compute different target positions. Need single coherent model.

## Root cause 4: CrateInventory.LateUpdate position lock

When items are inside opened crate, LateUpdate forces their position to
crate center every frame. Any physics engine moving them fights this lock.
