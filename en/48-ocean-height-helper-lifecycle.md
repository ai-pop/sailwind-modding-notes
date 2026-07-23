# 48. Ocean and water: OceanHeight, SampleHeightHelper, Ocean lifecycle, underwater effect

Exhaustive breakdown of water height API, Ocean lifecycle, underwater effect. From decompilation of `Assembly-CSharp.dll` (Sailwind v0.38). Related to notes 31, 43, 46.

## B6. OceanHeight — verbatim bodies

```csharp
public class OceanHeight : MonoBehaviour
{
    public static float GetHeight(SampleHeightHelper helper, Vector3 worldPos)
    {
        helper.Init(worldPos, 0.2f, false, null);
        float result = 0f;
        if (helper.Sample(ref result)) { return result; }
        return 0f;   // Crest not ready → 0f (not 0.4!)
    }

    public static float GetHeight(SampleHeightHelper helper, Vector3 worldPos, float accuracy)
    {
        helper.Init(worldPos, accuracy, false, null);
        float result = 0f;
        if (helper.Sample(ref result)) { return result; }
        return 0f;
    }

    public static bool IsUnderwater(SampleHeightHelper helper, Vector3 pos)
    {
        return GetHeight(helper, pos) > pos.y;
    }
}
```

**What's inside:**
- `GetHeight` calls `helper.Init(worldPos, accuracy, false, null)`, then `helper.Sample(ref result)`
- **Init + Sample in one call** — every sample requires Init
- Default accuracy = **0.2f** (Crest parameter)
- If `helper.Sample()` returns `false` (Crest not ready) → **0f is returned**, NOT garbage value
- `IsUnderwater`: simply `waterHeight > pos.y` — object is below water

**Conclusion for modder:**
- `OceanHeight.GetHeight` — safe wrapper; fallback = 0f when Crest is unready
- But 0f is **not the correct height**; when `canCheckBuoyancyNow[0] != 1` Crest may return false → result 0f → item "underwater" (pos.y > 0 → IsUnderwater = false, pos.y < 0 → true). Check `canCheckBuoyancyNow[0]==1` **before calling**, as `Boyant` does.

## B7. SampleHeightHelper — complete map

**SampleHeightHelper is a class from Crest (DLL), NOT decompiled in Assembly-CSharp.** Full class body is absent in the export. Only signatures from usage are available:

| Method | Signature (from usage) | What it does |
|--------|------------------------|--------------|
| `Init` | `Init(Vector3 worldPos, float accuracy, bool disabled, Object component)` | Initialize sample: position, accuracy, disable displacement?, optional component |
| `Sample` | `bool Sample(ref float result)` | Execute sample, return height in `result`; false = Crest not ready |

**Constructor:** `new SampleHeightHelper()` — created as a normal struct/class. In game code it's used **as a field** on objects:

```csharp
// AnimSplash
private SampleHeightHelper helper = new SampleHeightHelper();

// BoatSplashCrest  
private SampleHeightHelper helper = new SampleHeightHelper();

// BoatDamage
private SampleHeightHelper helper = new SampleHeightHelper();

// Rain
private SampleHeightHelper helper = new SampleHeightHelper();

// WaveSound
private SampleHeightHelper helper = new SampleHeightHelper();

// OceanHeight.GetHeight — helper comes as parameter (caller creates)
```

**Can you create one instance per session and call from a foreign FixedUpdate?**

- **Yes**, as every game class does: one `helper` per component, reused multiple times
- `Init()` is called **before every sample** — mandatory (repositions position/accuracy)
- Helper is **not thread-safe** (Crest is not thread-safe; all samples in main thread)
- Doesn't require "readiness" on creation — it's just a structural helper; real check is in `helper.Sample()` (returns false if not ready)

**Conclusion for modder:** create `private SampleHeightHelper helper = new SampleHeightHelper();` on your mod's MonoBehaviour, call `helper.Init(pos, 0.2f, false, null)` + `helper.Sample(ref result)` each frame. Check `canCheckBuoyancyNow[0]==1` before sampling (or check `result == 0f` as fallback).

## B8. Ocean object lifecycle

### Who/when creates the GO with Ocean

`Ocean` is a **MonoBehaviour on a GameObject in the scene** (not Instantiate at runtime). The "Ocean" GO is part of the scene hierarchy.

