# 42. Worked Example: Fast Travel / Point Teleport

Second tutorial example (first — note 34, loot crates). Task: **"player can teleport to saved points/ports"**. Exercises tricky spots: floating origin, port registry, player control, persistence between sessions. Links notes 11, 19, 30, 36, 39.

## What the Game Already Provides

| Need | Take | Note |
|-------|-------|---------|
| All ports and their positions | `Port.ports[34]`, `port.portIndex`, `port.GetPortName()`, `transform.position` | 19 |
| Player position | `Refs.observerMirror.transform.position` | 30 |
| Coordinates (world ↔ scene) | `FloatingOriginManager.instance.ShiftingPosToRealPos / RealPosToShiftingPos`, `outCurrentOffset` | 11 |
| Enable/disable control | `Refs.SetPlayerControl(bool)` | 30 |
| Screen blackout | `Blackout.FadeTo(target, duration)` (coroutine) | 25 |
| Mod data between sessions | `GameState.modData` | 11 |
| Smooth animation/sound | `Juicebox.juice`, `UISoundPlayer` | 39, 35 |

## Key Difficulty: Floating Origin

The world shifts (`outCurrentOffset`), so you **cannot** simply teleport to an "absolute" position. Two correct approaches:

**A. Teleport to existing object (port)** — safe:
```csharp
// port position is already in current scene coordinates — just take it
Vector3 dest = Port.ports[index].transform.position + Vector3.up * 2f;
```

**B. Teleport to custom saved point** — store in **real** coordinates, apply in **scene**:
```csharp
// saving (real coordinates, independent of world shift):
Vector3 real = FloatingOriginManager.instance.ShiftingPosToRealPos(player.position);
SaveRealPos(real);   // in GameState.modData

// applying (back to scene coordinates accounting for current shift):
Vector3 scene = FloatingOriginManager.instance.RealPosToShiftingPos(LoadRealPos());
```

> Pitfall: if you save position in scene coordinates and apply after world shift — player flies off kilometers. **Store in real, apply in scene.**

## What to Move

Player is `CharacterController`/`OVRPlayerController` (`Refs.ovrController`, `Refs.charController`), not camera. Teleport via controller `transform.position`:

```csharp
IEnumerator TeleportTo(Vector3 scenePos)
{
    if (GameState.sleeping || GameState.recovering || GameState.currentShipyard != null)
        yield break;                       // don't teleport in these states

    Refs.SetPlayerControl(false);          // disable control
    yield return Blackout.FadeTo(1f, 0.5f); // blackout

    // move player controller:
    Refs.ovrController.transform.position = scenePos;
    // observerMirror will follow (or force):
    Refs.observerMirror.transform.position = scenePos;

    yield return Blackout.FadeTo(0f, 0.5f); // fade back
    Refs.SetPlayerControl(true);           // restore control
}
```

**Pitfalls:**
- `CharacterController` sometimes needs "disable/enable" or moving via `controller.Move`/`transform.position` when controller is inactive — otherwise teleport may not apply (Unity `CharacterController` peculiarity).
- If player **on boat** (`GameState.currentBoat != null`) — teleporting player without boat leaves boat behind; decide whether to teleport boat too (its `Rigidbody`/`SaveableObject`) or disembark player.
- After teleporting into water, player will start swimming (`PlayerSwimming`, note 36) — normal, but check Y height (place slightly above water/deck).

## Point Persistence (Custom Data)

Store list of unlocked points in `GameState.modData` (autosave, note 11):

```csharp
const string KEY = "MyFastTravel.points";

void SavePoints(List<FastPoint> pts) {
    // serialize yourself (JSON/CSV) — modData only stores strings
    GameState.modData[KEY] = string.Join(";", pts.Select(p =>
        $"{p.name}|{p.realPos.x},{p.realPos.y},{p.realPos.z}"));
}

List<FastPoint> LoadPoints() { /* parse GameState.modData[KEY] */ }
```

> `modData` — `Dictionary<string,string>`; complex structures serialize to string (JSON, `key;key`). Keys — with mod prefix to avoid conflicts.

## Points = Ports (Simple Variant)

Can skip storing positions entirely and use **ports as fast travel points**:
- Point list = `Port.ports` (where `!= null`), name = `GetPortName()`.
- Teleport = `Port.ports[i].transform.position`.
- "Unlocked" ports = those player has visited (`GameState.lastVisitedPort`, note 12) or own flag in `modData`.
- Plus: port positions always current (no coordinate/shift issues).

## Feature Checklist

- [x] Points: ports (`Port.ports`) OR custom (store in **real** coordinates in `modData`).
- [x] Teleport: `Refs.SetPlayerControl(false)` → `Blackout.FadeTo` → move `Refs.ovrController` (+ observerMirror) → fade back → `SetPlayerControl(true)`.
- [x] Coordinates: apply via `RealPosToShiftingPos` (for custom points); ports — directly.
- [x] Don't teleport during sleep/recovery/shipyard.
- [x] Consider player boat (`GameState.currentBoat`) and Y height.
- [x] Point selection UI — custom IMGUI/`OnGUI` (like translator) + `Juicebox`/`UISoundPlayer` for polish.

## Generalization: "Teleport/Move Player" Recipe

1. **Where:** object in scene (port — directly) OR custom point (real coordinates → `RealPosToShiftingPos`).
2. **How:** disable control → blackout → `Refs.ovrController.position = dest` → blackout back → enable control.
3. **Constraints:** not during sleep/recovery/shipyard; resolve boat and height questions.
4. **Persistence:** custom points — in `GameState.modData` (string serialization).

Same skeleton covers: "return to boat", "recall to last port", teleport to quest markers, debug warp, etc.
