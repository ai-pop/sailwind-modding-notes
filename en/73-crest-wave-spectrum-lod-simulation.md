# 73. Crest waves: Gerstner spectrum, LOD data, and simulation layers

A review of wave generation and the LOD pipeline in Sailwind v0.38's `Crest.dll`. Covers `OceanWaveSpectrum`, `ShapeGerstnerBatched`, `LodTransform`, `LodDataMgr*`, dynamic waves, flow, foam, depth, and shader-input registration.

Related to [72](72-crest-oceanrenderer-query-contract.md).

## Crest data map

```text
OceanWaveSpectrum + ShapeGerstnerBatched
  → RegisterLodDataInput<LodDataMgrAnimWaves>
  → LodDataMgrAnimWaves RenderTexture array (ARGBHalf)
  → ShapeCombine command/compute pass
  → CollisionProvider / ocean surface shader

Additional layers:
  SeaFloorDepth  (RHalf)
  Flow           (RGHalf)
  DynamicWaves   (persistent render textures)
  Foam           (persistent RHalf)
  ClipSurface
  Shadow         (RG16)
```

All LOD result textures are `TextureDimension.Tex2DArray` with
`volumeDepth = CurrentLodCount`.

## `LodTransform`: LOD scale and positioning

`LodTransform` retains `RenderData` per LOD:

| Field | Meaning |
|---|---|
| `_texelWidth` | texel width in world units |
| `_textureRes` | LOD texture resolution |
| `_posSnapped` | XZ position snapped to texel grid |
| `_frame` | update frame |

For LOD `i`:

```text
lodScale = OceanRenderer.Scale × 2^i
texelWidth = 2 × (2 × lodScale) / LodDataResolution
```

`OceanRenderer.LateUpdateScale()` chooses `Scale` as a power of two from
viewpoint height above water. `ViewerAltitudeLevelAlpha` blends extreme LODs to
reduce popping.

### `MaxWavelength`

```text
maxWavelength(lod) = 2 × [4 × OceanRenderer.Scale × 2^lod / resolution]
                   × MinTexelsPerWave
```

`LodDataMgrAnimWaves.FilterWavelength` assigns a wave input to the LOD range
that can represent its wavelength.

## `OceanWaveSpectrum`

The spectrum has **14 octaves**:

```csharp
NUM_OCTAVES = 14
SMALLEST_WL_POW_2 = -4
SmallWavelength(octave) = 2^(-4 + octave)
```

The baseline wavelength sequence is:

```text
0.0625, 0.125, 0.25, ... , 512 units
```

Important parameters:

| Field | Default | Role |
|---|---:|---|
| `_windSpeed` | 10 | input to physical-spectrum asset generation |
| `_fetch` | 500000 | JONSWAP fetch |
| `_waveDirectionVariance` | 90° | component direction spread |
| `_gravityScale` | 1 | spectrum-context gravity multiplier |
| `_smallWavelengthMultiplier` | 1 | short-wave scale |
| `_multiplier` | 1 | global amplitude multiplier |
| `_chop` | 1.6 | horizontal displacement/choppiness |
| `_powerLog[14]` | serialized | log10 power for each octave |
| `_chopScales[14]` | 1 | per-octave chop |
| `_gravityScales[14]` | 1 | serialized per-octave gravity data |

### Deep-water wave speed

```csharp
ComputeWaveSpeed(wavelength, gravityMultiplier)
    = sqrt((abs(Physics.gravity.y) × gravityMultiplier) / (2π / wavelength))
```

Therefore:

```text
phase speed = sqrt(g × wavelength / 2π)
```

### Component amplitude

`GetAmplitude(wavelength, componentsPerOctave)` interpolates log-power between
octaves, accounts for spectral bandwidth, and returns:

```text
sqrt(2 × 10^power × spectralBandwidth) × random × multiplier
```

`ShapeGerstnerBatched.UpdateWaveData()` temporarily seeds `UnityEngine.Random`
with `_randomSeed`, then restores global state. The same seed/spectrum produces
a reproducible wave set.

### Built-in spectrum generators

| Method | Model |
|---|---|
| `ApplyPhillipsSpectrum` | Phillips |
| `ApplyPiersonMoskowitzSpectrum` | Pierson–Moskowitz |
| `ApplyJONSWAPSpectrum` | JONSWAP with wind/fetch |

JONSWAP uses peak enhancement `γ = 3.3`.

## `ShapeGerstnerBatched`

This is the primary global/local Gerstner input and can also act as a CPU
`ICollProvider`.

### Modes

```csharp
public enum GerstnerMode
{
    Global,    // generated invisible Quad; waves everywhere
    Geometry   // this GameObject's renderer is drawn into wave data
}
```

Global mode creates a hidden Quad with:

```text
Hidden/Crest/Inputs/Animated Waves/Gerstner Batch Global
```

### Components

- `_componentsPerOctave = 8` by default;
- 14 octaves → up to **112** wave components;
- GPU batch limit = **32** components;
- shader arrays receive `twoPiOverWavelength`, amplitudes, wave dir X/Z, phases,
  and chop amplitudes.

### CPU API

