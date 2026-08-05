# 78. Диалоги, слухи, процедурные анимации NPC и система таможни/контрабанды

Полный технический разбор систем взаимодействия игрока с NPC, процедурной анимации персонажей, генерации слухов в таверне и механик таможенного досмотра в Sailwind v0.38 (`Assembly-CSharp.dll`). Дополняет сюжетные квесты ([заметка 27](27-story-quests.md)) и население мира ([заметка 20](20-npcs-world-population.md)).

---

## 1. Таверны и генерация слухов (`TavernRumorsDude` и `PortRumors`)

В каждой таверне находится NPC-информатор (`TavernRumorsDude : MonoBehaviour, IGPButton`), который выдает слухи только в обмен на алкоголь.

### 1.1. Активация и проверка напитка (`OnTriggerEnter`)

Чтобы у NPC появилась кнопка «Угостить» (`drinkButton`), предмет в триггере должен удовлетворять строгим условиям:

```csharp
ShipItemBottle bottle = other.GetComponent<ShipItemBottle>();
if (bottle != null && bottle.amount > 1f && bottle.GetCapacity() < 30f &&
    bottle.sold && bottle.health >= bottle.GetCapacity())
{
    currentDrink = bottle;
    drinkButton.SetActive(true);
}
```

| Условие | Описание |
|---|---|
| `bottle.amount > 1f` | В бутылке должно быть более 1 единицы жидкости. |
| `bottle.GetCapacity() < 30f` | Емкость меньше 30 (исключает большие бочки для воды, принимаются только бутылки вина/рома/эля). |
| `bottle.sold == true` | Бутылка должна быть куплена/принадлежать игроку (нельзя угостить неоплаченным товаром из лавки). |
| `bottle.health >= capacity` | Бутылка не должна быть разбита. |

### 1.2. Уровень слуха и качество алкоголя (`ClickDrinkButton`)

При нажатии на кнопку выпивки предмет **уничтожается** (`currentDrink.DestroyItem()`), а качество информации зависит от оставшегося объема/типа напитка:

```csharp
int level = 0;
if (currentDrink.amount < 5f)
    level = 2; // Дорогой/концентрированный напиток -> детализированный слух
```

С вероятностью 50% (или если нет `specialRumors`) вызывается генератор экономических слухов порта: `PortRumors.GenerateRumorText(level)`. Иначе выводится случайная строка из `specialRumors[]`.

> **Ширина переноса текста:** `TavernRumorsDude.Wrap(text, 33)` переносит текст по ширине **33 символа**, тогда как диалоги `QuestDude` переносятся по **45 символам**.

---

## 2. Связь слухов с реальной экономикой (`PortRumors`)

Слухи в таверне — это не статичный текст, а **реальная разведывательная информация** о движении торговых судов (`TraderBoat`, [заметка 77](77-npc-ai-navigation-collision-avoidance-and-waypoint-graphs.md)).

```csharp
public Rumor GenerateRumor(int level)
{
    List<TraderBoat> boats = GetDepartingBoats();
    if (boats.Count <= 0)
        boats = GetIncomingBoats();
    if (boats.Count <= 0)
        return default(Rumor);

    TraderBoat boat = boats[Random.Range(0, boats.Count)];
    Vector2 topGood = FindHighestGoodCount(boat); // .x = ID товара, .y = количество
    ...
    result.mainGood = Mathf.RoundToInt(topGood.x);
    result.mainGoodCount = Mathf.RoundToInt(topGood.y);
    result.origin = boat.GetLastMarket();
    result.destination = boat.GetCurrentDestination();
    result.rumorLevel = level;
    return result;
}
```

### Влияние уровня слуха (`level`) на текст
В методе `GenerateRumorText(level)` количественная оценка груза скрывается или раскрывается в зависимости от `level`:

| Уровень (`level`) | Условие по количеству | Текст в слухе |
|:--:|---|---|
| `0` (дешевый напиток) | Любое количество | `"carrying some {GoodName}."` |
| `2` (дорогой напиток) | `mainGoodCount > 12` | `"carrying a large load of {GoodName}."` |
| `2` (дорогой напиток) | `mainGoodCount <= 4` | `"carrying some {GoodName}."` |
| `2` (дорогой напиток) | `5..12` | `"carrying a sizeable load of {GoodName}."` |

---

## 3. Процедурная анимация NPC и IK (`NPCAnimations` и `NPCPlayerCol`)

В Sailwind NPC не используют скелетную анимацию Unity (`Animator` / `SkinnedMeshRenderer`). Все движения персонажей анимируются процедурно через код в `NPCAnimations`.

```
        [ Transform: head ]
        ──► Slerp(3 Гц) к Camera.main при playerInRange
                 │
      ┌──────────┴──────────┐
      ▼                     ▼
[ breatheParts[] ]    [ lockParts[] ]
  Вращение по z         Фиксация в world space
  (QuadraticInOut)      (стопы/кисти на месте)
```

### 3.1. Процедурное дыхание и покачивание
Каждый кадр таймер `currentTime` колеблется между `0` и `breatheDuration`. Для каждого элемента массива `breatheParts[]` вычисляется угол:

