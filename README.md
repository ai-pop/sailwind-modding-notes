# Sailwind — технический справочник мододела

Документация по неочевидным архитектурным особенностям игры **Sailwind** (v0.38, Unity 2019.1.10f1), существенным для разработки модов. Содержит только информацию, которую невозможно получить без декомпиляции `Assembly-CSharp.dll` или эмпирического анализа runtime-поведения.

## Содержание

| № | Файл | Тема |
|---|------|------|
| 01 | [textmesh-not-tmp.md](01-textmesh-not-tmp.md) | Текст рендерится через TextMesh, не TextMeshPro |
| 02 | [baked-text-bypasses-setter.md](02-baked-text-bypasses-setter.md) | Зашитый в префабы текст обходит C# setter |
| 03 | [gopointer-input-system.md](03-gopointer-input-system.md) | Кастомная система ввода GoPointer вместо EventSystem |
| 04 | [static-font-trap.md](04-static-font-trap.md) | Особенности загрузки шрифтов в runtime |
| 05 | [tab-formatted-labels.md](05-tab-formatted-labels.md) | Форматированные подписи с разделителями колонок |
| 06 | [keycode-strings.md](06-keycode-strings.md) | Строки кодов клавиш, требующие исключения из перевода |
| 07 | [sre-false-harmony-works.md](07-sre-false-harmony-works.md) | Mono runtime без SRE — HarmonyX тем не менее работает |
| 08 | [lookui-hardcoded-strings.md](08-lookui-hardcoded-strings.md) | Полный список зашитых в код UI-строк |
| 09 | [google-translate-gotchas.md](09-google-translate-gotchas.md) | Особенности неофициального Google Translate endpoint |
| 10 | [class-reference.md](10-class-reference.md) | Краткий справочник классов Assembly-CSharp.dll |

## Предметная область

- **Игра:** [Sailwind](https://store.steampowered.com/app/1764530/Sailwind/) v0.38
- **Движок:** Unity 2019.1.10f1
- **Бэкенд:** Mono (рекомендуется), IL2CPP (экспериментально)
- **Фреймворк моддинга:** BepInEx 5.4.23.5
- **Метод анализа:** декомпиляция `Assembly-CSharp.dll` через ILSpy, runtime-логирование через BepInEx

## Лицензия

MIT. Материалы свободны для использования в любых модах для Sailwind.
