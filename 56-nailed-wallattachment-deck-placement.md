# 56. Прилепленность предметов к лодке: nailed, wallAttachment, палубная посадка

Разбор механизмов, которые «прикрепляют» предмет к лодке — `nailed`, `wallAttachment`, `HangableItem`, — и объяснение, почему предмет может лежать «на поверхности выше трюма» — ответ на запрос E6. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 51 (коллайдеры), 53 (Enter/ExitBoat), 16 (ShipItem/twin).

## `ShipItem.nailed` — «прибитый гвоздями» предмет

### Поле

| Поле | Тип | Содержание |
|------|-----|------------|
| `nailed` | `bool` | На `ShipItem` — предмет «прибит», treated как кинематический (не двигается). Устанавливается через `ShipItemHammer`. Сохраняется в `SavePrefabData.isNailed`. |

### Установка: `ShipItemHammer.NailItem(ShipItem item)` — вербатим

```csharp
private void NailItem(ShipItem item)
{
    if (item.big && !item.wallAttachment && !RaycastHitsFloor(item))
    {
        NotificationUi.instance.ShowNotification("Item must be on the floor.");
        return;
    }
    if (item.GetItemRigidbody().GetBody().velocity != Vector3.zero)
    {
        NotificationUi.instance.ShowNotification("Item is moving.");
        return;
    }
    item.nailed = true;
    UISoundPlayer.instance.PlayUISound(UISounds.winchUnclick, 1f, 0.6f);
    Debug.Log("Nailed " + item.name);
}
```

**Условия:** (1) предмет `sold` (продан/куплен); (2) если `big && !wallAttachment` — должен стоять на полу (raycast вниз 4 м, слой 8 = Player/HullPlayerCollider); (3) не двигается (`velocity == Vector3.zero`).

**Удаление:** `ShipItemHammer.OnAltActivate()` → если предмет уже `nailed` → `item.nailed = false` + звук.

### Кто может быть nailed: `ShipItemHammer.CanNail(ShipItem item)` — вербатим

```csharp
public static bool CanNail(ShipItem item)
{
    if (!item.sold) return false;
    if (item.big || ((object)item).GetType() == typeof(ShipItemHangable) || item.wallAttachment)
    {
        return true;
    }
    return false;
}
```

**Допускаются:** `big` предметы, `ShipItemHangable` (лампы), `wallAttachment` предметы (настенные). Мелкие обычные предметы — **нет**.

### Влияние `nailed` на ItemRigidbody — вербатим (FixedUpdate)

```csharp
// ItemRigidbody.FixedUpdate — ключевые строки
if (item.nailed)
{
    flag2 = true;  // flag2 → rigidbody.isKinematic = true
}
```

**`nailed = true` → `rigidbody.isKinematic = true` → twin перестаёт двигаться физикой.** Твин «заморожен» в текущей позиции. Коллайдеры twin при nailed не становятся trigger — они остаются **non-trigger**, но кинематический Rigidbody **не генерирует OnCollisionEnter** (для динамических тел столкновение с kinematic не fires, если kinematic не двигается).

> **Важно для мода:** если мод делает twin-коллайдер non-trigger и предмет `nailed`, twin = kinematic + non-trigger. Kinematic non-trigger коллайдер **сталкивается** с динамическими Rigidbody (выталкивает их), но **не получает** OnCollisionEnter. Это значит: nailed предмет **не наносит урон лодке** через BoatImpactSoundsCollider (CollisionEnter не fires), но **может выталкивать** динамические объекты (другие предметы, twin при холде мода).

### `nailed` в GoPointer/LookUI — ограничения взаимодействия

- `GoPointer.DoRaycast`: если предмет `nailed` → **не подбирается** (не входит в PickupableItem branch) — с оговоркой: `ShipItemCrate`, `ShipItemBottle`, `ShipItemBed` даже nailed можно «смотреть» (Look), но не подобрать.
- `GoPointer.MainButtonDown`: `!item.nailed` — условие для Pickup.
- `LookUI`: nailed предметы → **lookText = "Nailed"** (информация о прибитости), не показывают цену/ миссию.
- `CargoStorageUI`: nailed big-предметы **не загружаются** в cargo (`!component2.nailed`).

