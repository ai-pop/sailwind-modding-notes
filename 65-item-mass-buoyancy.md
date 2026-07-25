# Масса и плавучесть предметов

## ItemRigidbody.UpdateMass()

Вызывается при создании и при изменении содержимого.

**Базовая формула:**
```csharp
rigidbody.mass = item.mass
```

### Дополнительные расчёты:

**ShipItemCrate:**
```csharp
mass += containedPrefab.GetComponent<ShipItem>().mass * amount
```

**ShipItemBottle:**
```csharp
mass += item.health
```

**ShipItemTea / ShipItemSalt:**
```csharp
mass += item.amount * 0.1f
```

**ShipItemSoup:**
```csharp
mass += currentWater + currentEnergy / 20f + currentUncookedEnergy / 20f
```

## Плавучесть (SimpleFloatingObject)

- `_raiseObject = item.floaterHeight` (по умолчанию 1.6f)
- `_dragInWaterRotational = 0.02f`
- Создаётся автоматически в `ItemRigidbody.Start()`

## Слои и триггеры

- При создании: `gameObject.layer = 2`
- В инвентаре: `layer = 5`
- Коллайдеры на `ItemRigidbody`:
  - `isTrigger = true` когда предмет в руках (`held`)
  - `isTrigger = false` когда лежит

## Ссылки
- `ItemRigidbody.cs`
- `ShipItem.cs`
- `Good.cs` (GetCargoWeight)