# 15. Миссии: доставка грузов и почты

Разбор системы грузовых/почтовых миссий (доставка товаров между портами за награду). Это **отдельная** система от сюжетных «квестов» (`Quest`/`Quests`/`QuestDude`). Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Общая схема

```
IslandMissionOffice (на порту)
   └─ warehouse[25] — склад товаров, пополняется из рынка (IslandMarket)
   └─ GenerateMissions(page, world) — генерит предложения из ценового арбитража
            ↓
PlayerMissions.missions[5] — до 5 активных миссий у игрока
            ↓
Mission — конкретная миссия (товар, маршрут, цена, дедлайн)
            ↓
Good.RegisterToMission → доставка → DeliverGood → награда + репутация
```

## `Mission` — объект миссии

`[Serializable]`. Ключевые поля:

| Поле | Содержание |
|------|-----------|
| `missionIndex` | Слот в `PlayerMissions.missions` (0..4). |
| `missionName` | **Захардкоженный формат**: `"{ItemName} to {PortName}"`. |
| `originPort` / `destinationPort` | Порт отправления / назначения. |
| `goodPrefab` | Префаб товара (`GameObject`). |
| `goodCount` | Сколько единиц доставить. |
| `totalPrice` | Суммарная награда. |
| `insuranceLevel` | Уровень страховки (влияет на штраф за недоставку). |
| `distance` | Дистанция в «км» = `Vector3.Distance(origin, dest) / 100`. |
| `dueDay` | Игровой день-дедлайн. |
| `pricePerKm` | `totalPrice / distance` (для сортировки «выгодности»). |
| `deliveredGoods` | Сколько уже доставлено. |

### Награда за доставку единицы (`GetDeliveryPrice`)
```
perGood = totalPrice / goodCount
daysLate = max(0, GameState.day - dueDay)
penalty  = min(0.5, daysLate * 0.05)        // 5% за день просрочки, кап 50%
payment  = round(perGood - perGood * penalty)
```
Выплачивается в валюте региона назначения (`PlayerGold.currency[destRegion]`).

### Репутация за доставку (`GetDeliveryRep`)
```
latePenalty = min(1.0, daysLate * 0.2)       // 20% за день, кап 100%
base = distance / goodCount * 3
rep  = round(base - base * latePenalty)
```
Даётся региону отправления **и** региону назначения (если они разные).

### Штраф за провал (`EndMission`)
- За каждую **недоставленную** единицу: **−100 репутации** региону отправления.
- Денежный штраф (логируется): `goodValue * (1 - insuranceLevel) * undeliveredCount` — страховка снижает потери.

### Дедлайн-текст (`GetDueText`) — захардкожен (EN)
`"in N days"`, `"tomorrow"`, `"today"`, `"yesterday"`, `"N days ago"`.

### Карта миссии
- `UseOceanMap`: мировая карта, если у портов разный `localMap` или он `none`.
- `UseOceanMapFor`: мировая карта, если разный `region` или у порта `alwaysUseWorldMap`.

## `PlayerMissions` — активные миссии игрока

Статический класс, `missions[5]` — **максимум 5 одновременных миссий**.

| Метод | Действие |
|-------|----------|
| `AcceptMission(m)` | Занимает пустой слот, спавнит товары в порту отправления (`originPort.SpawnGoods`), снижает спрос в назначении, регистрирует в `IslandMissionOffice`. |
| `AbandonMission(i)` | Возвращает спрос, уничтожает товары, **−100 репутации**, очищает слот. |
| `CompleteMission(i)` | Вызывается автоматически при `deliveredGoods >= goodCount`. |
| `GetEmptySlot()` / `MissionsFull()` | Поиск свободного слота; при переполнении — нотификация `"Mission log full!"`. |

> Количество доступных миссий дополнительно ограничено репутацией: `PlayerReputation.maxMissions = level + 2` (кап 5) и `GetMaxMissionGoods` (число товаров). См. заметку 12.

## Генерация миссий (`IslandMissionOffice.GenerateMissions`)

