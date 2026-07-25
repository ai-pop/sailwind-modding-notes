# 47. Item holding mechanics: complete end-to-end flow

Complete breakdown of the pickup → hold → drop flow, participating classes, fields, phases. From decompilation of `Assembly-CSharp.dll` (Sailwind v0.38). Related to notes 16, 33, 43, 44.

## A1. Pickup/hold/drop flow (end-to-end)

### Who performs the pickup raycast

Class: **`GoPointer`** — central interaction class. Method: `DoRaycast()`.

**Phase:** `FixedUpdate()` — every fixed frame, constructs `raycastRay` and calls `DoRaycast()`.

**DoRaycast():**
```csharp
Ray val = debugEditorPointer ? raycastRay : Camera.main.ScreenPointToRay(Input.mousePosition);
LayerMask val2 = LayerMask.op_Implicit(-604165);   // item collider mask
float num = 1.8f;                                   // max ray distance
if (Physics.Raycast(val, ref hit, num, LayerMask.op_Implicit(val2)) 
    && !GameState.sleeping && !GameState.inBed && !BoatCamera.on)
```

Raycast hits **visual colliders** (`GoPointerButton`, `ShipItem`) on the layer defined by mask `-604165`. Visual colliders are **isTrigger** (set in `ShipItem.Awake`). If an `ItemSubcollider` tag is hit, the parent collider is retrieved.

**Pickup happens in `GoPointer.LateUpdate()`:**
```csharp
if (MainButtonDown() && !BoatCamera.on) {
    // ... stickyClickedButton or pointedAtButton ...
    if ((Object)(object)((Component)pointedAtButton).GetComponent<PickupableItem>() != (Object)null 
        && (!((Object)(object)((Component)pointedAtButton).GetComponent<ShipItem>() != (Object)null) 
             || !((Component)pointedAtButton).GetComponent<ShipItem>().nailed))
    {
        PickUpItem(((Component)clickedButton).GetComponent<PickupableItem>());
    }
}
```

**Important:** pickup is in `LateUpdate`, NOT `FixedUpdate`. Raycast is in `FixedUpdate`, but `PickUpItem` call is in `LateUpdate` (button press = MainButtonDown).

### The `held` field (technically `PickupableItem.held`)

Type: **`GoPointer`** (public, on `PickupableItem`, base class for `ShipItem`).

**Assignment locations:**
| Where | Code | Meaning |
|-------|------|---------|
| `GoPointer.PickUpItem()` | `heldItem.held = this;` | Set: the pointer holding the item |
| `GoPointer.DropItem()` | `heldItem.held = null;` | Clear on drop |
| `WorldItemSpawner.Update()` | `if (item.held != null)` → cooldown + `item.GetItemRigidbody().debugForceKinematic = false; item = null;` | Spawner tracks pickup, unfreezes and detaches |

**Can `held` briefly become `null` during holding?**  
No. `held` is assigned in `PickUpItem()` and only reset in `DropItem()`. No transitions between. **Exception:** a mod could call `DropItem()` via Harmony patch, but the vanilla flow keeps `held` stable from PickUp to Drop.

**However:** `held` references a `GoPointer`, which is a singleton component on the player. If `GoPointer` is destroyed (scene change, etc.) the reference would become `null`, but this doesn't happen in normal gameplay.

### Who moves the visual item each frame while held

**Class:** `GoPointer`  
**Method:** `LateUpdate()`  
**Phase:** `LateUpdate` (every frame, after Update)  
**No `[DefaultExecutionOrder]`** on `GoPointer`.

Movement depends on item type (`heldItem.big`):

**Small item (`!big`), not pointing at a button with allowPlacingItems:**
```csharp
float num2 = Mathf.InverseLerp(throwDelay, 1f, currentThrowPower);
Vector3 val2 = transform.position + transform.forward * heldItem.holdDistance + transform.up * heldItem.holdHeight;
Quaternion val3 = transform.rotation * Quaternion.Euler(heldItem.heldRotationOffset, 0f, 0f);
heldItem.transform.position = Vector3.Lerp(heldItem.transform.position, val2, timerAfterPickup);
heldItem.transform.rotation = Quaternion.Lerp(heldItem.transform.rotation, val3, timerAfterPickup);
heldItem.transform.Translate(transform.forward * (0f - num2) * 0.3f, Space.World);
```

