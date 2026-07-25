# 38. Ropes, Rigging, and Steering (Wheel, Winches)

Analysis of boat control subsystem — how the player steers and works sails via ropes. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Sails and wind — in note 17, anchor rope — in note 29.

## Rope Abstraction (`RopeController`)

Base class of all "controllable ropes":

```csharp
public class RopeController : MonoBehaviour {
    public float currentLength;        // current length/retraction
    public float currentResistance;    // current resistance (tension)
    public bool  changed;

    public virtual void UpdateSailAttachment() {}
    public virtual void Pull(float input)   {}   // pull
    public virtual void Loosen(float input) {}   // let out
    public virtual bool CanPull() => true;        // can pull (e.g. anchor)
}
```

### Subclasses (each — own type of control)
| Class | What It Does |
|-------|-----------|
| `RopeControllerAnchor` | Anchor rope (`resistanceMult`, `canPull`, note 29). |
| `RopeControllerSailAngle` | Sail angle (sheet). |
| `RopeControllerSailAngleJib` | Jib angle. |
| `RopeControllerSailAngleSquare` | Square sail angle. |
| `RopeControllerSailReef` | Sail reefing (area). |
| `RopeControllerSteeringWheel` | Steering wheel rope → rudder. |
| `RopeControllerMirror` | Mirror (NPC boats, auto-control). |

## Steering Wheel (`GPButtonSteeringWheel`)

Controls rudder via **`HingeJoint` spring**.

| Field | Content |
|------|-----------|
| `attachedRudder` (`HingeJoint`) | Connection to rudder. |
| `rudder` (`Rudder`) | Rudder component (`currentAngle`). |
| `gearRatio` (3) | Gear ratio: **wheel angle = rudder angle × 3**. |
| `currentInput` | Current steering input. |
| `rotationAngleLimit` | Turn limit (`limits.max * gearRatio`). |
| `locked` | **Wheel lock** (hold course). |

### Logic
- **Input** (mouse drag / keyboard / touch / VR) → `currentInput` (within `±rotationAngleLimit`).
- **Apply to rudder** (`ApplyRudderRotation`): `spring.targetPosition = limits.max * (currentInput / rotationAngleLimit)` — `HingeJoint` spring turns rudder.
- **Course lock** (`Lock/Unlock`, alt button): `locked = true` — wheel holds position (built-in "course hold autopilot").
- **Angle relationship:** `RudderToWheelAngle(a) = a * gearRatio`, `WheelToRudderAngle(a) = a / gearRatio`.
- **Steering modes:** `Settings.steeringWithMouse` (rotate with cursor capture) vs sticky-click; VR — `TouchRotateHandle`.
- Wheel creak sound louder during fast rotation.

## Winch/Cleat for Ropes (`GPButtonRopeWinch`)

Interface for pulling/letting out rope (sails, anchor).

| Field | Content |
|------|-----------|
| `rope` (`RopeController`) | Controlled rope. |
| `gearRatio` (5) | Winch gear ratio. |
| `rotationSpeed` (10) | Rotation speed. |
| `reverseWindResistance` | Inverse wind resistance. |
| `leftRightInput` / `continuousInput` | Input type. |
| `ropeEffect` (`LineRenderer`) | Rope visual (material changes on tension: `pullMaterial` vs `default`). |

- Player rotates winch → `rope.Pull(input)` / `rope.Loosen(input)`.
- `AttachToController(controller)` — bind winch to rope controller.
- `ShowWinch(state)` — show/hide.

## Relation to Other Systems

| System | Connection |
|---------|-------|
| `Sail` (note 17) | `RopeControllerSailAngle` sets sail angle to wind; `RopeControllerSailReef` — area (`currentUnroll`). |
| `Rudder` / `RudderNew` | Rudder (HingeJoint), turns boat. |
| `Anchor` (note 29) | `RopeControllerAnchor` — anchor rope length/resistance. |
| `NPCBoatController` (note 20) | NPCs use `RopeControllerSailAngle/Reef` + `RopeControllerMirror` for auto-control. |

## 👁 Modder's View: Boat Control Mods

| Want | How |
|------|-----|
| **Autopilot (hold course)** | Built-in lock: `GPButtonSteeringWheel.Lock()` + set `currentInput`. Or patch `ApplyRudderRotation`/rudder spring: hold `spring.targetPosition` by target course (comparing `transform.forward` with desired direction). |
| **Auto-trim sails** | Read `Sail.apparentWind` / `relativeWindAngle` and call `RopeControllerSailAngle.Pull/Loosen`, setting sail optimally to wind. |
| **Auto-reefing** | `RopeControllerSailReef` — change `currentUnroll` by wind force (`Wind.currentWind.magnitude`). |
| **Read rudder/sail state** | `rudder.currentAngle`, `Sail.currentUnroll`, `RopeController.currentLength/currentResistance`. |
| **Keyboard/custom UI control** | Call `rope.Pull/Loosen` and `steeringWheel.currentInput` programmatically (bypassing GoPointer). |
| **Remote control** | Patch `RopeController*` — single point: all ropes go through `Pull/Loosen/CanPull`. |

**Pitfalls:**
- **Rudder is via `HingeJoint` spring**, not direct rotation: for autopilot, control `spring.targetPosition` (as `ApplyRudderRotation` does), not `localEulerAngles` of rudder directly.
- **Gear ratios**: wheel ×3 (`gearRatio`), winch ×5 — account for when translating "input → real angle/length".
- `RopeController` methods are **virtual** — convenient to patch base `Pull/Loosen/CanPull` for global effects (e.g., "all ropes pull faster").
- Wheel/winch input goes through `GoPointer`/`GoPointerMovement` and `Settings.steeringWithMouse`/`winchesWithMouse` (note 03) — your input must coordinate with these settings.
- `CanPull()` on anchor depends on tension (`resistancePullLimit`, note 29) — not every rope can be pulled at any time.

## Practical Takeaways

1. **`RopeController`** — unified rope abstraction (`currentLength`, `Pull/Loosen/CanPull`); subclasses — anchor, sail angle/reef, steering wheel.
2. **Steering wheel** controls via **rudder `HingeJoint` spring** (gear ratio ×3), has **course lock** (`locked`) — foundation for autopilot.
3. **Winch** (`GPButtonRopeWinch`, ×5) pulls/lets out sail/anchor rope; tension visual via `LineRenderer`.
4. **Autopilot/auto-trim** built on `spring.targetPosition` of rudder and `RopeController*.Pull/Loosen` + reading `Sail.apparentWind`.
5. NPC boats use same controllers (`RopeControllerMirror`) — you can study their auto-control logic.
