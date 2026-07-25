# 36. Player Movement: Walking, Jumping, Swimming, Crouching

Analysis of player movement subsystem. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Global references — in note 30, `InputName` values — in note 24.

## Movement Architecture

Player movement is based on **`OVRPlayerController`** (`Refs.ovrController`, from OVR framework) + **`CharacterController`** (`Refs.charController`). On top — game layer components:

| Component | Role |
|-----------|------|
| `OVRPlayerController` | Base movement (OVRLocomotion): `Acceleration`, `Damping`, `swimSpeedMult`, `ratlinesSpeedMult`, `jumping`, `GetMoveInput()`, `ToggleWalking()`, `Stop()`. |
| `CharacterController` | Collisions/slopes: `slopeLimit`, `stepOffset`, `isGrounded`, `Move()`. |
| `PlayerClimb` | Jumping, gravity, slopes, ratlines. |
| `PlayerSwimming` | Swimming/diving. |
| `PlayerCrouching` | Crouching. |
| `PlayerUnstucker` | Anti-stuck mechanism. |

## `InputName` Refinement (for movement)

From movement code, reliably confirmed (supplement to note 24):

| `InputName` | Action |
|:--:|----------|
| `0` | Forward (also used for look-based dive while swimming). |
| `4` | **Jump** (also "surface" while swimming, stand up from crouch). |
| `7` | **Crouch / dive down**. |

## Jumping and Slopes (`PlayerClimb`)

Parameters (SerializeField, tunable):
- `jumpPower`, `minJumpDuration`/`maxJumpDuration`, `gravity`, `terminalVelocity`;
- `maxUpMomentum` (on ground), `maxSwimJumpMomentum` (in water);
- `accel`/`damping`, `jumpDampingMult` (in-air control is sluggish);
- `climbSlopeLimit` (90° — can climb any slope in jump), `climpStepOffset` (0.4);
- `normalSlopeLimit` (40°), `normalStepOffset` (0.1).

### Logic
- **Jump** (`InputName 4`): while key held and `jumpDuration < maxJumpDuration` — accumulate upward impulse (`fallMomentum -= jumpPower*dt`, cap `-currentMaxUpMomentum`). Release after `minJumpDuration` ends jump (hold jump = higher).
- **Falling**: not on ground and not in water → `fallMomentum += gravity*dt` (cap `terminalVelocity`), `controller.Move(down*fallMomentum*dt)`.
- **Ceiling**: upward raycast cancels jump on collision.
- **Slopes**: not on boat → `normalSlopeLimit * 2` (can climb steeper on land); in jump → `climbSlopeLimit` (90°).
- **In water** can only jump from surface (`PlayerSwimming.AllowJumping()` = `swimmingOnSurface`).
- **Debug mode** (`buildDebugModeOn`): `maxJumpDuration = 9999` — infinite "jump" (essentially flying upward).

### Ratlines / Shroud Ladders
- `GameState.onRatlines` determined by `OverlapSphere` by tag `Ratlines`.
- On ratlines: `slopeLimit` raised (can climb vertically), `ratlinesSpeedMult` = `0.08` (climbing up) or `0.38` (if looking down/away from them, >120°). `ClickRatlines()` statically stops player for 0.35 s on click.

## Swimming (`PlayerSwimming`)

Static flags (read from your mods):
- `swimming`, `inShallowWater`, `swimmingOnSurface`, `observerSwimming`, `dive`, `cameraWaterHeight`.

Parameters: `swimHeight`, `shallowWaterHeight` (0.6), `surfaceSwimHeight`, `diveSpeedMult`, `jumpKeyUndiveSpeed`.

### State Determination
```
waterHeight = Lerp(..., UnderwaterEffect.cameraWaterHeight, ...)   // water height (Crest)
swimming          = pos.y <= waterHeight + swimHeight
inShallowWater    = pos.y <= waterHeight + shallowWaterHeight
swimmingOnSurface = pos.y >= waterHeight + surfaceSwimHeight  (and swimming)
```

