# 60. All readers of PickupableItem.held, default Throw key binding

Complete table of all places where `PickupableItem.held` is read — answer to requests A7, A8. Related to notes 57 (DropItem), 58 (clickability).

## A7. ALL readers of PickupableItem.held — complete table

| Class | Method | What it does when held!=null | What it does on held→null transition |
|-------|-------|--------------------------|----------------------------------|
| **GoPointer** | `PickUpItem()` | `heldItem.held = this` | — |
| **GoPointer** | `DropItem()` | — | `heldItem.held = null` |
| **GoPointer** | `LateUpdate` (throw) | charge throw power | `currentThrowPower = 0f` |
| **GoPointer** | `LateUpdate` (positioning) | drives visual position | — (position frozen) |
| **GoPointer** | `DoRaycast()` | self-look prevention | — |
| **GoPointer** | `LateUpdate` (lookUI) | ShowHoldText | ClearText |
| **ItemRigidbody** | `FixedUpdate` (position sync) | twin = position slave (teleport) | twin = position master (visual follows twin) |
| **ItemRigidbody** | `FixedUpdate` (isKinematic) | `flag2=true → isKinematic=true` | `flag2=false → isKinematic=false` (dynamic) |
| **ItemRigidbody** | `FixedUpdate` (isTrigger) | colliders isTrigger=true | colliders isTrigger=false (solid) |
| **ItemRigidbody** | `FixedUpdate` (dynamicColTimer) | `SetDynamicColTimer() → 6f` | countdown collision mode |
| **ShipItem** | `OnPickup()` | `wallAttachment → attached=false` | — |
| **ShipItem** | `OnDrop()` | — | `wallAttachment → attached=true, twin snap` |
| **ShipItem** | `Update()` (wall raycast) | raycast forward 1.3m | `inRangeOfWall=false` |
| **ShipItem** | `ExtraFixedUpdate()` | — | `!held && !attached → ExitBoat possible` |
| **PickupableItemCollisionChecker** | `Update()` | `!held → collisions=0` | `held → check collisions` |
| **WorldItemSpawner** | `Update()` | `item.held → respawn cooldown, debugForceKinematic=false, item=null` | — |
| **MouthCol** | `Update()` | `currentFood.held → eat audio + EatFood()` | `eatAudio.Stop()` |
| **Cleaner** | `LateUpdate()` | `item.held → skinnedBroom, sweep, clean` | `staticBroom, cleanCooldown reset` |
| **CookableFood** | cook trigger | `item.held → cook on stove` | — |
| **StoveFuel** | `Update()` | `!item.held → auto-insert fuel` | `item.held → don't insert` |
| **FishingRodFish** | `Update()` | `rod.held && InWater → catch` | — |
| **Anchor** | `FixedUpdate()` | `held → DropItem()` | — |
| **PickupableBoatMooringRope** | `Update()` | `held && dist>max → DropItem()` | — |

### Key consequences on held→null transition

| System | What starts working | Critical for mod? |
|--------|--------------------|:--:|
| ItemRigidbody position sync | twin = **position master** (visual follows twin) | Yes |
| ItemRigidbody isKinematic | twin → **dynamic** | Yes |
| ItemRigidbody colliders | twin → **isTrigger=false** (solid) | Yes |
| ShipItem ExtraFixedUpdate | ExitBoat possible | Yes |
| CollisionChecker | collisions=0 | No |

## A8. Default Throw/drop key binding

### GoPointer.Start() — verbatim

```csharp
if (Application.isEditor) { mainKey = "r"; altKey = "f"; }
else { mainKey = "f"; altKey = "r"; }
```

**Default keys (build):** `mainKey = "f"` (interact/drop), `altKey = "r"` (alt activate/throw).

### InputName mapping (GameInput.cs not decompiled — see note 24)

| InputName | Purpose | Default key (build) | VR |
|-----------|---------|:--:|:--:|
| 8 | Main interact/drop | `"f"` | Oculus trigger |
| 9 | Alt activate | `"r"` | X/A buttons |
| 10 | Throw/drop dedicated | **unknown KeyCode** (runtime only) | **desktop-only** |

> **InputName 10 KeyCode** — requires runtime check (BepInEx log). Not visible from decompilation.

**Settings.autoThrow** — if enabled, InputName 8 also charges throw (same key becomes throw+charge on hold).

## Practical conclusions for modders

1. **held→null triggers massive state change:** twin dynamic, solid colliders, position master, ExitBoat possible. All via ItemRigidbody.FixedUpdate.
2. **held!=null → twin frozen** (kinematic, position slave, trigger). Mod bypassing held must handle layer, isTrigger, isKinematic.
3. **InputName 10 = throw key** — desktop-only, KeyCode unknown from decompilation. Separate from InputName 8 (interact) and 9 (alt).
4. **Settings.autoThrow** — InputName 8 also charges throw when enabled.