## `ShipItem.wallAttachment` — настенное прикрепление

### Поле и состояние

| Поле | Тип | Содержание |
|------|-----|------------|
| `wallAttachment` | `bool` | На `ShipItem` — предмет может крепиться к стене. При загрузке (`LoadAfterDelay`): если `wallAttachment` → `itemRigidbodyC.attached = true` (twin = kinematic, «прилеплен»). |
| `attachPos` | `Vector3` (private) | Точка прикрепления на стене — `RaycastHit.point` при raycast из twin вперёд. |
| `attachRot` | `Quaternion` (private) | Поворот прикрепления — `LookRotation(-hit.normal, ...)`. |
| `inRangeOfWall` | `bool` (private) | True, если raycast нашёл стену в пределах 0.1–1.3 м. |

### Обнаружение стены — `ShipItem.Update()` — вербатим

```csharp
// ShipItem.Update() — ветка wallAttachment (дословно)
if (Object.op_Implicit((Object)(object)held) && wallAttachment)
{
    Vector3 forward = itemRigidbody.forward;
    RaycastHit val = default(RaycastHit);
    if (Physics.Raycast(new Ray(itemRigidbody.position, forward), ref val, 1.3f))
    {
        if (((RaycastHit)(ref val)).distance < 0.1f)
        {
            inRangeOfWall = false;
        }
        else
        {
            attachPos = ((RaycastHit)(ref val)).point;
            attachRot = Quaternion.LookRotation(-((RaycastHit)(ref val)).normal, Vector3.up);
            if (((RaycastHit)(ref val)).normal.y > 0.8f)
            {
                attachRot = Quaternion.LookRotation(-((RaycastHit)(ref val)).normal, ((Component)this).transform.up);
            }
            inRangeOfWall = true;
            SetUpTargeter(attachPos, attachRot, ((Component)this).GetComponent<MeshFilter>().sharedMesh);
        }
    }
    else
    {
        inRangeOfWall = false;
    }
    if (!inRangeOfWall) { }
}
else
{
    inRangeOfWall = false;
}
```

**Механика:** при холде предмета (`held != null`) и `wallAttachment` — каждый `Update()` raycast из позиции twin (`itemRigidbody.position`) по направлению `itemRigidbody.forward` на 1.3 м. Если стена найдена (distance 0.1–1.3 м):

- `attachPos = hit.point` — точка контакта на стене.
- `attachRot`: если `normal.y > 0.8` (горизонтальная поверхность / потолок) → `LookRotation(-normal, item.transform.up)` (предмет «стоит» на нормали). иначе → `LookRotation(-normal, Vector3.up)` (предмет «прилеплен» к вертикальной стене, лицом к стене).
- `inRangeOfWall = true` → показывается `Targeter` (wireframe проекция предмета на стене).

**Критично:** raycast исходит из **twin позиции** (`itemRigidbody.position`), а не из visual. В ваниле twin позиция = visual позиция (ItemRigidbody.LateUpdate синхронизирует). В моде — twin позиция может отличаться (физика-поза) → raycast стены исходит из **физической позиции twin**, что может быть **внутри лодочного коллайдера** → raycast не видит стену (попадает в CapsuleCollider лодки) → `inRangeOfWall = false` → предмет **не прилепится** к стене при дропе.

### `SetUpTargeter` — визуальная проекция предмета на стене

```csharp
private void SetUpTargeter(Vector3 attachPos, Quaternion attachRot, Mesh mesh)
{
    Vector3 val = itemRigidbody.InverseTransformPoint(attachPos);
    Vector3 pos = ((Component)this).transform.TransformPoint(val);
    Quaternion val2 = Quaternion.Inverse(itemRigidbody.rotation) * attachRot;
    Quaternion rot = ((Component)this).transform.rotation * val2;
    held.GetTargeter().DisplayTargeter(pos, rot, mesh);
}
```

Отображает wireframe-сетку предмета в предполагаемой позиции/повороте на стене — «targeter» (подсказка мододелу, куда предмет прилепится).

### Drop с wallAttachment — `ShipItem.OnDrop()` — вербатим

