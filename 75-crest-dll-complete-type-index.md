# 75. `Crest.dll` Sailwind v0.38: полный индекс 80 типов

Полный source-map assembly `Crest.dll`, декомпилированного из Sailwind v0.38. Эта заметка не заменяет тематические [71](71-crest-simplefloatingobject-exact-model.md)–[74](74-crest-water-interactions-rendering-origin.md), а гарантирует, что ни один тип assembly не потерян в исследовании.

## Как читать индекс

- **Runtime** — тип можно использовать/наблюдать в запущенной игре.
- **Input** — Unity component, регистрирующий renderer/material в Crest LOD data.
- **Internal utility** — инфраструктура; обычно не точка API для мода.
- Имена файлов соответствуют namespace `Crest`, кроме `RenderWireFrame` в global namespace.

## Core ocean, LOD и command buffers

| Type | Роль |
|---|---|
| `OceanRenderer` | Главный Crest singleton; root, LOD, providers, command buffer lifecycle. |
| `OceanBuilder` | Строит patch meshes и ocean tile hierarchy. |
| `OceanChunkRenderer` | Renderer одного tile; bind LOD textures, reflection, render bounds. |
| `LodTransform` | Snapped LOD positions, texel scale, view/projection matrices; `IFloatingOrigin`. |
| `LodDataMgr` | Abstract base для всех LOD texture managers. |
| `LodDataMgrAnimWaves` | Animated wave buffers/combine/collision shape data. |
| `LodDataMgrClipSurface` | Clip/holes texture layer. |
| `LodDataMgrDynWaves` | Persistent dynamic-wave simulation. |
| `LodDataMgrFlow` | Flow texture layer. |
| `LodDataMgrFoam` | Persistent foam simulation. |
| `LodDataMgrPersistent` | Abstract ping-pong persistent simulation base. |
| `LodDataMgrSeaFloorDepth` | Sea floor depth texture layer. |
| `LodDataMgrShadow` | Water shadow texture/update from directional light. |
| `BuildCommandBufferBase` | Static last-update-frame base. |
| `BuildCommandBuffer` | Builds and executes all active LOD command buffers. |
| `ILodDataInput` | Input contract: wavelength, enabled, draw. |
| `IPropertyWrapper` | Abstract material/compute property setter contract. |
| `PropertyWrapperMaterial` | Material-backed property writer. |
| `PropertyWrapperMPB` | `MaterialPropertyBlock`-backed writer. |
| `PropertyWrapperCompute` | CommandBuffer compute-shader writer. |
| `PropertyWrapperComputeStandalone` | Direct compute-shader writer for queries. |

## CPU/GPU collision and flow queries

| Type | Роль |
|---|---|
| `ICollProvider` | Displacement/height/normal/velocity query interface. |
| `IFlowProvider` | XZ flow query interface. |
| `QueryBase` | GPU compute + async readback ring buffer, query registration/status. |
| `QueryDisplacements` | `ICollProvider` using `QueryDisplacements` compute shader. |
| `QueryFlow` | `IFlowProvider` using `QueryFlow` compute shader. |
| `CollProviderNull` | Successful zero-displacement fallback. |
| `FlowProviderNull` | Successful zero-flow fallback. |
| `SampleHeightHelper` | Convenient one-point height/displacement/normal/velocity query wrapper. |
| `SampleFlowHelper` | Convenient one-point XZ flow wrapper. |
| `RayTraceHelper` | Samples a line of Crest query points and finds a water-surface intersection. |
| `VisualiseCollisionArea` | Debug visualization of collision query coverage. |
| `VisualiseRayTrace` | Debug visualization for `RayTraceHelper`. |

## Wave spectrum and shape

| Type | Роль |
|---|---|
| `OceanWaveSpectrum` | 14-octave spectrum asset; amplitudes, chop, Phillips/PM/JONSWAP generators. |
| `ShapeGerstnerBatched` | Global/geometry Gerstner input; also synchronous CPU `ICollProvider`. |
| `SimSettingsAnimatedWaves` | Collision-source and max-query-count settings. |
| `SimSettingsBase` | Base ScriptableObject for simulation settings. |
| `SimSettingsWave` | Dynamic-wave stability, damping, Courant/substep settings. |
| `SimSettingsFlow` | Flow settings placeholder asset. |
| `SimSettingsFoam` | Foam fade/wave/shore generation settings. |
| `SimSettingsShadow` | Jitter/frame weights for water shadows. |
| `RegisterAnimWavesInput` | Registers renderer input into animated wave data. |
| `RegisterLodDataInputDisplacementCorrection<T>` | Generic input base correcting for surface displacement. |
| `RegisterLodDataInput<T>` | Generic renderer registration into a specific LOD manager. |
| `RegisterLodDataInputBase` | Static type registrar and common input draw behavior. |
| `RegisterDynWavesInput` | Dynamic-wave input component. |
| `RegisterFlowInput` | Flow input component. |
| `RegisterFoamInput` | Foam input component. |
| `RegisterClipSurfaceInput` | Clip/holes input with optional distance-to-surface condition. |
| `RegisterSeaFloorDepthInput` | Sea-floor-depth input component. |

