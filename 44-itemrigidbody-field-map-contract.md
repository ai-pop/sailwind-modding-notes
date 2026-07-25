# 44. `ItemRigidbody`: карта полей и контракт twin-объекта

Точный API-контракт физического twin-объекта предмета — чтобы не уходить в reflection наугад. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Архитектура «два GameObject» — в заметке 16; плавучесть — в заметке 43.

## Два объекта и как их получить

У каждого `ShipItem` есть **visual** (сам `ShipItem`) и **twin** (отдельный GameObject с физикой).

### Поля на `ShipItem`
| Поле | Модификатор | Тип | Что это |
|------|-------------|-----|---------|
| `itemRigidbody` | `protected` | `Transform` | Трансформ twin-объекта. **Не публичный** — снаружи напрямую не достать. |
| `itemRigidbodyC` | `public` | `ItemRigidbody` | Компонент `ItemRigidbody` на twin. **Публичный**. |

### Канонические доступоры (на `ShipItem`)
```csharp
public ItemRigidbody GetItemRigidbody()  // → itemRigidbody.GetComponent<ItemRigidbody>()
public void ResetRigidbody()             // → GetItemRigidbody().ResetPos()
public Rigidbody /* через twin */        // shipItem.GetItemRigidbody().GetBody()
```
- **Rigidbody twin'а** надёжно брать так: `shipItem.GetItemRigidbody().GetBody()` (`GetBody()` возвращает приватный `rigidbody` twin'а).
- Сам `ItemRigidbody`: `shipItem.itemRigidbodyC` (public) или `shipItem.GetItemRigidbody()`.

## ⚠ Тайминг создания twin (критично)

Twin создаётся **в корутине, стартованной из `Awake`**, не синхронно:

```
ShipItem.Awake()
  └─ StartCoroutine(LoadAfterDelay())
        ├─ (до первого yield, в том же кадре, но ПОСЛЕ Awake)
        │    CreateRigidbody()      ← создаёт twin-GameObject + ItemRigidbody
        │    AddCollisionChecker()
        │    AddLODGroup()
        │    visual.GetComponent<Rigidbody>().isKinematic = true
        ├─ yield WaitForEndOfFrame
        ├─ OnLoad()                 ← виртуальный хук подкласса
        ├─ yield WaitForEndOfFrame
        └─ (обработка currentCrateId — вложение в ящик)
```

**Следствие:** сразу после `Object.Instantiate(prefab)` поля `itemRigidbody` / `itemRigidbodyC` могут быть **ещё null** (корутина `LoadAfterDelay` не дошла до `CreateRigidbody`). Перед работой с twin'ом нужно **дождаться**:
```csharp
// вариант 1: подождать кадр(ы)
yield return new WaitForEndOfFrame();   // обычно достаточно 1–2

// вариант 2: поллить
while (shipItem.itemRigidbodyC == null) yield return null;
var twin = shipItem.GetItemRigidbody();
```
`CreateRigidbody()`:
```csharp
itemRigidbody = new GameObject().transform;   // пустой GO
itemRigidbody.parent = world;                  // world = FloatingOriginManager.instance.transform
var ir = itemRigidbody.gameObject.AddComponent<ItemRigidbody>();
ir.RegisterItem(this);
itemRigidbodyC = ir;
```
Имя twin'а ставится в `ItemRigidbody.Start()`: `gameObject.name = item.name + ":ItemRigidbody"`.

## Карта полей `ItemRigidbody`

| Поле | Модификатор | Тип | Назначение |
|------|-------------|-----|-----------|
| `debugForceKinematic` | `public` | `bool` | **Принудительная кинематика** (заморозка). Ставит `WorldItemSpawner.FreezeItem()`. **Поле на `ItemRigidbody`, НЕ на `ShipItem`** (доступ: `shipItem.GetItemRigidbody().debugForceKinematic`). |
| `debugLog` | `public` | `bool` | verbose-логирование. |
| `attached` | `public` | `bool` | прикреплён (к стене/плите) → кинематика. |
| `disableCol` | `public` | `bool` | выключить коллайдеры (+ `ToggleCollider(false)`). |
| `inStove` | `public` | `bool` | предмет на плите. |
| `rigidbody` | `private` | `Rigidbody` | физическое тело; наружу через `GetBody()`. |
| `floater` | `private` | `SimpleFloatingObject` | поплавок (Crest); создаётся в `Start()`. |
| `item` | `[SerializeField] private` | `ShipItem` | обратная ссылка на visual. |
| `boxCol` / `meshCol` / `capsuleCol` | `private` | коллайдеры | копия коллайдеров visual (созд. в `AddCollider()`). |
| `subcolliders` | `private` | `List<Collider>` | под-коллайдеры (тег `ItemSubcollider`). |
| `onBoat` | `private` | `bool` | на лодке. |
| `currentBox` / `currentInventorySlot` | `private` | `Transform` | в ящике / в слоте инвентаря. |
| `outOfRange` | `[SerializeField] private` | `bool` | дальше 600 ед. от камеры. |
| `idleTimer` | `private` | `float` | **мёртвое поле** — объявлено, но не используется. |
| `dynamicColTimer` | `private` | `float` | таймер режима точных коллизий (=6 при старте/подборе). |
| `fixedFramesSinceSpawn` | `private` | `float` | счётчик fixed-кадров (первые 6 — кинематика). |
| `distanceCheckTimer` | `private` | `float` | период проверки дистанции (5–8 c). |
| `framesUntilDestroy` | `private` | `int` | счётчик до уничтожения вне зоны (>10 → destroy). |

