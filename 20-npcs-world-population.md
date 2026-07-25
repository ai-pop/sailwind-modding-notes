# 20. NPC и население мира

Разбор неигровых персонажей и «населения» мира. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## NPC-лодки (`NPCBoatController`)

Фоновые суда, которые ходят по миру по **вейпоинтам** (`NPCBoatWaypoint` / `NPCBoatWaypointManager`).

| Поле | Содержание |
|------|-----------|
| `speed` / `turnSpeed` / `sailSpeed` / `sailResistance` | Параметры движения. |
| `sailAngleControllers` / `sailReefControllers` | NPC **реально управляют парусами** (угол + рифление). |
| `currentTarget` / `currentTargetIndex` | Текущий вейпоинт-цель. |
| `currentDock` / `currentDockIndex` | Вейпоинт-стоянка. |
| `parkedTimer` | Таймер стоянки. |
| `horizon` (`BoatHorizon`) | Оптимизация: лодка активна только рядом с игроком. |

### Логика движения (`FixedUpdate`)
1. **Только рядом с игроком:** если `!horizon.closeToPlayer` — выход (производительность).
2. **Избегание столкновений:** `Physics.OverlapSphere(pos, 15)` — если рядом другая лодка с тегом `Boat`, NPC ждёт (`otherBoatInRange`). Проверка раз в `0.5..1.5 c`.
3. **Навигация:** `AddForceTowards(target)` + `AddRotationTowards(target)` — простое наведение на цель.
4. **Вейпоинты** (`OnTriggerEnter`):
   - если вейпоинт `navigationWaypoint` → берёт следующий (`GetNextDestination`);
   - иначе это **док** → швартуется, крутится к `currentDock.rotation` (lerp 0.005), через `parkedTimer > 1` (× timescale) уходит к следующей цели.

### Персистентность (`NPCBoatData`)
Сохраняется: `currentTarget` (индекс вейпоинта), `currentDock`, `parkedTimer`. При загрузке индексы резолвятся через `NPCBoatWaypointManager.instance.GetWaypointTransform(idx)`. В сейве — массив `npcBoatData` (см. заметку 11).

## Рыбацкие лодки (`NPCFishingBoat`)

Расширение `NPCBoatController` с **распорядком дня** по локальному времени (`Sun.sun.localTime`):

| Время (локальное) | Поведение |
|-------------------|-----------|
| `5:30–9:30` и `13:30–17:30` | `GoFishing()` — идёт к точке лова (`target`). |
| прочее время | `GoHome()` — возвращается на стоянку. |

Позиции хранятся в «реальных» координатах (`ShiftingPosToRealPos`/`RealPosToShiftingPos`) — корректно относительно плавающего начала координат (см. заметку 11). Два «рейса» в день: утренний и послеобеденный.

## Торговые лодки (`TraderBoat`)

Экономические NPC — возят товары и **ценовые отчёты** между портами, связывая рынки. Подробно в заметке 13. Кратко: `carriedGoods[]`, `carriedPriceReports[34]`, маршрут по `destinations[]`, состояние в `TraderBoatData`.

## Портовый служащий (`PortDude`)

NPC у «стола миссий» в порту. Две роли:

### 1. Триггер доставки миссий
`OnTriggerEnter` по коллайдеру с тегом `Good`:
- если миссия товара имеет `destinationPort == этот порт` → `good.Deliver()` (доставка!);
- если товар из миссии, но порт **не тот** → нотификация `"You are at the wrong port!"` (**захардкожено**, EN).

> **Механика доставки:** чтобы сдать миссию, нужно физически принести товар к `PortDude` нужного порта.

### 2. Шлюз UI (миссии / торговля)
`ActivateMissionListUI(openEconomyUI)`:
- `openEconomyUI == false` → открыть список миссий порта (`MissionListUI`).
- `openEconomyUI == true` → если `PlayerReputation.GetRepLevel(region) < 1` → `"Not enough reputation"`; иначе открыть торговый UI (`EconomyUI`).

> **Торговля на рынке требует минимум 1 уровня репутации** в регионе.

## Лавочник (`Shopkeeper`)

NPC розничной торговли отдельными предметами (не рыночными товарами). Методы: `RegisterShop(ShopArea)`, `TryToSellItem(item)`, `TryToBuyItem(item)`, `GetLocalPrice(item)`, `GetLocalPriceString(item)`. Покупка/продажа идёт через `OnTriggerEnter/Exit` (игрок приносит предмет в зону лавки).

## Другие NPC / «жители»

| Класс | Роль |
|-------|------|
| `QuestDude` | Выдаёт сюжетные квесты (`Quest`/`Quests`, отдельная система от грузовых миссий). |
| `TavernRumorsDude` / `PortRumors` / `Rumor` | Слухи в таверне/порту (подсказки, атмосфера). |
| `CargoTransportDude` | Грузчик/перевозчик груза (`CargoCarrier`). |
| `NPCAnimations` | Анимации NPC. |
| `Seagulls` | Атмосферные чайки: частицы + звук, поднимаются/опускаются по высоте (цикл ±600 единиц). |
| `Balloon` / `Windchimes` | Декоративные объекты, реагирующие на ветер. |

## Практические выводы для мододела

1. **NPC-лодки** ходят по вейпоинтам (`NPCBoatWaypointManager`), используют паруса, избегают столкновений в радиусе 15 м, активны только возле игрока (`BoatHorizon.closeToPlayer`).
2. **Доставка миссий** = физически поднести товар к `PortDude` порта назначения (триггер по тегу `Good`).
3. **Торговля на рынке** требует ≥ 1 уровня репутации региона; список миссий доступен без репутации.
4. **Распорядок рыбаков** привязан к локальному солнечному времени (2 окна лова в день).
5. NPC-лодки сохраняют только индексы вейпоинтов + таймер стоянки — лёгкая персистентность.
6. Служебные строки NPC (`"You are at the wrong port!"`, `"Not enough reputation"`) захардкожены (EN) — см. заметку 08.
