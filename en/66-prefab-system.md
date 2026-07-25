# Prefab system and `SaveablePrefab`

## `PrefabsDirectory`

- Singleton: `PrefabsDirectory.instance`
- `directory[]` — array of prefab GameObjects
- Resolved through `prefabIndex`

## `SaveablePrefab`

Stores:

- `prefabIndex`
- `instanceId`
- `currentCrateId`
- `parentObject` (boat index, or `-1` / `-2` / `-3`)

Methods:

- `PrepareSaveData()` → `SavePrefabData`
- `Load(SavePrefabData)`
- `RegisterToSave()`
- `Unregister()`

## `SavePrefabData`

Serialized item data used for saving.

## `BoatLocalItems`

Caches items belonging to a boat.

- `cachedItems` — `List<SavePrefabData>`
- `SpawnCachedItems()` — instantiates cached items when the player approaches
- `CacheItemsOnOutOfRange()` — saves items when the player moves away

## Important `parentObject` values

- `-1` — world / boat context
- `-2` — destroy
- `-3` — lost during recovery

## References

- `BoatLocalItems.cs`
- `SaveablePrefab.cs`
- `ShipItem.cs` (`GetPrefabIndex`)
