# 03. Custom input system: GoPointer

## Summary

Input in Sailwind is built on its own `GoPointer` system (raycast against colliders), not on Unity EventSystem / Canvas raycaster. This is critical for mods that need to isolate or intercept input (e.g. for in-game UI).

## Architecture

```
GoPointer (MonoBehaviour)
├── DoRaycast()              — Physics.Raycast from screen center
│   ├── hit.collider → GetComponent<GoPointerButton>()
│   └── button.Look(this)    — pointing at object
├── MainButtonDown()         — LMB (InputName.8, checks !inCursorMenu)
├── AltButtonDown()          — RMB (InputName.9, checks !inCursorMenu)
├── AltButtonHeld()          — RMB held
└── LateUpdate()             — calls DoRaycast every frame
```

`GoPointerButton` — base class for all interactive objects (293+ `GPButtonRopeWinch` instances in a typical scene). Contains `public string lookText` and `public string description`, which `LookUI` copies into TextMesh on hover.

## Input isolation

### Method 1: built-in flag (partial)

```csharp
MouseLook.ToggleMouseLookAndCursor(false);
// Sets GameState.inCursorMenu = true
// Frees cursor (Cursor.visible = true, lockState = None)
```

**Limitation:** the `inCursorMenu` flag is checked by `MainButtonDown`, `AltButtonDown`, `MouseButtonPointer`, `GPButtonBed`. But **`DoRaycast` and `MouseLook.Update` check it inconsistently** — hover and camera rotation may continue working.

### Method 2: Harmony patches (complete)

Guaranteed isolation via prefix patches on key methods:

```csharp
// GoPointer.DoRaycast → return false when UI is open
[Prefix] static bool SkipDoRaycast() => !ModUI.IsVisible;

// GoPointer.MainButtonDown/AltButtonDown/AltButtonHeld → return false
[Prefix] static bool SkipButton(ref bool __result) {
    if (ModUI.IsVisible) { __result = false; return false; }
    return true;
}

// MouseLook.Update → return false (camera doesn't follow mouse)
[Prefix] static bool SkipMouseLook() => !ModUI.IsVisible;
```

Patches are installed via reflection (`AccessTools.Method`), since `GoPointer` and `MouseLook` types are in `Assembly-CSharp.dll`, not in plugin references.

## Cursor

Standard method `MouseLook.ToggleMouseLookAndCursor(bool newState)`:
- `true` → `Cursor.lockState = Locked`, `Cursor.visible = false`, `inCursorMenu = false` (gameplay)
- `false` → `Cursor.visible = true`, `Cursor.lockState = None`, `inCursorMenu = true` (menu)

When using this, **save the previous state** of `GameState.inCursorMenu` before releasing the cursor and restore it on UI close — otherwise the cursor hides in the game menu where it should stay visible.

## Warning

`Refs.SetPlayerControl(bool)` can throw `NullReferenceException` if called before full game initialization (observed in `EconomyUI.Awake()` and `CurrencyExchangeUI.Awake()`). Do not use in plugin `Awake`/`Start`.
