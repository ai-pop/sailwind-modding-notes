# 61. Каталог предметов: PrefabsDirectory, таблица масса/флаги, единицы массы

Разбор системы каталога предметов, таблица масс/флагов, кросс-чек единиц массы — ответ на запросы B1, B2, B3. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 16 (ShipItem), 44 (ItemRigidbody), 45 (crate/cargo).

## B1. КАТАЛОГ ПРЕДМЕТОВ: из какого файла достать список

### `PrefabsDirectory` — класс-каталог

`PrefabsDirectory : MonoBehaviour` — singleton (`PrefabsDirectory.instance`) — содержит **полный каталог всех ShipItem-префабов** в игре.

```csharp
public class PrefabsDirectory : MonoBehaviour
{
    public GameObject[] directory;      // ← ПОЛНЫЙ список префабов по индексу
    public GameObject[] sails;
    public Color[] sailColors;
    public ShipItem[] shipItems;        // ← ShipItem[] populated from directory
    public static PrefabsDirectory instance;

    private void PopulateShipItems()
    {
        shipItems = new ShipItem[directory.Length];
        for (int i = 0; i < directory.Length; i++)
        {
            if (directory[i] != null)
            {
                shipItems[i] = directory[i].GetComponent<ShipItem>();
            }
        }
    }

    public ShipItem GetItem(int itemIndex)
    {
        if (itemIndex == 0) return null;
        if (directory[itemIndex] == null) return null;
        return shipItems[itemIndex];
    }
}
```

**Как достать каталог:**
- `PrefabsDirectory.instance.directory` — `GameObject[]` массив всех префабов по `SaveablePrefab.prefabIndex`.
- `PrefabsDirectory.instance.shipItems` — `ShipItem[]` массив, populated в `Start()`.
- Каждый `directory[i]` → `GetComponent<ShipItem>()` → `shipItems[i]`.
- `prefabIndex` = позиция в `directory` array (validated в Start()).

**Файл в sailwind-decompiled:** `PrefabsDirectory.cs` — но **содержимое массивов directory[]/shipItems[] — runtime-only!** Массивы populated из Unity scene prefab references — **не сериализуются в C# код**. Для полного списка предмет/масса — нужен **runtime dump** (BepInEx mod, iterate PrefabsDirectory.instance.shipItems и log name/mass/big/category).

> **Для повторной выгрузки:** BepInEx plugin, `PrefabsDirectory.instance.Start()` → `shipItems` populated → iterate `for i=1..N`: `shipItems[i].name, mass, big, category, floaterHeight, value, nailed, wallAttachment`. Dump в CSV/JSON.

### `SaveablePrefab.prefabIndex` — маппинг предмет → префаб

`prefabIndex` — int, позиция в `PrefabsDirectory.directory`. Используется в save/load для маппинга instance → prefab. `prefabIndex = 0` → null (no item). `prefabIndex = 1..N` → конкретный ShipItem.

### Декомпиляция не содержит массив префабов

**Полный каталог предметов — НЕ доступен из декомпиляции.** PrefabsDirectory.directory — `GameObject[]` populated из Unity Inspector (scene/prefab references). ILSpy не может выгрузить Unity scene data. **Нужен runtime dump.**

## B2. Таблица предмет/масса (из декомпиляции известных подклассов)

