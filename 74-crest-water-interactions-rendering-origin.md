# 74. Crest interactions, визуал воды и floating origin

Разбор оставшихся runtime-систем `Crest.dll` Sailwind v0.38: `BoatProbes`, object-water interaction, dynamic-wave input, underwater effect, `WaterBody`, depth cache, reflections, ocean chunk rendering и собственный Crest floating origin.

Связано с [70](70-water-splash-particle-systems.md), [71](71-crest-simplefloatingobject-exact-model.md), [72](72-crest-oceanrenderer-query-contract.md), [73](73-crest-wave-spectrum-lod-simulation.md).

## `BoatProbes`: полноценная multi-point плавучесть Crest

`BoatProbes : FloatingObjectBase` — не `SimpleFloatingObject`. Он предназначен для судна и использует массив `FloaterForcePoints`:

```csharp
public class FloaterForcePoints
{
    public float _weight = 1f;
    public Vector3 _offsetPosition;
}
```

В `Start()` BoatProbes:

- получает Rigidbody;
- назначает `_rb.centerOfMass = _centerOfMass`;
- суммирует `_totalWeight` всех force points;
- создаёт query arrays длины `forcePoints + 1`.

В `FixedUpdate()` он query-ит displacement/velocity для каждого force point и центра тела.

### Формула BoatProbes buoyancy

Для каждой точки:

```text
submersion = SeaLevel + displacementY(point) - pointY
```

Если `submersion > 0`:

```csharp
force = 1000 × abs(Physics.gravity.y) × submersion
        × point.weight × forceMultiplier / totalWeight;
rb.AddForceAtPosition(force × Vector3.up, point);
```

В отличие от `SimpleFloatingObject`:

| Свойство | `BoatProbes` | `SimpleFloatingObject` |
|---|---|---|
| Точки | множество force points | одна transform point |
| Lift | линейный по глубине force | кубическая acceleration |
| Масса | Unity `ForceMode.Force` даёт mass/inertia effect | `Acceleration` отменяет массу |
| Query | batch point arrays | один helper query |
| Назначение | судно | простой float object/item |

`BoatProbes` — полезный vanilla reference для mod hull buoyancy: distributed force points, one query batch, force at position. Но его коэффициенты и веса prefab-specific; не копируйте их вслепую для cargo.

## `ObjectWaterInteraction` и `SphereWaterInteraction`

Эти классы не добавляют Rigidbody buoyancy. Они пишут velocity/weight в `MaterialPropertyBlock` для **dynamic wave simulation**.

### `ObjectWaterInteraction`

Если parent не наследует `FloatingObjectBase`, он добавляет `ObjectWaterInteractionAdaptor`. Каждый `LateUpdate`:

1. ищет active dynamic-wave simulations на соответствующем LOD;
2. ставит interaction object впереди по velocity (`_velocityPositionOffset = 0.2`);
3. считает velocity relative to flow;
4. сбрасывает velocity при teleport speed >500 km/h;
5. clamp-ит скорость до 100 km/h;
6. пишет `_Velocity`, `_Weight`, `_SimDeltaTime` в material property block.

`_Weight = 1 / activeSims`, но только когда parent `FloatingObjectBase.InWater`.

### `SphereWaterInteraction`

Сферический input для wake/ripple:

- `Radius = 0.5 × transform.lossyScale.x`;
- запрашивает water height с spatial length `2 × Radius`;
- `_weight` по умолчанию 1;
- ослабляет weight как над водой, так и глубоко под водой;
- пишет `_Radius`, `_Velocity`, `_Weight`, `_SimDeltaTime` shader properties.

> Эти interaction classes — visual/dynamic-wave input. Они не заменяют collision, mass или buoyancy physics body.

## `ObjectWaterInteractionAdaptor`

Если у parent нет `FloatingObjectBase`, adaptor создаётся автоматически. Он:

- query-ит одну point displacement в `Update`;
- считает `InWater` как `transform.position.y - sampledHeight <= 0`;
- вычисляет transform velocity по предыдущему frame.

Это хороший пример того, как Crest bridge превращает обычный GO в input для visual wave interaction, но не делает его физически плавающим.

## `UnderwaterEffect`

`Crest.UnderwaterEffect` публично хранит:

```csharp
public static float cameraWaterHeight;
```

В `LateUpdate`:

1. query-ит `SampleHeightHelper` в позиции effect object;
2. пишет `cameraWaterHeight = sampledHeight`;
3. включает renderer, если камера менее чем `_maxHeightAboveWater` (default 1.5) над водой;
4. копирует material ocean и привязывает animated-wave/depth/shadow LOD data.

### Ограничение для моддера

