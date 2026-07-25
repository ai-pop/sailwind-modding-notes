# 46. Water Physics: Reality vs Illusion

## Short Answer

**Water in Sailwind is visual.** There is no real water physics (hydrodynamics, resistance, buoyancy) for anyone. But the implementation for ship and player is **fundamentally different**, and this creates the dissonance: the ship behaves "physically correctly", while the player — like a levitating object.

## "Water" System Architecture

### 1. Ocean — Visual Mesh Based on FFT

The `Ocean` class (Crest fork) generates water mesh via fast Fourier transform. This is a **purely visual + sampling** system:

| Component | What It Does |
|-----------|------------|
| `vertices[]` / `normals[]` | Visual water surface (mesh) |
| `GetWaterHeightAtLocation2(x, z)` | Returns height of **visual** surface at point |
| `GetChoppyAtLocation2(x, z)` | Horizontal wave displacement (choppy) |
| `canCheckBuoyancyNow[0]` | Byte gate: `== 1` when wave data ready for query |

**Key:** `Ocean` doesn't create a physical water body. It's a "decoration with height API". No volume, density, viscosity, or hydrodynamics.

### 2. Ship — Physics Illusion via `Rigidbody` + `Buoyancy`

Ship uses **real `Rigidbody`** (Unity physics) + `Buoyancy` component. Classic blob-buoyancy model:

```
For each "blob" (point on BoxCollider grid):
    worldPos = TransformPoint(blobLocalPos)
    waterY = Ocean.GetWaterHeightAtLocation2(worldPos.x - choppy, worldPos.z)
    depth = worldPos.y - waterY          // submersion depth
    force = magnitude * depth            // buoyant force
    force += dampCoeff * velocity.y      // roll damping
    rigidbody.AddForceAtPosition(-up * force, worldPos)
```

**Why ship looks "physically correct":**
- Force applied at submersion points → torque → ship heels/trims realistically
- Center of mass lowered (`CenterOfMassOffset = -1`) → self-righting
- Damping (`dampCoeff = 0.1`) → oscillation suppression
- Interpolation (`interpolation = 3`) → smoothness
- Wind and choppy add random forces

**This is NOT water physics — it's rigid body physics with feedback from visual surface.** But it looks convincing.

### 3. Player — Bare Positioning Without Physics

Player uses `CharacterController` (NOT Rigidbody) + `PlayerSwimming`. **No physics:**

```csharp
// PlayerSwimming.LateUpdate()
waterHeight = Mathf.Lerp(waterHeight, UnderwaterEffect.cameraWaterHeight, deltaTime * lerp);

// State determination (just coordinate comparison):
swimming = (transform.position.y <= waterHeight + swimHeight);
inShallowWater = (transform.position.y <= waterHeight + shallowWaterHeight);
swimmingOnSurface = (transform.position.y >= waterHeight + surfaceSwimHeight);

// "Wave bobbing" — NOT physics, but teleportation:
if (swimming) {
    charController.Move(Vector3.up * (waterHeight - lastWaterHeight));
}
```

**What actually happens:**
1. Every frame, visual water height is read at player position
2. If player below threshold — flag `swimming = true`
3. Player **teleported** up/down by water height delta (`Move(up * delta)`)
4. No forces, inertia, resistance, buoyancy

**Why it looks like "bad levitation":**
- `CharacterController` — kinematic collider, doesn't respond to forces
- No inertia: player instantly follows water
- No water resistance on movement
- No wave bobbing (only vertical shift)
- No difference between surface and depth

### 4. `Boyant` — Simple Snap for Objects

There's a `Boyant` component (typo instead of Buoyant) — does the same thing but for non-physical objects:

```csharp
// Boyant.Update()
float y = Ocean.GetWaterHeightAtLocation2(x - choppy, z) + buoyancy;
transform.position = new Vector3(x, y, z);  // direct position setting
```

Used for fishing bobbers and similar objects. Not for player.

## Comparison Table

| Aspect | Ship | Player |
|--------|---------|-------|
| **Physics component** | `Rigidbody` (dynamic) | `CharacterController` (kinematic) |
| **Water interaction** | Buoyancy force at points | Direct positioning |
| **Inertia** | Yes (Rigidbody mass) | No (instant Move) |
| **Heel/trim** | Yes (torque) | No |
| **Wave bobbing** | Via forces + damping | Vertical teleport |
| **Water resistance** | Yes (drag, dampCoeff) | No |
| **Realism** | High (illusion) | Zero |

## Why It's Done This Way

1. **Performance:** `CharacterController` is cheaper for FPS controller, no simulation needed
2. **Predictability:** for gameplay, controllability matters more than realism
3. **OVR integration:** movement built on `OVRPlayerController` which works via CharacterController
4. **Ships separately:** ships are rare objects, can afford Rigidbody physics

## Consequences for Modding

### If You Want Realistic Swimming

**Variant A: Add Rigidbody physics to player (complex)**
- Replace `CharacterController` with `Rigidbody` + custom controller
- Add `Buoyancy` or equivalent
- Risk: break entire movement, jumping, climbing system

**Variant B: Improve illusion via CharacterController (realistic)**
- Add inertia: instead of instant `Move` — lerp to target height
- Add horizontal bobbing from `choppy`
- Add submersion resistance (slow down Move)
- Add waves via `AddForce` on separate dummy Rigidbody

**Variant C: Hybrid (recommended)**
- Keep CharacterController for movement
- Add child Rigidbody object for visual bobbing
- Sync position via `Move` with lag

### Key Tuning Fields

```csharp
// PlayerSwimming — main parameters
swimHeight = ...;            // depth threshold for swimming
shallowWaterHeight = 0.6f;   // shallow water threshold
surfaceSwimHeight = ...;     // surface swimming threshold
diveSpeedMult = ...;         // dive multiplier
jumpKeyUndiveSpeed = ...;    // surfacing speed
lerp = ...;                  // water-following speed (higher = more abrupt)

// For buoyancy modification — Buoyancy (on ship)
magnitude = 2f;              // buoyancy force
dampCoeff = 0.1f;            // damping
CenterOfMassOffset = -1f;    // center of mass
```

### Antipatterns

- ❌ Don't try adding `Rigidbody` to player prefab — breaks OVR movement
- ❌ Don't change `CharacterController.isKinematic` — that's not the same thing
- ❌ Don't use `Boyant` on player — overwrites position every frame, conflicting with movement
- ✅ For physics illusion — modify `PlayerSwimming` or add post-processing in `LateUpdate`

## Conclusion

**There is no water as a physical volume in Sailwind.** There is:
- Visual mesh with height API (`Ocean`)
- For ships — buoyancy forces on Rigidbody (physics illusion)
- For player — direct positioning by surface height (levitation)

Your feeling of "bad weightlessness" accurately describes what happens in the code.
