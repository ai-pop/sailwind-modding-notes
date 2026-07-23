# 21. Debugger, Hidden Debug Mode, and Global Multipliers

Analysis of the built-in debugger and hidden cheat mode. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Useful for testing mods, understanding balance points and "internal" mechanisms.

## Hidden Debug Mode (Works in Release!)

The `Debugger` class has a **secret activation code** that works even in the built game (not just in the editor):

```
Hold P + N and press T
→ buildDebugModeOn = true
→ PlayerNeeds.instance.godMode = true   (needs god mode)
→ speedometers enabled
```

Check: `Input.GetKey("p") && Input.GetKey("n") && Input.GetKeyDown("t")`.

### What `buildDebugModeOn` Enables
| Key | Action |
|---------|----------|
| `0` / `1` / `2` / `5` / `6` / `8` / `9` | `BoatProbes.ChangeEnginePower(...)`: boat engine power = `0 / 0.5 / 2 / 5 / -5 / 8 / 50` (essentially a "motor" speed cheat). |
| `Keypad5` (KeyCode 261) | Toggle `debugWind` (forces wind `(10,0,10)`). |
| `Keypad1` (257) | `Time.timeScale = debugTimescale` (default 2). |
| `Keypad3` (259) | `Time.timeScale = 1` (normal). |

> `buildDebugModeOn` also **removes time pause** in trade UI (`SunPaused`/`PlayerNeeds.LateUpdate` check `!Debugger.buildDebugModeOn`).

## Editor-Only Keys (`Application.isEditor`)

| Key (KeyCode) | Action |
|-------------------|----------|
| `Keypad2` (258) | `PlayerNeedsUI.DebugHideInventory()` |
| `Keypad7` (263) | `Sun.initialTimescale = 0.008` ("×1", very slow time) |
| `Keypad9` (265) | `Sun.initialTimescale = 0.8` ("×100") |
| `Keypad4` (260) | `Time.fixedDeltaTime = 0.02222` |
| `Keypad6` (262) | `Time.fixedDeltaTime = 0.002222` |
| `Keypad8` (264) | Log vitamins/protein |
| `KeypadPlus` (270) / `KeypadMinus` (269) | `debugYardMult ± 0.01` (minimum sail unfurl) |
| `F8` (289) | `targetFrameRate = 3` |
| `F9` (290) | `targetFrameRate = -1` (no limit) |
| `F11` (292) | Screenshot (developer path `D:/Pictures/Sailwind Promo/`) |
| `F5` (286) | **+250 reputation** (alankh region) |
| `F6` (287) | **+1000 to all 4 currencies** |
| `F4` (285) | `PlayerNeeds.water -= 10` |
| `-` | `Sun.globalTime += 1` (fast-forward +1 hour) |
| `j` | Toggle `debugForceKinematicBoat` |
| `b` | `recoveryStep = true` (step-by-step recovery debugging) |
| `v` | Spawn items (prefabs 110 and 79), `sold + RegisterToSave` |
| `m` + `v` | `clearSaveData` → `PlayerPrefs.DeleteAll()` |
| `i` | Log all `SaveablePrefab.existingInstanceIds` |

## Global Tuning Multipliers (`Debugger.instance`)

These fields **affect gameplay in release** (combat systems read them):

| Field | Default | Where Used |
|------|:--:|------------------|
| `debugSailAreaMult` | 1 | `Sail.GetRealSailPower()` — area/thrust multiplier for **all** sails (note 17). |
| `debugYardMult` | 0.015 | Minimum sail efficiency when canvas is furled (`max(debugYardMult, currentUnroll)`). |
| `debugItemColAudioThreshold` | 0.1 | Item collision sound threshold. |
| `debugTimescale` | 2 | Target for `Time.timeScale` via Keypad1. |

Static flags: `buildDebugModeOn`, `debugWind`, `debugForceKinematicBoat`, `recoveryStep`, `debugRecoveryFlag`, `sailForwardShare`, `kinematicItemsTimer`.

### Other "Internals"
- In `Awake()`, `PlayerGold.gold = 100` is always set (both editor and release) — starting "legacy" gold reserve.
- `Settings.hintTextEnabled` and `itemNameTextEnabled` forced to `true` on startup.
- Item disable flags: `disableItemCols`, `disableItemRigidbodyCols`, `disableItemRenderers`, `disableItemUpdate`, `disableItemRigidbodyUpdate`, `destroyAllItems` — useful for item physics/performance diagnostics.

## Other Debug Classes

| Class | Purpose |
|-------|-----------|
| `DebugMarketTracker` | Economy and mission balance hub (note 13). |
| `DebugWeather` | Force weather. |
| `DebugSunTime` / `DebugFogController` | Time/fog control. |
| `DebugUIBuilder` / `DebugUISample` | Debug UI building. |
| `DebugPainter` / `DebugSimpleDecollision` | Drawing/decollision debugging. |
| `DebugMarketTracker2/3` | Additional market trackers. |
| `BoatDebugger` / `EditorDebugger` | Boat/editor debugging. |

## Practical Takeaways for Modding

1. **Hidden cheat P+N+T** enables godMode and debug mode in release — can be used for testing, but note: this is "real" game code, not a mod Easter egg.
2. **`Debugger.instance.debugSailAreaMult`** — legitimate global sail thrust multiplier; convenient entry point for sail balance mods.
3. **`debugYardMult`** — minimum thrust of a furled sail.
4. `buildDebugModeOn` removes time pause in trade UI — account for this if your mod relies on the pause.
5. Keys F5/F6/engine digits and Keypad-time — for understanding only; in release, only a subgroup is active (see `buildDebugModeOn`).
6. Initial `PlayerGold.gold = 100` is set in `Debugger.Awake`.
