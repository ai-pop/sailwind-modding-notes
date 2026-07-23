# 14. Hull Physics: Buoyancy, Mass, Damage, and Sinking

Analysis of the boat physics subsystem. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. The ocean is built on a custom fork of **Crest** (`Ocean.Singleton`, `GetWaterHeightAtLocation2`, `GetChoppyAtLocation`). Useful for physics mods, survivability balance, and realism.

## Buoyancy (`Buoyancy`) — "Blob" Model

Classic blob-buoyancy: the hull is approximated by a grid of "blob" points over the `BoxCollider` area, each sampling wave height and applying an upward force.

| Field | Default | Purpose |
|------|:--:|-----------|
| `magnitude` | 2 | Buoyant force multiplier. |
| `SlicesX` / `SlicesZ` | 2 / 2 | Point grid (minimum 2×2 = 4 blobs). |
| `CenterOfMassOffset` | -1 | Center of mass offset downward (stability). |
| `dampCoeff` | 0.1 | Damping coefficient (roll suppression). |
| `interpolation` | 3 | Per-frame force smoothing. |
| `ypos` | 0 | Vertical offset of sample points. |
| `ChoppynessAffectsPosition` / `ChoppynessFactor` | false / 0.2 | "Choppy" wave displacement effect on position. |
| `WindAffectsPosition` / `WindFactor` | false / 0.1 | Wind drift. |
| `xAngleAddsSliding` / `slideFactor` | false / 0.1 | Sliding at heel angle. |
| `sink` / `sinkForce` | false / 3 | Forced sinking (force per blob). |
| `useFixedUpdate` | — | Tick in `FixedUpdate` vs `Update`. |

### How It Works
1. In `Start()`, a grid of `SlicesX × SlicesZ` points (`blobs`) is built from the `BoxCollider` dimensions.
2. Each blob is assigned a **random sinking weight** (`sinkForces`, square of a random number, normalized by `sinkForce`).
3. Per tick for each blob: world position → water height via `ocean.GetWaterHeightAtLocation2(x - choppy, z)` (accounting for `GetChoppyAtLocation`) → force proportional to submersion depth.
4. `rrigidbody.centerOfMass = (0, CenterOfMassOffset, 0)` — center of mass lowered for self-righting.

### Visibility Optimization
Flags `cvisible` / `wvisible` / `svisible` enable disabling physics off-screen:
- If object is invisible (`!_renderer.isVisible`) — `rigidbody.useGravity = false`, and when `wvisible && svisible`, buoyancy calculation **is completely skipped**.
- Upon returning to frame (and if > 15 frames have passed), the object "snaps" to current wave height to prevent falling through or flying up.

> This is important for mods teleporting boats: upon returning to frame, the Y position is forced to water height.

## Dynamic Mass (`BoatMass`)

Recalculated **every frame only for the active boat** (`GameState.lastBoat == this.transform`).

```
total mass = selfMass (hull)
              + partsMass (custom parts, BoatCustomParts)
              + Σ mass of items on deck (ItemRigidbody, not in inventory slot)
              + Σ sail mass (per mast)
              + 160, if player is aboard (GameState.currentBoat.parent == this)
```

### Center of Mass
```
centerOfMass = keel.centerOfMass + Σ(load_offset × leverageMult)
```
- Each item's offset is taken from `localPosition` rotated by `Euler(0,-90,0)`, weighted by `item.mass / selfMass` ratio.
- `leverageMult` — "lever arm" multiplier: how strongly cargo shifts the center of mass (trim/heel from loading).
- Player position is factored in via `Refs.observerMirror.transform.localPosition`.
- **Player weight is hardcoded = 160** mass units.

### Sail Mass (`GetSailMass`)
```
sail.mass = realSailPower × 20 + category_bonus
```
- `junk` / `gaff`: bonus `+realSailPower × 20`
- not `staysail` (square and others): bonus `+realSailPower × 10`
- `staysail`: bonus `0`

Large sails actually weigh more and raise the boat's center of gravity.

## Hull Damage and Sinking (`BoatDamage`)

Three-parameter hull state model:

| Field | Range | Content |
|------|:--:|-----------|
| `hullDamage` | 0..1 | Hull damage. 1 = maximally broken. |
| `waterLevel` | 0..1 | Bilge water level. **≥ 1 = boat sinks** (`sunk = true`). |
| `oakum` | ≥ 0 | Caulking — slows water intake. |