## Physics, buoyancy и dynamic-wave interaction

| Type | Роль |
|---|---|
| `FloatingObjectBase` | Base for objects exposing `ObjectWidth`, `InWater`, `Velocity`. |
| `SimpleFloatingObject` | One-point cubic `ForceMode.Acceleration` floater; предметный vanilla path. |
| `BoatProbes` | Multi-force-point boat buoyancy, drag, engine/turn controls. |
| `FloaterForcePoints` | Serializable `_weight` + `_offsetPosition` для BoatProbes. |
| `ObjectWaterInteraction` | Writes moving-object wake input into dynamic-wave material. |
| `ObjectWaterInteractionAdaptor` | Adds `FloatingObjectBase` behavior to ordinary object for interaction input. |
| `SphereWaterInteraction` | Sphere-shaped dynamic-wave wake/ripple input. |
| `WaterBody` | Static list of water-area AABBs for culling/effects. |

## Underwater, reflections, depth и surface visuals

| Type | Роль |
|---|---|
| `UnderwaterEffect` | Camera-local underwater mesh; static `cameraWaterHeight`. |
| `OceanPlanarReflection` | Reflection camera/texture for ocean surface. |
| `PreparedReflections` | Internal mapping camera instance ID → reflection RenderTexture. |
| `OceanDepthCache` | Baked/realtime top-down sea-depth texture generation. |
| `RenderAlphaOnSurface` | Aligns renderer alpha/clip data with ocean surface. |
| `TextureArrayHelpers` | Texture2DArray create/clear/black fallback helpers. |
| `ComputeShaderHelpers` | `Resources.Load<ComputeShader>(path)`. |
| `OceanDebugGUI` | Runtime Crest debug controls/visualization. |
| `BoundsHelper` | Bounds utility/drawing extensions. |
| `AssignLayer` | Assigns a configured layer recursively/on object. |
| `PredicatedFieldAttribute` | Inspector conditional-field metadata. |

## Origin, time и generic utilities

| Type | Роль |
|---|---|
| `FloatingOrigin` | Crest own origin shifting; moves transforms/particles/LOD and changes distant Rigidbody sleep threshold. |
| `IFloatingOrigin` | `SetOrigin(Vector3)` contract. |
| `ITimeProvider` | `CurrentTime`, `DeltaTime`, `DeltaTimeDynamics`. |
| `TimeProviderBase` | MonoBehaviour base for custom time provider. |
| `TimeProviderDefault` | Unity `Time.time` / `Time.deltaTime`. |
| `TimeProviderCustom` | Inspector-fed `_time`, `_deltaTime`. |
| `CrestSortedList<TKey,TValue>` | Sorted multi-value registrar storage. |
| `DuplicateKeyComparer<TKey>` | Allows duplicate sort keys (render queues). |

## Assembly/global compatibility types

| Type | Роль |
|---|---|
| `RenderWireFrame` | Global-namespace helper outside `Crest`; wireframe rendering/debug utility. |

## Что отсутствует из C# assembly

Crest logic зависит от resource assets, не содержащихся в C# типах:

| Asset class | Примеры |
|---|---|
| Compute shaders | `QueryDisplacements`, `QueryFlow`, `ShapeCombine`, `UpdateFoam`, `UpdateShadow`, `ClearToBlack` |
| Hidden shaders | `Hidden/Crest/Inputs/Animated Waves/Gerstner Batch Global`, shape combine и input shaders |
| Inspector values | OceanRenderer flags, spectrum power arrays, BoatProbes force points, prefab materials |
| Scene instances | конкретный `OceanRenderer`, `ShapeGerstnerBatched`, WaterBody, reflection camera/effect objects |

Их нужно получать runtime dump-ом/scene inspection; ILSpy source сообщает code path, но не serialized prefab/scene values.

## Исследовательские приоритеты по индексу

1. **Точная surface height:** `OceanRenderer.Instance` → `CollisionProvider` → `SampleHeightHelper` с bool result.
2. **Предсказуемый fallback:** scene `ShapeGerstnerBatched` CPU API, если GPU query недоступен.
3. **Реалистичная buoyancy:** изучать `BoatProbes`, а не переносить `SimpleFloatingObject`.
4. **Ripple/wake visual:** `ObjectWaterInteraction`/`SphereWaterInteraction`; не путать с force physics.
5. **Дальние тела:** учитывать Crest `FloatingOrigin` physics threshold и game `FloatingOriginManager` отдельно.
