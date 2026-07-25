# 52. Unity layers and collision matrix: items, boat, player

Breakdown of Unity layers and collision matrix — answer to request E2. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to note 51 (boat collider topology).

## Layers used by the game

Layers are set via `gameObject.layer = N` in code (and in scene prefabs):

| Number | Usage | Code / class |
|:--:|-----------|------------|
| 0 | Default (world, terrain, dock walls, etc.) | Unity default |
| 2 | **IgnoreRaycast** — twin ItemRigidbody, visual ShipItem (when not in inventory/on UI) | `ItemRigidbody.Start()`: `gameObject.layer = 2`; `ShipItem.EnterInventorySlot`: `layer = 2` |
| 5 | **UI** — visual ShipItem in inventory/on UI | `ItemRigidbody.EnterInventorySlot`: `item.gameObject.layer = 5` (and all children) |
| 8 | **Player** (character capsule) | Unity built-in or prefab |
| 12 | **HullPlayerCollider** — duplicate mesh-collider of the deck | `HullPlayerCollider.Start()`: `hullCol.gameObject.layer = 12` |
| 14 | Terrain/OceanBottom (as in Anchor.IsTouchingGround) | Prefabs, tag `Terrain`/`OceanBottom` |
| 26 | "In crate/special" — item in CrateInventory | `ShipItem.ExtraFixedUpdate`: `layer != 26` — check; item in crate is set to 26 |
| 23 | Map (MapChart when opened) | `MapChart`: when opening map, GO is set to layer 23 |
| 13 | Boat (tag `Boat`) | Boat prefab / boat colliders |
| ? | Water/Ocean | `Ocean` renderer |

> **Exact layer name by number** is not available from decompilation (LayerMask.NameToLayer is a runtime API). The numbers given are from direct `gameObject.layer = N` assignments in code. Layer names are defined in the Unity project (Tag Manager) and are not serialized in Assembly-CSharp.dll.

## `ItemRigidbody` — layer switching

| State | Twin layer (ItemRigidbody GO) | Visual layer (ShipItem GO) | Code |
|-----------|:--:|:--:|------|
| In world (free, `sold`) | **2** (IgnoreRaycast) | **2** | `Start()`, `ResetPos()` |
| In inventory (`EnterInventorySlot`) | unchanged | **5** (UI) | `EnterInventorySlot`: `item.gameObject.layer = 5` (children too) |
| From inventory (`ExitInventorySlot`) | 2 | **2** | `ExitInventorySlot`: `item.gameObject.layer = 2` (children too) |
| In crate | 2 | **26** | crate prefab layer = 26 |
| PickUp (`GoPointer.PickUpItem`) | **0** (Default) | — | `GoPointer`: `heldItem.gameObject.layer = 0` (visual on Default for raycast/interact) |
| Drop (`GoPointer.DropItem`) | 2 | 2 | `DropItem`: `heldItem.gameObject.layer = 0`→2? (in LateUpdate twin position = visual) |

> **Critical:** on `PickUp` the visual ShipItem is set to **layer 0 (Default)**. The twin remains on **layer 2 (IgnoreRaycast)**. This means the **twin colliders DO NOT physically collide with regular world objects** (IgnoreRaycast is excluded from the physics subsystem by default in Unity). A vanilla held item never physically touches the boat!

## GoPointer raycast mask

`GoPointer.LateUpdate` uses raycast with mask `LayerMask.op_Implicit(-604165)`.

Binary representation of `-604165` (32 bit): this is the inverse mask. `~(-604165) = 604164` in binary = which layers the raycast **does not** check, `-604165` = which it **does** check.

```
604164 = 0b 0000 1001 0011 0001 0000 0100
         layers: 2, 8, 12, 16, 19, 24 (approximately — needs runtime verification)
```

Exact decoding requires runtime (LayerMask.NameToLayer), but the principle: GoPointer raycast "sees" most physical layers and **ignores** UI and IgnoreRaycast.

> Layer 2 (IgnoreRaycast) is **excluded from GoPointer raycast** — twin-item is not "clickable" via raycast (pickup goes via visual, layer 0 or 5).

## Collision matrix — key pairs

Unity `Physics.GetIgnoreLayerCollision(a,b)` is set in the project (Physics Settings) and **is not visible from decompilation**. But it can be reconstructed from observations and code:

| Pair | Collide? | Reasoning |
|------|:--:|------------|
| **twin (2) ↔ boat (13?)** | **YES** | twin ItemRigidbody on layer 2 is IgnoreRaycast, but **not** IgnoreCollision. IgnoreRaycast only affects raycast, NOT physical collisions. The boat's CapsuleCollider will collide with twin on layer 2. |
| **twin (2) ↔ player (8)** | **YES** (if enabled in Physics Settings) | CharacterController interacts with most layers |
| **player (8) ↔ boat (12/walkCollider)** | **YES** | Player walks on deck (layer 12 = HullPlayerCollider) — confirmed by code |
| **twin (2) ↔ water** | **YES** | `SimpleFloatingObject` works via Ocean — twin is physically in water |
| **visual (0) ↔ boat** | **YES when held** | Visual at held on layer 0 — Default, collides with boat |

> **IgnoreRaycast (layer 2)** does not make an object "non-physical" — it only excludes it from raycast. Collisions (OnCollisionEnter/Exit, OnTriggerEnter/Exit) **work** on layer 2. This means: twin ItemRigidbody (layer 2) **physically collides with the boat's CapsuleCollider**.

## Why "player inside boat collision, but crate on its surface"

1. Player (`CharacterController`, layer 8) walks on **walkCollider** (layer 12) — duplicate mesh of the deck, whose surface coincides with the deck.
2. But **boat's CapsuleCollider** is a simplified volume (capsule), its surface is **above the deck** (the capsule encloses the entire hull including masts/bulwarks, and its top surface can be at the level of the gunwale or above).
3. Player **does not collide with CapsuleCollider** (or layer 8↔boat layer is configured so player ignores CapsuleCollider, walking on layer 12).
4. Item twin (layer 2) **collides with CapsuleCollider** (2↔boat = Yes). And "lies" on the capsule surface — which is **above the deck**.
5. In vanilla, twin on layer 2 + `isTrigger=true` on twin colliders → **no physical collisions** (trigger doesn't call OnCollisionEnter). Mod makes twin colliders non-trigger → physical collision with CapsuleCollider → item "stands in the air".

## Practical conclusions

1. **Layer 2 (IgnoreRaycast)** ≠ "no collisions". It's only a raycast exclusion. Physics on layer 2 works.
2. **Vanilla held item** doesn't collide with the boat because twin colliders are **isTrigger=true** → no physical collisions.
3. **Mod with non-trigger twin** = item on layer 2 physically collides with boat's CapsuleCollider → item "stands in the air above the hold".
4. Solution: `Physics.IgnoreLayerCollision(twinLayer, boatLayer, true)` for twin while held, or exclude boat's CapsuleCollider from item collisions via Harmony patch.
5. Visual layer when held = **0 (Default)** — this is the "visible" item for raycast/interact; twin when held = **2 (IgnoreRaycast)** — "not raycasted" but **physically active**.
