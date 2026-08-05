# 79. Продвинутый такелаж: контроллеры парусов (AngleMasters), самовыравнивание и зеркалирование марселей

Полный технический анализ компонентов продвинутого управления такелажем и координации парусов в Sailwind v0.38 (`Assembly-CSharp.dll`). Дополняет базовые сведения о ветре и парусах ([заметка 17](17-wind-and-sails.md)) и абстракциях тросов ([заметка 38](38-ropes-rigging-steering.md)).

---

## 1. Архитектура координации такелажа

В Sailwind простые паруса (косые/гафельные с одним шкотом) управляются напрямую через `RopeControllerSailAngle`. Однако сложные типы парусов (стаксели, кливеры, прямые паруса с брасами, марсели) требуют координации двух шкотов/брасов или подчинения нижестоящему парусу. Для этого используются **мастера углов (`AngleMaster`)** и **зеркальные контроллеры (`Mirror`)**.

```
[ Левый шкот/брас ]        [ Правый шкот/брас ]
         │                          │
         └────────────┬─────────────┘
                      ▼
         [ JibAngleMaster / SquareAngleMaster ]
                      │
                      ▼ (вычисление min/max лимитов + sway)
             [ HingeJoint паруса ]
                      │
                      ├─────────────────────────────────┐
                      ▼                                 ▼
           [ JibSelfRighting ]            [ SquareTopsailAngleMirror ]
       (пружина возврата при штиле)         (копирование limits на марсель)
```

---

## 2. Управление кливерами и стакселями (`JibAngleMaster`)

Косые носовые паруса (кливеры, стаксели) имеют два шкота (левый и правый). `JibAngleMaster` объединяет их натяжение и управляет ограничениями `HingeJoint`.

### 2.1. Пересечение ограничений шкотов (`Update`)
Каждый кадр компонент получает лимиты от левого и правого шкотов и находит их пересечение:

```csharp
currentLimitPlus  = Mathf.Min(limitPlusFromLeft, limitPlusFromRight);
currentLimitMinus = Mathf.Max(limitMinusFromLeft, limitMinusFromRight);

JointLimits limits = sailHinge.limits;
limits.max = currentLimitPlus;
limits.min = currentLimitMinus;
sailHinge.limits = limits;

if (limitPlusFromRight <= limitMinusFromLeft)
{
    limitMinusFromLeft = limitPlusFromRight = (limitPlusFromRight + limitMinusFromLeft) / 2f;
}
```

- `currentLimitPlus` — максимальный положительный угол (ограничивается тем шкотом, который натянут сильнее с плюсовой стороны).
- `currentLimitMinus` — максимальный отрицательный угол.
- **Клин шкотов (`CanPull`):** если оба шкота выбраны настолько сильно, что `limitPlusFromRight <= limitMinusFromLeft`, парус фиксируется посередине (`/ 2f`), а метод `CanPull()` начинает возвращать `false`, блокируя дальнейшее натяжение лебедок.

### 2.2. Симуляция ветрового трепетания (`ApplySway`)
Для реалистичного «хлопания» паруса на ветру в лимиты `HingeJoint` подмешивается осциллирующая добавка:

```csharp
sway += Time.deltaTime * swayDirection * Wind.currentWind.sqrMagnitude * 0.003f;
if (sway > 0.5f)  swayDirection = Random.Range(-1f, -2f);
if (sway < -0.5f) swayDirection = Random.Range(1f, 2f);

JointLimits limits = sailHinge.limits;
limits.max = currentLimitPlus + sway;
limits.min = currentLimitMinus - sway;
sailHinge.limits = limits;
```

Величина трепетания `sway` колеблется в диапазоне `[-0.5°, +0.5°]`, а скорость колебаний пропорциональна квадрату силы ветра (`sqrMagnitude * 0.003f`).

---

## 3. Самовыравнивание паруса при штиле (`JibSelfRighting`)

