# 77. Искусственный интеллект NPC: навигация, избегание столкновений, графы вейпоинтов и распорядок дня

Полный разбор алгоритмов искусственного интеллекта и управления неигровыми судами в Sailwind v0.38 (`Assembly-CSharp.dll`). Заметка дополняет общий обзор населения мира в [заметке 20](20-npcs-world-population.md) и раскрывает точные физические формулы навигации, логику работы с парусами и архитектуру графов путей.

---

## 1. Архитектура управления NPC-лодкой (`NPCBoatController`)

`NPCBoatController` — центральный компонент фоновых судов, перемещающихся по игровому миру. В отличие от корабля игрока, использующего аэродинамические расчеты сил паруса (`Sail`, [заметка 17](17-wind-and-sails.md)) и гидродинамику руля (`Rudder`, [заметка 80](80-rudder-hydrodynamics-centering-force-and-steering-torque.md)), NPC используют **упрощенную кинематико-динамическую модель**, позволяющую им двигаться стабильно и предсказуемо.

### Поля компонента

| Поле | Тип | Описание |
|---|---|---|
| `speed` | `float` | Базовая скорость линейного движения лодки. |
| `turnSpeed` | `float` | Скорость поворота (множитель крутящего момента). |
| `sailSpeed` | `float` | Скорость настройки (тримминга) углов парусов. |
| `sailResistance` | `float` | Порог сопротивления паруса для авто-тримминга. |
| `sailAngleControllers` | `RopeControllerSailAngle[]` | Массив контроллеров шкотов (угла поворота парусов). |
| `sailReefControllers` | `RopeControllerSailReef[]` | Массив контроллеров фалов/рифления (степени развертывания). |
| `currentTarget` | `Transform` | Текущий целевой вейпоинт (`NPCBoatWaypoint`). |
| `currentTargetIndex` | `int` | Индекс текущего вейпоинта в `NPCBoatWaypointManager`. |
| `currentDock` | `Transform` | Текущая стоянка (вейпоинт дока при швартовке). |
| `currentDockIndex` | `int` | Индекс вейпоинта стоянки. |
| `parkedTimer` | `float` | Таймер ожидания в порту/доке (в игровых часах). |
| `horizon` | `BoatHorizon` | Оптимизатор активности лодки в зависимости от расстояния до игрока. |
| `otherBoatInRange` | `bool` | Флаг обнаружения препятствия (другой лодки) в радиусе 15 метров. |

---

## 2. Алгоритмы движения и физики в `FixedUpdate()`

Каждый физический кадр (`FixedUpdate()`, ~45.5 Гц) `NPCBoatController` выполняет проверку оптимизации и расчет навигации:

```csharp
private void FixedUpdate()
{
    if (!horizon.closeToPlayer)
        return;

    if (boatColCheckTimer <= 0f)
        CheckOtherBoatCol();
    else
        boatColCheckTimer -= Time.deltaTime;

    if (otherBoatInRange)
        return;

    if (currentTarget != null)
    {
        AddForceTowards(currentTarget);
        AddRotationTowards(currentTarget);
    }

    if (currentDock != null)
    {
        AddForceTowards(currentDock);
        transform.rotation = Quaternion.Lerp(transform.rotation, currentDock.rotation, 0.005f);
        parkedTimer += Time.deltaTime * Sun.sun.timescale;
        if (parkedTimer > 1f)
        {
            parkedTimer = 0f;
            NPCBoatWaypoint waypoint = currentDock.GetComponent<NPCBoatWaypoint>();
            if (waypoint != null && waypoint.GetNextDestination() != null)
            {
                currentTarget = waypoint.GetNextDestination().transform;
                currentTargetIndex = waypoint.GetNextDestination().index;
                currentDock = null;
                currentDockIndex = -1;
            }
        }
    }
}
```

### 2.1. Линейный привод (`AddForceTowards`)

NPC-лодки не рассчитывают реальную силу ветра на парусах для создания тяги. Вместо этого применяется прямая сила в направлении цели `target`:

$$\vec{F}_{\text{linear}} = \vec{u}_{\text{target}} \cdot \left(\text{speed} + \|\vec{v}_{\text{wind}}\| \cdot \text{speed} \cdot 0.05\right) \cdot m_{\text{boat}}$$

Где:
- $\vec{u}_{\text{target}}$ — нормализованный вектор направления к цели (`(target.position - transform.position).normalized`).
- $\|\vec{v}_{\text{wind}}\|$ — текущая скорость ветра (`Wind.currentWind.magnitude`).
- $m_{\text{boat}}$ — масса лодки (`rigidbody.mass`).