### Behavior
- **Speed**: `swimSpeedMult` = 0.55 (swimming) / 0.77 (shallow water) / 1 (land).
- **Wave bobbing**: while swimming `charController.Move(up * (waterHeight - lastWaterHeight))` — player rises/falls with wave.
- **Diving/surfacing:**
  - `InputName 4` (jump) + swimming + not at surface → surface (`Move up`);
  - `InputName 7` + swimming → dive down (`Move down`);
  - forward movement (`InputName 0` / `GetPrimaryVertical > 0.01`) + swimming → look-angle dive (`dive = SignedAngle(body.forward, camera.forward) * diveSpeedMult`).
- At surface, can't dive forward (`dive = 0`), but can jump.

## Crouching (`PlayerCrouching`)

- Toggle by `InputName 7` (keydown), not in bed.
- Head height: `initialHeight` ↔ `0.2` (crouch), lerp `dt * 9`.
- Auto-stand: jump (`InputName 4`), swimming, entering bed.
- `Refs.ovrCameraRig` set here. `GetCurrentHeadHeight()` — current height.

## Anti-Stuck (`PlayerUnstucker`)

Over a 3.3 s cycle, accumulates input direction and actual displacement. If player pressed **all 4 directions** (`up && down && left && right`), but actually moved less than `stuckThreshold` (0.5) → log `"! STUCK !"` and **push up by 0.5** (`Translate(up*0.5)`).

## 👁 Modder's View: Movement Mods

| Want | How |
|------|-----|
| Faster/slower walking | `Refs.ovrController.Acceleration` / `Damping`; multipliers `swimSpeedMult`, `ratlinesSpeedMult`. |
| Higher/lower jump, custom gravity | `PlayerClimb` fields (`jumpPower`, `gravity`, `terminalVelocity`, `maxUpMomentum`) — patch or SerializeField. |
| Flight/noclip | Following debug mode pattern: `maxJumpDuration = 9999`, or `controller.detectCollisions = false` + manual `Move`. |
| Teleport player | Move `controller.transform` + `Refs.observerMirror` **in scene coordinates** (account for `FloatingOriginManager.outCurrentOffset`, note 11). |
| Sprint | Own multiplier to `ovrController.Acceleration` per key. |
| Custom swimming | Read `PlayerSwimming.swimming/swimmingOnSurface`; water — `UnderwaterEffect.cameraWaterHeight`. |
| Read movement input | `Refs.ovrController.GetMoveInput()` (Vector2), `GameInput.GetPrimaryVertical/Horizontal` (note 24). |
| Disable control | `Refs.SetPlayerControl(false)` (note 30). |

**Pitfalls:**
- Player position for logic is usually `Refs.observerMirror`, but physically moved by `CharacterController`/`OVRPlayerController`; teleport via controller, not camera.
- `swimSpeedMult`/`ratlinesSpeedMult` **overwritten every frame** by the game (`PlayerSwimming.LateUpdate`, `PlayerClimb.Update`) — own multiplier must be set **after** them or patch these methods.
- All movement tied to `GameState.playing` — doesn't tick in menu/loading.
- `InputName 4/7` — jump/crouch; full input — through missing `GameInput` (note 24).

## Practical Takeaways

1. **Movement** = `OVRPlayerController` (acceleration/damping/multipliers) + `CharacterController` (slopes/steps) + game layers (`PlayerClimb/Swimming/Crouching/Unstucker`).
2. **Jump** with hold (impulse accumulates), gravity/terminal velocity tunable; debug gives "flight" (`maxJumpDuration = 9999`).
3. **Swimming** determined by water height (Crest) relative to player; bobs on waves; dive by look direction, surface/dive by keys.
4. **InputName**: `4` = jump/surface, `7` = crouch/dive down, `0` = forward.
5. **Speed multipliers** (`swimSpeedMult`, `ratlinesSpeedMult`) set by game every frame — patch, don't just set.
6. **Teleport** — via controller, in scene coordinates (accounting for floating origin).
