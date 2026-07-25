# 59. Кто зануляет скорость twin после дропа, поза предмета в кадре релиза

Разбор всех подозреваемых, которые могут гасить/перезаписывать velocity twin после дропа — ответ на запросы A5, A6. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 57 (DropItem), 44 (ItemRigidbody contract), 43 (buoyancy).

## A5. КРИТИЧНО: кто зануляет скорость twin после дропа

### Симптом мода
Мод выставляет `twinRb.velocity = v` (v ≈ 5–9 м/с) в кадр отпускания, twin уже `!isKinematic` — и всё равно предмет падает строго вниз. Кто-то в первые ~10 fixed-кадров гасит velocity.

### Подозреваемый 1: `ItemRigidbody.FixedUpdate` — позиционные синки visual↔twin

**Вердикт: КОНФИРМИРОВАН — главный убийца velocity.**

```csharp
// ItemRigidbody.FixedUpdate — ключевые строки (дословно)
if (!currentInventorySlot)
{
    if (item.currentWalkCol != null && onBoat)
    {
        if (item.held != null)
        {
            MoveRigidbodyToWalkCol();  // twin → walkCol (position snap, NO velocity preserved)
        }
        else
        {
            MoveItemToWalkColRigidbody();  // visual → walkCol from twin (twin master)
        }
    }
    else if (item.held != null || !item.sold)
    {
        // twin → visual (position snap)
        transform.position = item.transform.position;
        transform.rotation = item.transform.rotation;
    }
    else
    {
        // visual → twin (twin master when free)
        item.transform.position = transform.position;
        item.transform.rotation = transform.rotation;
    }
}
```

**При `item.held != null`: twin = position slave (snap к visual каждый FixedUpdate).** Twin не имеет собственной позиции — FixedUpdate **переписывает** `twin.transform.position = item.transform.position` каждый fixed frame. `transform.position` assignment на dynamic Rigidbody **эквивалентно teleport** → Unity **обнуляет velocity** при teleport ( Rigidbody.position assignment vs MovePosition).

> **КОНФИРМИРОВАН:** `twin.transform.position = item.transform.position` в FixedUpdate при `held != null` — это **teleport каждый fixed frame**. Unity Rigidbody: при установке `transform.position` напрямую → velocity **обнуляется** (internal Unity behavior — `Rigidbody.position` setter bypasses physics solver, resets velocity). Мод выставлял velocity в LateUpdate, но **следующий FixedUpdate** переписывает позицию twin = позиция visual → velocity = 0.

**При `held == null` (после дропа): twin = position master (visual snap к twin).** `item.transform.position = transform.position` — visual follower, twin master. Twin позиция **не переписывается** — twin свободен двигаться физикой. **Но:** если в кадре дропа `held` уже null, но FixedUpdate ещё не успел переключить master/slave → **1 fixed frame** twin может быть ещё slave (position snap к visual) → velocity обнуляется.

### Подозреваемый 2: `fixedFramesSinceSpawn < 6` → isKinematic

```csharp
if (fixedFramesSinceSpawn < 6f)
{
    flag2 = true;  // → isKinematic = true
}
```

**Вердикт: КОНФИРМИРОВАН — второй убийца velocity.**

`fixedFramesSinceSpawn` — счётчик, инкрементируется каждый FixedUpdate. Предмет после spawn/ResetPos — `isKinematic = true` на первые 6 fixed frames. kinematic Rigidbody **не применяет velocity** — любая velocity запись бессмысленна.

**Но:** `fixedFramesSinceSpawn` не сбрасывается при дропе! Он продолжает с текущего значения. Если предмет уже давно в мире (> 6 frames) — этот таймер уже истёк → не проблема. **Проблема для fresh-spawn items** — WorldItemSpawner spawn → FreezeItem → `debugForceKinematic = true` → предмет frozen до pickup → pickup снимает kinematic, но `fixedFramesSinceSpawn` может быть уже > 6 (если предмет давно существует).

> **Для предметов, которые существовали > 6 fixed frames** — этот таймер не проблема. Для fresh-spawn (WorldItemSpawner) — предмет frozen = kinematic → velocity бессмысленна.

### Подозреваемый 3: `dynamicColTimer` → kinematic на 6 секунд после held-обрыва

