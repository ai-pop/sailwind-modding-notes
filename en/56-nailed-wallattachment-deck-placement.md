# 56. Item sticking to the boat: nailed, wallAttachment, deck placement

Breakdown of mechanisms that "attach" an item to the boat — `nailed`, `wallAttachment`, `HangableItem` — and explanation of why an item may lie "on the surface above the hold" — answer to request E6. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 51 (colliders), 53 (Enter/ExitBoat), 16 (ShipItem/twin).

## `ShipItem.nailed` — "nailed" item

### Field

| Field | Type | Content |
|------|-----|------------|
| `nailed` | `bool` | On `ShipItem` — item is "nailed", treated as kinematic (doesn't move). Set via `ShipItemHammer`. Saved in `SavePrefabData.isNailed`. |

### Setting: `ShipItemHammer.NailItem(ShipItem item)` — verbatim

```csharp
private void NailItem(ShipItem item)
{
    if (item.big && !item.wallAttachment && !RaycastHitsFloor(item))
    {
        NotificationUi.instance.ShowNotification("Item must be on the floor.");
        return;
    }
    if (item.GetItemRigidbody().GetBody().velocity != Vector3.zero)
    {
        NotificationUi.instance.ShowNotification("Item is moving.");
        return;
    }
    item.nailed = true;
    UISoundPlayer.instance.PlayUISound(UISounds.winchUnclick, 1f, 0.6f);
    Debug.Log("Nailed " + item.name);
}
```

**Conditions:** (1) item `sold` (purchased); (2) if `big && !wallAttachment` — must be on the floor (raycast down 4 m, layer 8 = Player/HullPlayerCollider); (3) not moving (`velocity == Vector3.zero`).

**Removal:** `ShipItemHammer.OnAltActivate()` → if item already `nailed` → `item.nailed = false` + sound.

### Who can be nailed: `ShipItemHammer.CanNail(ShipItem item)` — verbatim

```csharp
public static bool CanNail(ShipItem item)
{
    if (!item.sold) return false;
    if (item.big || ((object)item).GetType() == typeof(ShipItemHangable) || item.wallAttachment)
    {
        return true;
    }
    return false;
}
```

**Allowed:** `big` items, `ShipItemHangable` (lamps), `wallAttachment` items (wall-mountable). Small regular items — **no**.

### `nailed` effect on ItemRigidbody — verbatim (FixedUpdate)

```csharp
// ItemRigidbody.FixedUpdate — key lines
if (item.nailed)
{
    flag2 = true;  // flag2 → rigidbody.isKinematic = true
}
```

**`nailed = true` → `rigidbody.isKinematic = true` → twin stops moving via physics.** Twin is "frozen" in its current position. Twin colliders when nailed don't become trigger — they remain **non-trigger**, but kinematic Rigidbody **doesn't generate OnCollisionEnter** (for dynamic bodies, collision with kinematic doesn't fire, unless the kinematic moves).

> **Important for the mod:** if mod makes twin-collider non-trigger and item `nailed`, twin = kinematic + non-trigger. Kinematic non-trigger collider **collides** with dynamic Rigidbodies (pushes them out), but **doesn't receive** OnCollisionEnter. This means: nailed item **doesn't cause boat damage** via BoatImpactSoundsCollider (CollisionEnter doesn't fire), but **can push out** dynamic objects (other items, twin during mod hold).

### `nailed` in GoPointer/LookUI — interaction restrictions

- `GoPointer.DoRaycast`: if item `nailed` → **not picked up** (doesn't enter PickupableItem branch) — caveat: `ShipItemCrate`, `ShipItemBottle`, `ShipItemBed` even when nailed can be "looked at" (Look), but not picked up.
- `GoPointer.MainButtonDown`: `!item.nailed` — condition for Pickup.
- `LookUI`: nailed items → **lookText = "Nailed"** (nailed status info), don't show price/mission.
- `CargoStorageUI`: nailed big-items **cannot be loaded** into cargo (`!component2.nailed`).

## `ShipItem.wallAttachment` — wall-mountable attachment

### Field and state

| Field | Type | Content |
|------|-----|------------|
| `wallAttachment` | `bool` | On `ShipItem` — item can attach to wall. On load (`LoadAfterDelay`): if `wallAttachment` → `itemRigidbodyC.attached = true` (twin = kinematic, "stuck"). |
| `attachPos` | `Vector3` (private) | Wall attachment point — `RaycastHit.point` from raycast forward out of twin. |
| `attachRot` | `Quaternion` (private) | Attachment rotation — `LookRotation(-hit.normal, ...)`. |
| `inRangeOfWall` | `bool` (private) | True if raycast found wall within 0.1–1.3 m range. |

### Wall detection — `ShipItem.Update()` — verbatim

```csharp
// ShipItem.Update() — wallAttachment branch (verbatim)
if (Object.op_Implicit((Object)(object)held) && wallAttachment)
{
    Vector3 forward = itemRigidbody.forward;
    RaycastHit val = default(RaycastHit);
    if (Physics.Raycast(new Ray(itemRigidbody.position, forward), ref val, 1.3f))
    {
        if (((RaycastHit)(ref val)).distance < 0.1f)
        {
            inRangeOfWall = false;
        }
        else
        {
            attachPos = ((RaycastHit)(ref val)).point;
            attachRot = Quaternion.LookRotation(-((RaycastHit)(ref val)).normal, Vector3.up);
            if (((RaycastHit)(ref val)).normal.y > 0.8f)
            {
                attachRot = Quaternion.LookRotation(-((RaycastHit)(ref val)).normal, ((Component)this).transform.up);
            }
            inRangeOfWall = true;
            SetUpTargeter(attachPos, attachRot, ((Component)this).GetComponent<MeshFilter>().sharedMesh);
        }
    }
    else
    {
        inRangeOfWall = false;
    }
    if (!inRangeOfWall) { }
}
else
{
    inRangeOfWall = false;
}
```

**Mechanics:** while holding item (`held != null`) and `wallAttachment` — every `Update()` raycast from twin position (`itemRigidbody.position`) along `itemRigidbody.forward` for 1.3 m. If wall found (distance 0.1–1.3 m):

- `attachPos = hit.point` — contact point on wall.
- `attachRot`: if `normal.y > 0.8` (horizontal surface / ceiling) → `LookRotation(-normal, item.transform.up)` (item "stands" on normal). otherwise → `LookRotation(-normal, Vector3.up)` (item "stuck" to vertical wall, facing the wall).
- `inRangeOfWall = true` → `Targeter` is shown (wireframe projection of item on wall).

**Critical:** raycast originates from **twin position** (`itemRigidbody.position`), not from visual. In vanilla twin position = visual position (ItemRigidbody.LateUpdate synchronizes). In mod — twin position may differ (physics pose) → wall raycast originates from **physical twin position**, which may be **inside boat CapsuleCollider** → raycast doesn't see wall (hits CapsuleCollider) → `inRangeOfWall = false` → item **won't stick** to wall on drop.

### `SetUpTargeter` — visual projection of item on wall

```csharp
private void SetUpTargeter(Vector3 attachPos, Quaternion attachRot, Mesh mesh)
{
    Vector3 val = itemRigidbody.InverseTransformPoint(attachPos);
    Vector3 pos = ((Component)this).transform.TransformPoint(val);
    Quaternion val2 = Quaternion.Inverse(itemRigidbody.rotation) * attachRot;
    Quaternion rot = ((Component)this).transform.rotation * val2;
    held.GetTargeter().DisplayTargeter(pos, rot, mesh);
}
```

Displays wireframe mesh of the item at the assumed position/rotation on the wall — "targeter" (hint to the player where the item will stick).

### Drop with wallAttachment — `ShipItem.OnDrop()` — verbatim

```csharp
public override void OnDrop()
{
    if (!sold)
    {
        ReturnToShopPos();
    }
    else if (wallAttachment && inRangeOfWall && !forceDisableRedOutline)
    {
        ((Component)itemRigidbody).transform.position = attachPos;
        ((Component)itemRigidbody).transform.rotation = attachRot;
        ((Component)itemRigidbody).GetComponent<ItemRigidbody>().attached = true;
    }
}
```

**On wallAttachment item drop:** if `sold && inRangeOfWall && !forceDisableRedOutline` → twin (ItemRigidbody GO) snaps to `attachPos/attachRot` + `attached = true`.

**`attached = true`** → `ItemRigidbody.FixedUpdate` sets `flag2 = true` → `rigidbody.isKinematic = true` → twin **frozen** in wall position/rotation. Twin doesn't move via physics.

**Important:** on drop, **twin position** is set to `attachPos`, but **visual position** — doesn't change (GoPointer.DropItem sets visual.layer=0, held=null, but doesn't move visual). Visual↔twin synchronization: `ItemRigidbody.LateUpdate` — when `attached=true`, twin is kinematic → twin position is static, visual **doesn't synchronize** with twin (rather, when attached, twin position is "primary").

> **Critical for the mod:** on wallAttachment drop, twin snaps to wall → twin position coincides with wall surface (not boat CapsuleCollider). Visual then synchronizes with twin via LateUpdate. But **if twin is already inside boat CapsuleCollider** (mod with solid collider during hold), raycast from twin may not find wall (raycast inside CapsuleCollider → ray doesn't exit), and `inRangeOfWall = false` → item **won't stick**, just drops as regular → twin free-falls → collisions with CapsuleCollider → all bug scenarios from notes 51–55.

### Pickup with wallAttachment — `ShipItem.OnPickup()` — verbatim

```csharp
public override void OnPickup()
{
    if (wallAttachment)
    {
        itemRigidbodyC.attached = false;
    }
    // ... inventory slot withdrawal ...
    overrideEnableOutline = false;
}
```

**On pickup:** `attached = false` → twin stops being kinematic → can move via physics. In mod: twin can again collide with boat CapsuleCollider.

### Initial wallAttachment state on load

```csharp
// ShipItem.LoadAfterDelay()
if (wallAttachment)
{
    GetItemRigidbody().attached = true;
}
```

**On load from save:** wallAttachment items start `attached = true` → kinematic → "stuck" at saved position (don't move during load). If save is correct, twin position = wall.

## `ItemRigidbody.attached` — kinematic lock for twin

### Field

| Field | Type | Content |
|------|-----|------------|
| `attached` | `bool` | On `ItemRigidbody` — twin is "stuck" (wallAttachment/HangableItem). When `true` → `rigidbody.isKinematic = true` (twin frozen). |

### Effect in FixedUpdate — verbatim

```csharp
if (attached)
{
    flag2 = true;   // → isKinematic = true
}
```

### Related `ItemRigidbody` fields for "sticking"

| Field | Type | Content |
|------|-----|------------|
| `attached` | `bool` | Kinematic lock (wallAttachment/HangableItem) |
| `disableCol` | `bool` | Disable twin colliders (HangableItem: on hook → disableCol=true) |
| `inStove` | `bool` | Item "in stove/hook" (HangableItem: inStove=true when on hook) |

`disableCol = true` → `ToggleCollider(state: false)` — **twin colliders completely disabled** (enabled = false). This is for HangableItem: lamps on hooks have no physical twin colliders.

`inStove = true` → affects buoyancy/visibility in LateUpdate (twin position = hook position, not in water).

### `attached` blocks ExitBoat

```csharp
// ShipItem.ExtraFixedUpdate
else if (!currentlyStayedEmbarkCol && currentActualBoat && frameCounter > num
         && !GameState.sleeping && !disallowDisembarking && !itemRigidbodyC.attached)
{
    ExitBoat();
}
```

**`attached = true` → ExitBoat NOT called** — item is "stuck" to boat (wall/Hook) → can't accidentally exit EmbarkCol.

## `HangableItem` — hanging on a hook (lamps)

### Component and fields

`HangableItem : MonoBehaviour` — `[RequireComponent(typeof(ShipItem))]`, separate component for hook-hangable items (lamps, lanterns).

| Field | Type | Content |
|------|-----|------------|
| `shipItem` | `ShipItem` | private reference to ShipItem |
| `itemRigidbody` | `Rigidbody` | twin Rigidbody |
| `itemRigidbodyC` | `ItemRigidbody` | twin ItemRigidbody component |
| `currentHook` | `Collider` | current hook (ShipItemLampHook) |
| `disallowHangingOnTrigger` | `bool` | block automatic hanging (on inventory exit) |
| `lockX` | `bool` [SerializeField] | lock X rotation when hanging |
| `lockZ` | `bool` [SerializeField] | lock Z rotation when hanging |
| `rotX` | `float` [SerializeField] | fixed X angle when hanging |
| `rotZ` | `float` [SerializeField] | fixed Z angle when hanging |
| `framesAfterAwake` | `float` | frame counter — OnTriggerEnter only after ≥3 frames |

### `OnTriggerEnter(Collider other)` — verbatim

```csharp
public void OnTriggerEnter(Collider other)
{
    if (!(framesAfterAwake >= 3f) && ((Component)this).GetComponent<SaveablePrefab>().currentCrateId <= 0
        && !Object.op_Implicit((Object)(object)shipItem.held) 
        && shipItem.GetCurrentInventorySlot() == -1 
        && !disallowHangingOnTrigger 
        && ((Component)other).CompareTag("Hook"))
    {
        Debug.Log(shipItem.name + ".HangableItem: Connecting joint from trigger enter.");
        ConnectJoint(other);
    }
}
```

**Conditions:** ≥3 frames after Awake, not in crate, not held, not in inventory, not disallowHanging, other.CompareTag("Hook"). → Auto-hang on trigger enter.

### `ConnectJoint(Collider hook)` — verbatim

```csharp
public void ConnectJoint(Collider hook)
{
    currentHook = hook;
    ConfigurableJoint val = ((Component)currentHook).GetComponent<ShipItemLampHook>().CreateJoint();
    if (Object.op_Implicit((Object)(object)val))
    {
        if ((Object)(object)itemRigidbody == (Object)null)
        {
            itemRigidbody = ((Component)shipItem.GetItemRigidbody()).GetComponent<Rigidbody>();
            itemRigidbodyC = shipItem.GetItemRigidbody();
        }
        ((Joint)val).connectedBody = itemRigidbody;
        itemRigidbody.ResetCenterOfMass();
        itemRigidbodyC.attached = true;
        itemRigidbodyC.disableCol = true;
        itemRigidbodyC.inStove = true;
        itemRigidbodyC.ForceRigidbodyToWalkCol();
        shipItem.ToggleDisallowDisembarking(newState: true);
        Debug.Log("Connected joint.");
    }
}
```

**ConnectJoint consequences:**
- `ConfigurableJoint` on twin GO (created by `ShipItemLampHook.CreateJoint` → copy of joint from hook) → `connectedBody = itemRigidbody` → twin physically connected to hook via joint.
- `attached = true` → twin kinematic (frozen).
- `disableCol = true` → twin colliders disabled (don't participate in physics).
- `inStove = true` → twin not in water, visible = hook position.
- `ForceRigidbodyToWalkCol()` → twin snap to boat walkCol (if on boat).
- `ToggleDisallowDisembarking(true)` → item **cannot** ExitBoat.

> **HangableItem on hook:** twin = kinematic + colliders disabled + ConfigurableJoint → **doesn't physically participate** in collisions. Twin is "frozen" and "stuck" via joint to hook. Visual position synchronizes with hook in LateUpdate.

### `DisconnectJoint()` — verbatim

```csharp
public void DisconnectJoint()
{
    if (!((Object)(object)currentHook == (Object)null))
    {
        ((Component)currentHook).GetComponent<ShipItemLampHook>().RemoveJoint();
        currentHook = null;
        itemRigidbodyC.attached = false;
        itemRigidbodyC.disableCol = false;
        itemRigidbodyC.inStove = false;
        shipItem.ToggleDisallowDisembarking(newState: true);
        Debug.Log("Disconnected joint.");
    }
}
```

**DisconnectJoint consequences:**
- `RemoveJoint()` → Destroy ConfigurableJoint on twin GO.
- `attached = false` → twin stops being kinematic → can move.
- `disableCol = false` → twin colliders re-enabled → physically active.
- `inStove = false` → twin buoyancy/visibility normal.
- `ToggleDisallowDisembarking(true)` → **odd:** on disconnect disembarking is also blocked → item remains "on boat" until next event.

**DisconnectJoint is called from:**
- `ShipItem.ExitBoat()` → if `held == null && HangableItem != null` → DisconnectJoint.
- `ShipItemLampHook.OnPickup()` → disconnect on hook pickup.
- `ShipItemLampHook.OnEnterInventory()` → disconnect when hook enters inventory.

### `LateUpdate` — visual position = hook + offset

```csharp
private void LateUpdate()
{
    if (Object.op_Implicit((Object)(object)currentHook))
    {
        ((Component)shipItem).transform.position = ((Component)currentHook).transform.position 
            + ((Component)currentHook).transform.forward * -0.128f;
        Vector3 eulerAngles = ((Component)this).transform.eulerAngles;
        if (lockX) { eulerAngles.x = rotX; }
        if (lockZ) { eulerAngles.z = rotZ; }
        ((Component)this).transform.eulerAngles = eulerAngles;
    }
}
```

**Visual position = hook position - 0.128 m along hook.forward + fixed X/Z rotation.** This gives lamps a "hanging down" appearance from the hook.

## `ShipItemLampHook` — lamp hook

`ShipItemLampHook : ShipItem` — hook-item on the boat (wall hook for lamp).

### `CreateJoint()` / `RemoveJoint()` — verbatim

```csharp
public ConfigurableJoint CreateJoint()
{
    if (occupied) { return null; }
    ConfigurableJoint component = ((Component)this).GetComponent<ConfigurableJoint>();
    ConfigurableJoint result = CopyJoint(component, ((Component)itemRigidbody).gameObject);
    occupied = true;
    return result;
}

public void RemoveJoint()
{
    Object.Destroy((Object)(object)((Component)itemRigidbody).gameObject.GetComponent<ConfigurableJoint>());
    occupied = false;
}
```

**`CopyJoint(ConfigurableJoint original, GameObject targetObject):`** — copies all motion/limit settings from original ConfigurableJoint to new joint on twin GO. connectedBody is set in `ConnectJoint`.

**`occupied` = bool:** hook is occupied by a lamp. When occupied → `OnItemClick` rejects (return false). When occupied → pickup/inventory → disconnect lamp first.

## Deck placement of regular item (no wallAttachment/nailed)

### What happens when dropping a regular item on the boat

1. `GoPointer.LateUpdate` → `MainButtonUp` → if `collisions <= 0 || allowObstructedDropping || throwButtonUp` → `heldItem.OnDrop()` + `GoPointer.DropItem()`.
2. `ShipItem.OnDrop()` — **does nothing** for regular item (not wallAttachment): `if (!sold) → ReturnToShopPos()` or **empty** (no wallAttachment branch).
3. `GoPointer.DropItem()` → `heldItem.held = null; heldItem.gameObject.layer = 0` → item becomes "free".
4. Twin (ItemRigidbody) — **free** (not kinematic, not attached, not nailed) → **physics takes control of twin**.
5. Twin falls → `OnTriggerEnter(EmbarkCol)` → `ShipItem.ExtraFixedUpdate` → `frameCounter > 1` → `EnterBoat()` → twin reparents to `walkCollider` → twin position snaps to walkCol.
6. Twin on walkCol (layer 12) → **walkCol — deck surface (layer 12, MeshCollider)**. Twin physically "stands" on deck (walkCol mesh = hold/deck shape).

**In vanilla:** twin = trigger colliders → twin **doesn't collide** with boat CapsuleCollider → twin falls, EnterBoat, reparent to walkCol → twin trigger-collider "transparent" to physics → visual/twin on deck.

**In mod (non-trigger twin):** twin collides with boat CapsuleCollider → twin "lies" on CapsuleCollider surface (above deck) → twin still inside EmbarkCol trigger → EnterBoat → twin reparent to walkCol → twin snap to walkCol position → **but twin physically inside CapsuleCollider** → OnCollisionEnter → BoatImpactSoundsCollider → BoatDamage.Impact (twin Untagged) → boat damage.

> **Why item "on the surface of volume above hold":** vanilla twin (isTrigger=true) = "passes through CapsuleCollider" and sits on walkCol (deck, layer 12). Mod twin (non-trigger) = **cannot pass through CapsuleCollider** → twin "stands" on CapsuleCollider surface → this is **above the deck** (CapsuleCollider surface = top of capsule enclosing hull).

## Sequence diagram: item drop on boat (vanilla vs mod)

```
=== VANILLA (twin isTrigger=true) ===
Player.DropItem()
  → heldItem.held = null
  → twin: isTrigger=true, Rigidbody dynamic
  → twin falls through CapsuleCollider (isTrigger → no physical collision)
  → twin OnTriggerEnter(EmbarkCol) → ShipItem.EnterBoat()
  → twin reparent to walkCollider (layer 12)
  → twin snap to walkCol position = deck
  → twin stands on walkCol mesh (layer 12) ← actual deck
  → twin isTrigger → no further collision issues

=== MOD (twin non-trigger) ===
Player.DropItem()
  → heldItem.held = null
  → twin: isTrigger=false, Rigidbody dynamic
  → twin falls → HITS CapsuleCollider surface (above deck)
  → twin "stands in the air" above deck
  → twin OnTriggerEnter(EmbarkCol) → ShipItem.EnterBoat()
  → twin reparent to walkCollider → snap to walkCol position
  → BUT: twin non-trigger → OnCollisionEnter with BoatImpactSoundsCollider
  → BoatImpactSounds.Impact → BoatDamage.Impact (twin Untagged → not filtered!)
  → hullDamage += 0.15 every 1 s on contact
  → MOD SPRING can pull twin inside → physics push-out → OnCollisionEnter every step
  → Enter/ExitBoat flap (if twin exits EmbarkCol trigger)
```

## Practical conclusions for modders

1. **`nailed = true` → twin kinematic** → twin doesn't move, doesn't receive OnCollisionEnter → **doesn't cause boat damage**, doesn't flap. But **can push out dynamic bodies** (kinematic non-trigger vs dynamic).
2. **`wallAttachment`** — wall/floor sticking mechanism via raycast + snap + `attached=true`. In vanilla works correctly (twin isTrigger → raycast sees wall). In mod: if twin inside CapsuleCollider → raycast doesn't see wall → wallAttachment **fails** on drop → item drops as regular → bug scenarios 51–55.
3. **`HangableItem`** — lamps on hooks: twin kinematic + colliders disabled + ConfigurableJoint → **completely safe** for mod (twin doesn't participate in physics when on hook).
4. **`attached = true`** blocks ExitBoat → item is "stuck" to boat, can't accidentally exit. But on `attached = false` (pickup) → twin dynamic again → collisions.
5. **Regular item on deck:** vanilla — twin isTrigger → "transparent" for CapsuleCollider → sits on walkCol (deck, layer 12). Mod — twin non-trigger → "stands on CapsuleCollider surface" → above deck → boat damage via BoatDamage.Impact.
6. **Mod solution:** while holding item — `Physics.IgnoreLayerCollision(twinLayer, boatLayer)` → twin doesn't collide with CapsuleCollider → twin falls through → EnterBoat → twin on walkCol (like vanilla). On drop — restore collision. Alternative: on mod drop → temporarily make twin isTrigger=true → twin passes CapsuleCollider → EnterBoat → walkCol → then twin normal.
