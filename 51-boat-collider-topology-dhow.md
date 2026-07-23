# 51. Топология коллайдеров лодки (dhow): embark, walk, capsule, hull

Разбор дерева коллайдеров лодки и их назначения — ответ на запрос E1. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 14 (плавучесть), 16 (ShipItem/twin), 47 (удержание).

## `BoatEmbarkCollider` — главный триггер посадки/высадки

`BoatEmbarkCollider : MonoBehaviour` — компонент на **дочернем GO лодки**, который служит триггером `OnTriggerEnter/Exit` по тегу `EmbarkCol` для предметов (`ShipItem`) и по тегу `MainCamera` для игрока (`BoatEmbarkTrigger`).

### Ключевые поля
| Поле | Тип | Содержание |
|------|-----|-----------|
| `walkCollider` | `Transform` | **Ссылка на walk-коллайдер лодки** — Transform отдельного GO (BoxCollider/MeshCollider?), по которому ходит игрок и к которому репарентится twin предмета при `EnterBoat`. |
| `damage` | `BoatDamage` | Берётся в `Awake()`: `transform.parent.parent.GetComponent<BoatDamage>()`. → `BoatEmbarkCollider` находится на **вложенном дочернем** (parent = mesh group, parent.parent = корень лодки). |
| `col` | `Collider` | Коллайдер этого GO (тот, что тег `EmbarkCol`). |
| `capsuleCol` | `CapsuleCollider` | **CapsuleCollider на корне лодки** (`transform.parent.parent.GetComponent<CapsuleCollider>()`). Это главный «объём корпуса». |
| `initialRadius` | `float` | Исходный радиус `CapsuleCollider` лодки. |
| `embarkAllowed` | `bool` | Флаг: если лодка потоплена (`damage.sunk`) → триггер смещается на +100 по Y (отключает посадку). |

### `ToggleBoatCapsuleCol(bool newState)`
- `newState = true` → `capsuleCol.radius = initialRadius` (восстановить);
- `newState = false` → `capsuleCol.radius = 0` (отключить CapsuleCollider лодки).

Используется при потоплении / восстановлении лодки.

### `LateUpdate` — sinking teleport
- Если `sunk && embarkAllowed`: `transform.Translate(up * 100, Space.World)` → триггер улетает вверх, `embarkAllowed = false`.
- Если `!sunk && !embarkAllowed`: `transform.localPosition = initialPos` → триггер возвращается.

## `CapsuleCollider` на корне лодки — главный объём корпуса

`BoatEmbarkCollider.capsuleCol` = `CapsuleCollider` на **корневом GO лодки** (transform.parent.parent). Это тот самый «упрощённый объём», который:

- Охватывает **всю лодку** как капсулу (с initialRadius, который может быть ~1.5–3 м, и height ~8–12 м).
- Используется `Buoyancy` для расчёта блоб-плавучести (сетки `SlicesX × SlicesZ` по размеру этого `CapsuleCollider`, заметка 14).
- Игрок **игнорирует** его для ходьбы (player ходит по отдельному `walkCollider`, слой 12).
- **Но предметы (twin ItemRigidbody, слой 2) его НЕ игнорируют** — CapsuleCollider лодки находится на слое, который сталкивается со слоем 2 (ItemRigidbody), и это объясняет, почему бочка «встаёт на воздухе» над трюмом: её twin физически взаимодействует с поверхностью этой капсулы, а не с палубой.

> **Критичный вывод:** `CapsuleCollider` лодки — это **упрощённый объём корпуса**, поверхность которого выше палубы. Игрок его не «видит» (ходит по `walkCollider`/слой 12), но предметы с physics twin (слой 2) сталкиваются с ним. Это — источник «невидимой стены над трюмом» для мода с твёрдым коллайдером предмета в руке.

## `walkCollider` — поверхность ходьбы

`BoatEmbarkCollider.walkCollider` — Transform отдельного GameObject с коллайдером палубы (BoxCollider или MeshCollider). Этот GO:

- **Слой 12** (по `HullPlayerCollider.Start()` — создаёт дочерний `hull player collider` на слое 12).
- `CharacterController` игрока ходит по нему (player capsule сталкивается со слоем 12).
- При `ItemRigidbody.EnterBoat()` twin **репарентится** на `walkCollider` (`transform.parent = item.currentWalkCol`) и позиционируется через `MoveRigidbodyToWalkCol`.

`HullPlayerCollider : MonoBehaviour` (`[RequireComponent(MeshCollider)]`) в `Start()`:
- Создает **дочерний GO** «hull player collider» с MeshCollider (копия sharedMesh), слой **12**.
- **Отключает** собственный MeshCollider на хост-GO (`enabled = false`).
- Дочерний GO — **не дочерний трансформу лодки** (`parent = null`), но позиция/rotation синхронизируется каждый `Update()`.
- Также добавляет `CleanableObjectCollider` (для чистки корпуса).

