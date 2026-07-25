# 73. Crest waves: Gerstner spectrum, LOD data и симуляционные слои

Разбор генерации волны и LOD pipeline из `Crest.dll` Sailwind v0.38. Покрывает `OceanWaveSpectrum`, `ShapeGerstnerBatched`, `LodTransform`, `LodDataMgr*`, dynamic waves, flow, foam, depth и shader input registration.

Связано с [72](72-crest-oceanrenderer-query-contract.md).

## Карта данных Crest

```text
OceanWaveSpectrum + ShapeGerstnerBatched
  → RegisterLodDataInput<LodDataMgrAnimWaves>
  → LodDataMgrAnimWaves RenderTexture array (ARGBHalf)
  → ShapeCombine command/compute pass
  → CollisionProvider / ocean surface shader

Дополнительные слои:
  SeaFloorDepth  (RHalf)
  Flow           (RGHalf)
  DynamicWaves   (persistent render textures)
  Foam           (persistent RHalf)
  ClipSurface
  Shadow         (RG16)
```

Все LOD result textures создаются как `TextureDimension.Tex2DArray`; `volumeDepth = CurrentLodCount`.

## `LodTransform`: scale и позиционирование LOD

`LodTransform` содержит `RenderData` для каждого LOD:

| Поле | Содержание |
|---|---|
| `_texelWidth` | ширина texel в world units |
| `_textureRes` | resolution LOD texture |
| `_posSnapped` | XZ позиция, snapped к texel grid |
| `_frame` | frame обновления |

Для LOD `i`:

```text
lodScale = OceanRenderer.Scale × 2^i
texelWidth = 2 × (2 × lodScale) / LodDataResolution
```

`OceanRenderer.LateUpdateScale()` выбирает `Scale` как степень двойки, исходя из высоты viewpoint над водой. Это снижает popping при переключении LOD; `ViewerAltitudeLevelAlpha` используется для blend между крайними уровнями.

### `MaxWavelength`

```text
maxWavelength(lod) = 2 × [4 × OceanRenderer.Scale × 2^lod / resolution]
                   × MinTexelsPerWave
```

`LodDataMgrAnimWaves.FilterWavelength` распределяет wave input по LOD только в его допустимый диапазон wavelength.

## `OceanWaveSpectrum`

Spectrum содержит **14 октав**:

```csharp
NUM_OCTAVES = 14
SMALLEST_WL_POW_2 = -4
SmallWavelength(octave) = 2^(-4 + octave)
```

Базовый ряд wavelength без project scaling:

```text
0.0625, 0.125, 0.25, ... , 512 units
```

Важные параметры:

| Поле | Default | Роль |
|---|---:|---|
| `_windSpeed` | 10 | input для генерации физического spectrum asset |
| `_fetch` | 500000 | fetch для JONSWAP |
| `_waveDirectionVariance` | 90° | разброс направления component |
| `_gravityScale` | 1 | multiplier в spectrum context |
| `_smallWavelengthMultiplier` | 1 | scale коротких волн |
| `_multiplier` | 1 | общий amplitude multiplier |
| `_chop` | 1.6 | горизонтальное displacement/choppiness |
| `_powerLog[14]` | serialized | log10 power каждой октавы |
| `_chopScales[14]` | 1 | per-octave chop |
| `_gravityScales[14]` | 1 | serialized per-octave gravity data |

### Скорость глубоководной волны

```csharp
ComputeWaveSpeed(wavelength, gravityMultiplier)
    = sqrt((abs(Physics.gravity.y) × gravityMultiplier) / (2π / wavelength))
```

То есть:

```text
phase speed = sqrt(g × wavelength / 2π)
```

### Amplitude component

`GetAmplitude(wavelength, componentsPerOctave)` интерполирует log-power между октавами, учитывает bandwidth и возвращает:

```text
sqrt(2 × 10^power × spectralBandwidth) × random × multiplier
```

`random` берётся из `UnityEngine.Random.value`; `ShapeGerstnerBatched.UpdateWaveData()` временно устанавливает Random seed `_randomSeed`, затем восстанавливает global Random state. Это делает wave set воспроизводимым при одинаковом seed/spectrum.

### Готовые spectrum methods

| Метод | Модель |
|---|---|
| `ApplyPhillipsSpectrum` | Phillips |
| `ApplyPiersonMoskowitzSpectrum` | Pierson–Moskowitz |
| `ApplyJONSWAPSpectrum` | JONSWAP с wind/fetch |

JONSWAP использует peak enhancement `γ = 3.3`.

## `ShapeGerstnerBatched`

Это главный global/local Gerstner input и одновременно возможный CPU `ICollProvider`.

### Режимы

```csharp
public enum GerstnerMode
{
    Global,    // invisible generated Quad, waves everywhere
    Geometry   // renderer этого GameObject рендерится в wave data
}
```

В `Global` mode создаётся скрытый Quad с shader:

```text
Hidden/Crest/Inputs/Animated Waves/Gerstner Batch Global
```

### Компоненты

- `_componentsPerOctave = 8` default;
- 14 октав → до **112** wave components;
- GPU batch limit = **32** component на batch;
- на shader передаются массивы `twoPiOverWavelength`, amplitudes, wave dir X/Z, phases, chop amplitudes.

### CPU API

`ShapeGerstnerBatched` реализует `ICollProvider` и имеет:

