# 47. Механика удержания предмета в руках: полный поток (end-to-end)

Разбор потока подбора → удержания → дропа предмета, классы-участники, поля, фазы. Информация из декомпиляции `Assembly-CSharp.dll` (Sailwind v0.38). Связано с заметками 16, 33, 43, 44.

## A1. Поток подбора/удержания/дропа

### Кто выполняет raycast подбора

Класс: **`GoPointer`** — центральный класс взаимодействия. Метод: `DoRaycast()`.

**Фаза:** `FixedUpdate()` — каждый fixed-кадр формирует `raycastRay` и вызывает `DoRaycast()`.

**DoRaycast():**
```csharp
Ray val = debugEditorPointer ? raycastRay : Camera.main.ScreenPointToRay(Input.mousePosition);
LayerMask val2 = LayerMask.op_Implicit(-604165);   // маска коллайдеров предмета
float num = 1.8f;                                   // макс. дистанция луча
if (Physics.Raycast(val, ref hit, num, LayerMask.op_Implicit(val2)) 
    && !GameState.sleeping && !GameState.inBed && !BoatCamera.on)
```

Raycast бьёт по **коллайдерам visual** (`GoPointerButton`, `ShipItem`) на слое, заданном маской `-604165`. Коллайдеры visual — **isTrigger** (становятся в `ShipItem.Awake`). Если попали в `ItemSubcollider`, берётся родительский коллайдер.

**Подбор происходит в `LateUpdate()` `GoPointer`:**
```csharp
if (MainButtonDown() && !BoatCamera.on) {
    // ... stickyClickedButton или pointedAtButton ...
    if ((Object)(object)((Component)pointedAtButton).GetComponent<PickupableItem>() != (Object)null 
        && (!((Object)(object)((Component)pointedAtButton).GetComponent<ShipItem>() != (Object)null) 
             || !((Component)pointedAtButton).GetComponent<ShipItem>().nailed))
    {
        PickUpItem(((Component)clickedButton).GetComponent<PickupableItem>());
    }
}
```

**Важно:** подбор — в `LateUpdate`, НЕ в `FixedUpdate`. Raycast — в `FixedUpdate`, но вызов `PickUpItem` — в `LateUpdate` (кнопка нажата = MainButtonDown).

### Поле `ShipItem.held` (на самом деле `PickupableItem.held`)

Тип: **`GoPointer`** (public, на `PickupableItem`, базовый для `ShipItem`).

**Места присваивания:**
| Где | Код | Что |
|-----|------|------|
| `GoPointer.PickUpItem()` | `heldItem.held = this;` | Присвоить: указатель, который держит предмет |
| `GoPointer.DropItem()` | `heldItem.held = null;` | Очистить при дропе |
| `WorldItemSpawner.Update()` | `if (item.held != null)` → коoldown + `item.GetItemRigidbody().debugForceKinematic = false; item = null;` | Spawner отслеживает подбор, размораживает и отвязывается |

**Может ли `held` кратковременно становиться `null` в течение удержания?**  
Нет. `held` присваивается в `PickUpItem()` и сбрасывается только в `DropItem()`. Переходов между нет. **Исключение:** мод может вызвать `DropItem()` через Harmony-patch, но ванильный поток — `held` стабилен от PickUp до Drop.

**Однако:** `held` ссылается на `GoPointer`, а `GoPointer` — singleton-компонент на игроке. Если `GoPointer` уничтожается (смена сцены, etc.) — ссылка станет `null`, но этого в нормальном геймплеe не бывает.

### Кто двигает визуал предмета в руке

**Класс:** `GoPointer`  
**Метод:** `LateUpdate()`  
**Фаза:** `LateUpdate` (каждый кадр, после Update)  
**Нет `[DefaultExecutionOrder]`** на `GoPointer`.

Движение зависит от типа предмета (`heldItem.big`):

**Маленький предмет (`!big`), если не наведён на кнопку с allowPlacingItems:**
```csharp
float num2 = Mathf.InverseLerp(throwDelay, 1f, currentThrowPower);
Vector3 val2 = transform.position + transform.forward * heldItem.holdDistance + transform.up * heldItem.holdHeight;
Quaternion val3 = transform.rotation * Quaternion.Euler(heldItem.heldRotationOffset, 0f, 0f);
heldItem.transform.position = Vector3.Lerp(heldItem.transform.position, val2, timerAfterPickup);
heldItem.transform.rotation = Quaternion.Lerp(heldItem.transform.rotation, val3, timerAfterPickup);
heldItem.transform.Translate(transform.forward * (0f - num2) * 0.3f, Space.World);
```

