# 76. `DockPushCol`: dock-collider layer and water-adjacent physics

An examination of `DockPushCol` and `GPButtonBoatPushCol` in Sailwind v0.38. This matters to item/cargo mods because a dock collider can use the same Unity layer as an item physics twin.

Related to [52](52-layers-collision-matrix-items-boat-player.md) and [54](54-go-pointer-big-item-decollision.md).

## `DockPushCol : GoPointerButton`

`DockPushCol` is an interactive dock-side collider used to push a moored boat. In
`Awake()`, it remembers its prefab layer and collider:

```csharp
initialLayer = gameObject.layer;
col = GetComponent<Collider>();
```

### Critical layer switch

Every `ExtraFixedUpdate()` it runs:

```csharp
if (GameState.currentBoat != null)
    gameObject.layer = initialLayer;

if (GameState.currentBoat == null)
    gameObject.layer = 2;
```

Therefore:

| Player state | `DockPushCol` layer |
|---|---:|
| Player is on a boat (`GameState.currentBoat != null`) | original prefab layer |
| Player is not on a boat | **2** |

Layer 2 is also used by item physics twins (`ItemRigidbody.Start()`). Unity's
`IgnoreRaycast` convention does not mean "no physics": it only affects raycast
behavior. The actual collision matrix is defined in Physics Settings.

> A static dock collider can therefore occupy the same layer 2 as a solid item
> twin. A large crate intersecting dock geometry near the waterline can enter a
> normal PhysX penetration solve even when a mod expects layer 2 to be
> raycast-only.

## Boat pushing

`DockPushCol` does not push items. It acts only while clicked and only when
`GameState.currentBoat` exists:

```csharp
Rigidbody boat = GameState.currentBoat.parent.GetComponent<Rigidbody>();
Vector3 push = (pushForceMult * boat.mass * pointer.forward
              + upForceMult * boat.mass * Vector3.up) / max(boat.velocity.magnitude, 1f);
boat.AddForceAtPosition(push, pointer.position + Vector3.up * verticalOffset);
```

C# defaults:

| Field | Value |
|---|---:|
| `pushForceMult` | -0.55 |
| `upForceMult` | 0 (non-serialized default) |
| `verticalOffset` | 0 (non-serialized default) |

If a dock collider physically launches a crate, that is not script-side
`DockPushCol.AddForceAtPosition`; it is a Unity contact/penetration solve
between colliders.

## `GPButtonBoatPushCol`: a different case

`GPButtonBoatPushCol` is located on a boat and manages its collider differently:

```csharp
if (col.enabled && player is on this boat)
    col.enabled = false;
if (!col.enabled && player is not on this boat)
    col.enabled = true;
if (col.enabled && (PlayerSwimming.swimming || !Refs.charController.isGrounded))
    col.enabled = false;
```

The boat push collider is disabled while the player swims/is not grounded.
`DockPushCol` has no equivalent disable path; it only switches layer from
`GameState.currentBoat`.

## Practical implications for modders

1. Do not infer collider safety from layer 2 alone: dock collider and item twin
   can share that layer.
2. `DockPushCol` does not force items. Investigate PhysX contact state, not
   `DockPushCol.AddForceAtPosition`, for a catastrophic item launch near dock.
3. For custom cargo physics, log collider name, tag, layer,
   `attachedRigidbody`, contact normal, relative velocity, and penetration depth.
4. Dock/crate contact can depend on whether the player is embarked:
   `DockPushCol` restores its original layer only with `GameState.currentBoat`.
5. Do not globally disable dock colliders: vanilla interaction/push flow needs
   them. Use a per-body contact policy or predictable depenetration logic.
