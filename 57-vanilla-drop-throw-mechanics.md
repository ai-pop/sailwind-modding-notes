# 57. Vanilla drop/throw: DropItem, throw charge, ThrowItemAfterDelay

Полный разбор ванильного механизма дропа и броска предметов — ответ на запросы A1, A2. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 47 (holding flow), 44 (ItemRigidbody contract).

## A1. GoPointer.DropItem() — вербатим

```csharp
public void DropItem()
{
    if (Object.op_Implicit((Object)(object)heldItem))
    {
        UISoundPlayer.instance.PlaySmallItemDropSound();
        ((Component)heldItem).gameObject.layer = 0;
        heldItem.held = null;
        heldItem = null;
    }
}
```

**DropItem делает ТОЛЬКО 4 вещи:**
1. `PlaySmallItemDropSound()` — звук дропа (short click).
2. `heldItem.gameObject.layer = 0` — **visual GO на слой 0 (Default)** — предмет снова «видим» для raycast/interact.
3. `heldItem.held = null` — снимает флаг «в руках» с предмета.
4. `heldItem = null` — очищает ссылку GoPointer на предмет.

**DropItem НЕ делает:**
- Не вызывает `heldItem.OnDrop()` — это **отдельный вызов** перед DropItem!
- Не меняет collider.enabled/isTrigger на visual — OnDrop делает это для wallAttachment, но DropItem НЕ делает.
- Не меняет twin (ItemRigidbody) — twin управляется своим FixedUpdate.
- Не сбрасывает `currentThrowPower` — это делается **в LateUpdate** после вызова DropItem.
- Не меняет `nailed`, `attached`, `sold`, `pointedAtBy` — всё остаётся как было.

### Все вызовы DropItem — таблица

| Файл | Метод | Условие | Предшествует OnDrop? |
|------|-------|---------|---------------------|
| GoPointer.cs | LateUpdate | `GameInput.GetKeyUp(InputName 10)` или `(Settings.autoThrow && GetKeyUp(InputName 8))` + `collisions <= 0 || allowObstructedDropping || GetKeyUp(10)` | **Да**: `heldItem.OnDrop(); DropItem();` — OnDrop ВЫЗЫВАЕТСЯ ПЕРЕД DropItem |
| GoPointer.cs | LateUpdate | `MainButtonDown()` + `pointedAtButton.OnItemClick(heldItem)` → return true | **Да**: `heldItem.OnDrop(); DropItem();` |
| GoPointer.cs | LateUpdate | `MainButtonDown()` + heldItem не ShipItem (WorldItem) | **Да**: `heldItem.OnDrop(); DropItem();` |
| Anchor.cs | FixedUpdate | Anchor set/release: `if (held) held.DropItem()` | **Нет**: Anchor вызывает DropItem БЕЗ OnDrop — это вызов GoPointer.DropItem() на Anchor.held (другой контекст) |
| PickupableBoatMooringRope.cs | Update | Rope length exceeded: `held.DropItem()` | **Нет**: только DropItem без OnDrop |

> **КРИТИЧНО:** в 3 основных путях дропа (throw, item-click, world-item) **OnDrop вызывается ПЕРЕД DropItem**. Anchor и MooringRope вызывают только DropItem (без OnDrop). Мод, который ставил `heldItem=null; item.held=null` без OnDrop/DropItem — пропускает ВСЁ, что оба метода делают (см. заметку 58).

### Кто решает «положить» vs «бросить»

**Ключ:** `GameInput.GetKeyUp(InputName 10)` — отпускание кнопки броска (throw/drop key). Это **KeyUp**, не KeyDown.

Ветка в LateUpdate:
```csharp
if (Object.op_Implicit((Object)(object)heldItem) 
    && Object.op_Implicit((Object)(object)((Component)heldItem).GetComponent<ShipItem>()) 
    && timerAfterPickup >= 0.66f 
    && (Object)(object)pointedAtButton == (Object)null)
{
    // ЗАРЯД броска (hold)
    if (GameInput.GetKey((InputName)10) || (Settings.autoThrow && GameInput.GetKey((InputName)8)))
    {
        currentThrowPower += Time.deltaTime;
        if (currentThrowPower > 1f) { currentThrowPower = 1f; }
    }
    // ОТПУСКАНИЕ → дроп/бросок
    else if (GameInput.GetKeyUp((InputName)10) || (Settings.autoThrow && GameInput.GetKeyUp((InputName)8)))
    {
        if (heldItem.colChecker.collisions <= 0 
            || heldItem.colChecker.allowObstructedDropping 
            || GameInput.GetKeyUp((InputName)10))
        {
            Rigidbody component = ((Component)((Component)heldItem).GetComponent<ShipItem>().GetItemRigidbody()).GetComponent<Rigidbody>();
            heldItem.OnDrop();
            DropItem();
            if (currentThrowPower > throwDelay)  // throwDelay = 0.4
            {
                ((MonoBehaviour)this).StartCoroutine(ThrowItemAfterDelay(component, currentThrowPower - throwDelay));
            }
        }
        currentThrowPower = 0f;
    }
}
else
{
    currentThrowPower = 0f;  // сброс при отсутствии heldItem или pointing at something
}
```

**Логика:**
- Если `currentThrowPower > throwDelay (0.4)` → бросок (ThrowItemAfterDelay).
- Если `currentThrowPower <= throwDelay` → «просто положить» (DropItem без ThrowItemAfterDelay).
- `Settings.autoThrow` — если включён, InputName 8 (обычная кнопка interact/drop) тоже заряжает бросок.
- **Условие начала заряда:** `timerAfterPickup >= 0.66f` — нельзя бросить в первые 0.66 с после PickUp + **not pointing at any button** (pointedAtButton == null).

