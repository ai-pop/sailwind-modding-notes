# 66. Фриз вращения предметов — все механизмы

> Каждый код-путь, блокирующий, замораживающий или подавляющий вращение предмета.
> Критично для физических модов. Дополняет заметку 64 (Физическая модель).

---

## 1. MeshCollider → сон → полный кинематический фриз

**Место:** `ItemRigidbody.FixedUpdate()` строка ~521

```csharp
if (meshCol && rigidbody.IsSleeping() && dynamicColTimer <= 0f)
    flag2 = true;   // → isKinematic = true → ВРАЩЕНИЕ + ПОЗИЦИЯ ЗАМОРОЖЕНЫ
```

**Цепочка:**
1. Предмет с MeshCollider в воде
2. Колебания затухают → Rigidbody.Sleep()
3. `dynamicColTimer` — 6 секунд (установлен при отпускании)
4. После 6с сна: `isKinematic = true`
5. Предмет — статуя, angularVelocity обнулён Unity

**Затронуто:** Всё со сложной геометрией. Box/Capsule — не подвержены.

---

## 2. HangableItem.LateUpdate — блокировка осей

```csharp
if (currentHook != null)
{
    Vector3 ea = transform.eulerAngles;
    if (lockX) ea.x = rotX;    // ← ФРИЗ X
    if (lockZ) ea.z = rotZ;    // ← ФРИЗ Z
    transform.eulerAngles = ea;
}
```

Умолчания: `lockX=true, lockZ=true`. Свободно только Y.

---

## 3. attached=true → кинематик

| Путь | Место |
|------|-------|
| `wallAttachment` в префабе | `ShipItem.LoadAfterDelay()` |
| `CrateInventory.InsertItem()` | строка ~64 |
| `HangableItem.ConnectJoint()` | строка ~68 |

---

## 4. GameState.sleeping → глобальный кинематик

```csharp
if (!GameState.playing || GameState.sleeping || GameState.recovering
    || GameState.inBed || GameState.currentShipyard)
    flag2 = true;
```

После пробуждения: предметы снова не-кинематические, но angularVelocity обнулён.

---

## 5. angularDrag = масса × 0.1

| Масса | angularDrag |
|:-----:|:-----------:|
| 0.8 кг | 0.08 |
| 5.0 кг | 0.5 |
| 14 кг | 1.4 |

Ниже 1.0 — вращение почти без трения.

---

## 6. ResetPos() — скорость обнулена, угловая сохранена

```csharp
rigidbody.velocity = Vector3.zero;
// angularVelocity НЕ тронут
```

---

## 7. CrateInventory.LateUpdate — лок поворота

```csharp
item.transform.rotation = transform.rotation;  // Каждый кадр
```

---

## 8. Crest _dragInWaterRotational = 0.02

Ванильный floater: вращательное трение в воде практически отсутствует.

---

## Сводка

| # | Механизм | Эффект | Задержка |
|:--|----------|:------:|:--------:|
| 1 | MeshCollider сон → кинематик | **Полный фриз** | ~6с |
| 2 | HangableItem LateUpdate | **Блок X+Z** | Мгновенно |
| 3 | attached=true | **Полный фриз** | Мгновенно |
| 4 | GameState.sleeping | **Полный фриз** | Мгновенно |
| 5 | angularDrag=mass×0.1 | ~Нет трения | Мгновенно |
| 6 | ResetPos сохраняет angularVel | Вращение живёт | При дропе |
| 7 | CrateInventory LateUpdate | **Лок к ящику** | Каждый кадр |
| 8 | Crest _dragInWaterRotational | ~Нет трения | Мгновенно |

---

*Извлечено из Assembly-CSharp.dll (Sailwind v0.38).*