- **Якорь:** позиция `GoPointer` (pointer GO) + `forward * holdDistance` + `up * holdHeight`  
- `holdDistance` по умолч. = `1.15f`, `holdHeight` = `0f`  
- `timerAfterPickup` нарастает от 0 до 1 (`+= deltaTime * 4`), используется как Lerp-фактор → предмет плавно подтягивается к позиции руки
- **World-space `position`** (не SetParent, не local) — предмет ставится прямо в мировые координаты

**Маленький предмет, наведён на кнопку (allowPlacingItems):**
```csharp
heldItem.transform.position = transform.position + transform.forward * currentLookDistance + Vector3.up * heldItem.furniturePlaceHeight;
heldItem.transform.rotation = transform.rotation;
heldItem.transform.Rotate(Vector3.right, heldItem.heldRotationOffset, Space.Self);
```
- Предмет ставится на позицию raycast-хита (у дистанции взгляда)

**Большой предмет (`big`):**
```csharp
Quaternion rotation = transform.rotation * bigItemLocalRot;
heldItem.transform.position = transform.TransformPoint(decolLocalPos);
heldItem.transform.rotation = rotation;
heldItem.transform.Translate(transform.forward * (0f - num2) * 0.4f, Space.World);
heldItem.transform.Rotate(transform.right, heldItem.heldRotationOffset, Space.World);
// + деколлизия и ограничения decolLocalPos
```
- Позиция в локальных координатах pointer (`decolLocalPos`), затем TransformPoint в мировые
- `bigItemLocalPos` и `bigItemLocalRot` сохраняются в `PickUpItem()` из текущей позиции предмета относительно pointer

**Критично:** визуал предмета ставится **в мировых координатах**, без SetParent. Twin в `ItemRigidbody.FixedUpdate` при `held` **следует за visual**: `twin.position = visual.position`. Это означает, что **визуал = master**, twin = follower, и визуал ставится `GoPointer.LateUpdate`.

### Порядок выполнения фаз

```
FixedUpdate (ItemRigidbody)  → twin.position = visual.position (если held)
Update (ShipItem)            → ProcessSaveable, wallAttachment raycast
FixedUpdate (GoPointer)      → raycast
LateUpdate (GoPointer)       → двигает visual, подбирает/дропает
LateUpdate (ItemRigidbody)   → инвентарь лерп, scale
```

**Внимание:** `ItemRigidbody.FixedUpdate` читает `visual.position` и ставит `twin.position = visual.position`. Но `GoPointer.LateUpdate` ещё не выполнился в этом кадре! В `FixedUpdate` visual ещё на позиции **предыдущего LateUpdate**. Twin копирует «предыдущую» позицию, а затем `GoPointer.LateUpdate` двигает visual на новую. На следующем `FixedUpdate` twin снова копирует новую позицию.

**Для мода:** если мод двигает twin через `AddForce` в `FixedUpdate`, а затем `ItemRigidbody.FixedUpdate` перезаписывает twin.position = visual.position → сила теряется! Это **известный конфликт** для мода, перехватывающего удержание предмета.

## A2. Состояние twin (ItemRigidbody) в руках и при отпускании

### ResetPos при подборе?

**Нет.** `PickUpItem()` не вызывает `ResetPos()`. `ResetPos()` вызывается только в:
- `ItemRigidbody.Start()` (после создания twin)
- `ShipItem.ResetRigidbody()` (обёртка)
- `ShipItem.SmoothlyReturnToShop()` (возврат в магазин)

При подборе twin **остаётся в текущей позиции**, а `ItemRigidbody.FixedUpdate` в следующем кадре ставит `twin.position = visual.position` → twin **прыгает** к visual. Но `GoPointer.LateUpdate` тоже двигает visual → к концу кадра визуал уже на позиции руки, а на следующем `FixedUpdate` twin подтянется.

**Что происходит с коллайдерами twin при подборе?**

`ItemRigidbody.FixedUpdate`: при `held` → коллайдеры twin ставятся **isTrigger = true** (boxCol, meshCol, capsuleCol, subcolliders). Это означает twin **не толкает физику** в руках.

