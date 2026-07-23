# 23. Fishing and Food Preservation

Analysis of fishing and food spoilage/conservation subsystems. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSPy.

## Fish Selection (`OceanFishes`, `LocalFishesRegion`)

Type of caught fish is determined by **geography** (latitude), with local overrides.

`OceanFishes.instance.GetFish(pos)`:
1. **Local regions** (`LocalFishesRegion[]`): for each — influence = `InverseLerp(outerRadius, innerRadius, distance) * overrideInfluence`. If `Random(0,1) <= influence` → random fish from region's `localFishPrefabs`.
   - Default: `outerRadius = 5000`, `innerRadius = 2500`, `overrideInfluence = 0.75`.
2. **Otherwise by latitude:** `latitude = globeCoords.z + weightedDeviation` (random deviation, averaged over 6 iterations, amplitude `deviationDistance`). Fish with nearest `peakLatitude[i]` is selected.

Each fish prefab has a "peak latitude" habitat (`peakLatitude[]`). Debug: key `j` logs which fish will be caught (`DebugFishCatch`).

## Fishing Mini-Game (`FishingRodFish`)

### Bite
With cast bobber (`bobber` in water, line released, `linearLimit > 1`), rod in hand, no fish yet:
- `fishTimer` ticks once per second.
- **Bite chance** = `InverseLerp(3, 20, distance) * 2.5 + 0.5` % per second, where `distance` — distance to rod. The **longer the cast**, the higher the chance (from ~0.5% at 3 m to ~3% at 20 m).
- On success → `CatchFish()`: fish selected (`OceanFishes.GetFish`), bobber mesh replaced with fish model, `fishEnergy = 1`.

### Fight (Line Tension)
In `FixedUpdate`:
- Fish pulls (`fishPullForce`), while in water and has energy.
- `pullForce` = current joint force (`ConfigurableJoint.currentForce`) — line tension.
- `currentTargetTension` grows when `pullForce >= lowerForceThreshold` (player pulling against fish), accounting for reeling (`reelBendMult`, line length).
- **Line break:** if `tension > 0.95` — `snapTimer` grows, rod shakes, tension sound plays; **at `snapTimer > 3.1 s` → `ReleaseFish()`** (line breaks, fish lost).
- Tension drops when not pulling. `rod.SetRodTension(tension)` visually bends the rod.

### Catch and Landing (`CollectFish`)
- Instantiates caught fish as `ShipItem` (`sold = true`, `RegisterToSave`).
- **31% chance to lose hook:** `if Random(0,100) > 69 → rod.DetachHook()`.
- If bobber out of water — fish is "dead" (`fishDead`), fight ends.

## Food Preservation (`FoodState`)

Food spoils; it can be **preserved** by drying, smoking, salting, or cooking.

### Fields (all 0..1, except as noted)
| Field | Content |
|------|-----------|
| `spoilDuration` | Base duration until spoilage. |
| `dried` | Drying degree. |
| `smoked` | Smoking degree. |
| `salted` | Salting degree. |
| `spoiled` | Spoilage degree (`> 0.9` = "rotten"). |
| `slicePrefabIndex` / `slicesCount` | Slicing (pieces). |
| `inWater` | In water (softens). |

### Drying (`dried`)
```
rate = deltaTime * timescale / (30 * energyPerBite)
rate *= Lerp(1, 10, salted)          // salt accelerates drying
if on drying rack (DryingRackCol): rate *= 2
if layer 26 (in inventory): rate = 0
if inWater: rate = -100 * deltaTime * timescale   // softens in water
dried += rate   (clamped 0..1)
+ smoking: if stove in smoking mode → dried += currentHeat
```

### Spoilage (`spoiled`)
```
rate = deltaTime * timescale / spoilDuration
rate *= InverseLerp(1, 0, currentHeat)   // heating (cooking) stops spoilage
rate *= (1 - dried)                       // dried doesn't spoil
rate *= (1 - smoked)                      // smoked doesn't spoil
spoiled += rate
```
**Conclusion:** drying and smoking completely prevent spoilage; cooking (heat) — while hot; salt accelerates drying.

### Salting
`GetRequiredSalt() = energyPerBite * 0.33 * (1 - salted)` — amount of salt needed. `AddSmoked(amount)` adds smoking (cap 1).

### State Text Prefixes (hardcoded, EN)
Check order in `UpdateLookText` (first match):

| Condition | Prefix |
|---------|---------|
| `spoiled > 0.9` | `"rotten "` |
| `amount >= 1.5` | `"burnt "` (overcooked) |
| `salted >= 0.99 && smoked >= 0.99` | `"salted smoked "` |
| `salted >= 0.99` | `"salted "` |
| `smoked >= 0.99` | `"smoked "` |
| `dried >= 0.99` | `"dried "` |
| `amount >= 1` | `"cooked "` |
| `raw && spoiled < 0.2` | `"fresh raw "` |
| `raw` | `"raw "` |
| `spoiled < 0.2` | `"fresh "` |

Result: `description = prefix + name`. Drying rack detected by trigger `DryingRackCol`.

## Liquids (`LiquidType`)

```csharp
public enum LiquidType {
  none, water, rum, wine, cocoWine, mead, honeyBeer, riceBeer,
  cider, seaWater, coffee, blackTea, greenTea, whiteTea
}
```
14 liquid types (for kettle `ShipItemKettle`, bottles, drinks). Tea: black/green/white; alcohol: rum/wine/cocoWine/mead/honeyBeer/riceBeer/cider; non-alcoholic: water/coffee/seaWater.

## Practical Takeaways for Modding

1. **Fish by latitude:** type determined by `globeCoords.z` (+ randomness) and `peakLatitude[]` of prefabs; local regions (`LocalFishesRegion`) override catch within radius.
2. **Bite** depends on cast distance (3–20 m → 0.5–3%/s).
3. **Mini-game** = tension management: pull, but don't hold `tension > 0.95` for more than ~3.1 s, otherwise line breaks. 31% chance to lose hook on landing.
4. **Food preservation:** `dried`/`smoked` completely block spoilage; `salted` accelerates drying; cooking (heat) stops spoilage while hot.
5. **Food prefixes** (`"rotten"`, `"smoked"`, `"cooked"`…) are hardcoded in English in `FoodState.UpdateLookText` (see note 08).
6. Food state (`dried/smoked/salted/spoiled`) is saved via `SavePrefabData.extraValue0..3` (note 11).