```csharp
public override void OnDrop()
{
    if (!sold)
    {
        ReturnToShopPos();
    }
    else if (wallAttachment && inRangeOfWall && !forceDisableRedOutline)
    {
        ((Component)itemRigidbody).transform.position = attachPos;
        ((Component)itemRigidbody).transform.rotation = attachRot;
        ((Component)itemRigidbody).GetComponent<ItemRigidbody>().attached = true;
    }
}
```

**При дропе wallAttachment предмета:** если `sold && inRangeOfWall && !forceDisableRedOutline` → twin (ItemRigidbody GO) snap на `attachPos/attachRot` + `attached = true`.

**`attached = true`** → `ItemRigidbody.FixedUpdate` ставит `flag2 = true` → `rigidbody.isKinematic = true` → twin **заморожен** в позиции/повороте стены. Twin не двигается физикой.

**Важно:** при дропе **twin позиция** ставится на `attachPos`, а **visual позиция** — не меняется (GoPointer.DropItem ставит visual.layer=0, held=null, но не двигает visual). Синхронизация visual↔twin: `ItemRigidbody.LateUpdate` — при `attached=true`, twin kinematic → twin позиция статична, visual **не синхронизируется** с twin (наоборот, при attached twin позиция «главная»).

> **Критично для мода:** при wallAttachment дропе, twin snap на стену → twin позиция совпадает с поверхностью стены (не CapsuleCollider лодки). Visual потом синхронизируется с twin через LateUpdate. Но **если twin уже внутри CapsuleCollider лодки** (мод с твёрдым коллайдером при холде), raycast из twin может не найти стену (raycast внутри CapsuleCollider → луч не выходит), и `inRangeOfWall = false` → предмет **не прилепится**, просто дропнется как обычный → twin free-falls → столкновения с CapsuleCollider → всё баг-сценарии из заметок 51–55.

### Pickup с wallAttachment — `ShipItem.OnPickup()` — вербатим

```csharp
public override void OnPickup()
{
    if (wallAttachment)
    {
        itemRigidbodyC.attached = false;
    }
    // ... inventory slot withdrawal ...
    overrideEnableOutline = false;
}
```

**При подборе:** `attached = false` → twin перестаёт быть kinematic → может двигаться физикой. В моде: twin снова может столкнуться с CapsuleCollider лодки.

### Начальное состояние wallAttachment при загрузке

```csharp
// ShipItem.LoadAfterDelay()
if (wallAttachment)
{
    GetItemRigidbody().attached = true;
}
```

**При загрузке из save:** wallAttachment предметы сразу `attached = true` → kinematic → «прилеплены» в позиции из save (не двигаются при загрузке). Если save корректный, twin позиция = стена.

## `ItemRigidbody.attached` — кинематический замок twin

### Поле

| Поле | Тип | Содержание |
|------|-----|------------|
| `attached` | `bool` | На `ItemRigidbody` — twin «прилеплен» (wallAttachment/HangableItem). При `true` → `rigidbody.isKinematic = true` (twin frozen). |

### Влияние в FixedUpdate — вербатим

```csharp
if (attached)
{
    flag2 = true;   // → isKinematic = true
}
```

### Связанные поля `ItemRigidbody` для «прилепленности»

| Поле | Тип | Содержание |
|------|-----|------------|
| `attached` | `bool` | Kinematic lock (wallAttachment/HangableItem) |
| `disableCol` | `bool` | Disable twin colliders (HangableItem: при hook → disableCol=true) |
| `inStove` | `bool` | Item «в плите/крюке» (HangableItem: inStove=true при hook) |

`disableCol = true` → `ToggleCollider(state: false)` — **twin коллайдеры полностью отключаются** (enabled = false). Это для HangableItem: лампы на крюках не имеют физических коллайдеров twin.

`inStove = true` → влияет на buoyancy/visibility в LateUpdate (twin позиция = hook позиция, не в воде).

### `attached` блокирует ExitBoat

```csharp
// ShipItem.ExtraFixedUpdate
else if (!currentlyStayedEmbarkCol && currentActualBoat && frameCounter > num
         && !GameState.sleeping && !disallowDisembarking && !itemRigidbodyC.attached)
{
    ExitBoat();
}
```

**`attached = true` → ExitBoat НЕ вызывается** — предмет «прилеплен» к лодке (wall/Hook) → не может сам выйти из EmbarkCol.

