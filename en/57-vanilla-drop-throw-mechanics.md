# 57. Vanilla drop/throw: DropItem, throw charge, ThrowItemAfterDelay

Complete breakdown of vanilla drop and throw mechanism — answer to requests A1, A2. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 47 (holding flow), 44 (ItemRigidbody contract).

## A1. GoPointer.DropItem() — verbatim

```csharp
public void DropItem()
{
    if (Object.op_Implicit((Object)(object)heldItem))
    {
        UISoundPlayer.instance.PlaySmallItemDropSound();
        ((Component)heldItem).gameObject.layer = 0;
        heldItem.held = null;
        heldItem = null;
    }
}
```

**DropItem does ONLY 4 things:**
1. `PlaySmallItemDropSound()` — drop sound (short click).
2. `heldItem.gameObject.layer = 0` — **visual GO to layer 0 (Default)** — item visible again for raycast/interact.
3. `heldItem.held = null` — removes "held" flag from item.
4. `heldItem = null` — clears GoPointer's reference to item.

**DropItem DOES NOT:**
- Does not call `heldItem.OnDrop()` — that's a **separate call** before DropItem!
- Does not change collider.enabled/isTrigger on visual — OnDrop does this for wallAttachment, but DropItem does NOT.
- Does not change twin (ItemRigidbody) — twin managed by its own FixedUpdate.
- Does not reset `currentThrowPower` — done **in LateUpdate** after DropItem call.
- Does not change `nailed`, `attached`, `sold`, `pointedAtBy` — all remain as-is.

### All DropItem call sites — table

| File | Method | Condition | OnDrop precedes? |
|------|-------|---------|---------------------|
| GoPointer.cs | LateUpdate | `GameInput.GetKeyUp(InputName 10)` or `(Settings.autoThrow && GetKeyUp(InputName 8))` + `collisions <= 0 || allowObstructedDropping || GetKeyUp(10)` | **Yes**: `heldItem.OnDrop(); DropItem();` — OnDrop CALLED BEFORE DropItem |
| GoPointer.cs | LateUpdate | `MainButtonDown()` + `pointedAtButton.OnItemClick(heldItem)` → return true | **Yes**: `heldItem.OnDrop(); DropItem();` |
| GoPointer.cs | LateUpdate | `MainButtonDown()` + heldItem not ShipItem (WorldItem) | **Yes**: `heldItem.OnDrop(); DropItem();` |
| Anchor.cs | FixedUpdate | Anchor set/release: `if (held) held.DropItem()` | **No**: Anchor calls DropItem WITHOUT OnDrop |
| PickupableBoatMooringRope.cs | Update | Rope length exceeded: `held.DropItem()` | **No**: only DropItem without OnDrop |

> **CRITICAL:** in 3 main drop paths (throw, item-click, world-item) **OnDrop is called BEFORE DropItem**. Anchor and MooringRope call only DropItem (without OnDrop). Mod that set `heldItem=null; item.held=null` without OnDrop/DropItem — skips EVERYTHING both methods do (see note 58).

### Who decides "place" vs "throw"

**Key:** `GameInput.GetKeyUp(InputName 10)` — release of throw/drop key. This is **KeyUp**, not KeyDown.

Branch in LateUpdate:
```csharp
if (heldItem != null && heldItem.GetComponent<ShipItem>() != null 
    && timerAfterPickup >= 0.66f && pointedAtButton == null)
{
    // THROW CHARGE (hold)
    if (GameInput.GetKey(InputName 10) || (Settings.autoThrow && GameInput.GetKey(InputName 8)))
    {
        currentThrowPower += Time.deltaTime;
        if (currentThrowPower > 1f) { currentThrowPower = 1f; }
    }
    // RELEASE → drop/throw
    else if (GameInput.GetKeyUp(InputName 10) || (Settings.autoThrow && GameInput.GetKeyUp(InputName 8)))
    {
        if (heldItem.colChecker.collisions <= 0 
            || heldItem.colChecker.allowObstructedDropping 
            || GameInput.GetKeyUp(InputName 10))
        {
            Rigidbody component = heldItem.GetComponent<ShipItem>().GetItemRigidbody().GetComponent<Rigidbody>();
            heldItem.OnDrop();
            DropItem();
            if (currentThrowPower > throwDelay)  // throwDelay = 0.4
            {
                StartCoroutine(ThrowItemAfterDelay(component, currentThrowPower - throwDelay));
            }
        }
        currentThrowPower = 0f;
    }
}
else
{
    currentThrowPower = 0f;
}
```