**Awake:**
```csharp
Singleton = this;     // static singleton
mat[0] = material; mat[1] = material1; mat[2] = material2;
```

**Start:**
```csharp
canCheckBuoyancyNow = new byte[1];    // created at Start, NOT at Awake!
Initialize();                          // full init: FFT, meshes, etc.
```

**Signs "Ocean ready":**
1. `canCheckBuoyancyNow[0] == 1` — primary gate. Set in `calcPhase3()` (after FFT + mesh update complete). Reset to `0` in `updNoThreads()` before `calcComplex`.
2. `Ocean.Singleton != null` — object exists
3. `canCheckBuoyancyNow != null` — array created (only after Start)

**Readiness flow per frame:**
```
Start → canCheckBuoyancyNow = new byte[1] → Initialize()
  → calcComplex → FFT → calcPhase3 → canCheckBuoyancyNow[0] = 1  // READY after first Update
```

In `updNoThreads()` each frame:
```
canCheckBuoyancyNow[0] = 0    // before calcComplex (fr2)
calcComplex + FFT              // fr2 + fr2B
FFT(t_x)                       // fr3  
calcPhase3                      // fr4 → canCheckBuoyancyNow[0] = 1  // READY again
```

**If `everyXframe = 5`:** gate = 0 on frames fr2 (0), fr2B (1), fr3 (2), fr4 (3) — i.e. 0 on 1-3 out of 5 frames, 1 on frames 0 and 4. On fr4 frame (`calcPhase3`) gate = 1.

**Conclusion:** Ocean is ready **after the first Update**. Before that — `canCheckBuoyancyNow` may be null (not yet Start). In runtime: gate switches 0/1 per frame; 1 ≈ 2 out of 5 frames with default everyXframe.

## B9. Water height for camera / underwater effect

### Class with `cameraWaterHeight`

**Real name:** `PlayerSwimming` — has field `public static float cameraWaterHeight`.

But in `PlayerSwimming.LateUpdate` and `SwimEffects.Update` code uses **`UnderwaterEffect.cameraWaterHeight`** — this is a **different** static field! The `UnderwaterEffect` class **is NOT decompiled** (absent in Assembly-CSharp export). Likely from a separate DLL (Oculus plugin / AmplifyOcclusion / other).

**What happens with `PlayerSwimming.cameraWaterHeight`:**
- Declared as `public static float cameraWaterHeight` (line 37)
- **NOT read** in LateUpdate — instead `UnderwaterEffect.cameraWaterHeight` is read (line 102)
- `PlayerSwimming.cameraWaterHeight` is likely a **dead field** (not used in LateUpdate; decompiler may have created it due to name conflicts)

**`UnderwaterEffect.cameraWaterHeight`** — real source of water height for the camera. The `UnderwaterEffect` class is in an **external DLL**, not in Assembly-CSharp. Cannot determine exactly how it computes height from decompilation, but from context:
- `SwimEffects.Update`: `isUnderwater = Camera.main.transform.position.y < UnderwaterEffect.cameraWaterHeight && flag`
- `PlayerSwimming.LateUpdate`: `waterHeight = Mathf.Lerp(waterHeight, UnderwaterEffect.cameraWaterHeight, deltaTime * lerp)`
- Most likely, `UnderwaterEffect` is a component on the camera that samples water height at camera position and writes to its static `cameraWaterHeight`

**All members of PlayerSwimming (verbatim):**

| Field | Modifier | Type | Value/Purpose |
|-------|----------|------|---------------|
| `ovrController` | private | `OVRPlayerController` | VR controller |
| `charController` | private | `CharacterController` | Movement |
| `platform` | [SerializeField] private | Transform | — |
| `width` | [SerializeField] private | float | — |
| `lerp` | [SerializeField] private | float | Lerp speed to water height |
| `diveSpeedMult` | public | float | Dive multiplier |
| `swimHeight` | public | float | Depth threshold for swimming |
| `shallowWaterHeight` | public | float | 0.6f — shallow water threshold |
| `surfaceSwimHeight` | public | float | Surface swimming threshold |
| `jumpKeyUndiveSpeed` | public | float | Un-dive speed |
| `swimming` | **public static** | bool | In water |
| `inShallowWater` | **public static** | bool | In shallow water |
| `swimmingOnSurface` | **public static** | bool | On surface |
| `observerSwimming` | **public static** | bool | Observer in water |
| `cameraWaterHeight` | **public static** | float | **Dead field?** (not used in LateUpdate) |
| `waterHeight` | private | float | Lerped water height tracking |
| `lastWaterHeight` | private | float | Previous height (for delta) |
| `dive` | **public static** | float | Dive (camera angle * diveSpeedMult) |
| `defaultAccel` | private | float | Default OVR acceleration |
| `unswimTimer` | private | float | — |
| `debugLog` | [SerializeField] private | bool | — |

