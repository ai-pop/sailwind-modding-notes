# 08. Зашитые UI-строки в LookUI

## Суть

`LookUI.ShowLookText(GoPointerButton button)` содержит зашитые в код английские строки, записываемые в `controlsText.text` при наведении на интерактивные объекты. Эти строки не хранятся в префабах — они формируются в C#-коде и попадают в TextMesh через setter.

## Полный список (извлечён из декомпиляции)

### Действия (controlsText)

```
"pick up\n"
"pick up\nuse"
"pick up\nopen"
"pick up\nbuy " + component.name + " for " + price
"pick up\nload " + component.name + " for " + price
"pick up\nchange length"
"pick up\ntransport for " + price
"\nuse"
"\nopen"
"\nlock"
"\nunlock"
"\ncut"
"use\n"
"use"
"cook\n"
"fill\n"
"attach\n"
"push\n"
"place item\n"
"salt food\n"
"repair\n"
"repair hull"
"use charting kit\n"
"row"
"empty"
"drink"
"eat"
"add oil\nuse"
"replace candle\nuse"
```

### Контроллер

```
"↑ release\n↓ pull"
```

### Состояние

```
"(press a key)"
"(press a button)"
```

## Источник в коде

```csharp
// LookUI.ShowLookText — фрагмент
controlsText.text = "pick up\nbuy " + component.name + " for " + component.GetSellPriceString();
// ...
controlsText.text = "use\n";
// ...
controlsText.text = "repair hull";
```

Структура — многоуровневый `if/else if` на основе типа объекта (`ShipItem`, `GPButtonRopeWinch`, `GPButtonSteeringWheel`, и т.д.) и контекста (держит ли игрок предмет, куплен ли объект, и т.д.).

## Следствия для перевода

- Строки перехватываются Harmony-патчем на `TextMesh.text` setter (это программное присваивание, не десериализация).
- Многострочные строки (`"pick up\nuse"`) требуют токенизации по `\n` (см. [05-tab-formatted-labels.md](05-tab-formatted-labels.md)).
- Динамические части (`component.name`, `price`) — переводятся отдельно как самостоятельные строки.
- Конкатенация (`"buy " + name + " for " + price`) в TextMesh попадает уже собранной — переводчик получает `"buy Rope for 50 gold"` целиком.

## Дополнительно: GPButtonKeybinding

```csharp
text.text = "(press a key)";      // режим переназначения
text.text = "(press a button)";   // то же для контроллера
```

Эти строки переводимы, но `keyCode.ToString()` из того же метода — нет (см. [06-keycode-strings.md](06-keycode-strings.md)).
