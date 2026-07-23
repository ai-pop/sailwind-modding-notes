# 17. Wind and Sails: Movement Model

Analysis of the wind and sail propulsion subsystem — the heart of Sailwind gameplay. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## Wind (`Wind`)

Singleton `Wind.instance`. Wind vector — direction × force (magnitude). Static fields: `currentWind` (actual, with gusts), `currentBaseWind` (base), `windRotation`.

Starting wind: `normalized(1, 0, -0.25) * 9` (≈9 units).

### Wind Structure (3 Levels)
```
currentBaseWind   ← base (slow direction/force changes)
   └─ currentWindTarget  ← target with storm and ocean influence
        └─ currentGustTarget  ← gusts (fast oscillations ±33%)
             └─ currentWind  ← actual (lerp toward gustTarget)
```

### Wind Change (`SetNewWindTarget`, by timer `changeTimer * [0.5..2]`)
1. **Direction:** random unit vector mixed with **trade wind** (`tradeWindInfluence`, default 0.5) and with current base wind (`directionChaos` from region).
2. **Force:** `base ± magnitudeChaos`, clamped to `[minimumMagnitude, maximumMagnitude]`.
3. `directionChaos` / `magnitudeChaos` come from `Weather.instance.currentRegion.windDirChaos` / `.windChaos` — **wind varies by region**.

### Gusts (`SetNewGustTarget`, timer `gustChangeTimer`)
`currentGustTarget = currentWindTarget * Random(0.67, 1.33)` — fast force oscillations ±33%. `currentWind` smoothly approaches target at `finalLerpSpeed`.

### Wind Amplification
Added to base force (total cap **+20**):
- **Storm:** `+26 * InverseLerp(13000, 500, WeatherStorms.currentStormDistance)` — closer storm means stronger wind (up to +26, but total cap 20).
- **Open ocean:** `+baseMagnitude * InverseLerp(1500, 4000, GameState.distanceToLand) * 0.66` — wind is stronger far from shore.

### Trade Winds (`GetCurrentTradeWind`)
Direction based on **globe coordinates** (`FloatingOriginManager.GetGlobeCoords`, divisor 9000):

| Condition (globe x/z) | Trade wind vector |
|---------------------|----------------|
| `z < 30` | none (zero) |
| `z > northTradeWindBorder` | `normalized(1, 0, 0.5)` |
| `x < -2` | `normalized(0.75, 0, 0.75)` |
| `z < southTradeWindBorder` | `normalized(-1, 0, -0.5)` |
| otherwise | none |

This creates stable wind belts on the world map.

### External Control
- `ForceNewWind(v)` / `LoadWind(v)` — set wind instantly (used on save load and debugging). `LoadWind` clamps magnitude to `maximumMagnitude`.
- `debugConstantWind` (editor only) freezes wind; `Debugger.debugWind` forces `(10, 0, 10)`.
- Wind is saved to save as `currentBaseWind` (see note 11).

## Sail (`Sail`)

`[RequireComponent(HingeJoint, SailConnections)]`, uses Unity `Cloth` for canvas simulation.

### Key Fields
| Field | Content |
|------|-----------|
| `sailName` / `prefabIndex` | Name and index in `PrefabsDirectory.sails`. |
| `price` | Price (default 1000). |
| `category` | `SailCategory` (sail type). |
| `squareSail` / `junkType` | Square/junk sail flags. |
| `sailArea` | Area (calculated from mesh, `CalculateSurfaceArea`). |
| `sailAmplifier` | Thrust amplifier. |
| `upwindEfficiency` | Upwind effectiveness (0..1). |
| `minAngle` / `maxAngle` | Attack angle range. |
| `installHeight` / `scaledInstallHeight` | Mast installation height. |
| `mastOrder` | Mast ordinal number. |
| `currentUnroll` | Sail unfurl degree (0..1, reefing). |
| `apparentWind` | **Apparent wind** (vector). |

### `SailCategory` (enum)
`square` (square), `lateen` (lateen), `junk` (junk), `gaff` (gaff), `other`, `staysail` (staysail).