`ToggleCollider(true)` вызывается, если предмет **не в инвентаре** (включает коллайдеры, но выключает floater — всегда). При `held` + не в инвентаре → `ToggleCollider(true)` + `isTrigger = true` на коллайдерах.

### Что происходит при отпускании (дроп)

`GoPointer.DropItem()`:
```csharp
heldItem.held = null;
heldItem = null;
```

Затем (если бросок): `ThrowItemAfterDelay` — `AddForce` на Rigidbody twin в следующем `WaitForFixedUpdate`.

После дропа, в `ItemRigidbody.FixedUpdate`:
- `held` = null → `flag2` не ставится от held → тело может стать динамическим
- `SetDynamicColTimer()` — **НЕ вызывается** при дропе (он вызывался при подборе в каждом кадре). `dynamicColTimer` = что осталось от удержания
- Коллайдеры twin: `isTrigger = false` (held = null → не trigger)
- Master: **twin → visual** (`visual.position = twin.position`)

**Переход к первым кадрам полёта:**
1. `held = null` → FixedUpdate: `flag2` не включает кинематику от held, но `fixedFramesSinceSpawn < 6` и `dynamicColTimer` могли ещё быть > 0 → тело ещё кинематическое!
2. После нескольких fixed-кадров кинематика снимается → тело динамическое, летит с throwForce
3. `ToggleCollider(true)` — коллайдеры включены, floater выключен

## A3. ВСЕ пути, телепортирующие twin на сотни метров при живом визуале

### EnterInventorySlot / ExitInventorySlot

```csharp
public void EnterInventorySlot(Transform slot) {
    currentInventorySlot = slot;
    item.GetComponent<Collider>().enabled = false;  // visual collider OFF
    item.OnEnterInventory();
    item.gameObject.layer = 5;                        // UI layer
    // + все дети → layer 5
    inventoryEnterTimer = 0.3f;
}
```

**Где физически расположен `slot`?** `slot` = transform `GPButtonInventorySlot` (GO инвентарного слота). Это объект **под `PlayerNeedsUI`** (UI-панель инвентаря). GO `PlayerNeedsUI` — в сцене, **движется с игроком** (обновляет позицию к `Camera.main` в Update). Но `GPButtonInventorySlot.transform` — это transform слота **в UI-пространстве**. Он **не двигается с FloatingOrigin shift** — он ребёнок UI-панели, которая является частью Canvas/игрового мира.

**В `ItemRigidbody.LateUpdate` (инвентарь):**
```csharp
Vector3 val = currentInventorySlot.TransformPoint(slotLocalPos);
item.transform.position = Vector3.Lerp(val, currentInventorySlot.TransformPoint(Vector3.zero), deltaTime * num);
twin.transform.position = item.transform.position;
```
- Предмет лерпится к `slot.TransformPoint(slotLocalPos)` → **мировая позиция слота**
- Twin **копирует visual**
- Scale сжимается: `PlayerNeedsUI.instance.transform.localScale * 0.2f * item.inventoryScale`

**Если slot-local position где-то далеко (UI-панель в сцене под игроком, при FloatingOrigin shift) — предмет и twin **прыгают к слоту**. Но слот движется вместе с игроком, поэтому обычно не на сотни метров.**

**Критичный момент:** `inventoryEnterTimer = 0.3f` — в первые 0.3с `num = 14f` (быстрый лerp), потом `num = 99999f` (мгновенный snap). При выходе из инвентаря — `ExitInventorySlot()` просто сбрасывает `currentInventorySlot = null`, коллайдер visual включается, layer → 2.

### EnterBox / ExitBox / GetCurrentBox

```csharp
public void EnterBox(Transform box, Vector3 localPos, Quaternion localRot) {
    currentBox = box;
    boxLocalPos = localPos;
    boxLocalRot = localRot;
}
```

**Где физически `box`?** `box` = transform ящика (CrateInventory) — это GO ящика в мире. Ящик — `ShipItemCrate`, который **двигается по физике** (twin ящика — физическое тело). 

**В `ItemRigidbody.FixedUpdate` (в ящике, held):**
```csharp
if (currentBox && held) {
    boxLocalPos = currentBox.InverseTransformPoint(item.transform.position);
    boxLocalRot = Quaternion.Inverse(currentBox.rotation) * twin.rotation;
}
```
- Если предмет в руках И в ящике — **boxLocalPos обновляется** из текущей позиции (ячейка «подстраивается» под движение предмета в руке)

