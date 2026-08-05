# 77. NPC Artificial Intelligence: Navigation, Collision Avoidance, Waypoint Graphs, and Daily Schedules

A complete breakdown of the artificial intelligence and vessel control algorithms for non-player character (NPC) ships in Sailwind v0.38 (`Assembly-CSharp.dll`). This note supplements the general overview of world population in [Note 20](20-npcs-world-population.md) and reveals the exact physical navigation equations, sail trimming logic, and waypoint graph architecture.

---

## 1. NPC Boat Control Architecture (`NPCBoatController`)

`NPCBoatController` is the central component for ambient background vessels sailing across the world. Unlike the player's ship—which relies on aerodynamic lift and drag calculations (`Sail`, [Note 17](17-wind-and-sails.md)) and rudder hydrodynamics (`Rudder`, [Note 80](80-rudder-hydrodynamics-centering-force-and-steering-torque.md))—NPC boats use a **simplified kinematic-dynamic model** that ensures stable and predictable locomotion.

### Component Fields

| Field | Type | Description |
|---|---|---|
| `speed` | `float` | Base linear movement speed of the boat. |
| `turnSpeed` | `float` | Turning rate (torque multiplier). |
| `sailSpeed` | `float` | Trim rate for adjusting sail angles. |
| `sailResistance` | `float` | Resistance threshold for sail auto-trimming. |
| `sailAngleControllers` | `RopeControllerSailAngle[]` | Array of sheet controllers (governing sail angle to wind). |
| `sailReefControllers` | `RopeControllerSailReef[]` | Array of halyard/reefing controllers (governing sail unroll). |
| `currentTarget` | `Transform` | Current destination waypoint (`NPCBoatWaypoint`). |
| `currentTargetIndex` | `int` | Index of the current waypoint in `NPCBoatWaypointManager`. |
| `currentDock` | `Transform` | Current docked waypoint when parked. |
| `currentDockIndex` | `int` | Index of the docked waypoint. |
| `parkedTimer` | `float` | Dwell timer while docked/parked (in in-game hours). |
| `horizon` | `BoatHorizon` | Proximity optimizer that enables/disables simulation based on player distance. |
| `otherBoatInRange` | `bool` | Obstacle detection flag when another boat is within 15 meters. |

---

## 2. Movement and Physics Algorithms in `FixedUpdate()`

Every physics step (`FixedUpdate()`, ~45.5 Hz), `NPCBoatController` performs optimization checks and calculates navigation forces:

```csharp
private void FixedUpdate()
{
    if (!horizon.closeToPlayer)
        return;

    if (boatColCheckTimer <= 0f)
        CheckOtherBoatCol();
    else
        boatColCheckTimer -= Time.deltaTime;

    if (otherBoatInRange)
        return;

    if (currentTarget != null)
    {
        AddForceTowards(currentTarget);
        AddRotationTowards(currentTarget);
    }

    if (currentDock != null)
    {
        AddForceTowards(currentDock);
        transform.rotation = Quaternion.Lerp(transform.rotation, currentDock.rotation, 0.005f);
        parkedTimer += Time.deltaTime * Sun.sun.timescale;
        if (parkedTimer > 1f)
        {
            parkedTimer = 0f;
            NPCBoatWaypoint waypoint = currentDock.GetComponent<NPCBoatWaypoint>();
            if (waypoint != null && waypoint.GetNextDestination() != null)
            {
                currentTarget = waypoint.GetNextDestination().transform;
                currentTargetIndex = waypoint.GetNextDestination().index;
                currentDock = null;
                currentDockIndex = -1;
            }
        }
    }
}
```

### 2.1. Linear Propulsion (`AddForceTowards`)

NPC boats do not evaluate wind lift vectors on sails to generate propulsion. Instead, they apply a direct linear force toward the `target`:

$$\vec{F}_{\text{linear}} = \vec{u}_{\text{target}} \cdot \left(\text{speed} + \|\vec{v}_{\text{wind}}\| \cdot \text{speed} \cdot 0.05\right) \cdot m_{\text{boat}}$$

Where:
- $\vec{u}_{\text{target}}$ is the normalized direction vector toward the waypoint (`(target.position - transform.position).normalized`).
- $\|\vec{v}_{\text{wind}}\|$ is the current wind velocity magnitude (`Wind.currentWind.magnitude`).
- $m_{\text{boat}}$ is the vessel's rigidbody mass (`rigidbody.mass`).

