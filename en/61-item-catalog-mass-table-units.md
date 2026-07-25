# 61. Item catalog: PrefabsDirectory, mass table, mass units

Breakdown of item catalog system, mass/flags table, mass units cross-check — answer to requests B1, B2, B3. Related to notes 16 (ShipItem), 44 (ItemRigidbody), 45 (crate/cargo).

## B1. Item catalog: where to get the full list

### `PrefabsDirectory` — catalog singleton

`PrefabsDirectory.instance` — contains **complete catalog of all ShipItem prefabs**.

```csharp
public GameObject[] directory;      // ← FULL prefab list by index
public ShipItem[] shipItems;        // ← populated from directory in Start()
```

**How to dump:** Runtime BepInEx plugin iterating `PrefabsDirectory.instance.shipItems` → log name/mass/big/category/floaterHeight/value/nailed/wallAttachment → CSV/JSON.

**Decompilation does NOT contain prefab array** — `directory[]` populated from Unity Inspector (scene references), not serialized in C#. **Runtime dump required for complete table.**

## B2. Item mass table (from decompilation known subclasses)

| Class | `mass` default | `big` | wallAttachment | nailed | Category | Mass modifiers |
|-------|:--:|:--:|:--:|:--:|:--:|---|
| `ShipItem` (base) | 1.0 (default) | false | false | false | — | overridden per prefab |
| `ShipItemCrate` | prefab-dependent | true | false | false | bulk* | `mass += containedPrefab.mass × amount` |
| `ShipItemBottle` | prefab-dependent | true | false | false | — | `mass += item.health` (fill level) |
| `ShipItemTea` | prefab-dependent | false | false | false | — | `mass += amount × 0.1` |
| `ShipItemSalt` | prefab-dependent | false | false | false | — | `mass += amount × 0.1` |
| `ShipItemSoup` | prefab-dependent | false | false | false | — | `mass += water + energy/20 + uncooked/20` |

### All items have ItemRigidbody twin — no "twinless" ShipItem in vanilla

Twin can be **disabled** (disableCol=true) in: HangableItem (hook), CrateInventory (crate), InventorySlot.

### `ShipItem.floaterHeight` default = 1.6 (buoyancy raise height)

## B3. Mass units — `ShipItem.mass` ≈ kilograms

Cross-check: boat baseMass ~300-500 kg + crew ~80 kg + items 1-20 kg → realistic for dhow scale.

**Mass examples (approximate):**

| Item type | Approx mass (kg) | Reasoning |
|-----------|:--:|------------|
| Food (fish, bread) | 1–2 | base default=1 |
| Small tool (hammer) | 1–2 | lightweight |
| Big crate (empty) | 5–10 | shell only |
| Big crate (10 items) | 5 + 10×1 | shell + contents |
| Big barrel | 10–20 | heavy container |

> **Complete table requires runtime dump of PrefabsDirectory.** Code contains only subclass mass modifiers, not prefab defaults.

## Practical conclusions

1. **Item catalog — runtime only:** need BepInEx dump for full name/mass/big/category table.
2. **All items have twin** — no twinless ShipItem. Twin disabled only in specific states.
3. **Mass ≈ kilograms** — realistic scale, 1 unit = 1 kg.
4. **Crate mass = base + contents × amount** — empty crate = base mass only.
5. **floaterHeight default = 1.6** — buoyancy raise height above water.
