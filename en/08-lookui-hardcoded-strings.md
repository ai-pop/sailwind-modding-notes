# 08. Baked UI strings in LookUI

## Summary

`LookUI.ShowLookText(GoPointerButton button)` contains hardcoded English strings written into `controlsText.text` when hovering over interactive objects. These strings aren't stored in prefabs — they're formed in C# code and reach TextMesh through the setter.

## Complete list (extracted from decompilation)

### Actions (controlsText)

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

### Controller

```
"↑ release\n↓ pull"
```

### State

```
"(press a key)"
"(press a button)"
```

## Source in code

```csharp
// LookUI.ShowLookText — fragment
controlsText.text = "pick up\nbuy " + component.name + " for " + component.GetSellPriceString();
// ...
controlsText.text = "use\n";
// ...
controlsText.text = "repair hull";
```

The structure is a multi-level `if/else if` based on object type (`ShipItem`, `GPButtonRopeWinch`, `GPButtonSteeringWheel`, etc.) and context (is player holding an item, is object purchased, etc.).

## Consequences for translation

- Strings are intercepted by the Harmony patch on `TextMesh.text` setter (this is a programmatic assignment, not deserialization).
- Multi-line strings (`"pick up\nuse"`) require tokenization by `\n` (see [05-tab-formatted-labels.md](05-tab-formatted-labels.md)).
- Dynamic parts (`component.name`, `price`) — translated separately as standalone strings.
- Concatenation (`"buy " + name + " for " + price`) reaches TextMesh already assembled — translator receives `"buy Rope for 50 gold"` as a whole.

## Additional: GPButtonKeybinding

```csharp
text.text = "(press a key)";      // key remapping mode
text.text = "(press a button)";   // same for controller
```

These strings are translatable, but `keyCode.ToString()` from the same method is not (see [06-keycode-strings.md](06-keycode-strings.md)).