## `HangableItem` — подвешивание на крюк (лампы)

### Компонент и поля

`HangableItem : MonoBehaviour` — `[RequireComponent(typeof(ShipItem))]`, отдельный компонент для hook-подвешиваемых предметов (лампы, светильники).

| Поле | Тип | Содержание |
|------|-----|------------|
| `shipItem` | `ShipItem` | приватная ссылка на ShipItem |
| `itemRigidbody` | `Rigidbody` | Rigidbody twin |
| `itemRigidbodyC` | `ItemRigidbody` | twin ItemRigidbody component |
| `currentHook` | `Collider` | текущий крюк (ShipItemLampHook) |
| `disallowHangingOnTrigger` | `bool` | блокировка автоматического подвешивания (при выходе из inventory) |
| `lockX` | `bool` [SerializeField] | блокировка X rotation при подвешивании |
| `lockZ` | `bool` [SerializeField] | блокировка Z rotation при подвешивании |
| `rotX` | `float` [SerializeField] | фиксированный X угол при подвешивании |
| `rotZ` | `float` [SerializeField] | фиксированный Z угол при подвешивании |
| `framesAfterAwake` | `float` | счётчик frames — OnTriggerEnter только после ≥3 frames |

### `OnTriggerEnter(Collider other)` — вербатим

```csharp
public void OnTriggerEnter(Collider other)
{
    if (!(framesAfterAwake >= 3f) && ((Component)this).GetComponent<SaveablePrefab>().currentCrateId <= 0
        && !Object.op_Implicit((Object)(object)shipItem.held) 
        && shipItem.GetCurrentInventorySlot() == -1 
        && !disallowHangingOnTrigger 
        && ((Component)other).CompareTag("Hook"))
    {
        Debug.Log(shipItem.name + ".HangableItem: Connecting joint from trigger enter.");
        ConnectJoint(other);
    }
}
```

**Условия:** ≥3 frames after Awake, not in crate, not held, not in inventory, not disallowHanging, other.CompareTag("Hook"). → Auto-hang on trigger enter.

### `ConnectJoint(Collider hook)` — вербатим

```csharp
public void ConnectJoint(Collider hook)
{
    currentHook = hook;
    ConfigurableJoint val = ((Component)currentHook).GetComponent<ShipItemLampHook>().CreateJoint();
    if (Object.op_Implicit((Object)(object)val))
    {
        if ((Object)(object)itemRigidbody == (Object)null)
        {
            itemRigidbody = ((Component)shipItem.GetItemRigidbody()).GetComponent<Rigidbody>();
            itemRigidbodyC = shipItem.GetItemRigidbody();
        }
        ((Joint)val).connectedBody = itemRigidbody;
        itemRigidbody.ResetCenterOfMass();
        itemRigidbodyC.attached = true;
        itemRigidbodyC.disableCol = true;
        itemRigidbodyC.inStove = true;
        itemRigidbodyC.ForceRigidbodyToWalkCol();
        shipItem.ToggleDisallowDisembarking(newState: true);
        Debug.Log("Connected joint.");
    }
}
```

**Последствия ConnectJoint:**
- `ConfigurableJoint` на twin GO (создан `ShipItemLampHook.CreateJoint` → копия joint с hook) → `connectedBody = itemRigidbody` → twin физически соединён с hook через joint.
- `attached = true` → twin kinematic (frozen).
- `disableCol = true` → twin colliders disabled (не участвуют в физике).
- `inStove = true` → twin not in water, visible = hook position.
- `ForceRigidbodyToWalkCol()` → twin snap к walkCol лодки (если на лодке).
- `ToggleDisallowDisembarking(true)` → предмет **не может** ExitBoat.

> **HangableItem при hook:** twin = kinematic + colliders disabled + ConfigurableJoint → **физически не участвует** в столкновениях. twin «заморожен» и «прилеплен» через joint к hook. visual позиция синхронизируется с hook в LateUpdate.

### `DisconnectJoint()` — вербатим

```csharp
public void DisconnectJoint()
{
    if (!((Object)(object)currentHook == (Object)null))
    {
        ((Component)currentHook).GetComponent<ShipItemLampHook>().RemoveJoint();
        currentHook = null;
        itemRigidbodyC.attached = false;
        itemRigidbodyC.disableCol = false;
        itemRigidbodyC.inStove = false;
        shipItem.ToggleDisallowDisembarking(newState: true);
        Debug.Log("Disconnected joint.");
    }
}
```