```csharp
bool SampleHeight(ref Vector3 worldPos, float minSpatialLength, out float height)
bool SampleDisplacement(ref Vector3 worldPos, float minSpatialLength, out Vector3 displacement)
bool SampleNormal(ref Vector3 undisplacedPos, float minSpatialLength, out Vector3 normal)
bool GetSurfaceVelocity(ref Vector3 worldPos, float minSpatialLength, out Vector3 velocity)
```

`ComputeUndisplacedPosition()` делает до **4** итераций, компенсируя horizontal chop перед height/normal sample.

В `SampleDisplacement` component с `wavelength < minSpatialLength / 2` исключаются. Это тот же принцип spatial filtering, что нужен физике корпуса: маленькая лодка не должна реагировать на каждую микроволну.

### Формула displacement

Для component:

```text
k       = 2π / wavelength
phase   = k × (dot(direction, worldXZ) + phaseSpeed × time) + randomPhase
horizontal = amplitude × (-chop) × sin(phase) × direction
vertical   = amplitude × cos(phase)
```

`GetSurfaceVelocity` аналитически вычисляет производную этой поверхности по времени.

## Compute collision query vs CPU Gerstner

| Свойство | `QueryDisplacements` | `ShapeGerstnerBatched` CPU |
|---|---|---|
| Источник | итоговая animated-wave texture | только Gerstner components данного shape |
| Readback | async GPU, возможны stale/failure | синхронный C# |
| Dynamic waves / flow | может попасть в combined data | нет в самом Gerstner displacement |
| Массовые body | нужен batching/query budget | CPU cost O(components × samples) |
| Надёжность в fixed physics | нужен lifecycle/result guard | предсказуем при существующем shape |

Для mod physics CPU Gerstner может быть разумным fallback/diagnostic source, но только если найден правильный scene instance и mod осознанно принимает отсутствие dynamic-wave layer.

## Animated wave data

`LodDataMgrAnimWaves`:

- result texture format: `ARGBHalf`;
- отдельный `WaveBuffer` и `CombineBuffer`;
- сначала рендерит inputs per LOD с `FilterWavelength`;
- затем combine pass объединяет wave buffers, previous LOD, dynamic waves и flow;
- имеет `_readbackShapeForCollision = true` field (collision query intent);
- может использовать material ping-pong или compute `ShapeCombine` kernels.

## Dynamic waves

`LodDataMgrDynWaves` наследуется от persistent sim. `SimSettingsWave`:

| Поле | Default | Роль |
|---|---:|---|
| `_damping` | 0.25 | dissipate energy per frame |
| `_courantNumber` | 1 | stability / timestep criterion |
| `_maxSimStepsPerFrame` | 3 | cap substeps |
| `_horizDisplace` | 3 | sharpen horizontal displacement |
| `_displaceClamp` | 0.3 | prevent self-intersection |
| `_gravityMultiplier` | 1 | wave propagation speed |

Максимальный stable timestep для LOD:

```text
maxDt = 0.5 × courantNumber × gridSize / waveSpeed(maxWavelength(lod)/2)
```

Фактические substeps:

```text
ceil(frameDt / maxDt), clamped to [1, maxSimStepsPerFrame]
```

## Flow, foam, depth, clip, shadow

| Manager | Texture | Назначение |
|---|---|---|
| `LodDataMgrFlow` | `RGHalf` | горизонтальный current; включает `CREST_FLOW_ON_INTERNAL` |
| `LodDataMgrFoam` | configured, default `RHalf` | persistent foam; wave/shoreline generation |
| `LodDataMgrSeaFloorDepth` | `RHalf` | depth/shallows/shoreline inputs |
| `LodDataMgrClipSurface` | LOD data | дырки/clip surface ocean geometry |
| `LodDataMgrShadow` | `RG16` | water shadow data из directional light |

Foam defaults:

```text
fadeRate=0.8, waveStrength=1, waveCoverage=0.8,
shorelineMaxDepth=0.65, shorelineStrength=2
```

## Input registration

`RegisterLodDataInputBase` хранит static registrar по типу `LodDataMgr`; inputs сортируются по render queue. Производные классы:

| Component | Цель |
|---|---|
| `RegisterAnimWavesInput` | animation/wave displacement |
| `RegisterDynWavesInput` | dynamic-wave sim input |
| `RegisterFlowInput` | flow vector input |
| `RegisterFoamInput` | foam input |
| `RegisterClipSurfaceInput` | clip/holes |
| `RegisterSeaFloorDepthInput` | sea-floor depth |
| `RegisterLodDataInputDisplacementCorrection<T>` | input с correction для displacement у input position |

`RegisterAnimWavesInput` может сообщать ocean max vertical/horizontal displacement; это влияет на render bounds ocean tiles.

## Практические выводы

1. Crest wave surface — не один noise function: 14 октав, LOD filtering, combine pass, optional dynamic/flow layers.
2. `minSpatialLength` — физический filter knob: его нужно связывать с шириной объекта.
3. Для deterministic собственных волн можно воспроизвести CPU Gerstner path с seed/spectrum; не нужно угадывать random noise.
4. GPU query результаты могут быть stale, CPU Gerstner — нет, но они не идентичны полной combined texture.
5. Dynamic wave sim и foam имеют отдельные timestep/stability ограничения; не путайте их с buoyancy.
