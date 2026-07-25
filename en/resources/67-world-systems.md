# 67. World Systems: NPCs, Dialogues, Economy, Storms & More

> Deep-dive into all non-item game mechanics: NPCs, quests, tavern rumours,
> trader boats, reputation, recovery, storms, weather totems, ambient systems.
> Extracted from Assembly-CSharp.dll (Sailwind v0.38).

---

## 1. NPC Animation System

### NPCAnimations

Two-part animation:
- **Breathe parts** (`breatheParts[]`): oscillate along Z-axis with `QuadraticInOut` easing
- **Lock parts** (`lockParts[]`): world-space position locked, IK-style
- **Head tracking** (`head`): Slerp-follows camera when player is in trigger range, within `headLookAngle=30°` of NPC's forward direction

```csharp
// Breathing: triangle-wave period
currentTime += (goingDown ? -dt : +dt)
if currentTime >= breatheDuration → reverse
if currentTime < 0 → reverse
angle = QuadraticInOut(currentTime, 0, breatheAngles[i], breatheDuration)
```

### NPCPlayerCol

Minimal bridge: `RegisterAnimations(anims)`. Tag-based trigger detection.

---

## 2. Port Dude — Mission & Economy Gateway

### PortDude

Trigger-based interaction:
- **Player enters** → `ActivateMissionListUI(openEconomyUI=false)` → opens mission list
- **Alt-activate (right-click)** → if rep >= 1 → opens `EconomyUI`
- **Good tagged "Good" enters** → checks `GetAssignedMission().destinationPort == this.port`
  - Match → `Deliver()` (destroy good, complete mission)
  - Wrong port → "You are at the wrong port!"

---

## 3. Quest System

### Quest (data class)

| Field | Type | Description |
|-------|------|-------------|
| `questIndex` | int | Unique quest ID |
| `goldReward` | int | Gold reward on completion |
| `questLines[]` | string[] | NPC dialog lines |
| `playerResponses[]` | string[] | Player button text per line |
| `playerAltResponses[]` | string[] | Alt response text |
| `acceptPrefabIndex` | int | Prefab spawned when quest accepted (0=none) |
| `deliveredQuestItemIndex` | int | QuestItem index to deliver |
| `inProgressLine` | string | NPC line while quest active |
| `completionLine` | string | NPC line on delivery |
| `afterCompletedLine` | string | NPC line after completion |

### Quests (singleton)

```csharp
int[] currentQuests;  // 0=not started, -1=in progress, -5=completed
```

### QuestDude

Flow:
1. Player enters trigger → shows dialog UI
2. `currentQuests[quest.questIndex] == 0` → dialog line 0 (quest offer)
3. Click → advance through `questLines[]`
4. Last line reached → `currentQuests[quest.questIndex] = -1` → spawn `acceptPrefabIndex`
5. Visit again with `-1` → show `inProgressLine`
6. Bring `QuestItem` with matching `questItemIndex` → trigger `completionLine`
7. Click → `CompleteQuest()`: destroy item, add gold, set `-5`
8. After completion → `afterCompletedLine` + "(bye)"

---

## 4. Tavern System

### Tavern

- `rawPrice` base price, discounted by `PlayerReputation.retailDiscounts[region]`
- Currency conversion via `CurrencyMarket`
- `ClickSleepButton()`: deduct currency, set `GameState.sleepingInTavern=true`, call `Sleep.instance.FallAsleep()`
- Tavern sleep: timeskip mode, full stat restore

### TavernRumorsDude

- Accepts `ShipItemBottle` with `amount > 1` (alcohol) AND `capacity < 30` (bottle, not barrel) AND full
- `ClickDrinkButton()`: destroys drink, generates rumor
- **Rumor generation:**
  - 50% chance: `specialRumors[random]` (unique lines)
  - 50% chance: `PortRumors.GenerateRumorText(level)`
  - `level=2` if drink `amount < 5` (strong alcohol), else `level=0`

---

## 5. Port Rumors & Trader Boats

### PortRumors.GenerateRumor(level)

