# 31. Ocean and Waves: Wave Field Inertia

Analysis of the wave model. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. The ocean is a custom fork of **Crest**; boat buoyancy described in note 14, wind in note 17.

## Wave Inertia (`WavesInertia`)

The wave field **doesn't follow wind instantly** — it has inertia (like real sea: waves slowly rotate and build height).

| Field | Content |
|------|-----------|
| `wind` | Reference to `Wind`. |
| `directionChangeRate` | Wave rotation speed toward wind. |
| `inertiaChangeRate` | Inertia change speed. |
| `magnitudeChangeRate` | Wave height change speed. |
| `currentInertia` | Current inertia (min 1, cap 70 on growth). |
| `currentMagnitude` | Current wave height (min 1). |

### Wave Rotation (`Update`)
```
factor = max(0.25, 10 / currentInertia)
transform.rotation = Slerp(transform.rotation, wind.transform.rotation, dt * directionChangeRate * factor)
```
The **higher the inertia**, the **slower** waves rotate toward wind (factor is smaller).

### Inertia Change (`UpdateCurrentInertia`)
Depends on angle between wind direction and waves (`Quaternion.Angle`):
- **Angle ≤ 45°** (waves nearly following wind) → inertia **grows** (up to 70): waves "calm down", stabilize with wind.
- **Angle > 90°** (waves across wind) → inertia **drops**: field rebuilds faster.
- Cap: minimum `1`.

### Wave Height (`currentMagnitude`)
- Smoothly catches up to wind force: if `currentMagnitude < Wind.currentWind.magnitude` → grows, otherwise drops, at `magnitudeChangeRate`. Minimum `1`.
- So **wave height ≈ wind force**, but with delay.

### Persistence
`LoadInertia(rotation, inertia, magnitude)` restores from save; saves `wavesRotation`, `wavesInertia`, `wavesMagnitude` (note 11). Wave state survives loading — sea doesn't "reset".

## Wave Height Query (`OceanHeight`, static)

Canonical way to get water height at a point (via Crest `SampleHeightHelper`, absent from repo — note 24):

```csharp
// default accuracy 0.2
static float GetHeight(SampleHeightHelper helper, Vector3 worldPos);
static float GetHeight(SampleHeightHelper helper, Vector3 worldPos, float accuracy);

static bool IsUnderwater(SampleHeightHelper helper, Vector3 pos)
    => GetHeight(helper, pos) > pos.y;
```

`SampleHeightHelper.Init(worldPos, accuracy, false, null)` → `Sample(ref result)`. Used everywhere to check "is object underwater" and wave height.

## Relation to Other Systems

| System | How It Uses Waves |
|---------|---------------------|
| `Buoyancy` (note 14) | Samples height via `Ocean.GetWaterHeightAtLocation2` + `GetChoppyAtLocation` for buoyancy force. |
| `Wind` (note 17) | Source of direction/force; waves catch up to wind via `WavesInertia`. |
| `Ocean` (Crest) | `Ocean.Singleton`, `GetWaterHeightAtLocation2`, `GetChoppyAtLocation`, rendering (`RefsDirectory.oceanRenderer`). |
| `BoatDamage.Overflow` | Swashing waves add water to bilge. |
| `Fourier` / `Fourier2` | FFT wave spectrum generation (Gerstner, Crest). |

## Practical Takeaways for Modding

1. **Waves are inertial:** direction rotates toward wind at speed `~ directionChangeRate * (10/currentInertia)`, height catches up to wind force. Sudden wind change doesn't instantly change waves.
2. **Wave height ≈ `Wind.currentWind.magnitude`** (with delay, min 1). Want calm/storm — change wind, waves will follow.
3. **Water height at point:** `OceanHeight.GetHeight(helper, worldPos)` / `IsUnderwater(...)` (requires `SampleHeightHelper` from Crest).
4. Wave state (`wavesRotation/Inertia/Magnitude`) is saved to save.
5. For custom buoyancy use `Ocean.Singleton.GetWaterHeightAtLocation2(x, z)` (accounting for `outCurrentOffset` of floating origin).
