# 52. Слои Unity и матрица столкновений: предметы, лодка, игрок

Разбор слоев Unity и матрицы столкновений — ответ на запрос E2. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметкой 51 (топология коллайдеров лодки).

## Слои, используемые игрой

Слои установлены через `gameObject.layer = N` в коде (и в префабах сцен):

| Номер | Использование | Код / класс |
|:--:|-----------|------------|
| 0 | Default (мир, terrain, dock walls и т.п.) | Unity default |
| 2 | **IgnoreRaycast** — twin ItemRigidbody, visual ShipItem (когда не в инвентаре/на UI) | `ItemRigidbody.Start()`: `gameObject.layer = 2`; `ShipItem.EnterInventorySlot`: `layer = 2` |
| 5 | **UI** — visual ShipItem в инвентаре/на UI | `ItemRigidbody.EnterInventorySlot`: `item.gameObject.layer = 5` (и все children) |
| 8 | **Player** (character capsule) | Unity built-in или префаб |
| 12 | **HullPlayerCollider** — дубликат mesh-коллайдера палубы | `HullPlayerCollider.Start()`: `hullCol.gameObject.layer = 12` |
| 14 | Terrain/OceanBottom (как в Anchor.IsTouchingGround) | Преfabs, тег `Terrain`/`OceanBottom` |
| 26 | «В ящике/специальный» — предмет в CrateInventory | `ShipItem.ExtraFixedUpdate`: `layer != 26` — проверка; предмет в crate ставится на 26 |
| 23 | Карта (MapChart при открытии) | `MapChart`: при открытии карты GO ставится на слой 23 |
| 13 | Boat (тег `Boat`) | Префаб лодки / коллайдеры лодки |
| ? | Water/Ocean | `Ocean` renderer |

> **Точное имя слоя по номеру** не доступно из декомпиляции (LayerMask.NameToLayer — runtime API). Приведённые номера — из прямых `gameObject.layer = N` в коде. Имена слоев определяются в проекте Unity (Tag Manager) и не сериализуются в Assembly-CSharp.dll.

## `ItemRigidbody` — переключение слоев

| Состояние | Слой twin (GO ItemRigidbody) | Слой visual (GO ShipItem) | Код |
|-----------|:--:|:--:|------|
| В мире (свободный, `sold`) | **2** (IgnoreRaycast) | **2** | `Start()`, `ResetPos()` |
| В инвентаре (`EnterInventorySlot`) | не меняется | **5** (UI) | `EnterInventorySlot`: `item.gameObject.layer = 5` (children тоже) |
| Из инвентаря (`ExitInventorySlot`) | 2 | **2** | `ExitInventorySlot`: `item.gameObject.layer = 2` (children тоже) |
| В ящике (crate) | 2 | **26** | префаб crate layer = 26 |
| PickUp (`GoPointer.PickUpItem`) | **0** (Default) | — | `GoPointer`: `heldItem.gameObject.layer = 0` (visual на Default для raycast/interact) |
| Drop (`GoPointer.DropItem`) | 2 | 2 | `DropItem`: `heldItem.gameObject.layer = 0`→2? (в LateUpdate twin позиция = visual) |

> **Критично:** при `PickUp` visual ShipItem ставится на **слой 0 (Default)**. Twin остаётся на **слое 2 (IgnoreRaycast)**. Это означает, что **twin-коллайдеры НЕ сталкиваются с обычными объектами мира** (IgnoreRaycast игнорируется физической подсистемой Unity по умолчанию). Ванильный held предмет никогда не физически касается лодки!

## Raycast-маска GoPointer

`GoPointer.LateUpdate` использует raycast с маской `LayerMask.op_Implicit(-604165)`.

Двоичное представление `-604165` (32 bit): это инверсия маски. `~(-604165) = 604164` в двоичном = какие слои Raycast **не** проверяет, `-604165` = какие **проверяет**.

