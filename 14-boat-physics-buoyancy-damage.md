# 14. Физика корпуса: плавучесть, масса, урон и потопление

Разбор подсистемы физики лодки. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Океан построен на кастомном форке **Crest** (`Ocean.Singleton`, `GetWaterHeightAtLocation2`, `GetChoppyAtLocation`). Полезно для модов на физику, баланс живучести и реализм.

## Плавучесть (`Buoyancy`) — «блоб»-модель

Классическая blob-buoyancy: корпус аппроксимируется сеткой точек-«блобов» по площади `BoxCollider`, каждая сэмплирует высоту волны и прикладывает выталкивающую силу.

| Поле | По умолч. | Назначение |
|------|:--:|-----------|
| `magnitude` | 2 | Множитель выталкивающей силы. |
| `SlicesX` / `SlicesZ` | 2 / 2 | Сетка точек (минимум 2×2 = 4 блоба). |
| `CenterOfMassOffset` | -1 | Смещение центра масс вниз (остойчивость). |
| `dampCoeff` | 0.1 | Коэффициент демпфирования (гашение качки). |
| `interpolation` | 3 | Сглаживание силы по кадрам. |
| `ypos` | 0 | Вертикальное смещение точек сэмпла. |
| `ChoppynessAffectsPosition` / `ChoppynessFactor` | false / 0.2 | Влияние «choppy»-смещения волны на позицию. |
| `WindAffectsPosition` / `WindFactor` | false / 0.1 | Снос ветром. |
| `xAngleAddsSliding` / `slideFactor` | false / 0.1 | Скольжение при крене. |
| `sink` / `sinkForce` | false / 3 | Принудительное затопление (сила на блоб). |
| `useFixedUpdate` | — | Тик в `FixedUpdate` vs `Update`. |

### Как это работает
1. В `Start()` по размеру `BoxCollider` строится сетка `SlicesX × SlicesZ` точек (`blobs`).
2. Каждому блобу назначается **случайный вес** потопления (`sinkForces`, квадрат случайного числа, нормированный на `sinkForce`).
3. В тике для каждого блоба: мировая позиция → высота воды через `ocean.GetWaterHeightAtLocation2(x - choppy, z)` (с учётом `GetChoppyAtLocation`) → сила пропорциональна глубине погружения.
4. `rrigidbody.centerOfMass = (0, CenterOfMassOffset, 0)` — центр масс занижен для самовыравнивания.

### Оптимизация по видимости
Флаги `cvisible` / `wvisible` / `svisible` включают отключение физики вне экрана:
- Если объект невидим (`!_renderer.isVisible`) — `rigidbody.useGravity = false`, а при `wvisible && svisible` расчёт плавучести **полностью пропускается**.
- При возврате в кадр (и если прошло > 15 кадров) объект «прищёлкивается» к текущей высоте волны, чтобы не провалился/не взлетел.

> Это важно для модов, телепортирующих лодки: при возврате в кадр позиция по Y форсируется к высоте воды.

## Динамическая масса (`BoatMass`)

Пересчитывается **каждый кадр только для активной лодки** (`GameState.lastBoat == this.transform`).

```
итоговая масса = selfMass (корпус)
              + partsMass (кастомные детали, BoatCustomParts)
              + Σ масса предметов на палубе (ItemRigidbody, не в слоте инвентаря)
              + Σ масса парусов (по мачтам)
              + 160, если игрок на борту ( GameState.currentBoat.parent == this )
```

### Центр масс
```
centerOfMass = keel.centerOfMass + Σ(смещение_груза × leverageMult)
```
- Смещение каждого предмета берётся от `localPosition`, повёрнутого на `Euler(0,-90,0)`, и взвешивается отношением `item.mass / selfMass`.
- `leverageMult` — множитель «плеча»: насколько сильно груз смещает центр масс (дифферент/крен от загрузки).
- Позиция игрока учитывается через `Refs.observerMirror.transform.localPosition`.
- **Вес игрока захардкожен = 160** единиц массы.

### Масса парусов (`GetSailMass`)
```
sail.mass = realSailPower × 20 + бонус_категории
```
- `junk` / `gaff`: бонус `+realSailPower × 20`
- не `staysail` (прямые и др.): бонус `+realSailPower × 10`
- `staysail`: бонус `0`

Большие паруса реально утяжеляют и поднимают центр тяжести лодки.

## Урон корпуса и потопление (`BoatDamage`)

Трёхпараметрическая модель состояния корпуса:

| Поле | Диапазон | Содержание |
|------|:--:|-----------|
| `hullDamage` | 0..1 | Повреждение корпуса. 1 = максимально разбит. |
| `waterLevel` | 0..1 | Уровень воды в трюме. **≥ 1 = лодка тонет** (`sunk = true`). |
| `oakum` | ≥ 0 | Конопатка (caulking) — замедляет набор воды. |

