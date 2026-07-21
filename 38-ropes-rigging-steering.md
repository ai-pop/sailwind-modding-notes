# 38. Тросы, такелаж и управление (штурвал, лебёдки)

Разбор подсистемы управления судном — как игрок рулит и работает с парусами через тросы. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Паруса и ветер — в заметке 17, якорный трос — в заметке 29.

## Абстракция троса (`RopeController`)

Базовый класс всех «управляемых тросов»:

```csharp
public class RopeController : MonoBehaviour {
    public float currentLength;        // текущая длина/выборка
    public float currentResistance;    // текущее сопротивление (натяжение)
    public bool  changed;

    public virtual void UpdateSailAttachment() {}
    public virtual void Pull(float input)   {}   // тянуть
    public virtual void Loosen(float input) {}   // травить
    public virtual bool CanPull() => true;        // можно ли тянуть (напр. якорь)
}
```

### Подклассы (каждый — свой тип управления)
| Класс | Что делает |
|-------|-----------|
| `RopeControllerAnchor` | Якорный трос (`resistanceMult`, `canPull`, заметка 29). |
| `RopeControllerSailAngle` | Угол паруса (шкот). |
| `RopeControllerSailAngleJib` | Угол стакселя. |
| `RopeControllerSailAngleSquare` | Угол прямого паруса. |
| `RopeControllerSailReef` | Рифление паруса (площадь). |
| `RopeControllerSteeringWheel` | Трос штурвала → руль. |
| `RopeControllerMirror` | Зеркальный (NPC-лодки, авто-управление). |

## Штурвал (`GPButtonSteeringWheel`)

Управляет рулём через **пружину `HingeJoint`**.

| Поле | Содержание |
|------|-----------|
| `attachedRudder` (`HingeJoint`) | Соединение с рулём. |
| `rudder` (`Rudder`) | Компонент руля (`currentAngle`). |
| `gearRatio` (3) | Передаточное: **угол штурвала = угол руля × 3**. |
| `currentInput` | Текущий ввод руления. |
| `rotationAngleLimit` | Предел поворота (`limits.max * gearRatio`). |
| `locked` | **Фиксация штурвала** (удержание курса). |

### Логика
- **Ввод** (мышь-драг / клавиатура / тач / VR) → `currentInput` (в пределах `±rotationAngleLimit`).
- **Применение к рулю** (`ApplyRudderRotation`): `spring.targetPosition = limits.max * (currentInput / rotationAngleLimit)` — пружина `HingeJoint` доворачивает руль.
- **Фиксация курса** (`Lock/Unlock`, альт-кнопка): `locked = true` — штурвал держит положение (встроенный «автопилот удержания курса»).
- **Связь углов:** `RudderToWheelAngle(a) = a * gearRatio`, `WheelToRudderAngle(a) = a / gearRatio`.
- **Режимы руления:** `Settings.steeringWithMouse` (вращение мышью с захватом курсора) vs sticky-click; VR — `TouchRotateHandle`.
- Звук скрипа штурвала громче при быстром вращении.

## Лебёдка/утка для тросов (`GPButtonRopeWinch`)

Интерфейс для тянуть/травить трос (паруса, якорь).

| Поле | Содержание |
|------|-----------|
| `rope` (`RopeController`) | Управляемый трос. |
| `gearRatio` (5) | Передаточное лебёдки. |
| `rotationSpeed` (10) | Скорость вращения. |
| `reverseWindResistance` | Инверсия сопротивления ветру. |
| `leftRightInput` / `continuousInput` | Тип ввода. |
| `ropeEffect` (`LineRenderer`) | Визуал троса (материал меняется при натяжении: `pullMaterial` vs `default`). |

- Игрок вращает лебёдку → `rope.Pull(input)` / `rope.Loosen(input)`.
- `AttachToController(controller)` — привязать лебёдку к контроллеру троса.
- `ShowWinch(state)` — показать/скрыть.

## Связь с другими системами

| Система | Связь |
|---------|-------|
| `Sail` (заметка 17) | `RopeControllerSailAngle` задаёт угол паруса к ветру; `RopeControllerSailReef` — площадь (`currentUnroll`). |
| `Rudder` / `RudderNew` | Руль (HingeJoint), поворачивает лодку. |
| `Anchor` (заметка 29) | `RopeControllerAnchor` — длина/сопротивление якорного троса. |
| `NPCBoatController` (заметка 20) | NPC используют `RopeControllerSailAngle/Reef` + `RopeControllerMirror` для авто-управления. |

## 👁 Взгляд моддера: моды управления судном

| Хочу | Как |
|------|-----|
| **Автопилот (держать курс)** | Встроенная фиксация: `GPButtonSteeringWheel.Lock()` + задать `currentInput`. Либо патчить `ApplyRudderRotation`/пружину руля: держать `spring.targetPosition` по целевому курсу (сравнивая `transform.forward` с желаемым направлением). |
| **Авто-трим парусов** | Читать `Sail.apparentWind` / `relativeWindAngle` и дёргать `RopeControllerSailAngle.Pull/Loosen`, ставя парус оптимально к ветру. |
| **Авторифление** | `RopeControllerSailReef` — менять `currentUnroll` по силе ветра (`Wind.currentWind.magnitude`). |
| **Читать состояние руля/парусов** | `rudder.currentAngle`, `Sail.currentUnroll`, `RopeController.currentLength/currentResistance`. |
| **Управление с клавиатуры/своего UI** | Вызывать `rope.Pull/Loosen` и `steeringWheel.currentInput` программно (в обход GoPointer). |
| **Дистанционное управление** | Патч `RopeController*` — единая точка: все троса идут через `Pull/Loosen/CanPull`. |

**Грабли:**
- **Руль — через пружину `HingeJoint`**, не прямым поворотом: чтобы автопилот держал курс, управляйте `spring.targetPosition` (как делает `ApplyRudderRotation`), а не `localEulerAngles` руля напрямую.
- **Передаточные числа**: штурвал ×3 (`gearRatio`), лебёдка ×5 — учитывайте при переводе «ввод → реальный угол/длина».
- `RopeController`-методы **виртуальные** — удобно патчить базовый `Pull/Loosen/CanPull` для глобальных эффектов (например, «все троса тянутся быстрее»).
- Ввод штурвала/лебёдок идёт через `GoPointer`/`GoPointerMovement` и `Settings.steeringWithMouse`/`winchesWithMouse` (заметка 03) — свой ввод нужно согласовать с этими настройками.
- `CanPull()` у якоря зависит от натяжения (`resistancePullLimit`, заметка 29) — не всякий трос можно тянуть в любой момент.

## Практические выводы

1. **`RopeController`** — единая абстракция троса (`currentLength`, `Pull/Loosen/CanPull`); подклассы — якорь, угол/риф паруса, штурвал.
2. **Штурвал** рулит через **пружину `HingeJoint`** руля (передаточное ×3), есть **фиксация курса** (`locked`) — основа для автопилота.
3. **Лебёдка** (`GPButtonRopeWinch`, ×5) тянет/травит трос паруса/якоря; визуал натяжения через `LineRenderer`.
4. **Автопилот/авто-трим** строятся на `spring.targetPosition` руля и `RopeController*.Pull/Loosen` + чтении `Sail.apparentWind`.
5. NPC-лодки используют те же контроллеры (`RopeControllerMirror`) — можно подсмотреть их логику авто-управления.