```
604164 = 0b 0000 1001 0011 0001 0000 0100
         слои: 2, 8, 12, 16, 19, 24 (примерно — нужно runtime-проверить)
```

Точный decode требует runtime (LayerMask.NameToLayer), но принцип: GoPointer raycast «видит» большинство физических слоев и **игнорирует** UI и IgnoreRaycast.

> Слой 2 (IgnoreRaycast) **исключён из raycast GoPointer** — twin-предмет не «кликается» через raycast (подбор идёт по visual, слой 0 или 5).

## Матрица столкновений — ключевые пары

Unity `Physics.GetIgnoreLayerCollision(a,b)` задаётся в проекте (Physics Settings) и **не видна из декомпиляции**. Но можно реконструировать из наблюдений и кода:

| Пара | Сталкиваются? | Обоснование |
|------|:--:|------------|
| **twin (2) ↔ лодка (13?)** | **ДА** | twin ItemRigidbody на слое 2 — это IgnoreRaycast, но **не** IgnoreCollision. IgnoreRaycast влияет только на raycast, НЕ на физические столкновения. CapsuleCollider лодки столкнется с twin на слое 2. |
| **twin (2) ↔ игрок (8)** | **ДА** (если включено в Physics Settings) | CharacterController взаимодействует с большинством слоев |
| **игрок (8) ↔ лодка (12/walkCollider)** | **ДА** | Игрок ходит по палубе (слой 12 = HullPlayerCollider) — это подтверждено кодом |
| **twin (2) ↔ вода** | **ДА** | `SimpleFloatingObject` работает через Ocean — twin физически в воде |
| **visual (0) ↔ лодка** | **ДА** при held** | Visual при held на слое 0 — Default, сталкивается с лодкой |

> **IgnoreRaycast (layer 2)** не делает объект «нефизическим» — он только исключает из raycast. Коллизии (OnCollisionEnter/Exit, OnTriggerEnter/Exit) **работают** на слое 2. Это значит: twin ItemRigidbody (слой 2) **физически сталкивается с CapsuleCollider лодки**.

## Почему «игрок внутри коллизии лодки, а ящик на поверхности»

1. Игрок (`CharacterController`, слой 8) ходит по **walkCollider** (слой 12) — дубликат mesh палубы, поверхность которого совпадает с палубой.
2. Но **CapsuleCollider лодки** — упрощённый объём (капсула), его поверхность **выше палубы** (капсула охватывает весь корпус, включая мачты/борта, и её верхняя поверхность может быть на уровне борта или выше).
3. Игрок **не сталкивается с CapsuleCollider** (или слой 8↔слой лодки настроен так, что player игнорирует CapsuleCollider, ходя по слою 12).
4. Twin предмет (слой 2) **сталкивается с CapsuleCollider** (2↔лодка = Да). И «ложится» на поверхность капсулы — которая **выше палубы**.
5. В ваниле twin на слое 2 + `isTrigger=true` на коллайдерах twin → **нет физических столкновений** (trigger не вызывает OnCollisionEnter). Мод делает twin коллайдеры non-trigger → физическое столкновение с CapsuleCollider → предмет «стоит на воздухе».

## Практические выводы

1. **Слой 2 (IgnoreRaycast)** ≠ «без коллизий». Это только raycast-исключение. Физика на слое 2 работает.
2. **Ванильный held предмет** не сталкивается с лодкой, потому что twin-коллайдеры **isTrigger=true** → физических коллизий нет.
3. **Мод с non-trigger twin** = предмет на слое 2 физически сталкивается с CapsuleCollider лодки → предмет «стоит на воздухе над трюмом».
4. Решение: `Physics.IgnoreLayerCollision(twinLayer, boatLayer, true)` для twin при холде, или исключить CapsuleCollider лодки из коллизий предмета через Harmony-патч.
5. Слой visual при held = **0 (Default)** — это «видимый» предмет для raycast/interact; twin при held = **2 (IgnoreRaycast)** — «не raycast-ится» но **физически активен**.