`cameraWaterHeight` — это высота **в позиции камеры/effect**, не query в произвольной позиции предмета. Она годится как дешёвый sea-level fallback или player swimming input, но не как точная волновая поверхность cargo далеко от камеры.

## `WaterBody`

`WaterBody` поддерживает static list `WaterBodies` и AABB одного transform quad:

```csharp
public static List<WaterBody> WaterBodies
public Bounds AABB { get; private set; }
```

`OceanRenderer.LateUpdateTiles()` использует этот список для culling ocean chunks; `UnderwaterEffect` при `_turnOffOutsideWaterBodies` выключается за пределами всех AABB.

Для mod water-volume logic это означает: WaterBody — XZ area/culling marker, **не** физический volume collider и не источник высоты.

## Ocean depth cache

`OceanDepthCache` создаёт top-down orthographic camera и рендерит заданные layers/renderer-ы shader-ом:

```text
Crest/Inputs/Depth/Ocean Depth From Geometry
```

Результат — RHalf depth texture. Modes:

| Mode | Поведение |
|---|---|
| `Realtime` + `OnStart` | `PopulateCache()` при Start |
| `Realtime` + `OnDemand` | caller обязан вызвать `PopulateCache()` |
| `Baked` | использует serialized `_savedCache` |

Cache используется shore/shallows/foam, но сам по себе не является collision mesh.

## Ocean tiles и render bounds

`OceanBuilder.GenerateMesh()`:

- создаёт hidden Root под `OceanRenderer`;
- строит 10 типов patch mesh;
- создаёт 16 tiles для LOD0, 12 для остальных LOD;
- назначает слой по `_layerName` (`Water` default);
- крайний skirt patch может растягивать edge vertices ×100.

`OceanChunkRenderer.ExpandBoundsForDisplacements()` расширяет render bounds на:

```text
horizontal = OceanRenderer.MaxHorizDisplacement
vertical   = OceanRenderer.MaxVertDisplacement + 5
```

Следствие: renderer bounds ocean tile намеренно огромны. Их нельзя использовать как физический hull/object bounds.

## Planar reflections

`OceanPlanarReflection` создаёт отдельную reflection camera и RenderTexture.

Ключевые параметры:

| Поле | Default |
|---|---:|
| `_textureSize` | 256 |
| `_clipPlaneOffset` | 0.07 |
| `_hdr` | true |
| `_farClipPlane` | 1000 |
| `RefreshPerFrames` | 1 |

Reflection texture хранится в internal `PreparedReflections` по `camera.GetHashCode()` и подаётся в `OceanChunkRenderer` как `_ReflectionTex`.

## Crest `FloatingOrigin`

Это отдельный компонент Crest, не Sailwind `FloatingOriginManager`.

Defaults:

| Поле | Default |
|---|---:|
| `_threshold` | 16384 |
| `_physicsThreshold` | 1000 |
| `_defaultSleepThreshold` | 0.14 |

При превышении X/Z threshold:

1. сдвигает root transforms;
2. сдвигает world-space particles;
3. вызывает `SetOrigin` на `LodTransform`, `IFloatingOrigin` children и `ShapeGerstnerBatched`;
4. для Rigidbody дальше physics threshold устанавливает `sleepThreshold = float.MaxValue`.

Это отдельный потенциальный источник physics behaviour на дальних расстояниях. Mod, работающий с ocean/body persistence, должен различать Crest origin и игровой `FloatingOriginManager`.

## Shader inputs и полезные компоненты

| Тип | Что регистрирует |
|---|---|
| `RegisterAnimWavesInput` | wave displacement; может report max displacement |
| `RegisterDynWavesInput` | dynamic waves |
| `RegisterFlowInput` | flow |
| `RegisterFoamInput` | foam |
| `RegisterClipSurfaceInput` | holes / clipped water surface |
| `RegisterSeaFloorDepthInput` | depth |
| `RenderAlphaOnSurface` | material alpha/clip data на поверхности |

`RegisterLodDataInputBase.Draw()` query-ит displacement в позиции input и пишет `_DisplacementAtInputPosition`, чтобы input shader учитывал horizontal movement поверхности.

## Практические выводы

1. Для реалистичной mod buoyancy ближе всего к полезному reference `BoatProbes`, а не `SimpleFloatingObject`.
2. Dynamic-wave interaction — это render input, не force physics.
3. `cameraWaterHeight` — camera-local fallback, не global wave API.
4. `WaterBody` — area/culling marker, не water volume collider.
5. Bounds Crest ocean tile намеренно расширены и непригодны для оценки размера физического предмета.
6. Crest FloatingOrigin может менять sleep threshold Rigidbody за 1000 units; это важно для дальнего cargo/LOD.
