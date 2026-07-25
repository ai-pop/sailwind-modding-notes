# 75. Sailwind v0.38 `Crest.dll`: complete index of 80 types

A full source map for the `Crest.dll` decompiled from Sailwind v0.38. This note
does not replace thematic notes [71](71-crest-simplefloatingobject-exact-model.md)–[74](74-crest-water-interactions-rendering-origin.md); it guarantees that no assembly type was omitted from the investigation.

## Reading the index

- **Runtime** — a type that can be observed/used in a running game.
- **Input** — a Unity component registering a renderer/material into Crest LOD data.
- **Internal utility** — infrastructure, normally not a mod API entry point.
- Source filenames correspond to namespace `Crest`, except `RenderWireFrame` in the global namespace.

## Core ocean, LOD, and command buffers

| Type | Role |
|---|---|
| `OceanRenderer` | Main Crest singleton; root, LOD, providers, command-buffer lifecycle. |
| `OceanBuilder` | Builds patch meshes and the ocean-tile hierarchy. |
| `OceanChunkRenderer` | One-tile renderer; binds LOD textures, reflection, render bounds. |
| `LodTransform` | Snapped LOD positions, texel scale, view/projection matrices; `IFloatingOrigin`. |
| `LodDataMgr` | Abstract base for all LOD texture managers. |
| `LodDataMgrAnimWaves` | Animated-wave buffers/combine/collision-shape data. |
| `LodDataMgrClipSurface` | Clip/holes texture layer. |
| `LodDataMgrDynWaves` | Persistent dynamic-wave simulation. |
| `LodDataMgrFlow` | Flow texture layer. |
| `LodDataMgrFoam` | Persistent foam simulation. |
| `LodDataMgrPersistent` | Abstract ping-pong persistent-simulation base. |
| `LodDataMgrSeaFloorDepth` | Sea-floor-depth texture layer. |
| `LodDataMgrShadow` | Water-shadow texture/update from directional light. |
| `BuildCommandBufferBase` | Static last-update-frame base. |
| `BuildCommandBuffer` | Builds and executes active LOD command buffers. |
| `ILodDataInput` | Input contract: wavelength, enabled, draw. |
| `IPropertyWrapper` | Abstract material/compute property-setter contract. |
| `PropertyWrapperMaterial` | Material-backed property writer. |
| `PropertyWrapperMPB` | `MaterialPropertyBlock`-backed writer. |
| `PropertyWrapperCompute` | CommandBuffer compute-shader writer. |
| `PropertyWrapperComputeStandalone` | Direct compute-shader writer for queries. |

## CPU/GPU collision and flow queries

| Type | Role |
|---|---|
| `ICollProvider` | Displacement/height/normal/velocity query interface. |
| `IFlowProvider` | XZ flow query interface. |
| `QueryBase` | GPU compute plus async-readback ring buffer, query registration/status. |
| `QueryDisplacements` | `ICollProvider` using `QueryDisplacements` compute shader. |
| `QueryFlow` | `IFlowProvider` using `QueryFlow` compute shader. |
| `CollProviderNull` | Successful zero-displacement fallback. |
| `FlowProviderNull` | Successful zero-flow fallback. |
| `SampleHeightHelper` | Convenient one-point height/displacement/normal/velocity query wrapper. |
| `SampleFlowHelper` | Convenient one-point XZ flow wrapper. |
| `RayTraceHelper` | Samples a line of Crest query points and finds a water-surface intersection. |
| `VisualiseCollisionArea` | Debug visualization of collision-query coverage. |
| `VisualiseRayTrace` | Debug visualization for `RayTraceHelper`. |

## Wave spectrum and shape

| Type | Role |
|---|---|
| `OceanWaveSpectrum` | 14-octave spectrum asset; amplitudes, chop, Phillips/PM/JONSWAP generators. |
| `ShapeGerstnerBatched` | Global/geometry Gerstner input; also synchronous CPU `ICollProvider`. |
| `SimSettingsAnimatedWaves` | Collision-source and max-query-count settings. |
| `SimSettingsBase` | Base ScriptableObject for simulation settings. |
| `SimSettingsWave` | Dynamic-wave stability, damping, Courant/substep settings. |
| `SimSettingsFlow` | Flow-settings placeholder asset. |
| `SimSettingsFoam` | Foam fade/wave/shore-generation settings. |
| `SimSettingsShadow` | Jitter/frame weights for water shadows. |
| `RegisterAnimWavesInput` | Registers renderer input into animated-wave data. |
| `RegisterLodDataInputDisplacementCorrection<T>` | Generic input base with displacement correction. |
| `RegisterLodDataInput<T>` | Generic renderer registration into a LOD manager. |
| `RegisterLodDataInputBase` | Static type registrar and common input-draw behavior. |
| `RegisterDynWavesInput` | Dynamic-wave input component. |
| `RegisterFlowInput` | Flow input component. |
| `RegisterFoamInput` | Foam input component. |
| `RegisterClipSurfaceInput` | Clip/holes input with optional distance-to-surface condition. |
| `RegisterSeaFloorDepthInput` | Sea-floor-depth input component. |

