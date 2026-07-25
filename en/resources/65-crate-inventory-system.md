# 65. Crate & Barrel Inventory System

> How sealed/unsealed containers work: CrateInventory, CrateSealUI, ShipItemCrate, mass flow.
> Complements notes 45 (Crate/Cargo), 62 (Crate Mass), 64 (Physics Model).

---

## Architecture

A crate exists in TWO states:

### Sealed (ShipItemCrate with containedPrefab + amount)

- `containedPrefab` (GameObject) + `amount` (float) on ShipItemCrate
- No CrateInventory component yet
- Mass = `baseMass + containedPrefab.mass * amount`
- Unseal via CrateSealUI

### Opened (has CrateInventory component)

- `UnsealCrate()` spawns `amount` items at `pos + (0, 100.5, 0)`
- Each spawned item registers to save
- Crate `amount` decrements to 0 during loop
- CrateInventory manages `containedItems` list

---

## Key Classes

### ShipItemCrate.UnsealCrate()

```
n = (int)amount
for i in 0..n:
    go = Instantiate(containedPrefab, pos + (0, 100.5, 0), rot)
    amount -= 1                           // LIVE mass change each iteration
    go.SaveablePrefab.RegisterToSave()
    if smokedFood: go.FoodState.AddSmoked(1f)
```

Items spawn 100.5 units ABOVE the crate.

### CrateInventory

| Method | Effect |
|--------|--------|
| InsertItem(item) | `attached=true, disableCol=true, inStove=true, layer=26, scale=0.33` |
| WithdrawItem(item) | `attached=false, disableCol=false, inStove=false, layer=2, scale=1` |
| LateUpdate() | Locks all contained items to `transform.position/rotation` |

### CrateInventoryUI

Grid from `containerMeshes[]`:
- Mesh[0]=4x3, Mesh[1]=5x4, Mesh[2]=6x5, Mesh[3]=4x2, default=6x5

### CrateSealUI

`Activate()` calls `currentCrate.UnsealCrate()` then `HideUI()`.

---

## Mass Flow

### Vanilla UpdateMass
```
rb.mass = item.mass + containedPrefab.mass * amount  // sealed only
```
After unseal `amount=0` -> vanilla stops counting contents.

### With FloatCrateContentsMass=true

Mod probes: sealed path (`containedPrefab+amount`) -> opened path (`CrateInventory.containedItems`) -> child ShipItems -> subclass modifiers.

---

## Layers

| Layer | Use |
|:-----:|-----|
| 2 | Free items |
| 26 | Items in crate (spoilage frozen) |

---

## Save/Load

`ShipItem.LoadAfterDelay()`: if `saveable.currentCrateId > 0` -> finds crate by instanceId -> `InsertItem(this)`.

---

*Extracted from Sailwind v0.38 decompilation.*
