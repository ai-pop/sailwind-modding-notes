# 62. Масса ящика с лутом, материал предмета

Разбор массы ящика с содержимым и поля материала — ответ на запросы B4, B5. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 16 (ShipItem), 61 (каталог/массы).

## B4. Масса ящика с лутом (ящик «наполненный лутом»)

### `ShipItemCrate` — поля массы

| Поле | Тип | Содержание |
|------|-----|------------|
| `containedPrefab` | `GameObject` [SerializeField] | Префаб содержимого (еда/товар) |
| `amount` | `float` (на ShipItem) | Количество единиц в ящике |
| `smokedFood` | `bool` | Пища копчёная ( влияет на value и FoodState) |

### `ItemRigidbody.UpdateMass()` — вербатим (crate-часть)

```csharp
public void UpdateMass()
{
    rigidbody.mass = item.mass;  // base crate mass (prefab)
    ShipItemCrate component = item.GetComponent<ShipItemCrate>();
    if (component != null)
    {
        rigidbody.mass += component.GetContainedPrefab().GetComponent<ShipItem>().mass * component.amount;
    }
    // ... (bottle/tea/salt/soup modifiers)
}
```

**Формула:** `twinRigidbody.mass = crate.mass + containedPrefab.ShipItem.mass × amount`

**Учитывает ли содержимое CrateInventory в массе?** — **ДА**, но через `amount` (float), **не** через `CrateInventory.containedItems`.

> **КРИТИЧНО:** масса ящика = `item.mass` (ящик как контейнер) + `containedPrefab.mass × amount` (содержимое × количество). Это **sealed crate mass** — масса запечатанного ящика с N единицами содержимого.

**Но:** когда ящик **распечатан** (`UnsealCrate`) → `amount` уменьшается на 1 за каждый извлечённый предмет → `UpdateMass()` вызывается → масса уменьшается. После full unseal → `amount = 0` → `mass = item.mass` (пустой ящик).

### `ShipItemCrate.UnsealCrate()` — вербатим

```csharp
public void UnsealCrate()
{
    int num = (int)amount;
    for (int i = 0; i < num; i++)
    {
        GameObject val = Object.Instantiate<GameObject>(containedPrefab, transform.position + new Vector3(0, 100.5f, 0), transform.rotation);
        amount -= 1f;
        val.GetComponent<SaveablePrefab>().RegisterToSave();
        // ... food state handling
        StartCoroutine(InsertItem(val.GetComponent<ShipItem>()));
    }
    UpdateLookText();
    itemRigidbodyC.UpdateMass();  // ← МАССА ОБНОВЛЯЕТСЯ после unseal!
    // ...
}
```

**После unseal:** `amount` уменьшена → `UpdateMass()` вызывается → twin mass recalculated → **масса пустого ящика = item.mass (base only, без содержимого)**.

### `CrateInventory.containedItems` vs `ShipItemCrate.amount`

`CrateInventory.containedItems` — `List<ShipItem>` — предметы, **вставленные в открытый ящик** (manual insertion after unseal). Это **динамический** список.

`ShipItemCrate.amount` — float — **sealed count** (original запечатанное количество).

**Масса ящика учитывает `amount` (sealed), НЕ `containedItems.Count` (opened).** Это значит: если ящик запечатан (amount=10) → mass = crate.mass + content.mass×10. Если ящик открыт и игрок вручную положил 5 предметов в containedItems → **UpdateMass НЕ учитывает эти 5 предметов** в mass twin! Масса twin = crate.mass + 0 (amount=0 после unseal).

> **Bug/feature:** `UpdateMass()` не учитывает `CrateInventory.containedItems` для массы twin. Открытый ящик с 5 предметами внутри → twin mass = crate.mass (base), а не crate.mass + 5×content.mass. **Мод, который считает честную Архимеду, должен сам учесть содержимое CrateInventory.containedItems.**

### `CrateInventory.InsertItem()` / `WithdrawItem()` — влияние на массу

```csharp
public void InsertItem(ShipItem item)
{
    if (!containedItems.Contains(item)) { containedItems.Add(item); }
    // ... attached=true, disableCol=true, inStove=true, ForceRigidbodyToWalkCol, layer=26
}

public void WithdrawItem(ShipItem item)
{
    containedItems.Remove(item);
    // ... attached=false, disableCol=false, inStove=false, layer=2
}
```