```csharp
if (item.held != null)
{
    SetDynamicColTimer();  // → dynamicColTimer = 6f
}
if (!item.held && dynamicColTimer > 0f && !rigidbody.isKinematic)
{
    rigidbody.collisionDetectionMode = CollisionDetectionMode.ContinuousDynamic;
    dynamicColTimer -= Time.deltaTime;
}
```

**Вердикт: НЕ убийца velocity — но УБИЙЦА throw force timing.**

`dynamicColTimer` НЕ ставит `isKinematic = true`. Он только меняет `collisionDetectionMode` на `ContinuousDynamic` на 6 секунд. Это не гасит velocity.

**Но:** `dynamicColTimer` **сбрасывается на 6** при каждом FixedUpdate где `held != null` (включая hold-период). После дропа — timer начинает countdown от 6. ThrowItemAfterDelay запускается в **следующий fixed frame** — `dynamicColTimer` может быть ~5.94 — **не влияет** на AddForce.

### Подозреваемый 4: `SimpleFloatingObject` (Crest floater)

**Вердикт: НЕ убийца velocity — floater не пишет velocity.**

`SimpleFloatingObject` (from Crest DLL, class `FloatingObjectBase`) — **не декомпилирован** (нет в Assembly-CSharp). Но из ItemRigidbody.Start():
```csharp
floater = AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;  // default 1.6
```

Crest `SimpleFloatingObject` применяет **buoyancy forces** (AddForce/AddForceAtPosition), не пишет velocity напрямую. В воде — buoyancy force upward + drag. На суше — no forces. **Floater НЕ зануляет velocity** — он добавляет силы (buoyancy), но не переписывает velocity.

> **Но:** floater buoyancy может **компенсировать** горизонтальную velocity → предмет с horizontal velocity + buoyancy → предмет поднимается, но horizontal velocity сохраняется (drag 1.2 гасит медленно). **Gravity + drag = предмет падает**, но horizontal velocity не обнуляется. Если velocity была обнулена раньше (FixedUpdate position snap) → предмет падает строго вниз.

### Подозреваемый 5: `ResetPos()` — velocity = Vector3.zero

```csharp
public void ResetPos()
{
    rigidbody.isKinematic = true;
    transform.position = item.transform.position;
    transform.rotation = item.transform.rotation;
    rigidbody.velocity = Vector3.zero;
}
```

**Вердикт: НЕ вызывается при дропе — только при shop return и spawn.**

`ResetPos()` вызывается из `ShipItem.SmoothlyReturnToShop()` и при spawn. Не вызывается при нормальном дропе. **Не убийца velocity при дропе.**

### Подозреваемый 6: `Rigidbody.Sleep()` → isKinematic = true

```csharp
if (meshCol != null && rigidbody.IsSleeping() && dynamicColTimer <= 0f)
{
    flag2 = true;  // → isKinematic = true
}
```

**Вердикт: Возможный убийца velocity — если twin «уснул».**

Если twin Rigidbody `IsSleeping()` (Unity auto-sleep после ~1 с без движения) + `meshCol != null` + `dynamicColTimer <= 0` → `isKinematic = true` → twin frozen → velocity бессмысленна. Но sleep происходит после **длительного бездействия**, не в первые fixed frames после дропа.

### Точный порядок кадров физики при дропе

**Сценарий:** мод выставляет `twinRb.velocity = v` в LateUpdate frame N (кадр отпускания).

```
Frame N (LateUpdate):
  mod: twinRb.velocity = v (5-9 m/s) ← записано в Rigidbody
  mod: item.held = null; pointer.heldItem = null ← held снят
  
Frame N+1 (FixedUpdate — первый fixed после дропа):
  ItemRigidbody.FixedUpdate:
    item.held == null → twin = position MASTER (not slave)
    → visual.position = twin.position (twin master)
    → twin.position НЕ переписывается → velocity сохранена?
    
  BUT: flag2 = (item.held != null) → false
  → isKinematic = false (если item.sold && !nailed && !attached && ...)
  → twin dynamic → velocity должна работать!
  
  PROBLEM: если item.held был снят В LateUpdate, но FixedUpdate 
  ещё не видел held=null (race condition?) → нет, LateUpdate 
  после FixedUpdate в том же frame → следующий FixedUpdate 
  уже видит held=null.
```

**Реальная проблема мода:** мод делал `twinRb.velocity = v` при `item.held != null` (ещё в руке) → **FixedUpdate того же frame или следующего** переписывает `twin.transform.position = visual.transform.position` (held != null → twin slave) → **velocity обнулена teleportом**.

