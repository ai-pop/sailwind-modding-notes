# 81. Человеческие NPC: поведение, торговля, квесты, процедурные анимации и отсутствие навигации

Полный технический анализ всех классов **человеческих NPC (людей)** в Sailwind v0.38 (`Assembly-CSharp.dll`). В отличие от фоновых судов (`NPCBoatController`, [заметка 77](77-npc-ai-navigation-collision-avoidance-and-waypoint-graphs.md)), люди в мире Sailwind устроены совершенно иначе. Эта заметка раскрывает реальную архитектуру их поведения, систему процедурной анимации, торговые формулы лавочников, автоматы состояний квестодателей и объясняет, **почему у человеческих NPC нет активной навигации в игровом мире**.

---

## 1. Навигация людей-NPC: почему персонажи не ходят

В Sailwind v0.38 **все человеческие NPC являются стационарными (неподвижными) объектами**, закрепленными в точках портов, таверн и магазинов (`PortDude`, `QuestDude`, `Shopkeeper`, `TavernRumorsDude`, `CargoTransportDude`).

В игре **полностью отсутствует система `NavMesh`, `NavMeshAgent` или поиск путей (pathfinding) для людей**. Единственное упоминание `NavMesh` во всей сборке `Assembly-CSharp.dll` находится в коде примера телепортации для VR Oculus (`TeleportTargetHandlerNavMesh`).

### 1.1. Заброшенный код перемещения: корутина `Shopkeeper.WalkTo`

Единственная попытка разработчика сделать ходячего человека-NPC во всей игре содержится в приватной корутине класса `Shopkeeper`:

```csharp
private IEnumerator WalkTo(Vector3 targetLocalPos, bool walkingHome)
{
    Vector3 targetWorldPos = transform.parent.TransformPoint(targetLocalPos);
    atHome = false;
    walking = true;
    float remainingDistance = Vector3.Distance(transform.position, targetWorldPos);
    transform.LookAt(targetWorldPos, Vector3.up);

    float speedModifier = GameState.justStarted ? 99999f : 1f;

    while (remainingDistance > 0f)
    {
        float step = Time.deltaTime * 0.9f * speedModifier;
        if (step > remainingDistance)
            step = remainingDistance;

        transform.Translate(Vector3.forward * step, Space.Self);
        remainingDistance -= step;
        yield return new WaitForEndOfFrame();
    }
    walking = false;
    if (walkingHome)
        atHome = true;
    else
        transform.rotation = shopRotation;
}
```

#### Анализ корутины `WalkTo`
1. **Прямолинейное движение:** персонаж просто поворачивается лицом к цели (`LookAt`) и двигается по прямой со скоростью **0.9 м/с** (`Time.deltaTime * 0.9f`), без огибания препятствий.
2. **Мгновенный старт при загрузке:** если игра только началась (`GameState.justStarted`), скорость множится на `99999f`, чтобы NPC мгновенно телепортировался в целевую точку.
3. **Мертвый код (Dead Code):** в классе `Shopkeeper` **нет метода `Update()`**, и корутина `WalkTo` **ни разу не вызывается** ни из одного метода в `Assembly-CSharp.dll`. Разработчик экспериментировал с перемещением торговца между домом (`homePos`) и лавкой (`shopLocalPos`), но отказался от этой идеи, оставив всех людей-NPC стационарными.

---

## 2. Процедурная анимация и IK (`NPCAnimations` и `NPCPlayerCol`)

Поскольку человеческие NPC не ходят, им не требуется сложный скелетный `Animator` или `SkinnedMeshRenderer`. Все люди в игре анимируются **процедурно через компонент `NPCAnimations`**.

```
        [ Transform: head ]
        ──► Slerp (3 Гц) к Camera.main при playerInRange
                 │
      ┌──────────┴──────────┐
      ▼                     ▼
[ breatheParts[] ]    [ lockParts[] ]
  Вращение по z         Фиксация в world space
  (QuadraticInOut)      (стопы/кисти на месте)
```

### 2.1. Процедурное дыхание и покачивание
Каждый кадр таймер `currentTime` колеблется в цикле от `0` до `breatheDuration`. Для каждого Transform в массиве `breatheParts[]` рассчитывается угол поворота вокруг оси Z:

```csharp
float angle = QuadraticInOut(currentTime, 0f, breatheAngles[i], breatheDuration);
breatheParts[i].localRotation = baseRots[i];
breatheParts[i].Rotate(Vector3.forward, angle, Space.Self);
```

