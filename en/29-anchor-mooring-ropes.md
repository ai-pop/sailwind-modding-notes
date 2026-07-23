# 29. Anchor, Mooring, and Ropes

Analysis of anchor and mooring mechanics. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## Anchor (`Anchor : PickupableItem`)

`[RequireComponent(ConfigurableJoint, Rigidbody, CapsuleCollider)]`. Anchor is connected to boat via `ConfigurableJoint` (rope = joint limit).

| Field | Default | Content |
|------|:--:|-----------|
| `anchorDrag` | 5 | Anchor drag (in water). |
| `anchorDragInWater` | 4 | Drag in water. |
| `anchorDragUp` | 1 | Minimum drag (when raising). |
| `unsetResistance` | 150 | Force threshold at which anchor breaks free. |
| `resistancePullLimit` | 200 | Tension limit for "can pull". |
| `set` (private) | — | **Whether anchor is set** (hooked on bottom). |

### Anchor Resistance Scales with Boat Mass
`RegisterRopeController`:
```
unsetResistance = connectedBody.mass * 3.3 * rope.resistanceMult
resistancePullLimit = unsetResistance
```
Heavier boats — stronger anchor hold.

### Auto-Setting (`SetAnchor`)
Anchor **automatically hooks the bottom** when (in `ExtraFixedUpdate`):
- not set (`!set`), **touching ground** (`IsTouchingGround` — collisions with tags `Terrain`/`OceanBottom` or layer 14),
- tilted correctly (`Angle(anchor.forward, up) > 60°`),
- rope deployed sufficiently (`linearLimit > 9`).

On setting: `body.isKinematic = true`, `drag = anchorDrag`, sound.

### Auto-Release (`ReleaseAnchor`)
Anchor **breaks free** when set and:
- rope angle from vertical is large and tension exceeds `unsetResistance * InverseLerp(10,60,angle)` (strong horizontal pull dislodges anchor);
- or if angle ≥ 60° and distance < 3 m — `canPull = false` (anchor "at breaking point");
- or rope retracted too much (`linearLimit <= 8`).

On release: `isKinematic = false`, sound (pitch 1.44).

### Rope Length Management
- When anchor is held (`held`), rope **auto-deploys**: `currentLength` grows with distance; red outline at > 85%; at `currentLength >= 1` — drop (`DropItem`).
- `GetRopeLength() = rope.currentLength`; `IsSet() = set`.

### Mass Dynamics
- Rope short (`limit < 1`): anchor mass drops to 0.2 (easy to raise), center of mass shifted back.
- Rope deployed: mass returns to `initialMass`, center of mass `(0.2, 0, -0.2)`.

### Persistence
`set` → `SaveableObject.extraSetting`, rope length → `extraValue` (note 11). `OnLoad(isSet, ropeLength)` restores.

## Mooring (`BoatMooringRopes`)

Manages boat's mooring ropes.

| Field | Content |
|------|-----------|
| `anchor` | Boat anchor. |
| `ropes[]` (`PickupableBoatMooringRope[]`) | Mooring lines. |
| `mooringSet` | Set of mooring points. |
| `mooringFront` / `mooringBack` | Bow/stern mooring point. |
| `anchorController` (`RopeControllerAnchor`) | Anchor rope controller. |

### Key Methods
- **`AnyRopeMoored()`:** `true` if anchor is set (`anchor.IsSet()`) **or** any mooring line is moored. Used by `Sleep` (timeskip sleep, note 25) and `Recovery`.
- **`UnmoorAllRopes()`:** releases all mooring lines + resets positions.
- **`MoorClosestRope(mooring)`:** finds closest mooring line at origin position and moves it to mooring point (if dock mooring `GPButtonDockMooring` not already occupied).
- **Auto-mooring in `Start()`:** if `mooringFront`/`mooringBack` set — auto-moors to them (used by `Recovery.RecoverBoat` for placing boat in port, note 12).

### Mooring Line Availability
Mooring lines **are raycast-accessible only on purchased boats** (`boatSaveable.extraSetting`): on unpurchased boats, ropes are set to layer 2 (IgnoreRaycast) — can't be grabbed until boat is purchased.

## Rope Controller Family (`RopeController*`)

Different controllers for different ropes:

| Class | Purpose |
|-------|-----------|
| `RopeControllerAnchor` | Anchor rope (`currentLength`, `currentResistance`, `canPull`, `resistanceMult`). |
| `RopeControllerSailAngle` / `…Jib` / `…Square` | Sail angle (sheets). |
| `RopeControllerSailReef` | Sail reefing. |
| `RopeControllerSteeringWheel` | Steering wheel. |
| `RopeControllerMirror` | Mirror (NPC boats). |
| `RopeTrimControllerOld` | Sail trim (legacy). |

Ropes also: `SimpleRope`, `ClothRope`, `HangingRope`, `ChipLogRopeEnd` (visual/physics ropes).

## Practical Takeaways for Modding

1. **Anchor auto-sets** on bottom with proper tilt and sufficient rope length; **auto-releases** on strong horizontal pull.
2. **Anchor holding force** = `boat mass × 3.3 × resistanceMult` — heavy boats hold better.
3. **`AnyRopeMoored()`** (anchor or mooring line) gates timeskip sleep and affects recovery — convenient to check for your logic.
4. **Mooring lines on unpurchased boats** on layer 2 (no interaction) until purchase (`extraSetting`).
5. Anchor state (`set`, rope length) and mooring are saved (note 11); auto-mooring to `mooringFront/Back` used by recovery.
6. Control ropes (`RopeController*`) — entry points for sail/rudder/anchor control mods.