`ShapeGerstnerBatched` implements `ICollProvider` and exposes:

```csharp
bool SampleHeight(ref Vector3 worldPos, float minSpatialLength, out float height)
bool SampleDisplacement(ref Vector3 worldPos, float minSpatialLength, out Vector3 displacement)
bool SampleNormal(ref Vector3 undisplacedPos, float minSpatialLength, out Vector3 normal)
bool GetSurfaceVelocity(ref Vector3 worldPos, float minSpatialLength, out Vector3 velocity)
```

`ComputeUndisplacedPosition()` performs up to **4** iterations to compensate
horizontal chop before height/normal sampling.

Components with `wavelength < minSpatialLength / 2` are excluded. This is the
same spatial-filtering principle physical hulls need: a small boat should not
respond to every micro-wave.

### Displacement form

For one component:

```text
k       = 2π / wavelength
phase   = k × (dot(direction, worldXZ) + phaseSpeed × time) + randomPhase
horizontal = amplitude × (-chop) × sin(phase) × direction
vertical   = amplitude × cos(phase)
```

`GetSurfaceVelocity` analytically derives this surface with respect to time.

## Compute collision query vs CPU Gerstner

| Property | `QueryDisplacements` | CPU `ShapeGerstnerBatched` |
|---|---|---|
| Source | final animated-wave texture | only Gerstner components in that shape |
| Readback | async GPU; stale/failure possible | synchronous C# |
| Dynamic waves / flow | may be present in combined data | absent from Gerstner displacement itself |
| Many bodies | needs query batching/budget | CPU cost O(components × samples) |
| Fixed-physics behavior | needs lifecycle/result guard | predictable once the correct shape exists |

CPU Gerstner can be a sound fallback/diagnostic source, but only if the correct
scene shape is found and the mod intentionally accepts missing dynamic-wave
layers.

## Animated-wave data

`LodDataMgrAnimWaves`:

- result format: `ARGBHalf`;
- separate `WaveBuffer` and `CombineBuffer`;
- renders inputs per LOD through `FilterWavelength`;
- combine pass merges wave buffers, prior LOD, dynamic waves, and flow;
- has `_readbackShapeForCollision = true`;
- can use material ping-pong or `ShapeCombine` compute kernels.

## Dynamic waves

`LodDataMgrDynWaves` is a persistent simulation. `SimSettingsWave`:

| Field | Default | Role |
|---|---:|---|
| `_damping` | 0.25 | energy dissipation per frame |
| `_courantNumber` | 1 | stability/timestep criterion |
| `_maxSimStepsPerFrame` | 3 | substep cap |
| `_horizDisplace` | 3 | sharpen horizontal displacement |
| `_displaceClamp` | 0.3 | prevent self-intersection |
| `_gravityMultiplier` | 1 | wave propagation speed |

Maximum stable timestep for a LOD:

```text
maxDt = 0.5 × courantNumber × gridSize / waveSpeed(maxWavelength(lod)/2)
```

Actual substeps:

```text
ceil(frameDt / maxDt), clamped to [1, maxSimStepsPerFrame]
```

## Flow, foam, depth, clip, shadow

| Manager | Texture | Purpose |
|---|---|---|
| `LodDataMgrFlow` | `RGHalf` | horizontal current; enables `CREST_FLOW_ON_INTERNAL` |
| `LodDataMgrFoam` | configured, default `RHalf` | persistent foam, wave/shore generation |
| `LodDataMgrSeaFloorDepth` | `RHalf` | depth/shallows/shoreline inputs |
| `LodDataMgrClipSurface` | LOD data | holes/clip surface geometry |
| `LodDataMgrShadow` | `RG16` | water-shadow data from directional light |

Foam defaults:

```text
fadeRate=0.8, waveStrength=1, waveCoverage=0.8,
shorelineMaxDepth=0.65, shorelineStrength=2
```

## Input registration

`RegisterLodDataInputBase` stores a static registrar per `LodDataMgr` type;
inputs are sorted by render queue. Derived components:

| Component | Target |
|---|---|
| `RegisterAnimWavesInput` | animated-wave displacement |
| `RegisterDynWavesInput` | dynamic-wave simulation input |
| `RegisterFlowInput` | flow-vector input |
| `RegisterFoamInput` | foam input |
| `RegisterClipSurfaceInput` | clips/holes |
| `RegisterSeaFloorDepthInput` | sea-floor depth |
| `RegisterLodDataInputDisplacementCorrection<T>` | input with displacement correction at input position |

`RegisterAnimWavesInput` can report maximum vertical/horizontal displacement to
the ocean, which changes ocean-tile render bounds.

## Practical implications

1. Crest surface is not one noise function: 14 octaves, LOD filtering, combine
   pass, and optional dynamic/flow layers.
2. `minSpatialLength` is a physical filtering control and should relate to body width.
3. A deterministic custom wave model can reproduce the CPU Gerstner path using
   seed/spectrum rather than guessed random noise.
4. GPU query results can be stale; CPU Gerstner is not, but it differs from the
   final combined texture.
5. Dynamic waves and foam have independent timestep/stability limits; do not
   confuse them with buoyancy.
