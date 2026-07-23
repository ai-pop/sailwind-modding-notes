# 24. Decompilation Coverage: Missing but Used Classes

**Important warning for modding.** The `sailwind-decompiled` repository contains 677 game classes, but the decompilation is **incomplete**: a number of classes referenced by present code are absent from the repository (likely missed in the ILSpy export or located in other assemblies). Below is a list of critical gaps and **API reconstruction from usage sites**, so the reference remains useful.

## What's Missing (by number of references in game code)

| Class | References | Role |
|-------|:--:|------|
| `GameInput` | ~25 | Centralized input (keys, axes, scroll). |
| `InputName` | ~24 | Input action enumeration. |
| `BoatProbes` | ~9 | **Core boat physics** (buoyancy, drag, "engine"). |
| `SampleHeightHelper` | ~8 | Crest wave height sampling. |
| `SimpleFloatingObject` | ~4 | Item/bobber buoyancy. |
| `FloatingObjectBase` | ~3 | Base class for floating objects. |
| `ShapeGerstnerBatched` | ~2 | Crest Gerstner waves. |
| `Crest.*` | — | Crest ocean library namespace (custom fork). |
| `UnderwaterEffect` | ~2 | Water height at camera (`cameraWaterHeight`) for swimming. |
| `BoatAlignNormal` | ~1 | Simplified boat physics far from player (wave normal alignment). |

> The `Buoyancy` class present in the repository is **general blob-buoyancy** (note 14), but real per-component physics of a specific boat lives in **`BoatProbes`**, which is not in the export.

## Reconstruction of `GameInput` (static input class)

Reconstructed from calls in `GoPointer`, `LookUI`, `BoatCamera`, `GPButton*`, `GoPointerMovement`, `MapTableCamera`, etc.:

```csharp
static class GameInput {
    static bool controllerEnabled;                              // GPButtonSettingsCheckbo

    static bool   GetKey(InputName action);                     // held
    static bool   GetKeyDown(InputName action);                 // pressed this frame
    static bool   GetKeyUp(InputName action);                   // released

    static KeyCode GetKeyCode(InputName action, bool input2, bool controllerButton);
    static void    SetKeyMap(InputName action, KeyCode key, bool input2, bool controllerButton);
    static string  GetControllerKeyString(InputName action);    // controller button name
    static void    ResetToDefaults();                            // GPButtonResetKeybindings

    static float  GetScrollAxis();                               // mouse wheel (GoPointer, BoatCamera zoom)
    static float  GetPrimaryHorizontal();                        // analog axes
    static float  GetPrimaryVertical();
    static float  GetSecondaryHorizontal();
    static float  GetSecondaryVertical();
}
```

`input2` — alternative (second) key for the action; `controllerButton` — controller button (OVR). Remapping works via `SetKeyMap` (see `GPButtonKeybinding`, note 10).

## Reconstruction of `InputName` (action enum)

Numeric values appear as `(InputName)N`. By usage context (confirmed across all code):

| Value | Action | Where Used |
|:--:|--------------------|------------------|
| 0 | **Forward** (movement; while swimming — look-based dive) | `GoPointerMovement` (forward/turn), `PlayerSwimming` (dive) |
| 1 | **Backward** | `GoPointerMovement` |
| 2 | **Left/Right** (strafe) | `GoPointerMovement` |
| 3 | **Right/Left** (strafe) | `GoPointerMovement` |
| 4 | **Jump** (also surface while swimming) | `PlayerClimb`, `PlayerSwimming`, `PlayerCrouching` (stand up) |
| 5 | **Rope/winch** (pull) | `GPButtonRopeWinch` |
| 7 | **Crouch / dive down** | `PlayerCrouching`, `PlayerSwimming` |
| 8 | **Primary action** ("use", equivalent to LMB) | `GoPointer` (click/pickup), `LookUI` (left icon), `MapTableCamera`, `GPButtonSettingsCheckbo` |
| 9 | **Alt action** ("alt use", equivalent to RMB) | `GoPointer`, `CrateInventoryUI`, `CrateSealUI`, `MapChart`, `LookUI` (right icon), `Sleep` (exit bed) |
| 10 | **Throw / drop** (drop) | `GoPointer` (drop logic, `Settings.autoThrow`) |
| 11 | **Modifier** (when held, disables mouse-look) | `MouseLook`, `GoPointer` (346) |
| 15 | **Open inventory/needs** (UI) | `PlayerNeedsUI` (or OVR RawButton 256) |
| 16 | **Boat camera** (toggle/zoom) | `BoatCamera` |
| 17 | **Hints** (toggle hints) | `Hints` |

