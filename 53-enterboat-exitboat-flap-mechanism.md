# 53. EnterBoat/ExitBoat: механика вызовов и потенциал флапа

Разбор того, кто и когда вызывает `EnterBoat()`/`ExitBoat()` для предметов, и может ли происходить «флап» — ответ на запрос E3. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 51 (коллайдеры), 52 (слои), 16 (ShipItem/twin).

## Кто вызывает EnterBoat/ExitBoat для предметов

### `ShipItem.ExtraFixedUpdate` — главный драйвер

Вызывается каждый `FixedUpdate` (через `PickupableItem.ExtraFixedUpdate` → `ShipItem.ExtraFixedUpdate`):

```csharp
// ShipItem.cs, ExtraFixedUpdate — дословно
public override void ExtraFixedUpdate()
{
    if (Debugger.instance.disableItemUpdate) return;
    int num = 1;                          // порог frameCounter
    if (GameState.currentlyLoading || GameState.justStarted) num = 0;

    // Если УЖЕ на лодке — проверяем, нужно ли ВЫЙТИ
    if (currentActualBoat != null)
    {
        // Счётчик тикает, если мы не в нужном EmbarkCol
        if (!GameState.sleeping && gameObject.layer != 26 
            && (currentlyStayedEmbarkCol == null || currentlyStayedEmbarkCol != currentBoatCollider))
        {
            frameCounter++;               // тик к выходу
        }
        else
        {
            frameCounter = 0;              // мы в правильном EmbarkCol — сброс
        }
    }

    // Если НЕ на лодке — проверяем, нужно ли ВОЙТИ
    if (currentActualBoat == null)
    {
        if (currentlyStayedEmbarkCol != null)
        {
            frameCounter++;               // тик к входу
        }
        else
        {
            frameCounter = 0;              // нет EmbarkCol — сброс
        }
    }

    // Вход на лодку
    if (currentlyStayedEmbarkCol != null 
        && currentlyStayedEmbarkCol != currentBoatCollider 
        && frameCounter > num)
    {
        EnterBoat(currentlyStayedEmbarkCol);
    }
    // Выход с лодки
    else if (currentlyStayedEmbarkCol == null 
        && currentActualBoat != null 
        && frameCounter > num 
        && !GameState.sleeping 
        && !disallowDisembarking 
        && !itemRigidbodyC.attached)
    {
        ExitBoat();
    }
}
```

**Порог:** `frameCounter > 1` (при нормальном GameState) → смена состояния через **2+ fixed frames**. Это защита от мгновенного флапа, но **не от периодического**.

### Триггерные колбэки

`ShipItem.OnTriggerEnter(Collider other)`: если `other.CompareTag("EmbarkCol")` → `currentlyStayedEmbarkCol = other`.
`ShipItem.OnTriggerExit(Collider other)`: если `currentlyStayedEmbarkCol == other` → `currentlyStayedEmbarkCol = null`.

**Противоположный коллайдер:** `BoatEmbarkCollider` (тег `EmbarkCol`) на лодке. Это **trigger-коллайдер** (isTrigger=true).

## `ItemRigidbody.EnterBoat()` — что делает с twin

```csharp
// ItemRigidbody.cs — дословно
public void EnterBoat()
{
    transform.parent = item.currentWalkCol;          // twin → дочерний walkCollider
    MoveRigidbodyToWalkCol();                        // snap позиции twin к walkCol
    item.currentActualBoat.parent.GetComponent<BoatMass>().AddItem(this);
    onBoat = true;
}
```

### `MoveRigidbodyToWalkCol` — вербатим unavailable (ILSpy unresolved), но по контексту:
- twin позиция/rotation синхронизируется с walkCollider лодки.
- twin становится дочерним walkCollider — **репарент**.

## `ItemRigidbody.ExitBoat()` — что делает с twin

```csharp
// ItemRigidbody.cs — дословно
public void ExitBoat()
{
    if (item.currentActualBoat != null)
    {
        attached = false;
        transform.parent = world;                     // twin → дочерний FloatingOriginManager
        transform.position = item.transform.position; // twin snap к visual
        transform.rotation = item.transform.rotation; // twin snap к visual
        item.currentActualBoat.parent.GetComponent<BoatMass>().RemoveItem(this);
        onBoat = false;
    }
}
```

## `BoatMass.AddItem/RemoveItem` — идемпотентность

```csharp
// BoatMass.cs — дословно
public void AddItem(ItemRigidbody itemBody)
{
    if (!itemsOnBoat.Contains(itemBody))              // ← ИДЕМПОТЕНТНА
    {
        itemsOnBoat.Add(itemBody);
    }
}

public void RemoveItem(ItemRigidbody itemBody)
{
    itemsOnBoat.Remove(itemBody);                     // Remove для несуществующего → ничего не делает
}
```

**`AddItem` — идемпотентна:** повторный Add того же `ItemRigidbody` **не добавляется** (Contains check). `RemoveItem` для отсутствующего элемента — `List.Remove` вернет false, но не упадёт.

**Но:** `UpdateMass()` вызывается каждый `Update()` (для активной лодки) и **пересчитывает** всё:
- `itemsOnBoat.RemoveAll(item => item == null)` — чистит null-ссылки (если twin был уничтожен).
- Затем считает массу всех itemsOnBoat + player + sails.

