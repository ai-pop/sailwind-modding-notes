# 02. Baked text bypasses C# setter

## Summary

A Harmony patch on `TextMesh.text` setter only intercepts **programmatic** assignments from C# code. Text baked into prefabs and scenes is set by the Unity engine during deserialization **bypassing the managed setter**. The patch doesn't see it.

## Problem statement

Button text in the main menu ("New Game", "Options", "Quit") is stored in `.unity` scenes and `.prefab` files as the serialized `m_Text` field of the TextMesh component. When a scene loads, Unity restores this field through native deserialization code, bypassing the C# property accessor. The Harmony-prefix on the setter doesn't fire.

Symptom: Harmony patch is installed and confirmed working (`[DIAG] PATCH TextMesh.text TRIGGERED`), but baked labels remain in the original language.

## Solution

An active scene scanner — a MonoBehaviour that periodically enumerates all TextMesh objects and applies translation directly:

```csharp
var meshes = FindObjectsOfType<TextMesh>();
foreach (var tm in meshes)
{
    string cur = tm.text;
    if (string.IsNullOrEmpty(cur)) continue;
    string translated = Translate(cur);
    if (translated != cur)
        tm.text = translated; // programmatic assignment — setter fires,
                                // but patch sees Cyrillic and skips it
}
```

Writing `tm.text = translated` calls the setter — the patch fires again, but sees Cyrillic and returns unchanged. No recursion.

## Practical parameters

- **Scan interval:** 0.5 sec is sufficient for responsiveness without overhead.
- **Rescan trigger:** on `SceneManager.sceneLoaded` (scene loaded → text deserialized → scan).
- **Optimization:** skip empty TextMesh and already-translated ones (containing Cyrillic).

## Interaction with setter patch

Both mechanisms are needed:

| Mechanism | What it catches | What it misses |
|-----------|-----------------|----------------|
| Harmony setter patch | Dynamic text (LookUI.ShowLookText, GPButtonKeybinding.UpdateText) | Baked in prefabs |
| Active scanner | Baked in prefabs | Text set between scanner passes |

The scanner covers the setter patch's gap; the setter patch covers the scanner's gap (instant reaction to dynamic text). Combined use gives full coverage.
