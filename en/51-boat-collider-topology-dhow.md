# 51. Boat collider topology (dhow): embark, walk, capsule, hull

Breakdown of the boat's collider tree and their purposes — answer to request E1. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 14 (buoyancy), 16 (ShipItem/twin), 47 (holding).

## `BoatEmbarkCollider` — main embark/disembark trigger

`BoatEmbarkCollider : MonoBehaviour` — component on a **child GO of the boat**, which serves as `OnTriggerEnter/Exit` trigger by tag `EmbarkCol` for items (`ShipItem`) and by tag `MainCamera` for the player (`BoatEmbarkTrigger`).

### Key fields
| Field | Type | Content |
|------|-----|-----------|
| `walkCollider` | `Transform` | **Reference to the boat's walk-collider** — Transform of a separate GO (BoxCollider/MeshCollider?), which the player walks on and the item twin reparents to on `EnterBoat`. |
| `damage` | `BoatDamage` | Obtained in `Awake()`: `transform.parent.parent.GetComponent<BoatDamage>()`. → `BoatEmbarkCollider` is on a **nested child** (parent = mesh group, parent.parent = boat root). |
| `col` | `Collider` | Collider of this GO (the one with tag `EmbarkCol`). |
| `capsuleCol` | `CapsuleCollider` | **CapsuleCollider on the boat root** (`transform.parent.parent.GetComponent<CapsuleCollider>()`). This is the main "hull volume". |
| `initialRadius` | `float` | Original radius of the boat's `CapsuleCollider`. |
| `embarkAllowed` | `bool` | Flag: if boat is sunk (`damage.sunk`) → trigger shifts +100 Y (disables embark). |

### `ToggleBoatCapsuleCol(bool newState)`
- `newState = true` → `capsuleCol.radius = initialRadius` (restore);
- `newState = false` → `capsuleCol.radius = 0` (disable boat CapsuleCollider).

Used when sinking / restoring the boat.

### `LateUpdate` — sinking teleport
- If `sunk && embarkAllowed`: `transform.Translate(up * 100, Space.World)` → trigger flies up, `embarkAllowed = false`.
- If `!sunk && !embarkAllowed`: `transform.localPosition = initialPos` → trigger returns.

## `CapsuleCollider` on the boat root — main hull volume

`BoatEmbarkCollider.capsuleCol` = `CapsuleCollider` on the **root GO of the boat** (transform.parent.parent). This is the "simplified volume" that:

- Encloses the **entire boat** as a capsule (with initialRadius, which can be ~1.5–3 m, and height ~8–12 m).
- Used by `Buoyancy` for blob-buoyancy calculations (grid of `SlicesX × SlicesZ` across this CapsuleCollider's dimensions, note 14).
- The player **ignores** it for walking (player walks on separate `walkCollider`, layer 12).
- **But items (twin ItemRigidbody, layer 2) DO NOT ignore it** — the boat's CapsuleCollider is on a layer that collides with layer 2 (ItemRigidbody), and this explains why a barrel "stands in the air" above the hold: its twin physically interacts with the surface of this capsule, not the deck.

> **Critical conclusion:** The boat's `CapsuleCollider` is a **simplified hull volume** whose surface is above the deck. The player doesn't "see" it (walks on `walkCollider`/layer 12), but items with physics twin (layer 2) collide with it. This is the source of the "invisible wall above the hold" for a mod with a solid collider on the held item.

## `walkCollider` — walking surface

`BoatEmbarkCollider.walkCollider` — Transform of a separate GameObject with a deck collider (BoxCollider or MeshCollider). This GO:

- **Layer 12** (per `HullPlayerCollider.Start()` — creates a child `hull player collider` on layer 12).
- The player's `CharacterController` walks on it (player capsule collides with layer 12).
- On `ItemRigidbody.EnterBoat()` the twin **reparents** to `walkCollider` (`transform.parent = item.currentWalkCol`) and is positioned via `MoveRigidbodyToWalkCol`.

`HullPlayerCollider : MonoBehaviour` (`[RequireComponent(MeshCollider)]`) in `Start()`:
- Creates a **child GO** "hull player collider" with MeshCollider (copy of sharedMesh), layer **12**.
- **Disables** its own MeshCollider on the host-GO (`enabled = false`).
- The child GO is **not a child transform of the boat** (`parent = null`), but its position/rotation is synchronized every `Update()`.
- Also adds `CleanableObjectCollider` (for hull cleaning).

> `HullPlayerCollider` creates a **separate physical duplicate mesh-collider** for walking/cleaning on layer 12, disabling the original mesh on the boat. But **the boat's CapsuleCollider remains active** (on the boat's layer) and collides with items.

