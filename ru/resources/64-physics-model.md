# 64. Полная физическая модель предметов (Item Physics API)

> **Критически важно для моддинга физики.**
> Разбор ВСЕХ компонентов, участвующих в физике предметов: плавучесть, инерция, столкновения, масса, демпфирование.
> На основе декомпиляции Assembly-CSharp.dll (Sailwind v0.38).

---

## 🔴 КОРЕНЬ ПРОБЛЕМЫ С БОЧКАМИ

### Почему бочки «парят» над водой

Виновник — компонент `SimpleFloatingObject` из **Crest Ocean System** (сторонний ассет, не в декомпиляции). Он создаётся в `ItemRigidbody.Start()`:

```csharp
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;   // почти нет вращательного трения!
floater._raiseObject = item.floaterHeight; // ПО УМОЛЧАНИЮ 1.6 !!!
```

**`_raiseObject`** — это высота ПОДЪЁМА предмета над поверхностью воды. При значении 1.6 предмет поднимается на 1.6 Unity-единиц (метров) НАД водой. Для бочки это означает что она не погружается в воду, а стоит НАД ней.

Плюс `floaterHeight` по умолчанию = 1.6 (в `ShipItem`), что УСУГУБЛЯЕТ ситуацию.

### Как это исправить

Значение `_raiseObject` должно быть около `0.2–0.5` для частичного погружения, а не 1.6. Либо нужно использовать более точную модель плавучести (как `Buoyancy` для лодок) вместо `SimpleFloatingObject`.

---

## Архитектура физического тела предмета

Каждый предмет — это **ДВА** GameObject'а:

```
ShipItem (визуал)                    ItemRigidbody (физика)
├─ Collider: isTrigger = true        ├─ Rigidbody (реальная физика)
├─ Renderer + LOD                     ├─ SimpleFloatingObject (Crest)
├─ Сохраняет позицию/поворот          ├─ BoxCollider / MeshCollider / CapsuleCollider
└─ Слой: меняется динамически         ├─ Субколлайдеры (ItemSubcollider)
                                      └─ Слой: IgnoreRaycast (2)
```

**Поток позиций:**
- Когда предмет **держат** — `itemRigidbody` следует за `ShipItem`
- Когда предмет **свободен** — `ShipItem` следует за `itemRigidbody`

---

## Ключевые компоненты физики

### 1. ItemRigidbody — ядро

| Поле | Значение | Описание |
|------|:-------:|----------|
| `rigidbody.drag` | **1.2** | Линейное сопротивление |
| `rigidbody.angularDrag` | `item.mass * 0.1` | Вращательное сопротивление |
| `collisionDetectionMode` | 3 (ContinuousDynamic) или 2 (Continuous) | Для MeshCollider |
| `floater._dragInWaterRotational` | **0.02** | Вращательное трение В ВОДЕ |
| `floater._raiseObject` | `item.floaterHeight` (1.6) | **ВЫСОТА ПОДЪЁМА НАД ВОДОЙ** |
| `sleepThreshold` | стандартный Unity | Когда предмет «засыпает» |

### 2. Условия isKinematic

`ItemRigidbody.FixedUpdate()` определяет, когда Rigidbody становится кинематическим (не подвержен физике):

```csharp
isKinematic = true когда:
  - item.held != null                 // держат в руках
  - item.sold == false                // не куплен (на полке)
  - item.nailed == true               // прибит
  - attached == true                  // прикреплён к стене
  - onBoat && !held                   // на лодке — НЕТ, кинематик управляется иначе
  - GameState.sleeping                // сон
  - GameState.recovering              // восстановление
  - GameState.currentShipyard != null // в верфи
  - currentBox != null                // в контейнере
  - currentInventorySlot != null      // в инвентаре
  - outOfRange == true                // далеко от игрока
  - fixedFramesSinceSpawn < 6         // первые 6 кадров
  - Debugger.kinematicItemsTimer > 0  // отладка
  - meshCol sleeping && dynamicColTimer <= 0  // mesh collider «спит»
```

### 3. SimpleFloatingObject (Crest Ocean System)

