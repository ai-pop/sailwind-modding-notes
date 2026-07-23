# 45. Crates/Cargo Prefabs and Filtering (LootTable)

Analysis of `ShipItemCrate` and how to distinguish "cargo crate with loot" from other items with the same component. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 16 (ShipItem), 32 (inventory/cargo), 33 (spawning/pickup), 44 (twin).

## `ShipItemCrate` — Universal "Container/Stack"

`ShipItemCrate : ShipItem` — component "contains N copies of another item". Not only wooden cargo crates have this, but also other "stacks/packs" of goods — e.g., **`tobacco big`**, **`apples`** and similar. Therefore `GetComponent<ShipItemCrate>()` **gives false positives** if you need specifically a cargo crate.

### Fields
| Field | Type | Content |
|------|-----|-----------|
| `containedPrefab` | `GameObject` (`[SerializeField] private`) | Prefab of item **inside**. |
| `amount` | `float` (inherited from `ShipItem`) | **Number of items inside**; `> 0` = sealed, `0` = empty/unsealed. |
| `smokedFood` | `bool` | Contents are smoked food (affects price/state). |
| `goodC` | `Good` (private) | Reference to `Good` (if crate is trade/mission cargo). |
| `crateInventory` | `CrateInventory` (private) | Crate inventory (added in `OnLoad`). |
| `crates` | `static Dictionary<int, CrateInventory>` | Crate registry. |

### `amount` / `containedPrefab` Semantics
- `amount > 0` → crate **sealed**, contains `amount` copies of `containedPrefab`.
- `amount == 0` → crate **empty/unsealed** (contents already in `CrateInventory` or absent).
- `containedPrefab == null` → "strange"/decorative prefab without real contents (best to avoid in LootTable).
- `OnLoad()` recalculates value: `value = containedValue * amount + 20` (× 1.2 if `smokedFood`).

### Unseal vs Open (`OnAltActivate`)
- **Sealed** (`amount > 0`) and **not mission** (`goodC == null || goodC.GetMissionIndex() == -1`) → `CrateSealUI.ShowUI(this)` (confirmation of **unsealing** → `UnsealCrate`: spawns `amount` copies of contents **into crate inventory**, `amount → 0`).
- **Empty** (`amount <= 0`) and has `CrateInventory` → `OpenCrate()` (open crate inventory).
- **Mission** crate (`goodC.GetMissionIndex() > -1`) can't be freely unsealed (delivered via mission).

## How "Cargo Crate" Differs from "Goods Stack"

At component level — **nothing fundamentally different**: `ShipItemCrate` is the same. Distinguish by **prefab data**:

| Criterion | What It Gives |
|----------|----------|
| `containedPrefab != null` | truly contains something (excludes empty/broken prefabs). |
| `amount > 0` | sealed (has loot). |
| Prefab name / pattern | "wooden" cargo crates typically have characteristic names (`crate…`); `tobacco big`, `apples` — "crate/stack of goods", not cargo container. |
| `Good` + `TransactionCategory` | tradeable crate (traded/carried) vs household stack. |
| Visual/mass | cargo crate heavier and looks like a container. |

> **Exact list of indices/names for "real" cargo crates can't be obtained from decompilation alone** — this is prefab/scene data. List assembled by **runtime enumeration** (see below) and name/content filtering.

## Runtime Enumeration and LootTable Filter

```csharp
// collect candidates once (e.g., in mod Awake, AFTER PrefabsDirectory initialization)
List<GameObject> cargoCrates = new();
foreach (var p in PrefabsDirectory.instance.directory)
{
    if (p == null) continue;
    var crate = p.GetComponent<ShipItemCrate>();
    if (crate == null) continue;

    // base filter: has contents
    var contained = crate.GetContainedPrefab();      // public method
    if (contained == null) continue;

    // exclude "goods stacks" (tobacco big, apples, etc.) — by name/category
    string n = p.name.ToLowerInvariant();
    bool isCargoCrate = n.Contains("crate")          // name heuristic
                        /* or own name/index table */;
    if (!isCargoCrate) continue;

    cargoCrates.Add(p);
}
// log cargoCrates (name + prefabIndex + containedPrefab.name + amount),
// to see actual list and refine filter per game version.
```

- `GetContainedPrefab()` — **public** method of `ShipItemCrate` (contents prefab).
- `amount` — public field of `ShipItem`.
- Prefab names/indices for precise filter — **logged at runtime** (decompilation doesn't provide). `name.Contains("crate")` heuristic — starting point; refine from log.
- "Empty"/strange prefabs (`containedPrefab == null` or `amount <= 0`) exclude from LootTable.

## Behavior on Crate Spawn (What Player Gets)

- Spawned crate with `amount > 0` — **sealed**; player picks it up **in hand** (large item doesn't fit inventory slot, note 32), places on deck, RMB → unseal → contents in `CrateInventory` → extract one by one.
- Crate **doesn't disappear** on unsealing — remains as container (note 33).
- Crate mass = `crate.mass + contained.mass * amount` (`UpdateMass`, note 44) — full crate is heavy.

## Practical Takeaways

1. `ShipItemCrate` = universal "container/stack of N items"; `GetComponent<ShipItemCrate>()` **catches not just cargo crates** (tobacco big, apples, etc.).
2. `amount` = number of items inside (`>0` sealed); `containedPrefab` = what's inside (`GetContainedPrefab()`); `containedPrefab == null` → empty/strange prefab.
3. "Real cargo crate" differs by **prefab data** (name/content/category), not by component type; exact list — runtime enumeration + filter (`name.Contains("crate")` heuristic + log).
4. Unsealing (`amount>0`, not mission) → `CrateSealUI` → contents into `CrateInventory`; mission crate can't be freely unsealed.
5. For LootTable: filter `containedPrefab != null` + `amount > 0` + exclude "goods stacks" by name/category.
