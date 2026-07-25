# Item classes

## Base classes

### ShipItem
- Inherits from: `PickupableItem`
- Requires components: `Rigidbody`, `Collider`, `Renderer`
- Base class for all interactive items

**Fields:**
- `wallAttachment` (`bool`)
- `mass` (`float`) = 1f
- `value` (`int`)
- `name` (`string`)
- `category` (`TransactionCategory`)
- `inventoryScale` (`float`) = 1f
- `inventoryRotation` (`float`)
- `inventoryRotationX` (`float`)
- `floaterHeight` (`float`) = 1.6f
- `sold` (`bool`)
- `nailed` (`bool`)
- `health` (`float`)
- `amount` (`float`)

### Good
- Requires: `ShipItem`
- Fields:
  - `nativeRegion` (`PortRegion`)
  - `requiredRepLevel` (`int`)
  - `sizeDescription` (`string`)
  - `missionIndex` (`int`, private)
- Methods:
  - `GetCargoWeight()` — accounts for `ShipItemCrate` and `ShipItemBottle`
  - `RegisterToMission(int, int)`
  - `GetMissionIndex()`

### ItemRigidbody
- Creates a separate Rigidbody plus `SimpleFloatingObject`
- Manages colliders (Box/Mesh/Capsule plus subcolliders)
- Methods:
  - `UpdateMass()` — recomputes mass including crate/bottle contents
  - `EnterBoat()` / `ExitBoat()`
  - `EnterInventorySlot(Transform)` / `ExitInventorySlot()`

## Specialized classes

### ShipItemCrate
- `containedPrefab` (`GameObject`)
- `smokedFood` (`bool`)
- `amount` — number of contained items
- Methods: `UnsealCrate()`, `OnAltActivate()`

### ShipItemFood
- `eatenMeshes` (`Mesh[]`)
- `energyPerBite` (`float`)
- `protein`, `vitamins` (`float`)
- `rawEnergyMult` (`float`)

### ShipItemBottle
- Uses `health` as liquid volume
- Mass = `mass + health`

### ShipItemSoup, ShipItemTea, ShipItemSalt
- Included in `ItemRigidbody.UpdateMass()`

## Save system

- `SaveablePrefab` stores `prefabIndex`
- `SavePrefabData` holds serialized data
- `BoatLocalItems` caches boat items
- `PrefabsDirectory.instance.directory[prefabIndex]` resolves a prefab

## Related files

- `16-item-framework-shipitem.md`
- `32-inventory-cargo-storage.md`
- `33-item-spawning-pickup.md`
