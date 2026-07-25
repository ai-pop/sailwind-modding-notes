# 64. Физическая модель предметов

> Сухие факты о всех компонентах физики предметов: плавучесть, инерция, столкновения, масса.
> На основе декомпиляции Assembly-CSharp.dll (Sailwind v0.38).

---

## Архитектура: двойной GameObject

Каждый предмет существует как **два** GameObject'а одновременно:

```
ShipItem (визуал)                       ItemRigidbody (физика)
├─ Collider: isTrigger = true           ├─ Rigidbody
├─ Renderer + LODGroup                   ├─ SimpleFloatingObject (Crest)
├─ Сохраняет позицию/поворот             ├─ BoxCollider / MeshCollider / CapsuleCollider
└─ Слой: динамический (2/5/26)           ├─ Субколлайдеры (ItemSubcollider)
                                         └─ Слой: 2 (IgnoreRaycast)
```

### Поток позиций

| Состояние | Кто за кем следует |
|-----------|---------------------|
| Предмет держат (`held != null`) | `itemRigidbody` → позиция `ShipItem` |
| Предмет на полке (`sold == false`) | `itemRigidbody` → позиция `ShipItem` |
| Предмет свободен (`sold && !held`) | `ShipItem` → позиция `itemRigidbody` |
| На лодке, не держат | `ShipItem` → позиция `itemRigidbody` через walkCol |
| В инвентаре | Оба → позиция слота инвентаря |

### Создание в `ItemRigidbody.Start()`

```csharp
rigidbody.drag = 1.2f;
rigidbody.angularDrag = item.mass * 0.1f;
rigidbody.isKinematic = true;
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;  // по умолчанию 1.6
```

Коллайдеры копируются с `ShipItem` через `AddCollider()` с задержкой 3 FixedUpdate. Если есть `MeshCollider` — `collisionDetectionMode = 2` (Continuous), иначе `3` (ContinuousDynamic).

---

## SimpleFloatingObject (Crest Ocean System)

**Код этого компонента отсутствует в декомпиляции** — он находится в скомпилированном Assembly-CSharp.dll как часть интегрированного ассета Crest.

### Известные параметры (задаются в ItemRigidbody.Start)

| Параметр | Значение | Источник |
|----------|:-------:|----------|
| `_raiseObject` | `item.floaterHeight` (≈1.6) | `ShipItem.floaterHeight` |
| `_dragInWaterRotational` | `0.02` | Хардкод |

### Предполагаемое поведение

На основе наблюдаемых эффектов:
1. Проверяет высоту воды через `Ocean` API Crest
2. Если точка ниже воды — применяет силу вверх
3. `_raiseObject` добавляется к целевой высоте над водой
4. При `_raiseObject = 1.6` объект центрируется на ~1.6 единиц выше уровня воды

---

## Альтернативные системы плавучести

### Buoyancy (используется лодками и некоторыми крупными объектами)

Семплинг по сетке точек («блобы»), расчёт силы Архимеда на каждую точку.

| Поле | По умолчанию | Описание |
|------|:-----------:|----------|
| `SlicesX` | 2 | Точек замера по X |
| `SlicesZ` | 2 | Точек замера по Z (всего 2×2=4) |
| `magnitude` | 2.0 | Множитель выталкивающей силы |
| `dampCoeff` | 0.1 | Демпфирование вертикальной скорости |
| `CenterOfMassOffset` | −1.0 | Смещение центра масс вниз |
| `interpolation` | 3 | Сглаживание по кадрам |
| `ChoppynessAffectsPosition` | false | Течения смещают объект |
| `WindAffectsPosition` | false | Ветер двигает объект |
| `xAngleAddsSliding` | false | Наклон создаёт скольжение |
| `sink` | false | Режим затопления с доп. силой |
| `moreAccurate` | false | Билинейная интерполяция высоты воды |

