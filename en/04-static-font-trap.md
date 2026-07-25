# 04. Runtime font loading: static vs dynamic

## Summary

In Unity 2019.1 there are two ways to load a font from a `.ttf` file at runtime, and they behave fundamentally differently when replacing a `TextMesh` font. The wrong choice leads to **complete text disappearance**.

## `new Font(path)` — static font

```csharp
Font font = new Font("C:/path/to/font.ttf");
```

Creates a **static** font: glyph atlas is baked once on load. If needed glyphs (e.g. Cyrillic) aren't in the atlas at creation time — they never appear. Replacing a TextMesh font with this:

- `target.font = staticFont;`
- `target.GetComponent<Renderer>().sharedMaterial = staticFont.material;`

**Result:** text disappears completely. The log spams `Font size and style overrides are only supported for dynamic fonts`.

## `Font.CreateDynamicFontFromOSFont(name, size)` — dynamic font

```csharp
Font font = Font.CreateDynamicFontFromOSFont("Arial", 16);
```

Creates a **dynamic** font: glyphs are baked into texture on-demand during rendering. Supports any size, any character set, works correctly with TextMesh.

**Limitation:** searches for the font **by family name among OS-installed fonts**. A loose `.ttf` file from a plugin folder won't be found unless the name is registered in the system.

## Correct strategy for loose `.ttf`

1. **Parse family name from TTF.** Read the `name` table (nameID=1, platformID=3, UTF-16BE encoding):

```csharp
// Simplified: read offset table → find 'name' tag → nameID=1
string familyName = ReadFontFamilyName(path);
Font font = Font.CreateDynamicFontFromOSFont(familyName, 16);
```

2. **Cyrillic check** via `GetCharacterInfo` after `RequestCharactersInTexture`:

```csharp
font.RequestCharactersInTexture("Ая", 32);
CharacterInfo info;
bool hasCyrillic = font.GetCharacterInfo('А', out info, 32);
```

3. **Fallback to system Arial** if dynamic font creation by name fails or lacks Cyrillic glyphs.

## False negatives from `GetCharacterInfo`

For a freshly created dynamic font, the glyph texture **rebuilds asynchronously**. Calling `GetCharacterInfo` in the same frame may return `false` for a valid glyph. This doesn't mean the glyph is absent — it means the texture hasn't updated yet.

**Consequence:** you can't reject a font solely based on `GetCharacterInfo == false` in the first frame. A safe strategy is to accept the font if the file is valid, and let Unity finish baking glyphs on first render.

## Summary table

| Method | Font type | Cyrillic | Loose .ttf | TextMesh |
|--------|-----------|----------|------------|----------|
| `new Font(path)` | Static | Only pre-baked glyphs | Yes | **Breaks rendering** |
| `CreateDynamicFontFromOSFont(name)` | Dynamic | On-demand | Only OS-installed | Correct |
| `CreateDynamicFontFromOSFont(familyName from TTF)` | Dynamic | On-demand | Yes (via name parsing) | Correct |
