# 09. Неофициальный Google Translate endpoint

## Endpoint

```
https://translate.googleapis.com/translate_a/single?client=gtx&sl=en&tl=ru&dt=t&q=<URL-encoded-text>
```

Параметр `client=gtx` включает свободный режим без API-ключа. Работает без авторизации, без ограничений по количеству запросов (в отличие от официального API). Скорость — приемлемая для пакетной обработки (10–20 строк/сек при 4 параллельных потоках).

## Формат ответа

Нестандартный JSON (не валидный для строгих парсеров):

```json
[[["перевод","original",null,null,3]],null,"en", ...]
```

Каждый сегмент перевода — массив `["translatedText","sourceText",null,null,confidence]`. При длинном тексте ответ содержит несколько сегментов, которые нужно конкатенировать.

## Парсинг

Строгий JSON-парсер (Newtonsoft, System.Text.Json) не справится с этим форматом напрямую (лишние запятые, null-элементы). Рекомендуется regex-извлечение:

```csharp
var matches = Regex.Matches(json, @"\[\[""(?<tr>(?:[^""\\]|\\.)*)"",""(?<src>(?:[^""\\]|\\.)*)""");
var sb = new StringBuilder();
foreach (Match m in matches)
    sb.Append(Unescape(m.Groups["tr"].Value));
```

## Tracking-ID в конце перевода

Наблюдалось: Google иногда дописывает в конец перевода **32-символьный hex-хеш** (tracking identifier), например:

```
"Изумрудный архипелаг8e6adaf1f9ae06bcb9663531e5521abb"
```

Этот хеш — не часть перевода. Очистка:

```csharp
translated = Regex.Replace(translated, "[0-9a-fA-F]{16,}.*$", "");
```

16+ hex-символов подряд в конце строки — индикатор мусора. После очистки результат валиден.

## Fallback: MyMemory API

```
https://api.mymemory.translated.net/get?q=<text>&langpair=en|ru
```

- Без ключа, до ~5000 слов/день.
- Возвращает стандартный JSON: `{"responseData":{"translatedText":"..."}}`.
- При превышении лимита: `"MYMEMORY WARNING: YOU USED ALL AVAILABLE FREE TRANSLATIONS FOR TODAY"` —检测ить и пропустить.

## TLS 1.2

Mono runtime Unity 2019 может не включать TLS 1.2 по умолчанию. Перед HTTP-запросами:

```csharp
ServicePointManager.SecurityProtocol = (SecurityProtocolType)3072; // TLS 1.2
```

Без этого HTTPS-запросы к `translate.googleapis.com` могут завершаться с ошибкой соединения.

## Многопоточность

Перевод в фоновых потоках (`System.Threading.Thread`) с очередью (`HashSet` для дедупликации, `Monitor.Wait/Pulse` для ожидания). Главный поток только ставит строки в очередь и читает готовый результат из кеша. Обновление UI/сцены — через флаг `NeedsRescan`, опрашиваемый в `Update()`.
