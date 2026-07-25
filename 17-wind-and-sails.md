# 17. Ветер и паруса: модель движения

Разбор подсистемы ветра и парусной тяги — сердце геймплея Sailwind. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Ветер (`Wind`)

Синглтон `Wind.instance`. Вектор ветра — направление × сила (магнитуда). Статические поля: `currentWind` (фактический, со шквалами), `currentBaseWind` (базовый), `windRotation`.

Стартовый ветер: `normalized(1, 0, -0.25) * 9` (≈9 единиц).

### Структура ветра (3 уровня)
```
currentBaseWind   ← базовый (медленные смены направления/силы)
   └─ currentWindTarget  ← цель с учётом шторма и океана
        └─ currentGustTarget  ← шквалы (быстрые осцилляции ±33%)
             └─ currentWind  ← фактический (лерп к gustTarget)
```

### Смена ветра (`SetNewWindTarget`, по таймеру `changeTimer * [0.5..2]`)
1. **Направление:** случайный единичный вектор смешивается с **пассатом** (`tradeWindInfluence`, по умолч. 0.5) и с текущим базовым ветром (`directionChaos` из региона).
2. **Сила:** `base ± magnitudeChaos`, клампится в `[minimumMagnitude, maximumMagnitude]`.
3. `directionChaos` / `magnitudeChaos` берутся из `Weather.instance.currentRegion.windDirChaos` / `.windChaos` — **ветер меняется по регионам**.

### Шквалы (`SetNewGustTarget`, таймер `gustChangeTimer`)
`currentGustTarget = currentWindTarget * Random(0.67, 1.33)` — быстрые осцилляции силы ±33%. `currentWind` плавно догоняет цель со скоростью `finalLerpSpeed`.

### Усиление ветра
К базовой силе добавляются (кап суммарно **+20**):
- **Шторм:** `+26 * InverseLerp(13000, 500, WeatherStorms.currentStormDistance)` — чем ближе шторм, тем сильнее (до +26, но общий кап 20).
- **Открытый океан:** `+baseMagnitude * InverseLerp(1500, 4000, GameState.distanceToLand) * 0.66` — вдали от берега ветер сильнее.

### Пассаты (`GetCurrentTradeWind`)
Направление по **координатам глобуса** (`FloatingOriginManager.GetGlobeCoords`, делитель 9000):

| Условие (globe x/z) | Вектор пассата |
|---------------------|----------------|
| `z < 30` | нет (нулевой) |
| `z > northTradeWindBorder` | `normalized(1, 0, 0.5)` |
| `x < -2` | `normalized(0.75, 0, 0.75)` |
| `z < southTradeWindBorder` | `normalized(-1, 0, -0.5)` |
| иначе | нет |

Это создаёт устойчивые ветровые пояса на карте мира.

### Управление извне
- `ForceNewWind(v)` / `LoadWind(v)` — мгновенно задать ветер (используется при загрузке сейва и отладке). `LoadWind` клампит магнитуду до `maximumMagnitude`.
- `debugConstantWind` (только в редакторе) замораживает ветер; `Debugger.debugWind` форсирует `(10, 0, 10)`.
- Ветер сохраняется в сейв как `currentBaseWind` (см. заметку 11).

## Парус (`Sail`)

`[RequireComponent(HingeJoint, SailConnections)]`, использует Unity `Cloth` для симуляции полотна.

### Ключевые поля
| Поле | Содержание |
|------|-----------|
| `sailName` / `prefabIndex` | Имя и индекс в `PrefabsDirectory.sails`. |
| `price` | Цена (по умолч. 1000). |
| `category` | `SailCategory` (тип паруса). |
| `squareSail` / `junkType` | Флаги прямого/джонкового паруса. |
| `sailArea` | Площадь (вычисляется из меша, `CalculateSurfaceArea`). |
| `sailAmplifier` | Усилитель тяги. |
| `upwindEfficiency` | Эффективность против ветра (0..1). |
| `minAngle` / `maxAngle` | Диапазон углов атаки. |
| `installHeight` / `scaledInstallHeight` | Высота установки на мачте. |
| `mastOrder` | Порядковый номер мачты. |
| `currentUnroll` | Степень распускания паруса (0..1, рифление). |
| `apparentWind` | **Кажущийся ветер** (вектор). |

### `SailCategory` (enum)
`square` (прямой), `lateen` (латинский), `junk` (джонка), `gaff` (гафельный), `other`, `staysail` (стаксель).

