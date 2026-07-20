# 16. Фреймворк предметов: ShipItem и семейство

Разбор системы предметов (всё, что можно подобрать, купить, положить в инвентарь/трюм). Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. База для модов на инвентарь, новые предметы, торговлю.

## Иерархия классов

```
GoPointerButton            (интерактивный объект, см. заметку 10)
  └─ PickupableItem        (всё подбираемое)
       └─ ShipItem         (базовый «корабельный» предмет, [RequireComponent Rigidbody+Collider+Renderer])
            └─ 35 подклассов (еда, инструменты, мебель, напитки…)
```

### `PickupableItem : GoPointerButton`
| Поле | По умолч. | Назначение |
|------|:--:|-----------|
| `big` | — | Крупный (двуручный) предмет. |
| `holdDistance` | 1.15 | Дистанция удержания от камеры. |
| `furniturePlaceHeight` | 0.15 | Высота размещения мебели. |
| `holdHeight` | — | Вертикальное смещение в руках. |
| `held` | — | `GoPointer`, который держит предмет. |
| `heldRotationOffset` | — | Поворот удерживаемого предмета; `OnScroll(input)` крутит на `input * 3` (колесо мыши). |
| `colChecker` | — | `PickupableItemCollisionChecker`. |

Виртуальные хуки: `OnPickup()`, `OnDrop()`, `OnScroll()`.

## `ShipItem` — базовый предмет

### Основные поля
| Поле | По умолч. | Содержание |
|------|:--:|-----------|
| `mass` | 1 | Масса (влияет на `BoatMass`, см. заметку 14). |
| `value` | — | **Базовая цена** товара (используется рынком, см. заметку 13). |
| `name` | — | Имя предмета (используется в `lookText`, миссиях). |
| `category` | — | `TransactionCategory` — категория сделки. |
| `inventoryScale` / `inventoryRotation(X)` | 1 / 0 | Как предмет выглядит в слоте инвентаря. |
| `floaterHeight` | 1.6 | Высота поплавка (плавучесть в воде). |
| `sold` | false | Куплен ли. В магазине — `false`; покупка → `Sell()` → `true`. |
| `nailed` | false | Прибит/закреплён (не двигается). |
| `health` | — | Прочность/состояние (у еды — свежеть, у жидкостей — объём). |
| `amount` | — | Количество (стопки, объём). |
| `daysInStorage` | — | Дней в хранилище (портится). |
| `wallAttachment` | false | Можно вешать на стену (raycast-крепление). |

### Двух-объектная архитектура физики
**Каждый предмет — это два GameObject:**
1. **Основной** (`ShipItem`) — визуал (`Renderer`), триггерные коллайдеры (`isTrigger = true`), LOD.
2. **`itemRigidbody`** — отдельный runtime-объект с реальным физическим телом (`ItemRigidbody`), создаётся в `CreateRigidbody()` как дочерний «мира» (`FloatingOriginManager`).

Почему: физическое тело живёт в системе координат мира и переиспользуется при входе/выходе с лодки, помещении в инвентарь и т.п. `Rigidbody.collisionDetectionMode = 3` (ContinuousDynamic). В `Awake()` все коллайдеры основного объекта переводятся в триггеры.

`ItemRigidbody` дополнительно:
- держит `SimpleFloatingObject` (`floater`) — предметы **плавают** в воде;
- состояния `onBoat`, `inStove`;
- `idleTimer` / `dynamicColTimer` — оптимизация: «усыпление» физики неиспользуемых предметов.

### `ItemRigidbody` и размещение
- **Инвентарь:** слот `GPButtonInventorySlot` (`slotIndex`).
- **Грузовой носильщик** (`CargoCarrier`): `GetCurrentInventorySlot()` возвращает `portIndex + 100` (см. заметку 11 про `inventorySlot >= 100`).
- **Крепление к стене** (`wallAttachment`): raycast вперёд на 1.3 м; при попадании — вычисляется `attachPos`/`attachRot`, предмет вешается (`HangableItem`).

### Вход/выход с лодки
Предмет детектит триггер с тегом `EmbarkCol` (`BoatEmbarkCollider`). После `frameCounter > порога`:
- **EnterBoat:** предмет репарентится на лодку, `saveable.SetParentObject(sceneIndex лодки)`, `ItemRigidbody.EnterBoat()`.
- **ExitBoat:** репарент на мир, `SetParentObject(-1)`, `HangableItem.DisconnectJoint()`.

