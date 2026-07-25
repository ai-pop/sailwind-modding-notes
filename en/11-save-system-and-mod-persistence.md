# 11. Save system and mod persistence hook

Complete breakdown of the game serialization subsystem. From decompilation of `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. This is the most important system for any mod that needs to persist data between sessions.

## Quick: how a mod persists data

The game has a **built-in official hook** for mod data. In `GameState` there's a static dictionary:

```csharp
// GameState.cs
public static Dictionary<string, string> modData = new Dictionary<string, string>();
```

This dictionary **is automatically serialized to the save file and restored on load** — without any code from the mod side. Just write your data there:

```csharp
GameState.modData["MyMod.money"] = "1234";
GameState.modData["MyMod.flags"] = "a,b,c";
```

On the next game save, values go into the save file, and on load they return to the same dictionary. The developer left empty method hooks for this (see below), but the `modData` transfer mechanism already works fully in the release version.

> **Important:** values are only `string`. Serialize complex structures yourself (JSON, base64, `key;key;key`). Keys conflict globally between mods — use your mod name prefix (`MyMod.*`).

## Save architecture

```
SaveLoadManager (MonoBehaviour, singleton .instance)
│
├── currentObjects : SaveableObject[255]   ← "static" scene objects (boats, player, anchors...)
│       registration: SaveableObject.Awake() → AddObject(this) by sceneIndex
├── currentPrefabs : List<SaveablePrefab>  ← dynamic items (ShipItem)
│       registration: SaveablePrefab.RegisterToSave() → AddPrefab(this)
└── SaveGame(compressed) → DoSaveGame() (coroutine)
        collects SaveContainer → BinaryFormatter → slot{N}.save