Где `QuadraticInOut` — квадратичная функция плавного ускорения и замедления (easing).

### 2.2. Мировая фиксация стоп и кистей (`lockParts`)
Чтобы при покачивании туловища стопы NPC не скользили по полу таверны или прилавку, массив `lockParts[]` жестко удерживает их в исходных мировых координатах:

```csharp
for (int j = 0; j < lockParts.Length; j++)
{
    Vector3 worldPos = transform.TransformPoint(basePositions[j]);
    lockParts[j].localPosition = lockParts[j].parent.InverseTransformPoint(worldPos);
}
```

### 2.3. Динамический взгляд на игрока (`headLookAngle`)
У каждого человека-NPC есть дочерний триггер с компонентом `NPCPlayerCol`. Когда игрок входит в триггер, он вызывает `playerInRange = true`.
- Если камера игрока находится в переднем секторе обзора `headLookAngle` (по умолчанию **30°**), голова NPC плавно поворачивается к камере со скоростью **3 Гц** (`Time.deltaTime * 3f`).
- Если игрок выходит из сектора или триггера, голова возвращается в исходный поворот (`headInitialRot`) со скоростью **7 Гц** (`Time.deltaTime * 7f`).

---

## 3. Торговец-лавочник (`Shopkeeper`)

`Shopkeeper` отвечает за розничную торговлю в магазинах (`ShopArea`). Он взаимодействует с игроком через коллайдер-триггер: если игрок вносит предмет (`ShipItem`) в зону лавки, торговец проверяет статус `item.sold` и открывает UI сделки (`BuyItemUI`).

### 3.1. Полная таблица ценообразования (`GetPrice`)

Метод `GetPrice(ShipItem item)` рассчитывает стоимость товара в местной валюте региона (`parentRegion.portRegion`):

| Ситуация / Тип предмета | Условия | Итоговая формула цены |
|---|---|---|
| **Игрок продает товар** (`item.sold == true`) |
| `ShipItemLanternFuel` | Есть бутылочка для масла и она не полная | **12** |
| `ShipItemBottle` | Емкость `≥ 5` и `< 30` (пустая/частичная бутылка) | **2** |
| `ShipItemBottle` | Емкость `≥ 30` и `amount == 9f` (полный бочонок рома/вина) | **25** |
| `ShipItemCrate` | Полный ящик (`amount == maxAmount`) | **80%** от `item.value` (`value * 0.8f`) |
| `ShipItemCrate` | Пустой ящик (`amount == 0`) | **20** (или **24**, если внутри была копченая еда) |
| `ShipItemSalt` / `ShipItemOakum` | Соль или пакля | `value * (amount / maxAmount) * 0.8f` |
| Обычный предмет | Любой розничный предмет | **50%** от `item.value` (`value * 0.5f`) |
| **Игрок покупает товар** (`item.sold == false`) |
| Обычный розничный предмет | В лавке на прилавке | `value * priceMult - (value * priceMult * retailDiscounts[region])` |
| Приготовляемая еда (`CookableFood`) | В лавке (сырая/копченая) | База 75% от `value` (×1.2, если `amount ≥ 1`), умножается на `priceMult` с учетом скидки за репутацию |
| Оптовые товары (`IsBulk()`) | Бочки, ящики с рыночным товаром | `GetBulkBuyPrice()`: базовая цена рынка + наценка за спрос (`1f + demand * 0.25f`) минус оптовая скидка репутации |

> **Коэффициент прилавка (`priceMult`):** множитель розничной цены берется из компонента `ShopItemSpawner.priceMult`, установленного на родительском объекте прилавка.

---

## 4. Квестодатель сюжетных заданий (`QuestDude`)

`QuestDude` управляет сюжетными квестами (отличными от грузовых миссий) на основе автомата состояний в `Quests.instance.currentQuests[questIndex]`.

```csharp
// Вход в триггер (OnTriggerEnter)
if (other.CompareTag("Player"))
{
    int status = Quests.instance.currentQuests[quest.questIndex];
    if (status == 0)      ShowDialog(0);  // Начало диалога
    else if (status == -1) ShowDialog(-1); // Квест в процессе
    else                   ShowDialog(-5); // Квест завершен
}
```

### 4.1. Форматирование диалога и ширина переноса
Текст реплик NPC (`questLines[]`) автоматически переносится по словам методом `Wrap(string v, int size)` с шириной строки **45 символов**.