**В ящике, НЕ held:**
```csharp
item.transform.position = currentBox.TransformPoint(boxLocalPos);
twin.transform.position = item.transform.position;
item.transform.rotation = currentBox.rotation * boxLocalRot;
twin.transform.rotation = item.transform.rotation;
```
- Предмет **snap к позиции в ящике** (в мировых координатах ящика)
- Если ящик на лодке (onBoat) — позиция ящика = лодка. Ящик — ребёнок лодки
- Twin **копирует visual** → прыгает к ящику

**Может ли `box` быть на сотни метров?** Да, если ящик — на лодке, а лодка сдвинулась при FloatingOrigin shift, а предмет ещё не обновился. Но в нормальном flow это не происходит — `FixedUpdate` каждый кадр синхронизирует.

### EnterBoat / ExitBoat

```csharp
public void EnterBoat() {
    twin.parent = item.currentWalkCol;               // РЕПАРЕНТ twin к walkCol лодки!
    MoveRigidbodyToWalkCol();
    item.currentActualBoat.parent.GetComponent<BoatMass>().AddItem(this);
    onBoat = true;
}
```

**`currentWalkCol`** — Transform walk-коллайдера лодки (BoatEmbarkCollider.walkCollider). Это реальный объект **под лодкой** в сцене. 

**`MoveRigidbodyToWalkCol`:**
```csharp
Vector3 val = item.currentActualBoat.InverseTransformPoint(item.transform.position);
Quaternion val2 = Quaternion.Inverse(item.currentActualBoat.rotation) * twin.rotation;
twin.position = item.currentWalkCol.TransformPoint(val);
twin.rotation = item.currentWalkCol.rotation * val2;
```
- Берёт позицию предмета **в локальных координатах лодки** (`currentActualBoat`)
- Переводит в мировые через **walkCol** (`currentWalkCol.TransformPoint`)
- **Если `currentActualBoat` и `currentWalkCol` не совпадают** (разные трансформы лодки) — позиция может исказиться!

**`ExitBoat`:**
```csharp
twin.parent = world;  // FloatingOriginManager.instance.transform
twin.position = item.transform.position;
twin.rotation = item.transform.rotation;
onBoat = false;
```
- Twin **прыгает к позиции visual**, репарентится к world

**`MoveItemToWalkColRigidbody`** (для НЕ-held предмета на лодке):
```csharp
Vector3 val = item.currentWalkCol.InverseTransformPoint(twin.position);
Quaternion val2 = Quaternion.Inverse(item.currentWalkCol.rotation) * twin.rotation;
item.transform.position = item.currentActualBoat.TransformPoint(val);
item.transform.rotation = item.currentActualBoat.rotation * val2;
```
- **twin → local → walkCol → world через actualBoat → visual**
- Visual ставится по позиции twin (master = twin)

**Может ли walkCol указывать на лодку в сотнях метров?** walkCol — это дочерний Transform текущей лодки игрока. Лодка двигается с FloatingOrigin (под `FloatingOriginManager.instance.transform`). Если ссылка на walkCol устареет — нет, это живой Transform в сцене.

### Поток покупки в магазине

**Классы:** `ShopItemSpawner` (спавнер в магазине), `Shopkeeper` (продавец), `BuyItemUI` (UI покупки).

**`ShopItemSpawner.SpawnItem()`:**
```csharp
item = Object.Instantiate<GameObject>(itemPrefab, transform.position, transform.rotation).GetComponent<ShipItem>();
item.transform.parent = transform;  // ребёнок спавнера
```

- Создаётся **NEW ShipItem** (Instantiate) на позиции спавнера
- Предмет **не sold** (магазинный) → `!sold` → кинематика в `ItemRigidbody.FixedUpdate`
- Twin создаётся в корутине `LoadAfterDelay()` — **на позиции visual** (спавнера в магазине)

**Покупка (`Shopkeeper.SellItem` → `ShipItem.Sell()`):**
```csharp
public void Sell() {
    sold = true;
    transform.parent = FloatingOriginManager.instance.transform;
    UpdateLookText();
    saveable.RegisterToSave();
    if (pointedAtBy != null) {
        pointedAtBy.PickUpItem(this);  // подбор сразу после покупки!
    }
    OnBuy();
}
```