> `HullPlayerCollider` создаёт **отдельный физический дубликат mesh-коллайдера** для ходьбы/чистки на слое 12, отключая исходный mesh на лодке. Но **CapsuleCollider лодки остаётся активным** (на слое лодки) и сталкивается с предметами.

## `BoatEmbarkTrigger` — триггер посадки игрока

Отдельный компонент `BoatEmbarkTrigger : MonoBehaviour` на другом дочернем GO лодки:

- `OnTriggerEnter(Collider other)`: если `other.CompareTag("MainCamera")` → `EnterBoat()` — репарентит «Player Controller» на лодку.
- `OnTriggerExit(Collider other)`: если `other.CompareTag("MainCamera")` → `ExitBoat()` — отцепляет игрока от лодки.

Это **другой** объём, чем `BoatEmbarkCollider` (для предметов). Оба имеют тег `EmbarkCol`.

## Иерархия GO лодки (реконструкция dhow)

```
Dhow (root GO)                                  ← Rigidbody + CapsuleCollider (корпус-капсула) + BoatDamage + Buoyancy + BoatMass + BoatProbes + ShiftingRigidbody
 ├── BoatProbes child                           ← (blob-buoyancy точки)
 ├── BoatKeel                                   ← centerOfMass
 ├── CapsuleCollider (on root)                  ← главный объём, radius ≈ initialRadius, сталкивается с twin-предметами!
 ├── Mesh group (parent of BoatEmbarkCollider)  ← MeshRenderer корпуса
 │    ├── BoatEmbarkCollider                    ← Collider с тегом EmbarkCol (для предметов), ссылка на walkCollider, ссылка на CapsuleCollider root
 │    ├── HullPlayerCollider                    ← MeshCollider (disabled), создаёт "hull player collider" (слой 12) дубликат
 │    └── ... (walkCollider?)                   ← walkCollider = Transform отдельного GO палубы (BoxCollider/MeshCollider, слой 12)
 ├── BoatEmbarkTrigger                          ← Collider с тегом EmbarkCol (для игрока, MainCamera)
 ├── BoatImpactSoundsCollider                   ← OnCollisionEnter → BoatImpactSounds.Impact
 ├── BoatMooringRopes                           ← швартовы
 ├── ... (Deck objects, masts, sails)
```

## `ShiftingRigidbody` лодки

`ShiftingRigidbody` на корне лодки:
- `Rigidbody` лодки — **не кинематический** (динамический, массой вычисленной `BoatMass`).
- `ShiftingRigidbody` управляет сохранением velocity/angularVelocity при сдвиге мира (`FloatingOriginManager`): на shift — `isKinematic = true` + sleep, после — restore momentum.
- `boatProbes` (BoatProbes) — `stopProbes` при shifting; `dontUpdateVelocity = true/false`.
- Коллайдеры лодки: `CapsuleCollider` (root) + MeshCollider (отключён, дубликат на слое 12) + `BoatEmbarkCollider` (trigger, тег EmbarkCol) + `BoatEmbarkTrigger` (trigger, тег EmbarkCol) + `BoatImpactSoundsCollider` (non-trigger, OnCollisionEnter).

## Практические выводы для мододела

1. **CapsuleCollider лодки** — главный «упрощённый объём» на корневом GO. Его поверхность **выше палубы**; игрок его игнорирует (ходит по `walkCollider`/слой 12), но twin-предмет (слой 2) сталкивается с ним. Это — прямое объяснение бага «бочка лежит на воздухе над трюмом».
2. **`BoatEmbarkCollider.walkCollider`** — Transform отдельного GO палубы (слой 12). Twin предмета репарентится на него при `EnterBoat`.
3. **Два разных EmbarkCol-объёма**: один для предметов (`BoatEmbarkCollider`), другой для игрока (`BoatEmbarkTrigger`). Тег один, GO разные.
4. **При потоплении** CapsuleCollider отключается (`radius = 0` через `ToggleBoatCapsuleCol(false)`), и `BoatEmbarkCollider` улетает на +100Y.
5. **`HullPlayerCollider`** создаёт дубликат mesh-коллайдера на слое 12 (для ходьбы/чистки), отключая исходный mesh — **но CapsuleCollider root остаётся активным и сталкивается с предметами**.
6. Для мода с твёрдым коллайдером предмета: twin-предмет на слое 2 сталкивается с CapsuleCollider лодки. Решение — либо игнорировать слой лодки для twin при холде, либо сделать CapsuleCollider лодки trigger-only для предметных коллизий, либо использовать `Physics.IgnoreLayerCollision`.
