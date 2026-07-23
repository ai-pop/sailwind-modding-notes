# 41. Cameras and View Modes

Analysis of camera subsystem. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Camera shake — in note 39 (`Juicebox.ShakeScreen`), look/crosshair — in note 10 (`GoPointer`).

## Camera Views in the Game

| Camera | Class | Purpose |
|--------|-------|-----------|
| First person | `centerEye` (OVRCameraRig) + `MouseLook` | Normal player view. |
| "Boat camera" / overview | `BoatCamera` (singleton) | Side/top view of boat (for sails and shipyard). |
| Smooth follow | `CameraFollow` | Universal follow component. |
| Map camera | `MapTableCamera` (note 37) | Renders map on table. |
| Shipyard camera | `ShipyardCameraRotator` (note 22) | Rotation around boat at shipyard. |

## "Boat Camera" (`BoatCamera`, singleton `BoatCamera.instance`)

Side overview of boat mode. `BoatCamera.on` (static) — whether mode is active.

| Field | Content |
|------|-----------|
| `centerEye` | Camera transform (reused VR rig). |
| `camHeight` (2) | Camera height above boat. |
| `zoomSpeed` (10), `zoomLevel` (-8..-40) | Zoom (scroll): from `-8` (close) to `-40` (far, behind boat). |
| `playerLooks[]` / `boatLooks[]` | `MouseLook` sets, switched on enter/exit. |
| `outlines[]` (`OutlineEffect`) | Item outlines (disabled in camera mode). |
| `crosshair` | Crosshair (hidden in camera mode). |
| `currentPosOffset` | Offset (panorama at shipyard, ±25). |

### Switching
- **Key `InputName 16`** (not at shipyard) → `SwitchOn()` / `SwitchOff()`.
- **`SwitchOn`**: `on = true`, crosshair/outlines disabled, `boatLooks` enabled (instead of `playerLooks`), `centerEye` reparented to boat, splash displaced.
- **`SwitchOff`**: reverse — first person view, crosshair/outlines restored.

### Behavior (`Update`)
- When `on`: camera follows `GameState.currentBoat` (`position = currentBoat.position + currentPosOffset`), height `camHeight`, offset `zoomLevel`.
- **Zoom**: `zoomLevel += GameInput.GetScrollAxis() * zoomSpeed`, clamped `[-40, -8]`.
- **At shipyard** (`GameState.currentShipyard`): panorama via RMB (`currentPosOffset` by `Mouse X`, clamped ±25).
- If player left boat (`currentBoat == null`) → auto-`SwitchOff`.

## Smooth Follow (`CameraFollow`)

Simple reusable follow:
```csharp
public class CameraFollow : MonoBehaviour {
    public Transform target;
    public float positionLerp;   // position follow speed
    public float rotationLerp;   // rotation follow speed

    void Update() {
        transform.position = Vector3.Lerp(transform.position, target.position, dt * positionLerp);
        transform.rotation = Quaternion.Slerp(transform.rotation, target.rotation, dt * rotationLerp);
    }
}
```

## 👁 Modder's View: Camera/View Mods

| Want | How |
|------|-----|
| Enable/disable boat overview | `BoatCamera.instance.SwitchOn()/SwitchOff()`; check `BoatCamera.on`. |
| Custom zoom/overview height | `BoatCamera` fields `camHeight`, `zoomLevel`, `zoomSpeed` (or patch `UpdateZoom`). |
| Custom camera following an object | `CameraFollow` component (target + lerp) on your camera. |
| Third person / free camera | Disable `playerLooks`/`MouseLook`, control your own camera; read player position from `Refs.observerMirror` (note 30). |
| Camera shake | `Juicebox.juice.ShakeScreen(...)` (note 39). |
| Cinematic flyover | `Juicebox.juice.TweenPosition(camera, ...)` + `CameraFollow`. |
| Hide crosshair/outlines for screenshot | `BoatCamera.SetOutlines(false)` + `crosshair.SetActive(false)` (following `SwitchOn` pattern). |

**Pitfalls:**
- `centerEye` is the **VR rig**; in non-VR mode, main camera still tied to it + `MouseLook`. Better to make custom cameras as separate `Camera`, not breaking `centerEye`.
- `BoatCamera` switches `MouseLook` sets (`playerLooks`/`boatLooks`) — if adding own cameras, account for which `MouseLook` is active.
- `InputName 16` — boat camera key; at shipyard its behavior is different (panorama).
- Boat position — `GameState.currentBoat.position` (in scene coordinates, accounting for floating origin, note 11).

## Practical Takeaways

1. **First person view** = `centerEye` (OVRCameraRig) + `MouseLook`; **boat overview** = `BoatCamera` (toggle `InputName 16`, zoom by scroll -8..-40).
2. `BoatCamera.instance` (`SwitchOn/Off`, `camHeight`, `zoomLevel`) — overview control point.
3. **`CameraFollow`** — ready smooth follow for custom cameras.
4. Shake/flyovers — via `Juicebox` (note 39).
5. Custom cameras best done as separate `Camera`, not touching VR rig `centerEye`.