- **Anchor:** `GoPointer` (pointer GO) position + `forward * holdDistance` + `up * holdHeight`
- `holdDistance` default = `1.15f`, `holdHeight` = `0f`
- `timerAfterPickup` ramps from 0 to 1 (`+= deltaTime * 4`), used as Lerp factor → item smoothly pulls to hand position
- **World-space `position`** (not SetParent, not local) — item is placed directly in world coordinates

**Small item, pointing at a button (allowPlacingItems):**
```csharp
heldItem.transform.position = transform.position + transform.forward * currentLookDistance + Vector3.up * heldItem.furniturePlaceHeight;
heldItem.transform.rotation = transform.rotation;
heldItem.transform.Rotate(Vector3.right, heldItem.heldRotationOffset, Space.Self);
```
- Item placed at raycast hit position (at look distance)

**Big item (`big`):**
```csharp
Quaternion rotation = transform.rotation * bigItemLocalRot;
heldItem.transform.position = transform.TransformPoint(decolLocalPos);
heldItem.transform.rotation = rotation;
heldItem.transform.Translate(transform.forward * (0f - num2) * 0.4f, Space.World);
heldItem.transform.Rotate(transform.right, heldItem.heldRotationOffset, Space.World);
// + decollision and decolLocalPos limits
```
- Position in pointer-local coordinates (`decolLocalPos`), then TransformPoint to world
- `bigItemLocalPos` and `bigItemLocalRot` are saved in `PickUpItem()` from the item's current position relative to pointer

**Critical:** the visual item is placed **in world coordinates**, without SetParent. The twin in `ItemRigidbody.FixedUpdate` when `held` **follows the visual**: `twin.position = visual.position`. This means **visual = master**, twin = follower, and visual is positioned by `GoPointer.LateUpdate`.

### Execution order of phases

```
FixedUpdate (ItemRigidbody)  → twin.position = visual.position (if held)
Update (ShipItem)            → ProcessSaveable, wallAttachment raycast
FixedUpdate (GoPointer)      → raycast
LateUpdate (GoPointer)       → moves visual, pickup/drop
LateUpdate (ItemRigidbody)   → inventory lerp, scale
```

**Attention:** `ItemRigidbody.FixedUpdate` reads `visual.position` and sets `twin.position = visual.position`. But `GoPointer.LateUpdate` hasn't executed yet in this frame! In `FixedUpdate`, visual is still at the **previous LateUpdate** position. Twin copies the "previous" position, then `GoPointer.LateUpdate` moves visual to the new position. On the next `FixedUpdate`, twin copies the new position again.

**For mods:** if a mod moves the twin via `AddForce` in `FixedUpdate`, and then `ItemRigidbody.FixedUpdate` overwrites `twin.position = visual.position` → the force is lost! This is the **known conflict** for the mod intercepting item holding.

## A2. Twin (ItemRigidbody) state while held and on release

### Is ResetPos called on pickup?

**No.** `PickUpItem()` does not call `ResetPos()`. `ResetPos()` is only called in:
- `ItemRigidbody.Start()` (after twin creation)
- `ShipItem.ResetRigidbody()` (wrapper)
- `ShipItem.SmoothlyReturnToShop()` (return to shop)

On pickup, the twin **remains at its current position**, and `ItemRigidbody.FixedUpdate` in the next frame sets `twin.position = visual.position` → twin **jumps** to visual. But `GoPointer.LateUpdate` also moves visual → by end of frame, visual is already at hand position, and on the next `FixedUpdate` twin pulls to that.

**What happens with twin colliders on pickup?**

`ItemRigidbody.FixedUpdate`: when `held` → twin colliders set to **isTrigger = true** (boxCol, meshCol, capsuleCol, subcolliders). This means the twin **doesn't push physics** while held.

`ToggleCollider(true)` is called if item is **not in inventory** (enables colliders, but disables floater — always). When `held` + not in inventory → `ToggleCollider(true)` + `isTrigger = true` on colliders.

