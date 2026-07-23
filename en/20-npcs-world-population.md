# 20. NPCs and World Population

Analysis of non-player characters and world "population". Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSPy.

## NPC Boats (`NPCBoatController`)

Background vessels that traverse the world via **waypoints** (`NPCBoatWaypoint` / `NPCBoatWaypointManager`).

| Field | Content |
|------|-----------|
| `speed` / `turnSpeed` / `sailSpeed` / `sailResistance` | Movement parameters. |
| `sailAngleControllers` / `sailReefControllers` | NPCs **actually control sails** (angle + reefing). |
| `currentTarget` / `currentTargetIndex` | Current waypoint target. |
| `currentDock` / `currentDockIndex` | Waypoint dock. |
| `parkedTimer` | Dock timer. |
| `horizon` (`BoatHorizon`) | Optimization: boat only active near player. |

### Movement Logic (`FixedUpdate`)
1. **Only near player:** if `!horizon.closeToPlayer` — exit (performance).
2. **Collision avoidance:** `Physics.OverlapSphere(pos, 15)` — if another boat with tag `Boat` nearby, NPC waits (`otherBoatInRange`). Check every `0.5..1.5 s`.
3. **Navigation:** `AddForceTowards(target)` + `AddRotationTowards(target)` — simple heading to target.
4. **Waypoints** (`OnTriggerEnter`):
   - if waypoint `navigationWaypoint` → takes next (`GetNextDestination`);
   - otherwise it's a **dock** → moors, rotates toward `currentDock.rotation` (lerp 0.005), after `parkedTimer > 1` (× timescale) heads to next target.

### Persistence (`NPCBoatData`)
Saved: `currentTarget` (waypoint index), `currentDock`, `parkedTimer`. On load, indices resolved via `NPCBoatWaypointManager.instance.GetWaypointTransform(idx)`. In save — `npcBoatData` array (see note 11).

## Fishing Boats (`NPCFishingBoat`)

Extension of `NPCBoatController` with **daily schedule** by local time (`Sun.sun.localTime`):

| Time (local) | Behavior |
|-------------------|-----------|
| `5:30–9:30` and `13:30–17:30` | `GoFishing()` — heads to fishing spot (`target`). |
| other time | `GoHome()` — returns to dock. |

Positions stored in "real" coordinates (`ShiftingPosToRealPos`/`RealPosToShiftingPos`) — correct relative to floating origin (see note 11). Two "trips" per day: morning and afternoon.

## Trader Boats (`TraderBoat`)

Economic NPCs — carry goods and **price reports** between ports, linking markets. Detailed in note 13. Briefly: `carriedGoods[]`, `carriedPriceReports[34]`, route via `destinations[]`, state in `TraderBoatData`.

## Port Official (`PortDude`)

NPC at "mission desk" in port. Two roles:

### 1. Mission Delivery Trigger
`OnTriggerEnter` on collider with tag `Good`:
- if good's mission has `destinationPort == this port` → `good.Deliver()` (delivery!);
- if good is from a mission but **wrong port** → notification `"You are at the wrong port!"` (**hardcoded**, EN).

> **Delivery mechanic:** to complete a mission, you must physically bring the good to `PortDude` at the destination port.

### 2. UI Gateway (missions / trade)
`ActivateMissionListUI(openEconomyUI)`:
- `openEconomyUI == false` → open port mission list (`MissionListUI`).
- `openEconomyUI == true` → if `PlayerReputation.GetRepLevel(region) < 1` → `"Not enough reputation"`; otherwise open trade UI (`EconomyUI`).

> **Market trading requires minimum 1 reputation level** in the region.

## Shopkeeper (`Shopkeeper`)

Retail NPC for individual items (not market goods). Methods: `RegisterShop(ShopArea)`, `TryToSellItem(item)`, `TryToBuyItem(item)`, `GetLocalPrice(item)`, `GetLocalPriceString(item)`. Buying/selling via `OnTriggerEnter/Exit` (player brings item into shop area).

## Other NPCs / "Residents"

| Class | Role |
|-------|------|
| `QuestDude` | Issues story quests (`Quest`/`Quests`, separate system from cargo missions). |
| `TavernRumorsDude` / `PortRumors` / `Rumor` | Tavern/port rumors (hints, atmosphere). |
| `CargoTransportDude` | Cargo carrier/transporter (`CargoCarrier`). |
| `NPCAnimations` | NPC animations. |
| `Seagulls` | Atmospheric seagulls: particles + sound, rise/fall by height (cycle ±600 units). |
| `Balloon` / `Windchimes` | Decorative objects reacting to wind. |

## Practical Takeaways for Modding

1. **NPC boats** follow waypoints (`NPCBoatWaypointManager`), use sails, avoid collisions within 15 m, active only near player (`BoatHorizon.closeToPlayer`).
2. **Mission delivery** = physically bring goods to `PortDude` at destination port (trigger by tag `Good`).
3. **Market trading** requires ≥ 1 reputation level in region; mission list accessible without reputation.
4. **Fishing schedule** tied to local solar time (2 fishing windows per day).
5. NPC boats save only waypoint indices + dock timer — lightweight persistence.
6. NPC utility strings (`"You are at the wrong port!"`, `"Not enough reputation"`) are hardcoded (EN) — see note 08.