**InsertItem/WithdrawItem НЕ вызывают `UpdateMass()`** — масса twin не обновляется при вставке/извлечении предметов из открытого ящика. Только `UnsealCrate()` и `SpawnContainedPrefab()` вызывают `UpdateMass()`.

### Как получить суммарный вес содержимого открытого ящика

```python
# Pseudocode для мода
total_content_mass = sum(item.mass for item in crate_inventory.containedItems)
# + sub-modifiers (bottle.health, tea.amount*0.1, etc.) per item
total_crate_mass = crate.ShipItem.mass + total_content_mass
```

> **Мод должен считать массу содержимого `CrateInventory.containedItems` самостоятельно** — ванильный `UpdateMass()` не учитывает открытый ящик.

## B5. Материал предмета

### Есть ли на ShipItem поле материала/типа?

**Ответ: НЕТ — нет explicit material/material-type field на ShipItem.**

`ShipItem` fields (из декомпиляции):

```csharp
public bool wallAttachment;
public bool delayLook;
public float mass = 1f;
public int value;
public string name;
public TransactionCategory category;  // ← категория, НЕ материал
public float inventoryScale = 1f;
public float inventoryRotation;
public float inventoryRotationX;
public float floaterHeight = 1.6f;
public bool sold;
public bool nailed;
public float health;
public float amount;
```

**`TransactionCategory category`** — enum для экономической категории (bulkAlco, bulkFood, bulkWater, bulkGood), **не материал**.

### Implicit material indicators

| Класс | Implicit material | Evidence |
|-------|-------------------|----------|
| `ShipItemBottle` | Стекло/металл | BottleDrinking, health = fill level, collision sound = glass |
| `ShipItemCrate` | Дерево | CollisionSound = wood (ItemCollisionSoundPlayer), nailed |
| `ShipItemHammer` | Металл+дерево | CanNail, tool |
| `ShipItemBed` | Дерево+ткань | nailed, bed rest |
| `ShipItemSoup` | Металл (kettle) | Stove cooking, water fill |
| `ShipItemSalt` | Дерево (barrel) | amount, bulkGood |
| `ShipItemHangable` | Металл (lamp) | Hook, ConfigurableJoint |
| `ItemCollisionSoundPlayer` | Дерево (default) | `PlayWoodColSound` — default collision sound for ALL items |

> **Все предметы используют `PlayWoodColSound` по умолчанию** — ваниль **не различает материал по звуку**. Все падают с wood sound. **Нет material enum или field** — материал — implicit (visual/model только).

### Что влияет на «разрушаемость»/«плавание»

- `floaterHeight` (default 1.6) — buoyancy target. Все предметы с floaterHeight > 0 → float. **Нет «тонущих» предметов в ваниле** — все float с `_raiseObject` > 0.
- `nailed` — предмет frozen (kinematic), не двигается. **Не разрушаемость** — «прибитость».
- `health` — BottleDrinking fill level (0-1), **не прочность**. У ShipItemFood — food consumed (EatFood reduces health).
- `amount` — Tea/Salt/Crate count, **не прочность**.
- **Нет `durability` / `breakable` / `sinkable` field** — предметы **не разрушаются** и **не тонут** в ваниле (SimpleFloatingObject всегда выталкивает).

> **Ваниль не имеет понятия «тонущий предмет»** — `_raiseObject > 0` → все плавают. Мод с честным Архимедом должен добавить sinking logic (mass > displacement → sinks).

## Практические выводы для мододела

1. **Crate mass при sealed:** `mass = crate.mass + containedPrefab.mass × amount` — учитывает содержимое через `amount`.
2. **Crate mass при opened:** `UpdateMass()` НЕ учитывает `CrateInventory.containedItems` → **мод должен считать сам** для честной плавучести.
3. **Нет material field** — TransactionCategory (категория экономики), не материал. Все предметы = wood collision sound. Material = implicit (visual/model).
4. **Нет «тонущих» предметов в ваниле** — `_raiseObject > 0` → все плавают. Мод должен добавить sinking (mass > displacement → sink).
5. **floaterHeight default = 1.6** — buoyancy target. Мод с честным Архимедом может использовать mass/displacement ratio вместо fixed `_raiseObject`.
