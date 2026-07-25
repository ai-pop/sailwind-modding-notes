# 21. Отладчик, скрытый debug-режим и глобальные множители

Разбор встроенного отладчика и скрытого чит-режима. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Полезно для тестирования модов, понимания точек баланса и «внутренних» механизмов.

## Скрытый debug-режим (работает в релизе!)

В классе `Debugger` зашит **секретный код активации**, срабатывающий даже в собранной игре (не только в редакторе):

```
Удерживать P + N  и нажать T
→ buildDebugModeOn = true
→ PlayerNeeds.instance.godMode = true   (бессмертие потребностей)
→ включаются спидометры (speedometers)
```

Проверка: `Input.GetKey("p") && Input.GetKey("n") && Input.GetKeyDown("t")`.

### Что даёт `buildDebugModeOn`
| Клавиша | Действие |
|---------|----------|
| `0` / `1` / `2` / `5` / `6` / `8` / `9` | `BoatProbes.ChangeEnginePower(...)`: мощность двигателя лодки = `0 / 0.5 / 2 / 5 / -5 / 8 / 50` (фактически «моторный» чит на скорость). |
| `Keypad5` (KeyCode 261) | Переключить `debugWind` (форсирует ветер `(10,0,10)`). |
| `Keypad1` (257) | `Time.timeScale = debugTimescale` (по умолч. 2). |
| `Keypad3` (259) | `Time.timeScale = 1` (норма). |

> `buildDebugModeOn` также **снимает паузу времени** в UI торговли (`SunPaused`/`PlayerNeeds.LateUpdate` проверяют `!Debugger.buildDebugModeOn`).

## Клавиши только в редакторе Unity (`Application.isEditor`)

| Клавиша (KeyCode) | Действие |
|-------------------|----------|
| `Keypad2` (258) | `PlayerNeedsUI.DebugHideInventory()` |
| `Keypad7` (263) | `Sun.initialTimescale = 0.008` («×1», очень медленное время) |
| `Keypad9` (265) | `Sun.initialTimescale = 0.8` («×100») |
| `Keypad4` (260) | `Time.fixedDeltaTime = 0.02222` |
| `Keypad6` (262) | `Time.fixedDeltaTime = 0.002222` |
| `Keypad8` (264) | Лог vitamins/protein |
| `KeypadPlus` (270) / `KeypadMinus` (269) | `debugYardMult ± 0.01` (мин. раскрытие паруса) |
| `F8` (289) | `targetFrameRate = 3` |
| `F9` (290) | `targetFrameRate = -1` (без лимита) |
| `F11` (292) | Скриншот (путь разработчика `D:/Pictures/Sailwind Promo/`) |
| `F5` (286) | **+250 репутации** (регион alankh) |
| `F6` (287) | **+1000 ко всем 4 валютам** |
| `F4` (285) | `PlayerNeeds.water -= 10` |
| `-` | `Sun.globalTime += 1` (перемотка +1 час) |
| `j` | Переключить `debugForceKinematicBoat` |
| `b` | `recoveryStep = true` (пошаговая отладка восстановления) |
| `v` | Заспавнить предметы (префабы 110 и 79), `sold + RegisterToSave` |
| `m` + `v` | `clearSaveData` → `PlayerPrefs.DeleteAll()` |
| `i` | Лог всех `SaveablePrefab.existingInstanceIds` |

## Глобальные тюнинг-множители (`Debugger.instance`)

Эти поля **влияют на геймплей в релизе** (их читают боевые системы):

| Поле | По умолч. | Где используется |
|------|:--:|------------------|
| `debugSailAreaMult` | 1 | `Sail.GetRealSailPower()` — множитель площади/тяги **всех** парусов (заметка 17). |
| `debugYardMult` | 0.015 | Минимальная эффективность паруса при убранном полотне (`max(debugYardMult, currentUnroll)`). |
| `debugItemColAudioThreshold` | 0.1 | Порог звука столкновений предметов. |
| `debugTimescale` | 2 | Цель для `Time.timeScale` по Keypad1. |

Статические флаги: `buildDebugModeOn`, `debugWind`, `debugForceKinematicBoat`, `recoveryStep`, `debugRecoveryFlag`, `sailForwardShare`, `kinematicItemsTimer`.

### Прочие «внутренности»
- В `Awake()` всегда выставляется `PlayerGold.gold = 100` (и в редакторе, и в релизе) — стартовый «легаси»-золотой запас.
- `Settings.hintTextEnabled` и `itemNameTextEnabled` форсируются в `true` при старте.
- Флаги отключения предметов: `disableItemCols`, `disableItemRigidbodyCols`, `disableItemRenderers`, `disableItemUpdate`, `disableItemRigidbodyUpdate`, `destroyAllItems` — полезны для диагностики производительности/физики предметов.

## Другие отладочные классы

| Класс | Назначение |
|-------|-----------|
| `DebugMarketTracker` | Баланс-хаб экономики и миссий (заметка 13). |
| `DebugWeather` | Форсирование погоды. |
| `DebugSunTime` / `DebugFogController` | Управление временем/туманом. |
| `DebugUIBuilder` / `DebugUISample` | Построение отладочного UI. |
| `DebugPainter` / `DebugSimpleDecollision` | Отладка рисования/деколлизии. |
| `DebugMarketTracker2/3` | Дополнительные трекеры рынка. |
| `BoatDebugger` / `EditorDebugger` | Отладка лодок/редактора. |

## Практические выводы для мододела

1. **Скрытый чит P+N+T** включает godMode и debug-режим в релизе — можно использовать для тестирования, но учтите: это «настоящий» код игры, не пасхалка мода.
2. **`Debugger.instance.debugSailAreaMult`** — легальный глобальный множитель тяги парусов; удобная точка для модов баланса парусов.
3. **`debugYardMult`** — минимальная тяга убранного паруса.
4. `buildDebugModeOn` снимает паузу времени в UI торговли — учитывайте, если ваш мод полагается на паузу.
5. Клавиши F5/F6/цифры двигателя и Keypad-время — только для понимания; в релизе активна лишь подгруппа (см. `buildDebugModeOn`).
6. Начальный `PlayerGold.gold = 100` задаётся в `Debugger.Awake`.