## Physics, buoyancy, and dynamic-wave interaction

| Type | Role |
|---|---|
| `FloatingObjectBase` | Base exposing `ObjectWidth`, `InWater`, `Velocity`. |
| `SimpleFloatingObject` | One-point cubic `ForceMode.Acceleration` floater; vanilla item path. |
| `BoatProbes` | Multi-force-point boat buoyancy, drag, engine/turn controls. |
| `FloaterForcePoints` | Serializable `_weight` + `_offsetPosition` for BoatProbes. |
| `ObjectWaterInteraction` | Writes moving-object wake input into dynamic-wave material. |
| `ObjectWaterInteractionAdaptor` | Adds `FloatingObjectBase` behavior to an ordinary object for interaction input. |
| `SphereWaterInteraction` | Sphere-shaped dynamic-wave wake/ripple input. |
| `WaterBody` | Static list of water-area AABBs for culling/effects. |

## Underwater, reflections, depth, and surface visuals

| Type | Role |
|---|---|
| `UnderwaterEffect` | Camera-local underwater mesh; static `cameraWaterHeight`. |
| `OceanPlanarReflection` | Reflection camera/texture for the ocean surface. |
| `PreparedReflections` | Internal camera instance ID → reflection RenderTexture mapping. |
| `OceanDepthCache` | Baked/realtime top-down sea-depth texture generation. |
| `RenderAlphaOnSurface` | Aligns renderer alpha/clip data with ocean surface. |
| `TextureArrayHelpers` | Texture2DArray create/clear/black fallback helpers. |
| `ComputeShaderHelpers` | `Resources.Load<ComputeShader>(path)`. |
| `OceanDebugGUI` | Runtime Crest debug controls/visualization. |
| `BoundsHelper` | Bounds utility/drawing extensions. |
| `AssignLayer` | Assigns a configured layer recursively/on object. |
| `PredicatedFieldAttribute` | Inspector conditional-field metadata. |

## Origin, time, and generic utilities

| Type | Role |
|---|---|
| `FloatingOrigin` | Crest origin shift; moves transforms/particles/LOD and changes distant Rigidbody sleep threshold. |
| `IFloatingOrigin` | `SetOrigin(Vector3)` contract. |
| `ITimeProvider` | `CurrentTime`, `DeltaTime`, `DeltaTimeDynamics`. |
| `TimeProviderBase` | MonoBehaviour base for custom time provider. |
| `TimeProviderDefault` | Unity `Time.time` / `Time.deltaTime`. |
| `TimeProviderCustom` | Inspector-fed `_time`, `_deltaTime`. |
| `CrestSortedList<TKey,TValue>` | Sorted multi-value registrar storage. |
| `DuplicateKeyComparer<TKey>` | Allows duplicate sort keys (render queues). |

## Assembly/global compatibility type

| Type | Role |
|---|---|
| `RenderWireFrame` | Global-namespace helper outside `Crest`; wireframe render/debug utility. |

## What is absent from the C# assembly

Crest logic depends on resource assets absent from C# types:

| Asset class | Examples |
|---|---|
| Compute shaders | `QueryDisplacements`, `QueryFlow`, `ShapeCombine`, `UpdateFoam`, `UpdateShadow`, `ClearToBlack` |
| Hidden shaders | Gerstner Batch Global, shape-combine, and input shaders |
| Inspector values | OceanRenderer flags, spectrum power arrays, BoatProbes force points, prefab materials |
| Scene instances | concrete `OceanRenderer`, `ShapeGerstnerBatched`, WaterBody, reflection/effect objects |

These require runtime dump/scene inspection. ILSpy source reveals code paths, not
serialized prefab/scene values.

## Research priorities exposed by the index

1. **Exact surface height:** `OceanRenderer.Instance` → `CollisionProvider` → `SampleHeightHelper` with bool result.
2. **Predictable fallback:** scene `ShapeGerstnerBatched` CPU API when GPU query is unavailable.
3. **Realistic buoyancy:** study `BoatProbes`, not `SimpleFloatingObject`.
4. **Ripple/wake visuals:** `ObjectWaterInteraction`/`SphereWaterInteraction`; do not mistake them for force physics.
5. **Distant bodies:** account for Crest `FloatingOrigin` physics threshold separately from game `FloatingOriginManager`.
