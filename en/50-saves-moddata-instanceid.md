# 50. Saves: GameState.modData and SaveablePrefab.instanceId

Analysis of modData save mechanism, instanceId stability, write triggers. Decompilation of `Assembly-CSharp.dll` (Sailwind v0.38). Related to note 11.

## D15. GameState.modData

### When the Dictionary Actually Gets into the Save

```csharp
// GameState.cs
public static Dictionary<string, string> modData = new Dictionary<string, string>();

// SaveContainer.cs
public Dictionary<string, string> modData = new Dictionary<string, string>();

// SaveLoadManager.cs — on save:
saveContainer.modData = GameState.modData;  // direct reference assignment!

// SaveLoadManager.cs — on load:
if (saveContainer.modData != null) {
    GameState.modData = saveContainer.modData;  // direct reference assignment!
}
```

**Write trigger:** `GameState.modData` goes into save **on every explicit save** (autosave, manual save). **No separate trigger** — dictionary serialized as-is.

**Important:** on save — `saveContainer.modData = GameState.modData` — this is **reference assignment**, not a copy! Mod can modify `GameState.modData` at any time, and on next save — changes go into the file.

On load: `GameState.modData = saveContainer.modData` — reference assignment from deserialized container. Mod gets **ready dictionary** immediately after load.

**SaveLoadManager methods (context):**
- `SaveModData()` — called **before** `saveContainer.modData = GameState.modData` — but this is an abstract method (for mods? or placeholder)
- `LoadModData()` — called **after** `GameState.modData = saveContainer.modData` — also abstract

### Size Limits

`GameState.modData` serialized via `BinaryFormatter` (as part of `SaveContainer`). BinaryFormatter:
- **No explicit limit** on Dictionary<string, string> size in BinaryFormatter
- **Practical limit:** save file size (usually several MB for entire game)
- **Caution:** large strings (JSON-serialized whole structures) — safe, but increase save size
- **Recommendation:** keys — short (mod-prefix + id), values — reasonable size (< 1KB per entry). Don't store binary data as Base64 strings of megabyte size.

### Should You Worry About Large Strings?

**No technical restrictions**, but:
- `BinaryFormatter` serializes `Dictionary<string, string>` efficiently (key + value as strings)
- **Risk:** if mod creates thousands of entries → save grows → load slows
- **Recommendation:** use `modData` for configuration and light data, not logs or history

### Mod Identifiers (Best Practice)

```csharp
// Recommended key pattern:
string key = "MyMod_" + itemId;     // mod prefix + unique id
GameState.modData[key] = value;      // write (automatically on next save)

// Read:
if (GameState.modData.ContainsKey(key)) {
    string value = GameState.modData[key];
}

// Initialize when absent:
if (!GameState.modData.ContainsKey("MyMod_spawnTimer")) {
    GameState.modData["MyMod_spawnTimer"] = "30";
}
```

**Important:** mod should read `modData` **after load** (GameState.currentlyLoading = false, GameState.playing = true). At load time, `GameState.modData` is populated from save.

## SaveablePrefab.instanceId — Stability

### How Id Is Assigned

```csharp
private void AssignRandomInstanceId() {
    int num = Random.Range(1, int.MaxValue);
    while (existingInstanceIds.Contains(num)) {
        num = Random.Range(1, int.MaxValue);
    }
    instanceId = num;
}
```

- **Random.Range(1, int.MaxValue)** — random id on **first RegisterToSave**
- `existingInstanceIds` — static List<int>, uniqueness check
- If `instanceId == 0` (not assigned) at `RegisterToSave` → `AssignRandomInstanceId()`

### Id Stability

**After first save:** `instanceId` serialized in `SavePrefabData.instanceId` and **persisted**. On load: `instanceId = data.instanceId` (from save).

| Event | Is id stable? |
|---------|-----------------|
| Save reload | **Yes** — id from save |
| Purchase (Sell → RegisterToSave) | **Yes** — id assigned on first RegisterToSave |
| Move (to crate/inventory/boat) | **Yes** — id doesn't change |
| New Instantiate | **New id** — each Instantiate → new ShipItem → new instanceId on RegisterToSave |
| Destroy + recreate | **New id** — old id removed from existingInstanceIds on Unregister |

**Conclusion:** `instanceId` **stable** for one physical item (one GameObject) across saves. Doesn't change on move, crates, boats. **But:** on destroy and recreation — new id.

### Mod Use of instanceId

```csharp
// Get item's instanceId
int id = shipItem.GetComponent<SaveablePrefab>().instanceId;

// Save in modData (stable across saves)
GameState.modData["MyMod_item_" + id] = "state";

// Check: is item registered for saving?
if (shipItem.GetComponent<SaveablePrefab>() != null) {
    // instanceId assigned at RegisterToSave
}
```

**Risk:** if item destroyed (`DestroyItem`) — its instanceId disappears from `existingInstanceIds`. Reference to id in `modData` becomes **dead**. Mod should clean dead entries on load, or use `parentObject` as fallback (SaveablePrefab.GetParentObject() — parent object's scene index).

### currentCrateId — for Nested Items

```csharp
public int currentCrateId;  // id of crate containing item
```

- When item placed in crate: `currentCrateId` = crate's instanceId
- Serialized together with item
- On load: `LoadAfterDelay` finds crate by id and places item back
