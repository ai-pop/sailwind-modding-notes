# 41. Камеры и режимы вида

Разбор подсистемы камер. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Тряска камеры — в заметке 39 (`Juicebox.ShakeScreen`), обзор/прицел — в заметке 10 (`GoPointer`).

## Виды камер в игре

| Камера | Класс | Назначение |
|--------|-------|-----------|
| От первого лица | `centerEye` (OVRCameraRig) + `MouseLook` | Обычный вид игрока. |
| «Камера лодки» / обзор | `BoatCamera` (синглтон) | Вид на лодку сбоку/сверху (для парусов и верфи). |
| Плавное слежение | `CameraFollow` | Универсальный follow-компонент. |
| Камера карты | `MapTableCamera` (заметка 37) | Рендер карты на столе. |
| Камера верфи | `ShipyardCameraRotator` (заметка 22) | Вращение вокруг лодки в верфи. |

## «Камера лодки» (`BoatCamera`, синглтон `BoatCamera.instance`)

Режим обзора лодки со стороны. `BoatCamera.on` (static) — активен ли режим.

| Поле | Содержание |
|------|-----------|
| `centerEye` | Трансформ камеры (переиспользуется VR-риг). |
| `camHeight` (2) | Высота камеры над лодкой. |
| `zoomSpeed` (10), `zoomLevel` (-8..-40) | Зум (скролл): от `-8` (близко) до `-40` (далеко, позади лодки). |
| `playerLooks[]` / `boatLooks[]` | Наборы `MouseLook`, переключаемые при входе/выходе. |
| `outlines[]` (`OutlineEffect`) | Контуры предметов (выключаются в режиме камеры). |
| `crosshair` | Прицел (прячется в режиме камеры). |
| `currentPosOffset` | Смещение (панорама в верфи, ±25). |

### Переключение
- **Клавиша `InputName 16`** (не в верфи) → `SwitchOn()` / `SwitchOff()`.
- **`SwitchOn`**: `on = true`, прицел/контуры выключаются, `boatLooks` включаются (вместо `playerLooks`), `centerEye` репарентится к лодке, брызги отодвигаются.
- **`SwitchOff`**: обратное — вид от первого лица, прицел/контуры назад.

### Поведение (`Update`)
- Когда `on`: камера следует за `GameState.currentBoat` (`position = currentBoat.position + currentPosOffset`), высота `camHeight`, отвод `zoomLevel`.
- **Зум**: `zoomLevel += GameInput.GetScrollAxis() * zoomSpeed`, кламп `[-40, -8]`.
- **В верфи** (`GameState.currentShipyard`): панорама ПКМ (`currentPosOffset` по `Mouse X`, кламп ±25).
- Если игрок сошёл с лодки (`currentBoat == null`) → авто-`SwitchOff`.

## Плавное слежение (`CameraFollow`)

Простой переиспользуемый follow:
```csharp
public class CameraFollow : MonoBehaviour {
    public Transform target;
    public float positionLerp;   // скорость доводки позиции
    public float rotationLerp;   // скорость доводки поворота

    void Update() {
        transform.position = Vector3.Lerp(transform.position, target.position, dt * positionLerp);
        transform.rotation = Quaternion.Slerp(transform.rotation, target.rotation, dt * rotationLerp);
    }
}
```

## 👁 Взгляд моддера: моды камеры/вида

| Хочу | Как |
|------|-----|
| Включить/выключить обзор лодки | `BoatCamera.instance.SwitchOn()/SwitchOff()`; проверка `BoatCamera.on`. |
| Свой зум/высота обзора | Поля `BoatCamera` `camHeight`, `zoomLevel`, `zoomSpeed` (или патч `UpdateZoom`). |
| Своя камера, следящая за объектом | Компонент `CameraFollow` (target + lerp) на своей камере. |
| Камера от третьего лица / свободная | Отключить `playerLooks`/`MouseLook`, управлять своей камерой; читать позицию игрока из `Refs.observerMirror` (заметка 30). |
| Тряска камеры | `Juicebox.juice.ShakeScreen(...)` (заметка 39). |
| Кинематографичный пролёт | `Juicebox.juice.TweenPosition(camera, ...)` + `CameraFollow`. |
| Скрыть прицел/контуры для скриншота | `BoatCamera.SetOutlines(false)` + `crosshair.SetActive(false)` (по образцу `SwitchOn`). |

**Грабли:**
- `centerEye` — это **VR-риг**; в не-VR режиме основная камера всё равно завязана на него + `MouseLook`. Свои камеры лучше делать отдельным `Camera`, не ломая `centerEye`.
- `BoatCamera` переключает наборы `MouseLook` (`playerLooks`/`boatLooks`) — если добавляете свои камеры, учитывайте, какие `MouseLook` активны.
- `InputName 16` — клавиша камеры лодки; в верфи её поведение другое (панорама).
- Позиция лодки — `GameState.currentBoat.position` (в координатах сцены, с учётом плавающего начала координат, заметка 11).

## Практические выводы

1. **Вид от первого лица** = `centerEye` (OVRCameraRig) + `MouseLook`; **обзор лодки** = `BoatCamera` (тумблер `InputName 16`, зум скроллом -8..-40).
2. `BoatCamera.instance` (`SwitchOn/Off`, `camHeight`, `zoomLevel`) — точка управления обзором.
3. **`CameraFollow`** — готовый плавный follow для своих камер.
4. Тряска/пролёты — через `Juicebox` (заметка 39).
5. Свои камеры лучше делать отдельным `Camera`, не трогая VR-риг `centerEye`.