**ЭТОГО КОДА НЕТ В ДЕКОМПИЛЯЦИИ!** Это внешний компонент из Crest Ocean System.

Известные параметры (задаются в ItemRigidbody.Start):
| Параметр | Значение | Эффект |
|----------|:-------:|--------|
| `_raiseObject` | `item.floaterHeight` (1.6) | На сколько ПОДНЯТЬ над водой |
| `_dragInWaterRotational` | 0.02 | Вращательное трение в воде |

**Предполагаемая логика SimpleFloatingObject:**
1. Проверяет, находится ли объект ниже уровня воды
2. Если да — применяет выталкивающую силу (force вверх)
3. `_raiseObject` добавляется к целевой высоте — объект поднимается НАД водой
4. При `_raiseObject = 1.6` объект всегда будет на 1.6 единиц ВЫШЕ воды

### 4. Buoyancy — альтернативная система (для лодок)

`Buoyancy` — более сложная система, использующая выборку (блобы):

| Параметр | Значение | Описание |
|----------|:-------:|----------|
| `SlicesX / SlicesZ` | 2 | Сетка 2×2 = 4 точки замера |
| `magnitude` | 2.0 | Множитель силы |
| `dampCoeff` | 0.1 | Демпфирование вертикальной скорости |
| `CenterOfMassOffset` | -1.0 | Смещение центра масс вниз |
| `interpolation` | 3 | Сглаживание по 3 кадрам |
| `ChoppynessAffectsPosition` | — | Течения влияют на позицию |
| `WindAffectsPosition` | — | Ветер двигает объект |
| `sink` | — | Режим затопления |

**Логика Buoyancy:**
```csharp
Для каждой точки (блоб):
  waterHeight = ocean.GetWaterHeightAtLocation2(x - choppy, z)
  delta = magnitude * (blobY - waterHeight)
  // delta > 0: блоб НАД водой → сила ВНИЗ
  // delta < 0: блоб ПОД водой → сила ВВЕРХ
  force = -Vector3.up * (delta + dampCoeff * verticalVelocity)
  AddForceAtPosition(force, blobWorldPos)
```

Это **правильная** модель Архимедовой силы:
- Части над водой = гравитация тянет вниз
- Части под водой = выталкивание вверх
- Демпфирование гасит осцилляции
- Интерполяция сглаживает волны

### 5. Boyant — простейшая плавучесть

```csharp
height = ocean.GetWaterHeightAtLocation2(x - choppy, z) + buoyancy
transform.position = new Vector3(x, height, z)
```

Прямая установка Y-позиции. Без физики, без сил. `buoyancy` — просто смещение.

---

## Полная таблица масс

### Базовые правила
- `ShipItem.mass` (кг) — базовая масса, задаётся в префабе
- `Rigidbody.mass` = базовая + модификаторы
- Масса обновляется в `ItemRigidbody.UpdateMass()`

### Модификаторы массы по подклассам

| Подкласс | Формула |
|----------|---------|
| **ShipItemCrate** | `+ containedPrefab.mass × amount` |
| **ShipItemBottle** | `+ item.health` (объём жидкости) |
| **ShipItemTea** | `+ amount × 0.1` |
| **ShipItemSalt** | `+ amount × 0.1` |
| **ShipItemSoup** | `+ currentWater + currentEnergy/20 + currentUncookedEnergy/20` |
| **ShipItemKettle** | чайник = масса + вода |

### Масса лодки (BoatMass)

```csharp
totalMass = selfMass + partsMass
for each item on boat:
  totalMass += item.GetBody().mass
  centerOfMass += item.localPosition * (item.mass / selfMass) * leverageMult
totalMass += 160  // масса игрока (80 кг × 2?)
centerOfMass += playerPosition * (160 / selfMass) * leverageMult
body.mass = totalMass
body.centerOfMass = keel.centerOfMass + centerOfMass
```

---

## Система столкновений (Decollision)

### PickupableItemCollisionChecker

Компонент на itemRigidbody. Управляет декализией (выталкиванием из коллизий) когда предмет держат:

