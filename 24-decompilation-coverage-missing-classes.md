# 24. Покрытие декомпиляции: отсутствующие, но используемые классы

**Важное предупреждение для мододела.** Репозиторий `sailwind-decompiled` содержит 677 игровых классов, но декомпиляция **неполная**: ряд классов, на которые ссылается присутствующий код, в репозитории отсутствует (вероятно, не попали в выгрузку ILSpy или лежат в других сборках). Ниже — перечень критичных пробелов и **реконструкция их API по местам использования**, чтобы справочник оставался полезным.

## Что отсутствует (по числу ссылок в игровом коде)

| Класс | Ссылок | Роль |
|-------|:--:|------|
| `GameInput` | ~25 | Централизованный ввод (клавиши, оси, скролл). |
| `InputName` | ~24 | Перечисление действий управления. |
| `BoatProbes` | ~9 | **Ядро физики лодки** (плавучесть, сопротивление, «двигатель»). |
| `SampleHeightHelper` | ~8 | Сэмплирование высоты волн Crest. |
| `SimpleFloatingObject` | ~4 | Плавучесть предметов/поплавка. |
| `FloatingObjectBase` | ~3 | База плавающих объектов. |
| `ShapeGerstnerBatched` | ~2 | Gerstner-волны Crest. |
| `Crest.*` | — | Пространство имён океанской библиотеки Crest (кастомный форк). |
| `UnderwaterEffect` | ~2 | Высота воды у камеры (`cameraWaterHeight`) для плавания. |
| `BoatAlignNormal` | ~1 | Упрощённая физика лодки вдали от игрока (выравнивание по нормали волны). |

> Присутствующий в репозитории класс `Buoyancy` — это **общая блоб-плавучесть** (заметка 14), но реальная покомпонентная физика конкретной лодки живёт в **`BoatProbes`**, которого в выгрузке нет.

## Реконструкция `GameInput` (статический класс ввода)

Восстановлено по вызовам в `GoPointer`, `LookUI`, `BoatCamera`, `GPButton*`, `GoPointerMovement`, `MapTableCamera` и др.:

```csharp
static class GameInput {
    static bool controllerEnabled;                              // GPButtonSettingsCheckbo

    static bool   GetKey(InputName action);                     // удерживается
    static bool   GetKeyDown(InputName action);                 // нажатие в этом кадре
    static bool   GetKeyUp(InputName action);                   // отпускание

    static KeyCode GetKeyCode(InputName action, bool input2, bool controllerButton);
    static void    SetKeyMap(InputName action, KeyCode key, bool input2, bool controllerButton);
    static string  GetControllerKeyString(InputName action);    // имя кнопки контроллера
    static void    ResetToDefaults();                            // GPButtonResetKeybindings

    static float  GetScrollAxis();                               // колесо мыши (GoPointer, BoatCamera zoom)
    static float  GetPrimaryHorizontal();                        // аналоговые оси
    static float  GetPrimaryVertical();
    static float  GetSecondaryHorizontal();
    static float  GetSecondaryVertical();
}
```

`input2` — альтернативная (вторая) клавиша действия; `controllerButton` — кнопка контроллера (OVR). Переназначение работает через `SetKeyMap` (см. `GPButtonKeybinding`, заметка 10).

## Реконструкция `InputName` (enum действий)

Числовые значения встречаются как `(InputName)N`. По контексту использования (уточнено по всему коду):

| Значение | Действие | Где используется |
|:--:|--------------------|------------------|
| 0 | **Вперёд** (движение; при плавании — ныряние по взгляду) | `GoPointerMovement` (вперёд/поворот), `PlayerSwimming` (dive) |
| 1 | **Назад** | `GoPointerMovement` |
| 2 | **Влево/вправо** (strafe) | `GoPointerMovement` |
| 3 | **Вправо/влево** (strafe) | `GoPointerMovement` |
| 4 | **Прыжок** (также всплытие при плавании) | `PlayerClimb`, `PlayerSwimming`, `PlayerCrouching` (встать) |
| 5 | **Трос/лебёдка** (тянуть) | `GPButtonRopeWinch` |
| 7 | **Присесть / нырнуть вниз** | `PlayerCrouching`, `PlayerSwimming` |
| 8 | **Основное действие** («use», аналог ЛКМ) | `GoPointer` (клик/подбор), `LookUI` (левая иконка), `MapTableCamera`, `GPButtonSettingsCheckbo` |
| 9 | **Альт-действие** («alt use», аналог ПКМ) | `GoPointer`, `CrateInventoryUI`, `CrateSealUI`, `MapChart`, `LookUI` (правая иконка), `Sleep` (выход из кровати) |
| 10 | **Бросок / положить** (drop) | `GoPointer` (логика drop, `Settings.autoThrow`) |
| 11 | **Модификатор** (при удержании отключает mouse-look) | `MouseLook`, `GoPointer` (346) |
| 15 | **Открыть инвентарь/потребности** (UI) | `PlayerNeedsUI` (или OVR RawButton 256) |
| 16 | **Камера лодки** (переключение/зум) | `BoatCamera` |
| 17 | **Подсказки** (toggle hints) | `Hints` |