1. Get `GetDepartingBoats()` (lastMarket == this port)
2. If none → `GetIncomingBoats()` (currentDestination == this port)
3. If still none → return empty rumor → "It's been quiet..."
4. `FindHighestGoodCount(boat)` → most-carried good + count
5. Build text from template pools:
   - "I've heard of a boat coming here from {origin}" (5 variants)
   - "A boat departed here towards {destination}" (5 variants)
   - "carrying {size} {goodName}" (size="some"/"a sizeable load"/"a large load")

### TraderBoat

**State machine:**
```
WAITING_AT_ISLAND (waitTime > 0)
  → EnterIsland(): SellGoods(), ReceivePriceReports(), wait
  → LeaveIsland(): UpdatePriceReports(), FindNextDestination(), BuyGoods(), depart
  → TRAVELING (waitTime counts down)
  → arrives at currentDestination → cycle repeats
```

**Economy AI (`FindNextDestination`):**
- For each destination: calculate profit for all goods in `currentSupply`
- Consider `carriedPriceReports[dest].sellPrices[j]` vs `currentIslandMarket.GetBuyPrice(j)`
- Adjust profit by `wealthMult` per extra unit
- Sort by profit, fill `goodsCapacity` slots, respect `weightCapacity`
- Pick BEST destination (max profit — note: `Random.Range(0, 0)` = always index 0 = best)

---

## 6. Reputation System

### PlayerReputation

4 regions: Al'Ankh(0), Emerald(1), Medi(2), none(3)

| Level | Required Rep | Max Distance | Retail Discount | Max Missions |
|:-----:|:-----------:|:------------:|:---------------:|:------------:|
| 0 | 0 | 96 | 0% | 2 |
| 1 | 300 | 514 | 5% | 3 |
| 2 | 1,200 | 965 | 10% | 4 |
| 3 | 3,600 | 1,286 | 15% | 5 |
| 4 | 9,000 | 1,447 | 20% | 5 |
| 5 | 18,000 | 1,768 | 25% | 5 |
| 6 | 36,000 | 1,929 | 30% | 5 |
| 7 | 72,000 | 2,251 | 35% | 5 |
| 8 | 144,000 | ∞ | 40% | 5 |
| 9 | 288,000 | ∞ | 45% | 5 |
| 10 | 576,000 | ∞ | 50% | 5 |

---

## 7. Player Needs

| Stat | Drain Rate | Pass-out |
|------|:----------:|----------|
| `food` | `3 × timescale` /s | `RecoveryReason.food` |
| `water` | `4 × timescale` /s | `RecoveryReason.water` |
| `sleep` | `5 × timescale` /s awake | `FallAsleep()` |
| `vitamins` | `0.2 × timescale` /s | `RecoveryReason.vitamins` |
| `protein` | `0.2 × timescale` /s | `RecoveryReason.protein` |
| `alcohol` | `12 × timescale` /s decay | — |

- Running: extra water drain; swimming: extra food+water drain
- Alcohol accelerates sleep drain: `+15 × (alcohol/100)`

---

## 8. Recovery System

### Recovery.RecoverPlayer(reason)

1. `GameState.recovering = true`
2. Fade to black (3s), call `Sleep.instance.FallAsleep()`
3. Show reason text ("passed out from thirst/hunger/scurvy/malnutrition")
4. Reset ALL needs to 100
5. Teleport to last visited port (or closest)
6. `RecoverBoat()`: unmoor, clear damage, teleport boat, re-moor
7. Deduct `percentageCost`% of gold (5–20% based on distance)
8. Wake up, fade in

### Cargo Loss

`GetCargoLossChance()`: 0–100% based on distance (5km → 0%, 50km → 100%). Lost cargo: `parentObject = -3` → `DestroyItem()`.

---

## 9. Economy

### IslandEconomy

- `baseDemand[]` per good
- `currentDemand[]` randomized daily from `Random.Range(1, baseDemand[i]+1)`

### IslandMarket

Linked to `IslandEconomy` and `CurrencyMarket`. Manages `currentSupply[]`, buy/sell prices, `knownPrices[]`.

### CurrencyMarket

Currency exchange between Al'Ankh Lions, Emerald Dragons, Aestrin Crowns, Gold Lions.

**Currency names:**
| Index | Name | Symbol |
|:-----:|------|:------:|
| 0 | Al'Ankh Lions | A |
| 1 | Emerald Dragons | E |
| 2 | Aestrin Crowns | C |
| 3 | Gold Lions | G |

