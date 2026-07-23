# 07. SRE=False — HarmonyX nevertheless works

## Summary

BepInEx 5.4.23.5 reports `Supports SRE: False` in the log. This means the game's Mono runtime was compiled without `System.Reflection.Emit` (SRE). Nevertheless, HarmonyX (bundled with BepInEx) functions correctly.

## Expectation vs reality

| | Expectation | Reality |
|---|------------|---------|
| `PatchAll()` | Exception: SRE not available | Patches applied successfully |
| Prefix/Postfix | Don't work | Work |
| `ref` parameters | Not supported | Supported |

## Reason

HarmonyX (Harmony fork by BepInEx team) contains an alternative backend for environments without SRE. Instead of `DynamicMethod` + `ILGenerator`, it uses `Mono.Cecil` for IL manipulation at assembly level. This is slower during patch installation but functionally equivalent.

## Confirmation

Existing Harmony mods for Sailwind: NANDFixes, TowableBoats, SailInfo, ChronoCompassGPS — all work on the same runtime with `SRE: False`. A simple prefix patch with `ref string value` on `TextMesh.text` setter applies and executes without errors.

## Practical significance

Don't avoid Harmony when developing Sailwind mods due to `SRE: False`. Simply wrap `PatchAll` in `try/catch` and log the result for diagnostics.

```csharp
try {
    harmony.PatchAll(typeof(MyPatcher));
    Log.LogInfo("Harmony patches applied.");
} catch (Exception ex) {
    Log.LogError($"Harmony PatchAll failed: {ex}");
}
```

## Limitation

Transpliers (IL-level manipulation via `CodeInstruction`) may work unstable without SRE. Prefix and Postfix are reliable. Transpiler — at your own risk.
