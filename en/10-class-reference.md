# 10. Assembly-CSharp.dll class reference

Quick reference of classes significant for modding. From decompilation of `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy 8.2.0.

## Text and UI

### `LookUI` : MonoBehaviour

Class responsible for displaying hints when hovering over interactive objects.

| Field/method | Type | Description |
|-------------|------|-------------|
| `controlsText` | `TextMesh [SerializeField]` | Control hints ("pick up", "use", "buy") |
| `hintText` | `TextMesh [SerializeField]` | Description (`button.description`) |
| `extraText` | `TextMesh [SerializeField]` | Object name (`button.lookText`) |
| `ShowLookText(GoPointerButton)` | `void` | Main method — sets text in all TextMesh fields |
| `ShowHoldText(PickupableItem)` | `void` | Hints for held item |
| `ClearText()` | `void` | Clears all TextMesh fields |

### `StartMenuButton` : GoPointerButton

Main menu button.

| Field/method | Type | Description |
|-------------|------|-------------|
| `type` | `StartMenuButtonType [SerializeField]` | Button type (NewGame, Continue, Options, Quit, Slot, LoadBackupSave) |
| `SetButtonText(string)` | `void` | Sets button text via `GetComponent<TextMesh>().text` on child objects |

### `GPButtonKeybinding` : GoPointerButton

Keybinding button in control settings.

| Field/method | Type | Description |
|-------------|------|-------------|
| `inputName` | `string` | Action name (serialized in prefab) |
| `input2` | `bool` | Alternative key |
| `controllerButton` | `bool` | Controller button (OVR) |
| `text` | `TextMesh private` | Child TextMesh for key display |
| `UpdateText()` | `void` | Writes `keyCode.ToString()` into TextMesh |
| `OnActivate()` | `override void` | Enters remap mode: `text.text = "(press a key)"` |
| `Update()` | `void` | Catches next key press when remapping is active |

## Input

### `GoPointer` : MonoBehaviour

Main game raycast pointer.

| Field/method | Type | Description |
|-------------|------|-------------|
| `pointedAtButton` | `GoPointerButton private` | Current object under crosshair |
| `hit` | `RaycastHit private` | Latest raycast result |
| `DoRaycast()` | `void private` | Physics.Raycast from screen center |
| `MainButtonDown()` | `bool public` | LMB pressed (checks `!GameState.inCursorMenu`) |
| `AltButtonDown()` | `bool public` | RMB pressed |
| `AltButtonHeld()` | `bool private` | RMB held |
| `LateUpdate()` | `void private` | Calls `DoRaycast()` |

### `GoPointerButton` : MonoBehaviour

Base class for all interactive objects.

| Field | Type | Description |
|-------|------|-------------|
| `lookText` | `string public` | Hover text |
| `description` | `string public` | Description |
| `isLookedAt` | `bool protected` | Is cursor over this |
| `OnActivate()` | `virtual void` | Action on click |
| `Look(GoPointer)` | `void public` | Register hover |

Derived classes: `GPButtonRopeWinch`, `GPButtonSteeringWheel`, `GPButtonSailPusher`, `GPButtonBed`, `GPButtonTrapdoor`, `GPButtonDockMooring`, `GPButtonPurchaseBoat`, `GPButtonInventorySlot`, `GPButtonExtraMenus`, `GPButtonPortMissions`, `GPButtonKeybinding`, `StartMenuButton`, `CurrencySwitchButton`, `BoatDamageWaterButton`, `GPButtonBoatPushCol`, `GPButtonControlToggle`, `GPButtonSettingsCheckbo`, `GPButtonResetKeybindings`.

### `MouseLook` : MonoBehaviour

Mouse camera rotation control.

| Method | Type | Description |
|--------|------|-------------|
| `ToggleMouseLook(bool)` | `static void` | Enable/disable camera rotation |
| `ToggleMouseLookAndCursor(bool)` | `static void` | Full control: camera + cursor + `GameState.inCursorMenu` |
| `Update()` | `void private` | Camera rotation from mouse movement |

### `GameInput`

Static class for key management.

| Method | Type | Description |
|--------|------|-------------|
| `GetKeyCode(InputName, bool, bool)` | `KeyCode static` | Get key by action name |
| `SetKeyMap(InputName, KeyCode, bool, bool)` | `void static` | Set key binding |
| `ResetToDefaults()` | `void static` | Reset to default keys |
| `GetControllerKeyString(InputName)` | `string static` | String name of controller key |

## Game state

### `GameState`

Static fields for global state.

| Field | Type | Description |
|-------|------|-------------|
| `inCursorMenu` | `bool public static` | Is player in cursor menu |
| `currentBoat` | `Boat public static` | Player's current boat |
| `inBed` | `ShipItemBed public static` | Is player in bed |
| `sleeping` | `bool public static` | Is player sleeping |
| `lookingWithController` | `bool public static` | Is controller being used |
| `justStarted` | `bool public static` | Recent start flag |
| `currentlyLoading` | `bool public static` | Loading in progress |

### `Refs`

Static references to key objects.

| Method | Type | Description |
|--------|------|-------------|
| `SetPlayerControl(bool)` | `void static` | Enable/disable player control (`ovrController.enabled`, `charController.enabled`). **May throw NullReferenceException when called before initialization.** |
| `mouseCrosshair` | `GameObject static` | Mouse crosshair |

## Enumerations

### `InputName`

Enumeration of all control actions. Values are used in `GameInput.GetKeyCode`, `GPButtonKeybinding.inputNameEnum`. Obtained via `Enum.Parse(typeof(InputName), inputName)`.

### `StartMenuButtonType`

`NewGame`, `Continue`, `Options`, `Quit`, `Slot`, `LoadBackupSave`.