**SwimEffects — all members:**

| Field | Type | Purpose |
|-------|------|---------|
| `swimAudio` | AudioSource | Swimming sound |
| `underwaterAudio` | AudioSource | Underwater sound |
| `surfaceParticles` | ParticleSystem | Surface splashes |
| `playerWake` | GameObject | Wake effect |
| `stars` | Renderer | Stars (disabled underwater) |
| `bubbleParticles` | Transform | Bubbles |
| `underwaterCameraEffects` | GameObject[] | Underwater camera effects |
| `controller` | CharacterController | — |
| `swimAudioVolume` | float | 0.2f |
| `isUnderwater` | **public static bool** | Camera is underwater |

**Conclusion for modder:**
- `UnderwaterEffect.cameraWaterHeight` — **the only real source** of water height for the camera. Class inaccessible from decompilation. Runtime: can read `UnderwaterEffect.cameraWaterHeight` (static) — but type is unknown. Practical approach: use `PlayerSwimming.cameraWaterHeight` (static) — this is a **dead field**, but runtime could be populated by a mod, OR use `Ocean.Singleton.GetWaterHeightAtLocation2(x-chop, z)` + `canCheckBuoyancyNow[0]==1` gate.

## B10. OceanUpdaterCrest, OceanWavesUpdater, BoatSplashCrest

### OceanUpdaterCrest

No ready wave height/normal available directly. It manages **Gerstner wave weights** (wind and inertial). Available fields:

| Field | Type | Purpose |
|-------|------|---------|
| `timeBetweenUpdates` | float | Update interval |
| `inertiaWindScale` | float | Inertia → waves scale |
| `windWindScale` | float | Wind → waves scale |
| `lerpRateInertial` | float | Lerp-rate for inertial waves |
| `lerpRateWind` | float | Lerp-rate for wind waves |
| `smallWavesMult` | [SerializeField] float | Small waves multiplier |
| `windSpeedMult` | [SerializeField] float | Wind speed multiplier |
| `windWaves` | [SerializeField] ShapeGerstnerBatched | Wind waves |
| `inertiaWaves` | [SerializeField] ShapeGerstnerBatched[] | Inertial waves (DCT, 2 sets) |
| `wavesRotationAngle` | **static float** | Wave rotation angle |

**For modder:** `OceanUpdaterCrest` doesn't provide height/normal — it's a weight updater, not a sampler. Get height from `Ocean` or `OceanHeight`.

### OceanWavesUpdater

Also an updater (legacy, for non-Crest Ocean). Controls `ocean.scale`, `ocean.choppy_scale`, `ocean.windx`, `ocean.windy`.

| Field | Type | Purpose |
|-------|------|---------|
| `wavesRotationAngle` | **static float** | Wave rotation angle (0→lerp→rotationFixer) |
| `ocean` | Ocean (private) | Reference to Ocean |
| `oceanScale` | float | Scale for ocean.scale |
| `choppyFactor` | float | Choppy factor |

**For modder:** `OceanWavesUpdater.wavesRotationAngle` — wave rotation angle (static). For height — use Ocean.

### BoatSplashCrest — how the game spawns boat splashes

```csharp
public class BoatSplashCrest : MonoBehaviour
{
    [SerializeField] Rigidbody boatRigidbody;
    [SerializeField] float deltaThreshold;
    [SerializeField] float deltaMult;
    [SerializeField] float minSize, maxSize;
    [SerializeField] float minSpeed, maxSpeed;
    [SerializeField] float volumeFadeIn, volumeFadeOut;
    
    AudioSource audio;
    ParticleSystem particles;
    float verticalDelta;
    float lastDifference;
    SampleHeightHelper helper = new SampleHeightHelper();
}
```