> **Важная особенность:** поскольку вектор $\vec{F}_{\text{linear}}$ направлен строго к вейпоинту, NPC-лодка способна двигаться **прямо против ветра (в левтик)**, не совершая галсировки. Ветер лишь дает 5%-й бонус к скорости за каждую единицу силы ветра.

### 2.2. Управление курсом (`AddRotationTowards`)

Управление рысканием (yaw) также обходит гидродинамику руля:

```csharp
Vector3 dir = (target.position - transform.position).normalized;
float angle = Vector3.SignedAngle(transform.forward, dir, Vector3.up);
if (angle < 0f)
    rigidbody.AddTorque(Vector3.up * -turnSpeed * rigidbody.mass * 20f);
else if (angle > 0f)
    rigidbody.AddTorque(Vector3.up * turnSpeed * rigidbody.mass * 20f);
```

Крутящий момент зависит исключительно от знака угла отклонения (`SignedAngle`) и массы судна ($20 \cdot m_{\text{boat}} \cdot \text{turnSpeed}$).

---

## 3. Система избегания столкновений (`CheckOtherBoatCol`)

Для предотвращения аварий между NPC и игроком используется сферический запрос физики:

```csharp
private void CheckOtherBoatCol()
{
    boatColCheckTimer = Random.Range(0.5f, 1.5f);
    otherBoatInRange = false;
    Collider[] colliders = Physics.OverlapSphere(transform.position, 15f);
    foreach (Collider col in colliders)
    {
        if (col.CompareTag("Boat") && col != this.col)
        {
            otherBoatInRange = true;
            break;
        }
    }
}
```

### Анализ механики
1. **Интервал опроса:** опрос `Physics.OverlapSphere` выполняется асинхронно с рандомным интервалом от **0.5 до 1.5 секунд**, что снижает нагрузку на CPU.
2. **Радиус:** зона безопасности составляет **15 метров**.
3. **Реакция на тег `Boat`:** если в зоне обнаружен любой коллайдер с тегом `Boat` (кроме собственного), флаг `otherBoatInRange = true` мгновенно прерывает `FixedUpdate()`. Лодка перестает прикладывать линейную тягу и крутящий момент.
4. **Уязвимость в моддинге:** если мод создает кастомные объекты с тегом `Boat` или оставляет лодки рядом с NPC-вейпоинтами, NPC будут бесконечно стоять в радиусе 15 м, создавая «пробку».

---

## 4. Автоматическое управление парусами (`Update`)

Несмотря на то, что паруса не двигают лодку физически, NPC активно настраивают их визуальное состояние через `RopeControllerSailReef` (рифление) и `RopeControllerSailAngle` (шкоты):

```csharp
// 1. Управление развертыванием (рифлением)
if (currentTarget != null)
{
    foreach (RopeControllerSailReef reef in sailReefControllers)
        reef.currentLength = Mathf.Min(1f, reef.currentLength + 0.15f * Time.deltaTime);
}
else
{
    foreach (RopeControllerSailReef reef in sailReefControllers)
        reef.currentLength = Mathf.Max(0f, reef.currentLength - 0.15f * Time.deltaTime);
}

// 2. Управление углов к ветру (тримминг)
if (currentTarget != null)
{
    foreach (RopeController angleCol in sailAngleControllers)
    {
        angleCol.changed = true;
        if (angleCol.currentResistance > Wind.currentWind.magnitude)
            angleCol.currentLength = Mathf.Min(1f, angleCol.currentLength + sailSpeed * Time.deltaTime * 0.05f);
        else
            angleCol.currentLength = Mathf.Max(0f, angleCol.currentLength - sailSpeed * Time.deltaTime * 0.05f);
    }
}
```

### Алгоритм авто-тримминга
- **Поднятие/спуск:** при движении к цели (`currentTarget != null`) паруса разворачиваются со скоростью `0.15 / c` (полное поднятие занимает **6.67 секунды**). На стоянке (`currentTarget == null`) паруса сворачиваются с той же скоростью.
- **Адаптация к ветру:** AI сравнивает текущее натяжение троса (`currentResistance`) с силой ветра (`Wind.currentWind.magnitude`). Если сопротивление превышает скорость ветра, NPC травит шкот (`currentLength += ...`), если меньше — выбирает.

---

## 5. Графы навигационных вейпоинтов (`NPCBoatWaypoint`)

Сеть путей NPC в мире строится на направленном графе, управляемом синглтоном `NPCBoatWaypointManager.instance`.

```
[NPCBoatWaypoint 0: navigationWaypoint=true]
         │
         ▼ (случайный выбор из destinations[])
[NPCBoatWaypoint 3: navigationWaypoint=true]
         │
         ▼
[NPCBoatWaypoint 7: navigationWaypoint=false (Док/Порт)]
         │
         ├──► Ожидание: parkedTimer > 1.0 (в часах × timescale)
         │
         ▼ (выход по истечении таймера)
[NPCBoatWaypoint 12: navigationWaypoint=true]
```