---

## 10. NPC Boats

### NPCBoatController

**Navigation:** `AddForceTowards(target)` + `AddRotationTowards(target)`
```csharp
force = direction * (speed + |wind| * speed * 0.05) * rigidbody.mass
torque = Vector3.up * turnSpeed * rigidbody.mass * 20 * sign(angle)
```

**Sail control:** `sailAngleControllers[]` and `sailReefControllers[]` — auto-trim based on wind resistance.

**Docking:** OnTriggerEnter waypoint → if `navigationWaypoint` → next waypoint, else park. Parked timer → next destination.

**Collision avoidance:** `Physics.OverlapSphere(15m)` — if another boat nearby, pause movement.

### NPCBoatWaypointManager

Singleton managing `waypoints[]`. Used by both trader boats and fishing boats.

### NPCFishingBoat

Time-based behavior:
- 5:30–9:30 OR 13:30–17:30 → `GoFishing()` (target = fishing spot)
- Otherwise → `GoHome()` (target = parked position)

---

## 11. Weather & Storms

### WanderingStorm

- Moves with `Wind.currentWind * 12 m/s`
- `totemMult` + `WeatherStorms.totemAttraction` → drawn toward player if totem active
- Particles activate within range, deactivate beyond 12km
- Teleport near player if beyond 44km

### WeatherStorms

Manages `stormCount` per region, `totemAttraction` from weather totems.

### ShroomTrigger

Activates particle emission at night (19:00–7:00) when player inside trigger.

### Rainbow

Visual rainbow effect (no gameplay impact).

---

## 12. Ambient Systems

### UnderwaterMusic

- Only when `distanceToLand > 240m` AND NOT on boat
- Fades in based on camera depth (`-1.2m` to `-8m`)
- Day clip (5:00–19:00) vs night clip, with `volumeMultDay=0.02`, `volumeMultNight=1.0`

### Seagulls

- Particle system + audio
- Ascends/descends between `initialHeight` and `topHeight=initialHeight+600`
- State toggle every 2–5 seconds
- Auto-ascend after 18:00

### Stars

- Alpha = `nightLerp³` — fade in smoothly at night
- Constant render queue 2800

---

## 13. Special Locations

### KiciaAltar

- Accepts any `sold` ShipItem
- 33.3s fire effect with light + audio ramp
- 2s purr sound, sacrificial particles at item position
- Destroys the sacrificed item
- 3.6s cooldown between sacrifices

### OnsenExitTrigger

- On player enter: `cols.SetActive(false)` — disables collider barrier

### ElixirTribalDisco

- Rotates at 6°/sec
- Light intensity and audio ramps over 90 seconds (cubic ease)
- Then self-destroys

### Balloon

Simple buoyant force: `AddForceAtPosition(Vector3.up * buoyantForce, position)`

---

## 14. Multiplayer (Oculus Platform)

### SocialPlatformManager

Full Oculus Platform integration with:
- Room creation/joining via friends
- P2P avatar packet streaming
- VOIP via `VoipManager`

### RemotePlayer

Tracks remote user state: avatar, position/rotation (received+prior for interpolation), VOIP source, connection states.

### VoipManager

Oculus VOIP with connect/disconnect callbacks, auto-reconnect.

---

## 15. World Item Spawning

### WorldItemSpawner

- Respawns `itemPrefab` after cooldown
- Cooldown: `Random.Range(0.75×, 1.25× respawnTime)`
- Only spawns when player < 100m away
- Freezes spawned item via `debugForceKinematic = true`

### ShopItemSpawner / ShopItemSpawnerEditorPopulator

Manages items on shop shelves (editor + runtime placement).

---

## 16. Player Misc

### PlayerAlcohol

Bloom + color grading based on `PlayerNeeds.alcohol/100`:
- Bloom threshold: `lerp(1.05, 0, alcohol)`
- Bloom radius: `lerp(2.5, 5, alcohol)`
- Post-exposure: `lerp(0.66, -1, alcohol)`

### PlayerUnstucker

Presumably unsticks player from geometry (not analyzed).

### PlayerMask

Likely VR mask handling.

---

*All data extracted from Assembly-CSharp.dll decompilation (Sailwind v0.38).*
