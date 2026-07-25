# 74. Crest interactions, water visuals, and floating origin

A review of the remaining runtime systems in Sailwind v0.38's `Crest.dll`:
`BoatProbes`, object-water interaction, dynamic-wave inputs, underwater effect,
`WaterBody`, depth cache, reflections, ocean-chunk rendering, and Crest's own
floating origin.

Related to [70](70-water-splash-particle-systems.md), [71](71-crest-simplefloatingobject-exact-model.md), [72](72-crest-oceanrenderer-query-contract.md), and [73](73-crest-wave-spectrum-lod-simulation.md).

## `BoatProbes`: full Crest multi-point buoyancy

`BoatProbes : FloatingObjectBase` is not `SimpleFloatingObject`. It is intended
for vessels and uses `FloaterForcePoints`:

```csharp
public class FloaterForcePoints
{
    public float _weight = 1f;
    public Vector3 _offsetPosition;
}
```

At `Start()`, BoatProbes:

- obtains Rigidbody;
- applies `_rb.centerOfMass = _centerOfMass`;
- sums `_totalWeight` across force points;
- creates query arrays with length `forcePoints + 1`.

In `FixedUpdate()`, it queries displacement/velocity for every force point and
for the body center.

### BoatProbes buoyancy form

Per point:

```text
submersion = SeaLevel + displacementY(point) - pointY
```

When `submersion > 0`:

```csharp
force = 1000 × abs(Physics.gravity.y) × submersion
        × point.weight × forceMultiplier / totalWeight;
rb.AddForceAtPosition(force × Vector3.up, point);
```

Compared with `SimpleFloatingObject`:

| Property | `BoatProbes` | `SimpleFloatingObject` |
|---|---|---|
| Points | multiple force points | one transform point |
| Lift | linear-in-depth force | cubic acceleration |
| Mass | Unity `ForceMode.Force` supplies mass/inertia behavior | `Acceleration` cancels mass |
| Query | batched point arrays | one helper query |
| Intended use | vessel | simple float object/item |

BoatProbes is a useful vanilla reference for mod hull buoyancy: distributed
force points, a single query batch, and force at position. Its values are
prefab-specific, so do not copy them blindly to cargo.

## `ObjectWaterInteraction` and `SphereWaterInteraction`

These classes do not add Rigidbody buoyancy. They write velocity/weight into a
`MaterialPropertyBlock` for the **dynamic-wave simulation**.

### `ObjectWaterInteraction`

If its parent lacks `FloatingObjectBase`, it adds `ObjectWaterInteractionAdaptor`.
Each `LateUpdate` it:

1. finds active dynamic-wave simulations for the current LOD;
2. positions interaction ahead of travel (`_velocityPositionOffset = 0.2`);
3. computes velocity relative to flow;
4. discards velocity above 500 km/h as teleport;
5. clamps velocity to 100 km/h;
6. writes `_Velocity`, `_Weight`, `_SimDeltaTime` to its material property block.

`_Weight = 1 / activeSims`, but only if the parent `FloatingObjectBase.InWater`.

### `SphereWaterInteraction`

A spherical wake/ripple input:

- `Radius = 0.5 × transform.lossyScale.x`;
- samples water with spatial length `2 × Radius`;
- default `_weight` is 1;
- weakens weight above and deep below water;
- writes `_Radius`, `_Velocity`, `_Weight`, `_SimDeltaTime` shader properties.

> These interaction classes are visual/dynamic-wave inputs. They do not replace
> collision, mass, or buoyancy physics.

## `ObjectWaterInteractionAdaptor`

When a parent lacks `FloatingObjectBase`, this adaptor is auto-created. It:

- queries one displacement point in `Update`;
- defines `InWater` as `transform.position.y - sampledHeight <= 0`;
- derives transform velocity from the previous frame.

It is a Crest bridge turning an ordinary GameObject into a visual-wave input;
it does not make the object physically float.

## `UnderwaterEffect`

`Crest.UnderwaterEffect` exposes:

```csharp
public static float cameraWaterHeight;
```

In `LateUpdate`, it:

1. samples `SampleHeightHelper` at its own effect-object position;
2. assigns `cameraWaterHeight = sampledHeight`;
3. enables the renderer when the camera is less than `_maxHeightAboveWater`
   (default 1.5) above water;
4. copies the ocean material and binds animated-wave/depth/shadow LOD data.

### Modder limitation