### Key Tuning Fields
| Field | Default | Purpose |
|------|:--:|-----------|
| `durabilityDays` / `wearSteepness` | — | Daily wear curve. |
| `waterIntakeRate` / `waterDrainRate` | — | Water intake/pump rate. |
| `waterUnitsCapacity` | — | Hull "capacity" (normalization for caulking and rain). |
| `impactDamageMult` | — | Collision damage multiplier. |
| `minimumImpactVelocity` | 1.5 | Min velocity for damage. |
| `maxDamagePerImpact` | 0.15 | Damage cap per single hit. |
| `impactCooldown` | 1 | Seconds between damaging hits. |
| `waterDrag` | 0.2 | Drag from bilge water. |
| `safeAngleLimit` | 35 | Safe heel angle. |
| `sinkVerticalPos` | -4 | Y position for sunk boat. |

### Daily Wear (`DailyDamage`, on `Sun.OnNewDay`)
Only for the active boat (`GameState.lastBoat`):
- Caulking degrades: `oakum *= 0.88` (**−12% per day**).
- `hullDamage` grows via an **exponential wear curve**: parameterized by `durabilityDays` and `wearSteepness` (formula via `Exp`/`Log` — "+1 day of life" per tick). The older/worn the hull, the faster damage accumulates.

### Collision Damage (`Impact`)
Damage is **not** applied if:
- cooldown is active (`impactTimer > 0`, = `impactCooldown` = 1 s);
- player is sleeping and boat is moored;
- collision with tag `OceanBottom` (bottom);
- boat velocity `< minimumImpactVelocity` (1.5);
- hit against `ShipItem` (item).

Otherwise: `hullDamage += clamp(force * impactDamageMult, 0, maxDamagePerImpact=0.15)`.

### Water Dynamics (`UpdateWaterAndDrag`)
- **Pumping out:** `waterLevel -= deltaTime * waterDrainRate * timescale * 10 * InverseLerp(0.2, 0, hullDamage)` — faster with less damage. (Manual pumping via `BilgePump`.)
- **Water intake** (active boat only), accumulated in "chunks" (`waterIntakeChunk`):
  - rain: `+= deltaTime * GameState.rainIntensity * (1/waterUnitsCapacity) * 10`
  - leaks: `+= 1000 * hullDamage * waterIntakeRate * deltaTime * timescale * 0.65`
  - when `chunk ≥ 1`: `waterLevel += 0.001 * chunk * GetCaulkMult()`.
- **Caulking** reduces intake: `GetCaulkMult() = 1 - oakum / (hullDamage * waterUnitsCapacity)` (0 when no damage).
- **Wave overflow** (`Overflow`): swashing water adds `waterIntakeRate * 0.35 * heightDiff * 4`; wakes player if sleeping and `waterLevel > 0.1`.
- **Drag:** `rigidbody.drag = Lerp(0, waterDrag, waterLevel)` (cap 10). Bilge water slows the boat.

### Sinking
1. `waterLevel ≥ 1` → `sinkRotation` is saved (current orientation), local items are cached (`BoatLocalItems.CacheItemsOnSinking`), `sunk = true`.
2. Buoyancy drops: `boat._forceMultiplier -= deltaTime * baseBuoyancy * 0.33` down to 0; collider (`CapsuleCollider`) is disabled.
3. While sinking — `rigidbody.drag += deltaTime * 6.5` (viscous descent underwater).
4. **Buoyancy from bilge water** (before sinking): `_forceMultiplier = Lerp(baseBuoyancy, baseBuoyancy*0.66, (waterLevel-0.5)*2)` — at half-full, buoyancy drops to 66%.

### Recovery After Sinking
On save load `LoadDamage(...)`: if `waterLevel ≥ 1` — boat stays sunk (`sunk = true`, buoyancy 0, items "loaded" as cached). Full recovery is done by `Recovery.RecoverBoat` (zeroes `waterLevel`, see note 12).

## Practical Takeaways for Modding

1. **Buoyancy** — blob model over `BoxCollider`; force via `magnitude`, center of mass via `CenterOfMassOffset`. Changing `boat._forceMultiplier` (BoatProbes) can instantly sink/float.
2. **Survivability** is determined by the triple `hullDamage`/`waterLevel`/`oakum`. Caulking rots at 12%/day; collision damage is capped at 0.15 per hit.
3. **Player weight = 160**, sails add mass via `realSailPower` — affects draft and stability through `BoatMass`.
4. **Trim/heel from cargo** is controlled by `leverageMult` in `BoatMass`.
5. **Invisible boats** don't get physics (optimization) and snap to water upon returning to frame — account for this when teleporting.
6. Hull state (`waterLevel`, `hullDamage`, `oakum`, `sinkRotation`) is saved via `SaveableObject.extraValue*` (see note 11).