| Параметр | Значение |
|----------|:-------:|
| `allowObstructedDropping` | `true` если `penetration < 0.06` |
| Красный контур | когда `big item` + коллизия + `penetration >= 0.06` |

**GetDecollision():**
```csharp
decollisionVector = Σ(normal * penetration * 1.8) по всем коллизиям
return transform.InverseTransformVector(decollisionVector)
```

---

## Направленное сопротивление (RigidbodyDirectionalDrag)

Изменяет скорость в локальных осях:
```csharp
localVel = InverseTransformDirection(rb.velocity)
localVel.x *= x    // боковое
localVel.y *= y    // вертикальное
localVel.z *= (vel.z > 0) ? zForward : zBack  // вперёд/назад
rb.velocity = TransformDirection(localVel)
```

---

## Физика в цифрах

| Параметр | Значение | Где |
|----------|:-------:|-----|
| Гравитация | Стандартная Unity (-9.81) | Physics.gravity |
| drag предмета | 1.2 | ItemRigidbody |
| angularDrag предмета | `mass × 0.1` | ItemRigidbody |
| drag в воде (вращ.) | 0.02 | SimpleFloatingObject |
| подъём над водой | **1.6** | floaterHeight |
| демпфирование Buoyancy | 0.1 | Buoyancy.dampCoeff |
| магнитуда Buoyancy | 2.0 | Buoyancy.magnitude |
| декализия множитель | 1.8 | CollisionChecker |
| порог декализии | 0.06 | CollisionChecker |
| порог звука коллизии | debugItemColAudioThreshold | Debugger |
| дальность деактивации | 600 м | ItemRigidbody |
| таймер проверки дальности | 5–8 сек | ItemRigidbody |

---

## Проблемы и их решения для физического API

### 🔴 Проблема 1: Бочки парят над водой
**Причина:** `SimpleFloatingObject._raiseObject = 1.6`
**Решение:** Установить `_raiseObject` ≈ 0.3–0.5 для частичного погружения. Или заменить `SimpleFloatingObject` на кастомный buoyancy (как система `Buoyancy` для лодок).

### 🔴 Проблема 2: Нет вращательного трения в воде
**Причина:** `_dragInWaterRotational = 0.02` — почти ноль
**Решение:** Увеличить до 1.0–3.0 для реалистичного поведения.

### 🔴 Проблема 3: Предметы не трутся о воду
**Причина:** `rigidbody.drag = 1.2` применяется ВЕЗДЕ (и в воздухе)
**Решение:** Разделить drag для воздуха и воды. В воздухе ≈ 0.1, в воде ≈ 2–5.

### 🔴 Проблема 4: isKinematic блокирует физику на лодке
**Причина:** Слишком много условий для кинематики
**Решение:** Оставить некинематическим на лодке (только через parent-трансформации).

### 🟡 Проблема 5: Нет инерции предметов на лодке
**Причина:** Предметы физически не взаимодействуют с лодкой при движении
**Решение:** Добавить силы инерции на itemRigidbody при ускорении лодки.

---

## Резюме: где живёт физика предмета

```
ItemRigidbody.Start()
  ├─ Создаёт Rigidbody: drag=1.2, angularDrag=mass*0.1
  ├─ Создаёт SimpleFloatingObject: _raiseObject=floaterHeight, _dragInWaterRotational=0.02
  ├─ Копирует коллайдеры с ShipItem
  └─ Создаёт субколлайдеры

ItemRigidbody.FixedUpdate()
  ├─ Проверяет outOfRange (600м) → уничтожает если далеко
  ├─ Включает/выключает коллайдеры
  ├─ Перемещает ShipItem ↔ itemRigidbody
  ├─ Управляет isKinematic
  └─ Управляет collisionDetectionMode

ShipItem.ExtraFixedUpdate()
  └─ EnterBoat / ExitBoat логика

PickupableItemCollisionChecker.Update()
  └─ Декализия + красный контур
```

---

*Документ создан на основе полной декомпиляции Assembly-CSharp.dll (Sailwind v0.38).*
*SimpleFloatingObject — компонент Crest Ocean System, его код недоступен в декомпилированном виде.*
