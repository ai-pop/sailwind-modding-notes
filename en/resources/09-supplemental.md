# 09. Fishing, Quests & Boat Parts — Supplemental

> Additional resource details: fishing system mechanics, quest items, boat customization parts, and starter sets.
> Complements notes 23 (Fishing), 27 (Story Quests), 22 (Shipyard).

---

## Fishing System

### OceanFishes

Global fish spawning singleton at `OceanFishes.instance`.

| Field | Type | Description |
|-------|------|-------------|
| `fishPrefabs[]` | GameObject[] | All fish prefabs, indexed by latitude preference |
| `peakLatitude[]` | float[] | Ideal latitude for each fish (parallel to `fishPrefabs`) |
| `deviationDistance` | float | Random deviation for catch variety |
| `localFishesRegions[]` | LocalFishesRegion[] | Per-region fish overrides |

**Fish Selection Logic:**
```
GetFish(pos):
  1. Check localFishesRegions — if within radius, weighted chance to return local fish
  2. Otherwise: get globe latitude + random deviation
  3. Find fish with closest peakLatitude to current latitude
  4. Return that fish prefab
```

### LocalFishesRegion

Defines regional fish populations near islands/coasts.

| Field | Type | Default | Description |
|-------|------|:-------:|-------------|
| `outerRadius` | float | 5000 | Outer detection radius |
| `innerRadius` | float | 2500 | Inner (guaranteed) radius |
| `overrideInfluence` | float | 0.75 | Chance override (0–1) |
| `localFishPrefabs[]` | GameObject[] | — | Regional fish pool |

**Selection:** Random pick from `localFishPrefabs[]` when player is within influence radius.

### Fishing Mechanics (FishingRodFish)

| Field | Type | Default | Description |
|-------|------|:-------:|-------------|
| `fishPullForce` | float | 1.0 | Base fish pull force |
| `pullTensionMult` | float | 1.0 | Tension buildup multiplier |
| `lowerForceThreshold` | float | — | Minimum force for tension |
| `reelBendMult` | float | — | Reel effect on tension |
| `fishTimer` | float | 6.0 | Time between catch attempts |
| `fishEnergy` | float | 1.0 | Fish fight energy (0=exhausted) |

**Catch Flow:**
1. Hook in water + line length > 1 → timer counts down
2. On timer expire → `CatchFish()`: gets fish from `OceanFishes`
3. Player fights fish: tension builds via `currentTargetTension`
4. Tension > 0.95 → snap timer (3.1s max before line breaks)
5. Fish collected → `CollectFish()`: spawns `ShipItem` version, `sold=true`, 30% chance hook lost

---

## Quest Items

`QuestItem` is a simple MonoBehaviour that auto-marks items as sold:
```csharp
class QuestItem : MonoBehaviour {
    int questItemIndex;
    void Awake() { GetComponent<ShipItem>().sold = true; }
}
```

Known quest item prefabs:
| Prefab Index | Name | Purpose |
|:-----------:|------|---------|
| 330 | quest0 letter | Quest letter/document |
| 331 | quest0 cargo | Quest cargo item |

Additional quest indices likely exist for different quests (quest1, quest2, etc.) but their prefab indices weren't captured in the string extraction.

---

## Boat Customization Parts

### BoatPart System

Each boat has `BoatCustomParts` with a list of `BoatPart` entries.

| Class | Purpose |
|-------|---------|
| `BoatPart` | Logical part slot (e.g., "bow", "stern cabin") |
| `BoatPartOption` | Specific option for a part (e.g., "small cabin", "large cabin") |
| `BoatPartOptionChild` | Sub-option linked to parent option |
| `BoatPartsOrder` | Complete order of selected options |

### BoatPartOption Properties

| Property | Description |
|----------|-------------|
| `optionName` | Display name |
| `requires[]` | Other options that must be active |
| `requiresDisabled[]` | Other options that must be disabled |
| `childMast` | Child mast (if this part is a mast) |
| `walkColObject` | Walk collider for this configuration |
| `canInstall` | Whether this option can currently be installed |

### SaveSailData

Each sail on a boat is serialized as:

| Field | Type | Description |
|-------|------|-------------|
| `prefabIndex` | int | Sail prefab index (from PrefabsDirectory.sails[]) |
| `mastIndex` | int | Which mast it's on (orderIndex) |
| `installHeight` | float | Vertical position on mast |
| `minAngle` | float | Minimum sail angle |
| `maxAngle` | float | Maximum sail angle |
| `health` | float | Sail condition (100 = new) |
| `sailColor` | int | Color index from PrefabsDirectory.sailColors[] |
| `scaleY` | float | Y-axis scale |
| `scaleZ` | float | Z-axis scale |

### BoatCustomKeel

Simple component marking custom keel attachment:
```csharp
class BoatCustomKeel : MonoBehaviour { }
```

---

## Starter Sets

`StarterSet` — attached to scenes, provides initial items per region.

| Field | Type | Description |
|-------|------|-------------|
| `region` | PortRegion | Which region this starter set applies to |
| `starterBoat` | Transform | Initial boat prefab |

**Behavior:**
- On `GameState.justStarted` + matching `GameState.newGameRegion`:
  - Sets `GameState.lastBoat` and `lastOwnedBoat` to `starterBoat`
  - Activates all child transforms (the starter items)
  - Moves them down 15 units, then marks all `sold = true`, registers to save
- On non-matching region: destroys all child items

The actual starter items are children of the StarterSet GameObject in the Unity scene (not visible in decompiled code). They likely include basic navigation tools, some food, and a water bottle.

---

## Ocean/Wave System Items

### Ocean Color Zones
`OceanColorPalette` and `OceanColorZone` handle water color variations by region (tropical blue, northern dark, etc.).

### OceanPresets
Located in `/Sailwind_Data/StreamingAssets/OceanPresets/` — configuration files for wave parameters.

### Seagulls
`Seagulls` component — ambient wildlife, follows boats.

### Rain/Rainbow
`Rain` and `Rainbow` — weather visual effects.

### Stars
`Stars` — night sky rendering with navigation stars.

---

## Additional Unclassified Items

From code analysis, these exist as classes but their prefab indices weren't confirmed:

| Class | Likely Purpose |
|-------|----------------|
| `KiciaAltar` | Shrine/altar interaction |
| `ShroomTrigger` | Mushroom-related trigger |
| `OnsenExitTrigger` / `OnsenMusicTrigger` | Hot spring area |
| `ShipItemElixir` (index 96-98) | Elixirs (energy/sleep/random) |
| `Balloon` | Possibly a decorative balloon |
| `Tavern` | Tavern interaction point |
| `TavernRumorsDude` | NPC with rumors |

---

*Extracted from Sailwind v0.38 decompilation.*
