# 33. Спавн и подбор предметов в мире

Как игра создаёт предметы в мире и как игрок их подбирает. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Фреймворк предметов — в заметке 16, инвентарь — в заметке 32.

## Спавн предметов в мире (`WorldItemSpawner` + `ItemSpawners`)

### `ItemSpawners` (синглтон, реестр кулдаунов)
- `cooldowns[]` (`float`) — по одному кулдауну на спавнер. Тикают в `Update`: `cooldowns[i] -= deltaTime * Sun.sun.timescale` (зависят от ускорения времени).
- `RegisterNewSpawner()` — находит свободный слот (`cooldown < -2`), ставит `0`, возвращает индекс.
- Кулдауны сохраняются в сейв (`itemSpawnerCooldowns`, заметка 11) — респаун переживает загрузку.

### `WorldItemSpawner` (точка спавна одного предмета)
Ставится в мире как GameObject. Поля: `itemSpawnerIndex` (слот в `ItemSpawners`), `respawnTime`, `itemPrefab`, `item` (текущий заспавненный предмет).

**Логика (`Update`):**
1. **Предмет подобрали** (`item.held != null`): спавнер взводит кулдаун респауна — `respawnTime * Random(0.75, 1.25)` (или `-999`, если `respawnTime <= 0` = без респауна), регистрирует предмет в сейв (`RegisterToSave`), отпускает ссылку (`item = null`).
2. **Предмета нет и кулдаун истёк** (`0 >= cooldown > -10`):
   - если игрок **дальше 100 единиц** → отложить (`cooldown = 0.1`, повторить позже) — не спавнить вдали;
   - иначе → `SpawnItem()`.

**`SpawnItem()`** — «ритуал» создания предмета:
```csharp
item = Instantiate(itemPrefab, transform.position, transform.rotation).GetComponent<ShipItem>();
item.transform.parent = transform.parent;
item.sold = true;                       // «принадлежит миру», подобран
item.GetComponent<Good>()?.RegisterAsMissionless();   // не привязан к миссии
StartCoroutine(FreezeItem());           // kinematic = true через кадр (лежит неподвижно)
```

> **Важно для моддера:** заспавненный предмет **кинематический** (`debugForceKinematic = true`) — он не падает и не тонет, пока его не тронут. Плавучесть включается, когда предмет снимают с «заморозки».

## Плавучесть предметов (`ItemRigidbody` + `SimpleFloatingObject`)

Каждый предмет **плавает в воде**. В `ItemRigidbody` (отдельный физический объект предмета, заметка 16):
```csharp
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;   // высота «всплытия» (у ShipItem.floaterHeight, по умолч. 1.6)
```
- `SimpleFloatingObject` (класс отсутствует в репо, заметка 24) держит предмет на поверхности: `_raiseObject` — целевая высота над водой, `_dragInWaterRotational` — вращательное сопротивление.
- `floater.enabled = false` — когда предмет не должен плавать (в инвентаре, в руках, на плите).

## Подбор предмета (`GoPointer`)

Подбор идёт через raycast-указатель `GoPointer` (заметка 10/03).

### Основные методы
| Метод | Действие |
|-------|----------|
| `PickUpItem(item)` | `heldItem = item; item.held = this;` сброс поворота. Запускает плавную анимацию подъёма (`timerAfterPickup`). |
| `DropItem()` | Слой предмета → 0, `item.held = null`, `heldItem = null`. |
| `GetHeldItem()` | Текущий предмет в руках. |

### Удержание (`Update`, когда `heldItem != null`)
- **Позиция:** предмет держится у камеры — `camera.position + forward * holdDistance + up * holdHeight`, плавно доводится лерпом (`timerAfterPickup`).
- **Поворот колесом:** `OnScroll` крутит `heldRotationOffset` (или прямой поворот для крупных).
- **Размещение мебели** (мелкие, `allowPlacingItems`): `camera + forward*lookDistance + up*furniturePlaceHeight`.
- **Крупные предметы** учитывают декаллизию (выталкивание из стен).

### Коллизии и декаллизия (`PickupableItemCollisionChecker`)
На физическом объекте предмета. Считает коллизии (`collisions`) и считает вектор выталкивания через `Physics.ComputePenetration`:
- `GetDecollision()` — вектор, выталкивающий предмет из стен (только если `Settings.enableDecol`).
- `allowObstructedDropping` — можно бросить даже в коллизии, если проникновение `< 0.06`.
- **Красный контур** (`enableRedOutline`), если предмет в коллизии и его нельзя сюда положить (крупный, или зажата клавиша действия `InputName 8`).

## Ящики (`ShipItemCrate`) — контейнеры с лутом