**Гард от клика:** `collisions <= 0 || allowObstructedDropping || GetKeyUp(10)` — если предмет в столкновении (красный контур) → дроп блокируется, **если** только это не throw key (InputName 10). Throw key дропает даже с красным контуром.

## A2. Механика заряда/силы броска

### Поля

| Поле | Тип | Значение | Содержание |
|------|-----|----------|------------|
| `throwDelay` | `float` | 0.4 (public, SerializeField) | Порог: если `currentThrowPower > throwDelay` → бросок, иначе «положить» |
| `throwForce` | `float` | 10 (public, SerializeField) | Множитель силы броска |
| `currentThrowPower` | `float` | private | Текущий заряд (0–1), увеличивается `+= Time.deltaTime` при hold key |
| `timerAfterPickup` | `float` | private | Таймер после PickUp — бросок доступен только при `>= 0.66f` |

### Все места чтения/записи currentThrowPower

| Место | Чтение/Запись | Значение |
|-------|---------------|----------|
| LateUpdate charge | `+= Time.deltaTime` | Заряд при hold InputName 10/8 |
| LateUpdate charge clamp | `if > 1f → 1f` | Максимум заряда = 1 |
| LateUpdate release | `if > throwDelay → ThrowItemAfterDelay(rb, currentThrowPower - throwDelay)` | Передаётся `force = currentThrowPower - 0.4` (max = 0.6) |
| LateUpdate release | `= 0f` (после дропа) | Сброс заряда |
| LateUpdate else | `= 0f` (нет heldItem или pointing) | Сброс заряда |

### ThrowItemAfterDelay — вербатим

```csharp
private IEnumerator ThrowItemAfterDelay(Rigidbody heldRigidbody, float force)
{
    yield return (object)new WaitForFixedUpdate();
    if (force > 1f)
    {
        force = 1f;
    }
    heldRigidbody.AddForce(((Component)this).transform.forward * throwForce * force * heldRigidbody.mass);
}
```

**Формула:** `AddForce(pointer.forward * throwForce * force * mass)` = `pointer.forward * 10 * (currentThrowPower - 0.4) * mass`

**ForceMode:** по умолчанию `ForceMode.Force` (не Impulse, не VelocityChange) — сила применяется **за один fixed-кадр** как `F = m*a`, значит `Δv = F*dt/m = throwForce*force*dt`. При `fixedDt=0.022`: `Δv = 10 * 0.6 * 0.022 ≈ 0.13 m/s` на единицу массы — но **mass умножается** в AddForce → `Δv = throwForce*force*dt = 10*0.6*0.022 = 0.132 m/s` (mass cancels! `F = throwForce*force*mass → a = F/m = throwForce*force → Δv = a*dt`).

> **КРИТИЧНО:** `AddForce` с `ForceMode.Force` и массой в формуле → **mass НЕ влияет на итоговую скорость**. `Δv = throwForce * force * fixedDt ≈ 10 * 0.6 * 0.022 = 0.132 m/s` для максимального заряда. Это **очень маленькая** скорость — предмет после броска летит ~0.13 м/с вперёд, а не 5–9 м/с как мод пытался.

**Задержка:** `WaitForFixedUpdate()` — сила применяется **в следующем FixedUpdate** после дропа. Это значит: в кадре дропа `OnDrop() + DropItem()` снимают held → в том же кадре LateUpdate уже прошла → **следующий FixedUpdate** twin впервые dynamic (6 fixed frames after spawn timer) → **+1 fixed** → ThrowItemAfterDelay fires AddForce.

> **Проблема мода:** мод ставил velocity вручную (~5–9 м/с) в кадр отпускания. Но `ItemRigidbody.FixedUpdate` в первые ~6 fixed frames держит twin kinematic (`fixedFramesSinceSpawn < 6`) → **velocity записывается в kinematic Rigidbody → бессмысленно** (kinematic не применяет velocity). Когда twin наконец dynamic (frame 7+), его velocity = 0 (kinematic не сохраняет velocity) → предмет падает камнем вниз.

## Практические выводы для мододела

1. **DropItem — минимальный:** только звук + layer=0 + held=null. **OnDrop() — отдельный вызов**, перед DropItem. Пропуск OnDrop = пропуск wallAttachment snap, attached=false, inventory withdrawal — предмет теряет «прилепленность».
2. **Бросок — через AddForce с WaitForFixedUpdate:** сила применяется в **следующем fixed frame**, ForceMode.Force, масса в формуле cancelling → итоговая Δv ≈ 0.13 м/с (макс). Ванильный бросок — **слабый**, предмет почти не летит вперёд.
3. **«Положить» vs «бросить»** — порог `throwDelay = 0.4` с. Если держал кнопку < 0.4 с → DropItem без ThrowItemAfterDelay = «просто положить». > 0.4 с → бросок.
4. **TimerAfterPickup ≥ 0.66 с** — нельзя бросить сразу после Pickup. Дроп через обычную кнопку (InputName 8) при `Settings.autoThrow` тоже заряжает.
5. **Мод, который ставил velocity вручную** — twin kinematic в первые 6 fixed frames → velocity = 0 после kinematic→dynamic перехода. Нужно: либо ждать 6+ fixed frames перед выставлением velocity, либо делать `fixedFramesSinceSpawn = 7+` вручную, либо писать velocity после `isKinematic = false` в **следующем** FixedUpdate.