`cameraWaterHeight` is a height **at the camera/effect position**, not a query
at an arbitrary item. It is suitable as a cheap mean-sea-level fallback or
player-swimming input, not a precise cargo-wave surface far from camera.

## `WaterBody`

`WaterBody` maintains a static list and the AABB of one transform quad:

```csharp
public static List<WaterBody> WaterBodies
public Bounds AABB { get; private set; }
```

`OceanRenderer.LateUpdateTiles()` uses this list for ocean-chunk culling.
`UnderwaterEffect` may disable itself outside all WaterBody AABBs.

For mod water-volume logic, WaterBody is an XZ area/culling marker, **not** a
physical volume collider or a height source.

## Ocean depth cache

`OceanDepthCache` creates a top-down orthographic camera and renders selected
layers/renderers using:

```text
Crest/Inputs/Depth/Ocean Depth From Geometry
```

The result is an RHalf depth texture. Modes:

| Mode | Behavior |
|---|---|
| `Realtime` + `OnStart` | calls `PopulateCache()` at Start |
| `Realtime` + `OnDemand` | caller must call `PopulateCache()` |
| `Baked` | uses serialized `_savedCache` |

It supports shore/shallows/foam but is not a collision mesh by itself.

## Ocean tiles and render bounds

`OceanBuilder.GenerateMesh()`:

- creates a hidden Root under `OceanRenderer`;
- builds 10 patch-mesh types;
- creates 16 tiles at LOD0 and 12 at later LODs;
- assigns the `_layerName` layer (`Water` by default);
- may stretch outer skirt vertices ×100.

`OceanChunkRenderer.ExpandBoundsForDisplacements()` expands render bounds by:

```text
horizontal = OceanRenderer.MaxHorizDisplacement
vertical   = OceanRenderer.MaxVertDisplacement + 5
```

Crest ocean-tile renderer bounds are intentionally huge and must never be used
as a physical object hull/buoyancy volume.

## Planar reflections

`OceanPlanarReflection` creates a reflection camera and RenderTexture.

| Field | Default |
|---|---:|
| `_textureSize` | 256 |
| `_clipPlaneOffset` | 0.07 |
| `_hdr` | true |
| `_farClipPlane` | 1000 |
| `RefreshPerFrames` | 1 |

The texture is stored in internal `PreparedReflections` under
`camera.GetHashCode()` and bound to `OceanChunkRenderer` as `_ReflectionTex`.

## Crest `FloatingOrigin`

This is separate from Sailwind `FloatingOriginManager`.

| Field | Default |
|---|---:|
| `_threshold` | 16384 |
| `_physicsThreshold` | 1000 |
| `_defaultSleepThreshold` | 0.14 |

When X/Z crosses the threshold, it:

1. shifts root transforms;
2. shifts world-space particles;
3. calls `SetOrigin` on `LodTransform`, `IFloatingOrigin` children, and
   `ShapeGerstnerBatched`;
4. sets `Rigidbody.sleepThreshold = float.MaxValue` beyond physics threshold.

This is a separate potential source of distant-body behavior. Mods handling
ocean/body persistence must distinguish Crest origin from game
`FloatingOriginManager`.

## Shader inputs and useful components

| Type | Registers |
|---|---|
| `RegisterAnimWavesInput` | wave displacement; can report max displacement |
| `RegisterDynWavesInput` | dynamic waves |
| `RegisterFlowInput` | flow |
| `RegisterFoamInput` | foam |
| `RegisterClipSurfaceInput` | holes/clipped surface |
| `RegisterSeaFloorDepthInput` | depth |
| `RenderAlphaOnSurface` | material alpha/clip data at surface |

`RegisterLodDataInputBase.Draw()` queries displacement at the input position and
writes `_DisplacementAtInputPosition`, allowing input shaders to account for
horizontal surface motion.

## Practical implications

1. `BoatProbes`, not `SimpleFloatingObject`, is the closest useful vanilla
   reference for realistic distributed hull buoyancy.
2. Dynamic-wave interaction is a render input, not force physics.
3. `cameraWaterHeight` is a camera-local fallback, not a global wave API.
4. `WaterBody` is an area/culling marker, not a water-volume collider.
5. Crest ocean-tile bounds are deliberately expanded and unsuitable for item hull measurement.
6. Crest FloatingOrigin can change Rigidbody sleep threshold beyond 1000 units;
   this matters to distant cargo and LOD.