> **NaN не возникает из повторного Add/Remove** — но если Add происходит без Remove (флап Enter без Exit), предмет **остаётся в itemsOnBoat** → его масса **учитывается дважды** только если Add проходит (но Contains блокирует). Если ExitBoat прошёл, RemoveItem удаляет. Если EnterBoat-EnterBoat без Exit — второй Add блокируется Contains. **Флап Enter→Exit→Enter→Exit** — безопасен для BoatMass.**

## Может ли Enter/Exit флапать каждый физик-кадр?

### Механика `currentlyStayedEmbarkCol`

`ShipItem.OnTriggerEnter(EmbarkCol)` ставит `currentlyStayedEmbarkCol = other` — и `ExtraFixedUpdate` начинает тикать `frameCounter`. Порог = 1 (нормально). То Enter/Exit требует **≥2 fixed frames** в trigger volume.

**Проблема мода:** при твёрдом twin-коллайдере предмет может **физически сталкиваться** с CapsuleCollider лодки → физика выталкивает twin → twin выходит из trigger volume (`OnTriggerExit`) → `currentlyStayedEmbarkCol = null` → ExitBoat → предмет репарентится в мир → пружина мода тянет обратно → twin входит в trigger volume → EnterBoat → twin репарентится на walkCollider → физика выталкивает → OnTriggerExit → ExitBoat… **Цикл!**

**Дедукция:** `OnTriggerEnter/Exit` для `EmbarkCol` — trigger-коллайдер. Если twin физически внутри CapsuleCollider, но trigger-коллайдер `EmbarkCol` — отдельный объём (другой GO, другая позиция). В зависимости от геометрии:
- trigger `EmbarkCol` может быть **больше/меньше/отдельно** от CapsuleCollider.
- twin может быть внутри trigger `EmbarkCol` (OnTriggerEnter) → EnterBoat → репарент на walkCol → позиция twin меняется → **physically overlaps CapsuleCollider** → но trigger-статус по `EmbarkCol` не меняется (OnTriggerExit не вызывается, если twin не вышел из trigger `EmbarkCol`).

**Однако:** если пружина мода двигает twin **наружу из EmbarkCol trigger volume**, `OnTriggerExit` fires → ExitBoat → `currentlyStayedEmbarkCol = null` → `frameCounter++` → через 2 frames → ExitBoat again (already done). И наоборот: пружина тянет внутрь → `OnTriggerEnter` → EnterBoat. Каждый цикл = 2 fixed frames минимально.

> **Потенциальный флап:** Enter/Exit может циклиться с периодом ≥2 fixed frames (~0.044 c при fixedDt=0.022). Это не «каждый кадр», но ~22 Hz флап → массовые AddItem/RemoveItem (идемпотентны, но **перерасчёт BoatMass.UpdateMass каждый Update** — 22 раза в секунду меняет mass/centerOfMass → нестабильная симуляция).

## `ShipItem.ExitBoat` (visual-side) — полные побочки

```csharp
// ShipItem.cs — дословно
private void ExitBoat()
{
    Debug.Log(name + ": ExitBoat");
    if (saveable != null && saveable.GetParentObject() != -3)
    {
        saveable.SetParentObject(-1);         // parent = мир
    }
    if (itemRigidbody != null)
    {
        itemRigidbody.GetComponent<ItemRigidbody>().ExitBoat();  // twin ExitBoat
        if (held == null && GetComponent<HangableItem>() != null)
        {
            GetComponent<HangableItem>().DisconnectJoint();     // отцепить от крюка
        }
    }
    currentWalkCol = null;
    currentActualBoat = null;
    currentBoatCollider = null;
    transform.parent = world;                         // visual → мир
}
```

Побочки ExitBoat: `saveable.SetParentObject(-1)`, twin ExitBoat (reparent+mass), HangableItem disconnect, clear walkCol/actualBoat/boatCollider, visual reparent to world.

## Практические выводы

1. **Enter/Exit gated через `frameCounter > 1`** — не мгновенный, но ≥22 Hz флап возможен при пружине, выталкивающей twin из EmbarkCol trigger volume.
2. **BoatMass.AddItem идемпотентен** (Contains check) — повторный Add того же twin не ломает массу. RemoveItem безопасен. **NaN от флапа не возникает**, но `UpdateMass()` вызывается каждый `Update` — при флапе mass/centerOfMass «моргает» (item входит/выходит → масса лодки oscillates).
3. **Флап-сценарий для краша:** Enter→twin reparent на walkCol → twin позиция внутри CapsuleCollider → OnCollisionEnter на BoatImpactSoundsCollider → `BoatImpactSounds.Impact(collision)` → `BoatDamage.Impact` → **если HeldItem+твердый коллайдер = OnCollisionEnter каждый кадр** → BoatDamage cooldown (1c) не спасёт от массового вызова Impact → или звук спам, или многократный hullDamage.
4. **Решение мода:** при холде предмета — Harmony-prefix пропускать `ShipItem.ExtraFixedUpdate` (не Enter/Exit), или Harmony-prefix `OnTriggerEnter/Exit` для EmbarkCol при `held != null`, или `Physics.IgnoreLayerCollision` twin↔boat при холде.
5. **Важно:** trigger-коллайдер `EmbarkCol` — отдельный GO от CapsuleCollider лодки. Твердый twin может EnterBoat через trigger, затем столкнуться с CapsuleCollider (non-trigger) — две разные системы коллизий.
