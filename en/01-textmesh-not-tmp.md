# 01. TextMesh, not TextMeshPro

## Summary

Sailwind renders all in-game text through `UnityEngine.TextMesh` (legacy 3D-text component). TextMeshPro and `UnityEngine.UI.Text` are absent in the scene — zero instances of each.

## Confirmation

A runtime scene scanner (`FindObjectsOfType<TMP_Text>()` and `FindObjectsOfType<UnityEngine.UI.Text>()`) returns 0 for both types in any game state (main menu, settings, gameplay).

Decompilation of `Assembly-CSharp.dll` confirms: all classes working with text use `TextMesh` directly:

- `LookUI` — fields `[SerializeField] private TextMesh controlsText; hintText; extraText;`
- `StartMenuButton.SetButtonText(string)` — `GetComponent<TextMesh>().text = text;`
- `GPButtonKeybinding.UpdateText()` — `text.text = keyCode.ToString();`

## Consequences for modding

**Patch `TextMesh.text` setter, not `TMP_Text.text`.** Harmony patch:

```csharp
[HarmonyPatch(typeof(TextMesh), nameof(TextMesh.text), MethodType.Setter)]
[HarmonyPrefix]
public static void Prefix(ref string value) { /* ... */ }
```

A patch on `TMP_Text.text` or `UI.Text.text` won't work — such components don't exist in the scene.

**Fonts — `UnityEngine.Font`, not `TMP_FontAsset`.** `TMP_FontAsset.CreateFontAsset()` is not applicable. Cyrillic management is through `Font` and `Renderer.sharedMaterial` (see [04-static-font-trap.md](04-static-font-trap.md)).

**XUnity.AutoTranslator is ineffective.** By default it patches `TMP_Text` and `UI.Text` — neither exists. The plugin sits in memory doing nothing (`Skipping plugin scan because no plugin-specific translations has been registered`).

## Note

TextMesh is a deprecated component from Unity 4.x era. The Sailwind developer chose it deliberately (likely for VR compatibility and world-space 3D text). If the game transitions to TMP in future versions, this section would become irrelevant — but on v0.38 this is a key architectural constraint.
