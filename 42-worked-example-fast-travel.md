# 42. Рабочий пример: фаст-тревел / телепорт по точкам

Второй учебный пример (первый — заметка 34, лут-ящики). Задача: **«игрок может телепортироваться в сохранённые точки/порты»**. Упражняет хитрые места: плавающее начало координат, реестр портов, управление игроком, персистентность между сессиями. Связывает заметки 11, 19, 30, 36, 39.

## Что уже даёт игра

| Нужно | Берём | Заметка |
|-------|-------|---------|
| Все порты и их позиции | `Port.ports[34]`, `port.portIndex`, `port.GetPortName()`, `transform.position` | 19 |
| Позиция игрока | `Refs.observerMirror.transform.position` | 30 |
| Координаты (мир ↔ сцена) | `FloatingOriginManager.instance.ShiftingPosToRealPos / RealPosToShiftingPos`, `outCurrentOffset` | 11 |
| Включить/выключить управление | `Refs.SetPlayerControl(bool)` | 30 |
| Затемнение экрана | `Blackout.FadeTo(target, duration)` (корутина) | 25 |
| Данные мода между сессиями | `GameState.modData` | 11 |
| Плавная анимация/звук | `Juicebox.juice`, `UISoundPlayer` | 39, 35 |

## Ключевая сложность: плавающее начало координат

Мир сдвигается (`outCurrentOffset`), поэтому **нельзя** просто телепортировать в «абсолютную» позицию. Есть два корректных подхода:

**A. Телепорт к существующему объекту (порт)** — безопасно:
```csharp
// позиция порта уже в актуальных координатах сцены — просто берём её
Vector3 dest = Port.ports[index].transform.position + Vector3.up * 2f;
```

**B. Телепорт в «свою» сохранённую точку** — храним в **реальных** координатах, применяем в **сцене**:
```csharp
// сохранение (реальные координаты, не зависят от сдвига мира):
Vector3 real = FloatingOriginManager.instance.ShiftingPosToRealPos(player.position);
SaveRealPos(real);   // в GameState.modData

// применение (обратно в координаты сцены с учётом текущего сдвига):
Vector3 scene = FloatingOriginManager.instance.RealPosToShiftingPos(LoadRealPos());
```

> Грабли: если сохранить позицию в координатах сцены, а применить после сдвига мира — игрок улетит на километры. **Хранить — реальные, применять — сцены.**

## Кого двигать

Игрок — это `CharacterController`/`OVRPlayerController` (`Refs.ovrController`, `Refs.charController`), а не камера. Телепорт через `transform.position` контроллера:

```csharp
IEnumerator TeleportTo(Vector3 scenePos)
{
    if (GameState.sleeping || GameState.recovering || GameState.currentShipyard != null)
        yield break;                       // не телепортируем в этих состояниях

    Refs.SetPlayerControl(false);          // выключить управление
    yield return Blackout.FadeTo(1f, 0.5f); // затемнить

    // двигаем контроллер игрока:
    Refs.ovrController.transform.position = scenePos;
    // observerMirror подтянется сам (или форсируем):
    Refs.observerMirror.transform.position = scenePos;

    yield return Blackout.FadeTo(0f, 0.5f); // осветлить
    Refs.SetPlayerControl(true);           // вернуть управление
}
```

**Грабли:**
- `CharacterController` иногда нужно «выключить/включить» или двигать через `controller.Move`/`transform.position` при неактивном контроллере — иначе телепорт может не примениться (особенность Unity `CharacterController`).
- Если игрок **на лодке** (`GameState.currentBoat != null`) — телепорт игрока без лодки оставит лодку позади; решите, телепортировать ли лодку тоже (её `Rigidbody`/`SaveableObject`) или высаживать игрока.
- После телепорта в воду игрок начнёт плавать (`PlayerSwimming`, заметка 36) — это нормально, но проверьте высоту Y (ставьте чуть выше воды/палубы).

## Персистентность точек (свои данные)

Храним список открытых точек в `GameState.modData` (автосохранение, заметка 11):

```csharp
const string KEY = "MyFastTravel.points";

void SavePoints(List<FastPoint> pts) {
    // сериализуем сами (JSON/CSV) — modData хранит только string
    GameState.modData[KEY] = string.Join(";", pts.Select(p =>
        $"{p.name}|{p.realPos.x},{p.realPos.y},{p.realPos.z}"));
}

List<FastPoint> LoadPoints() { /* парсим GameState.modData[KEY] */ }
```

> `modData` — `Dictionary<string,string>`; сложные структуры сериализуйте в строку (JSON, `key;key`). Ключи — с префиксом мода, чтобы не конфликтовать.

## Точки = порты (простой вариант)

Можно вообще не хранить позиции, а использовать **порты как точки фаст-тревела**:
- Список точек = `Port.ports` (где `!= null`), имя = `GetPortName()`.
- Телепорт = `Port.ports[i].transform.position`.
- «Открытые» порты = те, где игрок бывал (`GameState.lastVisitedPort`, заметка 12) или свой флаг в `modData`.
- Плюс: позиции портов всегда актуальны (не нужно возиться с координатами/сдвигом).

## Чек-лист фичи

- [x] Точки: порты (`Port.ports`) ИЛИ свои (хранить в **реальных** координатах в `modData`).
- [x] Телепорт: `Refs.SetPlayerControl(false)` → `Blackout.FadeTo` → двигать `Refs.ovrController` (+ observerMirror) → fade back → `SetPlayerControl(true)`.
- [x] Координаты: применить через `RealPosToShiftingPos` (для своих точек); порты — напрямую.
- [x] Не телепортировать во сне/восстановлении/верфи.
- [x] Обдумать лодку игрока (`GameState.currentBoat`) и высоту Y.
- [x] UI выбора точки — свой IMGUI/`OnGUI` (как у переводчика) + `Juicebox`/`UISoundPlayer` для полировки.

## Обобщение: рецепт «телепорт/перемещение игрока»

1. **Куда:** объект в сцене (порт — напрямую) ИЛИ своя точка (реальные координаты → `RealPosToShiftingPos`).
2. **Как:** выключить управление → блэкаут → `Refs.ovrController.position = dest` → блэкаут назад → включить управление.
3. **Ограничения:** не в сне/восстановлении/верфи; решить вопрос с лодкой и высотой.
4. **Персистентность:** свои точки — в `GameState.modData` (сериализация в строку).

Тот же скелет покрывает: «return to boat», «recall to last port», телепорт к квестовым меткам, дебаг-варп и т.п.
