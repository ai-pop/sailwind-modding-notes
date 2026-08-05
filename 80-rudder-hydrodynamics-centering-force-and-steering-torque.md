# 80. Гидродинамика руля: поворотный момент, сопротивление, центрирующая сила и пружина штурвала

Полный разбор физической модели рулевого управления судна в Sailwind v0.38 (`Assembly-CSharp.dll`). Компонент `Rudder` отвечает за преобразование угла отклонения руля и скорости судна в крутящие моменты рыскания, гидродинамическое торможение и обратную центрирующую силу на штурвале. Дополняет обзор такелажа и штурвала из [заметки 38](38-ropes-rigging-steering.md).

---

## 1. Поля и параметры компонента `Rudder`

`Rudder` устанавливается на объект пера руля и требует наличия компонента `HingeJoint`, через который штурвал (`GPButtonSteeringWheel`) поворачивает руль.

| Поле | Тип / Значение по умолчанию | Описание |
|---|---|---|
| `shipRigidbody` | `Rigidbody` | Основной физический компонент корпуса судна. |
| `rudderPower` | `float = 1000f` | Множитель эффективности руля (сила поворота). |
| `pushbackPower` | `float = 200f` | Множитель гидродинамического давления воды, толкающего руль к центру. |
| `dragPower` | `float = 0.02f` | Коэффициент гидродинамического сопротивления (торможения при повороте). |
| `shipForwardVelocity` | `float` | Текущая продольная скорость судна относительно его корпуса. |
| `currentAngle` | `float` | Угол отклонения руля в градусах (`[-180, +180]`). |
| `currentTension` | `float` | Текущее расчетное натяжение/давление потока воды на руль. |
| `outAppliedForce` | `float` | Диагностическое значение итоговой приложенной силы. |

---

## 2. Физика поворота и торможения (`FixedUpdate`)

В каждом физическом кадре `Rudder` вычисляет локальную продольную скорость корабля, момент поворота и сопротивление воды:

```csharp
private void FixedUpdate()
{
    // 1. Продольная скорость судна в локальных координатах
    shipForwardVelocity = shipRigidbody.transform.InverseTransformDirection(shipRigidbody.velocity).z;
    float absSpeed = Mathf.Abs(shipForwardVelocity);

    // 2. Базовый момент рыскания
    float torque = -1f * rudderPower * absSpeed * currentAngle;

    // 3. Гидродинамическое торможение от отклоненного пера руля
    float dragForce = Mathf.Abs(torque) * dragPower;
    if (shipForwardVelocity < 0f)
        dragForce = -dragForce;
    shipRigidbody.AddRelativeForce(Vector3.back * dragForce, ForceMode.Force);

    // 4. Статический момент (работа руля на нулевой скорости)
    torque -= currentAngle * 3.1f;

    // 5. Приложение крутящего момента (с инверсией заднего хода)
    if (shipForwardVelocity > 0f)
        shipRigidbody.AddRelativeTorque(Vector3.up * torque, ForceMode.Force);
    else
        shipRigidbody.AddRelativeTorque(Vector3.up * -torque, ForceMode.Force);

    // 6. Расчет гидродинамического давления воды
    currentTension = shipRigidbody.angularVelocity.normalized.y * shipForwardVelocity * 0.25f;
    ApplyCenteringForce();
    ...
}
```

### 2.1. Крутящий момент рыскания (`torque`)
Основная сила поворота определяется формулой:

$$T_{\text{yaw}} = -\text{rudderPower} \cdot |v_{\text{forward}}| \cdot \theta_{\text{rudder}} - 3.1 \cdot \theta_{\text{rudder}}$$

- **Зависимость от скорости:** эффективность поворота прямо пропорциональна скорости лодки $|v_{\text{forward}}|$ и углу отклонения $\theta_{\text{rudder}}$.
- **Статический член (`-3.1 * angle`):** даже при нулевой скорости судна отклоненный руль создает небольшой постоянный момент рыскания. Это имитирует давление кильватерной струи или бокового дрейфа, позволяя поворачивать неподвижное судно под парусами.

### 2.2. Гидродинамическое сопротивление (`dragForce`)
Отклонение руля создает лобовое сопротивление, замедляющее лодку:

$$F_{\text{drag}} = |T_{\text{yaw}}| \cdot \text{dragPower}$$

Вектор силы `Vector3.back * dragForce` прикладывается в локальной системе координат корпуса. При стандартном `dragPower = 0.02` резкая перекладка руля на 30° ощутимо снижает скорость корабля.

### 2.3. Инверсия руления при заднем ходу
Если судно движется назад (`shipForwardVelocity < 0f`), крутящий момент прикладывается с обратным знаком (`Vector3.up * -torque`), что соответствует реальному поведению судна при реверсе.

---

## 3. Гидродинамическая центрирующая сила (`ApplyCenteringForce`)

Во время поворота набегающий поток воды давит на перо руля, стремясь вернуть его в нейтральное положение (0°):

```csharp
private void ApplyCenteringForce()
{
    float delta = currentTension * pushbackPower * Time.deltaTime;
    JointSpring spring = hinge.spring;
    spring.targetPosition += delta * 0.1f;
    hinge.spring = spring;
}
```

### Анализ работы пружины
1. Расчетное давление `currentTension` зависит от направления угловой скорости поворота (`angularVelocity.normalized.y`) и скорости хода (`shipForwardVelocity * 0.25f`).
2. Метод сдвигает целевое положение пружины (`hinge.spring.targetPosition`) в сторону нейтрали.
3. Именно поэтому брошенный штурвал на ходу медленно «раскручивается» обратно, а игроку требуется удерживать руль или использовать фиксатор курса (`GPButtonSteeringWheel.Lock()`, [заметка 38](38-ropes-rigging-steering.md)).

---

## 4. Загадка класса `RudderNew`

В декомпилированном Assembly-CSharp присутствует неиспользуемый класс `RudderNew : MonoBehaviour`:

```csharp
public class RudderNew : MonoBehaviour
{
    [SerializeField] private float rudderPower = 1000f;
    private float currentAngle;
    private float shipForwardVelocity;
    private float currentResistance;
    private Rigidbody shipRigidbody;
    private RopeControllerSteeringWheel steeringRope;
}
```

Класс состоит всего из 15 строк и не содержит методов `Update`/`FixedUpdate`. Это незаконченный черновик разработчика, указывающий на планировавшийся переход от управления рулем через `HingeJoint` к прямой связи с тросом штурвала (`steeringRope` / `currentResistance`). В релизе v0.38 все лодки продолжают использовать классический `Rudder`.

---

## 5. Практические выводы для мододела

1. **Тюнинг маневренности модовых лодок:** чтобы увеличить маневренность без потери скорости, повышайте `rudderPower`, но снижайте `dragPower` (например, с `0.02f` до `0.008f`).
2. **Компенсация центрирующей силы в автопилотах:** если ваш мод на автопилот управляет углом руля через `hinge.spring.targetPosition`, помните, что `ApplyCenteringForce()` каждый кадр смещает это значение. Для стабильного удержания курса перезаписывайте `targetPosition` в `LateUpdate()` или временно обнуляйте `pushbackPower`.
3. **Учет торможения при поворотах:** алгоритмы AI и автопилотов не должны перекладывать руль на максимальный угол (`±30°`) при незначительной ошибке по курсу, иначе лодка будет постоянно терять скорость из-за `F_drag`.
4. **Поворот с места:** статический компонент момента (`-3.1 * angle`) позволяет разворачивать лодку даже при `shipForwardVelocity == 0`. В модах на реалистичную физику этот компонент можно обнулять для судов без работающего винта/весел.