**Логика на точку (блоб):**
```
waterHeight = ocean.GetWaterHeightAtLocation2(x - choppy, z)
delta = magnitude × (blobWorldY − waterHeight)
// delta > 0: точка НАД водой → сила ВНИЗ
// delta < 0: точка ПОД водой → сила ВВЕРХ
verticalVel = rigidbody.GetPointVelocity(blobWorldPos).y
force = −Vector3.up × (delta + dampCoeff × verticalVel)
rigidbody.AddForceAtPosition(force, blobWorldPos)
```

При `moreAccurate = true`: используется `GetWaterHeightAtLocation2` (билинейная). Иначе: `GetWaterHeightAtLocation` (ближайший сосед) + lerp-сглаживание 0.5.

**Скольжение при наклоне:** если `xAngleAddsSliding`:
- Наклон 5°–90° → ускорение вперёд
- Наклон 270°–355° → ускорение назад
- Максимум ±20 единиц силы, затухание 0.05/такт

### Boyant (простейшая)

Прямая установка позиции Y без физики:
```
height = ocean.GetWaterHeightAtLocation2(x - choppy, z) + buoyancy
transform.position.y = height
```

---

## Условия isKinematic (ItemRigidbody.FixedUpdate)

Rigidbody предмета становится кинематическим (исключается из физической симуляции) при ЛЮБОМ из:

| # | Условие | Код |
|:--|---------|-----|
| 1 | Предмет держат | `item.held != null` |
| 2 | Не куплен (на полке магазина) | `!item.sold` |
| 3 | Прибит гвоздями | `item.nailed` |
| 4 | Прикреплён к стене | `attached` |
| 5 | Сон / восстановление / верфь | `GameState.sleeping \|\| recovering \|\| currentShipyard != null` |
| 6 | В ящике или инвентаре | `currentBox != null \|\| currentInventorySlot != null` |
| 7 | Вне зоны (600м) | `outOfRange` |
| 8 | Первые 6 FixedUpdate | `fixedFramesSinceSpawn < 6` |
| 9 | Отладка | `Debugger.kinematicItemsTimer > 0 \|\| debugForceKinematicBoat` |
| 10 | Принудительно | `debugForceKinematic` |
| 11 | MeshCollider «спит» | `meshCol != null && sleeping && dynamicColTimer <= 0` |

**Следствие:** предметы на лодке, которые не держат — кинематические. Их позиция задаётся через parent-трансформации (`MoveItemToWalkColRigidbody`), а не через физику.

---

## Управление коллайдерами

### dynamicColTimer

После того как предмет перестали держать — `collisionDetectionMode = 2` (Continuous) на 6 секунд. После — `3` (ContinuousDynamic), если нет MeshCollider.

### isTrigger

Коллайдеры становятся триггерами когда предмет держат, в ящике, в инвентаре, или прикреплён.

### outOfRange

Если расстояние до камеры > 600м и предмет не на лодке:
- Считает кадры (до 10)
- Уничтожает предмет (`DestroyItem()`)
- Проверка каждые 5–8 секунд

---

## Система масс

### Базовые правила

- `ShipItem.mass` (кг) — задаётся в префабе
- `Rigidbody.mass` обновляется через `UpdateMass()`
- `angularDrag = mass × 0.1`

### Модификаторы

| Подкласс | Добавка к массе |
|----------|-----------------|
| `ShipItemCrate` | `containedPrefab.mass × amount` |
| `ShipItemBottle` | `item.health` (fill level = масса жидкости) |
| `ShipItemTea` | `amount × 0.1` |
| `ShipItemSalt` | `amount × 0.1` |
| `ShipItemSoup` | `currentWater + currentEnergy/20 + currentUncookedEnergy/20` |

### BoatMass.UpdateMass()