- `sold = true` → предмет перестаёт быть кинематическим от `!sold`
- Репарентится к world (`FloatingOriginManager.instance.transform`)
- **Подбирается сразу** (`PickUpItem`) — переходит в руки
- Twin: в момент `Sell()`, `held` устанавливается → в следующем `FixedUpdate` twin.position = visual.position (visual уже в руках pointer)

**Важно:** НЕТ нового Instantiate при покупке! Предмет был создан `ShopItemSpawner`, и `Sell()` просто меняет `sold = true` + подбирает. Twin **не пересоздаётся**.

### Стриминг/деспавн предметов (по дистанции)

В `ItemRigidbody.FixedUpdate`:
```csharp
if (distanceCheckTimer <= 0f) {
    if (Vector3.Distance(Camera.main.transform.position, item.transform.position) > 600f) {
        outOfRange = true;
    } else {
        outOfRange = false;
    }
    distanceCheckTimer = Random.Range(5f, 8f);
}
```

- Проверка раз в **5–8 секунд**, не каждый кадр
- `outOfRange = true` → `flag2 = true` (кинематика) + `framesUntilDestroy` для sold-предметов не на лодке
- **Уничтожение:** sold-предмет вне лодки + вне зоны → `framesUntilDestroy` > 10 → `item.DestroyItem()` (GameObject.Destroy)
- **НЕ sold-предметы** (магазинные) → `framesUntilDestroy = 0` (не уничтожаются)

**Twin при outOfRange:**
- `FixedUpdate` делает ранний выход (`return` after flag check) → twin **не управляется**, позиция frozen
- При возврате в зону (`outOfRange = false`) → `fixedFramesSinceSpawn` не сбрасывается, но kinematic flag снимается при sold + free

**Нет отдельного стриминг/деспавн менеджера для предметов.** Всё внутри `ItemRigidbody.FixedUpdate`.

### WorldItemSpawner.FreezeItem

```csharp
private IEnumerator FreezeItem() {
    yield return new WaitForEndOfFrame();
    item.GetItemRigidbody().debugForceKinematic = true;
}
```

- Вызывается после `SpawnItem()` — через 1 WaitForEndOfFrame
- **Замораживает twin** (`debugForceKinematic = true`)
- Разморозка: `WorldItemSpawner.Update()` — когда предмет подбирается (`held != null`) → `debugForceKinematic = false`

## A4. Floating Origin

### Точный механизм FloatingOriginManager

**Shift:** если позиция `shifterObject` выходит за `shiftDistance` → сдвиг всех детей `FloatingOriginManager.instance.transform` на `shiftDistance * (x,z)`.

**`NewShift(shiftVector)` корутина:**
```csharp
MoveOriginOcean(-shiftVector);  // сдвиг Crest/OceanRenderer origin
foreach (Transform item in transform) {
    item.Translate(shiftVector, Space.World);  // сдвиг ВСЕХ детей
}
// + PrepareForShifting/RestoreMomentum для ShiftingRigidbody (лодки)
outCurrentOffset += new Vector3(shiftVector.x, 0, shiftVector.z);
```

- `item.Translate(shiftVector, Space.World)` — **двигает каждый ребёнок** `FloatingOriginManager.instance.transform`
- Twin предмета — ребёнок `FloatingOriginManager.instance.transform` (parent = world) → **двигается вместе со сдвигом**
- **Visual предмета НЕ двигается** (parent = currentActualBoat или world, но не обязательно ребёнок FloatingOriginManager). На лодке: visual — ребёнок лодки. В мире: visual.parent = world = FloatingOriginManager.transform → тоже двигается.

**Может ли twin пропустить сдвиг или сдвинуться дважды?**

- Twin.parent = `FloatingOriginManager.instance.transform` (всегда, когда предмет не на лодке)
- `NewShift` делает `foreach (Transform item in transform) { item.Translate(shiftVector) }` — это **рекурсивно двигает** детей (Unity Translate двигает сам объект, а не его детей). Дети twin'а (коллайдеры, floater) — на самом twin, они двигаются как часть twin-GameObject.
- `ShiftingRigidbody` (лодка) — `PrepareForShifting` + `RestoreMomentum` — отдельно. Twin предмета на лодке: parent = `currentWalkCol` (ребёнок лодки), сдвигается **через лодку**.
- **Двойной сдвиг:** если twin.parent = world (FloatingOriginManager) и twin **также** записан в `ShiftingRigidbody` → может сдвинуться дважды. Но twin предмета НЕ имеет `ShiftingRigidbody` — только лодка. → **Нет двойного сдвига для twin.**