**Logic:**
- If `currentThrowPower > throwDelay (0.4)` → throw (ThrowItemAfterDelay).
- If `currentThrowPower <= throwDelay` → "just place" (DropItem without ThrowItemAfterDelay).
- `Settings.autoThrow` — if enabled, InputName 8 (regular interact/drop) also charges throw.
- **Charge start condition:** `timerAfterPickup >= 0.66f` — cannot throw in first 0.66 s after PickUp + **not pointing at any button** (pointedAtButton == null).

**Obstruction guard:** `collisions <= 0 || allowObstructedDropping || GetKeyUp(10)` — if item colliding (red outline) → drop blocked, **except** throw key (InputName 10). Throw key drops even with red outline.

## A2. Throw charge/force mechanics

### Fields

| Field | Type | Value | Content |
|------|-----|----------|------------|
| `throwDelay` | `float` | 0.4 (public, SerializeField) | Threshold: if `currentThrowPower > throwDelay` → throw, else "place" |
| `throwForce` | `float` | 10 (public, SerializeField) | Throw force multiplier |
| `currentThrowPower` | `float` | private | Current charge (0–1), increases `+= Time.deltaTime` on hold |
| `timerAfterPickup` | `float` | private | Timer after PickUp — throw available only at `>= 0.66f` |

### ThrowItemAfterDelay — verbatim

```csharp
private IEnumerator ThrowItemAfterDelay(Rigidbody heldRigidbody, float force)
{
    yield return (object)new WaitForFixedUpdate();
    if (force > 1f)
    {
        force = 1f;
    }
    heldRigidbody.AddForce(((Component)this).transform.forward * throwForce * force * heldRigidbody.mass);
}
```

**Formula:** `AddForce(pointer.forward * throwForce * force * mass)` = `pointer.forward * 10 * (currentThrowPower - 0.4) * mass`

**ForceMode:** default `ForceMode.Force` (not Impulse, not VelocityChange) — force applied **for one fixed-frame** as `F = m*a`, so `Δv = F*dt/m = throwForce*force*dt`. With `fixedDt=0.022`: `Δv = 10 * 0.6 * 0.022 ≈ 0.13 m/s` — but **mass is multiplied** in AddForce → `Δv = throwForce*force*dt = 10*0.6*0.022 = 0.132 m/s` (mass cancels! `F = throwForce*force*mass → a = F/m = throwForce*force → Δv = a*dt`).

> **CRITICAL:** `AddForce` with `ForceMode.Force` and mass in formula → **mass does NOT affect final velocity**. `Δv = throwForce * force * fixedDt ≈ 10 * 0.6 * 0.022 = 0.132 m/s` at max charge. This is a **very small** velocity — item after throw flies ~0.13 m/s forward, not 5–9 m/s as mod attempted.

**Delay:** `WaitForFixedUpdate()` — force applied **in next FixedUpdate** after drop. This means: in drop frame `OnDrop() + DropItem()` remove held → same frame LateUpdate already passed → **next FixedUpdate** twin first becomes dynamic (6 fixed frames after spawn timer) → **+1 fixed** → ThrowItemAfterDelay fires AddForce.

> **Mod problem:** mod set velocity manually (~5–9 m/s) in release frame. But `ItemRigidbody.FixedUpdate` in first ~6 fixed frames keeps twin kinematic (`fixedFramesSinceSpawn < 6`) → **velocity written to kinematic Rigidbody → meaningless** (kinematic doesn't apply velocity). When twin finally dynamic (frame 7+), its velocity = 0 (kinematic doesn't preserve velocity) → item falls like a stone.

## Practical conclusions for modders

1. **DropItem is minimal:** only sound + layer=0 + held=null. **OnDrop() is a separate call**, before DropItem. Skipping OnDrop = skipping wallAttachment snap, attached=false, inventory withdrawal — item loses "sticking".
2. **Throw via AddForce with WaitForFixedUpdate:** force applied in **next fixed frame**, ForceMode.Force, mass in formula cancelling → final Δv ≈ 0.13 m/s (max). Vanilla throw is **weak**, item barely flies forward.
3. **"Place" vs "throw"** — threshold `throwDelay = 0.4` s. If held key < 0.4 s → DropItem without ThrowItemAfterDelay = "just place". > 0.4 s → throw.
4. **TimerAfterPickup ≥ 0.66 s** — cannot throw immediately after Pickup. Drop via regular button (InputName 8) with `Settings.autoThrow` also charges.
5. **Mod that set velocity manually** — twin kinematic in first 6 fixed frames → velocity = 0 after kinematic→dynamic transition. Must either: wait 6+ fixed frames before setting velocity, or set `fixedFramesSinceSpawn = 7+` manually, or write velocity after `isKinematic = false` in **next** FixedUpdate.