**Thrust multipliers by category** (`ApplyForce`):
- `junk`: force × **0.75**
- `gaff`: force × **0.85**
- others: × 1.0

(Category also affects sail mass in `BoatMass`, see note 14.)

### Thrust Calculation
1. **Apparent wind** = wind vector minus boat velocity (`apparentWind`).
2. **Total force:** `apparentWind.magnitude × GetWindHeightMult(height) × GetCurrentShadowMult()`.
   - `GetWindHeightMult` — wind is stronger higher above water.
   - `GetCurrentShadowMult` — **wind shadow**: sails blocked by other sails (`SailShadowCol`) get less wind.
3. **Angle efficiency:** `Lerp(force * upwindEfficiency, force, relativeWindAngle/90) × max(debugYardMult, currentUnroll)`.
4. **No-go zone:** multiplier `InverseLerp(13, 16, relativeWindAngle)` — at angle to wind **< 13–16°** sail barely works (can't sail close-hauled).
5. If boat is sunk (`damage.sunk`) — force = 0.
6. **Decomposition** relative to hull:
   - `forwardShare = Lerp((90-angle)/90, 1, upwindEfficiency)` — forward thrust share.
   - `sidewayShare = angle/90` — lateral drift share.
   - If force is backward (>90° from course) — factor **−0.33** (sail "pushes back", but weaker).

### Force Application (`ApplyForce`)
```
realSailPower = sailArea * Debugger.instance.debugSailAreaMult
forwardForce  = unamplifiedForward * realSailPower * 50
sideForce     = unamplifiedSideway * realSailPower * 50 * 1.5   // lateral × 1.5
shipRigidbody.AddForceAtPosition(forward, windcenter)
shipRigidbody.AddForceAtPosition(side,    windcenter)
```
- Base force multiplier = **50**, lateral component additionally **× 1.5** (sails heel/drift more than push forward).
- `GetRealSailPower()` = `sailArea * debugSailAreaMult` — same value used for sail mass.

### Force Direction (`GetSailForceDirection`)
Sail pushes along its normal: for regular sails — `up`/`down`, for square sails (`squareSail`) — `right`/`left`. The face blown by apparent wind is selected.

### Sail Scaling (`ChangeScale`)
- Size changes within **0.2..3.0** on axes.
- Square sails scale differently (Z limited relative to Y: `Y±[0.2..0.6]`, X=Z).
- On change, `scaledInstallHeight` and `sailArea` are recalculated, shadow collider updated.

### Wind on Hull (`BoatWind`)
Separate component applies wind **to the hull itself** (windage/drift without sails):
```
apparent = Wind.currentWind - body.velocity
angle    = |SignedAngle(boat.forward, apparent)|   // >90 → 180-angle
force    = apparent * Lerp(frontForce, sideForce, angle/90)
body.AddForceAtPosition(force, TransformPoint(0,0,-2))
```
Headwind → `frontForce`, crosswind → `sideForce`. Application point offset aft (`forceOffset = (0,0,-2)`).

## Practical Takeaways for Modding

1. **Wind** = base (regional chaos + trade winds by globe coords) + gusts (±33%) + amplification from storm/ocean (cap +20). Regions define `windChaos`/`windDirChaos`.
2. **Trade winds** are tied to globe coordinates (divisor 9000) — prevailing wind can be predicted by position.
3. **Sail no-go zone** — 13–16° to wind; `upwindEfficiency` determines how well the sail works on a close-hauled course.
4. **Sail categories:** junk ×0.75, gaff ×0.85 thrust (but also affect mass). Square sails are better on downwind courses.
5. **Lateral force × 1.5** over longitudinal — sails strongly heel; account for this in balance.
6. **Wind shadow** (`SailShadowCol`): sails behind others lose thrust.
7. **Global tuning multipliers:** `Debugger.instance.debugSailAreaMult` (area/force), `debugYardMult` (min. unfurl). `Wind.ForceNewWind` for weather scripts.
8. Wind and sail state (scale, unroll, color) are saved to save (`SaveSailData`, `SaveBoatCustomizationData`).