**Последствия DisconnectJoint:**
- `RemoveJoint()` → Destroy ConfigurableJoint на twin GO.
- `attached = false` → twin перестаёт быть kinematic → может двигаться.
- `disableCol = false` → twin colliders re-enabled → физически активен.
- `inStove = false` → twin buoyancy/visibility нормальные.
- `ToggleDisallowDisembarking(true)` → **странно:** при disconnect тоже блокируется disembarking → предмет остаётся «на лодке» до следующего события.

**DisconnectJoint вызывается из:**
- `ShipItem.ExitBoat()` → если `held == null && HangableItem != null` → DisconnectJoint.
- `ShipItemLampHook.OnPickup()` → disconnect при поднятии hook.
- `ShipItemLampHook.OnEnterInventory()` → disconnect при hook в инвентаре.

### `LateUpdate` — позиция visual = hook + offset

```csharp
private void LateUpdate()
{
    if (Object.op_Implicit((Object)(object)currentHook))
    {
        ((Component)shipItem).transform.position = ((Component)currentHook).transform.position 
            + ((Component)currentHook).transform.forward * -0.128f;
        Vector3 eulerAngles = ((Component)this).transform.eulerAngles;
        if (lockX) { eulerAngles.x = rotX; }
        if (lockZ) { eulerAngles.z = rotZ; }
        ((Component)this).transform.eulerAngles = eulerAngles;
    }
}
```

**Visual позиция = hook позиция - 0.128 м по hook.forward + фиксированные X/Z rotation.** Это даёт лампам «свисание» с крюка.

## `ShipItemLampHook` — крюк для ламп

`ShipItemLampHook : ShipItem` — hook-предмет на лодке (настенный крюк для лампы).

### `CreateJoint()` / `RemoveJoint()` — вербатим

```csharp
public ConfigurableJoint CreateJoint()
{
    if (occupied) { return null; }
    ConfigurableJoint component = ((Component)this).GetComponent<ConfigurableJoint>();
    ConfigurableJoint result = CopyJoint(component, ((Component)itemRigidbody).gameObject);
    occupied = true;
    return result;
}

public void RemoveJoint()
{
    Object.Destroy((Object)(object)((Component)itemRigidbody).gameObject.GetComponent<ConfigurableJoint>());
    occupied = false;
}
```

**`CopyJoint(ConfigurableJoint original, GameObject targetObject):`** — копирует все motion/limit настройки с original ConfigurableJoint на новый joint на twin GO. connectedBody устанавливается в `ConnectJoint`.

**`occupied` = bool:** крюк занят лампой. При occupied → `OnItemClick`拒绝了 (return false). При occupied → pickup/inventory → disconnect lamp first.

## Палубная посадка обычного предмета (без wallAttachment/nailed)

### Что происходит при дропе обычного предмета на лодке

1. `GoPointer.LateUpdate` → `MainButtonUp` → если `collisions <= 0 || allowObstructedDropping || throwButtonUp` → `heldItem.OnDrop()` + `GoPointer.DropItem()`.
2. `ShipItem.OnDrop()` — **не делает ничего** для обычного предмета (не wallAttachment): `if (!sold) → ReturnToShopPos()` или **пусто** (нет wallAttachment branch).
3. `GoPointer.DropItem()` → `heldItem.held = null; heldItem.gameObject.layer = 0` → предмет становится «свободным».
4. Twin (ItemRigidbody) — **свободный** (not kinematic, not attached, not nailed) → **физика берёт управление twin**.
5. Twin падает → `OnTriggerEnter(EmbarkCol)` → `ShipItem.ExtraFixedUpdate` → `frameCounter > 1` → `EnterBoat()` → twin репарентится на `walkCollider` → twin позиция snap к walkCol.
6. Twin на walkCol (слой 12) → **walkCol — поверхность палубы (слой 12, MeshCollider)**. Twin физически «стоит» на палубе (walkCol mesh = форма трюма/палубы).

