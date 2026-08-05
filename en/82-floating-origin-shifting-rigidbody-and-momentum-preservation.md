# 82. Floating Origin: ShiftingRigidbody, PhysX Jumps, and Momentum Preservation

A complete technical analysis of the floating origin mechanic (`FloatingOriginManager`) and the momentum preservation component (`ShiftingRigidbody`) in Sailwind v0.38 (`Assembly-CSharp.dll`). This note is **critical for item and ship physics mods**: it explains why physical bodies lose velocity or clip through decks during origin shifts, and how the engine prevents these artifacts in vanilla gameplay.

---

## 1. The Floating Origin Problem

In Sailwind's open world, distances between islands span tens of kilometers. To prevent Unity world coordinates from exceeding floating-point (`float`) precision limits and causing PhysX jitter, `FloatingOriginManager.instance` shifts the world origin every ~1000 meters:

```csharp
public void ShiftOrigin(Vector3 shift)
{
    // Translates all root scene objects by -shift
    ...
}
```

### 1.1. PhysX Behavior During a 1000 m Shift
When a GameObject's world position is instantaneously modified by `shift`, the PhysX engine can misinterpret the translation as:
1. An instantaneous, massive velocity impulse (`v = dS / dt`);
2. A contact breakdown between items resting on the deck and the ship's collider hierarchy;
3. Spurious buoyancy probe readings (`BoatProbes`, `SimpleFloatingObject`) due to wave mesh index rebuilding.

---

## 2. `ShiftingRigidbody` Architecture

To shield physical bodies (ships and active items) from desynchronization during an origin shift, the `ShiftingRigidbody : MonoBehaviour` component is attached. Upon initialization, it registers itself with the global manager:

```csharp
private void Start()
{
    rigidbody = GetComponent<Rigidbody>();
    wasKinematic = rigidbody.isKinematic;
    wasSleepThreshold = rigidbody.sleepThreshold;
    boatProbes = GetComponent<BoatProbes>();
    FloatingOriginManager.shiftingRigidbodies.Add(this);
}
```

### 2.1. Shift Preparation (`PrepareForShifting`)

Before `FloatingOriginManager` shifts scene coordinates, it invokes `PrepareForShifting()` across all registered `ShiftingRigidbody` instances:

```csharp
public void PrepareForShifting()
{
    shifting = true;
    preservedVelocity = rigidbody.velocity;
    preservedAngularVel = rigidbody.angularVelocity;
    if (stopProbes)
        boatProbes.dontUpdateVelocity = true;
    if (setKinematic)
        rigidbody.isKinematic = true;
    if (setToSleep)
        rigidbody.Sleep();
}
```

| Parameter / Action | Purpose |
|---|---|
| `preservedVelocity` / `preservedAngularVel` | Caches exact linear and angular velocity vectors **prior** to translation. |
| `stopProbes == true` | Locks hydrodynamic force evaluation in `BoatProbes` (`dontUpdateVelocity = true`), preventing coordinate jumps from injecting massive vertical buoyancy impulses. |
| `setKinematic == true` | Temporarily forces the `Rigidbody` into kinematic mode, suspending collision and force solvers during the shift step. |
| `setToSleep == true` | Forcibly puts the physical body to sleep (`Sleep()`). |

---

## 3. Momentum Restoration and Multi-Frame Delays (`DoRestoreMomentum`)

Following the coordinate shift, `RestoreMomentum()` is called, executing the `DoRestoreMomentum()` coroutine:

```csharp
private IEnumerator DoRestoreMomentum()
{
    if (preserveVelocity)
        rigidbody.velocity = preservedVelocity;
    if (preserveAngular)
        rigidbody.angularVelocity = preservedAngularVel;
    if (wakeUp)
        rigidbody.WakeUp();
    shifting = false;

    for (int i = 0; i < waitForFrames; i++)
        yield return new WaitForEndOfFrame();

    for (int j = 0; j < waitForFixedFrames; j++)
        yield return new WaitForFixedUpdate();

    if (stopProbes)
        boatProbes.dontUpdateVelocity = false;
}
```

### 3.1. Why `waitForFixedFrames` Is Essential
Notice the multi-frame gating inside the restoration coroutine:
1. Linear and angular velocities are restored immediately in the first frame after the shift.
2. However, **hydrodynamic buoyancy force calculations (`BoatProbes.dontUpdateVelocity = false`) remain locked** for `waitForFrames` render frames and `waitForFixedFrames` physics steps (`WaitForFixedUpdate`)!
3. **Why?** After an origin shift, Crest's wave mesh and Unity's collider hierarchy rebuild their broadphase spatial trees. Re-enabling buoyancy calculation immediately can sample invalid transient wave heights, launching boats or items into the air.

---

## 4. Practical Modding Conclusions for Item Physics

1. **Mandatory Registration for Custom Bodies:** If your mod spawns custom dynamic `Rigidbody` objects (e.g., physically active cargo crates or custom boats), **always attach `ShiftingRigidbody`** or register them with `FloatingOriginManager.shiftingRigidbodies`.
2. **Non-Kinematic Deck Item Hazards:** In vanilla Sailwind, deck items are parented to the boat's local frame (`BoatLocalItems`). If your mod converts deck items into independent dynamic Rigidbodies, they can suffer penetration spikes and fall through the deck during `ShiftOrigin`.
3. **Safe Teleportation:** When translating items or vessels programmatically, always convert coordinates via `FloatingOriginManager.instance.ShiftingPosToRealPos` and `RealPosToShiftingPos`.
4. **Wave Physics Gating:** If you implement custom buoyancy for modded items, suppress vertical buoyancy forces for `2–3` physics frames (`WaitForFixedUpdate`) after any floating origin shift.
