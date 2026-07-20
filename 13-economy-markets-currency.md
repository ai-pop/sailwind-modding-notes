# 13. Экономика: рынки, валюты, цены, торговые лодки

Полный разбор экономической симуляции. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Критично для модов на торговлю, баланс цен и миссии.

## Общая архитектура

Экономика четырёхслойная:

```
1. CurrencyMarket        — курсы 4 валют (единый на весь мир)
2. IslandMarket (×порт)  — рынок товаров в каждом порту (спрос/предложение/цены)
3. IslandEconomy (×остров) — «потребности» острова (сколько товара нужно)
4. TraderBoat (×N)        — NPC-торговцы, физически возят товар и цены между портами
        + DebugMarketTracker — глобальный тюнинг-хаб (статические множители)
```

Всё тикает от игрового времени: рынки — каждый `econCycleDuration`, валюта и спрос — по событию `Sun.OnNewDay` (новый игровой день).

## 1. Валютная биржа (`CurrencyMarket`)

Синглтон `CurrencyMarket.instance`. `currentPrices[4]` (`float`) — курсы четырёх валют (`Currency` enum: `alAnkh=0, emerald=1, aestrin=2, gold=3`).

| Механика | Формула |
|----------|---------|
| Ежедневное изменение (`MarketCycle`, на `Sun.OnNewDay`) | `price[i] += Random(-1,1) * price[i] * dailyChangeFactor` (для всех, **кроме последней** — она базовая) |
| Влияние сделки на курс | `BuyCurrency`: `price -= amount * changeFactor * 0.001`; `SellCurrency`: `price += …` (крупные сделки двигают курс) |
| Курс обмена | `GetExchangeRate(sell,buy,fee) = currentPrices[buy] / currentPrices[sell]`, с комиссией `* (1 - exchangeFee)` |
| Цена продажи в валюте | `GetSellPriceInCurrency` → `Floor(price * raw * (1-fee))` |
| Цена покупки в валюте | `GetBuyPriceInCurrency` → `Ceil(price * raw * (1+fee))` |

`exchangeFee` — комиссия обмена (SerializeField). Округление (floor при продаже, ceil при покупке) гарантирует, что «дом всегда выигрывает».

## 2. Рынок товаров порта (`IslandMarket`)

По одному компоненту на порт. Ключевые поля:

| Поле | Содержание |
|------|-----------|
| `production[]` | Базовое «производство» каждого товара. |
| `currentSupply[]` | Текущее предложение (стартует как копия `production`). Может быть **отрицательным** (дефицит). |
| `knownPrices[34]` | `PriceReport` — известные цены по всем портам (индекс = `portIndex`, максимум 34 порта). |
| `supplyPurchaseLimit` | Порог предложения, ниже которого товар нельзя купить. |
| `pricePerGoodMult`, `wealthMult`, `goodsSoftCapOverride` | Локальные множители порта. |
| `allowCurrencyConversion` | Разрешён ли обмен валют в этом порту. |
| `econCycleDuration` | Период экономического цикла (по умолч. `0.2`). |

### Экономический цикл (`EconCycle`)
- Таймер: `econTimer -= deltaTime * Sun.sun.timescale * 100f`; сбрасывается в `econCycleDuration / DebugMarketTracker.marketSpeedMult`.
- При старте — «прогрев»: 50 циклов подряд (`DoPreheat`).
- Рост предложения: `supply[i] += production[i] * Random(0, 0.42) * InverseLerp(softCap, 0, |supply|) * productionMult`. Чем ближе к софт-капу, тем медленнее рост (логистическая кривая).

### Модель цены (`GetGoodPriceAtSupply`)
1. **База** = `ShipItem.value` предмета (через `PrefabsDirectory.GoodToItemIndex`).
2. **Спец-случай — ящик с едой** (`ShipItemCrate` с `ShipItemFood` внутри): `value = (containedValue * amount + 20) * 2`.
3. **Фактор предложения** (квадратичный):
   - `supply >= 0`: `f = InverseLerp(softCap, 0, supply)`; `factor = 1 - f²`
   - `supply < 0` (дефицит): то же с обратным знаком → цена **выше** базовой.
4. **Итог**: `price = value - value * priceMult * factor`, где `priceMult` = `positivePriceMult` (избыток) или `negativePriceMult` (дефицит) из `DebugMarketTracker.instance`.

Избыток товара **снижает** цену, дефицит — **повышает**. Зависимость квадратичная, что даёт плавное насыщение.

### Покупка/продажа игроком
- `GetBuyPrice = price * (1 + spread)`; `GetSellPrice = price(at supply+1) * (1 - spread)`.
- **Спред** (`GetSpread`): `InverseLerp(1000, 30000, rawPrice)` → `Lerp(0.005, 0.0001, …)`. Т.е. спред **0.5%** на дешёвых товарах и падает до **0.01%** на дорогих (≥30000).
- `PurchaseGood(i)` / `SellGood(i)` меняют `currentSupply[i]` на ∓1 и обновляют отчёт цен.
- `HasGood(i)`: товар доступен для покупки, если `currentSupply[i] >= supplyPurchaseLimit`. **Товар 64 продаётся только в порту 33** (захардкожено).

### Распространение ценовой информации (`ReceivePriceReports`)
Порт принимает чужие `PriceReport[]` только если отчёт `approved` и его `day` не старше имеющегося. Это механизм «знания цен»: игрок видит актуальные цены только по посещённым/свежим портам. `UpdateSelfPriceReport` помечает отчёт `approved = true` и ставит `day = GameState.day`.

