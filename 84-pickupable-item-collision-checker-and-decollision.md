# 84. Система проверки столкновений предметов и деколлизия (PickupableItemCollisionChecker)

Технический анализ подсистем проверки столкновений удерживаемых предметов (`PickupableItemCollisionChecker`) и переопределения триггеров (`ItemTriggerOverride`) в Sailwind v0.38 (`Assembly-CSharp.dll`). Эта заметка напрямую относится к **моддингу физики предметов и систем размещения груза на палубе**.

Связано с [47](47-item-holding-pickup-flow.md), [54](54-go-pointer-big-item-decollision.md) и [58](58-clickability-layers-manual-held-break.md).

---

## 1. Проверка столкновений удерживаемого предмета (`PickupableItemCollisionChecker`)

Когда игрок держит предмет в руках (`item.held != null`), визуальная модель предмета следует за камерой или курсором. Чтобы предмет не застревал в стенах или палубе при броске, компонент `PickupableItemCollisionChecker` отслеживает пересечения с внешними коллайдерами через список `collidedCols`.

### 1.1. Алгоритм вычисления деколлизии (`GetDecollision`)

Если в настройках включено выталкивание (`Settings.enableDecol == true`), метод `GetDecollision()` вычисляет вектор выталкивания предмета из препятствий:

```csharp
public Vector3 GetDecollision()
{
    if (!Settings.enableDecol)
        return Vector3.zero;

    decollisionVector = Vector3.zero;
    Vector3 dir = default(Vector3);
    float dist = default(float);

    foreach (Collider collidedCol in collidedCols)
    {
        Physics.ComputePenetration(
            GetComponent<Collider>(), transform.position, transform.rotation,
            collidedCol, collidedCol.transform.position, collidedCol.transform.rotation,
            ref dir, ref dist
        );
        decollisionVector += dir * dist * 1.8f;
    }
    return transform.InverseTransformVector(decollisionVector);
}
```

#### Анализ коэффициента выталкивания (1.8× Over-relaxation)
Обратите внимание на множитель `1.8f` в строке `dir * dist * 1.8f`:
- Движок не просто сдвигает предмет на глубину проникновения `dist`. Он применяет **коэффициент сверхрелаксации (Over-relaxation) 1.8×**, выталкивая предмет на 80% дальше границы препятствия.
- Это гарантирует, что на следующем кадре после броска физический twin-коллайдер предмета гарантированно окажется снаружи палубы или стены и не вызовет взрывной контакт PhysX.

---

## 2. Порог допустимого пересечения при броске (`allowObstructedDropping`)

В каждом кадре `Update()` система определяет максимальную глубину проникновения (`currentDecolDistance`) среди всех коллайдеров в `collidedCols`:

```csharp
public void Update()
{
    if (item.held == null || item.GetCurrentInventorySlot() > -1)
    {
        collisions = 0;
        return;
    }

    UpdateDecolDistance();
    allowObstructedDropping = currentDecolDistance < 0.06f;

    bool enableRedOutline = false;
    if (collisions > 0)
    {
        if (item.big && !allowObstructedDropping)
            enableRedOutline = true;
    }
    ...
}
```

### 2.1. Правило 6 сантиметров (`0.06f`)
| Глубина проникновения | Значение флага | Поведение при броске/размещении |
|---|:--:|---|
| `< 0.06 м` (6 см) | `allowObstructedDropping = true` | Предмет **можно бросить или поставить**, даже если его край слегка врезается в палубу, стол или стену. Красный контур не загорается. |
| `≥ 0.06 м` | `allowObstructedDropping = false` | Для крупных предметов (`item.big == true`) загорается красный контур, запрещающий размещение. Бросок заблокирован. |

> **Зачем это нужно?** Палубы кораблей в Sailwind имеют сложную геометрию с изгибами и досками. Без 6-сантиметрового допуска игроки не смогли бы ровно ставить ящики друг на друга или на палубу.

---

## 3. Отложенное копирование триггеров (`ItemTriggerOverride`)

Для оптимизации проверки взаимодействия у предметов используется компонент `ItemTriggerOverride`, который копирует размеры физического `BoxCollider` в триггерный коллайдер с задержкой в 3 кадра:

```csharp
private IEnumerator LoadAfterDelay()
{
    yield return new WaitForEndOfFrame();
    yield return new WaitForEndOfFrame();
    yield return new WaitForEndOfFrame();

    BoxCollider targetBox = item.GetComponent<BoxCollider>();
    targetBox.center = col.center;
    targetBox.size = col.size;
    col.enabled = false;
    ...
}
```

**Причина 3-кадровой задержки:** при инициализации префаба (`Awake`) компоненты масштабирования или кастомные модификаторы палубы могут изменять размеры `item`. Задержка в 3 `WaitForEndOfFrame` гарантирует, что триггер скопирует итоговые финальные габариты предмета.

---

## 4. Практические выводы для мододела

1. **Использование порога `0.06f` в модах на размещение:** если ваш мод добавляет магнитное прилипание грузов (snap) или точное палубное позиционирование, следите за тем, чтобы итоговое пересечение с палубой не превышало **0.05 м (5 см)**. Иначе сработает защита `PickupableItemCollisionChecker`, и ящик подсветится красным.
2. **Учет множителя `1.8f` при деколлизии:** если вы патчите алгоритмы выбрасывания предметов, учитывайте, что ванильная деколлизия выталкивает объект почти в два раза дальше реального пересечения. Это может приводить к смещению грузов на противоположный край узких полок.
3. **Трехкадровая задержка геометрии:** при программном спавне предметов через код не пытайтесь читать размеры триггерных коллайдеров `ItemTriggerOverride` в кадре `Instantiate()`. Ждите минимум 3 кадра `WaitForEndOfFrame()`.
