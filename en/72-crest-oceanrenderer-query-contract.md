# 72. Crest `OceanRenderer` and the CPU surface-query contract

A full review of the core runtime API in Sailwind v0.38's `Crest.dll`: `OceanRenderer`, collision/flow providers, `SampleHeightHelper`, `SampleFlowHelper`, and the GPU query pipeline. This is directly decompiled from the shipped `Crest.dll`, not inferred from game call sites.

Related to [31](../31-ocean-waves-inertia.md), [48](48-ocean-height-helper-lifecycle.md), and [71](71-crest-simplefloatingobject-exact-model.md).

## `OceanRenderer`: the Crest singleton, not `Ocean.Singleton`

The main object in `Crest.dll` is:

```csharp
public class OceanRenderer : MonoBehaviour
{
    public static OceanRenderer Instance { get; private set; }
    public Transform Root { get; private set; }
    public ICollProvider CollisionProvider { get; private set; }
    public IFlowProvider FlowProvider { get; private set; }
    public float SeaLevel => Root.position.y;
}
```

It is architecturally distinct from Sailwind's `Ocean` class in
`Assembly-CSharp.dll` (`Ocean.Singleton`). Sailwind contains both layers:

| Layer | DLL / type | Role |
|---|---|---|
| Sailwind wrapper | `Ocean`, `OceanHeight` | game logic, legacy height API, weather/visual integration |
| Crest runtime | `Crest.OceanRenderer` | LOD mesh, GPU data, collision/flow providers, wave inputs |

The presence of `OceanHeight` does not prove that `OceanRenderer.Instance`, a
`CollisionProvider`, or GPU readback is ready.

## `OceanRenderer` lifecycle

`OnEnable()`:

1. checks `SystemInfo.supportsComputeShaders` and `supports2DArrayTextures`;
2. assigns `OceanRenderer.Instance = this`;
3. creates `LodTransform` and LOD data;
4. calls `OceanBuilder.GenerateMesh()` for the ocean-tile root;
5. creates subsystems (animated waves always; flow/foam/dynamic waves/depth/
   shadows/clip based on flags);
6. creates `CollisionProvider` and `FlowProvider`;
7. chooses a viewpoint (`_viewpoint`, else `Camera.main`).

`LateUpdate()` runs `RunUpdate()` only while `Time.timeScale > 0`:

```text
CollisionProvider.UpdateQueries()
FlowProvider.UpdateQueries()
→ global shader parameters
→ Root follows viewpoint XZ
→ scale/LOD update
→ all LOD data update
→ command-buffer build/execute
```

CPU queries and their results can therefore be frame-delayed; a query posted
before `RunUpdate()` need not have a fresh result immediately.

## Important `OceanRenderer` parameters

| Field / property | Default / meaning |
|---|---|
| `_layerName` | `Water`; `OceanBuilder` resolves it through `LayerMask.NameToLayer` |
| `_gravityMultiplier` | 1; `Gravity = multiplier × Physics.gravity.magnitude` |
| `_minTexelsPerWave` | 3; controls spatial filtering of short waves |
| `_minScale` / `_maxScale` | 8 / 256; ocean LOD scale around the viewpoint |
| `_lodDataResolution` | 256 |
| `_geometryDownSampleFactor` | 2 |
| `_lodCount` | 7; the system maximum is 15 |
| `_createSeaFloorDepthData` | true |
| `_createFoamSim` | true |
| `_createDynamicWaveSim` | optional, false in Crest's field default |
| `_createFlowSim` | optional |
| `_createShadowData` / `_createClipSurfaceData` | optional |

`SeaLevel` is `Root.position.y`; it is not necessarily zero.

## Collision-provider modes

`SimSettingsAnimatedWaves.CollisionSources`:

```csharp
public enum CollisionSources
{
    None,
    GerstnerWavesCPU,
    ComputeShaderQueries
}
```

| Mode | Provider | Property |
|---|---|---|
| `None` | `CollProviderNull` | always yields zero displacement, `Vector3.up`, zero velocity, and successful status |
| `GerstnerWavesCPU` | scene `ShapeGerstnerBatched` | synchronous analytic CPU Gerstner evaluation |
| `ComputeShaderQueries` | `QueryDisplacements` | GPU compute plus `AsyncGPUReadback`; current data can arrive late |

If a chosen provider cannot be made, `CreateCollisionProvider()` falls back to
`CollProviderNull`, not `null`.

## `ICollProvider` / `IFlowProvider`

```csharp
public interface ICollProvider
{
    int Query(int ownerHash, float minSpatialLength, Vector3[] points,
              float[] heights, Vector3[] normals, Vector3[] velocities);
    int Query(int ownerHash, float minSpatialLength, Vector3[] points,
              Vector3[] displacements, Vector3[] normals, Vector3[] velocities);
    bool RetrieveSucceeded(int status);
    void UpdateQueries();
    void CleanUp();
}
```