**Множители тяги по категории** (`ApplyForce`):
- `junk`: сила × **0.75**
- `gaff`: сила × **0.85**
- остальные: × 1.0

(Категория также влияет на массу паруса в `BoatMass`, см. заметку 14.)

### Расчёт тяги
1. **Кажущийся ветер** = вектор ветра минус скорость лодки (`apparentWind`).
2. **Суммарная сила:** `apparentWind.magnitude × GetWindHeightMult(высота) × GetCurrentShadowMult()`.
   - `GetWindHeightMult` — ветер сильнее выше над водой.
   - `GetCurrentShadowMult` — **ветровая тень**: паруса, закрытые другими парусами (`SailShadowCol`), получают меньше ветра.
3. **Эффективность по углу:** `Lerp(force * upwindEfficiency, force, relativeWindAngle/90) × max(debugYardMult, currentUnroll)`.
4. **Мёртвая зона (no-go):** множитель `InverseLerp(13, 16, relativeWindAngle)` — при угле к ветру **< 13–16°** парус почти не работает (нельзя идти круто к ветру).
5. Если лодка потоплена (`damage.sunk`) — сила = 0.
6. **Разложение на составляющие** относительно корпуса:
   - `forwardShare = Lerp((90-angle)/90, 1, upwindEfficiency)` — доля тяги вперёд.
   - `sidewayShare = angle/90` — доля бокового сноса (дрейф).
   - Если сила направлена назад (`>90°` от курса) — фактор **−0.33** (парус «толкает назад», но слабее).

### Приложение силы (`ApplyForce`)
```
realSailPower = sailArea * Debugger.instance.debugSailAreaMult
forwardForce  = unamplifiedForward * realSailPower * 50
sideForce     = unamplifiedSideway * realSailPower * 50 * 1.5   // боковая × 1.5
shipRigidbody.AddForceAtPosition(forward, windcenter)
shipRigidbody.AddForceAtPosition(side,    windcenter)
```
- Базовый множитель силы = **50**, боковая составляющая дополнительно **× 1.5** (парус больше кренит/сносит, чем толкает).
- `GetRealSailPower()` = `sailArea * debugSailAreaMult` — та же величина используется для массы паруса.

### Направление силы (`GetSailForceDirection`)
Парус толкает вдоль своей нормали: для обычных парусов — `up`/`down`, для прямых (`squareSail`) — `right`/`left`. Выбирается грань, в которую дует кажущийся ветер.

### Масштабирование паруса (`ChangeScale`)
- Размер меняется в пределах **0.2..3.0** по осям.
- Прямые паруса масштабируются иначе (Z ограничен относительно Y: `Y±[0.2..0.6]`, X=Z).
- При изменении пересчитываются `scaledInstallHeight` и `sailArea`, обновляется коллайдер тени.

### Ветер на корпусе (`BoatWind`)
Отдельный компонент применяет ветер **к самому корпусу** (парусность/дрейф без парусов):
```
apparent = Wind.currentWind - body.velocity
angle    = |SignedAngle(boat.forward, apparent)|   // >90 → 180-angle
force    = apparent * Lerp(frontForce, sideForce, angle/90)
body.AddForceAtPosition(force, TransformPoint(0,0,-2))
```
Лобовой ветер → `frontForce`, боковой → `sideForce`. Точка приложения смещена назад (`forceOffset = (0,0,-2)`).

## Практические выводы для мододела

1. **Ветер** = база (региональный хаос + пассаты по координатам глобуса) + шквалы (±33%) + усиление от шторма/океана (кап +20). Регионы задают `windChaos`/`windDirChaos`.
2. **Пассаты** привязаны к координатам глобуса (делитель 9000) — можно предсказать prevailing wind по позиции.
3. **Мёртвая зона** паруса — 13–16° к ветру; `upwindEfficiency` определяет, насколько хорошо парус работает в бейдевинд.
4. **Категории парусов:** junk ×0.75, gaff ×0.85 тяги (но влияют и на массу). Прямые (square) лучше на попутных курсах.
5. **Боковая сила × 1.5** от продольной — паруса сильно кренят; учитывайте в балансе.
6. **Ветровая тень** (`SailShadowCol`): паруса за другими теряют тягу.
7. **Глобальные тюнинг-множители:** `Debugger.instance.debugSailAreaMult` (площадь/сила), `debugYardMult` (мин. раскрытие). `Wind.ForceNewWind` для скриптов погоды.
8. Ветер и состояние парусов (scale, unroll, color) сохраняются в сейв (`SaveSailData`, `SaveBoatCustomizationData`).
