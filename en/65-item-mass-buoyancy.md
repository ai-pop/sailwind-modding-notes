# Item mass and buoyancy

## `ItemRigidbody.UpdateMass()`

Called when the item is created and when its contents change.

**Base formula:**

```csharp
rigidbody.mass = item.mass;
```

### Additional calculations

**`ShipItemCrate`:**

```csharp
mass += containedPrefab.GetComponent<ShipItem>().mass * amount;
```

**`ShipItemBottle`:**

```csharp
mass += item.health;
```

**`ShipItemTea` / `ShipItemSalt`:**

```csharp
mass += item.amount * 0.1f;
```

**`ShipItemSoup`:**

```csharp
mass += currentWater + currentEnergy / 20f + currentUncookedEnergy / 20f;
```

## Buoyancy (`SimpleFloatingObject`)

- `_raiseObject = item.floaterHeight` (default 1.6f)
- `_dragInWaterRotational = 0.02f`
- Created automatically in `ItemRigidbody.Start()`

## Layers and triggers

- On creation: `gameObject.layer = 2`
- In inventory: `layer = 5`
- `ItemRigidbody` colliders:
  - `isTrigger = true` while the item is held (`held`)
  - `isTrigger = false` while it is lying freely

## References

- `ItemRigidbody.cs`
- `ShipItem.cs`
- `Good.cs` (`GetCargoWeight`)