**Splash mechanics:**
- `FixedUpdate → UpdateDeltas + UpdateIntensity`
- **UpdateDeltas:** `verticalDelta = lastDifference - transform.position.y` — height difference current vs last frame
- If `verticalDelta < deltaThreshold` → delta = 0 (no splash)
- **UpdateIntensity:** `num = verticalDelta * deltaMult * boatRigidbody.velocity.magnitude`
  - `startSize = Lerp(minSize, maxSize, num)` — particle size
  - `startSpeed = Lerp(minSpeed, maxSpeed, num)` — particle speed
  - Audio volume = Lerp to `num` with fade-in/fade-out
  - If volume ≤ 0.01 → audio OFF; otherwise → Play

**For modder:** reference for consistent splash effects. Use `verticalDelta` (Y-position difference) + `velocity.magnitude` → particle scale + volume. `SampleHeightHelper` — not used in BoatSplashCrest for height (computes delta via lastDifference, not through Ocean).

## B11. Boyant — class body (height snap confirmed)

```csharp
public class Boyant : MonoBehaviour
{
    private Ocean ocean;
    public float buoyancy;
    private bool hasChoppy;
    private Vector3 oldPos;

    void Start() {
        ocean = Ocean.Singleton;
        hasChoppy = ocean.choppy_scale > 0f;
        oldPos = transform.position;
    }

    void Update() {
        if (ocean.canCheckBuoyancyNow[0] == 1) {
            float num = 0f;
            if (hasChoppy) {
                num = ocean.GetChoppyAtLocation2(transform.position.x, transform.position.z);
            }
            float num2 = ocean.GetWaterHeightAtLocation2(
                transform.position.x - num, transform.position.z) + buoyancy;
            transform.position = new Vector3(transform.position.x, num2, transform.position.z);
            oldPos = transform.position;
        } else {
            transform.position = oldPos;  // hold old position when Ocean not ready
        }
    }
}
```

**Confirmed:** Boyant is a **simple height snap** (note 43 is correct). Additional detail:
- **hasChoppy** checked at Start (choppy_scale > 0) — not every frame
- **oldPos** — fallback when Ocean not ready (`canCheckBuoyancyNow[0] != 1`)
- X/Z don't change — only Y snap
- **buoyancy** — public field, upward offset from water (default on prefabs)

## B12. Splash sound (item entering water)

**Class:** `ItemCollisionSoundPlayer` — singleton on an object in the scene.

```csharp
public class ItemCollisionSoundPlayer : MonoBehaviour
{
    public float speedMult;
    public float minPitchMass = 50f;
    public float maxPitchMass = 1f;
    public float basePitch = 0.5f;
    public float pitchMult = 1f;
    public float volumeReductionFromPitch = 0.5f;
    public AudioSource[] audios;
    public static ItemCollisionSoundPlayer instance;
    private float globalCooldown;
}
```

**Method:** `PlayWoodColSound(Vector3 position, float mass, float speed)`
- Plays item collision sound (wood collision)
- Called from `ItemRigidbody.OnCollisionEnter` (non-trigger collision, relativeVelocity.sqrMagnitude > threshold, mass > 1)
- Pitch = `basePitch + InverseLerp(minPitchMass, maxPitchMass, mass) * pitchMult`
- Volume = `speed * speedMult - pitch * volumeReductionFromPitch`
- Audio source positioned at collision point
- Cooldown = 0.12s (global)

**For item entering water:** `ItemCollisionSoundPlayer` has no splash sound for items. Collision sound is wood collision (item → item/boat). Splash sounds for items in water **are not implemented** in vanilla.

**Boat splash sounds:**
- `AnimSplash` (on boat) — AudioSource + ParticleSystem, controlled by height delta + speed
- `BoatSplashCrest` (on boat) — AudioSource + ParticleSystem, controlled by verticalDelta + boat speed

**UISoundPlayer (for UI sounds):**
```csharp
UISoundPlayer.instance.PlayUISound(UISounds.itemPickup, 0.45f, pitch);
UISoundPlayer.instance.PlaySmallItemDropSound();
```
- `UISounds.itemPickup`, `UISounds.itemInventoryIn`, `UISounds.winchClick` — enum
- Pitch/volume configurable

**Conclusion for modder:** for splash sound of item entering water — you need to create your own. Vanilla doesn't have one. Reference: `AnimSplash` / `BoatSplashCrest` for boats (deltaY + velocity → scale/volume).