### Логика триггеров (`OnTriggerEnter`)

Когда лодка достигает вейпоинта (`other.transform == currentTarget`), происходит переключение:

| Тип вейпоинта | Значение поля | Поведение NPC-лодки |
|---|---|---|
| **Навигационный** | `navigationWaypoint == true` | Мгновенно выбирает следующий вейпоинт из массива `destinations[]` методом `Random.Range(0, destinations.Length)`. |
| **Док / Стоянка** | `navigationWaypoint == false` | Переходит в режим парковки: `currentDock = currentTarget; currentTarget = null;`. Начинается отсчет `parkedTimer`. |

---

## 6. Специализированные AI-суда

### 6.1. Рыбацкие лодки (`NPCFishingBoat`)

`NPCFishingBoat` расширяет `NPCBoatController`, подчиняя лодку суточному распорядку на основе местного солнечного времени (`Sun.sun.localTime`):

```csharp
private void Update()
{
    bool isFishingTime = (Sun.sun.localTime > 5.5f && Sun.sun.localTime < 9.5f) ||
                         (Sun.sun.localTime > 13.5f && Sun.sun.localTime < 17.5f);
    if (isFishingTime && !goingFishing)
        GoFishing();
    else if (!isFishingTime && goingFishing)
        GoHome();
}
```

- **Утренний лов:** `05:30 — 09:30`.
- **Вечерний лов:** `13:30 — 17:30`.
- **Синхронизация координат:** позиции точек лова (`target`) и дома хранятся в реальных координатах (`RealPos`), преобразуемых в текущие локальные (`ShiftingPos`) через `FloatingOriginManager.instance.RealPosToShiftingPos(realPos)`, что гарантирует стабильность AI при сдвиге плавающего начала координат.

### 6.2. Торговые суда-симуляции (`TraderBoat`)

`TraderBoat` ([заметка 13](13-economy-markets-currency.md)) не является физической лодкой с `Rigidbody`. Это фоновый экономический агент, связывающий рынки островов:

```csharp
private void Update()
{
    if ((!EconomyUI.instance.uiActive || Application.isEditor) && !GameState.currentShipyard)
    {
        if (waitTime > 0f)
            waitTime -= Time.deltaTime * Sun.sun.timescale * 100f;
        else if (currentDestination != null)
            EnterIsland();
        else if (currentIslandMarket != null)
            LeaveIsland();

        if (lastIslandMarket != null && currentDestination != null)
            transform.position = Vector3.Lerp(currentDestination.transform.position, lastIslandMarket.transform.position, waitTime / currentTripTime);
    }
}
```

- **Ускоренное время:** таймер ожидания `waitTime` убывает в **100 раз быстрее** игрового времени (`timescale * 100f`).
- **Блокировка при UI:** перемещение и расчеты `TraderBoat` приостанавливаются, когда игрок открывает `EconomyUI` или находится в верфи (`currentShipyard`), чтобы цены на рынке не менялись прямо во время сделки.
- **Интерполяция:** позиция в мире вычисляется линейной интерполяцией (`Vector3.Lerp`) между портами отправления и назначения.

---

## 7. Практические выводы для мододела

1. **Создание кастомных путей NPC:** для добавления новых маршрутов достаточно создать объекты с компонентом `NPCBoatWaypoint`, зарегистрировать их в `NPCBoatWaypointManager.instance` и связать через массив `destinations[]`.
2. **Абсолютная проходимость против ветра:** NPC-лодки не требуют попутного ветра; при балансировке скорости кастомных NPC учитывайте, что формула `speed + wind * speed * 0.05` дает постоянную тягу в любую сторону.
3. **Безопасность тега `"Boat"`:** если ваш мод добавляет на воду объекты, не являющиеся кораблями (крупные буи, плавающие платформы), **не назначайте им тег `Boat`**, иначе проплывающие NPC будут останавливаться за 15 метров от них.
4. **Перехват торговых путей:** список всех активных торговых судов доступен глобально через `TraderBoat.traderBoats`. Изменение `carriedGoods` и `carriedPriceReports` позволяет модам влиять на межостровную экономику и генерируемые в тавернах слухи ([заметка 78](78-dialogues-rumors-bribery-and-contraband-customs.md)).
5. **Сохранение состояния (`NPCBoatData`):** сохраняются только индексы `currentDock`, `currentTarget` и значение `parkedTimer`. Кастомные изменения позиции или скорости после перезагрузки сбросятся к параметрам префаба и координатам текущего вейпоинта.
