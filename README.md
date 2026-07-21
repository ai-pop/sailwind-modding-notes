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
| 11 | [save-system-and-mod-persistence.md](11-save-system-and-mod-persistence.md) | Система сохранений, формат сейвов, хук `GameState.modData` |
| 12 | [player-state-needs-recovery.md](12-player-state-needs-recovery.md) | Потребности, валюты, репутация, механика обморока |
| 13 | [economy-markets-currency.md](13-economy-markets-currency.md) | Экономика: рынки, валюты, модель цен, торговые лодки |
| 14 | [boat-physics-buoyancy-damage.md](14-boat-physics-buoyancy-damage.md) | Физика корпуса: плавучесть, масса, урон и потопление |
| 15 | [missions-cargo-mail.md](15-missions-cargo-mail.md) | Миссии: генерация из арбитража, доставка грузов и почты |
| 16 | [item-framework-shipitem.md](16-item-framework-shipitem.md) | Фреймворк предметов: ShipItem, физика, инвентарь, семейство |
| 17 | [wind-and-sails.md](17-wind-and-sails.md) | Модель ветра (пассаты, шквалы, шторм) и парусной тяги |
| 18 | [time-weather-storms.md](18-time-weather-storms.md) | Время (OnNewDay), луна, региональная погода, блуждающие штормы |
| 19 | [world-ports-regions.md](19-world-ports-regions.md) | Регионы, порты (34 макс), локальные карты, граница мира |
| 20 | [npcs-world-population.md](20-npcs-world-population.md) | NPC: лодки-вейпоинты, рыбаки, PortDude, Shopkeeper, жители |
| 21 | [debugger-cheats-tuning.md](21-debugger-cheats-tuning.md) | Отладчик, скрытый debug-режим (P+N+T), глобальные множители |
| 22 | [shipyard-customization-purchase.md](22-shipyard-customization-purchase.md) | Верфь, кастомизация лодок (зависимости частей), покупка |
| 23 | [fishing-and-food.md](23-fishing-and-food.md) | Рыбалка (клёв, натяжение, улов), порча и консервация еды, жидкости |
| 24 | [decompilation-coverage-missing-classes.md](24-decompilation-coverage-missing-classes.md) | Покрытие декомпиляции: отсутствующие классы и реконструкция их API |
| 25 | [sleep-bed-rest.md](25-sleep-bed-rest.md) | Сон, кровать, timeskip и ускорение времени |
| 26 | [cooking-stove-fuel.md](26-cooking-stove-fuel.md) | Кулинария: плита, топливо, варка/копчение, степень готовности |
| 27 | [story-quests.md](27-story-quests.md) | Сюжетные квесты: диалоги QuestDude, автомат состояний, награды |
| 28 | [navigation-instruments-world-scale.md](28-navigation-instruments-world-scale.md) | Навигация (компас, лаг, флюгер, хронометр) и вывод масштаба: 1 ед. = 1 м |
| 29 | [anchor-mooring-ropes.md](29-anchor-mooring-ropes.md) | Якорь (авто-зацеп/срыв), швартовка, контроллеры тросов |

## Предметная область

- **Игра:** [Sailwind](https://store.steampowered.com/app/1764530/Sailwind/) v0.38
- **Движок:** Unity 2019.1.10f1
- **Бэкенд:** Mono (рекомендуется), IL2CPP (экспериментально)
- **Фреймворк моддинга:** BepInEx 5.4.23.5
- **Метод анализа:** декомпиляция `Assembly-CSharp.dll` через ILSpy, runtime-логирование через BepInEx

## Лицензия

MIT. Материалы свободны для использования в любых модах для Sailwind.
