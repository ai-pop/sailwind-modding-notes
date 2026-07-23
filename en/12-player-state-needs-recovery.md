# 12. Player state: needs, currency, reputation, blackout

Breakdown of the player survival and progression subsystem. From decompilation of `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Useful for survival, economy, balance and cheat-panel mods.

## Needs (`PlayerNeeds`)

All values — **static `float` in range `0..100`**, tick in `LateUpdate()` at rate proportional to `Time.deltaTime * Sun.sun.timescale` (i.e. depends on time acceleration).

| Field (static) | Start | Drain rate | What happens at 0 |
|---------------|:-----:|-----------|-------------------|
| `food` | 95 | `3/s` (+ movement drain) | `foodDebt` starts draining |
| `foodDebt` | 100 | `5/s` when `food==0` | blackout (`RecoveryReason.food`) |
| `water` | 95 | `4/s` (+ running/swimming) | blackout (`RecoveryReason.water`) |
| `sleep` | 75 | `5/s` awake; `+15/s * (alcohol/100)` additionally | `sleepDebt` starts draining |
| `sleepDebt` | 100 | grows on sleep deprivation; restored during sleep | `Sleep.instance.FallAsleep()` (forced sleep) |
| `vitamins` | 100 | `0.2/s` | blackout — scurvy (`RecoveryReason.vitamins`) |
| `protein` | 100 | `0.2/s` | blackout — malnutrition (`RecoveryReason.protein`) |
| `alcohol` | 0 | `12/s` (metabolized) | — (drunkenness, see below) |

### "Debt" mechanic
Food and sleep have a two-tier system: the main stat drains first, and when it reaches 0 — "debt" (`foodDebt` / `sleepDebt`) starts draining. Blackout/forced sleep happens only when **debt** hits zero/negative. While the main stat is empty but debt still exists, player lives "on credit". During sleep, `sleepDebt` refills first (then `sleep`, 5x slower: `num *= 0.2f`).

### Movement drain (`DrainEnergyFromMovement`)
- Running (`Refs.ovrController.IsRunning()`): drains `runningWaterCost`.
- Swimming + running: `swimmingFoodCost` + `swimmingWaterCost`.
- Drain scaled by squared horizontal speed (`outLastMovement.sqrMagnitude`, `y=0`).
- When `food < 0` movement drain goes directly into `foodDebt`.

### Sleep
- During sleep: `sleep += 8/s * timescale`; in tavern (`GameState.sleepingInTavern`) — **4x faster**.
- Tavern sleep additionally **fully restores** `food`, `foodDebt`, `water`, `protein`, `vitamins` to 100.

### Guards and overrides (important for mods)
- **NaN guard:** `if (food != food) food = 100f;` — self-comparison catches NaN and resets to 100. Same for `water`, `sleep`. If your mod writes NaN — game silently resets the need.
- **`godMode`** (instance field): completely disables the entire needs tick.
- **`applyOverride` + `overrideFood/Water/Sleep/Vitamins/Protein`**: set `override*` values and flip `applyOverride = true` → next frame values are forced. This is the standard way to "cheat" needs.
- Needs tick **pauses** when: `godMode`, `!GameState.playing`, `GameState.recovering`, player at shipyard (`GameState.currentShipyard`), trade UI open (`EconomyUI.instance.uiActive` and `Debugger.buildDebugModeOn` not enabled).

## Drunkenness (`PlayerAlcohol`)

`PlayerNeeds.alcohol` (0..100) doesn't cause blackout, but affects visuals and sleep:

- **Post-processing:** `bloom.threshold` lerp `1.05 → 0`, `bloom.radius` `2.5 → 5`, `postExposure` `0.66 → -1` as `alcohol/100` grows. Visually — bloom and overexposure when drunk.
- **Sleep:** alcohol accelerates `sleep` drop by `15/s * (alcohol/100)` (see above).

## Tobacco (`PlayerTobacco`)

Four varieties, each a separate float "charge" (0..100), instance fields on `PlayerTobacco.instance`:

| Field | Type (tobaccoType) | Effect when smoking (`Smoke`) | Withdrawal in `Update` |
|-------|:--:|-------------------------------|----------------------|
| `white` | 1 | `sleep +0.66/s` | `sleep -0.33 * (white/...)` |
| `green` | 2 | `sleep -0.11/s` | `sleep -0.11` |
| `black` | 3 | `sleep +1.2/s` | `sleep -0.6`, dampens `bloom.intensity` `0.66 → 0` |
| `brown` | 4 | `sleep +0.8/s` | `sleep -0.4` |

- Each variety's charge grows `2/s` while smoking and drops `0.5/s` spontaneously.
- `saturationOverride = (-white + green)/100 + 1` — affects image saturation (static, read by post-processing).

## Currency (`PlayerGold`)

Game has **4 currencies**, stored in `PlayerGold.currency` (`int[4]`). Legacy field `gold` (int) is no longer used directly (converted on old save load via `ConvertOldGold`).

| Index | Name | Symbol |
|:--:|------|:--:|
| 0 | Al'Ankh Lions | `A` |
| 1 | Emerald Dragons | `E` |
| 2 | Aestrin Crowns | `C` |
| 3 | Gold Lions | `G` |

Names/symbols are hardcoded in `GetCurrencyName(int, bool)` / `GetCurrencySymbol(int)` (switch expressions). `withLineBreak` inserts `\n` between the name and "Lions/Dragons/Crowns".

> **For translator/mod:** currency names are hardcoded strings in code (see note 08 about hardcoded UI strings).

## Reputation (`PlayerReputation`)

Static class. Reputation tracked **per 4 regions** (`PortRegion` enum → array index).

| Member | Contents |
|--------|----------|
| `reputation[4]` | Reputation points per region (capped `0..999999`). |
| `retailDiscounts[4]` | Retail discount = `0.05 * level` (5% per level). |
| `bulkDiscounts[4]` | Always `0` (no bulk discount). |
| `maxMissions[4]` | `level + 2`, cap `5`. |

### Levels (0..10) and point thresholds (`GetRequiredRep`)

| Level | Points needed | Level | Points needed |
|:--:|--:|:--:|--:|
| 1 | 300 | 6 | 36 000 |
| 2 | 1 200 | 7 | 72 000 |
| 3 | 3 600 | 8 | 144 000 |
| 4 | 9 000 | 9 | 288 000 |
| 5 | 18 000 | 10 | 576 000 |

### Level-derived values

- **Max goods in mission** (`GetMaxMissionGoods`): lvl0→3, 1→4, 2→5, 3→6, 4→7, 5→8, 6+→`999999`.
- **Max mission distance** (`GetMaxDistance`, world units): lvl0→`96.45`, 1→`514.4`, 2→`964.5`, 3→`1286`, 4→`1446.75`, 5→`1768.25`, 6→`1929`, 7→`2250.5`, 8+→`999999`.
- Access methods: `GetRep(region)`, `GetRepLevel(region)`, `GetLevel(rep)`, `ChangeReputation(delta, region)`, `GetHighestLevel()`.
- Persistence: `GetSaveData()` / `LoadReputation(int[])` (see note 11).

## Blackout and recovery (`Recovery`, `RecoveryReason`)

When a need hits zero (or player drowns at world border), `Recovery.RecoverPlayer(reason)` is called.

### `RecoveryReason` (enum)
`requested`, `water`, `food`, `worldBorder`, `protein`, `vitamins`.

Blackout texts (hardcoded in `Recovery.DoRecoverPlayer`):
- `water` → "You passed out from thirst."
- `food` → "You passed out from hunger."
- `vitamins` → "You passed out from scurvy."
- `protein` → "You passed out from malnutrition."

### Recovery sequence
1. `GameState.recovering = true`, player falls asleep (`Sleep.instance.FallAsleep()`), 3s pause.
2. Reason text shown, then **all needs set to 100** (`food/foodDebt/water/sleep/sleepDebt/vitamins/protein`).
3. Player teleported to **last visited port** (`RecoveryPort`, linked to `GameState.lastVisitedPort`; if none — nearest port).
4. Player's boat (`GameState.lastOwnedBoat`): moorings removed, anchor reset, `waterLevel`/damage zeroed, boat placed at port position and moored (bow + stern lines). If sunk — `GetSinkRotation()` temporarily applied for correct local item unpacking.
5. If `lastOwnedBoat == null` — nearest **purchased** boat found (`SaveableObject` with `BoatRefs` and `extraSetting == true`).
6. **Penalty:** percentage deducted from each currency (`GetRecoveryPercentageCost`).
7. `GameState.recovering = false`, control restored.

### Penalties and losses (depend on distance to recovery port)

| Distance to port | Money percentage (`GetRecoveryPercentageCost`) | Fixed cost (`GetRecoveryCost`) |
|------------------|:--:|:--:|
| `< 5000` | 5% | 100 |
| `< 30000` | 10% | 250 |
| `< 70000` | 15% | 500 |
| `>= 70000` | 20% | 2000 |

- **Cargo loss chance** (`GetCargoLossChance`): `InverseLerp(5000, 50000, distance) * 100`% — from 0% at close range to 100% at `>= 50000`.
- Money deduction logged in `DayLogs` as `LogRecovery(-amount)`.

## Practical conclusions for modding

1. **Immortality/god mode:** `PlayerNeeds.instance.godMode = true` — standard flag, disables entire tick. Or hold `applyOverride` + `override* = 100`.
2. **Don't write NaN** into needs — game silently resets to 100 (NaN guard).
3. **Currency** — `PlayerGold.currency[0..3]` (int). Names/symbols hardcoded.
4. **Reputation** — per region, `ChangeReputation(delta, region)`; levels give discounts (5%/level) and mission slots.
5. **Blackout** teleports player and boat to last port and takes a percentage of money — consider this if your mod invokes `Recovery.RecoverPlayer` or changes needs.
6. All state (`food/water/sleep/.../alcohol/tobacco*`) is saved automatically (see note 11).
