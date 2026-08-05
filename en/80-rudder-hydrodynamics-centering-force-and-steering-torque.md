# 80. Rudder Hydrodynamics: Turning Torque, Drag, Centering Force, and Steering Wheel Spring

A complete breakdown of the ship steering physics model in Sailwind v0.38 (`Assembly-CSharp.dll`). The `Rudder` component is responsible for translating rudder deflection angle and vessel speed into yaw turning torque, hydrodynamic turning drag, and centering feedback forces on the helm. This note supplements the rigging and steering overview in [Note 38](38-ropes-rigging-steering.md).

---

## 1. `Rudder` Component Fields and Parameters

`Rudder` is attached to the rudder blade GameObject and requires a `HingeJoint`, through which the steering wheel (`GPButtonSteeringWheel`) rotates the rudder.

| Field | Type / Default Value | Description |
|---|---|---|
| `shipRigidbody` | `Rigidbody` | The main physical Rigidbody of the boat hull. |
| `rudderPower` | `float = 1000f` | Rudder effectiveness multiplier (turning force). |
| `pushbackPower` | `float = 200f` | Hydrodynamic water pressure multiplier pushing the rudder back to neutral. |
| `dragPower` | `float = 0.02f` | Hydrodynamic resistance coefficient (speed loss while turning). |
| `shipForwardVelocity` | `float` | Current longitudinal velocity of the boat relative to its hull. |
| `currentAngle` | `float` | Rudder deflection angle in degrees (`[-180, +180]`). |
| `currentTension` | `float` | Calculated water flow tension/pressure against the rudder blade. |
| `outAppliedForce` | `float` | Diagnostic read-out of the resulting torque applied. |

---

## 2. Steering and Turning Drag Physics (`FixedUpdate`)

Every physics step, `Rudder` computes local forward velocity, yaw torque, and turning resistance:

```csharp
private void FixedUpdate()
{
    // 1. Longitudinal velocity in local hull coordinates
    shipForwardVelocity = shipRigidbody.transform.InverseTransformDirection(shipRigidbody.velocity).z;
    float absSpeed = Mathf.Abs(shipForwardVelocity);

    // 2. Base yaw torque
    float torque = -1f * rudderPower * absSpeed * currentAngle;

    // 3. Hydrodynamic drag from the deflected rudder blade
    float dragForce = Mathf.Abs(torque) * dragPower;
    if (shipForwardVelocity < 0f)
        dragForce = -dragForce;
    shipRigidbody.AddRelativeForce(Vector3.back * dragForce, ForceMode.Force);

    // 4. Static torque (rudder effectiveness at zero speed)
    torque -= currentAngle * 3.1f;

    // 5. Apply yaw torque (with reverse-gear inversion)
    if (shipForwardVelocity > 0f)
        shipRigidbody.AddRelativeTorque(Vector3.up * torque, ForceMode.Force);
    else
        shipRigidbody.AddRelativeTorque(Vector3.up * -torque, ForceMode.Force);

    // 6. Evaluate hydrodynamic water pressure
    currentTension = shipRigidbody.angularVelocity.normalized.y * shipForwardVelocity * 0.25f;
    ApplyCenteringForce();
    ...
}
```

### 2.1. Yaw Turning Torque (`torque`)
Primary turning torque is evaluated as:

$$T_{\text{yaw}} = -\text{rudderPower} \cdot |v_{\text{forward}}| \cdot \theta_{\text{rudder}} - 3.1 \cdot \theta_{\text{rudder}}$$

- **Speed Dependence:** Turning effectiveness scales linearly with boat speed $|v_{\text{forward}}|$ and rudder angle $\theta_{\text{rudder}}$.
- **Static Term (`-3.1 * angle`):** Even when boat speed is zero, a turned rudder generates a small constant yaw torque. This simulates prop wash or lateral drift, allowing stationary ships to steer slightly when sails generate side thrust.

### 2.2. Hydrodynamic Turning Drag (`dragForce`)
Deflecting the rudder creates induced drag that slows the ship down:

$$F_{\text{drag}} = |T_{\text{yaw}}| \cdot \text{dragPower}$$

The force vector `Vector3.back * dragForce` is applied in the hull's local space. With default `dragPower = 0.02`, putting the helm hard over (30° deflection) noticeably decelerates the vessel.

### 2.3. Reverse-Gear Steering Inversion
When the vessel moves backward (`shipForwardVelocity < 0f`), yaw torque is applied with an inverted sign (`Vector3.up * -torque`), accurately mimicking real-world stern-way steering behavior.

---

## 3. Hydrodynamic Centering Force (`ApplyCenteringForce`)

While turning, water flow exerts pressure against the rudder blade, trying to force it back toward neutral (0°):

```csharp
private void ApplyCenteringForce()
{
    float delta = currentTension * pushbackPower * Time.deltaTime;
    JointSpring spring = hinge.spring;
    spring.targetPosition += delta * 0.1f;
    hinge.spring = spring;
}
```

### Spring Analysis
1. Calculated water pressure `currentTension` scales with turning angular velocity direction (`angularVelocity.normalized.y`) and forward speed (`shipForwardVelocity * 0.25f`).
2. This method continuously shifts the HingeJoint spring target (`hinge.spring.targetPosition`) toward neutral.
3. Because of this centering force, an unattended steering wheel slowly unwinds back to center while underway unless locked (`GPButtonSteeringWheel.Lock()`, [Note 38](38-ropes-rigging-steering.md)).

---

## 4. The `RudderNew` Mystery

The decompiled Assembly-CSharp contains an unused class named `RudderNew : MonoBehaviour`:

```csharp
public class RudderNew : MonoBehaviour
{
    [SerializeField] private float rudderPower = 1000f;
    private float currentAngle;
    private float shipForwardVelocity;
    private float currentResistance;
    private Rigidbody shipRigidbody;
    private RopeControllerSteeringWheel steeringRope;
}
```

This 15-line class contains no `Update` or `FixedUpdate` methods. It is an incomplete developer prototype indicating an abandoned attempt to rewrite rudder physics—moving from `HingeJoint` spring control to direct coupling with `RopeControllerSteeringWheel` (`steeringRope` / `currentResistance`). In Sailwind v0.38, all vanilla boats continue to use the legacy `Rudder` component.

---

## 5. Practical Modding Conclusions

1. **Tuning Modded Boat Maneuverability:** To increase turning rates without sacrificing speed, raise `rudderPower` but lower `dragPower` (e.g., from `0.02f` to `0.008f`).
2. **Compensating for Centering Force in Autopilots:** If an autopilot mod controls rudder angle by manipulating `hinge.spring.targetPosition`, remember that `ApplyCenteringForce()` shifts this target every physics step. To maintain a steady course, overwrite `targetPosition` in `LateUpdate()` or set `pushbackPower = 0f` while autopilot is engaged.
3. **Avoiding Excessive Turning Drag:** Autopilot and AI navigation algorithms should avoid slamming the rudder to full deflection (`±30°`) for minor heading corrections; otherwise, `F_drag` will repeatedly stall the boat.
4. **Zero-Speed Turning:** The static torque term (`-3.1 * angle`) allows ships to turn even when `shipForwardVelocity == 0`. Ultra-realistic physics mods may want to zero out this term for vessels without active propellers or oars.