**`ShiftingPosToRealPos(pos)` = `pos - outCurrentOffset`** — из scene → real.  
**`RealPosToShiftingPos(pos)` = `pos + outCurrentOffset`** — из real → scene.  
Сдвиг происходит **одновременно** для всех детей: ocean + world + лодки (через ShiftingRigidbody).

## A5. Полное тело ItemRigidbody.FixedUpdate (дословно)

```csharp
private void FixedUpdate() {
    if (Debugger.instance.disableItemRigidbodyUpdate) { enabled = false; }
    
    fixedFramesSinceSpawn += 1f;
    bool flag = false;                    // "живой" флаг
    
    // --- Дистанция / culling ---
    if (distanceCheckTimer <= 0f) {
        outOfRange = Vector3.Distance(Camera.main.position, item.position) > 600f;
        distanceCheckTimer = Random.Range(5f, 8f);
    } else { distanceCheckTimer -= deltaTime; }
    
    if (outOfRange && !GameState.recovering) {
        if (item.sold && item.currentWalkCol == null) {
            framesUntilDestroy++;
            if (framesUntilDestroy > 10 && item.gameObject.layer != 26) {
                item.DestroyItem(); return;
            }
        } else { framesUntilDestroy = 0; }
    } else { flag = true; }    // в зоне → "живой"
    
    if (inStove) { flag = false; }
    
    // --- Disable col ---
    if (disableCol) { ToggleCollider(false); }
    if (!flag) { return; }     // РАННИЙ ВЫХОД (outOfRange + inStove)
    
    // --- Коллайдеры ---
    if (currentInventorySlot) { ToggleCollider(false); }
    else { ToggleCollider(true); }   // ← floater.enabled = false ВСЕГДА
    
    // --- Позиционирование ---
    if (!currentInventorySlot) {
        if (item.currentWalkCol != null && onBoat) {
            if (held) { MoveRigidbodyToWalkCol(); }
            else { MoveItemToWalkColRigidbody(); }
        } else if (held || !sold) {
            // VISUAL → TWIN (master = visual)
            twin.position = visual.position;
            twin.rotation = visual.rotation;
        } else {
            // TWIN → VISUAL (master = twin, sold + free)
            visual.position = twin.position;
            visual.rotation = twin.rotation;
        }
    }
    
    // --- Ящик ---
    if (currentBox) {
        if (held) {
            boxLocalPos = currentBox.InverseTransformPoint(visual.position);
            boxLocalRot = Quaternion.Inverse(currentBox.rotation) * twin.rotation;
        } else {
            visual.position = currentBox.TransformPoint(boxLocalPos);
            twin.position = visual.position;
            visual.rotation = currentBox.rotation * boxLocalRot;
            twin.rotation = visual.rotation;
        }
    }
    
    // --- dynamicColTimer ---
    if (held) { SetDynamicColTimer(); }  // reset to 6
    if (!held && dynamicColTimer > 0f && !isKinematic) {
        rigidbody.collisionDetectionMode = Continuous;
        dynamicColTimer -= deltaTime;
    }
    if (dynamicColTimer <= 0f && meshCol == null) {
        rigidbody.collisionDetectionMode = ContinuousDynamic;
    }
    
    // --- Kinematic flag ---
    bool flag2 = (held != null);          // held → kinematic
    
    // isTrigger на коллайдерах = held
    boxCol.isTrigger = held;
    meshCol.isTrigger = held;
    capsuleCol.isTrigger = held;
    SetSubcollidersTriggers(held);
    
    // meshCol sleeping → kinematic
    if (meshCol && rigidbody.IsSleeping() && dynamicColTimer <= 0f) { flag2 = true; }
    
    // GameState гейты → kinematic
    if (!playing || recovering || sleeping || inBed || currentShipyard) { flag2 = true; }
    
    // box/inventory → kinematic
    if (currentBox || currentInventorySlot) { flag2 = true; }
    
    // attached → kinematic
    if (attached) { flag2 = true; }
    
    // !sold → kinematic
    if (!item.sold) { flag2 = true; }
    
    // nailed → kinematic
    if (item.nailed) { flag2 = true; }
    
    // first 6 frames → kinematic
    if (fixedFramesSinceSpawn < 6f) { flag2 = true; }
    
    // outOfRange → kinematic
    if (outOfRange) { flag2 = true; }
    
    // debugger → kinematic
    if (Debugger.kinematicItemsTimer > 0f) { flag2 = true; }
    if (Debugger.debugForceKinematicBoat) { flag2 = true; }
    if (debugForceKinematic) { flag2 = true; }
    
    // apply
    if (isKinematic != flag2) { rigidbody.isKinematic = flag2; }
    
    if (Debugger.instance.disableItemRigidbodyCols) { ToggleCollider(false); }
}
```

