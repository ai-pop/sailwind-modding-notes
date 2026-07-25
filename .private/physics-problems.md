# 🔒 Приватный анализ: физика предметов — первопричины и векторы исправления

> Только для внутреннего использования. Содержит конкретные рецепты исправления физики.
> Не публиковать в основной ветке заметок.

---

## 1. Первопричина парения бочек над водой

### Цепочка

```
ShipItem.floaterHeight = 1.6 (умолчание)
    ↓
ItemRigidbody.Start():
    floater._raiseObject = item.floaterHeight  // = 1.6
    ↓
SimpleFloatingObject (Crest):
    поднимает объект на 1.6 единиц НАД расчётной ватерлинией
    ↓
Бочка стоит НАД водой, не погружаясь
```

### Корень

`_raiseObject` в SimpleFloatingObject — это **добавочная высота над водой**, а не осадка. Значение 1.6 означает «поднять центр объекта на 1.6м выше поверхности». Для бочки радиусом ~0.5м это означает полный отрыв от воды.

### Вектор исправления

Установить `_raiseObject ≈ 0.3` через Harmony-патч на `ItemRigidbody.Start()`:
```
floater._raiseObject = item.floaterHeight * 0.2f;  // вместо 1.6 → 0.32
```
Либо полностью заменить `SimpleFloatingObject` на кастомный buoyancy на основе `Buoyancy` (как у лодок), что даст правильную модель Архимеда.

---

## 2. Отсутствие вращательного трения в воде

```
floater._dragInWaterRotational = 0.02f;
```

Это значение означает что предмет в воде вращается практически без сопротивления. Для сравнения: у лодок `Buoyancy.dampCoeff = 0.1` для линейного демпфирования.

Рекомендуемый диапазон: 1.0–3.0.

---

## 3. Drag не зависит от среды

```
rigidbody.drag = 1.2f;  // Всегда, и в воздухе и в воде
```

В реальности drag в воздухе ~0.01–0.1, в воде ~2–10.

Требуется разделение: проверка `floater.InWater` → разный drag.
Либо использование `RigidbodyDirectionalDrag` с динамическими параметрами.

---

## 4. Предметы на лодке — кинематические

Условие `onBoat && !held` → `isKinematic = true`.

Это означает что предметы на палубе не испытывают инерции при качке/ускорении лодки. Их позиция жёстко привязана через `MoveItemToWalkColRigidbody`.

Для реалистичной физики нужно:
- Оставлять `isKinematic = false` на лодке
- Использовать `ConfigurableJoint` или `FixedJoint` с ограниченным движением
- Добавлять силы инерции при ускорении лодки

---

## 5. Нет сил инерции на предметах в лодке

При ускорении/повороте лодки предметы должны смещаться. Сейчас — нет.

Решение: в `ItemRigidbody.FixedUpdate()` при `onBoat`:
```csharp
Vector3 boatAccel = (boatRigidbody.velocity - lastBoatVelocity) / Time.fixedDeltaTime;
rigidbody.AddForce(-boatAccel * rigidbody.mass, ForceMode.Force);
```

---

## 6. flotaterHeight используется некорректно

`ShipItem.floaterHeight` передаётся в `_raiseObject` (смещение ВВЕРХ), но семантически это «высота поплавка» (осадка). Правильное использование:
- Если floaterHeight = центр плавучести: передавать как есть
- Если floaterHeight = осадка: инвертировать знак

---

## 7. SimpleFloatingObject — чёрный ящик

Код этого компонента вшит в Assembly-CSharp.dll как часть Crest Ocean System. Его нельзя прочитать через ILSpy/dnSpy потому что это нативный код или обфусцированный IL.

**Варианты обхода:**
1. **Harmony-патч** на `ItemRigidbody.Start()` — изменить параметры до передачи в SimpleFloatingObject
2. **Harmony-патч** на методы SimpleFloatingObject — перехватить `Update`/`FixedUpdate` и корректировать силы
3. **Замена компонента** — удалить SimpleFloatingObject и добавить свой Buoyancy-компонент
4. **Префиксный патч** на `ItemRigidbody.Start()` — предотвратить создание SimpleFloatingObject, создать свой

---

## 8. Таблица: что и где патчить

| Что меняем | Где | Как |
|------------|-----|-----|
| `_raiseObject` | `ItemRigidbody.Start()` | Postfix: `__instance.floater._raiseObject = newValue` |
| `_dragInWaterRotational` | `ItemRigidbody.Start()` | То же |
| drag в воздухе/воде | `ItemRigidbody.FixedUpdate()` | Добавить проверку `floater.InWater` |
| isKinematic на лодке | `ItemRigidbody.FixedUpdate()` | Убрать условие, заменить на Joint |
| Силы инерции | `ItemRigidbody.FixedUpdate()` | Добавить расчёт ускорения лодки |
| Decollision multiplier | `PickupableItemCollisionChecker` | Изменить 1.8 |
| Масса игрока в лодке | `BoatMass.UpdateMass()` | Изменить 160 |

---

*Приватный документ. Не для публичного репозитория.*
