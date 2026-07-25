# Система префабов и SaveablePrefab

## PrefabsDirectory
- Синглтон: `PrefabsDirectory.instance`
- `directory[]` — массив GameObject префабов
- Доступ по `prefabIndex`

## SaveablePrefab
- Хранит:
  - `prefabIndex`
  - `instanceId`
  - `currentCrateId`
  - `parentObject` (boat index или -1/-2/-3)
- Методы:
  - `PrepareSaveData()` → `SavePrefabData`
  - `Load(SavePrefabData)`
  - `RegisterToSave()`
  - `Unregister()`

## SavePrefabData
- Сериализуемые данные предмета при сохранении

## BoatLocalItems
- Отвечает за кэширование предметов на лодке
- `cachedItems` — `List<SavePrefabData>`
- `SpawnCachedItems()` — инстанцирует предметы при приближении игрока
- `CacheItemsOnOutOfRange()` — сохраняет предметы при отдалении

## Ключевые индексы parentObject
- `-1` — в мире / на лодке
- `-2` — уничтожить
- `-3` — затонул (recovery)

## Ссылки
- `BoatLocalItems.cs`
- `SaveablePrefab.cs`
- `ShipItem.cs` (GetPrefabIndex)