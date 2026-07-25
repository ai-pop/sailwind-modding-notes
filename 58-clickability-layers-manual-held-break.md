# 58. Clickability и слои: что ломает ручной разрыв хвата, луч подбора

Разбор того, что делает OnDrop/OnPickup, и почему обход этих методов ломает кликабельность — ответ на запросы A3, A4. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 47 (holding), 57 (DropItem).

## A3. ShipItem.OnDrop() — вербатим

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

**OnDrop делает (для sold предмета):**
- Если `wallAttachment && inRangeOfWall` → twin snap к стене + `attached = true` (kinematic lock).
- Если `!sold` → `ReturnToShopPos()` (предмет не куплен → вернуть в магазин).

**OnDrop НЕ делает для обычного sold предмета (без wallAttachment):** ничего — пустой pass-through. Побочки дропа для обычного предмета — только в `GoPointer.DropItem()` (layer=0, held=null).

### ShipItem.OnPickup() — вербатим

```csharp
public override void OnPickup()
{
    if (wallAttachment)
    {
        itemRigidbodyC.attached = false;
    }
    if (Object.op_Implicit((Object)(object)itemRigidbodyC) 
        && (Object)(object)itemRigidbodyC.GetCurrentInventorySlot() != (Object)null)
    {
        GPButtonInventorySlot component = ((Component)itemRigidbodyC.GetCurrentInventorySlot()).GetComponent<GPButtonInventorySlot>();
        if (Object.op_Implicit((Object)(object)component))
        {
            component.WithdrawItem();
        }
    }
    overrideEnableOutline = false;
}
```

**OnPickup делает:**
1. `wallAttachment → attached = false` — снимает kinematic lock (twin dynamic).
2. Inventory slot withdrawal — вытаскивает из инвентаря.
3. `overrideEnableOutline = false` — отключает override обводки.

### PickupableItem.OnPickup() / OnDrop() — вербатим

```csharp
public virtual void OnPickup() { }
public virtual void OnDrop() { }
```

**Базовый класс — пустые виртуальные методы.** Всё действие — в overrides на ShipItem и его подклассах.

## Таблица: что ваниль делает при дропе → последствие пропуска модом

Мод ставил `pointer.heldItem=null; item.held=null` напрямую, **не вызывая OnDrop() и DropItem()**.

| Ванильное действие | Метод | Последствие пропуска модом |
|-------------------|-------|---------------------------|
| `PlaySmallItemDropSound()` | DropItem | Нет звука дропа — косметика |
| `heldItem.gameObject.layer = 0` | DropItem | **КРИТИЧНО: visual остаётся на слое 2 (IgnoreRaycast) → raycast GoPointer не видит предмет → предмет НЕКЛИКАБелен навсегда!** |
| `heldItem.held = null` | DropItem | Мод делал сам — OK |
| `heldItem = null` (GoPointer) | DropItem | **Мод делал сам — OK** |
| `wallAttachment → attached=false` (OnPickup!) | OnPickup | Если предмет был wallAttachment → при pickup mod не снял attached → twin kinematic → предмет не двигается |
| `wallAttachment → twin snap to wall + attached=true` | OnDrop | Пропуск → предмет не прилепился к стене, twin free-falls |
| `overrideEnableOutline = false` | OnPickup | Обводка предмета может остаться синей/красной |
| Inventory withdrawal | OnPickup | Если предмет был в слоте → не вытащен → stuck в инвентаре |
| `itemRigidbodyC.attached = false` (wallAttachment OnPickup) | OnPickup | twin kinematic lock не снят → предмет frozen |

> **Корневая причина некликабельности:** `DropItem` ставит `heldItem.gameObject.layer = 0`. Мод **пропустил это** → visual GO остался на **layer 2 (IgnoreRaycast)**. GoPointer raycast mask `-604165` **исключает layer 2** (confirmed в заметке 52) → предмет **никогда** не попадает под луч подбора. Layer 2 = IgnoreRaycast — предмет «прозрачен» для raycast.

**Фикс:** после ручного разрыва хвата — установить `item.gameObject.layer = 0` (как DropItem делает). Это восстановит кликабельность.

## A4. Слои visual GO по жизненному циклу предмета

### Таблица: метод → layer visual GO