**Полное тело `ItemRigidbody.LateUpdate`:**
```csharp
void LateUpdate() {
    if (currentInventorySlot) {
        float num = 99999f;
        if (inventoryEnterTimer > 0f) { num = 14f; inventoryEnterTimer -= deltaTime; }
        Vector3 val = currentInventorySlot.TransformPoint(slotLocalPos);
        item.position = Vector3.Lerp(val, currentInventorySlot.TransformPoint(Vector3.zero), deltaTime * num);
        twin.position = item.position;
        slotLocalPos = currentInventorySlot.InverseTransformPoint(item.position);
        item.rotation = Quaternion.Lerp(item.rotation, currentInventorySlot.rotation * Euler(invRotX, invRot, 0), deltaTime * num);
        twin.rotation = item.rotation;
        twin.localScale = Vector3.Lerp(twin.localScale, PlayerNeedsUI.instance.transform.localScale * 0.2f * item.inventoryScale, deltaTime * num);
        item.localScale = twin.localScale;
    } else if (!inStove) {
        twin.localScale = Vector3.one;
        item.localScale = Vector3.one;
    }
}
```

## Sequence: pickup → hold → drop (все кадры)

```
[FixedUpdate N]   GoPointer: raycast, запоминает pointedAtButton
[LateUpdate N]    GoPointer: MainButtonDown → PickUpItem(item)
                    item.held = this (GoPointer)
                    item.OnPickup() → wallAttachment: attached=false, exitInventorySlot
[FixedUpdate N+1] ItemRigidbody: held=true → twin.position = visual.position (копирует СТАРУЮ позицию!)
                   flag2=true → isKinematic=true → twin кинематический
                   ToggleCollider(true) → floater OFF
                   SetDynamicColTimer() → dynamicColTimer=6
[LateUpdate N+1]  GoPointer: двигает visual к позиции руки (Lerp(timerAfterPickup))
                   visual → новая позиция руки
[FixedUpdate N+2] twin.position = visual.position → twin на позиции руки
...
[LateUpdate]      GoPointer: MainButtonUp + throw → DropItem()
                    heldItem.held = null; heldItem = null;
                    item.OnDrop() → sold? visual → свободен; wallAttachment? → twin.position = attachPos, attached=true
[FixedUpdate]     ItemRigidbody: held=null → flag2 убирает held-кинематику
                   если sold && свободен → flag2=false → динамический
                   twin → visual: visual.position = twin.position (twin = master)
                   ThrowItemAfterDelay: AddForce на twin
```

## Вывод для моддера (баг twin на ~462м)

**Ключевое наблюдение:** twin в руках **всегда** ставится = visual.position в `FixedUpdate`. Если мод ставит twin.position через `AddForce` → в `FixedUpdate` twin.position **переписывается** = visual.position (held = master = visual). Сила теряется.

**Баг twin на 462м при живом визуале:**  
Наиболее вероятная причина — `EnterBoat()`. Когда предмет подбирается на лодке, `MoveRigidbodyToWalkCol()` использует `currentActualBoat.InverseTransformPoint` и `currentWalkCol.TransformPoint`. Если `currentWalkCol` и `currentActualBoat` — разные GO (walkCol = дочерний Transform, actualBoat = корень лодки), и **лодка сдвинулась при FloatingOrigin shift** между кадрами — координатное преобразование может дать позицию **сотни метров от реальной**.

Также: `EnterInventorySlot` — если `currentInventorySlot.TransformPoint(slotLocalPos)` даёт позицию UI-панели (PlayerNeedsUI), которая при сдвиге могла быть **на старой позиции** → twin прыгает к UI.

**Для поиска бага:** проверить, не вызывается ли `EnterBoat` / `EnterInventorySlot` / `EnterBox` в момент shift, или не репарентится twin при shift в неподходящий момент (twin.parent = currentWalkCol на лодке, но shift двигает только детей FloatingOriginManager.transform, а twin на лодке двигается через лодку/ShiftingRigidbody).
