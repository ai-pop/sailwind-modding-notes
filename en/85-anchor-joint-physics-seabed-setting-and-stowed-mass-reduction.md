# 85. Anchor Physics: ConfigurableJoint, Seabed Setting, and Stowed Mass Reduction

A technical breakdown of the anchor physics model (`Anchor : PickupableItem` and `RopeControllerAnchor`) in Sailwind v0.38 (`Assembly-CSharp.dll`). This note supplements the mooring overview in [Note 29](29-anchor-mooring-ropes.md) and reveals the exact physical formulas governing seabed fluke setting, break-loose tension, and dynamic mass reduction when stowed.

---

## 1. Anchor Joint Architecture (`ConfigurableJoint`)

Unlike the ship's rudder (`HingeJoint`, [Note 80](80-rudder-hydrodynamics-centering-force-and-steering-torque.md)), an anchor couples to the vessel via a flexible `ConfigurableJoint`. Upon initialization, the anchor calculates break-loose resistance thresholds:

```csharp
public void RegisterRopeController(RopeControllerAnchor ropeController)
{
    rope = ropeController;
    unsetResistance = joint.connectedBody.mass * 3.3f * rope.resistanceMult;
    resistancePullLimit = unsetResistance;
}
```

- `unsetResistance`: The rope tension required to dislodge the anchor from the seabed. It scales directly with ship mass (`connectedBody.mass * 3.3f`).

---

## 2. Seabed Setting Logic (`SetAnchor`)

An anchor does not set instantaneously upon seabed contact. Four physical criteria must be satisfied simultaneously inside `ExtraFixedUpdate()` for `set = true`:

```csharp
else if (IsTouchingGround() && Vector3.Angle(transform.forward, Vector3.up) > 60f && !audio.isPlaying)
{
    if (joint.linearLimit.limit > 9f)
        SetAnchor();
}
```

| Criterion | Description and Physical Meaning |
|---|---|
| `IsTouchingGround()` | The anchor collider contacts `"Terrain"`, layer `14`, or `"OceanBottom"`. |
| `Angle(forward, up) > 60°` | The angle between the anchor's forward vector and world vertical exceeds 60°. The anchor must **lie on its side/fluke**, rather than standing vertically on its crown. |
| `joint.linearLimit.limit > 9f` | Let-out anchor rope length exceeds **9 meters**. Setting an anchor with less than 9 meters of scope is physically impossible. |
| `!audio.isPlaying` | The anchor has ceased dragging across the bottom and is no longer emitting scraping audio. |

---

## 3. Break-Loose Mechanics (`ReleaseAnchor`)

When set (`set == true`), the system evaluates break-loose conditions every physics step:

```csharp
if (set)
{
    float dy = rope.transform.position.y - transform.position.y;
    float dist = Vector3.Distance(rope.transform.position, transform.position);
    double angleDeg = Math.Acos(dy / dist) * 180.0 / Math.PI;

    if (angleDeg >= 60.0 && dist < 3f)
        rope.canPull = false; // Winch lockout under steep lateral tension

    float t = Mathf.InverseLerp(10f, 60f, (float)angleDeg);
    if (angleDeg < 60.0)
    {
        Vector3 force = joint.currentForce;
        if (force.magnitude > unsetResistance * t)
            ReleaseAnchor(); // Anchor breakout under upward tension
    }

    if (joint.linearLimit.limit <= 8f)
        ReleaseAnchor();     // Automatic release when winched to <= 8 m
}
```

### 3.1. Breakout Analysis
1. **Rope Angle (`angleDeg`):** Evaluates the angle between vertical and the line of sight to the ship's bow cleat.
2. **Upward Breakout:** When `angleDeg < 60°` (meaning the ship sits nearly above the anchor, pulling upward), the resistance threshold scales down via `t`. If joint tension (`currentForce.magnitude`) exceeds `unsetResistance * t`, the anchor breaks loose (`ReleaseAnchor()`).
3. **8-Meter Scope Limit:** Winching the rope to `≤ 8 m` automatically trips anchor release.

---

## 4. Dynamic Stowed Mass Reduction

One of the most fascinating physics solutions in Sailwind is **dynamic mass loss when an anchor is stowed at the bow**:

```csharp
if (joint.linearLimit.limit < 1f)
{
    body.centerOfMass = Vector3.back;
    if (body.mass > 0.2f)
        body.mass -= Time.deltaTime * initialMass;
    if (body.mass < 0.2f)
        body.mass = 0.2f;
}
else
{
    body.centerOfMass = new Vector3(0.2f, 0f, -0.2f);
    if (body.mass < initialMass)
        body.mass = initialMass;
}
```

### 4.1. Why Stowed Anchors Weigh 0.2 kg
- **Deployed (`limit >= 1f`):** The anchor retains its full nominal mass (`initialMass`, typically 100–150 kg) with center of mass offset at `(0.2, 0, -0.2)` to encourage proper seabed fluke tipping.
- **Stowed at Bow (`limit < 1f`):** As soon as the anchor is winched fully into the hawsepipe/cleat, its mass **decay-lerps down to 0.2 kg** (`body.mass = 0.2f`), and its center of mass shifts to `Vector3.back`.
- **Reason:** A 150 kg iron mass resting at the extreme forward tip of a ship's bow would induce severe bow-down trim and cause parasitic pitch oscillation in wave troughs.

---

## 5. Practical Modding Conclusions

1. **Custom Anchor Grip Strength:** Do not attempt to increase anchor holding power by increasing `Rigidbody.mass`—breakout threshold `unsetResistance` is calculated from **ship mass** (`connectedBody.mass * 3.3f`). To create a high-grip anchor, patch `rope.resistanceMult`.
2. **The 9-Meter Minimum Scope:** When designing shallow-water ports or custom mooring areas, remember that an anchor will never set if seabed depth plus bow height require less than **9 meters** of rope scope.
3. **Stowed Mass Mitigation:** When developing heavy deck equipment or custom winches, adopt vanilla's stowed mass reduction trick (`body.mass = 0.2f`) when items are locked in travel positions to avoid upsetting ship trim.