| Событие | Метод/Код | Layer visual | Layer twin |
|---------|-----------|:------------:|:----------:|
| Spawn (Start) | `ItemRigidbody.Start()` → `gameObject.layer = 2` (twin); visual наследует prefab layer | 0 (Default) или 2 (prefab) | 2 |
| Load (LoadAfterDelay) | `ProcessSaveable()`: если `parentObject == -3` → `layer = 2` | 2 | 2 |
| ResetPos | `ItemRigidbody.ResetPos()`: twin position = visual | — (unchanged) | 2 |
| PickUp | `GoPointer.PickUpItem()` → `item.gameObject.layer = 2` | **2** (IgnoreRaycast) | 2 |
| Held (ongoing) | LateUpdate drives visual position; no layer changes | 2 | 2 |
| Drop (normal) | `GoPointer.DropItem()` → `item.gameObject.layer = 0` | **0** (Default) | 2 |
| Drop (wallAttachment) | OnDrop + DropItem → `layer = 0` | 0 | 2 (attached=true → kinematic) |
| Enter inventory | `ItemRigidbody.EnterInventorySlot()` → `layer = 5` (UI) + all children | **5** | 2 |
| Exit inventory | `ItemRigidbody.ExitInventorySlot()` → `layer = 2` + all children | **2** | 2 |
| Enter crate | `CrateInventory.InsertItem()` → `layer = 26` + all children | **26** | 2 |
| Exit crate | `CrateInventory.WithdrawItem()` → `layer = 2` + all children | **2** | 2 |

> **Слои visual при critical transitions:** PickUp → 2 (не raycast-ится, т.к. уже в руке). Drop → 0 (raycast-ится, можно подобрать). Inventory → 5 (UI layer, не raycast-ится). Crate → 26 (специальный, не raycast-ится).

### DoRaycast: точные условия

```csharp
Ray val = (!debugEditorPointer) ? raycastRay : Camera.main.ScreenPointToRay(Input.mousePosition);
LayerMask val2 = LayerMask.op_Implicit(-604165);
float num = 1.8f;
if (Physics.Raycast(val, ref hit, num, LayerMask.op_Implicit(val2)) 
    && !GameState.sleeping 
    && !(GameState.inBed != null) 
    && !BoatCamera.on)
```

**Маска `-604165` побитово (32-bit signed):**

```
-604165 = ~604164
604164 in binary (bits 0..31):
bit 2  = 1  (layer 2 IgnoreRaycast — excluded from raycast!)
bit 8  = 1  (layer 8 Player)
bit 12 = 1  (layer 12 HullPlayerCollider)
... (exact decode needs runtime)
```

**Ключ:** layer 2 (IgnoreRaycast) **исключён из raycast mask**. Это значит:
- visual на layer 2 → **не попадает под луч** → не кликабелен.
- visual на layer 0 → **попадает под луч** → кликабелен.
- `Physics.queriesHitTriggers` — Unity default = true → trigger colliders (isTrigger=true) **попадают под raycast**. Visual colliders — isTrigger (set in Awake), но на layer 0 они raycast-ятся.

**Дистанция:** `num = 1.8f` — raycast 1.8 метра от камеры/pointer.

**Что ломается при пропуске DropItem:**
- visual layer остаётся 2 → raycast mask excludes layer 2 → **предмет невидим для GoPointer** → навсегда некликабелен.
- visual collider.enabled — не менялся (mod не вызывал DropItem, DropItem не меняет collider.enabled). collider.enabled = true (как при spawn). Но layer 2 → raycast не доходит.
- isTrigger на visual — при held: isTrigger=true (ItemRigidbody.FixedUpdate ставит). При дропе: `held != null` снят → FixedUpdate ставит isTrigger=false → visual collider becomes solid → raycast может hit trigger=false collider (queriesHitTriggers не влияет, т.к. уже non-trigger на layer 0). **Но при обходе:** held=null модом → FixedUpdate меняет isTrigger=false → visual solid на layer 2 → raycast mask excludes layer 2 → всё равно невидим.

> **Вердикт:** единственная причина некликабельности — **layer 2 вместо layer 0 на visual GO**. Фикс: `item.gameObject.layer = 0` после модового разрыва хвата.

## Практические выводы для мододела

1. **OnDrop + DropItem — обязательная пара.** DropItem делает layer=0 — это **единственный** механизм, возвращающий предмет в raycast-видимость. Пропуск → предмет навсегда некликабелен.
2. **OnPickup тоже обязательна** для wallAttachment предметов — снимает attached=true (kinematic lock).
3. **Правильный ручной разрыв хвата для мода:**
   - `item.OnDrop()` — для wallAttachment snap и inventory withdrawal
   - `pointer.DropItem()` — для layer=0, sound, held=null
   - Или минимум: `item.gameObject.layer = 0; item.held = null; pointer.heldItem = null`
4. **Layer lifecycle:** spawn→2, pickup→2, drop→0, inventory→5, crate→26. После дропа layer=0 — это **Default**, видимый для raycast mask.
5. **Raycast mask `-604165` excludes layer 2** — предмет на layer 2 невидим для GoPointer, независимо от collider type/isTrigger.