```csharp
totalMass = selfMass + partsMass
for each item on boat where inventorySlot == null:
  if item.held && player on this boat:
    totalMass += item.mass
    comShift += playerLocalPos × (item.mass / selfMass) × leverageMult
  else:
    totalMass += item.mass
    comShift += itemLocalPos × (item.mass / selfMass) × leverageMult
totalMass += 160  // игрок
comShift += playerLocalPos × (160 / selfMass) × leverageMult
for each mast sail:
  totalMass += GetSailMass(sail)
body.mass = totalMass
body.centerOfMass = keel.centerOfMass + comShift
```

### GetSailMass

```
junk/gaff: sailPower × 20 + sailPower × 20  (= sailPower × 40)
staysail: sailPower × 20 + 0                 (= sailPower × 20)
остальные: sailPower × 20 + sailPower × 10  (= sailPower × 30)
```

---

## Декализия (PickupableItemCollisionChecker)

Компонент на `itemRigidbody`.

| Константа | Значение |
|-----------|:-------:|
| Множитель выталкивания | 1.8 |
| Порог разрешённого дропа | 0.06 (глубина проникновения) |

**Алгоритм:**
```
decollisionVector = Σ(normal × penetration × 1.8) для всех коллизий
возврат в локальных координатах itemRigidbody
```

При удержании: если `penetration >= 0.06` → красный контур, дроп запрещён (для big items или при зажатом Throw).

---

## RigidbodyDirectionalDrag

Покадровое масштабирование скорости в локальных осях:

```csharp
localVel = transform.InverseTransformDirection(rb.velocity)
localVel.x *= x
localVel.y *= y
localVel.z *= (localVel.z > 0) ? zForward : zBack
rb.velocity = transform.TransformDirection(localVel)
```

Значения `x, y, zForward, zBack` задаются в инспекторе, должны быть ≤ 1.0.

---

## HullDrag (сопротивление корпуса лодки)

```csharp
addedDrag = boat._dragInWaterForward
addedSideDrag = boat._dragInWaterRight
speed = rb.velocity.magnitude
boat.addedHullDrag = speed² × addedDrag + boat._dragInWaterForward × 2
boat.addedSideDrag = speed × addedSideDrag + boat._dragInWaterRight × baseSideDragMult
```

---

## ShiftingRigidbody (FloatingOrigin)

При смещении FloatingOrigin:
1. `PrepareForShifting()`: сохраняет velocity/angularVelocity, устанавливает isKinematic
2. `RestoreMomentum()`: восстанавливает через корутину с задержкой кадров

---

## Сводка констант

| Константа | Значение | Источник |
|-----------|:-------:|----------|
| drag (линейный) | 1.2 | `ItemRigidbody.Start` |
| angularDrag | `mass × 0.1` | `ItemRigidbody.Start` |
| `_dragInWaterRotational` | 0.02 | `ItemRigidbody.Start` |
| `_raiseObject` | `floaterHeight` (≈1.6) | `ItemRigidbody.Start` |
| `floaterHeight` default | 1.6 | `ShipItem` |
| Buoyancy.magnitude default | 2.0 | `Buoyancy` |
| Buoyancy.dampCoeff default | 0.1 | `Buoyancy` |
| Decollision multiplier | 1.8 | `PickupableItemCollisionChecker` |
| Decollision threshold | 0.06 | `PickupableItemCollisionChecker` |
| Out-of-range distance | 600 м | `ItemRigidbody.FixedUpdate` |
| dynamicColTimer | 6 сек | `ItemRigidbody.SetDynamicColTimer` |
| Масса игрока в BoatMass | 160 | `BoatMass.UpdateMass` |
| Гравитация | Стандартная Unity (−9.81) | `Physics.gravity` |
| collisionDetectionMode (mesh) | 2 (Continuous) | `ItemRigidbody.AddCollider` |
| collisionDetectionMode (primitive) | 3 (ContinuousDynamic) | `ItemRigidbody.AddCollider` |

---

*Извлечено из декомпиляции Assembly-CSharp.dll (Sailwind v0.38).*
*SimpleFloatingObject — компонент Crest Ocean System; его исходный код недоступен.*