## `BoatEmbarkTrigger` — player embark trigger

Separate component `BoatEmbarkTrigger : MonoBehaviour` on another child GO of the boat:

- `OnTriggerEnter(Collider other)`: if `other.CompareTag("MainCamera")` → `EnterBoat()` — reparents "Player Controller" onto the boat.
- `OnTriggerExit(Collider other)`: if `other.CompareTag("MainCamera")` → `ExitBoat()` — detaches the player from the boat.

This is **different** volume from `BoatEmbarkCollider` (for items). Both have tag `EmbarkCol`.

## GO hierarchy of the boat (reconstructed dhow)

```
Dhow (root GO)                                  ← Rigidbody + CapsuleCollider (hull-capsule) + BoatDamage + Buoyancy + BoatMass + BoatProbes + ShiftingRigidbody
 ├── BoatProbes child                           ← (blob-buoyancy points)
 ├── BoatKeel                                   ← centerOfMass
 ├── CapsuleCollider (on root)                  ← main volume, radius ≈ initialRadius, collides with twin-items!
 ├── Mesh group (parent of BoatEmbarkCollider)  ← MeshRenderer of hull
 │    ├── BoatEmbarkCollider                    ← Collider with tag EmbarkCol (for items), reference to walkCollider, reference to CapsuleCollider root
 │    ├── HullPlayerCollider                    ← MeshCollider (disabled), creates "hull player collider" (layer 12) duplicate
 │    └── ... (walkCollider?)                   ← walkCollider = Transform of separate deck GO (BoxCollider/MeshCollider, layer 12)
 ├── BoatEmbarkTrigger                          ← Collider with tag EmbarkCol (for player, MainCamera)
 ├── BoatImpactSoundsCollider                   ← OnCollisionEnter → BoatImpactSounds.Impact
 ├── BoatMooringRopes                           ← mooring ropes
 ├── ... (Deck objects, masts, sails)
```

## `ShiftingRigidbody` of the boat

`ShiftingRigidbody` on the boat root:
- Boat `Rigidbody` — **non-kinematic** (dynamic, with mass computed by `BoatMass`).
- `ShiftingRigidbody` manages velocity/angularVelocity preservation during world shift (`FloatingOriginManager`): on shift → `isKinematic = true` + sleep, after → restore momentum.
- `boatProbes` (BoatProbes) — `stopProbes` during shifting; `dontUpdateVelocity = true/false`.
- Boat colliders: `CapsuleCollider` (root) + MeshCollider (disabled, duplicate on layer 12) + `BoatEmbarkCollider` (trigger, tag EmbarkCol) + `BoatEmbarkTrigger` (trigger, tag EmbarkCol) + `BoatImpactSoundsCollider` (non-trigger, OnCollisionEnter).

## Practical conclusions for modders

1. **Boat's CapsuleCollider** — main "simplified volume" on the root GO. Its surface is **above the deck**; the player ignores it (walks on `walkCollider`/layer 12), but the item twin (layer 2) collides with it. This directly explains the bug "barrel floating above the hold".
2. **`BoatEmbarkCollider.walkCollider`** — Transform of a separate deck GO (layer 12). The item twin reparents to it on `EnterBoat`.
3. **Two different EmbarkCol volumes**: one for items (`BoatEmbarkCollider`), another for the player (`BoatEmbarkTrigger`). Same tag, different GOs.
4. **On sinking** CapsuleCollider is disabled (`radius = 0` via `ToggleBoatCapsuleCol(false)`), and `BoatEmbarkCollider` flies +100Y.
5. **`HullPlayerCollider`** creates a duplicate mesh-collider on layer 12 (for walking/cleaning), disabling the original mesh — **but the root CapsuleCollider remains active and collides with items**.
6. For a mod with a solid item collider: twin-item on layer 2 collides with the boat's CapsuleCollider. Solution — either ignore the boat's layer for twin while held, or make the boat's CapsuleCollider trigger-only for item collisions, or use `Physics.IgnoreLayerCollision`.
