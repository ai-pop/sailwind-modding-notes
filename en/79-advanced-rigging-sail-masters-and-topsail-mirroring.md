# 79. Advanced Rigging: Sail Angle Masters, Self-Righting, and Topsail Mirroring

A comprehensive technical analysis of advanced rigging control components and sail coordination in Sailwind v0.38 (`Assembly-CSharp.dll`). This note supplements core wind/sail mechanics ([Note 17](17-wind-and-sails.md)) and rope abstractions ([Note 38](38-ropes-rigging-steering.md)).

---

## 1. Rigging Coordination Architecture

In Sailwind, simple sails (single-sheet fore-and-aft or gaff sails) are controlled directly via `RopeControllerSailAngle`. However, complex sail configurations (jibs, staysails, square sails with braces, topsails) require coordinating two sheets/braces or slaving a sail to a lower yard. This is accomplished using **Angle Masters (`AngleMaster`)** and **Mirror Controllers (`Mirror`)**.

```
  [ Left Sheet/Brace ]        [ Right Sheet/Brace ]
         │                             │
         └─────────────┬───────────────┘
                       ▼
          [ JibAngleMaster / SquareAngleMaster ]
                       │
                       ▼ (Evaluates min/max limits + sway)
              [ Sail HingeJoint ]
                       │
                       ├─────────────────────────────────┐
                       ▼                                 ▼
            [ JibSelfRighting ]             [ SquareTopsailAngleMirror ]
        (No-wind centering spring)          (Copies limits to upper yard)
```

---

## 2. Jib and Staysail Control (`JibAngleMaster`)

Fore-and-aft headsails (jibs, staysails) operate via two sheets (left and right). `JibAngleMaster` combines their tension and regulates `HingeJoint` limits.

### 2.1. Sheet Limit Intersection (`Update`)
Every frame, the component retrieves angular limits from both sheets and computes their intersection:

```csharp
currentLimitPlus  = Mathf.Min(limitPlusFromLeft, limitPlusFromRight);
currentLimitMinus = Mathf.Max(limitMinusFromLeft, limitMinusFromRight);

JointLimits limits = sailHinge.limits;
limits.max = currentLimitPlus;
limits.min = currentLimitMinus;
sailHinge.limits = limits;

if (limitPlusFromRight <= limitMinusFromLeft)
{
    limitMinusFromLeft = limitPlusFromRight = (limitPlusFromRight + limitMinusFromLeft) / 2f;
}
```

- `currentLimitPlus`: Maximum positive hinge angle (constrained by whichever sheet is tighter on the positive side).
- `currentLimitMinus`: Maximum negative hinge angle.
- **Sheet Lockup (`CanPull`):** If both sheets are pulled so taut that `limitPlusFromRight <= limitMinusFromLeft`, the hinge locks at the midpoint (`/ 2f`), and `CanPull()` returns `false`, preventing winches from pulling further.

### 2.2. Wind-Induced Flutter Simulation (`ApplySway`)
To simulate natural sail flutter in the wind, an oscillating offset is injected into the hinge limits:

```csharp
sway += Time.deltaTime * swayDirection * Wind.currentWind.sqrMagnitude * 0.003f;
if (sway > 0.5f)  swayDirection = Random.Range(-1f, -2f);
if (sway < -0.5f) swayDirection = Random.Range(1f, 2f);

JointLimits limits = sailHinge.limits;
limits.max = currentLimitPlus + sway;
limits.min = currentLimitMinus - sway;
sailHinge.limits = limits;
```

The flutter value `sway` oscillates within `[-0.5°, +0.5°]`, with frequency proportional to the square of wind speed (`sqrMagnitude * 0.003f`).

---

## 3. No-Wind Self-Righting (`JibSelfRighting`)

Without wind, a jib on a `HingeJoint` would hang limply at arbitrary angles. `JibSelfRighting` applies a centering spring that smoothly disengages as wind force increases:

```csharp
private void Update()
{
    float num = sail.appliedWindForce / maxWind;
    float spring = Mathf.Lerp(springWithNoWind, 0f, num);
    SetSpring(spring);
}

private void SetSpring(float value)
{
    JointSpring spring = joint.spring;
    spring.spring = value;
    joint.spring = spring;
}
```

- When `appliedWindForce == 0`, the joint spring equals `springWithNoWind`, centering the sail along the ship's centerline.
- As wind picks up, the spring lerps toward `0`, yielding full control to aerodynamic forces and sheet limits.

---

## 4. Square Sail Control (`SquareAngleMaster`)

Square sail yards rotate via left and right braces. `SquareAngleMaster` operates similarly to `JibAngleMaster`, but applies a small tolerance (`0.01f`) to prevent physics joint sticking:

```csharp
JointLimits limits = sailHinge.limits;
limits.max = limitFromLeft + 0.01f;
limits.min = limitFromRight - 0.01f;
sailHinge.limits = limits;
```

If braces are pulled taut (`limitFromLeft <= limitFromRight`), yard rotation locks (`CanPull() == false`).

---

## 5. Topsail Mirroring (`SquareTopsailAngleMirror` & `RopeControllerMirror`)

Topsails and topgallants (upper square sails) in Sailwind do not require dedicated winches; they automatically copy the orientation of the lower sail.

### 5.1. `SquareTopsailAngleMirror`
In `LateUpdate()`, this component synchronizes the upper yard with the lower sail (`sailBelow`):

```csharp
private void LateUpdate()
{
    sail.limits = sailBelow.limits;
    connections.angleControllerLeft.transform.position = leftConnection.position;
    connections.angleControllerRight.transform.position = rightConnection.position;
    leftRope.currentRopeLength = 0f;
    rightRope.currentRopeLength = 0f;
}
```

- Angular rotation limits (`limits`) are copied verbatim from the lower sail.
- Visual rope attachment points are snapped to the lower yard's positions (`leftConnection`, `rightConnection`), with rope length clamped at `0f`.

### 5.2. Universal `RopeControllerMirror`
Used across multi-tier fore-and-aft rigs and NPC vessels:

```csharp
private void Update()
{
    sail.currentUnroll = parentSail.currentUnroll;
    hinge.limits = parentHinge.limits;
}
```

Copies both angular limits (`limits`) and unroll percentage (`currentUnroll`) from `parentSail` every frame.

---

## 6. Practical Modding Conclusions

1. **Multi-Tier Rigs Without Extra UI:** When designing large ships with topsails or topgallants, avoid adding redundant winches. Attach `SquareTopsailAngleMirror` or `RopeControllerMirror` to upper sails to slave them to the lower yard.
2. **Preventing Winch Jamming:** In autopilot or auto-trim mods, check `AngleMaster.CanPull()` before pulling winches. A `false` return value indicates opposing sheets/braces are fully taut; pulling further causes physics jitter.
3. **No-Wind Centering for Custom Sails:** Add `JibSelfRighting` to any freely rotating headsails so they return cleanly to the ship's centerline in becalmed weather.
4. **Tuning Flutter:** The `ApplySway()` flutter amplitude in `JibAngleMaster` scales with wind speed squared (`sqrMagnitude * 0.003f`). For large custom jibs, reduce `0.003f` to prevent sails from clipping through rigging in storms.