```csharp
float num = QuadraticInOut(currentTime, 0f, breatheAngles[i], breatheDuration);
breatheParts[i].localRotation = baseRots[i];
breatheParts[i].Rotate(Vector3.forward, num, Space.Self);
```

Где `QuadraticInOut` — квадратичная функция плавного входа/выхода (easing).

### 3.2. Фиксация стоп и кистей (`lockParts`)
Чтобы при покачивании корпуса ноги NPC не скользили по земле, компонент сохраняет их исходные мировые координаты и принудительно возвращает обратно:

```csharp
for (int j = 0; j < lockParts.Length; j++)
{
    Vector3 worldPos = transform.TransformPoint(basePositions[j]);
    lockParts[j].localPosition = lockParts[j].parent.InverseTransformPoint(worldPos);
}
```

### 3.3. Динамический взгляд на игрока (`headLookAngle`)
Дочерний триггер `NPCPlayerCol` при входе игрока включает `playerInRange = true`. Если камера игрока находится в пределах конуса обзора `headLookAngle` (по умолчанию **30°**):

```csharp
Quaternion targetLook = Quaternion.LookRotation(transform.position - Camera.main.transform.position, Vector3.up);
if (Quaternion.Angle(transform.rotation, targetLook) > 180f - headLookAngle)
    head.rotation = Quaternion.Slerp(head.rotation, Quaternion.LookRotation(Camera.main.transform.position - head.position, Vector3.up), Time.deltaTime * 3f);
else
    head.localRotation = Quaternion.Slerp(head.localRotation, headInitialRot, Time.deltaTime * 7f);
```

Голова плавно поворачивается к камере игрока со скоростью `3 Гц`, а при выходе из зоны — возвращается в исходное положение со скоростью `7 Гц`.

---

## 4. Таможня, контрабанда и взятки (`QuestItemDetector`)

В портах существует система таможенного контроля (`QuestItemDetector`), проверяющая пронос запрещенных грузов.

### 4.1. Временные окна патрулирования
Коллайдер таможни активен только в заданные часы местного времени:

```csharp
public void Update()
{
    col.enabled = (Sun.sun.localTime >= activeFrom && Sun.sun.localTime <= activeUntil);
}
```

### 4.2. Конфискация и каскад взяток (`OnTriggerEnter`)
Если предмет с `item.GetPrefabIndex() == itemPrefabIndex` (контрабанда) попадает в активный триггер, он немедленно уничтожается (`DestroyItem()`), и запускается каскад списания штрафа/взятки:

```
[Обнаружена контрабанда: DestroyItem()]
                 │
                 ▼
     У игрока >= 5 Gold Lions?
     ├── YES ──► Списать 5 Gold Lions
     │           "You paid a 5 Gold Lions bribe..."
     ▼ NO
 У игрока >= 200 Al'Ankh Lions?
     ├── YES ──► Списать 200 Al'Ankh Lions
     │           "You paid a 200 Al'Ankh Lions bribe..."
     ▼ NO
 [Сброс репутации в Al'Ankh]
 PlayerReputation.ChangeReputation(-99999999, PortRegion.alankh)
 "Failed to bribe the guards. Your reputation in Al'Ankh has been reset."
```

---

## 5. Временные ограничения сдачи квестов (`QuestWaypoint`)

Квестовые точки доставки (`QuestWaypoint`) могут блокировать прием предметов днем или ночью:

```csharp
if (Sun.sun.localTime >= unavailableFrom && Sun.sun.localTime <= unavailableTo)
{
    NotificationUi.instance.ShowNotification("Come back at night!");
    return;
}
```

При успешной сдаче (`questItemIndex` совпадает):
1. Принесенный предмет уничтожается.
2. Запускается таймер `timer = 9f`.
3. Через **9 секунд** на месте точки спавнится награда или следующий предмет из каталога `PrefabsDirectory.instance.directory[spawnedItemPrefab]` с флагом `sold = true` и автоматической регистрацией в системе сохранений (`RegisterToSave()`).

---

## 6. Практические выводы для мододела

1. **Создание кастомных таверн и слухов:** для добавления новых NPC со слухами используйте `TavernRumorsDude` и `PortRumors`. Если напиток имеет `amount < 5f`, NPC выдаст точное количество груза (`level = 2`), иначе только общее упоминание.
2. **Процедурная анимация модовых NPC:** не пытайтесь экспортировать сложные скелетные FBX со `SkinnedMeshRenderer`, игра использует простые Transform-иерархии. Добавьте `NPCAnimations` и укажите части тела в `breatheParts` (вращение) и `lockParts` (фиксация стоп).
3. **Обход таможни:** моды на контрабанду могут использовать «ночное окно», когда `Sun.sun.localTime` выходит за пределы `[activeFrom, activeUntil]` и `QuestItemDetector.col.enabled == false`.
4. **Валюта взяток:** штрафы в `QuestItemDetector` жестко зашиты на **Gold Lions (5)** или **Al'Ankh Lions (200)**. Если у игрока нет этих валют, репутация в регионе Al'Ankh сбрасывается к `-99999999` независимо от того, в каком регионе находится порт.
5. **Задержка спавна награды:** учитывайте 9-секундную задержку (`timer = 9f`) в `QuestWaypoint`: предмет-награда появляется не мгновенно после сдачи квестового предмета.
