# 60. Все читатели PickupableItem.held, дефолтный биндинг Throw

Полная таблица всех мест, где читается `PickupableItem.held` — ответ на запросы A7, A8. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 57 (DropItem), 58 (clickability).

## A7. ВСЕ читатели PickupableItem.held — полная таблица

| Класс | Метод | Что делает при held!=null | Что делает при переходе held→null |
|-------|-------|--------------------------|----------------------------------|
| **GoPointer** | `PickUpItem()` | `heldItem.held = this` — ставит held | — |
| **GoPointer** | `DropItem()` | — | `heldItem.held = null` — снимает held |
| **GoPointer** | `LateUpdate` (throw charge) | `if heldItem!=null && timerAfterPickup>=0.66f` → charge throw power | `else → currentThrowPower = 0f` |
| **GoPointer** | `LateUpdate` (positioning) | `heldItem.transform.position = pointer+forward*holdDistance` → drives visual | — (position frozen after DropItem) |
| **GoPointer** | `DoRaycast()` | `if heldItem == pointedAtItem → pointedAtButton = null` (self-look prevention) | — |
| **GoPointer** | `LateUpdate` (lookUI) | `if heldItem!=null && !AltButtonHeld() → ShowHoldText` | `else → ClearText()` |
| **GoPointer** | `LateUpdate` (interact) | `if heldItem!=null && !wasInSettingsMenu → OnItemClick` (place item on button) | `else → PickUpItem` (pickup what's pointed at) |
| **ItemRigidbody** | `FixedUpdate` (position sync) | `if item.held → twin.position = visual.position` (twin slave) | `else → visual.position = twin.position` (visual slave) |
| **ItemRigidbody** | `FixedUpdate` (isKinematic flag) | `flag2 = item.held != null → true → isKinematic` (twin frozen) | `flag2 = false → isKinematic=false` (twin dynamic) |
| **ItemRigidbody** | `FixedUpdate` (collider isTrigger) | `boxCol.isTrigger = item.held != null → true` (visual trigger) | `boxCol.isTrigger = false` (visual solid) |
| **ItemRigidbody** | `FixedUpdate` (dynamicColTimer) | `if item.held → SetDynamicColTimer() → 6f` (reset timer) | `if !item.held && timer>0 → countdown (collision mode change)` |
| **ItemRigidbody** | `FixedUpdate` (exitBoat guard) | — | `if !item.held && !attached → ExitBoat possible` |
| **ShipItem** | `OnPickup()` | `if wallAttachment → attached=false; overrideEnableOutline=false; inventory withdrawal` | — |
| **ShipItem** | `OnDrop()` | — | `if wallAttachment && inRangeOfWall → attached=true, twin snap` |
| **ShipItem** | `Update()` (wallAttachment raycast) | `if held!=null && wallAttachment → raycast forward 1.3m → inRangeOfWall` | `else → inRangeOfWall=false` |
| **ShipItem** | `ExtraFixedUpdate()` (EnterBoat) | — | `if held==null && !attached → ExitBoat possible` |
| **PickupableItemCollisionChecker** | `Update()` | `if !held || inInventory → collisions=0` (no collision check) | `if held → check collisions + decolDistance` |
| **PickupableItemCollisionChecker** | `OnTriggerEnter` | — | Item enters collision checker (decollision) |
| **WorldItemSpawner** | `Update()` | `if item.held → schedule respawn cooldown, item.debugForceKinematic=false, item=null` | — |
| **MouthCol** | `Update()` (eat) | `if currentFood.held && sold && slot==-1 → eat audio + timer → EatFood()` | `else → eatAudio.Stop()` |
| **Cleaner** | `LateUpdate()` (broom) | `if item.held → skinnedBroom enabled, sweep animation, clean logic` | `else → staticBroom, cleanCooldown=spacing, targetVolume=0` |
| **CookableFood** | `OnTriggerStay` | `if item.held → cook when touching stove trigger` | — |
| **StoveFuel** | `Update()` | `if !item.held → auto-insert fuel into stove` | `if item.held → don't insert` |
| **FishingRodFish** | `Update()` | `if rod.held && InWater → catch fish` | — |
| **Anchor** | `FixedUpdate()` | `if held → DropItem()` (auto-drop anchor on set/release) | — |
| **PickupableBoatMooringRope** | `Update()` | `if held && dist > maxLength → DropItem()` (auto-drop rope) | — |
| **ItemBox** | `???` | `if !held → allow box insertion` | — |
| **Shopkeeper** | `BuyItem/SellItem` | `if item.held → buy/sell transaction` | — |
| **Mug** | `Update()` | `if bottle.held → drinking from mug` | — |
| **Windchimes** | `Update()` | `if item.held || !sold || IsHanging → chimes sound` | — |
| **BoatMass** | `UpdateMass()` | `if item.GetShipItem().held → item mass NOT added to boat mass?` (check: held items excluded?) | — |

### Ключевые последствия при переходе held→null

| Система | Что начинает работать при held=null | Критично для мода? |
|---------|-------------------------------------|:----------------:|
| ItemRigidbody position sync | twin → position **master** (visual follows twin) | Да — twin может двигаться физикой |
| ItemRigidbody isKinematic | twin → **dynamic** (если sold, !nailed, !attached) | Да — twin впервые участвует в физике |
| ItemRigidbody colliders | twin colliders → **isTrigger=false** (solid) | Да — twin сталкивается с миром |
| ShipItem ExtraFixedUpdate | ExitBoat possible (if !attached) | Да — может выйти из лодки |
| PickupableItemCollisionChecker | collisions=0 → decol stops | Нет — cosmetic |
| WorldItemSpawner | respawn timer scheduled | Нет — cosmetic |
| MouthCol (eat) | eatAudio stops → eating stops | Нет — cosmetic |
| Cleaner (broom) | static broom → cleaning stops | Нет — cosmetic |

## A8. Дефолтный биндинг Throw/drop

### GoPointer.Start() — вербатим

```csharp
private void Start()
{
    if (Application.isEditor)
    {
        mainKey = "r";
        altKey = "f";
    }
    else
    {
        mainKey = "f";
        altKey = "r";
    }
    pointerInitialPosition = pointer.localPosition;
    // ...
}
```

**Дефолтные key strings:** `mainKey = "f"` (interact/drop), `altKey = "r"` (alt activate/throw). В Editor: `"r"` main, `"f"` alt.

### InputName mapping

`GameInput` — **не декомпилирован** (отсутствует в Assembly-CSharp, как указано в заметке 24). Но из контекста GoPointer:

| InputName | Назначение | Дефолтный key (build) | Дефолтный key (editor) | VR |
|-----------|-----------|:--:|:--:|:--:|
| 8 | Main interact/drop (LMB-like) | `GameInput.GetKeyDown(8)` → MainButtonDown → `"f"` (build) | `"r"` | Oculus trigger |
| 9 | Alt activate (RMB-like) | `GameInput.GetKeyDown(9)` → AltButtonDown → `"r"` (build) | `"f"` | Oculus grip |
| 10 | Throw/drop dedicated key | `GameInput.GetKey(10)` → throw charge; `GetKeyUp(10)` → throw release | — (unknown KeyCode) | — |

**InputName 10 — throw key:** `GameInput.GetKey(10)` используется для charge, `GameInput.GetKeyUp(10)` для release. KeyCode — **не виден** из декомпиляции (GameInput.cs отсутствует). Из runtime/logical: это **отдельный key** от InputName 8 (main interact). Возможные кандидаты: `G` (common throw key in games), `Q`, или mouse middle button.

> **Точный KeyCode для InputName 10** — требует runtime-проверки (BepInEx log GameInput mappings). Из decompilation — только enum number, не key string.

### Secondary/gamepad bindings

```csharp
// MainButtonDown
if (type == leftTouch && OVRInput.GetUp(Button.PrimaryIndexTrigger, Controller.LTouch)) return true;
if (type == rightTouch && OVRInput.GetUp(Button.PrimaryIndexTrigger, Controller.RTouch)) return true;
if (type == crosshairMouse && !GameState.inCursorMenu && GameInput.GetKeyUp(InputName 8)) return true;

// AltButtonDown  
if (type == leftTouch && OVRInput.GetDown(Button.Two, Controller.LTouch)) return true;  // X button
if (type == rightTouch && OVRInput.GetDown(Button.One, Controller.RTouch)) return true;  // A button
if (type == crosshairMouse && !GameState.inCursorMenu && GameInput.GetKeyDown(InputName 9)) return true;
```

**VR:** Main = Oculus trigger (index), Alt = X/A buttons. Throw key (InputName 10) — **не виден в VR bindings** из GoPointer (VR использует только InputName 8/9 + direct OVRInput). InputName 10 — **desktop-only** (mouse/keyboard).

## Практические выводы для мододела

1. **held → null triggers massive state change:** twin becomes dynamic, solid colliders, position master, ExitBoat possible. **All held→null consequences happen in ItemRigidbody.FixedUpdate** — not instantly in LateUpdate.
2. **held!=null → twin frozen (kinematic, position slave, trigger colliders).** Mod that bypassed held must also handle layer, isTrigger, isKinematic changes.
3. **WorldItemSpawner reads held** — schedules respawn cooldown when item picked up. Not critical for throw mechanics.
4. **MouthCol (eat) reads held** — eating stops when held=null. Not critical for throw.
5. **InputName 10 = throw key** — desktop-only, KeyCode unknown from decompilation (need runtime). Separate from InputName 8 (main interact/drop) and 9 (alt activate).
6. **Settings.autoThrow** — if enabled, InputName 8 also charges throw (same key as main interact becomes throw+charge on hold).