> **Key Takeaway:** Because $\vec{F}_{\text{linear}}$ points directly at the waypoint, NPC boats can sail **directly into the wind (in irons)** without tacking. Wind strength simply provides a 5% speed boost per unit of wind magnitude.

### 2.2. Heading Control (`AddRotationTowards`)

Yaw control also bypasses rudder hydrodynamics:

```csharp
Vector3 dir = (target.position - transform.position).normalized;
float angle = Vector3.SignedAngle(transform.forward, dir, Vector3.up);
if (angle < 0f)
    rigidbody.AddTorque(Vector3.up * -turnSpeed * rigidbody.mass * 20f);
else if (angle > 0f)
    rigidbody.AddTorque(Vector3.up * turnSpeed * rigidbody.mass * 20f);
```

Turning torque is proportional solely to the sign of the heading error (`SignedAngle`) and boat mass ($20 \cdot m_{\text{boat}} \cdot \text{turnSpeed}$).

---

## 3. Collision Avoidance System (`CheckOtherBoatCol`)

To prevent collisions between ambient NPCs and the player, boats perform a spherical physics query:

```csharp
private void CheckOtherBoatCol()
{
    boatColCheckTimer = Random.Range(0.5f, 1.5f);
    otherBoatInRange = false;
    Collider[] colliders = Physics.OverlapSphere(transform.position, 15f);
    foreach (Collider col in colliders)
    {
        if (col.CompareTag("Boat") && col != this.col)
        {
            otherBoatInRange = true;
            break;
        }
    }
}
```

