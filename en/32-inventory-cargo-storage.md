# 32. Inventory, Cargo, and Storage

Analysis of item storage systems for the player and cargo transport. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Base item framework — in note 16.

## Storage Slot Encoding

Unified scheme (used in `SavePrefabData.inventorySlot`, note 11):

| `inventorySlot` Value | Where Item Is Placed |
|:--:|----------------------|
| `< 100` | Personal inventory slot (`GPButtonInventorySlot.inventorySlots[slot]`). |
| `>= 100` | Cargo carrier (`CargoCarrier.carriers[slot - 100]`, index = `portIndex`). |
| `-1` | Not in storage (in world/in hand). |

## Personal Inventory (`GPButtonInventorySlot`)

Static registry `inventorySlots[5]` — **5 personal player slots**.

| Field | Content |
|------|-----------|
| `slotIndex` | Slot index (0..4). |
| `currentItem` (`ShipItem`) | Item in slot. |

### Placing an Item (`OnItemClick`)
Item is placed in slot if **all** conditions are met:
- slot is empty (`currentItem == null`);
- item is purchased (`sold == true`);
- item is **not large** (`!big`);
- if it's a bottle (`ShipItemBottle`) — its volume `<= 10` (large bottles don't fit).

> **Important for loot:** large item (`big`), including **crate** (`ShipItemCrate`), **cannot go into inventory slot** — it's held **in hand**, placed **on deck**, and the crate is then **unsealed** → contents go into the crate's **`CrateInventory`**, not personal inventory (see notes 33, 45).

`InsertItem`: item enters slot (`ItemRigidbody.EnterInventorySlot`), sound. `WithdrawItem`: item removal (`ExitInventorySlot`).

## Cargo Carrier (`CargoCarrier`)

"Cart"/warehouse for transporting goods **between ports**. Static registry `carriers[50]` — **up to 50 carriers** (index = `portIndex`).

| Field | Content |
|------|-----------|
| `portIndex` | Port index (and index in `carriers`). |
| `currency` | Currency for carrier service payment. |
| `transportPriceMult` | Transport price multiplier. |
| `storagePriceMult` | Storage price multiplier. |
| `cargo` (`List<ShipItem`) | Stored goods. |

Subscribed to `Sun.OnNewDay` → `RegisterDayPassed`: every in-game day **`daysInStorage++`** for all cargo (storage fee accumulation).

### Loading (`InsertItem`)
- **Cannot load unsealed crates:** `ShipItemCrate` with `amount <= 0` → `"Cannot transport unsealed crates."`
- Money check; insufficient → `"Not enough money."`
- Transport price deducted, item hidden (`localScale = 0`), `daysInStorage = 0`.

### Transport Price (`GetTransportPrice`)
```
price = (121 + mass) * transportPriceMult * 0.016
       → GetBuyPriceInCurrency(currency, ...)
```
Heavier goods → more expensive transport (base surcharge 121 to mass).

### Withdrawal/Storage Price (`GetWithdrawPrice`)
```
days = daysInStorage - 1
fee  = transportPrice * days * storagePriceMult * 2.5
fee  = min(fee, item.value)         // no more than item's value
```
**Storage fee grows every day** — the longer cargo sits, the more expensive to retrieve (but never more than its value).

### Withdrawal (`WithdrawItem`)
Pay withdrawal price → item extracted (shifted forward 10 units), picked up by player, `daysInStorage = 0`. Cannot withdraw if already holding an item.

## Relation to Other Systems
- **Saving:** `inventorySlot` in `SavePrefabData` (note 11); on load `>= 100` → `CargoCarrier.LoadSavedItem`, `< 100` → `GPButtonInventorySlot.InsertItem`.
- **Spoilage in storage:** `daysInStorage` affects food spoilage (`FoodState`, note 23) and storage fees.
- **Crates** (`ShipItemCrate`): must be sealed (`amount > 0`) for transport; sealing — `CrateSealUI`.

## Practical Takeaways for Modding

1. **5 personal slots**, only for small (`!big`) purchased items; bottles — volume ≤ 10.
2. **Up to 50 cargo carriers** (by `portIndex`); item slot `>= 100` = `portIndex + 100`.
3. **Transport** costs `(121 + mass) * transportPriceMult * 0.016` in port currency.
4. **Storage gets more expensive daily** (`transportPrice * (days-1) * storagePriceMult * 2.5`, cap = item value) — incentive not to hold cargo long.
5. **Unsealed crates** cannot be transported (need sealing, `amount > 0`).
6. `daysInStorage` — shared day counter in storage (affects spoilage and fees); resets to 0 on load/unload.