### What happens on drop

`GoPointer.DropItem()`:
```csharp
heldItem.held = null;
heldItem = null;
```

Then (if throwing): `ThrowItemAfterDelay` — `AddForce` on twin's Rigidbody in next `WaitForFixedUpdate`.

After drop, in `ItemRigidbody.FixedUpdate`:
- `held` = null → `flag2` is not set from held → body can become dynamic
- `SetDynamicColTimer()` — **NOT called** on drop (it was called each frame during holding). `dynamicColTimer` = whatever remains from hold period
- Twin colliders: `isTrigger = false` (held = null → not trigger)
- Master: **twin → visual** (`visual.position = twin.position`)

**Transition to first frames of flight:**
1. `held = null` → FixedUpdate: `flag2` doesn't include held kinematic, but `fixedFramesSinceSpawn < 6` and `dynamicColTimer` may still be > 0 → body is still kinematic!
2. After several fixed frames kinematic is removed → body becomes dynamic, flies with throwForce
3. `ToggleCollider(true)` — colliders enabled, floater disabled

## A3. ALL paths that teleport the twin hundreds of meters while visual is alive

### EnterInventorySlot / ExitInventorySlot

```csharp
public void EnterInventorySlot(Transform slot) {
    currentInventorySlot = slot;
    item.GetComponent<Collider>().enabled = false;  // visual collider OFF
    item.OnEnterInventory();
    item.gameObject.layer = 5;                        // UI layer
    // + all children → layer 5
    inventoryEnterTimer = 0.3f;
}
```

**Where is `slot` physically?** `slot` = transform of `GPButtonInventorySlot` (inventory slot GO). It's an object **under `PlayerNeedsUI`** (inventory UI panel). The `PlayerNeedsUI` GO is in the scene, **moves with the player** (updates position to `Camera.main` in Update). But `GPButtonInventorySlot.transform` is the transform of the slot **in UI space**. It **does NOT move with FloatingOrigin shift** — it's a child of the UI panel, which is part of the Canvas/game world.

**In `ItemRigidbody.LateUpdate` (inventory):**
```csharp
Vector3 val = currentInventorySlot.TransformPoint(slotLocalPos);
item.transform.position = Vector3.Lerp(val, currentInventorySlot.TransformPoint(Vector3.zero), deltaTime * num);
twin.transform.position = item.transform.position;
```
- Item lerps to `slot.TransformPoint(slotLocalPos)` → **world position of the slot**
- Twin **copies visual**
- Scale shrinks: `PlayerNeedsUI.instance.transform.localScale * 0.2f * item.inventoryScale`

**If the slot-local position is somewhere far away (UI panel in scene under player, during FloatingOrigin shift) — the item and twin **jump to the slot**. But the slot moves with the player, so usually not hundreds of meters.**

**Critical moment:** `inventoryEnterTimer = 0.3f` — in the first 0.3s `num = 14f` (fast lerp), after that `num = 99999f` (instant snap). On exit: `ExitInventorySlot()` simply resets `currentInventorySlot = null`, visual collider enabled, layer → 2.

### EnterBox / ExitBox / GetCurrentBox

```csharp
public void EnterBox(Transform box, Vector3 localPos, Quaternion localRot) {
    currentBox = box;
    boxLocalPos = localPos;
    boxLocalRot = localRot;
}
```

**Where is `box` physically?** `box` = transform of the crate (CrateInventory) — it's the crate GO in the world. The crate is a `ShipItemCrate`, which **moves by physics** (crate's twin is a physics body).

**In `ItemRigidbody.FixedUpdate` (in box, held):**
```csharp
if (currentBox && held) {
    boxLocalPos = currentBox.InverseTransformPoint(item.transform.position);
    boxLocalRot = Quaternion.Inverse(currentBox.rotation) * twin.rotation;
}
```
- If item is held AND in a box — **boxLocalPos is updated** from current position (cell "adapts" to item movement in hand)