### 4.2. Выдача и сдача квеста
- **Принятие:** при проклике диалога до конца состояние меняется на `-1`. Если в квесте задан `acceptPrefabIndex != 0`, рядом с кнопкой диалога спавнится стартовый предмет квеста с флагом `sold = true`.
- **Сдача:** когда игрок вносит в триггер `QuestDude` предмет с `questItemIndex == quest.deliveredQuestItemIndex`, NPC показывает реплику сдачи (`ShowDialog(-3)`). При подтверждении предмет уничтожается (`DestroyItem()`), игрок получает награду **в золотых львах** (`PlayerGold.currency[3] += quest.goldReward`), а статус квеста переходит в `-5` (завершен).

---

## 5. Портовый служащий и грузчик (`PortDude` и `CargoTransportDude`)

### 5.1. `PortDude` (Служащий порта)
Выполняет две ключевые функции у стола миссий:
1. **Приемка грузов (`OnTriggerEnter`):** проверяет коллайдеры с тегом `"Good"`. Если товар принадлежит миссии, где `destinationPort == port`, вызывается `good.Deliver()` (сдача груза). Если порт не совпадает, выводится уведомление: `"You are at the wrong port!"`.
2. **Открытие UI порта (`ActivateMissionListUI`):**
   - При `openEconomyUI == false` открывает список миссий порта (`MissionListUI`).
   - При `openEconomyUI == true` проверяет репутацию игрока (`PlayerReputation.GetRepLevel(port.region) >= 1`). Если уровень репутации меньше 1, доступ к рынку блокируется сообщением `"Not enough reputation"`.

### 5.2. `CargoTransportDude` (Грузчик)
Стационарный NPC, отвечающий за аренду грузовых тележек/носильщиков. При приближении игрока его триггер вызывает `CargoCarrierUI.instance.ShowUI(transform, carrierIndex)`.

---

## 6. Информатор в таверне (`TavernRumorsDude`)

`TavernRumorsDude` генерирует слухи о торговых кораблях и ценах ([заметка 78](78-dialogues-rumors-bribery-and-contraband-customs.md)).

- **Ориентация диалога (Billboarding):** текстовый пузырь таверны в `Update()` поворачивается к зеркалу камеры игрока: `speechUI.transform.rotation = Refs.observerMirror.transform.rotation`.
- **Проверка напитка:** требует купленную бутылку с алкоголем (`amount > 1f && capacity < 30f && sold && health >= capacity`).
- **Ширина переноса текста:** в отличие от `QuestDude` (45 символов), текст слухов `TavernRumorsDude` переносится по ширине **33 символа** (`Wrap(str, 33)`).

---

## 7. Практические выводы для мододела

1. **Создание ходячих NPC с нуля:** не ищите в игре встроенный NavMesh или готовый AI ходьбы для людей — их нет. Если вашему моду нужны ходячие горожане или патрульные, вам придется:
   - Создать собственный NavMesh или систему вейпоинтов.
   - Реализовать перемещение (или расконсервировать корутину `Shopkeeper.WalkTo` через Harmony/рефлексию, подключив её к таймеру в `Update`).
2. **Анимация кастомных NPC:** не используйте тяжелые скелетные FBX с `SkinnedMeshRenderer`. Разбейте модель NPC на отдельные Transform-части (голова, торс, плечи, стопы), добавьте компонент `NPCAnimations` и назначьте `breatheParts` (для анимации дыхания) и `lockParts` (для удержания стоп на полу).
3. **Создание новых торговцев:** чтобы создать кастомный магазин, добавьте на сцену `Shopkeeper` и свяжите его с `Region`, `IslandEconomy` и зоной `ShopArea`. Чтобы переопределить цены или сделать скупку определенного лута дороже, патчьте метод `Shopkeeper.GetPrice(ShipItem)`.
4. **Квесты без написания кода:** новые диалоговые квесты можно добавлять просто путем заполнения сериализуемых данных `Quest` и регистрации их в `Quests.instance.currentQuests`, так как `QuestDude` универсально обрабатывает любые диалоги по состояниям `0 -> -1 -> -5`.
5. **Ширина диалоговых окон:** при переводе или модификации текста учитывайте жестко зашитую ширину переноса строк: **45 символов** для `QuestDude` и **33 символа** для `TavernRumorsDude`.
