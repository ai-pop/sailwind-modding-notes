# 53. EnterBoat/ExitBoat: call mechanics and flap potential

Breakdown of who and when calls `EnterBoat()`/`ExitBoat()` for items, and whether "flapping" can occur — answer to request E3. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 51 (colliders), 52 (layers), 16 (ShipItem/twin).

## Who calls EnterBoat/ExitBoat for items

### `ShipItem.ExtraFixedUpdate` — main driver

Called every `FixedUpdate` (via `PickupableItem.ExtraFixedUpdate` → `ShipItem.ExtraFixedUpdate`):

```csharp
// ShipItem.cs, ExtraFixedUpdate — verbatim
public override void ExtraFixedUpdate()
{
    if (Debugger.instance.disableItemUpdate) return;
    int num = 1;                          // frameCounter threshold
    if (GameState.currentlyLoading || GameState.justStarted) num = 0;

    // If ALREADY on boat — check whether we need to EXIT
    if (currentActualBoat != null)
    {
        // Counter ticks if we're not in the correct EmbarkCol
        if (!GameState.sleeping && gameObject.layer != 26 
            && (currentlyStayedEmbarkCol == null || currentlyStayedEmbarkCol != currentBoatCollider))
        {
            frameCounter++;               // tick towards exit
        }
        else
        {
            frameCounter = 0;              // we're in correct EmbarkCol — reset
        }
    }

    // If NOT on boat — check whether we need to ENTER
    if (currentActualBoat == null)
    {
        if (currentlyStayedEmbarkCol != null)
        {
            frameCounter++;               // tick towards enter
        }
        else
        {
            frameCounter = 0;              // no EmbarkCol — reset
        }
    }

    // Enter boat
    if (currentlyStayedEmbarkCol != null 
        && currentlyStayedEmbarkCol != currentBoatCollider 
        && frameCounter > num)
    {
        EnterBoat(currentlyStayedEmbarkCol);
    }
    // Exit boat
    else if (currentlyStayedEmbarkCol == null 
        && currentActualBoat != null 
        && frameCounter > num 
        && !GameState.sleeping 
        && !disallowDisembarking 
        && !itemRigidbodyC.attached)
    {
        ExitBoat();
    }
}
```

**Threshold:** `frameCounter > 1` (under normal GameState) → state change requires **2+ fixed frames**. This prevents instant flapping, but **not periodic** flapping.

### Trigger callbacks

`ShipItem.OnTriggerEnter(Collider other)`: if `other.CompareTag("EmbarkCol")` → `currentlyStayedEmbarkCol = other`.
`ShipItem.OnTriggerExit(Collider other)`: if `currentlyStayedEmbarkCol == other` → `currentlyStayedEmbarkCol = null`.

**Opposite collider:** `BoatEmbarkCollider` (tag `EmbarkCol`) on the boat. This is a **trigger collider** (isTrigger=true).

## `ItemRigidbody.EnterBoat()` — what it does with twin

```csharp
// ItemRigidbody.cs — verbatim
public void EnterBoat()
{
    transform.parent = item.currentWalkCol;          // twin → child of walkCollider
    MoveRigidbodyToWalkCol();                        // snap twin position to walkCol
    item.currentActualBoat.parent.GetComponent<BoatMass>().AddItem(this);
    onBoat = true;
}
```

### `MoveRigidbodyToWalkCol` — verbatim unavailable (ILSpy unresolved), but from context:
- twin position/rotation is synchronized with the boat's walkCollider.
- twin becomes a child of walkCollider — **reparent**.

## `ItemRigidbody.ExitBoat()` — what it does with twin

```csharp
// ItemRigidbody.cs — verbatim
public void ExitBoat()
{
    if (item.currentActualBoat != null)
    {
        attached = false;
        transform.parent = world;                     // twin → child of FloatingOriginManager
        transform.position = item.transform.position; // twin snap to visual
        transform.rotation = item.transform.rotation; // twin snap to visual
        item.currentActualBoat.parent.GetComponent<BoatMass>().RemoveItem(this);
        onBoat = false;
    }
}
```

## `BoatMass.AddItem/RemoveItem` — idempotency

```csharp
// BoatMass.cs — verbatim
public void AddItem(ItemRigidbody itemBody)
{
    if (!itemsOnBoat.Contains(itemBody))              // ← IDEMPOTENT
    {
        itemsOnBoat.Add(itemBody);
    }
}

public void RemoveItem(ItemRigidbody itemBody)
{
    itemsOnBoat.Remove(itemBody);                     // Remove for non-existent → does nothing
}
```

**`AddItem` is idempotent:** re-adding the same `ItemRigidbody` **is not added** (Contains check). `RemoveItem` for a missing element — `List.Remove` returns false but doesn't crash.

**But:** `UpdateMass()` is called every `Update()` (for active boats) and **recalculates** everything:
- `itemsOnBoat.RemoveAll(item => item == null)` — cleans null references (if twin was destroyed).
- Then computes mass of all itemsOnBoat + player + sails.