> Значения 0–4, 7–10, 15–17 надёжны (подтверждены контекстом); 5 и 11 — по единичному использованию. Точные имена enum-констант неизвестны (определения нет). `GPButtonKeybinding` парсит имя через `Enum.Parse(typeof(InputName), inputName)` — имена задаются строкой в префабе кнопки. Движение также идёт через аналоговые оси `GameInput.GetPrimaryVertical/Horizontal`.

## Реконструкция `BoatProbes` (ядро физики лодки)

По использованию в `BoatDamage`, `BoatMass`, `Debugger`, `HullDrag`, `BoatPerformanceSwitcher`, `Recovery`:

```csharp
class BoatProbes : MonoBehaviour {
    // Плавучесть
    public float _forceMultiplier;        // ГЛАВНЫЙ множитель плавучести (0 = тонет)
    public /* точки */ _forcePoints;       // точки приложения силы плавучести
    public /* ... */ appliedBuoyancyForces;// приложенные силы (визуализация/отладка)

    // Сопротивление воды
    public float _dragInWaterForward;      // сопротивление воды (вперёд)
    public float _dragInWaterRight;        // сопротивление воды (в борт)
    public float addedHullDrag;            // добавленное сопротивление корпуса
    public float addedSideDrag;            // добавленное боковое сопротивление

    // «Двигатель»
    public void ChangeEnginePower(float power);   // чит Debugger: 0/0.5/2/5/-5/8/50
}
```

- `BoatDamage` управляет `_forceMultiplier` для потопления: `Lerp(base, base*0.66, …)` и ступенчато к 0 при `sunk` (заметка 14).
- `Debugger` в debug-режиме вызывает `ChangeEnginePower` для разгона лодки (заметка 21).
- `baseBuoyancy` в `BoatDamage` = `boat._forceMultiplier` на старте.
- `BoatPerformanceSwitcher` ожидает `BoatProbes` **или** `BoatAlignNormal` (переключатель режимов физики по удалённости).

## Реконструкция плавающих объектов и Crest

```csharp
class FloatingObjectBase {
    public bool InWater;                  // объект в воде (читает FishingRodFish)
}
class SimpleFloatingObject : FloatingObjectBase {
    public float _raiseObject;            // высота «всплытия» над водой (= ShipItem.floaterHeight)
    public float _dragInWaterRotational;  // вращательное сопротивление в воде
}

struct/class SampleHeightHelper {         // Crest: сэмплирование высоты волны
    void Init(Vector3 pos, float accuracy, bool _, Object __);
    bool Sample(ref float result);        // высота воды в точке
}
```

### `UnderwaterEffect` (статический)
```csharp
static class UnderwaterEffect {
    public static float cameraWaterHeight;  // высота воды у камеры (читает PlayerSwimming)
}
```
Источник «высоты воды» для плавания: `PlayerSwimming` лерпит `waterHeight` к `UnderwaterEffect.cameraWaterHeight`.

### `BoatAlignNormal`
Альтернативный (упрощённый) режим физики лодки — выравнивание по нормали волны вместо полной блоб-плавучести `BoatProbes`. `BoatPerformanceSwitcher` (заметка 14) ожидает **либо** `BoatProbes`, **либо** `BoatAlignNormal`: для лодок вдали от игрока включается лёгкий `BoatAlignNormal`, вблизи — полный `BoatProbes`.

`Ocean` (присутствует) предоставляет высокоуровневые `GetWaterHeightAtLocation2(x, z)` и `GetChoppyAtLocation(x, z)`; низкоуровневая Gerstner-математика (`ShapeGerstnerBatched`, `SampleHeightHelper`) — в отсутствующем модуле Crest.

## Практические выводы для мододела

1. **Справочник неполон:** `GameInput`, `InputName`, `BoatProbes`, Crest-хелперы отсутствуют в `sailwind-decompiled`. Не удивляйтесь, что их нет при поиске.
2. **Ввод** идёт через `GameInput` (не напрямую `UnityEngine.Input` в игровой логике); переназначение — `SetKeyMap`/`GetKeyCode` с флагами `input2`/`controllerButton`.
3. **Действия 8/9** = основное/альтернативное («use»/«alt use»), **10** = бросок, **5** = лебёдка, **16** = камера, **17** = подсказки, **0–3** = движение.
4. **Плавучесть лодки** реально контролируется `BoatProbes._forceMultiplier` (а не только общим `Buoyancy`); «мотор» лодки — `ChangeEnginePower`.
5. Для патчинга этих классов через Harmony их **нужно сначала найти в `Assembly-CSharp.dll`** (ILSpy/dnSpy) — они есть в сборке, просто не выгружены в этот репозиторий. Имена типов и сигнатуры выше точны (восстановлены по IL-вызовам).
6. Если нужна полная картина — декомпилируйте `Assembly-CSharp.dll` заново и дополните репозиторий недостающими файлами (`GameInput`, `InputName`, `BoatProbes`, `SimpleFloatingObject`, `FloatingObjectBase`, Crest).
