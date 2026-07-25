# 67. Item physical-twin collider construction: timing, copying, and CCD

A precise examination of `ItemRigidbody.Start()`, `AddCollider()`, and `CreateSubcollider()` in Sailwind v0.38. This complements the twin model in notes [16](../16-item-framework-shipitem.md), [43](../43-item-buoyancy-water.md), and [44](../44-itemrigidbody-field-map-contract.md).

## Short version: when the twin is actually physics-ready

The existence of `ShipItem.itemRigidbodyC` **does not** mean that the twin already has usable colliders.

```text
ShipItem.LoadAfterDelay coroutine
  └─ CreateRigidbody()
       └─ AddComponent<ItemRigidbody>()
            └─ ItemRigidbody.Start()
                 ├─ AddComponent<Rigidbody>()
                 ├─ StartCoroutine(AddCollider())
                 ├─ creates SimpleFloatingObject
                 └─ immediately copies child ItemSubcollider objects

AddCollider()
  ├─ yield WaitForFixedUpdate × 3
  ├─ yield WaitForEndOfFrame
  └─ creates root Box/Mesh/CapsuleCollider on the twin
```

**Practical rule:** a mod requiring twin physics colliders must wait at least
**three physics ticks plus EndOfFrame** after `ItemRigidbody` appears. Polling
for an actual `Collider` on the twin is even safer. Do not measure bounds or
change collision mode solely because `itemRigidbodyC != null`.

## What `Start()` creates immediately

`ItemRigidbody.Start()` does the following:

```csharp
rigidbody = gameObject.AddComponent<Rigidbody>();
rigidbody.drag = 1.2f;
rigidbody.angularDrag = item.mass * 0.1f;
rigidbody.isKinematic = true;
UpdateMass();
StartCoroutine(AddCollider());
floater = gameObject.AddComponent<SimpleFloatingObject>();
gameObject.layer = 2;
```

The Rigidbody therefore exists before the root collider. It begins kinematic;
`ItemRigidbody.FixedUpdate` later decides its final `isKinematic` state (notes
43–44).

## Root colliders: source and copy rules

After the delay, `AddCollider()` checks **only** root components on the visual
item object:

```csharp
BoxCollider     box     = item.GetComponent<BoxCollider>();
MeshCollider    mesh    = item.GetComponent<MeshCollider>();
CapsuleCollider capsule = item.GetComponent<CapsuleCollider>();
```

| Visual collider | Created on twin | Copied properties |
|---|---|---|
| `BoxCollider` | new `BoxCollider` | `center`, `size` |
| `MeshCollider` | new `MeshCollider` | `sharedMesh`; always `convex = true` |
| `CapsuleCollider` | new `CapsuleCollider` | `center`, `radius`, `height`, `direction` |

It does not copy `material`, `contactOffset`, `isTrigger`, `enabled`, layer,
tag, or other custom collider settings. A mod must explicitly configure those
on the twin after it exists.

### Important side effect: visual MeshCollider is destroyed

When the visual has a `MeshCollider` **and** a `BoxCollider` or
`CapsuleCollider`, vanilla runs this after copying:

```csharp
Destroy(item.GetComponent<MeshCollider>());
```

The twin has already received its own convex mesh, but the **visual** object's
MeshCollider is removed. A mod must not retain a long-lived reference to the
visual MeshCollider and expect it to survive item initialization.

## Child colliders: only `ItemSubcollider` is copied

A visual child Transform is copied to the twin only when it is tagged
`ItemSubcollider`:

```csharp
foreach (Transform child in item.transform)
    if (child.CompareTag("ItemSubcollider")) CreateSubcollider(child);
```

`CreateSubcollider()` instantiates the entire child GameObject, reparents it
to the twin, keeps local position/rotation, and then forces:

```csharp
clone.tag = "Untagged";
clone.layer = 2;
clone.GetComponent<Collider>().isTrigger = false;
```

| Clone property | Value |
|---|---|
| Parent | twin `ItemRigidbody.transform` |
| Pose | same `localPosition` and `localRotation` |
| Tag | `Untagged` |
| Layer | `2` (`IgnoreRaycast`) |
| Collider | non-trigger until later hold/inventory logic |

An ordinary child collider without that tag is **not copied**. `CreateSubcollider()`
runs in `Start()` before the coroutine delay, so a tagged subcollider can exist
before any root collider.

## Collision detection mode: actual mapping

In Unity 2019, `CollisionDetectionMode` values are:

| Decompiled number | Unity value |
|---:|---|
| `2` | `ContinuousDynamic` |
| `3` | `ContinuousSpeculative` |

After `AddCollider()`, vanilla chooses:

```csharp
rigidbody.collisionDetectionMode = meshCol != null
    ? (CollisionDetectionMode)2
    : (CollisionDetectionMode)3;
```

| Twin form | Mode after `AddCollider()` |
|---|---|
| Root `MeshCollider` exists | `ContinuousDynamic` (2) |
| Box/Capsule only or no root mesh | `ContinuousSpeculative` (3) |

This corrects an inaccuracy in note 16: value `3` is **ContinuousSpeculative**,
not `ContinuousDynamic`.

### Temporary upgrade after release

`dynamicColTimer` is set to `6f` at startup and on the held path. In
`FixedUpdate`:

```csharp
if (!item.held && dynamicColTimer > 0f && !rigidbody.isKinematic)
{
    rigidbody.collisionDetectionMode = (CollisionDetectionMode)2;
    dynamicColTimer -= Time.deltaTime;
}
if (dynamicColTimer <= 0f && meshCol == null)
    rigidbody.collisionDetectionMode = (CollisionDetectionMode)3;
```

A Box/Capsule twin therefore uses `ContinuousDynamic` for roughly six seconds
after release, then returns to `ContinuousSpeculative`. A mesh twin stays
`ContinuousDynamic`, as the final condition applies only when `meshCol == null`.

## Practical implications for modders

1. **Wait for collider readiness, not merely for the twin.** `itemRigidbodyC`
   arrives about 3 fixed frames plus EndOfFrame before a root collider.
2. **Operate on the twin.** It owns the collider copies and Rigidbody; visual
   colliders are separate interaction/raycast geometry.
3. **Do not use magic CCD numbers.** In Sailwind v0.38, `2 = ContinuousDynamic`
   and `3 = ContinuousSpeculative`.
4. **Mesh + primitive on visual does not mean two permanent visual colliders.**
   The visual MeshCollider is destroyed, while the twin retains a convex copy.
5. **To transfer a child physical shape to the twin**, the child GameObject
   needs the `ItemSubcollider` tag; vanilla ignores normal child colliders.
6. **The clone does not retain its tag:** `ItemSubcollider` becomes `Untagged`
   on layer 2. This matters to tag filters and the collision matrix.
