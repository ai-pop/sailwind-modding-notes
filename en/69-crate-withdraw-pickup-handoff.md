# 69. Withdrawing from a crate: the `WithdrawItem → PickUpItem` handoff

The exact vanilla sequence for taking an item from an open `CrateInventory` in Sailwind v0.38. Related to [45](../45-crate-cargo-prefabs-filter.md), [47](47-item-holding-pickup-flow.md), and [68](68-crate-unseal-timing-and-mass.md).

## Critical call order

When the player's hands are empty, the crate button (`CrateInventoryButton.OnActivate`) makes **two consecutive calls in the same frame**:

```csharp
CrateInventoryUI.instance.currentCrate.WithdrawItem(currentItem);
activatingPointer.PickUpItem(currentItem);
```

There is no coroutine, physics tick, or EndOfFrame between them.

```text
CrateInventoryButton.OnActivate
  1. CrateInventory.WithdrawItem(item)
  2. GoPointer.PickUpItem(item)
  3. CrateInventoryUI.RefreshButtons()
  4. later: GoPointer.LateUpdate writes the in-hand item pose
  5. later: ItemRigidbody.FixedUpdate synchronizes the twin
```

This is a transfer between two pose owners: container/UI → hand/pointer. A mod
that wakes free physics in the middle of this handoff can leave the twin at the
old UI/crate pose while the visual has already moved into the player's hand.

## What `CrateInventory.WithdrawItem` actually does

```csharp
containedItems.Remove(item);
item.SaveablePrefab.currentCrateId = 0;
item.GetItemRigidbody().attached = false;
item.GetItemRigidbody().disableCol = false;
item.GetItemRigidbody().inStove = false;
item.transform.localScale = Vector3.one;
item.gameObject.layer = 2;
foreach (Transform t in item.GetComponentsInChildren<Transform>(true))
    t.gameObject.layer = 2;
```

It does **not**:

- call `ResetPos()`;
- put the twin at the visual pose;
- directly make the Rigidbody dynamic;
- move the item to the player;
- notify the parent crate's `ItemRigidbody.UpdateMass()`.

It only removes container flags and the list entry. Until the next stage, the
item can still be at the crate or UI-button pose.

## What `GoPointer.PickUpItem` immediately does

After `WithdrawItem`, vanilla pointer code performs:

```csharp
heldItem = item;
item.gameObject.layer = 2;
heldItem.held = this;
item.OnPickup();
```

`held` becomes non-null in the same call stack. However, the final visual pose
in the hand is written later in pointer `LateUpdate`; the twin follows the
visual in `ItemRigidbody.FixedUpdate`.

## Why the UI can temporarily own the item pose

While the crate UI is open, `CrateInventoryUI.LateUpdate()` calls:

```csharp
button.UpdateItemPos();
```

for every button. `CrateInventoryButton.UpdateItemPos()` writes:

```csharp
currentItem.transform.position = button.transform.position;
currentItem.transform.rotation = button.transform.rotation * inventoryRotation;
currentItem.itemRigidbodyC.ForceRigidbodyToWalkCol();
```

`RefreshButtons()` clears the withdrawn button, but an external mod must not
assume that the old UI pose is gone before the frame ends.

## Safe recipe for a mod with its own physics

In a postfix of `CrateInventory.WithdrawItem`:

1. mark the body as `containerTransfer` and **do not apply buoyancy/forces**;
2. wait for `WaitForEndOfFrame` — vanilla has time to call `PickUpItem` and
   write visual pose;
3. wait for the first `WaitForFixedUpdate`;
4. call vanilla `shipItem.ResetRigidbody()` or equivalently place the twin at
   the current visual pose;
5. return ownership to free physics only afterwards, and only when `held == null`.

Do not run custom free physics between `WithdrawItem` and `PickUpItem`: this is
a one-frame race between the container pose and the hand pose.

## Practical implications

1. `WithdrawItem()` is not a full physical release; it merely clears container
   flags.
2. `held` is assigned immediately afterwards but hand pose is later; a handoff
   guard must last through EndOfFrame plus the next fixed tick.
3. The opened crate and withdrawn item need independent mass updates: the parent
   loses mass while the child regains its own mass.
4. If an item hangs at an old crate/UI position, first check for a missing
   post-handoff `ResetRigidbody()`, not buoyancy.