**In box, NOT held:**
```csharp
item.transform.position = currentBox.TransformPoint(boxLocalPos);
twin.transform.position = item.transform.position;
item.transform.rotation = currentBox.rotation * boxLocalRot;
twin.transform.rotation = item.transform.rotation;
```
- Item **snaps to position in crate** (in crate's world coordinates)
- If crate is on a boat (onBoat) — crate position = boat. Crate is child of boat
- Twin **copies visual** → jumps to crate

**Can `box` be hundreds of meters away?** Yes, if the crate is on a boat, and the boat shifted during a FloatingOrigin shift, and the item hasn't updated yet. But in normal flow this doesn't happen — `FixedUpdate` synchronizes every frame.

### EnterBoat / ExitBoat

```csharp
public void EnterBoat() {
    twin.parent = item.currentWalkCol;               // REPARENT twin to boat walkCol!
    MoveRigidbodyToWalkCol();
    item.currentActualBoat.parent.GetComponent<BoatMass>().AddItem(this);
    onBoat = true;
}
```

**`currentWalkCol`** — Transform of the boat's walk collider (BoatEmbarkCollider.walkCollider). It's a real object **under the boat** in the scene.

**`MoveRigidbodyToWalkCol`:**
```csharp
Vector3 val = item.currentActualBoat.InverseTransformPoint(item.transform.position);
Quaternion val2 = Quaternion.Inverse(item.currentActualBoat.rotation) * twin.rotation;
twin.position = item.currentWalkCol.TransformPoint(val);
twin.rotation = item.currentWalkCol.rotation * val2;
```
- Takes item position **in boat local coordinates** (`currentActualBoat`)
- Converts to world via **walkCol** (`currentWalkCol.TransformPoint`)
- **If `currentActualBoat` and `currentWalkCol` are different GOs** (different boat transforms) — position can be distorted!

**`ExitBoat`:**
```csharp
twin.parent = world;  // FloatingOriginManager.instance.transform
twin.position = item.transform.position;
twin.rotation = item.transform.rotation;
onBoat = false;
```
- Twin **jumps to visual position**, reparents to world

**`MoveItemToWalkColRigidbody`** (for NOT-held item on boat):
```csharp
Vector3 val = item.currentWalkCol.InverseTransformPoint(twin.position);
Quaternion val2 = Quaternion.Inverse(item.currentWalkCol.rotation) * twin.rotation;
item.transform.position = item.currentActualBoat.TransformPoint(val);
item.transform.rotation = item.currentActualBoat.rotation * val2;
```
- **twin → local → walkCol → world via actualBoat → visual**
- Visual is set from twin position (master = twin)

**Can walkCol point to a boat hundreds of meters away?** walkCol is a child Transform of the player's current boat. The boat moves with FloatingOrigin (under `FloatingOriginManager.instance.transform`). If the walkCol reference becomes stale — no, it's a live Transform in the scene.

### Purchase flow in shop

**Classes:** `ShopItemSpawner` (shop spawner), `Shopkeeper` (vendor), `BuyItemUI` (purchase UI).

**`ShopItemSpawner.SpawnItem()`:**
```csharp
item = Object.Instantiate<GameObject>(itemPrefab, transform.position, transform.rotation).GetComponent<ShipItem>();
item.transform.parent = transform;  // child of spawner
```

- Creates a **NEW ShipItem** (Instantiate) at spawner position
- Item is **not sold** (shop item) → `!sold` → kinematic in `ItemRigidbody.FixedUpdate`
- Twin is created in `LoadAfterDelay()` coroutine — **at visual position** (shop spawner)

**Purchase (`Shopkeeper.SellItem` → `ShipItem.Sell()`):**
```csharp
public void Sell() {
    sold = true;
    transform.parent = FloatingOriginManager.instance.transform;
    UpdateLookText();
    saveable.RegisterToSave();
    if (pointedAtBy != null) {
        pointedAtBy.PickUpItem(this);  // pickup immediately after purchase!
    }
    OnBuy();
}
```

- `sold = true` → item stops being kinematic from `!sold`
- Reparents to world (`FloatingOriginManager.instance.transform`)
- **Picked up immediately** (`PickUpItem`) — goes into hand
- Twin: at `Sell()` moment, `held` is set → in next `FixedUpdate` twin.position = visual.position (visual already at pointer hand)

**Important:** NO new Instantiate on purchase! Item was created by `ShopItemSpawner`, and `Sell()` simply sets `sold = true` + picks up. Twin **is not recreated**.

### Item streaming/despawn (by distance)

In `ItemRigidbody.FixedUpdate`:
```csharp
if (distanceCheckTimer <= 0f) {
    if (Vector3.Distance(Camera.main.transform.position, item.transform.position) > 600f) {
        outOfRange = true;
    } else {
        outOfRange = false;
    }
    distanceCheckTimer = Random.Range(5f, 8f);
}
```

- Check every **5–8 seconds**, not every frame
- `outOfRange = true` → `flag2 = true` (kinematic) + `framesUntilDestroy` for sold items not on boat
- **Destruction:** sold item off boat + out of range → `framesUntilDestroy` > 10 → `item.DestroyItem()` (GameObject.Destroy)
- **NOT sold items** (shop items) → `framesUntilDestroy = 0` (not destroyed)

**Twin on outOfRange:**
- `FixedUpdate` does early exit (`return` after flag check) → twin **is not managed**, position frozen
- On return to range (`outOfRange = false`) → `fixedFramesSinceSpawn` is not reset, but kinematic flag drops when sold + free

**There is no separate streaming/despawn manager for items.** Everything is inside `ItemRigidbody.FixedUpdate`.

### WorldItemSpawner.FreezeItem

```csharp
private IEnumerator FreezeItem() {
    yield return new WaitForEndOfFrame();
    item.GetItemRigidbody().debugForceKinematic = true;
}
```

- Called after `SpawnItem()` — via 1 WaitForEndOfFrame
- **Freezes twin** (`debugForceKinematic = true`)
- Unfreeze: `WorldItemSpawner.Update()` — when item is picked up (`held != null`) → `debugForceKinematic = false`

## A4. Floating Origin

### Exact mechanism of FloatingOriginManager

**Shift:** if `shifterObject` position exceeds `shiftDistance` → shift all children of `FloatingOriginManager.instance.transform` by `shiftDistance * (x,z)`.

**`NewShift(shiftVector)` coroutine:**
```csharp
MoveOriginOcean(-shiftVector);  // shift Crest/OceanRenderer origin
foreach (Transform item in transform) {
    item.Translate(shiftVector, Space.World);  // shift ALL children
}
// + PrepareForShifting/RestoreMomentum for ShiftingRigidbody (boats)
outCurrentOffset += new Vector3(shiftVector.x, 0, shiftVector.z);
```

- `item.Translate(shiftVector, Space.World)` — **moves each child** of `FloatingOriginManager.instance.transform`
- Item twin is a child of `FloatingOriginManager.instance.transform` (parent = world) → **moves with shift**
- **Item visual does NOT always move** (parent = currentActualBoat or world, but not necessarily a direct child of FloatingOriginManager). On boat: visual is child of boat. In world: visual.parent = world = FloatingOriginManager.transform → also moves.

**Can twin miss a shift or shift twice?**

- twin.parent = `FloatingOriginManager.instance.transform` (always, when item is not on boat)
- `NewShift` does `foreach (Transform item in transform) { item.Translate(shiftVector) }` — this **moves each direct child**. Children of twin (colliders, floater) are on the twin GameObject itself and move as part of it.
- `ShiftingRigidbody` (boat) — `PrepareForShifting` + `RestoreMomentum` — handled separately. Twin of item on boat: parent = `currentWalkCol` (child of boat), shifts **through the boat**.
- **Double shift:** if twin.parent = world (FloatingOriginManager) and twin **is also** registered in `ShiftingRigidbody` → could shift twice. But item twin does NOT have `ShiftingRigidbody` — only boats do. → **No double shift for item twin.**

**`ShiftingPosToRealPos(pos)` = `pos - outCurrentOffset`** — from scene → real.  
**`RealPosToShiftingPos(pos)` = `pos + outCurrentOffset`** — from real → scene.  
Shift happens **simultaneously** for all children: ocean + world + boats (via ShiftingRigidbody).

## A5. Complete body of ItemRigidbody.FixedUpdate (verbatim)

```csharp
private void FixedUpdate() {
    if (Debugger.instance.disableItemRigidbodyUpdate) { enabled = false; }
    
    fixedFramesSinceSpawn += 1f;
    bool flag = false;                    // "alive" flag
    
    // --- Distance / culling ---
    if (distanceCheckTimer <= 0f) {
        outOfRange = Vector3.Distance(Camera.main.position, item.position) > 600f;
        distanceCheckTimer = Random.Range(5f, 8f);
    } else { distanceCheckTimer -= deltaTime; }
    
    if (outOfRange && !GameState.recovering) {
        if (item.sold && item.currentWalkCol == null) {
            framesUntilDestroy++;
            if (framesUntilDestroy > 10 && item.gameObject.layer != 26) {
                item.DestroyItem(); return;
            }
        } else { framesUntilDestroy = 0; }
    } else { flag = true; }    // in range → "alive"
    
    if (inStove) { flag = false; }
    
    // --- Disable col ---
    if (disableCol) { ToggleCollider(false); }
    if (!flag) { return; }     // EARLY EXIT (outOfRange + inStove)
    
    // --- Colliders ---
    if (currentInventorySlot) { ToggleCollider(false); }
    else { ToggleCollider(true); }   // ← floater.enabled = false ALWAYS
    
    // --- Positioning ---
    if (!currentInventorySlot) {
        if (item.currentWalkCol != null && onBoat) {
            if (held) { MoveRigidbodyToWalkCol(); }
            else { MoveItemToWalkColRigidbody(); }
        } else if (held || !sold) {
            // VISUAL → TWIN (master = visual)
            twin.position = visual.position;
            twin.rotation = visual.rotation;
        } else {
            // TWIN → VISUAL (master = twin, sold + free)
            visual.position = twin.position;
            visual.rotation = twin.rotation;
        }
    }
    
    // --- Box ---
    if (currentBox) {
        if (held) {
            boxLocalPos = currentBox.InverseTransformPoint(visual.position);
            boxLocalRot = Quaternion.Inverse(currentBox.rotation) * twin.rotation;
        } else {
            visual.position = currentBox.TransformPoint(boxLocalPos);
            twin.position = visual.position;
            visual.rotation = currentBox.rotation * boxLocalRot;
            twin.rotation = visual.rotation;
        }
    }
    
    // --- dynamicColTimer ---
    if (held) { SetDynamicColTimer(); }  // reset to 6
    if (!held && dynamicColTimer > 0f && !isKinematic) {
        rigidbody.collisionDetectionMode = Continuous;
        dynamicColTimer -= deltaTime;
    }
    if (dynamicColTimer <= 0f && meshCol == null) {
        rigidbody.collisionDetectionMode = ContinuousDynamic;
    }
    
    // --- Kinematic flag ---
    bool flag2 = (held != null);          // held → kinematic
    
    // isTrigger on colliders = held
    boxCol.isTrigger = held;
    meshCol.isTrigger = held;
    capsuleCol.isTrigger = held;
    SetSubcollidersTriggers(held);
    
    // meshCol sleeping → kinematic
    if (meshCol && rigidbody.IsSleeping() && dynamicColTimer <= 0f) { flag2 = true; }
    
    // GameState gates → kinematic
    if (!playing || recovering || sleeping || inBed || currentShipyard) { flag2 = true; }
    
    // box/inventory → kinematic
    if (currentBox || currentInventorySlot) { flag2 = true; }
    
    // attached → kinematic
    if (attached) { flag2 = true; }
    
    // !sold → kinematic
    if (!item.sold) { flag2 = true; }
    
    // nailed → kinematic
    if (item.nailed) { flag2 = true; }
    
    // first 6 frames → kinematic
    if (fixedFramesSinceSpawn < 6f) { flag2 = true; }
    
    // outOfRange → kinematic
    if (outOfRange) { flag2 = true; }
    
    // debugger → kinematic
    if (Debugger.kinematicItemsTimer > 0f) { flag2 = true; }
    if (Debugger.debugForceKinematicBoat) { flag2 = true; }
    if (debugForceKinematic) { flag2 = true; }
    
    // apply
    if (isKinematic != flag2) { rigidbody.isKinematic = flag2; }
    
    if (Debugger.instance.disableItemRigidbodyCols) { ToggleCollider(false); }
}
```

**Complete body of `ItemRigidbody.LateUpdate`:**
```csharp
void LateUpdate() {
    if (currentInventorySlot) {
        float num = 99999f;
        if (inventoryEnterTimer > 0f) { num = 14f; inventoryEnterTimer -= deltaTime; }
        Vector3 val = currentInventorySlot.TransformPoint(slotLocalPos);
        item.position = Vector3.Lerp(val, currentInventorySlot.TransformPoint(Vector3.zero), deltaTime * num);
        twin.position = item.position;
        slotLocalPos = currentInventorySlot.InverseTransformPoint(item.position);
        item.rotation = Quaternion.Lerp(item.rotation, currentInventorySlot.rotation * Euler(invRotX, invRot, 0), deltaTime * num);
        twin.rotation = item.rotation;
        twin.localScale = Vector3.Lerp(twin.localScale, PlayerNeedsUI.instance.transform.localScale * 0.2f * item.inventoryScale, deltaTime * num);
        item.localScale = twin.localScale;
    } else if (!inStove) {
        twin.localScale = Vector3.one;
        item.localScale = Vector3.one;
    }
}
```

## Sequence: pickup → hold → drop (all frames)

```
[FixedUpdate N]   GoPointer: raycast, stores pointedAtButton
[LateUpdate N]    GoPointer: MainButtonDown → PickUpItem(item)
                    item.held = this (GoPointer)
                    item.OnPickup() → wallAttachment: attached=false, exitInventorySlot
[FixedUpdate N+1] ItemRigidbody: held=true → twin.position = visual.position (copies OLD position!)
                   flag2=true → isKinematic=true → twin kinematic
                   ToggleCollider(true) → floater OFF
                   SetDynamicColTimer() → dynamicColTimer=6
[LateUpdate N+1]  GoPointer: moves visual to hand position (Lerp(timerAfterPickup))
                   visual → new hand position
[FixedUpdate N+2] twin.position = visual.position → twin at hand position
...
[LateUpdate]      GoPointer: MainButtonUp + throw → DropItem()
                    heldItem.held = null; heldItem = null;
                    item.OnDrop() → sold? visual → free; wallAttachment? → twin.position = attachPos, attached=true
[FixedUpdate]     ItemRigidbody: held=null → flag2 removes held-kinematic
                   if sold && free → flag2=false → dynamic
                   twin → visual: visual.position = twin.position (twin = master)
                   ThrowItemAfterDelay: AddForce on twin
```

## Conclusion for modder (twin ~462m bug)

**Key observation:** twin while held **always** is set to visual.position in `FixedUpdate`. If a mod sets twin.position via `AddForce` → in `FixedUpdate` twin.position **is overwritten** = visual.position (held = master = visual). Force is lost.

**Twin 462m bug with alive visual:**  
Most likely cause is `EnterBoat()`. When an item is picked up on a boat, `MoveRigidbodyToWalkCol()` uses `currentActualBoat.InverseTransformPoint` and `currentWalkCol.TransformPoint`. If `currentWalkCol` and `currentActualBoat` are different GOs (walkCol = child Transform, actualBoat = boat root), and **the boat shifted during FloatingOrigin shift** between frames — the coordinate transformation can yield a position **hundreds of meters from actual**.

Also: `EnterInventorySlot` — if `currentInventorySlot.TransformPoint(slotLocalPos)` yields the UI panel position (PlayerNeedsUI), which during shift could be **at the old position** → twin jumps to UI.

**To find the bug:** check whether `EnterBoat` / `EnterInventorySlot` / `EnterBox` is called during a shift moment, or whether twin is reparented during shift at an inappropriate moment (twin.parent = currentWalkCol on boat, but shift only moves children of FloatingOriginManager.transform, while twin on boat moves through the boat/ShiftingRigidbody).