```

### Entry points

| Field/method | Type | Purpose |
|-------------|------|---------|
| `SaveLoadManager.instance` | `static` | Manager singleton. |
| `SaveLoadManager.readyToSave` | `static bool` | Save gate: `SaveGame` does nothing while `false`. |
| `SaveGame(bool compressed)` | `void` | Start saving (via `DoSaveGame` coroutine). |
| `LoadGame(int backupIndex)` | `void` | Load: `0` = current save, `1..5` = backups. |
| `save` / `saveCompressed` / `load` | `bool` (public) | Trigger flags: set to `true` → fires in `Update()`. |
| `SaveModData()` / `LoadModData()` | `void` | **Empty hooks** — only `Debug.Log("Saving/Loading mod data...")`. Extension points reserved by developer. |

### Conditions when save does NOT happen

`SaveGame` silently refuses (logs `"not ready to save"`), if any:

- `busy` (save/load already in progress);
- player in bed (`GameState.inBed != null`);
- `readyToSave == false`;
- player at shipyard (`GameState.currentShipyard != null`).

### Autosave

In manager `Update()`:

- Interval = `Settings.autosaveInterval` **minutes** (`* 60f` seconds). If `<= 0` — autosave disabled.
- Autosave skipped when: `enableAutosave == false`, player sleeping (`GameState.sleeping`), recovery (`GameState.recovering`), crate UI open (`CrateInventoryUI.instance.showingUI`).
- Autosave always **compressed** (`compressed: true`) — hull dirt textures encoded to PNG. Manual save can be uncompressed (raw textures).
- `enableAutosave` forced on outside Unity editor (`!Application.isEditor`).

## Save files

Paths built by `SaveSlots` from `Application.persistentDataPath` (on Windows: `%USERPROFILE%\AppData\LocalLow\<Company>\<Product>\`):

| File | Purpose |
|------|---------|
| `slot{N}.save` | Current save for slot N. |
| `slot{N}_backup{1..5}.save` | Ring backups: before each save, current save shifts to `backup1`, old `backup1` → `backup2`, ..., `backup5` is deleted. |

- **6 slots** (indices `0..5`). `SaveSlots.currentSlot` — active one.
- `SaveSlots.slotsActive[6]` / `activeSlotsCount` — which slots are occupied (determined by file presence in `Awake`).
- Format — **`BinaryFormatter`** (legacy .NET binary serialization) over `[Serializable]` class `SaveContainer`. Important: format is binary, tied to field names/assembly; versioning via `gameVersion` field (current = `1`).

> **For modders:** you don't need to read/write save files directly, and it's dangerous (BinaryFormatter). Use `GameState.modData`. External tools for save parsing would need the same `SaveContainer` structure.

## SaveContainer — what exactly is saved

`SaveContainer` (`[Serializable]`) — flat object with all world data. Main field groups:

### Player and economy
| Field | Type | Contents |
|-------|------|----------|
| `playerCurrency` | `int[]` | Wallet by currencies (`PlayerGold.currency`). |
| `currentCurrency` | `int` | Active currency. |
| `currencyRates` | `float[]` | Current currency rates (`CurrencyMarket.instance.currentPrices`). |
| `playerReputation` | `int[]` | Reputation (`PlayerReputation.GetSaveData()`). |
| `playerKnownPrices` | `PriceReport[]` | Prices known to player. |
| `tradeReceipts` | `TradeReceipt[7]` | Transaction receipts. |
| `charts` | `List<ChartData>` | Drawn nautical charts. |
| `currencyDayLogs` | `DaySheet[][]` | Income history by days/currencies. |

### Time and world
| Field | Type | Contents |
|-------|------|----------|
| `time` | `float` | Global time (`Sun.sun.globalTime`). |
| `moonPhase` | `float` | Moon phase (`Moon.instance.currentPhase`). |
| `day` | `int` | Game day number (`GameState.day`). |
| `wind` | `SerializableVector3` | Base wind (`Wind.currentBaseWind`). |
| `wavesRotation` / `wavesInertia` / `wavesMagnitude` | `Quaternion`/`float` | Wave state (`WavesInertia`). |
| `lastVisitedPort` | `int` | Last port index (`-1` = none). |

### Needs and habits
`food`, `foodDebt`, `water`, `sleep`, `sleepDebt`, `vitamins`, `protein`, `alcohol` (float) — from `PlayerNeeds`; `tobaccoWhite/Green/Black/Brown` (float) — from `PlayerTobacco`.

### World (ports, markets, missions, NPCs)
| Field | Type | Contents |
|-------|------|----------|
| `marketsSupply` | `float[][]` | Market supplies by ports (`IslandMarket.currentSupply`). |
| `marketsKnownPrices` | `PriceReport[][]` | Known prices by ports. |
| `portDemands` | `int[][]` | Island demands (`port.island.GetDemandSaveData()`). |
| `missionWarehouses` | `int[][]` | Mission warehouses (`IslandMissionOffice.GetData()`). |
| `quests` | `int[]` | Active quests (`Quests.instance.currentQuests`). |
| `itemSpawnerCooldowns` | `float[]` | Item spawner cooldowns. |
| `traderBoatData` | `TraderBoatData[]` | Trader boat state. |
| `npcBoatData` | `NPCBoatData[]` | NPC boat state. |
| `savedMissions` | `SaveMissionData[]` | Active missions (`PlayerMissions.missions`). |
| `loggedMissions` | — | Mission log (`MissionLog`). |

### Objects and items
| Field | Type | Contents |
|-------|------|----------|
| `savedObjects` | `List<SaveObjectData>` | All `SaveableObject` in scene. |
| `savedPrefabs` | `List<SavePrefabData>` | All dynamic items (`SaveablePrefab`). |
| `modData` | `Dictionary<string,string>` | **Mod data** (`GameState.modData`). |
| `gameVersion` | `int` | Format version (= `1`). |

## SaveableObject — "static" scene objects

Boats, player, anchors, moorings etc. — objects with a fixed `sceneIndex`.

- Array of **255** elements in `SaveLoadManager.currentObjects`.
- Registration in `Awake()`: `SaveLoadManager.instance.AddObject(this)`. On index collision — `LogError("scene index conflict")`.
- **`sceneIndex == 0` is the player** (on load, `PlayerControllerMirror.controller` is moved).
- Position saved **relative to floating origin**: `transform.position - FloatingOriginManager.instance.outCurrentOffset`.
- Universal "extra fields" reused for different purposes:

| Field | For boat (BoatDamage) | For anchor (Anchor) |
|-------|----------------------|---------------------|
| `extraSetting` | — | Anchor set (`IsSet()`) |
| `extraValue` | `waterLevel` | Rope length (`GetRopeLength()`) |
| `extraValue1` | `hullDamage` | — |
| `extraValue2` | `oakum` (caulking) | — |
| `extraTexture` | `Texture2D` hull dirt/cleanliness texture (128×128) | — |
| `customization` | `SaveBoatCustomizationData` (boat customization) | — |

- `sinkRotation` — orientation of sunk boat (`BoatDamage.GetSinkRotation()`).
- Dirt texture: uncompressed = `GetRawTextureData()` (exactly `16777216` bytes = 128×128×RGBA32... actually 128×128×4×... see below), compressed = `EncodeToPNG()`. On load, game distinguishes formats **by array length**: `== 16777216` → `LoadRawTextureData`, otherwise → `ImageConversion.LoadImage` (PNG) with resize to 128×128.
- Boat local items (`BoatLocalItems`) saved as separate `SavePrefabData` with parent binding.

## SaveablePrefab — dynamic items (ShipItem)

All pickupable/purchasable items (`[RequireComponent(typeof(ShipItem))]`).

- `prefabIndex` — index in `PrefabsDirectory.directory[]`. On load, item **is instantiated anew** from prefab and populated with data.
- `instanceId` — random unique int (to distinguish instances; stored in static `existingInstanceIds`).
- Position: relative to parent object (`itemParentObject > 0` → `InverseTransformPoint`), otherwise relative to floating origin.
- **`inventorySlot`**: `< 100` → inventory slot (`GPButtonInventorySlot.inventorySlots[slot]`); `>= 100` → cargo carrier (`CargoCarrier.carriers[slot-100]`).
- Universal `extraValue0..4` reused:

| Component | extraValue0 | extraValue1 | extraValue2 | extraValue3 | extraValue4 |
|-----------|-------------|-------------|-------------|-------------|-------------|
| `FoodState` | dried | smoked | salted | spoiled | — |
| `ShipItemSoup` | uncookedEnergy | spoiled | salted | vitamins | protein |
| `ShipItemKettle` | teaAmount | cookedTeaAmount | — | — | — |

  (for soup/kettle: `itemHealth`=water, `itemAmount`=energy/teaType)
- `chartData` — nautical chart data for foldable items (`ShipItemFoldable.allowCharting`).
- Mission binding: `itemMissionIndex` → `PlayerMissions.missions[idx].RegisterGood(...)`, otherwise `RegisterAsMissionless()`.

## PrefabsDirectory — prefab registry

`PrefabsDirectory.instance` holds arrays:

| Array | Contents |
|-------|----------|
| `directory[]` | All item prefabs, index = `prefabIndex`. Index `0` = null. |
| `sails[]` | Sail prefabs. |
| `sailColors[]` | Sail color palette. |
| `shipItems[]` | Cached `ShipItem` for each prefab. |

**Good↔Item mapping** (asymmetric, critical for economy mods):

```csharp
GoodToItemIndex(goodIndex): goodIndex > 30 ? goodIndex + 170 : goodIndex
ItemToGoodIndex(itemIndex): itemIndex > 200 ? itemIndex - 170 : itemIndex
```

So goods `0..30` share indices with items `0..30`, while goods `31+` map to items starting at index `201`.

## FloatingOriginManager — floating origin

Sailwind's world is huge, so it uses **floating origin** (player stays near scene origin, world shifts). Ocean uses Crest.

| Member | Purpose |
|--------|---------|
| `instance` | Singleton. |
| `outCurrentOffset` (`Vector3`) | Current world offset. **All saves store positions minus this offset.** |
| `ShiftingPosToRealPos(pos)` | `pos - outCurrentOffset` (scene → "real" coordinates). |
| `RealPosToShiftingPos(pos)` | `pos + outCurrentOffset` ("real" → scene). |
| `GetGlobeCoords(obj)` | `(pos - outCurrentOffset - globeOffset) / 9000f` — globe coordinates (divider **9000**). |
| `shiftDistance` | Player displacement threshold triggering world shift. |
| `ShiftingThisFrame` / `CurrentShiftVector` | Static flags for current shift frame. |
| `shiftingRigidbodies` | List of `ShiftingRigidbody` to correct during shift. |

> **For modder:** any persistent or network-transmitted coordinates must be in "real" system (minus `outCurrentOffset`), otherwise objects drift after world shift. Globe divider `9000f` links world units to globe/map coordinates.

## Helper serializable types

`SerializableVector3` and `SerializableQuaternion` — `[Serializable]` structs with implicit conversion operators to/from `UnityEngine.Vector3`/`Quaternion`. Used because `BinaryFormatter` can't directly serialize Unity types.

## Practical conclusions for modding

1. **Persistence:** write to `GameState.modData["MyMod.*"]` — saves automatically. Don't touch `BinaryFormatter` and save files.
2. **Timing:** if you need to do something strictly before/after saving — patch `SaveLoadManager.SaveModData()` / `LoadModData()` (they're empty and stable). Alternative: Harmony patch `DoSaveGame`/`LoadGame`.
3. **Coordinates:** always account for `FloatingOriginManager.instance.outCurrentOffset`.
4. **Don't save** while in bed/at shipyard — game blocks saves in these moments; account for this in your logic.
5. **Indices:** `sceneIndex` (0..254, 0=player), `prefabIndex` (item registry), goods/items mapping `>30 → +170`.