> **NaN does not arise from repeated Add/Remove** — but if Add occurs without Remove (flap: Enter without Exit), the item **remains in itemsOnBoat** → its mass **is counted twice** only if Add passes (but Contains blocks). If ExitBoat occurred, RemoveItem removes. If EnterBoat-EnterBoat without Exit — second Add is blocked by Contains. **Enter→Exit→Enter→Exit flap** — safe for BoatMass.

## Can Enter/Exit flap every physics frame?

### `currentlyStayedEmbarkCol` mechanics

`ShipItem.OnTriggerEnter(EmbarkCol)` sets `currentlyStayedEmbarkCol = other` — and `ExtraFixedUpdate` starts ticking `frameCounter`. Threshold = 1 (normally). So Enter/Exit requires **≥2 fixed frames** in trigger volume.

**Mod problem:** with a solid twin-collider, the item can **physically collide** with the boat's CapsuleCollider → physics pushes twin out → twin exits trigger volume (`OnTriggerExit`) → `currentlyStayedEmbarkCol = null` → ExitBoat → item reparents to world → mod spring pulls back → twin enters trigger volume → EnterBoat → twin reparents to walkCollider → physics pushes out → OnTriggerExit → ExitBoat… **Loop!**

**Deduction:** `OnTriggerEnter/Exit` for `EmbarkCol` is a trigger-collider. If twin is physically inside CapsuleCollider, but the trigger-collider `EmbarkCol` is a **separate volume** (different GO, different position). Depending on geometry:
- trigger `EmbarkCol` may be **larger/smaller/separate** from CapsuleCollider.
- twin may be inside trigger `EmbarkCol` (OnTriggerEnter) → EnterBoat → reparent to walkCol → twin position changes → **physically overlaps CapsuleCollider** → but trigger status per `EmbarkCol` doesn't change (OnTriggerExit not called if twin didn't exit from `EmbarkCol` trigger).

**However:** if the mod spring moves twin **outside the EmbarkCol trigger volume**, `OnTriggerExit` fires → ExitBoat → `currentlyStayedEmbarkCol = null` → `frameCounter++` → after 2 frames → ExitBoat again (already done). And conversely: spring pulls inside → `OnTriggerEnter` → EnterBoat. Each cycle = 2 fixed frames minimum.

> **Potential flap:** Enter/Exit can cycle with period ≥2 fixed frames (~0.044 s at fixedDt=0.022). This is not "every frame", but ~22 Hz flap → massive AddItem/RemoveItem (idempotent, but **BoatMass.UpdateMass recalculated every Update** — 22 times per second changes mass/centerOfMass → unstable simulation).

## `ShipItem.ExitBoat` (visual-side) — full side effects

```csharp
// ShipItem.cs — verbatim
private void ExitBoat()
{
    Debug.Log(name + ": ExitBoat");
    if (saveable != null && saveable.GetParentObject() != -3)
    {
        saveable.SetParentObject(-1);         // parent = world
    }
    if (itemRigidbody != null)
    {
        itemRigidbody.GetComponent<ItemRigidbody>().ExitBoat();  // twin ExitBoat
        if (held == null && GetComponent<HangableItem>() != null)
        {
            GetComponent<HangableItem>().DisconnectJoint();     // detach from hook
        }
    }
    currentWalkCol = null;
    currentActualBoat = null;
    currentBoatCollider = null;
    transform.parent = world;                         // visual → world
}
```

ExitBoat side effects: `saveable.SetParentObject(-1)`, twin ExitBoat (reparent+mass), HangableItem disconnect, clear walkCol/actualBoat/boatCollider, visual reparent to world.

## Practical conclusions

1. **Enter/Exit gated via `frameCounter > 1`** — not instant, but ≥22 Hz flap is possible with a spring pushing twin out of EmbarkCol trigger volume.
2. **BoatMass.AddItem is idempotent** (Contains check) — repeated Add of the same twin doesn't break mass. RemoveItem is safe. **NaN from flap does not occur**, but `UpdateMass()` is called every `Update` — during flap, mass/centerOfMass "flickers" (item enters/leaves → boat mass oscillates).
3. **Flap scenario for crash:** Enter→twin reparent to walkCol → twin position inside CapsuleCollider → OnCollisionEnter on BoatImpactSoundsCollider → `BoatImpactSounds.Impact(collision)` → `BoatDamage.Impact` → **if HeldItem+solid collider = OnCollisionEnter every frame** → BoatDamage cooldown (1s) won't save from massive Impact calls → or sound spam, or repeated hullDamage.
4. **Mod solution:** while holding item — Harmony-prefix skip `ShipItem.ExtraFixedUpdate` (don't Enter/Exit), or Harmony-prefix `OnTriggerEnter/Exit` for EmbarkCol when `held != null`, or `Physics.IgnoreLayerCollision` twin↔boat while held.
5. **Important:** trigger-collider `EmbarkCol` — separate GO from boat's CapsuleCollider. Solid twin can EnterBoat via trigger, then collide with CapsuleCollider (non-trigger) — two different collision systems.
