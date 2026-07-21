# Sailwind — технический справочник мододела

Документация по архитектуре и внутреннему устройству игры **Sailwind** (v0.38, Unity 2019.1.10f1), существенная для разработки модов. Содержит информацию, которую невозможно получить без декомпиляции `Assembly-CSharp.dll` или эмпирического анализа runtime-поведения: системы сохранений, экономики, физики, погоды, предметов, NPC и многое другое.

Справочник вырос из заметок по моду-русификатору (заметки 01–10) в **полное описание игровых систем** (заметки 11–32). Начните с заметки [30 (глобальные ссылки и синглтоны)](30-global-refs-singletons.md) и [24 (покрытие декомпиляции)](24-decompilation-coverage-missing-classes.md), если хотите быстро сориентироваться.

## Содержание по разделам

### Текст, UI и ввод (из моду-русификатора)
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

### Ядро: сохранение, состояние, ссылки
| № | Файл | Тема |
|---|------|------|
| 11 | [save-system-and-mod-persistence.md](11-save-system-and-mod-persistence.md) | Система сохранений, формат сейвов, **хук `GameState.modData`** |
| 30 | [global-refs-singletons.md](30-global-refs-singletons.md) | Глобальные ссылки (Refs/RefsDirectory), карта синглтонов, константы |
| 24 | [decompilation-coverage-missing-classes.md](24-decompilation-coverage-missing-classes.md) | Покрытие декомпиляции: отсутствующие классы и реконструкция их API |
| 21 | [debugger-cheats-tuning.md](21-debugger-cheats-tuning.md) | Отладчик, скрытый debug-режим (P+N+T), глобальные множители |
| 35 | [audio-mixer-ui-ambient.md](35-audio-mixer-ui-ambient.md) | Звук: микшер-снапшоты, UI-звуки, амбиент по времени суток |

### Игрок и выживание
| № | Файл | Тема |
|---|------|------|
| 12 | [player-state-needs-recovery.md](12-player-state-needs-recovery.md) | Потребности, валюты, репутация, механика обморока |
| 25 | [sleep-bed-rest.md](25-sleep-bed-rest.md) | Сон, кровать, timeskip и ускорение времени |

### Судно: физика, паруса, управление
| № | Файл | Тема |
|---|------|------|
| 14 | [boat-physics-buoyancy-damage.md](14-boat-physics-buoyancy-damage.md) | Физика корпуса: плавучесть, масса, урон и потопление |
| 17 | [wind-and-sails.md](17-wind-and-sails.md) | Модель ветра (пассаты, шквалы, шторм) и парусной тяги |
| 29 | [anchor-mooring-ropes.md](29-anchor-mooring-ropes.md) | Якорь (авто-зацеп/срыв), швартовка, контроллеры тросов |
| 22 | [shipyard-customization-purchase.md](22-shipyard-customization-purchase.md) | Верфь, кастомизация лодок (зависимости частей), покупка |

### Мир, время, океан
| № | Файл | Тема |
|---|------|------|
| 18 | [time-weather-storms.md](18-time-weather-storms.md) | Время (`OnNewDay`), луна, региональная погода, блуждающие штормы |
| 19 | [world-ports-regions.md](19-world-ports-regions.md) | Регионы, порты (34 макс), локальные карты, граница мира |
| 31 | [ocean-waves-inertia.md](31-ocean-waves-inertia.md) | Океан Crest: инерция волнового поля, запрос высоты волны |
| 28 | [navigation-instruments-world-scale.md](28-navigation-instruments-world-scale.md) | Навигация и вывод масштаба: **1 единица = 1 метр** |

### Экономика, миссии, предметы
| № | Файл | Тема |
|---|------|------|
| 13 | [economy-markets-currency.md](13-economy-markets-currency.md) | Экономика: рынки, валюты, модель цен, торговые лодки |
| 15 | [missions-cargo-mail.md](15-missions-cargo-mail.md) | Миссии: генерация из арбитража, доставка грузов и почты |
| 27 | [story-quests.md](27-story-quests.md) | Сюжетные квесты: диалоги QuestDude, автомат состояний, награды |
| 16 | [item-framework-shipitem.md](16-item-framework-shipitem.md) | Фреймворк предметов: ShipItem, физика, инвентарь, семейство |
| 32 | [inventory-cargo-storage.md](32-inventory-cargo-storage.md) | Инвентарь (5 слотов), грузовые носители, цены перевозки/хранения |
| 23 | [fishing-and-food.md](23-fishing-and-food.md) | Рыбалка (клёв, натяжение, улов), порча и консервация еды |
| 26 | [cooking-stove-fuel.md](26-cooking-stove-fuel.md) | Кулинария: плита, топливо, варка/копчение, степень готовности |
| 33 | [item-spawning-pickup.md](33-item-spawning-pickup.md) | Спавн предметов в мире, подбор, плавучесть, ящики + взгляд моддера |
| 34 | [worked-example-floating-loot.md](34-worked-example-floating-loot.md) | Рабочий пример: плавающие лут-ящики в море (рецепт BepInEx-мода) |

### Население
| № | Файл | Тема |
|---|------|------|
| 20 | [npcs-world-population.md](20-npcs-world-population.md) | NPC: лодки-вейпоинты, рыбаки, PortDude, Shopkeeper, жители |

## Ключевые факты для быстрого старта

- **Персистентность мода:** пишите в `GameState.modData["MyMod.*"]` — сохранится автоматически (заметка 11).
- **Масштаб мира:** 1 единица Unity = 1 метр; «км» миссий = 100 м; координаты глобуса = метры / 9000 (заметка 28).
- **Центральный дневной хук:** `Sun.OnNewDay` (заметка 18).
- **Декомпиляция неполная:** отсутствуют `GameInput`, `InputName`, `BoatProbes`, Crest-хелперы — их API реконструированы (заметка 24).
- **Скрытый debug-режим в релизе:** удерживать P+N, нажать T (заметка 21).

## Предметная область

- **Игра:** [Sailwind](https://store.steampowered.com/app/1764530/Sailwind/) v0.38
- **Движок:** Unity 2019.1.10f1 (океан — кастомный форк Crest)
- **Бэкенд:** Mono (рекомендуется), IL2CPP (экспериментально)
- **Фреймворк моддинга:** BepInEx 5.4.23.5 (HarmonyX)
- **Метод анализа:** декомпиляция `Assembly-CSharp.dll` через ILSpy, runtime-логирование через BepInEx
- **Исходный материал:** [sailwind-decompiled](https://github.com/ai-pop/sailwind-decompiled) (677 игровых классов), мод [sailwind-translator](https://github.com/ai-pop/sailwind-translator)

## Лицензия

MIT. Материалы свободны для использования в любых модах для Sailwind.
