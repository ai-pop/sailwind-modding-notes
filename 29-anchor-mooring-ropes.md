# 29. Якорь, швартовка и тросы

Разбор механик якоря и швартовки. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Якорь (`Anchor : PickupableItem`)

`[RequireComponent(ConfigurableJoint, Rigidbody, CapsuleCollider)]`. Якорь соединён с лодкой через `ConfigurableJoint` (трос = лимит соединения).

| Поле | По умолч. | Содержание |
|------|:--:|-----------|
| `anchorDrag` | 5 | Сопротивление якоря (в воде). |
| `anchorDragInWater` | 4 | Сопротивление в воде. |
| `anchorDragUp` | 1 | Минимальное сопротивление (при подъёме). |
| `unsetResistance` | 150 | Порог силы, при которой якорь срывает. |
| `resistancePullLimit` | 200 | Предел натяжения для «можно тянуть». |
| `set` (private) | — | **Закреплён ли якорь** (зацепился за дно). |

### Сопротивление якоря масштабируется массой лодки
`RegisterRopeController`:
```
unsetResistance = connectedBody.mass * 3.3 * rope.resistanceMult
resistancePullLimit = unsetResistance
```
Чем тяжелее лодка, тем сильнее держит якорь.

### Автозакрепление (`SetAnchor`)
Якорь **сам цепляется за дно**, когда (в `ExtraFixedUpdate`):
- не закреплён (`!set`), **касается грунта** (`IsTouchingGround` — коллизии с тегами `Terrain`/`OceanBottom` или слоем 14),
- наклонён правильно (`Angle(anchor.forward, up) > 60°`),
- трос выпущен достаточно (`linearLimit > 9`).

При закреплении: `body.isKinematic = true`, `drag = anchorDrag`, звук.

### Автосрыв (`ReleaseAnchor`)
Якорь **срывается**, когда закреплён и:
- угол троса от вертикали большой и сила натяжения превышает `unsetResistance * InverseLerp(10,60,angle)` (сильный горизонтальный рывок выдёргивает якорь);
- либо если угол ≥ 60° и расстояние < 3 м — `canPull = false` (якорь «на переломе»);
- либо трос выбран слишком сильно (`linearLimit <= 8`).

При срыве: `isKinematic = false`, звук (питч 1.44).

### Управление длиной троса
- Когда якорь в руках (`held`), трос **автоматически травится**: `currentLength` растёт с дистанцией; красный контур при > 85%; при `currentLength >= 1` — сброс (`DropItem`).
- `GetRopeLength() = rope.currentLength`; `IsSet() = set`.

### Динамика массы
- Трос короток (`limit < 1`): масса якоря падает до 0.2 (легко поднять), центр масс назад.
- Трос выпущен: масса возвращается к `initialMass`, центр масс `(0.2, 0, -0.2)`.

### Персистентность
`set` → `SaveableObject.extraSetting`, длина троса → `extraValue` (заметка 11). `OnLoad(isSet, ropeLength)` восстанавливает.

## Швартовка (`BoatMooringRopes`)

Управление швартовыми тросами лодки.

| Поле | Содержание |
|------|-----------|
| `anchor` | Якорь лодки. |
| `ropes[]` (`PickupableBoatMooringRope[]`) | Швартовы. |
| `mooringSet` | Набор точек швартовки. |
| `mooringFront` / `mooringBack` | Носовая/кормовая точка швартовки. |
| `anchorController` (`RopeControllerAnchor`) | Контроллер якорного троса. |

### Ключевые методы
- **`AnyRopeMoored()`:** `true`, если якорь закреплён (`anchor.IsSet()`) **или** любой швартов пришвартован. Используется `Sleep` (timeskip-сон, заметка 25) и `Recovery`.
- **`UnmoorAllRopes()`:** отдаёт все швартовы + сбрасывает позиции.
- **`MoorClosestRope(mooring)`:** находит ближайший швартов в исходной позиции и переносит его к швартовной точке (если док-муринг `GPButtonDockMooring` ещё не занят).
- **Авто-швартовка в `Start()`:** если заданы `mooringFront`/`mooringBack` — автоматически швартует к ним (используется `Recovery.RecoverBoat` для постановки лодки в порту, заметка 12).

### Доступность швартовов
Швартовы **доступны для raycast только у купленных лодок** (`boatSaveable.extraSetting`): у некупленных лодок тросы переводятся на слой 2 (IgnoreRaycast) — их нельзя схватить, пока лодка не куплена.

## Семейство контроллеров тросов (`RopeController*`)

Разные контроллеры для разных тросов:

| Класс | Назначение |
|-------|-----------|
| `RopeControllerAnchor` | Якорный трос (`currentLength`, `currentResistance`, `canPull`, `resistanceMult`). |
| `RopeControllerSailAngle` / `…Jib` / `…Square` | Угол паруса (шкоты). |
| `RopeControllerSailReef` | Рифление паруса. |
| `RopeControllerSteeringWheel` | Штурвал. |
| `RopeControllerMirror` | Зеркальный (NPC-лодки). |
| `RopeTrimControllerOld` | trim паруса (легаси). |

Тросы также: `SimpleRope`, `ClothRope`, `HangingRope`, `ChipLogRopeEnd` (визуал/физика тросов).

## Практические выводы для мододела

1. **Якорь сам цепляется** за дно при правильном наклоне и достаточной длине троса; **сам срывается** при сильном горизонтальном рывке.
2. **Сила удержания якоря** = `масса лодки × 3.3 × resistanceMult` — тяжёлые лодки держатся лучше.
3. **`AnyRopeMoored()`** (якорь или швартов) гейтит timeskip-сон и влияет на восстановление — удобно проверять для своей логики.
4. **Швартовы некупленных лодок** на слое 2 (нельзя взаимодействовать) до покупки (`extraSetting`).
5. Состояние якоря (`set`, длина троса) и швартовка сохраняются в сейв (заметка 11); авто-швартовка к `mooringFront/Back` используется recovery.
6. Управляющие тросы (`RopeController*`) — точки для модов управления парусами/рулём/якорем.
