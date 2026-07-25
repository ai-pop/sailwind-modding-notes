# 58. Clickability and layers: what manual held-break breaks, pickup raycast

Breakdown of what OnDrop/OnPickup do, and why bypassing these methods breaks clickability — answer to requests A3, A4. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 47 (holding), 57 (DropItem).

## A3. ShipItem.OnDrop() — verbatim

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

**OnDrop does (for sold item):**
- If `wallAttachment && inRangeOfWall` → twin snap to wall + `attached = true` (kinematic lock).
- If `!sold` → `ReturnToShopPos()` (item not purchased → return to shop).

**OnDrop does NOT do for regular sold item (no wallAttachment):** nothing — empty pass-through. Drop side-effects for regular item — only in `GoPointer.DropItem()` (layer=0, held=null).

### ShipItem.OnPickup() — verbatim

```csharp
public override void OnPickup()
{
    if (wallAttachment)
    {
        itemRigidbodyC.attached = false;
    }
    if (itemRigidbodyC != null && itemRigidbodyC.GetCurrentInventorySlot() != null)
    {
        GPButtonInventorySlot component = itemRigidbodyC.GetCurrentInventorySlot().GetComponent<GPButtonInventorySlot>();
        if (component != null)
        {
            component.WithdrawItem();
        }
    }
    overrideEnableOutline = false;
}
```

**OnPickup does:**
1. `wallAttachment → attached = false` — removes kinematic lock (twin dynamic).
2. Inventory slot withdrawal — pulls out of inventory.
3. `overrideEnableOutline = false` — disables outline override.

### PickupableItem.OnPickup() / OnDrop() — verbatim

```csharp
public virtual void OnPickup() { }
public virtual void OnDrop() { }
```

**Base class — empty virtual methods.** All action in overrides on ShipItem and subclasses.

## Table: what vanilla does on drop → consequence of mod skipping it

Mod set `pointer.heldItem=null; item.held=null` directly, **not calling OnDrop() or DropItem()**.

| Vanilla action | Method | Consequence of mod skipping |
|----------------|-------|----------------------------|
| `PlaySmallItemDropSound()` | DropItem | No drop sound — cosmetic |
| `heldItem.gameObject.layer = 0` | DropItem | **CRITICAL: visual stays on layer 2 (IgnoreRaycast) → GoPointer raycast doesn't see item → item UNCLICKABLE forever!** |
| `heldItem.held = null` | DropItem | Mod did this itself — OK |
| `heldItem = null` (GoPointer) | DropItem | Mod did this itself — OK |
| `wallAttachment → attached=false` (OnPickup!) | OnPickup | If item was wallAttachment → mod didn't remove attached on pickup → twin kinematic → item doesn't move |
| `wallAttachment → twin snap to wall + attached=true` | OnDrop | Skip → item didn't stick to wall, twin free-falls |
| `overrideEnableOutline = false` | OnPickup | Item outline may stay blue/red |
| Inventory withdrawal | OnPickup | If item was in slot → not withdrawn → stuck in inventory |
| `itemRigidbodyC.attached = false` (wallAttachment OnPickup) | OnPickup | twin kinematic lock not removed → item frozen |

> **Root cause of unclickability:** `DropItem` sets `heldItem.gameObject.layer = 0`. Mod **skipped this** → visual GO stays on **layer 2 (IgnoreRaycast)**. GoPointer raycast mask `-604165` **excludes layer 2** (confirmed in note 52) → item **never** hit by pickup ray. Layer 2 = IgnoreRaycast — item "transparent" for raycast.

**Fix:** after manual held-break — set `item.gameObject.layer = 0` (as DropItem does). This restores clickability.

## A4. Visual GO layers across item lifecycle

### Table: method → visual layer

| Event | Method/Code | Visual layer | Twin layer |
|---------|-----------|:--:|:--:|
| Spawn (Start) | `ItemRigidbody.Start()` → `gameObject.layer = 2` (twin); visual inherits prefab layer | 0 (Default) or 2 (prefab) | 2 |
| Load (LoadAfterDelay) | `ProcessSaveable()`: if `parentObject == -3` → `layer = 2` | 2 | 2 |
| ResetPos | `ItemRigidbody.ResetPos()`: twin position = visual | — | 2 |
| PickUp | `GoPointer.PickUpItem()` → `item.gameObject.layer = 2` | **2** | 2 |
| Held (ongoing) | LateUpdate drives visual position; no layer changes | 2 | 2 |
| Drop (normal) | `GoPointer.DropItem()` → `item.gameObject.layer = 0` | **0** | 2 |
| Drop (wallAttachment) | OnDrop + DropItem → `layer = 0` | 0 | 2 (attached=true → kinematic) |
| Enter inventory | `ItemRigidbody.EnterInventorySlot()` → `layer = 5` + all children | **5** | 2 |
| Exit inventory | `ItemRigidbody.ExitInventorySlot()` → `layer = 2` + all children | **2** | 2 |
| Enter crate | `CrateInventory.InsertItem()` → `layer = 26` + all children | **26** | 2 |
| Exit crate | `CrateInventory.WithdrawItem()` → `layer = 2` + all children | **2** | 2 |

### DoRaycast: exact conditions

```csharp
LayerMask val2 = LayerMask.op_Implicit(-604165);
float num = 1.8f;
if (Physics.Raycast(val, ref hit, num, LayerMask.op_Implicit(val2)) 
    && !GameState.sleeping && !(GameState.inBed != null) && !BoatCamera.on)
```

**Mask `-604165` bitwise:** layer 2 (IgnoreRaycast) **excluded from raycast mask**. Meaning:
- visual on layer 2 → **not hit by ray** → not clickable.
- visual on layer 0 → **hit by ray** → clickable.
- `Physics.queriesHitTriggers` — Unity default = true → trigger colliders (isTrigger=true) **hit by raycast**.

**Distance:** `num = 1.8f` — raycast 1.8 m from camera/pointer.

**What breaks when skipping DropItem:**
- visual layer stays 2 → raycast mask excludes layer 2 → **item invisible to GoPointer** → forever unclickable.

> **Verdict:** sole cause of unclickability — **layer 2 instead of layer 0 on visual GO**. Fix: `item.gameObject.layer = 0` after mod's held-break.

## Practical conclusions for modders

1. **OnDrop + DropItem — mandatory pair.** DropItem sets layer=0 — **only** mechanism restoring raycast visibility. Skipping → item forever unclickable.
2. **OnPickup also mandatory** for wallAttachment items — removes attached=true (kinematic lock).
3. **Correct manual held-break for mod:**
   - `item.OnDrop()` — for wallAttachment snap and inventory withdrawal
   - `pointer.DropItem()` — for layer=0, sound, held=null
   - Or minimum: `item.gameObject.layer = 0; item.held = null; pointer.heldItem = null`
4. **Layer lifecycle:** spawn→2, pickup→2, drop→0, inventory→5, crate→26. After drop layer=0 — **Default**, visible for raycast mask.
5. **Raycast mask `-604165` excludes layer 2** — item on layer 2 invisible to GoPointer, regardless of collider type/isTrigger.
