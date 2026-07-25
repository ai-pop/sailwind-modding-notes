# 05. Formatted labels with tabs

## Summary

Some TextMesh objects in the game contain **tabs** (`\t`, code 0x09) for column alignment — primarily in control settings. Structure: `Action\t\tKey`, where multiple tabs define the offset between columns. Naive translation (replacing the entire string via an online service) destroys the format.

## Symptom

When translating `Walk\tForward\tStop` through Google Translate:
- The translator treats `\t` as a separator or ignores it.
- Result may contain literal `/t/t/t` or lost columns.
- Labels in settings "shift" — text no longer aligns with buttons.

## Solution: tokenization by delimiters

Translate by segments separated by control characters, preserving original delimiters during reassembly:

```
Original:     Walk\tForward\tStop
                ↓
Tokenization:  [Walk] [\t] [Forward] [\t] [Stop]
                ↓
Translation:   [Go] [\t] [Forward] [\t] [Stop]
                ↓
Assembly:      Go\tForward\tStop
```

Delimiters for tokenization: `\t` `\n` `\r` `\f` `\v` (all standard C-format escapes).

## Implementation

```csharp
// Splitting with delimiter preservation.
// Consecutive delimiters are NOT collapsed — "Walk\t\tForward"
// yields [Walk] [\t] [\t] [Forward], preserving column width.
List<Token> Tokenize(string text) {
    var tokens = new List<Token>();
    var current = new StringBuilder();
    foreach (char c in text) {
        if (IsSeparator(c)) {
            if (current.Length > 0) {
                tokens.Add(Text(current.ToString()));
                current.Clear();
            }
            tokens.Add(Delimiter(c));
        } else {
            current.Append(c);
        }
    }
    if (current.Length > 0) tokens.Add(Text(current.ToString()));
    return tokens;
}
```

Each text segment is translated independently (via cache or online service). Already-translated segments (containing Cyrillic) are skipped. Technical segments (keycodes, numbers) are filtered (see [06-keycode-strings.md](06-keycode-strings.md)).