`ShipItemCrate : ShipItem` — запечатанный ящик с `amount` копиями `containedPrefab`.

| Механика | Детали |
|----------|--------|
| `containedPrefab` | Префаб предмета внутри (например, сыр в ящике сыра). |
| `amount` | Число предметов внутри; `> 0` = **запечатан**, `0` = распечатан. |
| `UnsealCrate()` | Инстанцирует `amount` копий содержимого **за кадром** (на `+100.5` по Y, чтобы не показывать появление) и в следующем кадре **перекладывает их в инвентарь ящика** (`CrateInventory.InsertItem`); `amount → 0`; звук `crateSealBreak`; открывает UI ящика. **Ящик не исчезает** — остаётся контейнером. |
| `OnAltActivate()` | ПКМ: запечатан (`amount>0`, не миссия) → `CrateSealUI` (подтверждение вскрытия); распечатан (`amount<=0`) → `OpenCrate()` (открыть инвентарь ящика). |
| `OnLoad()` | `value = containedValue * amount + 20` (× 1.2 если `smokedFood`). |
| `lookText` | `"name (amount)"` (запечатан) или `"crate"` (пуст); для миссионных — текст миссии. |
| `crates` (static) | `Dictionary<int, CrateInventory>` — реестр ящиков. |

> **Как это ощущается в игре:** ящик — это товар/миссионный груз (запечатан, `amount>0`). Его **распечатывают** — содержимое не выпадает в мир, а **переходит в инвентарь ящика**, и ящик дальше используется как **многоразовый контейнер** (открыть/закрыть). Распечатанный ящик нельзя перевозить грузовым носителем (заметка 32: `"Cannot transport unsealed crates"`).

## 👁 Взгляд моддера: «хочу спавнить подбираемые предметы в мире»

**Минимальный путь (используя готовое):**
1. Создать префаб предмета: GameObject с `ShipItem` (+ подкласс), `Rigidbody`, `Collider`, `Renderer`, `SaveablePrefab` (с уникальным `prefabIndex` в `PrefabsDirectory.directory`). Плавучесть добавится сама (`SimpleFloatingObject`).
2. Поставить в мире `WorldItemSpawner`, указать `itemPrefab`, `respawnTime`, зарегистрировать индекс в `ItemSpawners.instance`. Готово — предмет будет респауниться при приближении игрока.

**Свой спавнер (полный контроль, например «рандомный лут по морю»):**
```csharp
var go = Object.Instantiate(myItemPrefab, worldPos, Quaternion.identity);
var item = go.GetComponent<ShipItem>();
item.sold = true;                              // иначе вернётся в «магазин»
go.GetComponent<SaveablePrefab>().RegisterToSave();   // сохранение
// позицию Y взять от поверхности воды:
//   OceanHeight.GetHeight(helper, worldPos)  (заметка 31)
// учесть плавающее начало координат:
//   worldPos = FloatingOriginManager.instance.RealPosToShiftingPos(realPos)  (заметка 11)
```
- **Подбор уже работает**: игрок наведётся (GoPointer raycast) и кликнет → `PickUpItem`. Ничего делать не нужно.
- **Плавание уже работает**: `ItemRigidbody` сам добавит `SimpleFloatingObject` с `_raiseObject = floaterHeight`.
- **Сохранение**: `RegisterToSave()` → предмет попадёт в `savedPrefabs` сейва (заметка 11) и переживёт загрузку.
- **Не спавнить вдали** от игрока (как делает `WorldItemSpawner`, порог 100 ед.) — и учитывать сдвиг мира (`outCurrentOffset`), иначе предмет «уедет».

## Практические выводы

1. **Спавн** = `WorldItemSpawner` (респаун по кулдауну, только в радиусе 100 ед.) + «ритуал» `Instantiate → sold=true → RegisterAsMissionless → freeze kinematic`.
2. **Кулдауны спавнеров** тикают от игрового времени и сохраняются в сейв.
3. **Все предметы плавают** через `SimpleFloatingObject` (`_raiseObject = floaterHeight`).
4. **Подбор** полностью обеспечивает `GoPointer` (`PickUpItem`/`DropItem`); удержание у камеры, поворот колесом, декаллизия, красный контур.
5. **Ящик** (`ShipItemCrate`) — готовый «лут-контейнер»: запечатан (`amount>0`); вскрытие **перекладывает содержимое в инвентарь ящика** (ящик остаётся многоразовым контейнером, не исчезает).
6. Для своего лута: префаб с `ShipItem`+`SaveablePrefab` → `Instantiate` + `sold=true` + `RegisterToSave` → подбор и плавание работают из коробки.
