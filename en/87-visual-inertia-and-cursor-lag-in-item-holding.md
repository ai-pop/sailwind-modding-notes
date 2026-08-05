# 87. Held Item Inertia: Why Boxes 'Lag Behind the Cursor' and How Vanilla Grip Works

A technical investigation into user feedback on item physics mods (reviews on SailwindItemPhysics: *"boxes kind of lag behind the cursor ... move around a little slower than you move your mouse"*). This note explains how vanilla 1:1 item holding operates, why introducing physical mass or visual inertia creates perceived input lag, and how to balance these parameters.

Related to [Note 03](03-gopointer-input-system.md), [Note 47](47-item-holding-pickup-flow.md), and [Note 59](59-velocity-zeroing-release-frame-pose.md).

---

## 1. How Vanilla Item Holding Operates (`GoPointerMovement`)

In vanilla Sailwind, a held item possesses no inertia, damping, or apparent mass in the player's hands. Item holding relies on **instantaneous 1:1 tracking**:

```csharp
// Every LateUpdate() inside GoPointerMovement:
item.visual.position = pointerAimPos;
item.visual.rotation = pointerAimRot;
```

1. **Zero Input Latency:** The visual GameObject (`ShipItem`) is hard-parented/snapped directly to the camera's pointer aim position (`GoPointer.instance`).
2. **Instantaneous Tracking:** When the user flicks the mouse, the crate translates and rotates across the screen in a single render frame without smoothing or inertia.
3. **Slaved Twin Collider:** Every physics step (`FixedUpdate()`), the physical twin (`ItemRigidbody`) is kinematically teleported to match the visual GameObject's pose.

---

## 2. Why Physics Mods Cause Cursor Lag

Item physics mods (such as SailwindItemPhysics) introduce two hold modes that intentionally break vanilla's 1:1 instantaneous tracking:

```
[ Player rotates camera / moves pointer ]
                        │
                        ├─────────────────────────────────┐
                        ▼                                 ▼
           [ Physics-In-Hands Mode ]              [ Visual Imitation Mode ]
          (HoldPhysicsInHands = true)            (HoldImitationOnBoat = true)
                        │                                 │
                        ▼                                 ▼
             HingeJoint / PD Spring              Lerp pose interpolation
             F = k * error - d * vel            pos = Lerp(current, aim, speed)
                        │                                 │
                        └─────────────────┬───────────────┘
                                          ▼
                         [ PERCEIVED "CURSOR LAG" ]
                         Heavy boxes lag behind cursor aim,
                         jarring players accustomed to vanilla
```

### 2.1. Physics-In-Hands Mode (`HoldPhysicsInHands`)
When an item is driven by a Proportional-Derivative (PD) spring in physical space, its acceleration is governed by Newton's second law ($F = ma$):

$$\vec{F}_{\text{spring}} = k \cdot (\vec{p}_{\text{aim}} - \vec{p}_{\text{item}}) - d \cdot \vec{v}_{\text{item}}$$

In default configurations:
- `HoldSpringStiffness = 40f` ($k$)
- `HoldSpringDamping = 15f` ($d$)
- For a heavy crate weighing **30–50 kg**, a stiffness of `40f` cannot accelerate the mass instantaneously. The crate stretches the spring and trails behind the pointer by **100–250 ms**.

### 2.2. Visual Imitation Mode (`HoldImitationOnBoat`)
To imitate mass on a boat even when physics-in-hands is disabled, the mod applies visual smoothing:
- Configuration: `HoldImitationOnBoat = true`, `HoldImitationStrength = 1f`.
- The script lerp-interpolates the visual item's position toward the pointer aim. Players accustomed to vanilla 1:1 responsiveness perceive this smoothing not as "heft," but as unpleasant **mouse input lag**.

---

## 3. Practical Modding Conclusions

1. **Opt-In Visual Inertia:** "Weighty feel" and input responsiveness are subjective preferences. Always expose visual imitation in config and **disable it by default** (`HoldImitationOnBoat = false`), allowing players to opt into inertia.
2. **Mass-Scaled Spring Stiffness:** When using physics-in-hands (`HoldPhysicsInHands`), scale spring stiffness with item mass so that trailing latency never exceeds **30–50 ms**:
   ```csharp
   float effectiveStiffness = Cfg.HoldSpringStiffness * Mathf.Sqrt(rb.mass);
   ```
3. **Higher Baseline Stiffness:** To eliminate user complaints about "rubbery" or "laggy" crates, raise baseline default configuration parameters:
   - `HoldSpringStiffness`: recommended minimum **120f** (up from 40f).
   - `HoldSpringDamping`: recommended minimum **25f** (up from 15f).
4. **Instantaneous Initial Snap:** On the frame an item is picked up (`OnPickup`), explicitly snap the body to the pointer position (`transform.position = aimPos`) to prevent a visible spring-stretch artifact from its old world position.
