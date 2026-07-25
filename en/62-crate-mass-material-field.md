# 62. Crate mass with loot, item material field

Breakdown of crate mass with contents and material field — answer to requests B4, B5. Related to notes 16, 61.

## B4. Crate mass with loot

### Formula: `twinRigidbody.mass = crate.mass + containedPrefab.ShipItem.mass × amount`

**Crate mass accounts for sealed contents** via `amount`, **NOT** via `CrateInventory.containedItems`.

After unseal → `amount` decremented → `UpdateMass()` recalculated → empty crate mass = `item.mass` (base only).

**Bug/feature:** `UpdateMass()` does NOT account for `CrateInventory.containedItems` in opened crate mass. Opened crate with 5 items inside → twin mass = crate.mass (base), not crate.mass + 5×content.mass. **Mod must count contained items itself for buoyancy.**

### `CrateInventory.InsertItem/WithdrawItem` do NOT call `UpdateMass()` — only `UnsealCrate` and `SpawnContainedPrefab` call it.

## B5. Item material field

**Answer: NO — no explicit material/material-type field on ShipItem.**

`TransactionCategory category` — economic category enum, **not material**.

All items use `PlayWoodColSound` by default — vanilla doesn't differentiate material by sound.

**No `durability` / `breakable` / `sinkable` field** — items **don't break** and **don't sink** in vanilla (SimpleFloatingObject always pushes up with `_raiseObject > 0`).

**Implicit material indicators:** ShipItemBottle = glass, ShipItemCrate = wood, ShipItemHangable = metal (lamp).

## Practical conclusions

1. **Crate mass (sealed):** accounts for `amount × containedPrefab.mass`.
2. **Crate mass (opened):** UpdateMass does NOT count containedItems → **mod must count itself**.
3. **No material field** — TransactionCategory is economic, not material.
4. **No sinking items in vanilla** — `_raiseObject > 0` → all float. Mod must add sinking logic.
5. **floaterHeight default = 1.6** — buoyancy target height.