### Специальные коды родительского объекта (`saveable.GetParentObject()`)
`ProcessSaveable()` обрабатывает магические значения `parentObject`:

| Код | Значение |
|:--:|----------|
| `-1` | В мире (не на лодке). |
| `-2` | **Уничтожить предмет** (`DestroyItem()`). |
| `-3` | **Потерян** (cargo-loss при обмороке): слой = 2 (`IgnoreRaycast`), уничтожается при `GameState.recovering && eyesFullyClosed`. Это реализация «потери груза» из `Recovery.GetCargoLossChance` (заметка 12). |

### Покупка/продажа
- `AddToShop(shop)`: запоминает `shopPos`/`shopRot`, добавляет в `shop.itemsForSale`.
- `Sell()`: `sold = true`, репарент на мир, `RegisterToSave()`, подбирается игроком, `OnBuy()`.
- `ReturnToShopPos()`: если предмет не куплен и брошен — плавно (лерп 2/c) возвращается на магазинную полку.
- `OnAltActivate()`: ПКМ по некупленному товару → `shopkeeper.TryToSellItem(this)` (попытка продать магазину).

### `DestroyItem()`
Снимает предмет с лодки, `SaveablePrefab.Unregister()` (убирает из сохранения), `Destroy(gameObject)`. Вызов без `SaveablePrefab` логируется как `CRITICAL ERROR`.

## Семейство ShipItem (35 подклассов)

| Категория | Классы |
|-----------|--------|
| **Еда/питьё** | `ShipItemFood`, `ShipItemSoup`, `ShipItemTea`, `ShipItemKettle`, `ShipItemBottle`, `ShipItemSalt` |
| **Приготовление** | `ShipItemStove`, `ShipItemStoveFuel`, `ShipItemKnife` |
| **Инструменты** | `ShipItemHammer`, `ShipItemBroom`, `ShipItemOakum` (конопатка), `ShipItemOar` (весло) |
| **Навигация** | `ShipItemCompass`, `ShipItemClock`, `ShipItemQuadrant`, `ShipItemSpyglass`, `ShipItemChipLog` |
| **Освещение** | `ShipItemLight`, `ShipItemLanternFuel`, `ShipItemLampHook` |
| **Отдых/привычки** | `ShipItemBed`, `ShipItemPipe` (трубка), `ShipItemTobacco`, `ShipItemElixir`, `ShipItemRandomElixir` |
| **Хранение/груз** | `ShipItemCrate` (ящик), `ShipItemFoldable` (складное, карты) |
| **Рыбалка** | `ShipItemFishingRod`, `ShipItemFishingHook` |
| **Прочее** | `ShipItemHangable`, `ShipItemInkSet` (чернила), `ShipItemScroll` (свиток), `ShipItemTotem` |

Подклассы переопределяют `OnLoad()`, `OnBuy()`, `OnEnterInventory()`, `UpdateLookText()` и хранят спец-состояние (у супа — `currentWater/currentEnergy/...`, у чайника — `currentTeaType/Amount` и т.д.), которое сериализуется через `extraValue0..4` (см. заметку 11).

## `TransactionCategory` (enum)

Категория операции — используется в экономике/журнале доходов (`DayLogs`, `TransactionCategory`):

`retailFood, retailWater, retailAlco, toolsAndSupplies, furniture, otherItems, bulkFood, bulkWater, bulkAlco, bulkGood, mission, boat, recovery, other, currencyExchange`

`ShipItem.IsBulk()` = true для `bulkAlco/bulkFood/bulkWater/bulkGood` (оптовые/трюмные товары).

## Практические выводы для мододела

1. **Новый предмет** = GameObject с `ShipItem`(+подкласс), `Rigidbody`, `Collider`, `Renderer`, `SaveablePrefab` (с уникальным `prefabIndex` в `PrefabsDirectory.directory`). Без `SaveablePrefab` предмет не сохраняется и ломает `DestroyItem`.
2. **Физика отделена** от визуала (`itemRigidbody`) — учитывайте при спавне/телепортации предметов.
3. **`value`** — базовая цена для рынка; **`mass`** — вклад в массу лодки.
4. **`sold`** отличает магазинный товар от купленного; **`nailed`** — закреплённый.
5. **Коды `parentObject`** `-1/-2/-3` управляют миром/уничтожением/потерей груза — не используйте эти значения вслепую.
6. **Инвентарь:** слот `< 100`; грузовой носильщик = `portIndex + 100`.
7. Подсказка миссионного товара — `"{name}\nto {port}\ndue: {text}"` (захардкожено, EN).
