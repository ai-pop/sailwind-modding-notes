# 82. Плавающий центр мира: ShiftingRigidbody, скачки PhysX и сохранение импульса

Полный технический анализ механики плавающего начала координат (`FloatingOriginManager`) и компонента сохранения импульса (`ShiftingRigidbody`) в Sailwind v0.38 (`Assembly-CSharp.dll`). Эта заметка **критически важна для разработки модов на физику предметов и судов**: она объясняет, почему физические тела теряют скорость или проваливаются сквозь палубу при сдвиге мира, и как движок предотвращает эти баги в ванильной игре.

---

## 1. Проблема сдвига начала координат (Floating Origin)

В игровом мире Sailwind расстояния между островами составляют десятки километров. Чтобы координаты в Unity не превышали пределы точности чисел с плавающей запятой (`float`) и физический движок PhysX не дрожал, синглтон `FloatingOriginManager.instance` каждые ~1000 метров вызывает сдвиг мира:

```csharp
public void ShiftOrigin(Vector3 shift)
{
    // Сдвигает все корневые объекты сцены на -shift
    ...
}
```

### 1.1. Что происходит с PhysX при сдвиге на 1000 м
Когда координата объекта мгновенно изменяется на вектор `shift`, движок PhysX может интерпретировать это как:
1. Мгновенный колоссальный импульс скорости (`v = dS / dt`);
2. Нарушение контактов между предметом на палубе и коллайдером лодки;
3. Ложное срабатывание датчиков воды и плавучести (`BoatProbes`, `SimpleFloatingObject`).

---

## 2. Архитектура `ShiftingRigidbody`

Для защиты физических тел (кораблей и активных предметов) от десинхронизации при сдвиге мира на объект вешается компонент `ShiftingRigidbody : MonoBehaviour`. При инициализации он регистрируется в глобальном списке:

```csharp
private void Start()
{
    rigidbody = GetComponent<Rigidbody>();
    wasKinematic = rigidbody.isKinematic;
    wasSleepThreshold = rigidbody.sleepThreshold;
    boatProbes = GetComponent<BoatProbes>();
    FloatingOriginManager.shiftingRigidbodies.Add(this);
}
```

### 2.1. Подготовка к сдвигу (`PrepareForShifting`)

Перед тем как `FloatingOriginManager` сдвинет координаты сцены, он вызывает `PrepareForShifting()` у всех зарегистрированных `ShiftingRigidbody`:

```csharp
public void PrepareForShifting()
{
    shifting = true;
    preservedVelocity = rigidbody.velocity;
    preservedAngularVel = rigidbody.angularVelocity;
    if (stopProbes)
        boatProbes.dontUpdateVelocity = true;
    if (setKinematic)
        rigidbody.isKinematic = true;
    if (setToSleep)
        rigidbody.Sleep();
}
```

| Параметр / Действие | Назначение |
|---|---|
| `preservedVelocity` / `preservedAngularVel` | Кэширует точные векторы линейной и угловой скорости тела **до** перемещения. |
| `stopProbes == true` | Блокирует расчет гидродинамических сил в `BoatProbes` (`dontUpdateVelocity = true`), чтобы скачок позиций относительно сетки волн не вызвал гигантский вертикальный импульс. |
| `setKinematic == true` | Временно переводит `Rigidbody` в кинематический режим, отключая расчет столкновений и сил PhysX на время сдвига. |
| `setToSleep == true` | Принудительно усыпляет твердое тело (`Sleep()`). |

---

## 3. Восстановление импульса и многокадровая задержка (`DoRestoreMomentum`)

После сдвига координат вызывается `RestoreMomentum()`, запускающий корутину `DoRestoreMomentum()`:

```csharp
private IEnumerator DoRestoreMomentum()
{
    if (preserveVelocity)
        rigidbody.velocity = preservedVelocity;
    if (preserveAngular)
        rigidbody.angularVelocity = preservedAngularVel;
    if (wakeUp)
        rigidbody.WakeUp();
    shifting = false;

    for (int i = 0; i < waitForFrames; i++)
        yield return new WaitForEndOfFrame();

    for (int j = 0; j < waitForFixedFrames; j++)
        yield return new WaitForFixedUpdate();

    if (stopProbes)
        boatProbes.dontUpdateVelocity = false;
}
```

### 3.1. Почему требуется ожидание `waitForFixedFrames`
Обратите внимание на ключевую особенность корутины:
1. Линейная и угловая скорости возвращаются немедленно в первом кадре после сдвига.
2. Однако **расчет сил плавучести (`BoatProbes.dontUpdateVelocity = false`) остается заблокированным** на протяжении `waitForFrames` графических кадров и `waitForFixedFrames` физических тактов (`WaitForFixedUpdate`)!
3. **Зачем это сделано?** После сдвига плавающего центра сетка волн Crest и иерархия коллайдеров Unity пересчитывают свои пространственные индексы (Broadphase / Tree rebuild). Если включить расчет плавучести сразу, корабль или предмет может получить ошибочную высоту волны и подлететь в воздух.

---

## 4. Практические выводы для мода на физику предметов

1. **Обязательная регистрация кастомных тел:** Если ваш мод создает новые динамические `Rigidbody` (например, реалистичную физику грузов, ящиков или кастомные суда), **обязательно добавляйте на них `ShiftingRigidbody`** или вручную подписывайтесь на события сдвига `FloatingOriginManager`.
2. **Проблема некинматических предметов на палубе:** В ванильной игре предметы на палубе прикреплены к локальной системе координат лодки (`BoatLocalItems`). Если ваш мод делает предметы полностью независимыми динамическими `Rigidbody` на палубе, во время вызова `ShiftOrigin` они могут получить импульс проникновения (penetration) сквозь палубу.
3. **Безопасная телепортация предметов:** При программном перемещении предметов или судов через скрипт всегда синхронизируйте координаты через `FloatingOriginManager.instance.ShiftingPosToRealPos` / `RealPosToShiftingPos`.
4. **Блокировка физики волн:** Если вы реализуете кастомную плавучесть для предметов, приостанавливайте применение вертикальных сил на `2–3` кадра (`WaitForFixedUpdate`) после любого сдвига плавающего начала координат.