**В ваниле:** twin = trigger коллайдеры → twin **не сталкивается** с CapsuleCollider лодки → twin falls, EnterBoat, reparent to walkCol → twin trigger-коллайдер «прозрачный» для физики → visual/twin на палубе.

**В моде (non-trigger twin):** twin сталкивается с CapsuleCollider лодки → twin «ложится» на поверхность CapsuleCollider (выше палубы) → twin ещё внутри EmbarkCol trigger → EnterBoat → twin reparent на walkCol → twin snap к walkCol позиции → **но twin физически внутри CapsuleCollider** → OnCollisionEnter → BoatImpactSoundsCollider → BoatDamage.Impact (twin Untagged) → урон лодке.

> **Почему предмет «на поверхности объёма выше трюма»:** ванильный twin (isTrigger=true) = «проходит через CapsuleCollider» и садится на walkCol (палуба, слой 12). Модный twin (non-trigger) = **не может пройти через CapsuleCollider** → twin «стоит» на поверхности CapsuleCollider → это **выше палубы** (CapsuleCollider поверхность = верх капсулы, охватывающей корпус).

## Sequence diagram: дроп предмета на лодке (ванила vs мод)

```
=== ВАНИЛА (twin isTrigger=true) ===
Player.DropItem()
  → heldItem.held = null
  → twin: isTrigger=true, Rigidbody dynamic
  → twin falls through CapsuleCollider (isTrigger → no physical collision)
  → twin OnTriggerEnter(EmbarkCol) → ShipItem.EnterBoat()
  → twin reparent to walkCollider (слой 12)
  → twin snap to walkCol position = палуба
  → twin stands on walkCol mesh (слой 12) ← настоящая палуба
  → twin isTrigger → no further collision issues

=== МОД (twin non-trigger) ===
Player.DropItem()
  → heldItem.held = null
  → twin: isTrigger=false, Rigidbody dynamic
  → twin falls → HITS CapsuleCollider surface (above deck)
  → twin «стоит на воздухе» над палубой
  → twin OnTriggerEnter(EmbarkCol) → ShipItem.EnterBoat()
  → twin reparent to walkCollider → snap к walkCol position
  → BUT: twin non-trigger → OnCollisionEnter с BoatImpactSoundsCollider
  → BoatImpactSounds.Impact → BoatDamage.Impact (twin Untagged → не фильтруется!)
  → hullDamage += 0.15 каждые 1 с при контакте
  → ПРУЖИНА мода может тянуть twin внутрь → physics push-out → OnCollisionEnter каждый step
  → Флап Enter/ExitBoat (если twin выходит из EmbarkCol trigger)
```

## Практические выводы для мододела

1. **`nailed = true` → twin kinematic** → twin не двигается, не получает OnCollisionEnter → **не наносит урон лодке**, не флапает. Но **может выталкивать динамические тела** (kinematic non-trigger vs dynamic).
2. **`wallAttachment`** — механизм «прилепленности» к стене/полу через raycast + snap + `attached=true`. В ваниле работает корректно (twin isTrigger → raycast видит стену). В моде: если twin внутри CapsuleCollider → raycast не видит стену → wallAttachment **не срабатывает** при дропе → предмет падает как обычный → баг-сценарии 51–55.
3. **`HangableItem`** — лампы на крюках: twin kinematic + colliders disabled + ConfigurableJoint → **полностью безопасно** для мода (twin не участвует в физике при hook).
4. **`attached = true`** блокирует ExitBoat → предмет «прилеплен» к лодке, не может случайно выйти. Но при `attached = false` (подбор) → twin снова dynamic → столкновения.
5. **Обычный предмет на палубе:** ванила — twin isTrigger → «прозрачный» для CapsuleCollider → садится на walkCol (палуба, слой 12). Мод — twin non-trigger → «стоит на CapsuleCollider surface» → выше палубы → урон лодке через BoatDamage.Impact.
6. **Решение для мода:** при холде предмета — `Physics.IgnoreLayerCollision(twinLayer, boatLayer)` → twin не сталкивается с CapsuleCollider → twin falls through → EnterBoat → twin на walkCol (как ванила). При дропе — восстановить collision. Альтернатива: при дропе мода → временно сделать twin isTrigger=true → twin проходит CapsuleCollider → EnterBoat → walkCol → затем twin normal.
