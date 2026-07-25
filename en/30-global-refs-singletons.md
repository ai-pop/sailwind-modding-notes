# 30. Global References and Singletons (Quick Reference)

Consolidated reference of access points to key game objects. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Keep handy when writing mods.

## `Refs` (static references to player and world)

```csharp
public class Refs {
    public static OVRPlayerController   ovrController;          // player controller (movement)
    public static Transform             ovrCameraRig;           // camera rig
    public static CharacterController   charController;         // character collider-controller
    public static PlayerControllerMirror observerMirror;        // observer "mirror" (position reference)
    public static Transform             controllerCameraRotMirror;
    public static Transform             shiftingWorld;          // floating origin world root
    public static MouthCol              playerMouthCol;         // player "mouth" (drinking/smoking)
    public static GameObject            mouseCrosshair;         // mouse crosshair
    public static Material              mouseCrosshairMaterial;
    public static Material              emptyMaterial;
    public static Transform[]           islands;                // all islands

    public const int goodCount = 65;     // max goods
    public const int portCount = 34;     // max ports

    public static void SetPlayerControl(bool state);  // enable/disable ovrController + charController
}
```

### Practical Application of `Refs`
- **Player/observer position:** `Refs.observerMirror.transform.position` — frequently used as "player position" (recovery, wind, fish, trade winds).
- **Control:** `Refs.SetPlayerControl(false/true)` — standard way to disable/enable control (used by sleep, recovery, UI). **Can throw NullReferenceException before initialization** (see note 10).
- **Running/movement:** `Refs.ovrController.IsRunning()`, `Refs.ovrController.outLastMovement` (see note 12).
- **World:** `Refs.shiftingWorld` = floating origin root (parent of items, `FloatingOriginManager`, note 11).

## `RefsDirectory` (singleton `RefsDirectory.instance` — asset registry)

```csharp
public class RefsDirectory : MonoBehaviour {
    public Material       emptyMat;
    public LODGroup       LODtemplateItems;          // LOD template for items (note 16)
    public OceanRenderer  oceanRenderer;             // Crest ocean renderer
    public GameObject     depthFixerPrefab;
    public GameObject     clothRopePrefab;           // cloth ropes
    public GameObject     clothRopeSheetPrefab;
    public GameObject     clothRopeJibSheetPrefab;
    public GameObject     sailSnapSoundPrefab;       // sail snap sound
    public ParticleSystem seaWaterSpillParticles;    // sea water splash particles
    public Color          damageColor;               // hull damage color
    public Color          caulkColor;                // caulking color
    public static RefsDirectory instance;
}
```
Used for instantiating shared prefabs (ropes, particles, sounds) and getting templates (item LOD).

## Game Singleton Map

Most managers are singletons via `public static X instance`, set in `Awake`/`Start`. Main ones:

| Singleton | Access | Purpose |
|----------|--------|-----------|
| `SaveLoadManager.instance` | note 11 | Saves, `modData`. |
| `CurrencyMarket.instance` | note 13 | Currency rates. |
| `DebugMarketTracker.instance` | note 13 | Economy/mission balance hub. |
| `PrefabsDirectory.instance` | notes 11/16 | Item/sail prefab registry. |
| `FloatingOriginManager.instance` | note 11 | Floating origin. |
| `PlayerNeeds.instance` | note 12 | Needs (`godMode`). |
| `PlayerTobacco.instance` | note 12 | Tobacco. |
| `Sun.sun` | note 18 | Time (`globalTime`, `OnNewDay`). |
| `Moon.instance` | note 18 | Moon phase. |
| `Weather.instance` | note 18 | Weather (`currentRegion`). |
| `WeatherStorms.instance` | note 18 | Storms (`currentStormDistance`). |
| `Sleep.instance` | note 25 | Sleep. |
| `Quests.instance` | note 27 | Quest states. |
| `OceanFishes.instance` | note 23 | Fish selection. |
| `Wind.instance` | note 17 | Wind (`ForceNewWind`). |
| `RefsDirectory.instance` | above | Asset registry. |
| `ItemSpawners.instance` | note 11 | Item spawners (cooldowns). |
| `MissionLog.instance` | note 15 | Mission log. |
| `DayLogs.instance` | notes 12/13 | Daily income log. |

### Static Classes (no instance)
`GameState`, `PlayerGold`, `PlayerReputation`, `PlayerMissions`, `Settings`, `Refs`, `GameInput` (missing, note 24), `Recovery`, `MouseLook`, `Wind` (partially), `WeatherStorms` (partially).

## Practical Takeaways for Modding

1. **Initialization:** singletons are set in `Awake`/`Start` — don't access `X.instance` too early (in another component's `Awake`); better in `Start` or later. `Refs.SetPlayerControl` can NPE before initialization.
2. **"Player position"** is usually `Refs.observerMirror.transform.position`, not `Camera.main`.
3. **Constants:** goods — **65**, ports — **34** (`Refs.goodCount`/`portCount`).
4. **Floating origin:** all world positions should be calculated relative to `FloatingOriginManager.instance.outCurrentOffset` (note 11), world root is `Refs.shiftingWorld`.
5. **Disabling control:** `Refs.SetPlayerControl(false)` + `MouseLook.ToggleMouseLookAndCursor(...)` for your UI/cutscenes.
6. **Shared assets** (ropes, particles, LOD template) — get from `RefsDirectory.instance`, don't search the scene.
