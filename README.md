# Sailwind — Технический справочник мододела

[![Engine](https://img.shields.io/badge/Unity-2019.1.10f1-blue?logo=unity)]()
[![Backend](https://img.shields.io/badge/Mono-primary%20%7C%20IL2CPP-exp.-orange)]()
[![Framework](https://img.shields.io/badge/BepInEx-5.4.23.5_(HarmonyX)-purple)]()
[![Game version](https://img.shields.io/badge/Sailwind-v0.38-green)]()
[![Notes](https://img.shields.io/badge/notes-56-brightgreen)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey)]()

> [English version → `en/README.md`](en/README.md)

Полная документация по архитектуре и внутреннему устройству игры **Sailwind** (v0.38), полученная декомпиляцией `Assembly-CSharp.dll` и runtime-анализом. 56 заметок покрывают: системы сохранений, экономики, физики корпуса и океана, предметов (twin-модель), погоды, NPC, UI/текст, камеры, координатные системы и полный разбор бага мода SailwindItemPhysics v4.2 (краш при столкновении held-item twin с лодкой).

---

## Быстрый старт

| Что нужно | Где найти |
|-----------|-----------|
| Персистентность мода → `GameState.modData["MyMod.*"]` | [Заметка 11](11-save-system-and-mod-persistence.md) |
| Масштаб мира: **1 Unity unit = 1 метр**, mission km = 100 m | [Заметка 28](28-navigation-instruments-world-scale.md) |
| Глобальные ссылки и синглтоны | [Заметка 30](30-global-refs-singletons.md) |
| Центральный хук нового дня: `Sun.OnNewDay` | [Заметка 18](18-time-weather-storms.md) |
| Скрытый debug-режим: P+N, нажать T | [Заметка 21](21-debugger-cheats-tuning.md) |
| Неполная декомпиляция — какие классы отсутствуют | [Заметка 24](24-decompilation-coverage-missing-classes.md) |
| Фреймворк предметов: ShipItem → twin → ItemRigidbody | [Заметка 16](16-item-framework-shipitem.md) + [44](44-itemrigidbody-field-map-contract.md) |
| Краш мода с твёрдым twin-коллайдером | [Заметки 47–56](#исследование-мода-sailwinditemphysics-v42---раунд-2-e1e6) |

---

## Содержание

### Текст, UI и ввод
| № | Файл | Тема |
|---|------|------|
| 01 | [textmesh-not-tmp.md](01-textmesh-not-tmp.md) | Текст через TextMesh, не TextMeshPro |
| 02 | [baked-text-bypasses-setter.md](02-baked-text-bypasses-setter.md) | Зашитый текст обходит C# setter |
| 03 | [gopointer-input-system.md](03-gopointer-input-system.md) | Кастомная система ввода GoPointer |
| 04 | [static-font-trap.md](04-static-font-trap.md) | Загрузка шрифтов в runtime |
| 05 | [tab-formatted-labels.md](05-tab-formatted-labels.md) | Форматированные подписи с табами |
| 06 | [keycode-strings.md](06-keycode-strings.md) | Ключ-коды строк — исключить из перевода |
| 07 | [sre-false-harmony-works.md](07-sre-false-harmony-works.md) | Mono без SRE — HarmonyX работает |
| 08 | [lookui-hardcoded-strings.md](08-lookui-hardcoded-strings.md) | Полный список зашитых UI-строк |
| 09 | [google-translate-gotchas.md](09-google-translate-gotchas.md) | Google Translate endpoint особенности |
| 10 | [class-reference.md](10-class-reference.md) | Краткий справочник классов Assembly-CSharp.dll |

### Ядро: сохранение, состояние, ссылки
| № | Файл | Тема |
|---|------|------|
| 11 | [save-system-and-mod-persistence.md](11-save-system-and-mod-persistence.md) | Система сохранений, `GameState.modData` hook |
| 30 | [global-refs-singletons.md](30-global-refs-singletons.md) | Глобальные ссылки (Refs/RefsDirectory), синглтоны |
| 24 | [decompilation-coverage-missing-classes.md](24-decompilation-coverage-missing-classes.md) | Покрытие декомпиляции: отсутствующие классы |
| 21 | [debugger-cheats-tuning.md](21-debugger-cheats-tuning.md) | Отладчик, debug-режим (P+N+T), множители |
| 35 | [audio-mixer-ui-ambient.md](35-audio-mixer-ui-ambient.md) | Звук: mixer snapshots, UI-звуки, амбиент |
| 39 | [juicebox-tweens-gamefeel.md](39-juicebox-tweens-gamefeel.md) | Juicebox: твины, shake, hit-stop, частицы |

### Игрок и выживание
| № | Файл | Тема |
|---|------|------|
| 12 | [player-state-needs-recovery.md](12-player-state-needs-recovery.md) | Потребности, валюты, репутация, обморок |
| 25 | [sleep-bed-rest.md](25-sleep-bed-rest.md) | Сон, кровать, timeskip |
| 36 | [player-movement-climb-swim.md](36-player-movement-climb-swim.md) | Движение: прыжки, плавание, ванты, anti-stuck |

### Судно: физика, паруса, управление
| № | Файл | Тема |
|---|------|------|
| 14 | [boat-physics-buoyancy-damage.md](14-boat-physics-buoyancy-damage.md) | Физика корпуса: плавучесть, масса, урон, потопление |
| 17 | [wind-and-sails.md](17-wind-and-sails.md) | Ветер (пассаты, шквалы, шторм) и парусная тяга |
| 29 | [anchor-mooring-ropes.md](29-anchor-mooring-ropes.md) | Якорь, швартовка, тросы |
| 22 | [shipyard-customization-purchase.md](22-shipyard-customization-purchase.md) | Верфь: кастомизация лодок, покупка |
| 38 | [ropes-rigging-steering.md](38-ropes-rigging-steering.md) | Такелаж: штурвал (пружина руля), лебёдки, автопилот |
| 40 | [hull-dirt-cleaning.md](40-hull-dirt-cleaning.md) | Грязь корпуса и чистка (MasterPainter) |

### Мир, время, океан
| № | Файл | Тема |
|---|------|------|
| 18 | [time-weather-storms.md](18-time-weather-storms.md) | Время (`OnNewDay`), луна, погода, блуждающие штормы |
| 19 | [world-ports-regions.md](19-world-ports-regions.md) | Регионы, порты (34 макс), границы мира |
| 31 | [ocean-waves-inertia.md](31-ocean-waves-inertia.md) | Океан Crest: инерция волн, запрос высоты |
| 43 | [item-buoyancy-water.md](43-item-buoyancy-water.md) | Плавучесть предметов: ItemRigidbody/SimpleFloatingObject |
| 28 | [navigation-instruments-world-scale.md](28-navigation-instruments-world-scale.md) | Навигация: **1 unit = 1 m**, координаты |
| 37 | [maps-charts-drawing.md](37-maps-charts-drawing.md) | Карты/чарты: ChartData, рисование, GPS |
| 41 | [cameras-view-modes.md](41-cameras-view-modes.md) | Камеры: BoatCamera, CameraFollow, свои камеры |
| 46 | [water-physics-illusion.md](46-water-physics-illusion.md) | Физика воды: иллюзия объёма, SampleHeightHelper lifecycle |

### Экономика, миссии, предметы
| № | Файл | Тема |
|---|------|------|
| 13 | [economy-markets-currency.md](13-economy-markets-currency.md) | Экономика: рынки, валюты, цены |
| 15 | [missions-cargo-mail.md](15-missions-cargo-mail.md) | Миссии: генерация из арбитража, доставка |
| 27 | [story-quests.md](27-story-quests.md) | Сюжет: QuestDude, автомат состояний, награды |
| 16 | [item-framework-shipitem.md](16-item-framework-shipitem.md) | Фреймворк предметов: ShipItem, физика, инвентарь |
| 44 | [itemrigidbody-field-map-contract.md](44-itemrigidbody-field-map-contract.md) | `ItemRigidbody`: карта полей, twin-контракт |
| 45 | [crate-cargo-prefabs-filter.md](45-crate-cargo-prefabs-filter.md) | Ящики: `amount`/`containedPrefab`, лут-фильтр |
| 32 | [inventory-cargo-storage.md](32-inventory-cargo-storage.md) | Инвентарь (5 слотов), cargo carriers |
| 23 | [fishing-and-food.md](23-fishing-and-food.md) | Рыбалка, порча и консервация еды |
| 26 | [cooking-stove-fuel.md](26-cooking-stove-fuel.md) | Кулинария: плита, топливо, варка/копчение |
| 33 | [item-spawning-pickup.md](33-item-spawning-pickup.md) | Спавн предметов, подбор, плавучесть |
| 34 | [worked-example-floating-loot.md](34-worked-example-floating-loot.md) | Пример: плавающие лут-ящики (BepInEx рецепт) |
| 42 | [worked-example-fast-travel.md](42-worked-example-fast-travel.md) | Пример: фаст-тревел/телепорт |

### Население
| № | Файл | Тема |
|---|------|------|
| 20 | [npcs-world-population.md](20-npcs-world-population.md) | NPC: waypoint boats, рыбаки, PortDude, Shopkeeper |

### Исследование мода SailwindItemPhysics v4.2 — раунд 1 (A1–A5, B6–B12, C13–C14, D15)
| № | Файл | Тема |
|---|------|------|
| 47 | [item-holding-pickup-flow.md](47-item-holding-pickup-flow.md) | Удержание предмета: end-to-end flow, twin vs visual, teleport при холде, `held` lifecycle |
| 48 | [ocean-height-helper-lifecycle.md](48-ocean-height-helper-lifecycle.md) | Ocean height: `SampleHeightHelper` lifecycle, canCheckBuoyancyNow, batch-запросы |
| 49 | [camera-render-shaders.md](49-camera-render-shaders.md) | Камеры/шейдеры: BoatCamera, rendering path, Z-write, depth fog |
| 50 | [saves-moddata-instanceid.md](50-saves-moddata-instanceid.md) | Saves/modData: instanceId, SaveablePrefab, modData persistence, что теряется |

### Исследование мода SailwindItemPhysics v4.2 — раунд 2 (E1–E6)
> Краш/зависание при столкновении held-item твёрдого twin-коллайдера с лодкой.

| № | Файл | Тема |
|---|------|------|
| 51 | [boat-collider-topology-dhow.md](51-boat-collider-topology-dhow.md) | **E1:** Топология коллайдеров лодки (dhow): CapsuleCollider (root), BoatEmbarkCollider, walkCollider, HullPlayerCollider, иерархия GO |
| 52 | [layers-collision-matrix-items-boat-player.md](52-layers-collision-matrix-items-boat-player.md) | **E2:** Слои Unity (0/2/5/8/12/13/14/23/26), матрица столкновений, почему twin (слой 2) сталкивается с CapsuleCollider, а игрок — нет |
| 53 | [enterboat-exitboat-flap-mechanism.md](53-enterboat-exitboat-flap-mechanism.md) | **E3:** EnterBoat/ExitBoat: trigger callbacks, `frameCounter>1` gate, BoatMass.AddItem idempotent, ≥22 Hz flap |
| 54 | [go-pointer-big-item-decollision.md](54-go-pointer-big-item-decollision.md) | **E4:** Деколлизия big-item: нет while-циклов, ComputePenetration single-pass, Boat tag исключён |
| 55 | [crash-scenarios-boat-item-collision.md](55-crash-scenarios-boat-item-collision.md) | **E5:** Краш-сценарии: twin Untagged → BoatDamage.Impact не фильтрует → hullDamage 0.15/s |
| 56 | [nailed-wallattachment-deck-placement.md](56-nailed-wallattachment-deck-placement.md) | **E6:** nailed (kinematic lock), wallAttachment (raycast+snap), HangableItem, палубная посадка |

---

## Техническая информация

### Декомпиляция

| Параметр | Значение |
|----------|----------|
| DLL | `Assembly-CSharp.dll` |
| Версия игры | Sailwind v0.38 (Steam) |
| Инструмент | ILSpy 7.x |
| Покрытие | ~828 классов (полное за исключением `GameInput`, `InputName`, `BoatProbes`, Crest helpers — API реконструирован в заметке 24) |
| Runtime-верификация | BepInEx console + Harmony hooks + Debug.Log |

### Twin-модель предмета (ключевой концепт)

У каждого `ShipItem` два GameObject:
- **Visual GO** (`ShipItem` + Rigidbody kinematic + Collider isTrigger) — что видит игрок, raycast, interact
- **Twin GO** (`ItemRigidbody` + Rigidbody dynamic + Collider isTrigger в vanilla / non-trigger в моде) — физика, buoyancy, `EnterBoat`/`ExitBoat`

В vanilla twin = isTrigger — «прозрачный» для CapsuleCollider лодки. Мод SailwindItemPhysics v4.2 делает twin non-trigger — twin физически сталкивается с CapsuleCollider, что приводит к краш-сценариям, разобранным в заметках 51–55.

### Unity-специфика Sailwind

| Система | Детали |
|---------|--------|
| Океан | Кастомный форк Crest (OceanRenderer, SampleHeightHelper, batch queries) |
| FloatingOrigin | `FloatingOriginManager` — сдвиг мира каждые ~1 км для precision (заметки 43, 47) |
| Слои | 0=Default, 2=IgnoreRaycast(twin), 5=UI, 8=Player, 12=HullPlayerCollider(walkCol), 13=Boat, 14=Terrain, 23=Map, 26=Crate (заметка 52) |
| Raycast mask | `LayerMask.op_Implicit(-604165)` — GoPointer видит большинство слоёв, не видит 2/5 |
| Fixed timestep | `fixedDeltaTime = 0.022s` (~45.5 Hz physics) |
| Debug mode | P+N → T — скрытые множители, kinematic items, disableItemUpdate (заметка 21) |

---

## Структура репозитория

```
sailwind-modding-notes/
├── README.md                      ← Русский (основной)
├── LICENSE                        ← MIT
├── 01-textmesh-not-tmp.md ... 56-*.md   ← 56 заметок (RU)
└── en/
    ├── README.md                  ← English translation
    ├── 01-textmesh-not-tmp.md ... 56-*.md   ← 56 заметок (EN)
```

Каждая заметка — самостоятельный Markdown-документ с:
- Заголовком, ссылкой на research request (для 47–56)
- Таблицами полей/типов/значений
- Вербатимными телами ключевых методов (ILSpy)
- Sequence-диаграммами и блок-схемами
- Практическими выводами «что это значит для мододела»

---

## Лицензия

MIT — материалы свободны для использования в любых модах для Sailwind. См. [LICENSE](LICENSE).
