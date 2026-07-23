# 54. Деколлизия больших предметов в GoPointer: полный алгоритм

Разбор алгоритма деколлизии `GoPointer.LateUpdate` для `big`-предметов — ответ на запрос E4. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 52 (слои), 53 (EnterBoat/ExitBoat), 33 (подбор).

## Поля деколлизии в GoPointer

| Поле | Тип | Содержание |
|------|-----|-----------|
| `bigItemLocalPos` | `Vector3` | Целевая позиция big-item в **локальных координатах камеры** (forward × holdDistance). |
| `decolLocalPos` | `Vector3` | **Реальная позиция big-item** в локальных координатах камеры — с учётом деколлизии (отодвинут от стен). Инициализируется = `bigItemLocalPos` при PickUp. |
| `bigItemLocalRot` | `Quaternion` | Поворот big-item относительно камеры (scroll-rotation). |
| `bigItemLocked` | `bool` | (не видно в LateUpdate — возможно старое поле) |
| `bigItemLookTimer` | `float` | Таймер для задержки смены деколлизии |
| `currentBigItemLook` | `ShipItem` | Предмет, который «смотрится» как big |
| `lastPointerRot` | `Quaternion` | Предыдущий поворот GoPointer — для доворота decolLocalPos при повороте камеры |

## Полный алгоритм big-item в `GoPointer.LateUpdate`

```csharp
// GoPointer.LateUpdate — ветка big item (дословно ключевые строки)
// Предмет уже heldItem.big == true

float num2 = Mathf.InverseLerp(throwDelay, 1f, currentThrowPower);

// 1. Установка позиции/поворота из decolLocalPos:
Quaternion rotation = transform.rotation * bigItemLocalRot;
heldItem.transform.position = transform.TransformPoint(decolLocalPos);
heldItem.transform.rotation = rotation;

// 2. Отодвигание назад при подготовке броска:
heldItem.transform.Translate(transform.forward * (0f - num2) * 0.4f, Space.World);

// 3. Scroll rotation:
heldItem.transform.Rotate(transform.right, heldItem.heldRotationOffset, Space.World);

// 4. ДЕКОЛЛИЗИЯ — ComputePenetration:
Vector3 decollision = heldItem.colChecker.GetDecollision();
decollection = heldItem.transform.TransformVector(decollection); // world-space

// 5. Преобразование в локальные координаты камеры:
Vector3 val = transform.InverseTransformPoint(heldItem.transform.position + decollection);

// 6. Лимитные проверки (если деколлизия выталкивает слишком далеко):
if (val.z < 0.6f)         { val = decolLocalPos; Debug.Log("Close decol limit"); }
else if (val.z < -1.4f)    { val = decolLocalPos; Debug.Log("Decol limit"); }
else if (Mathf.Abs(val.x) > 1.4f) { val = decolLocalPos; Debug.Log("Decol limit"); }
else if (Mathf.Abs(val.y) > 1.4f) { val = decolLocalPos; Debug.Log("Decol limit"); }

// 7. Плавное сведение:
decolLocalPos = Vector3.Lerp(decolLocalPos, val, Time.deltaTime * 3f);

// 8. Доворот при повороте камеры:
float num3 = Quaternion.Angle(transform.rotation, lastPointerRot);
decolLocalPos = Vector3.Lerp(decolLocalPos, bigItemLocalPos, Time.deltaTime * num3 * 1.2f);
lastPointerRot = transform.rotation;

// 9. Красный контур:
if (heldItem.colChecker.collisions > 0) { /* (пустой блок) */ }
```

**Нет `while`-циклов!** Деколлизия — single-pass: `GetDecollision()` → Lerp. **While-зависание невозможно в этом коде.**

## `PickupableItemCollisionChecker` — подробности

