# Классы предметов

## Базовые классы

### ShipItem
- Наследуется от: `PickupableItem`
- Требует компоненты: `Rigidbody`, `Collider`, `Renderer`
- Основной класс для всех интерактивных предметов

**Поля:**
- `wallAttachment` (bool)
- `mass` (float) = 1f
- `value` (int)
- `name` (string)
- `category` (TransactionCategory)
- `inventoryScale` (float) = 1f
- `inventoryRotation` (float)
- `inventoryRotationX` (float)
- `floaterHeight` (float) = 1.6f
- `sold` (bool)
- `nailed` (bool)
- `health` (float)
- `amount` (float)

### Good
- Требует: `ShipItem`
- Поля:
  - `nativeRegion` (PortRegion)
  - `requiredRepLevel` (int)
  - `sizeDescription` (string)
  - `missionIndex` (int, private)
- Методы:
  - `GetCargoWeight()` — учитывает `ShipItemCrate` и `ShipItemBottle`
  - `RegisterToMission(int, int)`
  - `GetMissionIndex()`

### ItemRigidbody
- Создаёт отдельный Rigidbody + SimpleFloatingObject
- Управляет коллайдерами (Box/Mesh/Capsule + subcolliders)
- Методы:
  - `UpdateMass()` — пересчитывает массу с учётом содержимого ящиков/бутылок
  - `EnterBoat()` / `ExitBoat()`
  - `EnterInventorySlot(Transform)` / `ExitInventorySlot()`

## Специализированные классы

### ShipItemCrate
- `containedPrefab` (GameObject)
- `smokedFood` (bool)
- `amount` — количество предметов внутри
- Методы: `UnsealCrate()`, `OnAltActivate()`

### ShipItemFood
- `eatenMeshes` (Mesh[])
- `energyPerBite` (float)
- `protein`, `vitamins` (float)
- `rawEnergyMult` (float)

### ShipItemBottle
- Использует `health` как объём жидкости
- Масса = `mass + health`

### ShipItemSoup, ShipItemTea, ShipItemSalt
- Учитываются в `ItemRigidbody.UpdateMass()`

## Система сохранения

- `SaveablePrefab` — хранит `prefabIndex`
- `SavePrefabData` — данные для сохранения
- `BoatLocalItems` — кэширует предметы на лодке
- `PrefabsDirectory.instance.directory[prefabIndex]` — доступ к префабу

## Ссылки на другие файлы
- `16-item-framework-shipitem.md`
- `32-inventory-cargo-storage.md`
- `33-item-spawning-pickup.md`