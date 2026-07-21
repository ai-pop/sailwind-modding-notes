# 36. Движение игрока: ходьба, прыжки, плавание, присед

Разбор подсистемы движения игрока. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Глобальные ссылки — в заметке 30, значения `InputName` — в заметке 24.

## Архитектура движения

Движение игрока — на **`OVRPlayerController`** (`Refs.ovrController`, из OVR-фреймворка) + **`CharacterController`** (`Refs.charController`). Поверх них — игровые надстройки:

| Компонент | Роль |
|-----------|------|
| `OVRPlayerController` | Базовое перемещение (OVRLocomotion): `Acceleration`, `Damping`, `swimSpeedMult`, `ratlinesSpeedMult`, `jumping`, `GetMoveInput()`, `ToggleWalking()`, `Stop()`. |
| `CharacterController` | Коллизии/уклоны: `slopeLimit`, `stepOffset`, `isGrounded`, `Move()`. |
| `PlayerClimb` | Прыжки, гравитация, уклоны, ванты (ratlines). |
| `PlayerSwimming` | Плавание/ныряние. |
| `PlayerCrouching` | Приседание. |
| `PlayerUnstucker` | Анти-застревание. |

## Уточнение `InputName` (по движению)

Из кода движения достоверно (дополнение к заметке 24):

| `InputName` | Действие |
|:--:|----------|
| `0` | Вперёд (используется для ныряния по взгляду при плавании). |
| `4` | **Прыжок** (также «всплыть» при плавании, встать при приседе). |
| `7` | **Присесть / нырнуть вниз**. |

## Прыжки и уклоны (`PlayerClimb`)

Параметры (SerializeField, тюнятся):
- `jumpPower`, `minJumpDuration`/`maxJumpDuration`, `gravity`, `terminalVelocity`;
- `maxUpMomentum` (на земле), `maxSwimJumpMomentum` (в воде);
- `accel`/`damping`, `jumpDampingMult` (управление в прыжке вялое);
- `climbSlopeLimit` (90° — в прыжке лезет по любому склону), `climpStepOffset` (0.4);
- `normalSlopeLimit` (40°), `normalStepOffset` (0.1).

### Логика
- **Прыжок** (`InputName 4`): пока клавиша держится и `jumpDuration < maxJumpDuration` — набор импульса вверх (`fallMomentum -= jumpPower*dt`, кап `-currentMaxUpMomentum`). Отпускание после `minJumpDuration` прекращает прыжок (прыжок с удержанием = выше).
- **Падение**: не на земле и не в воде → `fallMomentum += gravity*dt` (кап `terminalVelocity`), `controller.Move(down*fallMomentum*dt)`.
- **Крыша**: raycast вверх гасит прыжок при столкновении.
- **Уклоны**: не на лодке → `normalSlopeLimit * 2` (на суше можно круче); в прыжке → `climbSlopeLimit` (90°).
- **В воде** прыгать можно только с поверхности (`PlayerSwimming.AllowJumping()` = `swimmingOnSurface`).
- **Debug-режим** (`buildDebugModeOn`): `maxJumpDuration = 9999` — бесконечный «прыжок» (фактически полёт вверх).

### Ванты / вант-трапы (ratlines)
- `GameState.onRatlines` определяется `OverlapSphere` по тегу `Ratlines`.
- На вантах: `slopeLimit` повышается (можно лезть вертикально), `ratlinesSpeedMult` = `0.08` (лезешь вверх) или `0.38` (если смотришь вниз/от них, >120°). `ClickRatlines()` статически стопит игрока на 0.35 c при клике.

## Плавание (`PlayerSwimming`)

Статические флаги (читать из своих модов):
- `swimming`, `inShallowWater`, `swimmingOnSurface`, `observerSwimming`, `dive`, `cameraWaterHeight`.

Параметры: `swimHeight`, `shallowWaterHeight` (0.6), `surfaceSwimHeight`, `diveSpeedMult`, `jumpKeyUndiveSpeed`.

### Определение состояния
```
waterHeight = Lerp(..., UnderwaterEffect.cameraWaterHeight, ...)   // высота воды (Crest)
swimming          = pos.y <= waterHeight + swimHeight
inShallowWater    = pos.y <= waterHeight + shallowWaterHeight
swimmingOnSurface = pos.y >= waterHeight + surfaceSwimHeight  (и swimming)
```

