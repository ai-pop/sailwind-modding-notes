# 06. Keycode strings

## Summary

`GPButtonKeybinding.UpdateText()` writes the result of `KeyCode.ToString()` into TextMesh — Unity key names (`W`, `F1`, `Space`, `LeftShift`, `UpArrow`). These strings **must not be translated**: attempting to translate via an online service gives nonsensical results.

## Examples of incorrect translation

| Original | Google translation | Correct |
|----------|---------------------|---------|
| `F1` | `Ф1` | F1 (do not translate) |
| `W` | `Вт` | W |
| `Space` | `Космос` (Cosmos) | Space (or "Пробел" — but only manually) |
| `Tab` | `Вкладка` (Tab/Sheet) | Tab |
| `Mouse0` | `Мышь` (Mouse) | Mouse0 |

## Filter criteria

A string is excluded from translation if any condition is true:

1. **Length ≤ 2 characters** — single letters/keys (`W`, `A`, `F1`).
2. **Consists only of digits and separators** — prices, quantities (`75`, `42 gold` processed per segment; `75` skipped, `gold` translated).
3. **Matches a known KeyCode** — `Space`, `Tab`, `CapsLock`, `LeftShift`, `RightShift`, `LeftControl`, `RightControl`, `LeftAlt`, `RightAlt`, `UpArrow`, `DownArrow`, `LeftArrow`, `RightArrow`, `Return`, `Escape`, `Backspace`, `Delete`, `Insert`, `Home`, `End`, `PageUp`, `PageDown`, `KeypadEnter`, `Numlock`.
4. **KeyCode prefix** — `F1`–`F15` (`F` + digit), `Alpha0`–`Alpha9`, `Keypad0`–`Keypad9`, `Joystick*`.
5. **Contains no Latin letters** — control sequences, special characters.

## Data source

`GPButtonKeybinding` (decompilation of `Assembly-CSharp.dll`):

```csharp
public void UpdateText() {
    if (controllerButton) {
        text.text = GameInput.GetControllerKeyString(inputNameEnum);
        return;
    }
    KeyCode keyCode = GameInput.GetKeyCode(inputNameEnum, input2, controllerButton);
    text.text = keyCode.ToString(); // ← source
}
```

When a key is remapped (`SetNewKey`) the method is called again. A Harmony patch on the setter catches these strings — filtering must be applied at the translator level, not at the patch level.
