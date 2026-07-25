# 28. Navigation Instruments and World Scale

Analysis of navigation instruments and — importantly — **conclusion about world unit scale**. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## World Scale: 1 Unit = 1 Meter

`Speedometer` converts speed (`Rigidbody.velocity`, m/s) to display units:

```csharp
outSpeed = velocity.magnitude * 1.944f;   // → knots  (1 m/s = 1.944 knots)
outSpeed = velocity.magnitude * 3.6f;      // → km/h  (1 m/s = 3.6 km/h)
```

Both coefficients correspond to conversion **from m/s**, meaning `velocity` is in m/s and positions are in **meters**. **1 Unity world unit = 1 meter.**

### Related Scales (summary from other notes)
| Quantity | Formula | Units |
|----------|---------|---------|
| Speed | `velocity` | m/s (× 3.6 = km/h, × 1.944 = knots) |
| Mission "distance" | `Vector3.Distance / 100` | **hectometers** (conventional "km", note 15) |
| Globe coordinates | `(pos - offset) / 9000` | "globe degrees" (note 11/17) |
| Play zone | globe x[-12,32] z[26,46] | in globe units (note 19) |

> So the world is in meters, but mission "km" is 100 m (hectometers), and globe coordinates are meters / 9000.

## Compass (`Compass`)

Magnetic needle pointing "north":
- `northAngles` (default `(-90, 0, 180)`) — north orientation (`Quaternion.Euler`).
- `magnetism` (1) — needle rotation speed toward north (`Slerp`, `deltaTime * magnetism`).
- `lockAxis` — axis mask (which needle axes are locked/free).
- North is a **fixed world direction** (position-independent), needle smoothly tracks it.

## Log / Speedometer (`Speedometer`)

- Takes horizontal velocity (`velocity.y = 0`).
- Outputs to `TextMesh` two lines: `"{knots} knots"` and `"{km/h} km/h"`.
- `outSpeed` — externally accessible (for other systems/debug).

## Weather Vane (`WindDirectionDisplay`)

Displays **apparent wind** direction:
- If `shipRigidbody` assigned → `LookAt(pos + (Wind.currentWind - shipRigidbody.velocity))` (wind minus boat velocity = apparent wind).
- Otherwise → `Wind.windRotation` (true wind).

## Time Chronometer (`ChronometerTime`)

Clock face driven by game time:
```
rotation.z = 180 + Sun.sun.globalTime * 15    // 15° per hour
```
Displays **global** time (`globalTime`, not local), 15°/hour like real clocks.

## Latitude Meter (`ChronometerLatitude`)

Instrument for determining latitude from sun (quadrant/octant type):
- `Scroll(input)` rotates `currentRot` by ±0.5, **clamped `[-45, -11]`** — corresponds to in-game latitude range.
- `gnomonAligner` (sun alignment): rotation `1.475 * currentRot + 30`.
- `latitudeDial` (latitude scale): `num = Angle(forward, parent.up)*2 - 90`; rotation = `latitudeDialRotOffset + 198 + num*2.5`.
- Together with sun compass (`GameState.holdingSunCompass`, note 18) allows determining latitude from sun position.

## Practical Takeaways for Modding

1. **1 world unit = 1 meter** (conclusion from speedometer coefficients 3.6 and 1.944). Use this for distance/speed calculations in mods.
2. **Mission "km" ≠ real km**: it's `meters / 100` (hectometers). Don't confuse when calculating `pricePerKm` and deadlines.
3. **Globe coordinates** = meters / 9000 (for latitude/longitude effects: trade winds, day length, fish selection).
4. **Compass north** — fixed direction `Euler(-90,0,180)`; no magnetic anomalies.
5. **Weather vane** shows apparent wind (wind − boat velocity) — important for implementing your own wind indicator.
6. **Chronometer** runs on `globalTime` (15°/hour), latitude meter — by sun, latitude range `[-45,-11]`.
7. All instruments use `TextMesh`/transform rotations; label translation (`"knots"`, `"km/h"`) via text substitution (notes 01/08).