### Ключевые тюнинг-поля
| Поле | По умолч. | Назначение |
|------|:--:|-----------|
| `durabilityDays` / `wearSteepness` | — | Кривая ежедневного износа. |
| `waterIntakeRate` / `waterDrainRate` | — | Скорость набора/откачки воды. |
| `waterUnitsCapacity` | — | «Ёмкость» корпуса (нормировка конопатки и дождя). |
| `impactDamageMult` | — | Множитель урона от ударов. |
| `minimumImpactVelocity` | 1.5 | Мин. скорость для получения урона. |
| `maxDamagePerImpact` | 0.15 | Кап урона за один удар. |
| `impactCooldown` | 1 | Секунды между ударами с уроном. |
| `waterDrag` | 0.2 | Сопротивление от воды в трюме. |
| `safeAngleLimit` | 35 | Безопасный угол крена. |
| `sinkVerticalPos` | -4 | Позиция по Y для затонувшей лодки. |

### Ежедневный износ (`DailyDamage`, на `Sun.OnNewDay`)
Только для активной лодки (`GameState.lastBoat`):
- Конопатка деградирует: `oakum *= 0.88` (**−12% в день**).
- `hullDamage` растёт по **экспоненциальной кривой износа**: параметризуется `durabilityDays` и `wearSteepness` (формула через `Exp`/`Log` — «+1 день жизни» на каждый тик). Чем старше/изношеннее корпус, тем быстрее накапливается урон.

### Урон от столкновений (`Impact`)
Урон **не** начисляется, если:
- идёт кулдаун (`impactTimer > 0`, = `impactCooldown` = 1 c);
- игрок спит и лодка пришвартована;
- столкновение с тегом `OceanBottom` (дно);
- скорость лодки `< minimumImpactVelocity` (1.5);
- удар о `ShipItem` (предмет).

Иначе: `hullDamage += clamp(force * impactDamageMult, 0, maxDamagePerImpact=0.15)`.

### Динамика воды (`UpdateWaterAndDrag`)
- **Откачка:** `waterLevel -= deltaTime * waterDrainRate * timescale * 10 * InverseLerp(0.2, 0, hullDamage)` — тем быстрее, чем меньше повреждение. (Ручная откачка — `BilgePump`.)
- **Набор воды** (только активная лодка), накапливается «чанками» (`waterIntakeChunk`):
  - дождь: `+= deltaTime * GameState.rainIntensity * (1/waterUnitsCapacity) * 10`
  - пробоины: `+= 1000 * hullDamage * waterIntakeRate * deltaTime * timescale * 0.65`
  - когда `chunk ≥ 1`: `waterLevel += 0.001 * chunk * GetCaulkMult()`.
- **Конопатка** снижает набор: `GetCaulkMult() = 1 - oakum / (hullDamage * waterUnitsCapacity)` (0 при отсутствии урона).
- **Перелив волн** (`Overflow`): захлёстывающая вода добавляет `waterIntakeRate * 0.35 * heightDiff * 4`; будит игрока, если он спит и `waterLevel > 0.1`.
- **Сопротивление:** `rigidbody.drag = Lerp(0, waterDrag, waterLevel)` (кап 10). Вода в трюме тормозит лодку.

### Потопление
1. `waterLevel ≥ 1` → фиксируется `sinkRotation` (текущая ориентация), локальные предметы кешируются (`BoatLocalItems.CacheItemsOnSinking`), `sunk = true`.
2. Плавучесть падает: `boat._forceMultiplier -= deltaTime * baseBuoyancy * 0.33` до 0; коллайдер (`CapsuleCollider`) отключается.
3. Пока тонет — `rigidbody.drag += deltaTime * 6.5` ( вязко уходит под воду).
4. **Плавучесть от воды в трюме** (до потопления): `_forceMultiplier = Lerp(baseBuoyancy, baseBuoyancy*0.66, (waterLevel-0.5)*2)` — при заполнении наполовину плавучесть падает до 66%.

### Восстановление после затопления
При загрузке сейва `LoadDamage(...)`: если `waterLevel ≥ 1` — лодка остаётся потопленной (`sunk = true`, плавучесть 0, предметы «загружены» как закешированные). Полное восстановление делает `Recovery.RecoverBoat` (обнуляет `waterLevel`, см. заметку 12).

## Практические выводы для мододела

1. **Плавучесть** — блоб-модель по `BoxCollider`; сила через `magnitude`, центр масс через `CenterOfMassOffset`. Меняя `boat._forceMultiplier` (BoatProbes), можно мгновенно топить/всплывать.
2. **Живучесть** определяется тройкой `hullDamage`/`waterLevel`/`oakum`. Конопатка гниёт на 12%/день; урон от ударов капится на 0.15 за hit.
3. **Вес игрока = 160**, паруса добавляют массу по `realSailPower` — влияет на осадку и остойчивость через `BoatMass`.
4. **Дифферент/крен от груза** управляется `leverageMult` в `BoatMass`.
5. **Невидимые лодки** не считаются физически (оптимизация) и прищёлкиваются к воде при возврате в кадр — учитывайте при телепортации.
6. Состояние корпуса (`waterLevel`, `hullDamage`, `oakum`, `sinkRotation`) сохраняется в сейв через `SaveableObject.extraValue*` (см. заметку 11).
