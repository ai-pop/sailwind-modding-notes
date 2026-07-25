# 40. Грязь корпуса и чистка

Разбор механики обрастания/загрязнения корпуса и его чистки. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с сохранением текстуры (заметка 11, `extraTexture`), дневным тиком (заметка 18, `Sun.OnNewDay`) и состоянием корпуса (заметка 14, `BoatDamage`).

## Как устроено

Корпус лодки имеет **текстуру-оверлей грязи** (второй материал `dirtMaterial` с `mainTexture`). Грязь — это не число, а **реально нарисованная текстура** (128×128), которая копится и стирается «кистью».

```
CleanableObject (на корпусе)
   └─ dirtMaterial.mainTexture  ← текстура грязи
        └─ меняется через MasterPainter (ApplyCoat / PaintObject)
        └─ сохраняется в SaveableObject.extraTexture (заметка 11)
Cleaner (предмет-скребок/метла)
   └─ при трении о корпус стирает грязь (MasterPainter.PaintObject)
```

## `CleanableObject` (`[RequireComponent(HullPlayerCollider)]`)

| Поле/метод | Содержание |
|------------|-----------|
| `dirtCoat` (`Texture2D`) | Кисть/маска грязи. |
| `dirtMaterial` | Материал оверлея грязи на корпусе. |
| `ApplyDailyDirt()` | **Каждый день** (`Sun.OnNewDay`): нанести тонкий слой грязи — `MasterPainter.instance.ApplyCoat(this, dirtCoat, 0.02f)`. Корпус обрастает постепенно. |
| `ApplyNewDirtTexture(tex)` | Заменить текстуру грязи + сохранить в `saveable.extraTexture`. |
| `GetCurrentDirtTex()` | Текущая текстура грязи. |
| `LoadTexture(tex)` / `RegisterToSaveable(obj)` | Загрузка из сейва / привязка к `SaveableObject`. |
| `CleanFully()` | Полная очистка (debug: `MasterPainter.DebugApplyRenderTex`). |

- Подписка на `Sun.OnNewDay` в `OnEnable` / отписка в `OnDisable`.
- Текстура сохраняется как `SaveableObject.extraTexture` (заметка 11): в сейве — raw (16 777 216 байт) или PNG, 128×128.

## `Cleaner` (предмет для чистки)

`Cleaner` — это `ShipItem` (скребок/метла) с анимированной «костью» (`Rigidbody bone`), двумя рендерами (skinned в руках / static на палубе).

| Поле | Содержание |
|------|-----------|
| `range` (0.6) | Радиус чистки. |
| `spacing` (1) | Шаг мазков. |
| `minSpeed` (0.5) | Мин. скорость движения для чистки. |
| `activated` | Трёт ли сейчас. |
| `uvPos` | Позиция мазка в UV корпуса. |

- Когда предмет в руках и игрок **трёт корпус** (движение со скоростью ≥ `minSpeed`), `Cleaner` стирает грязь в точке контакта через `MasterPainter` (мазки по UV с шагом `spacing`, радиус `range`).
- Анимация «подметания» (sidesweep), звук, частицы.

## `MasterPainter` (движок покраски)

Синглтон `MasterPainter.instance`, рисует по render-текстуре корпуса:
- `ApplyCoat(cleanable, coatTex, amount)` — нанести слой (грязь/краска) с силой `amount`.
- `PaintObject(cleanable, uv, ...)` — мазок в UV-точке (чистка/покраска).
- `DebugApplyRenderTex(cleanable)` — применить/сбросить (полная очистка).

## Связь с другими системами

| Система | Связь |
|---------|-------|
| Сохранение (заметка 11) | Текстура грязи = `SaveableObject.extraTexture` (128×128, raw/PNG). |
| `Sun.OnNewDay` (заметка 18) | Ежедневное нанесение грязи (`ApplyDailyDirt`). |
| `BoatDamage`/ходкость (заметка 14) | Грязный корпус, вероятно, влияет на сопротивление/скорость (`HullDrag`); чистка возвращает ход. |
| `SaveablePaint` (заметка 22) | Покраска корпуса — тот же механизм `MasterPainter`. |

## 👁 Взгляд моддера

| Хочу | Как |
|------|-----|
| Изменить скорость обрастания | Патч `CleanableObject.ApplyDailyDirt` (множитель `0.02f`) или `MasterPainter.ApplyCoat`. |
| Мгновенно очистить корпус | `cleanable.CleanFully()` / `MasterPainter.instance.DebugApplyRenderTex(cleanable)`. |
| Добавить грязь/краску | `MasterPainter.instance.ApplyCoat(cleanable, tex, amount)`. |
| Прочитать «чистоту» | `cleanable.GetCurrentDirtTex()` и анализ текстуры (средняя альфа/яркость). |
| Авто-чистка / «необрастающий» корпус | Отписаться от `Sun.OnNewDay` (`OnDisable`-патч) или нулить `ApplyCoat`. |
| Визуализировать чистоту в HUD | Читать текстуру грязи и выводить индикатор. |

**Грабли:**
- Грязь — **текстура**, а не скаляр: «процент чистоты» придётся вычислять по пикселям (например, средняя альфа `GetCurrentDirtTex()`).
- Текстура 128×128; при сохранении игра различает raw/PNG **по длине массива** (заметка 11) — если подменяете `extraTexture`, соблюдайте размер/формат.
- `ApplyDailyDirt` срабатывает по `Sun.OnNewDay` и только при `GameState.playing`; условие в коде учитывает `saveable.extraSetting` (куплена ли лодка).
- `MasterPainter` оперирует render-текстурой/UV — для своей покраски нужны UV-координаты корпуса.

## Практические выводы

1. **Грязь корпуса = текстура-оверлей** (128×128), копится ежедневно (`OnNewDay`, слой 0.02) и стирается предметом `Cleaner` через `MasterPainter`.
2. **Чистка** — физическое трение корпуса скребком (мазки по UV, радиус `range`, мин. скорость `minSpeed`).
3. **Текстура сохраняется** в `SaveableObject.extraTexture` (raw/PNG) — обрастание переживает сейв.
4. Управление через `MasterPainter.instance` (`ApplyCoat`/`PaintObject`/`DebugApplyRenderTex`) и `CleanableObject` (`CleanFully`/`GetCurrentDirtTex`).
5. Та же механика (`MasterPainter` + `SaveablePaint`) используется для **покраски** корпуса (заметка 22).
