# 27. Сюжетные квесты (диалоги с NPC)

Разбор системы сюжетных квестов — **отдельной** от грузовых миссий (заметка 15). Это диалоговые квесты: NPC даёт задание репликами, игрок принимает, приносит предмет, получает награду. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## `Quest` (`[Serializable]`)

Описание квеста (данные диалога):

| Поле | Содержание |
|------|-----------|
| `questIndex` | Индекс квеста (ключ в `Quests.currentQuests`). |
| `goldReward` | Награда золотом (в **Gold Lions**, валюта 3). |
| `questLines[]` | Реплики NPC (вступительный диалог, постранично). |
| `playerResponses[]` / `playerAltResponses[]` | Варианты ответа игрока (кнопка). |
| `acceptPrefabIndex` | Предмет, выдаваемый при принятии квеста (индекс в `PrefabsDirectory.directory`; 0 = нет). |
| `deliveredQuestItemIndex` | Индекс предмета, который нужно принести для сдачи. |
| `inProgressLine` / `inProgressResponse` | Реплика/ответ «квест ещё выполняется». |
| `playerRepeatQuestion` | Кнопка-повтор вопроса. |
| `completionLine` / `playerCompletionResponse` | Реплика/ответ при сдаче. |
| `afterCompletedLine` | Реплика после завершения. |

## Состояния квеста (`Quests.currentQuests[]`)

Синглтон `Quests.instance`, массив `currentQuests` (сохраняется в сейв как `quests`). Коды состояний:

| Код | Состояние | Поведение `QuestDude` |
|:--:|-----------|----------------------|
| `0` (и `>=0`) | Доступен / идёт диалог | Показывает `questLines[]`, кнопка ведёт дальше. |
| `-1` | Принят, выполняется | `inProgressLine` / `inProgressResponse`. |
| `-5` | Завершён | `afterCompletedLine`, кнопка `"(bye)"`. |

## NPC-квестодатель (`QuestDude`, реализует `IGPButton`)

### Вход в триггер (`OnTriggerEnter`)
- **Игрок** (тег `Player`, предмета в триггере нет) → показывает `dialogUI`, звук `smallPop`, и по состоянию квеста:
  - `== 0` → `ShowDialog(0)` (вступление);
  - `== -1` → `ShowDialog(-1)` (в процессе);
  - иначе (`-5`) → `ShowDialog(-5)` (после завершения).
- **Предмет квеста** (`QuestItem` с `questItemIndex == quest.deliveredQuestItemIndex`) → `ShowDialog(-3)` (сдача).

### `ShowDialog(index)`
- `index >= 0` → `questLines[index]` + кнопка `playerResponses[index]`.
- `-1` → `inProgressLine` + `playerRepeatQuestion`.
- `-2` → `inProgressResponse` + `playerRepeatQuestion`.
- `-3` → `completionLine` + `playerCompletionResponse`.
- `-5` (default) → `afterCompletedLine` + `"(bye)"`.

Текст NPC переносится по словам с шириной **45 символов** (`Wrap`).

### Прогресс диалога (`ClickButton`)
1. Если в триггере есть предмет квеста (`cargoInTrigger`) → **`CompleteQuest`** → `ShowDialog(-5)`.
2. Иначе если состояние `-1` → `ShowDialog(-2)` (напоминание).
3. Иначе если состояние `>= 0` → следующая реплика (`questLinesIndex++`); **по достижении последней реплики:**
   - состояние → `-1` (квест принят);
   - если `acceptPrefabIndex != 0` → **спавнится предмет квеста** (`sold = true`, `RegisterToSave`) рядом с кнопкой.
4. Иначе → закрыть диалог.

### Завершение (`CompleteQuest`)
- Уничтожает принесённый предмет (`DestroyItem`).
- **Награда:** `PlayerGold.currency[3] += quest.goldReward` (Gold Lions), звук золота.
- Состояние квеста → `-5`.

## Предмет квеста (`QuestItem`)

Компонент на `ShipItem`: `questItemIndex`. В `Awake()` принудительно `sold = true` (предмет квеста всегда «куплен»/принадлежит игроку). Сдача происходит, когда предмет с нужным `questItemIndex` попадает в триггер `QuestDude`.

## Практические выводы для мододела

1. **Квесты — данные:** `Quest` содержит все реплики и параметры; логика универсальна в `QuestDude`. Добавлять квесты можно, наполняя `Quest`-объекты и `Quests.currentQuests`.
2. **Автомат состояний:** `0` (диалог) → `-1` (принят) → `-5` (выполнен). `currentQuests[questIndex]` — единственный источник истины.
3. **Награда** всегда в **Gold Lions** (валюта 3), не в региональных деньгах.
4. **Предмет квеста** (`QuestItem.questItemIndex`) сдаётся физически (триггер у NPC), как и грузовые миссии через `PortDude`.
5. **При принятии** может выдаваться предмет (`acceptPrefabIndex`), спавнится у кнопки.
6. Реплики NPC — массивы строк (перенос по 45 символов); кнопки-ответы — отдельные строки. Всё захардкожено в данных квеста (перевод — через подмену строк, см. заметки 01/08).
7. `Quests.currentQuests` сохраняется в сейв (`quests`, заметка 11).
