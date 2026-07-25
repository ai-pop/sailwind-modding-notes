# 67. Построение коллайдеров физического twin: тайминг, копирование и CCD

Точный разбор `ItemRigidbody.Start()`, `AddCollider()` и `CreateSubcollider()` в Sailwind v0.38. Эта заметка дополняет twin-модель из заметок [16](16-item-framework-shipitem.md), [43](43-item-buoyancy-water.md) и [44](44-itemrigidbody-field-map-contract.md).

## Коротко: когда twin действительно готов к физике

Наличие `ShipItem.itemRigidbodyC` **ещё не означает**, что на twin уже есть рабочие коллайдеры.

```text
ShipItem.LoadAfterDelay coroutine
  └─ CreateRigidbody()
       └─ AddComponent<ItemRigidbody>()
            └─ ItemRigidbody.Start()
                 ├─ AddComponent<Rigidbody>()
                 ├─ StartCoroutine(AddCollider())
                 ├─ создаёт SimpleFloatingObject
                 └─ немедленно копирует дочерние ItemSubcollider

AddCollider()
  ├─ yield WaitForFixedUpdate × 3
  ├─ yield WaitForEndOfFrame
  └─ создаёт root Box/Mesh/CapsuleCollider на twin
```

**Практическое правило:** мод, которому нужны физические коллайдеры twin, должен ждать как минимум **три physics tick и EndOfFrame** после появления `ItemRigidbody`; ещё надёжнее — поллить конкретный `Collider` на twin. Нельзя измерять bounds или менять collision mode только по факту `itemRigidbodyC != null`.

## Что `Start()` создаёт сразу

В `ItemRigidbody.Start()` игра:

```csharp
rigidbody = gameObject.AddComponent<Rigidbody>();
rigidbody.drag = 1.2f;
rigidbody.angularDrag = item.mass * 0.1f;
rigidbody.isKinematic = true;
UpdateMass();
StartCoroutine(AddCollider());
floater = gameObject.AddComponent<SimpleFloatingObject>();
gameObject.layer = 2;
```

То есть Rigidbody уже существует, но root collider ещё нет. В начале тело кинематическое; окончательное решение о `isKinematic` принимает `ItemRigidbody.FixedUpdate` (заметки 43–44).

## Root-коллайдеры: откуда и как копируются

После задержки `AddCollider()` смотрит **только** на компоненты верхнего уровня visual-объекта:

```csharp
BoxCollider     box     = item.GetComponent<BoxCollider>();
MeshCollider    mesh    = item.GetComponent<MeshCollider>();
CapsuleCollider capsule = item.GetComponent<CapsuleCollider>();
```

| Коллайдер visual | Что создаётся на twin | Копируемые свойства |
|---|---|---|
| `BoxCollider` | новый `BoxCollider` | `center`, `size` |
| `MeshCollider` | новый `MeshCollider` | `sharedMesh`; всегда `convex = true` |
| `CapsuleCollider` | новый `CapsuleCollider` | `center`, `radius`, `height`, `direction` |

Скрипт не копирует `material`, `contactOffset`, `isTrigger`, `enabled`, слои, тег или дополнительные custom-настройки коллайдера. Настраивать это для twin нужно явно и после его создания.

### Важный побочный эффект: visual MeshCollider удаляется

Если на visual есть `MeshCollider` **и** `BoxCollider` либо `CapsuleCollider`, игра после копирования вызывает:

```csharp
Destroy(item.GetComponent<MeshCollider>());
```

Twin уже получил собственный convex mesh, но исходный `MeshCollider` на **visual** будет уничтожен. Следовательно, мод не должен хранить долгоживущую ссылку на visual `MeshCollider` и ожидать, что она переживёт инициализацию предмета.

## Дочерние коллайдеры: только тег `ItemSubcollider`

Дочерний Transform visual-объекта копируется в twin только при теге `ItemSubcollider`:

```csharp
foreach (Transform child in item.transform)
    if (child.CompareTag("ItemSubcollider")) CreateSubcollider(child);
```

`CreateSubcollider()` инстанцирует целиком GameObject, репарентит его на twin, сохраняет local position/rotation, затем принудительно задаёт:

```csharp
clone.tag = "Untagged";
clone.layer = 2;
clone.GetComponent<Collider>().isTrigger = false;
```

| Свойство clone | Значение |
|---|---|
| Родитель | twin `ItemRigidbody.transform` |
| Поза | те же `localPosition` и `localRotation` |
| Тег | `Untagged` |
| Layer | `2` (`IgnoreRaycast`) |
| Collider | non-trigger до последующей логики hold/inventory |

Обычный дочерний `Collider` без этого тега **не копируется**. В то же время `CreateSubcollider()` работает в `Start()` до coroutine-задержки, поэтому tagged subcollider может появиться раньше root collider.

## Collision detection mode: фактическая таблица

В Unity 2019 `CollisionDetectionMode` имеет значения:

| Число в декомпиляции | Значение Unity |
|---:|---|
| `2` | `ContinuousDynamic` |
| `3` | `ContinuousSpeculative` |

После `AddCollider()` ваниль выбирает режим так:

```csharp
rigidbody.collisionDetectionMode = meshCol != null
    ? (CollisionDetectionMode)2
    : (CollisionDetectionMode)3;
```

Следовательно:

| Форма twin | Режим после `AddCollider()` |
|---|---|
| Есть root `MeshCollider` | `ContinuousDynamic` (2) |
| Только Box/Capsule/no root mesh | `ContinuousSpeculative` (3) |

Это исправляет неточность в заметке 16: значение `3` — не `ContinuousDynamic`, а **`ContinuousSpeculative`**.

### Временное усиление после drop/pickup

`dynamicColTimer` выставляется в `6f` при старте и при held-пути. В `FixedUpdate`:

```csharp
if (!item.held && dynamicColTimer > 0f && !rigidbody.isKinematic)
{
    rigidbody.collisionDetectionMode = (CollisionDetectionMode)2;
    dynamicColTimer -= Time.deltaTime;
}
if (dynamicColTimer <= 0f && meshCol == null)
    rigidbody.collisionDetectionMode = (CollisionDetectionMode)3;
```

Значит box/capsule twin после освобождения примерно шесть секунд использует `ContinuousDynamic`, затем возвращается к `ContinuousSpeculative`. Mesh twin остаётся в `ContinuousDynamic`, потому что последний `if` применим только при `meshCol == null`.

## Практические выводы для моддера

1. **Ждите readiness коллайдера, а не только twin.** `itemRigidbodyC` появляется раньше root collider примерно на 3 fixed-frame + EndOfFrame.
2. **Работайте с twin.** Именно на нём находятся collider copies и Rigidbody; visual collider — отдельная interaction/raycast-геометрия.
3. **Не используйте магические числа CCD.** Для Sailwind v0.38 `2 = ContinuousDynamic`, `3 = ContinuousSpeculative`.
4. **Mesh + primitive на visual не означает два вечных visual коллайдера.** Visual `MeshCollider` будет уничтожен; twin сохранит convex copy.
5. **Чтобы дочерняя физическая форма попала на twin**, дочерний GO должен иметь тег `ItemSubcollider`; обычный child collider ваниль игнорирует.
6. **Тег clone не наследуется:** `ItemSubcollider` превращается в `Untagged` на layer 2. Это важно для фильтров по тегам и collision matrix.