### Analysis
1. **Polling Interval:** The `Physics.OverlapSphere` query is executed asynchronously with a randomized delay between **0.5 and 1.5 seconds**, reducing CPU overhead.
2. **Safety Radius:** The avoidance detection zone is **15 meters**.
3. **Tag Filtering (`"Boat"`):** If any collider tagged `"Boat"` (other than the boat's own collider) enters the radius, `otherBoatInRange = true` immediately aborts `FixedUpdate()`. The NPC boat ceases applying propulsion and steering torque.
4. **Modding Pitfall:** If a mod creates custom objects or structures tagged `"Boat"` near NPC waypoints, passing NPC vessels will halt 15 meters away indefinitely, causing traffic jams.

---

## 4. Automatic Sail Trimming (`Update`)

Although sails do not physically propel NPC vessels, the AI actively animates their visual state via `RopeControllerSailReef` (reefing) and `RopeControllerSailAngle` (trimming):

```csharp
// 1. Reefing / Unrolling Control
if (currentTarget != null)
{
    foreach (RopeControllerSailReef reef in sailReefControllers)
        reef.currentLength = Mathf.Min(1f, reef.currentLength + 0.15f * Time.deltaTime);
}
else
{
    foreach (RopeControllerSailReef reef in sailReefControllers)
        reef.currentLength = Mathf.Max(0f, reef.currentLength - 0.15f * Time.deltaTime);
}

// 2. Sail Angle Trimming
if (currentTarget != null)
{
    foreach (RopeController angleCol in sailAngleControllers)
    {
        angleCol.changed = true;
        if (angleCol.currentResistance > Wind.currentWind.magnitude)
            angleCol.currentLength = Mathf.Min(1f, angleCol.currentLength + sailSpeed * Time.deltaTime * 0.05f);
        else
            angleCol.currentLength = Mathf.Max(0f, angleCol.currentLength - sailSpeed * Time.deltaTime * 0.05f);
    }
}
```

### Trimming Logic
- **Unroll Rate:** While navigating (`currentTarget != null`), sails unroll at `0.15 / sec` (taking **6.67 seconds** from furled to full sail). When docked (`currentTarget == null`), sails furl at the same rate.
- **Wind Adaptation:** The AI compares rope tension (`currentResistance`) against `Wind.currentWind.magnitude`. If resistance exceeds wind speed, it loosens the sheet (`currentLength += ...`); otherwise, it sheets in.

---

## 5. Waypoint Navigation Graphs (`NPCBoatWaypoint`)

The NPC routing network is a directed graph managed by the singleton `NPCBoatWaypointManager.instance`.

```
[NPCBoatWaypoint 0: navigationWaypoint=true]
         │
         ▼ (Random selection from destinations[])
[NPCBoatWaypoint 3: navigationWaypoint=true]
         │
         ▼
[NPCBoatWaypoint 7: navigationWaypoint=false (Dock/Port)]
         │
         ├──► Dwell state: parkedTimer > 1.0 (in hours × timescale)
         │
         ▼ (Departure after timer expires)
[NPCBoatWaypoint 12: navigationWaypoint=true]
```

### Waypoint Trigger Logic (`OnTriggerEnter`)

When a boat reaches a waypoint (`other.transform == currentTarget`), its transition depends on the waypoint type:

| Waypoint Type | Field State | NPC Boat Behavior |
|---|---|---|
| **Navigation** | `navigationWaypoint == true` | Immediately selects the next waypoint from `destinations[]` via `Random.Range(0, destinations.Length)`. |
| **Dock / Park** | `navigationWaypoint == false` | Enters docked state: `currentDock = currentTarget; currentTarget = null;`. Begins incrementing `parkedTimer`. |

---

## 6. Specialized AI Vessels

### 6.1. Fishing Boats (`NPCFishingBoat`)

`NPCFishingBoat` extends `NPCBoatController`, subjecting the boat to a daily routine based on local solar time (`Sun.sun.localTime`):

```csharp
private void Update()
{
    bool isFishingTime = (Sun.sun.localTime > 5.5f && Sun.sun.localTime < 9.5f) ||
                         (Sun.sun.localTime > 13.5f && Sun.sun.localTime < 17.5f);
    if (isFishingTime && !goingFishing)
        GoFishing();
    else if (!isFishingTime && goingFishing)
        GoHome();
}
```

- **Morning Shift:** `05:30 — 09:30`.
- **Afternoon Shift:** `13:30 — 17:30`.
- **Coordinate Synchronization:** Target fishing locations (`target`) and home positions are stored in real-world coordinates (`RealPos`) and converted to local shifting coordinates (`ShiftingPos`) via `FloatingOriginManager.instance.RealPosToShiftingPos(realPos)` to prevent drift across floating origin resets.

### 6.2. Simulated Trader Boats (`TraderBoat`)

`TraderBoat` ([Note 13](13-economy-markets-currency.md)) is not a physics vessel with a `Rigidbody`. It is a background economic agent connecting island markets:

```csharp
private void Update()
{
    if ((!EconomyUI.instance.uiActive || Application.isEditor) && !GameState.currentShipyard)
    {
        if (waitTime > 0f)
            waitTime -= Time.deltaTime * Sun.sun.timescale * 100f;
        else if (currentDestination != null)
            EnterIsland();
        else if (currentIslandMarket != null)
            LeaveIsland();

        if (lastIslandMarket != null && currentDestination != null)
            transform.position = Vector3.Lerp(currentDestination.transform.position, lastIslandMarket.transform.position, waitTime / currentTripTime);
    }
}
```

- **Accelerated Timing:** The `waitTime` countdown ticks **100 times faster** than game time (`timescale * 100f`).
- **UI Freeze:** Movement and market transactions pause whenever the player opens `EconomyUI` or enters a shipyard (`currentShipyard`), ensuring prices do not shift mid-trade.
- **Interpolation:** World position is linearly interpolated (`Vector3.Lerp`) between origin and destination island markets.

---

## 7. Practical Modding Conclusions

1. **Creating Custom Routes:** To add new NPC shipping lanes, create GameObjects with `NPCBoatWaypoint`, register them in `NPCBoatWaypointManager.instance`, and populate their `destinations[]` arrays.
2. **Upwind Locomotion:** NPC vessels do not require favorable wind angles; when balancing custom NPC boat speeds, remember that `speed + wind * speed * 0.05` provides constant thrust in any direction.
3. **`"Boat"` Tag Safety:** Avoid tagging non-ship water objects (such as large buoys or floating docks) with `"Boat"`, as nearby NPC vessels will freeze within 15 meters of them.
4. **Intercepting Trade Schedules:** The complete list of active trading vessels is accessible via `TraderBoat.traderBoats`. Modifying `carriedGoods` or `carriedPriceReports` allows mods to influence inter-island markets and tavern rumors ([Note 78](78-dialogues-rumors-bribery-and-contraband-customs.md)).
5. **Save State (`NPCBoatData`):** Only `currentDock`, `currentTarget` indices, and `parkedTimer` are persisted. Custom runtime modifications to position or velocity are reset on load to the prefab default and current waypoint coordinates.