```csharp
// PickupableItemCollisionChecker.cs — ключевые методы

// OnTriggerEnter: если other не isTrigger, не own item GO, не тег Boat → collisions++, add to list
// OnTriggerExit:  если other не isTrigger, не own item GO, не тег Boat → collisions--, remove from list

// GetDecollision():
//   foreach collidedCol in collidedCols:
//     Physics.ComputePenetration(this.collider, this.pos, this.rot, collidedCol, col.pos, col.rot, ref dir, ref dist)
//     decollisionVector += dir * dist * 1.8f
//   return transform.InverseTransformVector(decollectionVector)

// Update():
//   if held && not in inventory → UpdateDecolDistance() + allowObstructedDropping = currentDecolDistance < 0.06f
//   else → collisions = 0
//   enableRedOutline = (collisions>0 && big && !allowObstructedDropping) || (collisions>0 && InputName8 && !allowObstructedDropping)
```

### Критичные детали для мода

1. **`OnTriggerEnter/Exit`** — на **visual GO** ShipItem (триггерные коллайдеры). Это `isTrigger=true` коллайдеры на visual.
2. **Фильтр `!other.CompareTag("Boat")`:** деколлизия **игнорирует лодку** (тег `Boat`). То есть ванильная деколлизия **не считает** столкновение с лодкой как коллизию → `collisions` не инкрементируется → красный контур не показывается → деколлизия не выталкивает от лодки.
3. **`ComputePenetration`** вызывается для **visual-коллайдера** (isTrigger=true) против collidedCols (non-trigger). `ComputePenetration` работает **независимо** от isTrigger — она проверяет геометрическое пересечение двух коллайдеров. Но: **collidedCols** — только non-trigger коллайдеры (фильтр в OnTriggerEnter: `!other.isTrigger`). Триггерные коллайдеры лодки **не попадают** в collidedCols.

> **Ключевой вывод:** ванильная деколлизия **исключает лодку** (тег `Boat`) и **исключает trigger-коллайдеры** из списка collidedCols. Twin-коллайдеры предмета (на слое 2, isTrigger=true в ваниле) — тоже trigger → не попадают в collidedCols. Мод делает twin non-trigger → **twin коллайдер попадает в OnTriggerEnter других предметов** — но **Boat тег исключает** лодку.

## Видит ли предмет «сам себя»?

`OnTriggerEnter`: условие `(Object)(object)((Component)other).gameObject != (Object)(object)((Component)item).gameObject` — **исключает own visual GO**. Но twin GO — **другой gameObject** (с именем `itemName:ItemRigidbody`). Twin-коллайдер (non-trigger) **не исключён** из collidedCols.

> **Если twin-коллайдер non-trigger и overlap с visual trigger-коллайдером того же предмета** → twin попадает в collidedCols → ComputePenetration visual vs twin → деколлизия выталкивает предмет от самого себя. Это маловероятно (visual и twin обычно далеко друг от друга при held), но при определённых геометриях — возможно.

## Практические выводы

1. **Деколлизия — нет while-циклов**: `GetDecollision()` — single-pass по collidedCols, Lerp в GoPointer.LateUpdate. **Зависание от деколлизии невозможно.**
2. **Деколлизия исключает лодку** (тег `Boat`) → ванильный held big-item **не отодвигается от лодки**.
3. **Мод с non-trigger twin:** twin-коллайдер может попасть в OnTriggerEnter (visual-side) как non-trigger → collisions++ → деколлизия пытается выталкивать от twin'а самого предмета → **potential self-interaction bug**.
4. **Краш не от деколлизии, а от `OnCollisionEnter`** на twin (non-trigger) → `ItemRigidbody.OnCollisionEnter` → `ItemCollisionSoundPlayer.PlayWoodColSound` — не крашит (AudioSource pool), но **BoatImpactSoundsCollider.OnCollisionEnter → BoatImpactSounds.Impact → BoatDamage.Impact** — вот потенциальный краш-путь (см. заметку 55).
5. Лимиты деколлизии: z < 0.6 / z < -1.4 / |x| > 1.4 / |y| > 1.4 → fallback to `decolLocalPos` (без выталкивания). При предмете внутри CapsuleCollider лодки — ComputePenetration может вернуть большой вектор → лимитный fallback → предмет «замирает» на последней safe-позиции.
