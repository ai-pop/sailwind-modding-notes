# 68. Crate unsealing: frame order, `CrateInventory`, and mass

A precise examination of `ShipItemCrate.UnsealCrate()`, `CrateInventory.InsertItem()`, and `ItemRigidbody.UpdateMass()` in Sailwind v0.38. Complements notes [45](../45-crate-cargo-prefabs-filter.md), [61](../61-item-catalog-mass-table-units.md), and [65](65-item-mass-buoyancy.md).

## Main fact

A sealed crate calculates mass from `containedPrefab × amount`. After unsealing,
vanilla turns that data into live objects in `CrateInventory.containedItems`, but
**does not recalculate the crate mass from that new list**.

An unsealed crate can therefore contain goods while its vanilla `Rigidbody.mass`
is only the shell mass.

This is not an inference: the order of calls in the decompiled code is explicit.

## `UnsealCrate()` sequence

For each of `int amount` goods, the method does:

```csharp
GameObject go = Instantiate(
    containedPrefab,
    transform.position + new Vector3(0f, 100.5f, 0f),
    transform.rotation);

amount -= 1f;
go.GetComponent<SaveablePrefab>().RegisterToSave();
StartCoroutine(InsertItem(go.GetComponent<ShipItem>()));
```

`InsertItem` delays actual insertion:

```csharp
private IEnumerator InsertItem(ShipItem item)
{
    yield return new WaitForEndOfFrame();
    item.sold = true;
    crateInventory.InsertItem(item);
}
```

After starting all coroutines, but **before** their `WaitForEndOfFrame`,
`UnsealCrate()` immediately calls:

```csharp
UpdateLookText();
itemRigidbodyC.UpdateMass();
StartCoroutine(OpenAfterDelay());
```

At that point `amount` is already `0`, while `containedItems` is still empty.
`ItemRigidbody.UpdateMass()` consequently performs:

```csharp
rigidbody.mass = item.mass;
rigidbody.mass += containedPrefab.mass * amount; // amount == 0
```

The Rigidbody receives the **empty shell mass**.

## When goods actually enter the crate

At the next `EndOfFrame`, each coroutine calls:

```csharp
crateInventory.InsertItem(item);
```

`CrateInventory.InsertItem()`:

1. adds the `ShipItem` to `public List<ShipItem> containedItems`;
2. writes `SaveablePrefab.currentCrateId`;
3. sets the item's `ItemRigidbody.attached = true`, `disableCol = true`, and
   `inStove = true`;
4. reduces scale to `inventoryScale × 0.33`;
5. assigns layer `26` to the visual object and its children.

`CrateInventory.LateUpdate()` then holds every contained item at the crate's
position and rotation.

### Important missing call

Neither `CrateInventory.InsertItem()` nor `CrateInventory.WithdrawItem()` calls
`crate.itemRigidbodyC.UpdateMass()`.

Once the list becomes non-empty, no automatic mass correction occurs. The
vanilla formula knows only the old sealed `containedPrefab`/`amount` pair, not
`containedItems`.

## Full timeline

```text
frame N, UnsealCrate()
  amount: N → 0
  spawned goods: y = crate.y + 100.5
  InsertItem coroutines scheduled
  crate ItemRigidbody.UpdateMass()
    → shell mass + containedPrefab.mass × 0
    → shell mass

EndOfFrame N
  each InsertItem coroutine:
    item.sold = true
    CrateInventory.InsertItem(item)
    → containedItems gains real item
    → no crate UpdateMass call

EndOfFrame N+1
  OpenAfterDelay() → CrateInventory.OpenCrate()
```

## Practical implications for modders

1. `amount == 0` does not prove a crate is empty: after unsealing, its goods
   are in `CrateInventory.containedItems`.
2. For physics, weight, cargo capacity, or UI, calculate an opened crate as:
   ```text
   crate shell mass + Σ(item.mass for item in CrateInventory.containedItems)
   ```
   rather than `containedPrefab.mass × amount`.
3. A postfix on `UnsealCrate()` alone is too early: live goods appear only at
   EndOfFrame. Reliable integration points are postfixes on
   `CrateInventory.InsertItem` and `WithdrawItem`, or a deferred list read.
4. Do not use a spawned good's position at `Instantiate`: it is intentionally
   created at `+100.5Y` before being moved into the container.
5. `WithdrawItem()` removes the good and clears `attached`/`disableCol`/
   `inStove`; the container's aggregated mass must be recomputed again.