```csharp
public interface IFlowProvider
{
    int Query(int ownerHash, float minSpatialLength, Vector3[] points,
              Vector3[] flows);
    bool RetrieveSucceeded(int status);
    void UpdateQueries();
    void CleanUp();
}
```

### `ownerHash` is not decorative

`QueryBase` uses it as a segment-registration key in a ring buffer. Reuse a
**stable** hash per query owner; do not create a random ID every frame.

Compute-query limits:

| Limit | Value |
|---|---:|
| default maximum query points | 4096 (`SimSettingsAnimatedWaves._maxQueryCount`) |
| maximum registered GUID/owners | 1024 |
| ring-buffer segment pool | 7 |
| stale registrations retained after acquire | about 10 frames |
| finite-difference normal stencil | 0.1 world unit |

Ring exhaustion logs:

```text
Query ring buffer exhausted. Please report this to developers.
```

## Query status bits

`QueryBase.QueryStatus`:

| Bit | Enum | Meaning |
|---:|---|---|
| 0 | `OK` | success |
| 1 | `RetrieveFailed` | latest GPU/readback result is unavailable for this owner |
| 2 | `PostFailed` | query points could not be posted |
| 4 | `NotEnoughDataForVels` | fewer than two snapshots exist for velocity |
| 8 | `VelocityDataInvalidated` | registration segment changed |
| 16 | `InvalidDtForVelocity` | snapshot interval is too small |

`RetrieveSucceeded(status)` tests only `(status & 1) == 0`. A caller requiring
surface velocity must separately respect bits 4/8/16: height can be valid while
velocity is not.

## `SampleHeightHelper`: exact behavior

`SampleHeightHelper` owns one-element arrays and offers four `Sample` overloads.

```csharp
helper.Init(worldPosition, minSpatialLength, allowMultipleCallsPerFrame, context);
bool ok = helper.Sample(out float height);
```

On success:

```text
height = displacement.y + OceanRenderer.Instance.SeaLevel
```

When `CollisionProvider == null`:

```text
returns false
height = 0
normal = up
velocity = zero
```

When the provider query exists but its result is not retrieved yet:

```text
returns false
height = SeaLevel
normal = up
velocity = zero
```

**Never ignore the bool result.** A returned 0/SeaLevel can be fallback rather
than measured wave data.

## Sailwind `OceanHeight.GetHeight` wrapper trap

The game wrapper does:

```csharp
helper.Init(worldPos, accuracy, false, null);
float result = 0f;
if (helper.Sample(ref result)) return result;
return 0f;
```

It returns `0f` on failure and discards the bool. A float from `OceanHeight`
alone is not proof of a live surface.

> For robust mod code, call `SampleHeightHelper.Sample(...)` directly and keep
> the success flag, or maintain explicit fallback/diagnostic state.

## `SampleFlowHelper`

```csharp
helper.Init(position, minSpatialLength);
bool ok = helper.Sample(out Vector2 flow);
```

The helper maps its `Vector3` result to XZ:

```csharp
flow.x = result.x;
flow.y = result.z;
```

Missing/failed flow produces `false` and `Vector2.zero`. If `_createFlowSim` is
off, `OceanRenderer` uses `FlowProviderNull`, which returns zero flow with a
successful status.

## Safe mod-query template

```csharp
var helper = new Crest.SampleHeightHelper(); // retain per body/service; do not allocate every frame
helper.Init(scenePosition, minSpatialLength: objectWidth,
            allowMultipleCallsPerFrame: true);

float height;
Vector3 normal, surfaceVelocity;
bool exact = helper.Sample(out height, out normal, out surfaceVelocity);
if (!exact)
{
    // Do not turn 0f into "exact water".
    // Use last valid sample, an explicit fallback, or suspend.
}
```

### Choosing `minSpatialLength`

Crest converts it to:

```text
minGridSize = minSpatialLength / 2 / OceanRenderer.MinTexelsPerWave
```

A larger value filters short waves and demands less detail; a small value asks
for fine LOD and creates a noisier response. A good physics starting point is
body hull width, not zero and not world size.

## Practical implications

1. `OceanRenderer.Instance`, `CollisionProvider`, and ready readback are three
   separate readiness stages.
2. `OceanHeight.GetHeight()` loses failure information; returned 0 can be fallback.
3. Compute queries have limits: 4096 points by default, 1024 owners, 7 ring segments.
4. Crest posts three points per requested normal: center, +X 0.1, +Z 0.1.
5. CPU `ShapeGerstnerBatched` is a separate safe path without GPU readback; see note 73.
6. Do not query from an uncontrolled number of bodies every `FixedUpdate`; a
   batch/cache/query service is necessary for large mod populations.