Миссии **не предзаданы** — они динамически создаются из разницы цен между портами.

### Склад (`goodsInWarehouse[25]`)
- Пополняется в `MarketCycle()`: офис «покупает» товар у рынка (`market.PurchaseGood`) и кладёт в слот (ротация по кругу). Когда слот переиспользуется — прежний товар продаётся обратно (`market.SellGood`).
- Пока склад заполнен меньше чем наполовину — предпочитает товары с `requiredRepLevel == 0` (доступные новичкам).
- Цикл тикает от `Sun.sun.timescale * 125`, период = `1/marketSpeedMult * 0.1`.

### Алгоритм генерации
Для каждого порта назначения с **известными** (`approved`) ценами:

1. **Тип миссии по дистанции:**
   - `distance < 140` → локальная (`world: false`), лимит товаров = `maxGoodsPerMission` (по умолч. 3), доля прибыли = `missionProfitShareLocal`.
   - `distance >= 140` → мировая (`world: true`), лимит товаров **×2**, доля прибыли = `missionProfitShareWorld`.
   - Лимит товаров дополнительно капится `PlayerReputation.GetMaxMissionGoods(region)`.
2. **Для каждого товара на складе** (счётчик доступных единиц):
   - `profit = (destSellPrice - originPrice) * count` — арбитражная прибыль.
   - `reward = (profit * profitShare + distance * missionDistanceFee) * missionFinalMult`.
   - **Фильтры (миссия отбрасывается):** плата за дистанцию превышает прибыль (`missionDistanceFee*distance > profit`); `requiredRepLevel` товара выше уровня игрока; дистанция больше `GetMaxDistance(region)`; у игрока уже есть такая же миссия.
   - `totalPrice = round(reward) / count * count` (кратно числу товаров).
3. **Почтовая миссия** (товар **51** = почта): генерируется почти всегда, если позволяет дистанция и нет дубля.
   - `count = max(1, 3 - числоПрочихМиссий)`, награда = `round(distance * missionDistanceFee * (count*0.5 + 0.5) * 4)`.
4. **Сортировка** по убыванию `pricePerKm` (сначала самые выгодные).
5. **Подрезка** (`PruneNoSupplyMissions`): `goodCount` ограничивается реальным остатком на складе; письма (товар 51) не режутся.
6. **Конвертация валюты**: итоговый `totalPrice` переводится в валюту региона назначения через `CurrencyMarket.GetSellPriceInCurrency` (без комиссии), пересчитывается `pricePerKm`.

Все тюнинг-множители (`missionProfitShareLocal/World`, `missionDistanceFee`, `missionFinalMult`) берутся из `DebugMarketTracker.instance` (см. заметку 13).

## Персистентность (`SaveMissionData`)

`[Serializable]`, хранит: `missionIndex`, `originPort`/`destinationPort` (индексы), `goodPrefabIndex`, `goodCount`, `totalPrice`, `insuranceLevel`, `distance`, `deliveredGoods`, `dueDay`. Восстанавливается через конструктор `Mission(SaveMissionData)`. Массив `PlayerMissions.missions` сериализуется в сейв как `savedMissions` (см. заметку 11).

## Практические выводы для мододела

1. **Награда = доля арбитражной прибыли + плата за км**, всё через `DebugMarketTracker.instance`. Меняя эти 4 множителя, можно глобально перенастроить доходность миссий.
2. **Просрочка** режет оплату на 5%/день (кап 50%) и репутацию на 20%/день (кап 100%).
3. **Провал/отмена** = −100 репутации за каждую недоставленную единицу.
4. **Почта (товар 51)** — особый тип миссии, генерируется отдельно.
5. **Лимит 5 миссий** (массив) + репутационные ограничения на число миссий/товаров/дистанцию.
6. Все строки миссий (`"{item} to {port}"`, `"Delivered ..."`, `"Mission log full!"`, дедлайн-тексты) — **захардкожены на английском** (см. заметку 08).
7. Дистанция «км» = мировые единицы / 100.