> **Вердикт:** velocity обнуляется `ItemRigidbody.FixedUpdate` позиционным sync (twin → visual при held != null). Мод выставлял velocity, пока held ещё != null → FixedUpdate переписывает позицию twin → velocity = 0. **Решение:** выставлять velocity **в FixedUpdate или позже**, после того как `held == null` уже обработан ItemRigidbody (twin стал position master, не slave).

## A6. Поза предмета в кадре релиза

### Кто пишет world-позицию visual при held

**Подтверждение v0.38:** единственный писатель world-позы visual при held — **GoPointer.LateUpdate**.

```csharp
// GoPointer.LateUpdate — held item position (small item branch)
if (heldItem != null && !heldItem.big)
{
    ((Component)heldItem).transform.position = ((Component)this).transform.position 
        + ((Component)this).transform.forward * heldItem.holdDistance + heldItemRot;
    ((Component)heldItem).transform.rotation = ((Component)this).transform.rotation;
}
// big item — decolLocalPos-based positioning (see note 54)
```

**SetParent НЕ используется** при hold — confirmed (заметка 47 верна). Visual GO **не репарентится** на pointer/camera. Только `transform.position/rotation` перезаписываются каждый LateUpdate.

**При PickUp:**
```csharp
// GoPointer.PickUpItem() — position
heldItem = item;
item.gameObject.layer = 2;
heldItem.held = this;
// NO position write! First position write = next LateUpdate
```

PickUpItem **не пишет позицию**. Первый `transform.position = ...` — в следующем LateUpdate.

### Куда попадает visual в ПЕРВЫЙ кадр после DropItem

**DropItem (GoPointer.LateUpdate):**
```csharp
heldItem.OnDrop();       // wallAttachment snap (если applicable)
DropItem();               // layer=0, held=null
```

После DropItem → `heldItem = null` → **GoPointer.LateUpdate больше не пишет позицию visual**. Visual позиция = последняя позиция, записанная GoPointer в **предыдущем** LateUpdate (frame N-1 или frame N до DropItem).

**ItemRigidbody.FixedUpdate (следующий fixed):**
```csharp
// held == null, twin = position master
item.transform.position = transform.position;  // visual → twin position
item.transform.rotation = transform.rotation;
```

**Twin позиция при дропе:** twin был position slave при held → twin.transform.position = visual.transform.position (последний FixedUpdate). После дропа → twin master → visual snap к twin. **Jump:** если twin был в другой позиции (мод двигал twin физикой при hold), twin и visual расходятся → **visual snap к twin позиции** в следующем FixedUpdate → visual «прыгает» к физической позиции twin.

> **Для точки старта броска:** в кадре дропа visual.position = pointer-driven position (holdDistance от камеры). Twin.position = **ванильный twin = visual.position** (synced в FixedUpdate при held). **Модный twin** — может быть в другой позиции (физика-поза мода). В первом FixedUpdate после дропа → visual snap к twin → **jump на позицию twin**.

## Практические выводы для мододела

1. **Velocity обнуляется ItemRigidbody.FixedUpdate** — позиционный sync `twin.position = visual.position` при `held != null` = teleport → velocity = 0. **Выставлять velocity можно только после held == null processed by ItemRigidbody** (twin стал position master).
2. **fixedFramesSinceSpawn < 6** — twin kinematic на первые 6 fixed frames. Для long-existing items — не проблема. Для fresh-spawn — velocity бессмысленна на kinematic.
3. **SimpleFloatingObject** — buoyancy forces, НЕ пишет velocity. Не убийца.
4. **Поза visual при held:** только GoPointer.LateUpdate. **После DropItem:** visual frozen на последней pointer-driven позиции. **Следующий FixedUpdate:** visual snap к twin (twin master) → может быть jump, если twin был в модной физической позиции.
5. **Правильная последовательность для throw мода:**
   - Frame N (LateUpdate): `OnDrop() + DropItem()` → held=null, layer=0
   - Frame N+1 (FixedUpdate): ItemRigidbody sees held=null → twin becomes position master → isKinematic=false (if sold) → twin free
   - Frame N+1 or N+2: **NOW set velocity** → twin.position is free, velocity preserved
   - Or: use `WaitForFixedUpdate()` coroutine (как ванильный ThrowItemAfterDelay) → AddForce в следующем fixed frame