### Поведение
- **Скорость**: `swimSpeedMult` = 0.55 (вплавь) / 0.77 (мелководье) / 1 (суша).
- **Покачивание на волнах**: при плавании `charController.Move(up * (waterHeight - lastWaterHeight))` — игрок поднимается/опускается с волной.
- **Ныряние/всплытие:**
  - `InputName 4` (прыжок) + плавание + не на поверхности → всплытие (`Move up`);
  - `InputName 7` + плавание → нырнуть (`Move down`);
  - движение вперёд (`InputName 0` / `GetPrimaryVertical > 0.01`) + плавание → ныряние по углу взгляда (`dive = SignedAngle(body.forward, camera.forward) * diveSpeedMult`).
- На поверхности нырнуть вперёд нельзя (`dive = 0`), но можно прыгать.

## Приседание (`PlayerCrouching`)

- Переключение по `InputName 7` (keydown), не в кровати.
- Высота головы: `initialHeight` ↔ `0.2` (присед), лерп `dt * 9`.
- Автовставание: прыжок (`InputName 4`), плавание, вход в кровать.
- `Refs.ovrCameraRig` устанавливается здесь. `GetCurrentHeadHeight()` — текущая высота.

## Анти-застревание (`PlayerUnstucker`)

За цикл 3.3 c копит вводимое направление и реальное смещение. Если игрок жал **все 4 направления** (`up && down && left && right`), но реально сдвинулся меньше `stuckThreshold` (0.5) → лог `"! STUCK !"` и **подталкивание вверх на 0.5** (`Translate(up*0.5)`).

## 👁 Взгляд моддера: моды движения

| Хочу | Как |
|------|-----|
| Быстрее/медленнее ходить | `Refs.ovrController.Acceleration` / `Damping`; множители `swimSpeedMult`, `ratlinesSpeedMult`. |
| Выше/ниже прыжок, своя гравитация | Поля `PlayerClimb` (`jumpPower`, `gravity`, `terminalVelocity`, `maxUpMomentum`) — патч или SerializeField. |
| Полёт/noclip | По образцу debug-режима: `maxJumpDuration = 9999`, либо `controller.detectCollisions = false` + ручное `Move`. |
| Телепорт игрока | Двинуть `controller.transform` + `Refs.observerMirror` **в координатах сцены** (учесть `FloatingOriginManager.outCurrentOffset`, заметка 11). |
| Спринт | Свой множитель к `ovrController.Acceleration` по клавише. |
| Плавание (своё) | Читать `PlayerSwimming.swimming/swimmingOnSurface`; вода — `UnderwaterEffect.cameraWaterHeight`. |
| Читать ввод движения | `Refs.ovrController.GetMoveInput()` (Vector2), `GameInput.GetPrimaryVertical/Horizontal` (заметка 24). |
| Отключить управление | `Refs.SetPlayerControl(false)` (заметка 30). |

**Грабли:**
- Позиция игрока для логики — чаще `Refs.observerMirror`, но физически двигается `CharacterController`/`OVRPlayerController`; телепорт делать через контроллер, не через камеру.
- `swimSpeedMult`/`ratlinesSpeedMult` **перезаписываются каждый кадр** игрой (`PlayerSwimming.LateUpdate`, `PlayerClimb.Update`) — свой множитель надо ставить **после** них или патчить эти методы.
- Всё движение завязано на `GameState.playing` — в меню/загрузке не тикает.
- `InputName 4/7` — прыжок/присед; полный ввод — через отсутствующий `GameInput` (заметка 24).

## Практические выводы

1. **Движение** = `OVRPlayerController` (ускорение/демпфирование/множители) + `CharacterController` (уклоны/ступени) + надстройки (`PlayerClimb/Swimming/Crouching/Unstucker`).
2. **Прыжок** с удержанием (импульс копится), гравитация/терминальная скорость тюнятся; debug даёт «полёт» (`maxJumpDuration = 9999`).
3. **Плавание** определяется высотой воды (Crest) относительно игрока; покачивается на волнах; ныряние по взгляду, всплытие/нырок клавишами.
4. **InputName**: `4` = прыжок/всплыть, `7` = присесть/нырнуть, `0` = вперёд.
5. **Множители скорости** (`swimSpeedMult`, `ratlinesSpeedMult`) ставятся игрой каждый кадр — патчить, а не просто выставлять.
6. **Телепорт** — через контроллер, в координатах сцены (с учётом плавающего начала координат).
