# 09. Unofficial Google Translate endpoint

## Endpoint

```
https://translate.googleapis.com/translate_a/single?client=gtx&sl=en&tl=ru&dt=t&q=<URL-encoded-text>
```

The `client=gtx` parameter enables free mode without an API key. Works without authorization, no request count limits (unlike the official API). Speed — acceptable for batch processing (10–20 strings/sec with 4 parallel threads).

## Response format

Non-standard JSON (not valid for strict parsers):

```json
[[["translation","original",null,null,3]],null,"en", ...]
```

Each translation segment is an array `["translatedText","sourceText",null,null,confidence]`. For long text, the response contains multiple segments that need concatenation.

## Parsing

A strict JSON parser (Newtonsoft, System.Text.Json) won't handle this format directly (extra commas, null elements). Regex extraction is recommended:

```csharp
var matches = Regex.Matches(json, @"\[\[""(?<tr>(?:[^""\\]|\\.)*)"",""(?<src>(?:[^""\\]|\\.)*)""");
var sb = new StringBuilder();
foreach (Match m in matches)
    sb.Append(Unescape(m.Groups["tr"].Value));
```

## Tracking-ID appended to translation

Observed: Google sometimes appends a **32-character hex hash** (tracking identifier) to the end of translations, e.g.:

```
"Emerald Archipelago8e6adaf1f9ae06bcb9663531e5521abb"
```

This hash is not part of the translation. Cleanup:

```csharp
translated = Regex.Replace(translated, "[0-9a-fA-F]{16,}.*$", "");
```

16+ consecutive hex characters at the end of a string indicate garbage. After cleanup, the result is valid.

## Fallback: MyMemory API

```
https://api.mymemory.translated.net/get?q=<text>&langpair=en|ru
```

- No key, up to ~5000 words/day.
- Returns standard JSON: `{"responseData":{"translatedText":"..."}}`.
- On limit exceeded: `"MYMEMORY WARNING: YOU USED ALL AVAILABLE FREE TRANSLATIONS FOR TODAY"` — detect and skip.

## TLS 1.2

Unity 2019 Mono runtime may not enable TLS 1.2 by default. Before HTTP requests:

```csharp
ServicePointManager.SecurityProtocol = (SecurityProtocolType)3072; // TLS 1.2
```

Without this, HTTPS requests to `translate.googleapis.com` may fail with connection errors.

## Multithreading

Translation in background threads (`System.Threading.Thread`) with a queue (`HashSet` for deduplication, `Monitor.Wait/Pulse` for waiting). Main thread only queues strings and reads ready results from cache. UI/scene updates — via `NeedsRescan` flag polled in `Update()`.
