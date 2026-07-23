# 19. World Structure: Ports, Regions, Maps, Border

Analysis of geography and world structure. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## Regions (`PortRegion`)

```csharp
public enum PortRegion { alankh, emerald, medi, none }   // indices 0..3
```

Three in-game regions + `none`. **Important:** region index is used as index in reputation arrays (`PlayerReputation.reputation[4]`) and currency arrays (`PlayerGold.currency[4]`). Region ↔ currency mapping:

| `PortRegion` | Index | Currency (`Currency`) |
|--------------|:--:|---------------------|
| `alankh` | 0 | Al'Ankh Lions |
| `emerald` | 1 | Emerald Dragons |
| `medi` | 2 | Aestrin Crowns |
| `none` | 3 | (Gold Lions — base/exchange, no region) |

> Currency `gold` (index 3) — base for exchange (`CurrencyMarket` doesn't change its rate daily), not tied to a region. Mission reward is paid in destination region currency (see note 15).

## Local Maps (`LocalMap`)

```csharp
public enum LocalMap { none, alankh, emerald, medi, lagoon }   // 5 values
```

Each port has a `localMap`. Missions use the **world map** if ports have different `localMap` or it's `none` (`Mission.UseOceanMap`); with `alwaysUseWorldMap` on a port — always world map (see note 15). `lagoon` — separate special location.

## Ports (`Port`)

Static registry `Port.ports[34]` — **maximum 34 ports**, index = `portIndex`. Populated in `Start()` (`ports[portIndex] = this`).

| Field | Content |
|------|-----------|
| `portIndex` | Index in `Port.ports`. |
| `portName` | Name (via `GetPortName()`). |
| `region` (`PortRegion`) | Port region. |
| `localMap` (`LocalMap`) | Local map. |
| `hubPort` | "Hub" port (affects mission generation). |
| `alwaysUseWorldMap` | Always world map for missions. |
| `island` (`IslandEconomy`) | Island economy/demand. |
| `localMapLocation` / `oceanMapLocation` | Points on local/world map. |
| `producedGoodPrefabs` | Goods produced by port. |
| `destinationPorts` | Destination ports for missions. |
| `dude` (`PortDude`) | Port NPC (`RegisterDude`/`GetDude`). |

### Port Missions
- `GetMissions(page, world)` → delegates to `IslandMissionOffice.GenerateMissions` (active generator, note 15).
- `Port` also has a **legacy generator** `GenerateMissions` based on demand (`island.GetDemand`), with price formula:
  ```
  price = distance*3 × (1+0.3*count) × (1+1.2e-5*value) × (1+distance*0.0005) × (1+0.0006*mass) × 0.6
  ```
  where `distance` = world units / 100.
- **Deadline** (`GetDueDay`): base = `2 * (day length in hours)`, multipliers from `requiredRepLevel`: `>1` → ×3.5, `==1` → ×2.5, otherwise ×1.5. Higher-reputation missions give more time. `dueDay = GameState.day + round(distance/1000 / dayFactor) + 1`.
- `SpawnGoods`: instantiates mission goods at port (with step-offset, batches of 6).
- `ReduceDemand`/`IncreaseDemand`: accepted mission reduces demand at destination port, cancelled — returns it.

### Debug Teleport
`teleportPlayer = true` teleports player to `port.position + up*501` (debug).

## World Border (`WorldBorder`)

Play zone — **rectangle in globe coordinates** (`FloatingOriginManager.GetGlobeCoords`, divisor 9000):

```
globeX ∈ [-12, 32]
globeZ ∈ [26, 46]
```

Behavior when exiting boundaries (only when `GameState.playing && !recovering`):
1. Countdown starts `outTimer = 120` seconds.
2. Text shown: `"out of game area\nrecovery in N seconds..."` (**hardcoded**, EN).
3. Upon expiry → `Recovery.RecoverPlayer(RecoveryReason.worldBorder)` — teleport to last port with penalty (see note 12).
4. Upon returning to zone, timer resets to 120.

> **For modders:** globe world coordinates = `(scenePos - outCurrentOffset - globeOffset) / 9000`. The play zone is small by "ocean" standards — this explains the floating origin.

## Practical Takeaways for Modding

1. **3 regions** (alankh/emerald/medi), region index = currency and reputation index. `gold` — regionless exchange currency.
2. **34 ports maximum** (`Port.ports[34]`); `portIndex` — stable identifier (used in saves).
3. **Play zone** — globe rectangle x[-12,32] z[26,46]; exit = auto-recovery after 120 s.
4. **Mission deadlines** scale with `requiredRepLevel` (×1.5/×2.5/×3.5 to base time).
5. **Legacy mission price formula** (in `Port.GetTotalPrice`) accounts for distance, count, value, mass — useful for understanding balance, though the arbitrage generator from `IslandMissionOffice` is active.
6. Coordinates: `globeCoords = (pos - outCurrentOffset - globeOffset) / 9000` (see note 11 on floating origin).