Без ветра кливер на `HingeJoint` мог бы безвольно свисать под любым углом. `JibSelfRighting` удерживает парус по центру, плавно отключая пружину при появлении ветра:

```csharp
private void Update()
{
    float num = sail.appliedWindForce / maxWind;
    float spring = Mathf.Lerp(springWithNoWind, 0f, num);
    SetSpring(spring);
}

private void SetSpring(float value)
{
    JointSpring spring = joint.spring;
    spring.spring = value;
    joint.spring = spring;
}
```

- При `appliedWindForce == 0` пружина равна `springWithNoWind` — парус выравнивается в диаметральную плоскость судна.
- При усилении ветра пружина стремится к `0`, полностью передавая управление ветровым силам и натяжению шкотов.

---

## 4. Управление прямыми парусами (`SquareAngleMaster`)

Прямые паруса управляются через поворот рея левым и правым брасами. `SquareAngleMaster` работает аналогично `JibAngleMaster`, но задает лимиты с небольшим зазором (`0.01f`), предотвращающим залипание физического джойнта:

```csharp
JointLimits limits = sailHinge.limits;
limits.max = limitFromLeft + 0.01f;
limits.min = limitFromRight - 0.01f;
sailHinge.limits = limits;
```

Если брасы натянуты воистину жестко (`limitFromLeft <= limitFromRight`), рей блокируется (`CanPull() == false`).

---

## 5. Зеркалирование марселей (`SquareTopsailAngleMirror` и `RopeControllerMirror`)

Марсели и брамсели (верхние ярусы прямых парусов) в Sailwind не требуют отдельных лебедок для поворота реев — они автоматически копируют положение нижнего паруса.

### 5.1. `SquareTopsailAngleMirror`
В `LateUpdate()` компонент синхронизирует верхний рей с нижним (`sailBelow`):

```csharp
private void LateUpdate()
{
    sail.limits = sailBelow.limits;
    connections.angleControllerLeft.transform.position = leftConnection.position;
    connections.angleControllerRight.transform.position = rightConnection.position;
    leftRope.currentRopeLength = 0f;
    rightRope.currentRopeLength = 0f;
}
```

- Лимиты угла поворота (`limits`) в точности копируются с нижнего паруса.
- Точки крепления тросов визуально привязываются к координатам нижнего рея (`leftConnection`, `rightConnection`), а длина троса удерживается на `0f`.

### 5.2. Универсальный `RopeControllerMirror`
Используется на многоярусных косых парусах и NPC-лодках:

```csharp
private void Update()
{
    sail.currentUnroll = parentSail.currentUnroll;
    hinge.limits = parentHinge.limits;
}
```

Копирует как угол поворота (`limits`), так и степень развертывания паруса (`currentUnroll`) с родительского паруса каждый кадр.

---

## 6. Практические выводы для мододела

1. **Многоярусные паруса без лишнего UI:** при создании крупных судов с марселями/брамселями не добавляйте лишние лебедки. Повесьте на верхние паруса `SquareTopsailAngleMirror` или `RopeControllerMirror` — они автоматически синхронизируются с нижним реем.
2. **Предотвращение клина лебедок:** в модах на автопилот или авто-тримминг проверяйте `AngleMaster.CanPull()`. Если метод вернул `false`, противоположный шкот/брас натянут до предела, и попытка тянуть лебедку дальше приведет к дерганию физики.
3. **Пружина штиля для кастомных парусов:** добавляйте `JibSelfRighting` на любые свободно вращающиеся паруса, чтобы при штиле они красиво возвращались в диаметральную плоскость, а не болтались.
4. **Визуальное трепетание:** эффект `ApplySway()` в `JibAngleMaster` привязан к квадрату скорости ветра (`sqrMagnitude * 0.003f`). Для парусов большой площади коэффициент `0.003f` рекомендуется уменьшать, чтобы парус не проходил сквозь ванты при шторме.
