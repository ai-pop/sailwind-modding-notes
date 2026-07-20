# 06. Строки кодов клавиш

## Суть

`GPButtonKeybinding.UpdateText()` записывает в TextMesh результат `KeyCode.ToString()` — имена клавиш Unity (`W`, `F1`, `Space`, `LeftShift`, `UpArrow`). Эти строки **не подлежат переводу**: попытка перевести через онлайн-сервис даёт бессмысленный результат.

## Примеры некорректного перевода

| Оригинал | Перевод Google | Корректно |
|----------|----------------|-----------|
| `F1` | `Ф1` | F1 (не переводить) |
| `W` | `Вт` | W |
| `Space` | `Космос` | Space (или «Пробел» — но только вручную) |
| `Tab` | `Вкладка` | Tab |
| `Mouse0` | `Мышь` | Mouse0 |

## Критерии фильтрации

Строка исключается из перевода, если выполняется хотя бы одно условие:

1. **Длина ≤ 2 символа** — одиночные буквы/клавиши (`W`, `A`, `F1`).
2. **Состоит только из цифр и разделителей** — цены, количества (`75`, `42 gold` обрабатывается по сегментам; `75` пропускается, `gold` переводится).
3. **Соответствует известному KeyCode** — `Space`, `Tab`, `CapsLock`, `LeftShift`, `RightShift`, `LeftControl`, `RightControl`, `LeftAlt`, `RightAlt`, `UpArrow`, `DownArrow`, `LeftArrow`, `RightArrow`, `Return`, `Escape`, `Backspace`, `Delete`, `Insert`, `Home`, `End`, `PageUp`, `PageDown`, `KeypadEnter`, `Numlock`.
4. **Префикс KeyCode** — `F1`–`F15` (`F` + цифра), `Alpha0`–`Alpha9`, `Keypad0`–`Keypad9`, `Joystick*`.
5. **Не содержит латинских букв** — управляющие последовательности, спецсимволы.

## Источник данных

`GPButtonKeybinding` (декомпиляция `Assembly-CSharp.dll`):

```csharp
public void UpdateText() {
    if (controllerButton) {
        text.text = GameInput.GetControllerKeyString(inputNameEnum);
        return;
    }
    KeyCode keyCode = GameInput.GetKeyCode(inputNameEnum, input2, controllerButton);
    text.text = keyCode.ToString(); // ← вот источник
}
```

При переназначении клавиши (`SetNewKey`) метод вызывается повторно. Harmony-патч на setter ловит эти строки — фильтрация должна применяться на уровне переводчика, не на уровне патча.
