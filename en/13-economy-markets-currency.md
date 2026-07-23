# 13. Economy: markets, currencies, prices, trader boats

Complete breakdown of the economic simulation. From decompilation of `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Critical for trading, price balance and mission mods.

## Overall architecture

The economy is four-layered:

```
1. CurrencyMarket        — rates for 4 currencies (worldwide single)
2. IslandMarket (×port)  — goods market at each port (supply/demand/prices)
3. IslandEconomy (×island) — "needs" of the island (how much goods needed)
4. TraderBoat (×N)        — NPC traders, physically carry goods and prices between ports
        + DebugMarketTracker — global tuning hub (static multipliers)
```

Everything ticks on game time: markets every `econCycleDuration`, currency and demand on `Sun.OnNewDay` (new game day).

## 1. Currency exchange (`CurrencyMarket`)

Singleton `CurrencyMarket.instance`. `currentPrices[4]` (`float`) — rates for four currencies (`Currency` enum: `alAnkh=0, emerald=1, aestrin=2, gold=3`).

| Mechanic | Formula |
|----------|---------|
| Daily change (`MarketCycle`, on `Sun.OnNewDay`) | `price[i] += Random(-1,1) * price[i] * dailyChangeFactor` (for all, **except the last** — it's the base) |
| Trade impact on rate | `BuyCurrency`: `price -= amount * changeFactor * 0.001`; `SellCurrency`: `price += …` (large trades move rates) |
| Exchange rate | `GetExchangeRate(sell,buy,fee) = currentPrices[buy] / currentPrices[sell]`, with commission `* (1 - exchangeFee)` |
| Sell price in currency | `GetSellPriceInCurrency` → `Floor(price * raw * (1-fee))` |
| Buy price in currency | `GetBuyPriceInCurrency` → `Ceil(price * raw * (1+fee))` |

`exchangeFee` — exchange commission (SerializeField). Rounding (floor on sell, ceil on buy) ensures "the house always wins".

## 2. Port goods market (`IslandMarket`)

One component per port. Key fields:

| Field | Contents |
|-------|----------|
| `production[]` | Base "production" for each good. |
| `currentSupply[]` | Current supply (starts as copy of `production`). Can be **negative** (deficit). |
| `knownPrices[34]` | `PriceReport` — known prices across all ports (index = `portIndex`, max 34 ports). |
| `supplyPurchaseLimit` | Supply threshold below which a good cannot be purchased. |
| `pricePerGoodMult`, `wealthMult`, `goodsSoftCapOverride` | Local port multipliers. |
| `allowCurrencyConversion` | Whether currency exchange is allowed at this port. |
| `econCycleDuration` | Economic cycle period (default `0.2`). |

### Economic cycle (`EconCycle`)
- Timer: `econTimer -= deltaTime * Sun.sun.timescale * 100f`; resets to `econCycleDuration / DebugMarketTracker.marketSpeedMult`.
- On start — "warmup": 50 consecutive cycles (`DoPreheat`).
- Supply growth: `supply[i] += production[i] * Random(0, 0.42) * InverseLerp(softCap, 0, |supply|) * productionMult`. Closer to soft cap → slower growth (logistic curve).

### Price model (`GetGoodPriceAtSupply`)
1. **Base** = `ShipItem.value` of the item (via `PrefabsDirectory.GoodToItemIndex`).
2. **Special case — food crate** (`ShipItemCrate` with `ShipItemFood` inside): `value = (containedValue * amount + 20) * 2`.
3. **Supply factor** (quadratic):
   - `supply >= 0`: `f = InverseLerp(softCap, 0, supply)`; `factor = 1 - f²`
   - `supply < 0` (deficit): same with opposite sign → price **higher** than base.
4. **Result**: `price = value - value * priceMult * factor`, where `priceMult` = `positivePriceMult` (surplus) or `negativePriceMult` (deficit) from `DebugMarketTracker.instance`.

Surplus **decreases** price, deficit **increases** it. Quadratic dependence gives smooth saturation.

### Player buy/sell
- `GetBuyPrice = price * (1 + spread)`; `GetSellPrice = price(at supply+1) * (1 - spread)`.
- **Spread** (`GetSpread`): `InverseLerp(1000, 30000, rawPrice)` → `Lerp(0.005, 0.0001, …)`. Spread is **0.5%** on cheap goods and drops to **0.01%** on expensive ones (≥30000).
- `PurchaseGood(i)` / `SellGood(i)` change `currentSupply[i]` by ∓1 and update price reports.
- `HasGood(i)`: good available for purchase if `currentSupply[i] >= supplyPurchaseLimit`. **Good 64 only sold at port 33** (hardcoded).

### Price information propagation (`ReceivePriceReports`)
Port accepts foreign `PriceReport[]` only if report is `approved` and its `day` is not older than existing one. This is the "price knowledge" mechanism: player sees current prices only at visited/fresh ports. `UpdateSelfPriceReport` marks report `approved = true` and sets `day = GameState.day`.

### `PriceReport`
`[Serializable]`: `sellPrices[65]`, `buyPrices[65]`, `day`, `approved`. **65** — max goods in a report. (In `SaveLoadManager` price arrays expand from 34 to 65 on old save load — `UpdateReportsArrayLength(…, 34, 65)`.)

## 3. Island needs (`IslandEconomy`)

Component on island (not port). `baseDemand[]` / `currentDemand[]` (`int`) — how many units of each good the island "needs".

- On `Sun.OnNewDay` → `RandomizeDemand()`: for each good with `baseDemand > 0`, demand = `Random.Range(1, baseDemand+1)`.
- `GetDemand(prefabIndex)`: index mapping same as `PrefabsDirectory` (`>200 → -170`).
- `ChangeDemand` and `EconCycle` — **empty stubs** (mechanic unfinished/disabled).
- Persistence: `GetDemandSaveData()` / `LoadDemandData(int[])`.

## 4. Trader boats (`TraderBoat`)

NPC vessels that **physically** transport goods and price reports between ports — the "bloodstream" of the economy connecting isolated markets.

| Field | Contents |
|-------|----------|
| `goodsCapacity` / `weightCapacity` | Capacity (units / weight). |
| `destinations[]` | Destination ports list (`IslandMarket[]`). |
| `carriedGoods[]` | Transported goods (`int[goodsCapacity]`). |
| `carriedPriceReports[34]` | Transported price reports (information propagates with voyage delay). |
| `islandWaitTime` / `debugTravelTime` | Stay time / travel time. |
| `currentDestination` / `currentIslandMarket` / `lastIslandMarket` | Current/last port. |
| `traderBoats` (static List) | Registry of all trader boats. |

- In `Start()` boat picks random destination port and initializes price reports.
- State saved via `TraderBoatData` (`GetData()`/`LoadData()`), including `currentTripTime` and `waitTime` — trade continues correctly after load.
- `ProfitThisTrip` — boat's trip profit (for debugging/balancing).

## Global tuning (`DebugMarketTracker`)

Despite "Debug" name, this is the **live balancing hub** — static multipliers read by the entire economy. `DebugMarketTracker.instance` copies its public `*L` fields to statics in `Update()` (first ~12 seconds after load), then disables itself.

| Static | Impact |
|--------|---------|
| `marketSpeedMult` | Economic cycle speed (timer divisor). |
| `marketProductionMult` | Supply growth multiplier. |
| `goodsAmountSoftCap` | Supply soft cap (price saturation). |
| `positivePriceMult` / `negativePriceMult` | Surplus/deficit price impact strength. |
| `priceDiffFixedMult` / `priceDiffValueMult` | Price difference multipliers. |

Instance fields for mission tuning (reward calculation):

| Field | Purpose |
|-------|---------|
| `missionProfitShareLocal` / `missionProfitShareWorld` | Profit share (local/world). |
| `missionDistanceFee` | Distance fee. |
| `missionFinalMult` | Final reward multiplier. |

> **For balance modder:** changing `DebugMarketTracker.instance.negativePriceMult` / `positivePriceMult` / `goodsAmountSoftCap` on the fly globally retunes price volatility and spread without patches. `*L` fields are "local" values copied to statics.

## Good as object (`Good`)

`Good` — component on `ShipItem` (`[RequireComponent(typeof(ShipItem))]`), describing the commodity entity:

| Field/method | Contents |
|-------------|----------|
| `nativeRegion` (`PortRegion`) | Good's native region. |
| `requiredRepLevel` | Required reputation level for trading (see note 12). |
| `sizeDescription` | Text size description. |
| `missionIndex` | Mission binding (`-1` = not on mission). |
| `RegisterToMission(idx, dueDay)` | Bind good to mission with deadline. |
| `Deliver()` | Delivery: removes tag, calls `Mission.DeliverGood()`, destroys item. |
| `GetCargoWeight()` | Cargo weight: crate = `mass + contained.mass*amount`; bottle = `mass + health`; otherwise `ShipItem.mass`. |

## Practical conclusions for modding

1. **Item price** = `ShipItem.value` × quadratic function of `currentSupply` relative to `goodsAmountSoftCap`, with spread 0.5%→0.01%. For predictable trading — work with port's `currentSupply[]`.
2. **Deficit** (`currentSupply < 0`) — legitimate way to spike price; market allows negative supply.
3. **Prices "propagate"** via `TraderBoat.carriedPriceReports` and `ReceivePriceReports` (only `approved` and fresh by `day`). Price information decays.
4. **Good 64 — port 33** (special good, hardcoded in `HasGood`).
5. **Balance hub** — `DebugMarketTracker.instance` (price/production/mission multipliers). Changes on the fly.
6. All market state (supply, known prices, port demands, trader boats) is saved (see note 11).