> Values 0–4, 7–10, 15–17 are reliable (confirmed by context); 5 and 11 — by single usage. Exact enum constant names are unknown (definition is absent). `GPButtonKeybinding` parses name via `Enum.Parse(typeof(InputName), inputName)` — names set as string in button prefab. Movement also goes through analog axes `GameInput.GetPrimaryVertical/Horizontal`.

## Reconstruction of `BoatProbes` (core boat physics)

From usage in `BoatDamage`, `BoatMass`, `Debugger`, `HullDrag`, `BoatPerformanceSwitcher`, `Recovery`:

```csharp
class BoatProbes : MonoBehaviour {
    // Buoyancy
    public float _forceMultiplier;        // MAIN buoyancy multiplier (0 = sinking)
    public /* points */ _forcePoints;       // buoyancy force application points
    public /* ... */ appliedBuoyancyForces;// applied forces (visualization/debug)

    // Water drag
    public float _dragInWaterForward;      // water drag (forward)
    public float _dragInWaterRight;        // water drag (sideways)
    public float addedHullDrag;            // added hull drag
    public float addedSideDrag;            // added side drag

    // "Engine"
    public void ChangeEnginePower(float power);   // read by Debugger: 0/0.5/2/5/-5/8/50
}
```

- `BoatDamage` manages `_forceMultiplier` for sinking: `Lerp(base, base*0.66, …)` and stepping down to 0 at `sunk` (note 14).
- `Debugger` in debug mode calls `ChangeEnginePower` to accelerate boat (note 21).
- `baseBuoyancy` in `BoatDamage` = `boat._forceMultiplier` at start.
- `BoatPerformanceSwitcher` expects `BoatProbes` **or** `BoatAlignNormal` (physics mode switcher by distance).

## Reconstruction of Floating Objects and Crest

```csharp
class FloatingObjectBase {
    public bool InWater;                  // object in water (read by FishingRodFish)
}
class SimpleFloatingObject : FloatingObjectBase {
    public float _raiseObject;            // "float-up" height above water (= ShipItem.floaterHeight)
    public float _dragInWaterRotational;  // rotational drag in water
}

struct/class SampleHeightHelper {         // Crest: wave height sampling
    void Init(Vector3 pos, float accuracy, bool _, Object __);
    bool Sample(ref float result);        // water height at point
}
```

### `UnderwaterEffect` (static)
```csharp
static class UnderwaterEffect {
    public static float cameraWaterHeight;  // water height at camera (read by PlayerSwimming)
}
```
Source of "water height" for swimming: `PlayerSwimming` lerps `waterHeight` toward `UnderwaterEffect.cameraWaterHeight`.

### `BoatAlignNormal`
Alternative (simplified) boat physics mode — alignment by wave normal instead of full blob-buoyancy `BoatProbes`. `BoatPerformanceSwitcher` (note 14) expects **either** `BoatProbes`, **or** `BoatAlignNormal`: for boats far from player, lightweight `BoatAlignNormal` is used; near player — full `BoatProbes`.

`Ocean` (present) provides high-level `GetWaterHeightAtLocation2(x, z)` and `GetChoppyAtLocation(x, z)`; low-level Gerstner math (`ShapeGerstnerBatched`, `SampleHeightHelper`) — in the missing Crest module.

## Practical Takeaways for Modding

1. **Reference is incomplete:** `GameInput`, `InputName`, `BoatProbes`, Crest helpers are absent from `sailwind-decompiled`. Don't be surprised if you can't find them.
2. **Input** goes through `GameInput` (not directly `UnityEngine.Input` in game logic); remapping — `SetKeyMap`/`GetKeyCode` with `input2`/`controllerButton` flags.
3. **Actions 8/9** = primary/alternative ("use"/"alt use"), **10** = throw, **5** = winch, **16** = camera, **17** = hints, **0–3** = movement.
4. **Boat buoyancy** is actually controlled by `BoatProbes._forceMultiplier` (not just general `Buoyancy`); boat "motor" — `ChangeEnginePower`.
5. To patch these classes via Harmony, you **need to find them in `Assembly-CSharp.dll` first** (ILSpy/dnSpy) — they exist in the assembly, just weren't exported to this repository. Type names and signatures above are accurate (reconstructed from IL calls).
6. If you need the full picture — re-decompile `Assembly-CSharp.dll` and supplement the repository with missing files (`GameInput`, `InputName`, `BoatProbes`, `SimpleFloatingObject`, `FloatingObjectBase`, Crest).
