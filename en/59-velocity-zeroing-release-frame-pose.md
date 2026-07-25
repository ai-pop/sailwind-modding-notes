# 59. Who zeroes twin velocity after drop, item pose in release frame

Breakdown of all suspects that can zero/overwrite twin velocity after drop — answer to requests A5, A6. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 57 (DropItem), 44 (ItemRigidbody contract), 43 (buoyancy).

## A5. CRITICAL: who zeroes twin velocity after drop

### Suspect 1: `ItemRigidbody.FixedUpdate` — position syncs visual↔twin

**Verdict: CONFIRMED — main velocity killer.**

When `item.held != null`: twin is position slave (snap to visual every FixedUpdate). `twin.transform.position = item.transform.position` — **teleport every fixed frame**. Unity Rigidbody: setting `transform.position` directly on dynamic Rigidbody → **velocity zeroes** (internal Unity behavior — position assignment bypasses physics solver, resets velocity). Mod set velocity in LateUpdate, but **next FixedUpdate** overwrites twin position = visual position → velocity = 0.

When `held == null` (after drop): twin becomes position master (`visual.position = twin.position`). Twin position **not overwritten** — twin free to move via physics. **But:** if held was just set null in LateUpdate, but FixedUpdate hasn't processed yet → **1 fixed frame** twin may still be slave → velocity zeroed.

### Suspect 2: `fixedFramesSinceSpawn < 6` → isKinematic

**Verdict: CONFIRMED — second velocity killer (for fresh-spawn items).**

`fixedFramesSinceSpawn` — counter incremented every FixedUpdate. After spawn/ResetPos — `isKinematic = true` for first 6 fixed frames. Kinematic Rigidbody **doesn't apply velocity** — any velocity write is meaningless.

**But:** for items that existed > 6 fixed frames — timer already expired → not a problem. Problem only for WorldItemSpawner fresh items.

### Suspect 3: `dynamicColTimer`

**Verdict: NOT a velocity killer — but affects throw timing.**

`dynamicColTimer` does NOT set `isKinematic = true`. Only changes `collisionDetectionMode` to `ContinuousDynamic` for 6 seconds. Doesn't zero velocity.

### Suspect 4: `SimpleFloatingObject` (Crest floater)

**Verdict: NOT a velocity killer — floater doesn't write velocity.**

Crest `SimpleFloatingObject` applies **buoyancy forces** (AddForce/AddForceAtPosition), doesn't write velocity directly. In water — buoyancy upward + drag. On land — no forces. **Floater does NOT zero velocity** — it adds forces, but doesn't overwrite velocity.

### Suspect 5: `ResetPos()` — velocity = Vector3.zero

**Verdict: NOT called on drop — only on shop return and spawn.**

### Suspect 6: `Rigidbody.Sleep()` → isKinematic = true

**Verdict: Possible velocity killer — if twin "sleeps".** If twin `IsSleeping()` (Unity auto-sleep after ~1s no movement) + `meshCol != null` + `dynamicColTimer <= 0` → `isKinematic = true` → frozen. But sleep happens after **long inactivity**, not in first fixed frames after drop.

### Exact physics frame order on drop

```
Frame N (LateUpdate):
  mod: twinRb.velocity = v (5-9 m/s) ← written to Rigidbody
  mod: item.held = null ← held removed
  
Frame N+1 (FixedUpdate — first fixed after drop):
  ItemRigidbody.FixedUpdate:
    item.held == null → twin = position MASTER
    → twin.position NOT overwritten → velocity preserved?
    flag2 = false → isKinematic = false → twin dynamic → velocity works!
```

**Real mod problem:** mod set `twinRb.velocity = v` while `item.held != null` (still held) → **FixedUpdate of same/next frame** writes `twin.transform.position = visual.transform.position` (held != null → twin slave) → **velocity zeroed by teleport**.

> **Verdict:** velocity zeroed by `ItemRigidbody.FixedUpdate` position sync (twin → visual when held != null). Mod set velocity while held != null → FixedUpdate overwrites twin position → velocity = 0. **Solution:** set velocity **in FixedUpdate or later**, after `held == null` processed by ItemRigidbody (twin becomes position master).

## A6. Item pose in release frame

### Who writes visual world-pose when held

**Confirmation v0.38:** only writer of visual world-pose when held — **GoPointer.LateUpdate**. SetParent NOT used during hold — confirmed.

### Where visual lands in FIRST frame after DropItem

After DropItem → `heldItem = null` → GoPointer.LateUpdate no longer writes visual position. Visual position = last position written by GoPointer in **previous** LateUpdate.

**ItemRigidbody.FixedUpdate (next fixed):**
```csharp
// held == null, twin = position master
item.transform.position = transform.position;  // visual → twin position
```

**Twin position on drop:** twin was position slave when held → twin.transform.position = visual.transform.position. After drop → twin master → visual snap to twin. **If mod moved twin to physics position** → visual snap to twin physics position → **jump to twin pose** on next FixedUpdate.

> **For throw start point:** in drop frame, visual.position = pointer-driven position. Twin.position = visual.position (vanilla sync). **Mod twin** may be at different physics pose → first FixedUpdate after drop → visual jumps to twin physics position.

## Practical conclusions for modders

1. **Velocity zeroed by ItemRigidbody.FixedUpdate** — position sync `twin.position = visual.position` when `held != null` = teleport → velocity = 0. **Set velocity only after held == null processed** (twin becomes position master).
2. **fixedFramesSinceSpawn < 6** — twin kinematic for first 6 fixed frames. For long-existing items — not a problem.
3. **SimpleFloatingObject** — buoyancy forces, NOT velocity writer.
4. **Visual pose when held:** only GoPointer.LateUpdate. **After DropItem:** visual frozen at last pointer position. **Next FixedUpdate:** visual snap to twin (may jump).
5. **Correct throw sequence for mod:**
   - Frame N (LateUpdate): `OnDrop() + DropItem()` → held=null, layer=0
   - Frame N+1 (FixedUpdate): ItemRigidbody sees held=null → twin position master → isKinematic=false → twin free
   - Frame N+1 or N+2: **NOW set velocity** → twin free, velocity preserved
   - Or: use `WaitForFixedUpdate()` coroutine (like vanilla ThrowItemAfterDelay) → AddForce in next fixed frame