### Публичные методы `ItemRigidbody`
| Метод | Действие |
|-------|----------|
| `GetBody()` | → `Rigidbody` twin'а. |
| `GetShipItem()` | → `ShipItem` (visual). |
| `RegisterItem(ShipItem)` | связать с visual. |
| `ResetPos()` | `isKinematic = true`, позиция/поворот = visual, `velocity = 0`. |
| `UpdateMass()` | пересчёт массы (item.mass + содержимое crate/bottle/tea/salt/soup). |
| `EnterInventorySlot(Transform)` / `ExitInventorySlot()` | в/из инвентаря (слой 5 / 2, коллайдеры). |
| `EnterBox(...)` / `ExitBox()` / `GetCurrentBox()` | в/из ящика. |
| `GetCurrentInventorySlot()` | текущий слот (или null). |
| `EnterBoat()` / `ExitBoat()` | на/с лодки (репарент, `BoatMass.Add/RemoveItem`). |
| `ToggleCollider(bool state)` | коллайдеры вкл/выкл; **всегда** `floater.enabled = false` (см. заметку 43). |
| `ForceRigidbodyToWalkCol()` | прижать к walk-коллайдеру лодки. |

## Freeze / unfreeze (как корректно разморозить предмет)

Кинематика twin'а управляется флагом `flag2` в `FixedUpdate` (полная таблица — заметка 43). Принудительная заморозка — `debugForceKinematic`.

**Разморозить предмет для плавания** (не ломая подбор):
```csharp
var ir = shipItem.GetItemRigidbody();
ir.debugForceKinematic = false;     // снять принудительную кинематику
// НЕ трогать visual, НЕ добавлять компоненты на visual
// тело станет динамическим само, когда: sold && !held && !attached && в зоне && playing
```
- `debugForceKinematic` — **на `ItemRigidbody`**, не на `ShipItem`. Попытка `shipItem.debugForceKinematic` → **CS1061** (нет такого члена).
- Не ставить `debugForceKinematic = true`, если нужно плавание (это заморозка).
- Предмет **должен быть `sold = true`** (иначе `!sold` → кинематика, заметка 43).

## Кто master по позиции (visual ↔ physics)

В `ItemRigidbody.FixedUpdate` синхронизация зависит от состояния:

| Состояние | Master | Код |
|-----------|--------|-----|
| `held` **или** `!sold` (в руках / магазинный) | **visual** → twin следует | `twin.position = visual.position` |
| `sold` && свободно в мире (не held) | **twin (физика)** → visual следует | `visual.position = twin.position` |
| в инвентаре/ящике | слот/ящик | лерп к слоту / `box.TransformPoint` |
| на лодке (`onBoat`) | walk-коллайдер лодки | `MoveRigidbodyToWalkCol` / `MoveItemToWalkColRigidbody` |

**Критично для моддера:** для свободного купленного предмета (кейс «плавает в мире») **master — физический twin**. Двигать/плавать нужно **twin** (`GetBody()`/twin.transform), **не visual**. Если двигать visual (или повесить поплавок/`Rigidbody` на visual) — `FixedUpdate` перезапишет visual позицией twin'а → конфликт, и это **ломает подбор** (у visual триггер-коллайдеры `isTrigger`, по ним идёт raycast подбора).

## «Сон» физики и дальность

- **`idleTimer` не используется** (мёртвое поле) — не искать в нём логику.
- **Unity-сон тела:** `rigidbody.IsSleeping()` учитывается только для mesh-коллайдерных предметов (`meshCol != null && IsSleeping() && dynamicColTimer <= 0` → кинематика).
- **Дальность:** `outOfRange` = дистанция до `Camera.main` > **600** ед. При `outOfRange`: `FixedUpdate` делает ранний выход (физика не управляется); купленный предмет не на лодке через >10 кадров **уничтожается** (`DestroyItem`).
- **Первые 6 fixed-кадров** после спавна — кинематика (`fixedFramesSinceSpawn < 6`).

## Шпаргалка доступа

```csharp
ShipItem item = go.GetComponent<ShipItem>();
yield return WaitUntilItemRigidbody(item);        // дождаться twin (см. выше)

ItemRigidbody ir   = item.GetItemRigidbody();      // или item.itemRigidbodyC
Rigidbody     body = ir.GetBody();                 // физическое тело twin
Transform     twin = ir.transform;                 // трансформ twin
SimpleFloatingObject fl = /* приватный floater — только патчем/Harmony */;

// разморозка:
ir.debugForceKinematic = false;
// master по позиции для свободного sold-предмета = twin (двигать twin, не visual)
```

> `floater` — приватное поле `ItemRigidbody`; снаружи напрямую не взять. Управляется игрой через `ToggleCollider` (выключается). Для своей плавучести — свой компонент на **twin** или патч (заметка 43).

## Практические выводы

1. Twin-доступ: `shipItem.GetItemRigidbody()` (или `itemRigidbodyC`, public); `Rigidbody` — через `GetBody()`. Поле `itemRigidbody` (Transform) — protected.
2. **Twin создаётся в корутине из Awake** → после `Instantiate` подождать кадр / поллить `itemRigidbodyC != null`.
3. `debugForceKinematic` — **на `ItemRigidbody`** (public), не на `ShipItem` (иначе CS1061). Разморозка = `false` + `sold=true` + не держать/не крепить.
4. Для свободного предмета **master = физический twin**; двигать/плавать twin, **никогда visual** (иначе ломается подбор).
5. `idleTimer` — мёртвое поле; «сон» = Unity `IsSleeping()` (mesh-предметы) + culling на >600 ед.