### `PriceReport`
`[Serializable]`: `sellPrices[65]`, `buyPrices[65]`, `day`, `approved`. **65** — максимум товаров в отчёте. (В `SaveLoadManager` массивы цен расширяются с 34 до 65 при загрузке старых сейвов — `UpdateReportsArrayLength(…, 34, 65)`.)

## 3. Потребности острова (`IslandEconomy`)

Компонент на острове (не на порту). `baseDemand[]` / `currentDemand[]` (`int`) — сколько единиц каждого товара «нужно» острову.

- На `Sun.OnNewDay` → `RandomizeDemand()`: для каждого товара с `baseDemand > 0` запрос = `Random.Range(1, baseDemand+1)`.
- `GetDemand(prefabIndex)`: маппинг индекса тот же, что в `PrefabsDirectory` (`>200 → -170`).
- `ChangeDemand` и `EconCycle` — **пустые заглушки** (механика не доделана/отключена).
- Персистентность: `GetDemandSaveData()` / `LoadDemandData(int[])`.

## 4. Торговые лодки (`TraderBoat`)

NPC-суда, которые **физически** перемещают товары и ценовые отчёты между портами — это «кровеносная система» экономики, связывающая изолированные рынки.

| Поле | Содержание |
|------|-----------|
| `goodsCapacity` / `weightCapacity` | Вместимость (единиц / вес). |
| `destinations[]` | Список портов назначения (`IslandMarket[]`). |
| `carriedGoods[]` | Перевозимые товары (`int[goodsCapacity]`). |
| `carriedPriceReports[34]` | Перевозимые ценовые отчёты (информация распространяется с задержкой рейса). |
| `islandWaitTime` / `debugTravelTime` | Время стоянки / переезда. |
| `currentDestination` / `currentIslandMarket` / `lastIslandMarket` | Текущий/прошлый порт. |
| `traderBoats` (static List) | Реестр всех торговых лодок. |

- В `Start()` лодка выбирает случайный порт назначения и инициализирует ценовые отчёты.
- Состояние сохраняется в сейв через `TraderBoatData` (`GetData()`/`LoadData()`), включая `currentTripTime` и `waitTime` — торговля продолжается корректно после загрузки.
- `ProfitThisTrip` — прибыль лодки за рейс (для отладки/баланса).

## Глобальный тюнинг (`DebugMarketTracker`)

Несмотря на название «Debug», это **боевой балансировочный хаб** — статические множители, которые читает вся экономика. `DebugMarketTracker.instance` копирует свои публичные поля-`L` в статики в `Update()` (первые ~12 секунд после загрузки), затем отключается.

| Статик | Влияние |
|--------|---------|
| `marketSpeedMult` | Скорость экономических циклов (делитель таймера). |
| `marketProductionMult` | Множитель роста предложения. |
| `goodsAmountSoftCap` | Софт-кап предложения (насыщение цены). |
| `positivePriceMult` / `negativePriceMult` | Сила влияния избытка/дефицита на цену. |
| `priceDiffFixedMult` / `priceDiffValueMult` | Множители разницы цен. |

Instance-поля тюнинга миссий (для расчёта награды):

| Поле | Назначение |
|------|-----------|
| `missionProfitShareLocal` / `missionProfitShareWorld` | Доля прибыли (локальная/мировая). |
| `missionDistanceFee` | Плата за дистанцию. |
| `missionFinalMult` | Итоговый множитель награды. |

> **Для моддера баланса:** меняя `DebugMarketTracker.instance.negativePriceMult` / `positivePriceMult` / `goodsAmountSoftCap` на лету, можно глобально перенастроить волатильность и разброс цен без патчей. Поля `*L` — «локальные» значения, которые копируются в статики.

## Товар как объект (`Good`)

`Good` — компонент на `ShipItem` (`[RequireComponent(typeof(ShipItem))]`), описывающий товарную сущность:

| Поле/метод | Содержание |
|------------|-----------|
| `nativeRegion` (`PortRegion`) | Родной регион товара. |
| `requiredRepLevel` | Требуемый уровень репутации для торговли (см. заметку 12). |
| `sizeDescription` | Текстовое описание размера. |
| `missionIndex` | Привязка к миссии (`-1` = вне миссии). |
| `RegisterToMission(idx, dueDay)` | Привязать товар к миссии с дедлайном. |
| `Deliver()` | Доставка: снимает тег, вызывает `Mission.DeliverGood()`, уничтожает предмет. |
| `GetCargoWeight()` | Вес груза: ящик = `mass + contained.mass*amount`; бутылка = `mass + health`; иначе `ShipItem.mass`. |

## Практические выводы для мододела

1. **Цена товара** = `ShipItem.value` × квадратичная функция от `currentSupply` относительно `goodsAmountSoftCap`, со спредом 0.5%→0.01%. Хотите предсказуемую торговлю — работайте с `currentSupply[]` порта.
2. **Дефицит** (`currentSupply < 0`) — легальный способ взвинтить цену; рынок допускает отрицательное предложение.
3. **Цены «распространяются»** через `TraderBoat.carriedPriceReports` и `ReceivePriceReports` (только `approved` и свежее по `day`). Информация о ценах устаревает.
4. **Товар 64 — порт 33** (особый товар, захардкожено в `HasGood`).
5. **Баланс-хаб** — `DebugMarketTracker.instance` (множители цен/производства/миссий). Меняется на лету.
6. Всё состояние рынков (supply, known prices, port demands, trader boats) сохраняется в сейв (см. заметку 11).
