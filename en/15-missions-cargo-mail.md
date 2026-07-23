# 15. Missions: Cargo and Mail Delivery

Analysis of cargo/mail mission system (delivering goods between ports for reward). This is a **separate** system from story "quests" (`Quest`/`Quests`/`QuestDude`). Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## Overall Scheme

```
IslandMissionOffice (at port)
   └─ warehouse[25] — goods warehouse, replenished from market (IslandMarket)
   └─ GenerateMissions(page, world) — generates offers from price arbitrage
            ↓
PlayerMissions.missions[5] — up to 5 active player missions
            ↓
Mission — specific mission (goods, route, price, deadline)
            ↓
Good.RegisterToMission → delivery → DeliverGood → reward + reputation
```

## `Mission` — Mission Object

`[Serializable]`. Key fields:

| Field | Content |
|------|-----------|
| `missionIndex` | Slot in `PlayerMissions.missions` (0..4). |
| `missionName` | **Hardcoded format**: `"{ItemName} to {PortName}"`. |
| `originPort` / `destinationPort` | Origin / destination port. |
| `goodPrefab` | Goods prefab (`GameObject`). |
| `goodCount` | Number of units to deliver. |
| `totalPrice` | Total reward. |
| `insuranceLevel` | Insurance level (affects penalty for non-delivery). |
| `distance` | Distance in "km" = `Vector3.Distance(origin, dest) / 100`. |
| `dueDay` | In-game deadline day. |
| `pricePerKm` | `totalPrice / distance` (for sorting "profitability"). |
| `deliveredGoods` | Number already delivered. |

### Per-Unit Delivery Reward (`GetDeliveryPrice`)
```
perGood = totalPrice / goodCount
daysLate = max(0, GameState.day - dueDay)
penalty  = min(0.5, daysLate * 0.05)        // 5% per overdue day, cap 50%
payment  = round(perGood - perGood * penalty)
```
Paid in destination region currency (`PlayerGold.currency[destRegion]`).

### Delivery Reputation (`GetDeliveryRep`)
```
latePenalty = min(1.0, daysLate * 0.2)       // 20% per day, cap 100%
base = distance / goodCount * 3
rep  = round(base - base * latePenalty)
```
Awarded to origin region **and** destination region (if different).

### Failure Penalty (`EndMission`)
- For each **undelivered** unit: **−100 reputation** to origin region.
- Financial penalty (logged): `goodValue * (1 - insuranceLevel) * undeliveredCount` — insurance reduces losses.

### Deadline Text (`GetDueText`) — hardcoded (EN)
`"in N days"`, `"tomorrow"`, `"today"`, `"yesterday"`, `"N days ago"`.

### Mission Map
- `UseOceanMap`: world map if ports have different `localMap` or it's `none`.
- `UseOceanMapFor`: world map if different `region` or port has `alwaysUseWorldMap`.

## `PlayerMissions` — Player's Active Missions

Static class, `missions[5]` — **maximum 5 simultaneous missions**.

| Method | Action |
|-------|----------|
| `AcceptMission(m)` | Takes empty slot, spawns goods at origin port (`originPort.SpawnGoods`), reduces demand at destination, registers in `IslandMissionOffice`. |
| `AbandonMission(i)` | Returns demand, destroys goods, **−100 reputation**, clears slot. |
| `CompleteMission(i)` | Called automatically when `deliveredGoods >= goodCount`. |
| `GetEmptySlot()` / `MissionsFull()` | Find free slot; on overflow — notification `"Mission log full!"`. |

> Available mission count is additionally limited by reputation: `PlayerReputation.maxMissions = level + 2` (cap 5) and `GetMaxMissionGoods` (number of goods). See note 12.

## Mission Generation (`IslandMissionOffice.GenerateMissions`)

Missions are **not predefined** — they are dynamically created from price differences between ports.

### Warehouse (`goodsInWarehouse[25]`)
- Replenished in `MarketCycle()`: office "buys" goods from market (`market.PurchaseGood`) and places in slot (circular rotation). When slot is reused — previous goods sold back (`market.SellGood`).
- While warehouse is less than half full — prefers goods with `requiredRepLevel == 0` (accessible to newcomers).
- Cycle ticks from `Sun.sun.timescale * 125`, period = `1/marketSpeedMult * 0.1`.

### Generation Algorithm
For each destination port with **known** (`approved`) prices:

1. **Mission type by distance:**
   - `distance < 140` → local (`world: false`), goods limit = `maxGoodsPerMission` (default 3), profit share = `missionProfitShareLocal`.
   - `distance >= 140` → world (`world: true`), goods limit **×2**, profit share = `missionProfitShareWorld`.
   - Goods limit additionally capped by `PlayerReputation.GetMaxMissionGoods(region)`.
2. **For each warehouse good** (counter of available units):
   - `profit = (destSellPrice - originPrice) * count` — arbitrage profit.
   - `reward = (profit * profitShare + distance * missionDistanceFee) * missionFinalMult`.
   - **Filters (mission discarded):** distance fee exceeds profit (`missionDistanceFee*distance > profit`); good's `requiredRepLevel` above player level; distance exceeds `GetMaxDistance(region)`; player already has same mission.
   - `totalPrice = round(reward) / count * count` (multiple of goods count).
3. **Mail mission** (good **51** = mail): generated almost always if distance allows and no duplicate.
   - `count = max(1, 3 - numberOfOtherMissions)`, reward = `round(distance * missionDistanceFee * (count*0.5 + 0.5) * 4)`.
4. **Sorting** by descending `pricePerKm` (most profitable first).
5. **Pruning** (`PruneNoSupplyMissions`): `goodCount` is limited to actual warehouse stock; letters (good 51) are not pruned.
6. **Currency conversion**: final `totalPrice` is converted to destination region currency via `CurrencyMarket.GetSellPriceInCurrency` (no commission), `pricePerKm` recalculated.

All tuning multipliers (`missionProfitShareLocal/World`, `missionDistanceFee`, `missionFinalMult`) come from `DebugMarketTracker.instance` (see note 13).

## Persistence (`SaveMissionData`)

`[Serializable]`, stores: `missionIndex`, `originPort`/`destinationPort` (indices), `goodPrefabIndex`, `goodCount`, `totalPrice`, `insuranceLevel`, `distance`, `deliveredGoods`, `dueDay`. Restored via `Mission(SaveMissionData)` constructor. The `PlayerMissions.missions` array is serialized into save as `savedMissions` (see note 11).

## Practical Takeaways for Modding

1. **Reward = share of arbitrage profit + per-km fee**, all via `DebugMarketTracker.instance`. Changing these 4 multipliers allows global rebalancing of mission profitability.
2. **Overdue delivery** cuts payment by 5%/day (cap 50%) and reputation by 20%/day (cap 100%).
3. **Failure/abandon** = −100 reputation per undelivered unit.
4. **Mail (good 51)** — special mission type, generated separately.
5. **5 mission limit** (array) + reputation restrictions on number of missions/goods/distance.
6. All mission strings (`"{item} to {port}"`, `"Delivered ..."`, `"Mission log full!"`, deadline texts) — **hardcoded in English** (see note 08).
7. Distance "km" = world units / 100.