Из декомпиляции доступны **подклассы ShipItem** с известными mass-значениями (в C# code, не prefab defaults). Это **подмножество** полного каталога. Полная таблица требует runtime dump (см. B1).

### Известные ShipItem-подклассы и их массы

| Класс | `mass` дефолт (если указан в коде) | `big` | wallAttachment | nailed (default) | Category | Особенности массы |
|-------|:--:|:--:|:--:|:--:|:--:|---|
| `ShipItem` (base) | 1.0 (SerializeField default) | false | false | false | — | base mass, overridden per prefab |
| `ShipItemCrate` | prefab-dependent (amount × containedPrefab.mass + crate mass) | true | false | false | bulkFood/bulkAlco/bulkWater/bulkGood | `UpdateMass()`: `rigidbody.mass = item.mass + containedPrefab.mass * amount` |
| `ShipItemBottle` | prefab-dependent | true | false | false | — | `UpdateMass()`: `rigidbody.mass += item.health` (water fill level) |
| `ShipItemTea` | prefab-dependent | false | false | false | — | `UpdateMass()`: `rigidbody.mass += item.amount * 0.1` |
| `ShipItemSalt` | prefab-dependent | false | false | false | — | `UpdateMass()`: `rigidbody.mass += item.amount * 0.1` |
| `ShipItemSoup` | prefab-dependent | false | false | false | — | `UpdateMass()`: `rigidbody.mass += currentWater + currentEnergy/20 + currentUncookedEnergy/20` |
| `ShipItemHammer` | prefab-dependent | false | false | false | tool | CanNail: big/wallAttachment/hangable → nailable |
| `ShipItemHangable` (лампы) | prefab-dependent | false | true? | false | — | HangableItem, ConnectJoint |
| `ShipItemLampHook` | prefab-dependent | false | false | false | — | Hook for lamps, occupied bool |
| `ShipItemBed` | prefab-dependent | true | false | false | — | CanNail: nailed in LookUI |
| `ShipItemFood` | prefab-dependent | false | false | false | food | MouthCol eats: held+sold+slot==-1 → 2.55s → EatFood() |
| `ShipItemChipLog` | prefab-dependent | false | false | false | — | Chip log + bobber (SimpleFloatingObject on bobber) |

### Ключевые массы из `ItemRigidbody.UpdateMass()` — вербатим

```csharp
public void UpdateMass()
{
    rigidbody.mass = item.mass;  // base mass from prefab
    ShipItemCrate component = item.GetComponent<ShipItemCrate>();
    if (component != null)
    {
        rigidbody.mass += component.GetContainedPrefab().GetComponent<ShipItem>().mass * component.amount;
    }
    if (item.GetComponent<ShipItemBottle>() != null)
    {
        rigidbody.mass += item.health;  // water fill level (0..1?)
    }
    if (item.GetComponent<ShipItemTea>() != null)
    {
        rigidbody.mass += item.amount * 0.1f;
    }
    if (item.GetComponent<ShipItemSalt>() != null)
    {
        rigidbody.mass += item.amount * 0.1f;
    }
    ShipItemSoup component2 = item.GetComponent<ShipItemSoup>();
    if (component2 != null)
    {
        rigidbody.mass += component2.currentWater + component2.currentEnergy / 20f + component2.currentUncookedEnergy / 20f;
    }
}
```

### Twinless предметы

**У КАКИХ предметов twin НЕТ?** — **У ВСЕХ ShipItem есть twin** (создается в `ShipItem.CreateRigidbody()` → `new GameObject()` + `ItemRigidbody`). Нет «twinless» ShipItem в ваниле. Twin создаётся для **всех** предметов, даже еда-поштучно.

**Но:** twin может быть **disabled** (disableCol=true, ToggleCollider(false)):
- HangableItem на крюке → disableCol=true → twin colliders disabled
- CrateInventory.InsertItem → disableCol=true → twin colliders disabled
- InventorySlot → ToggleCollider(false) → twin colliders disabled

> **Все предметы имеют ItemRigidbody twin GO.** «Twinless» — это только состояние twin (disabled colliders + kinematic), не отсутствие twin.

### ShipItem.floaterHeight — buoyancy raise

```csharp
public float floaterHeight = 1.6f;  // SerializeField default
```

**Default `_raiseObject` для SimpleFloatingObject = 1.6 м.** Это высота, на которую предмет «поднимается» над водой (buoyancy target). Prefab может override это значение.

## B3. Единицы массы

### `ShipItem.mass` — это «кг»?

**Кросс-чек из BoatMass:**

```csharp
// BoatMass.UpdateMass() — масса лодки
float totalMass = baseMass;  // hull base mass
foreach (ItemRigidbody itemBody in itemsOnBoat)
{
    totalMass += itemBody.rigidbody.mass;
}
totalMass += playerMass;  // ~80 кг (CharacterController?)
// лодка dhow: baseMass ≈ 300-500? (prefab runtime)
```

**Масштаб:** если `ShipItem.mass` = кг → типичный ящик (mass=5-10 кг) → лодка с 10 ящиками = +50-100 кг → total 300-500 кг — **реалистично для dhow** (маленькая лодка ~300 кг empty + crew).

**Лёгкий предмет:** еда (mass ≈ 1 кг) — реалистично.
**Тяжёлый предмет:** большой ящик/бочка (mass ≈ 10-20 кг) — реалистично.

> **Вердикт:** `ShipItem.mass` ≈ **килограммы**. 1 Unity unit = 1 метр, `Rigidbody.mass` = кг (Unity convention). Кросс-чек с лодкой и предметами — **масштаб реалистичен**.

### Примерные массы (из контекста кода и логики)

| Тип предмета | Примерный mass (кг) | Обоснование |
|-------------|:--:|------------|
| Еда (рыба, хлеб) | 1–2 | base default=1, food lightweight |
| Tea/Salt (amount) | 1 + amount×0.1 | base + fill mass |
| Bottle (empty) | 1–2 | base + health (water fill) |
| Bottle (full) | 2–3 | base + health≈1 |
| Soup (empty) | 1 + water≈0 | base only |
| Soup (full) | 1+2+0.5 | water + energy/20 |
| Small tool (hammer) | 1–2 | lightweight |
| Big crate (empty) | 5–10 | crate shell only |
| Big crate (10 items) | 5 + 10×1 | crate + contents |
| Big barrel | 10–20 | heavy container |

> **Для точной таблицы — нужен runtime dump PrefabsDirectory.** Код содержит только подклассовые mass-модификаторы, не prefab-дефолты.

## Практические выводы для мододела

1. **Каталог предметов — runtime-only:** `PrefabsDirectory.instance.shipItems[]` populated в Start() из Unity scene. Декомпиляция не содержит массива. **Нужен BepInEx runtime dump** для полной таблицы name/mass/big/category.
2. **Все предметы имеют twin** — нет «twinless» ShipItem. Twin может быть disabled (disableCol=true при hook/crate/inventory).
3. **Mass ≈ килограммы** — 1 unit mass = 1 kg, кросс-чек с лодкой (300-500 кг base) + crew (~80 кг) + предметы (1-20 кг) — реалистичный масштаб.
4. **Crate mass = base + containedPrefab.mass × amount** — масса ящика = масса ящика + масса содержимого × количество. Если ящик пуст (amount=0) → mass = base only.
5. **floaterHeight default = 1.6** — высота всплытия предмета над водой. Prefab может override.
