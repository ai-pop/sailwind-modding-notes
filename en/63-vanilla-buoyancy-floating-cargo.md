# 63. Vanilla buoyancy reference and cargo overboard

> **Update:** an exact decompilation of `Crest.dll` is now published in [note 71](71-crest-simplefloatingobject-exact-model.md). The older discussion below about whether `SimpleFloatingObject` runs is retained as research history; use note 71 for actual cubic `ForceMode.Acceleration` lift, `_raiseObject`, drag, and ownership behavior.

Breakdown of vanilla buoyancy behavior and floating cargo spawning — answer to requests B6, B7. Related to notes 43, 61.

## B6. Vanilla reference buoyancy

### `SimpleFloatingObject` — Crest DLL (not decompiled)

Known from ItemRigidbody.Start():
```csharp
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;  // default 1.6
```

| Field | Type | Default | Content |
|-------|------|---------|---------|
| `_raiseObject` | float | 1.6 | Target height above water surface |
| `_dragInWaterRotational` | float | 0.02 | Rotational drag in water |
| `InWater` | bool | runtime | Object currently in water |

**Vanilla reference:** item floats with **bottom ≈ at water level, top ≈ at `_raiseObject` (1.6 m) above**. Item sits **high** on water — very visible.

### `ToggleCollider` disables floater

`ToggleCollider(true)` → `floater.enabled = false`. Twin free (sold, dynamic) → floater OFF → buoyancy NOT applied. **But vanilla items float** — exact mechanism unclear from decompilation (may involve script execution order or other Crest paths).

> **Mod approach — kill floater, replace with honest Archimedes — is correct.** Vanilla floater is "magical" force-based, not physics-accurate.

### No "sinking" items in vanilla

All items have `_raiseObject > 0` → all float. **No vanilla mechanism for sinking.** Mod must add sinking (mass > displacement → sinks).

## B7. Cargo overboard in vanilla

**Vanilla does NOT spawn floating/lost cargo.** WorldItemSpawner — fixed positions on docks/shore, not sea.

**Only "cargo overboard"** = player's dropped item in water → twin dynamic → SimpleFloatingObject buoyancy → item **floats** at `_raiseObject` level.

### Calibration reference

Vanilla item → sits 1.6 m above water. Honest Archimedes → draft = mass/(bounds×1025). Light items → close to vanilla. Heavy items → lower — realistic, but may deviate from player expectations.

**Need runtime dump of twin collider bounds.size** for each item to compute displacement volume.

## Practical conclusions

1. **SimpleFloatingObject — Crest DLL, not decompiled** — `_raiseObject = 1.6` default, force-based, not physics-accurate.
2. **Mod killing floater + honest Archimedes — correct approach.**
3. **Vanilla does NOT spawn cargo overboard** — only player drops.
4. **Calibration:** draft = mass/(bounds×1025). Need runtime bounds dump per item.
5. **No sinking items in vanilla** — all float. Mod must add sinking logic.
