# 19. Структура мира: порты, регионы, карты, граница

Разбор географии и мировой структуры. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Регионы (`PortRegion`)

```csharp
public enum PortRegion { alankh, emerald, medi, none }   // индексы 0..3
```

Три игровых региона + `none`. **Важно:** индекс региона используется как индекс в массивах репутации (`PlayerReputation.reputation[4]`) и валюты (`PlayerGold.currency[4]`). Соответствие регион ↔ валюта:

| `PortRegion` | Индекс | Валюта (`Currency`) |
|--------------|:--:|---------------------|
| `alankh` | 0 | Al'Ankh Lions |
| `emerald` | 1 | Emerald Dragons |
| `medi` | 2 | Aestrin Crowns |
| `none` | 3 | (Gold Lions — базовая/обменная, без региона) |

> Валюта `gold` (индекс 3) — базовая для обмена (`CurrencyMarket` не меняет её курс daily), не привязана к региону. Награда за миссию выплачивается в валюте региона назначения (см. заметку 15).

## Локальные карты (`LocalMap`)

```csharp
public enum LocalMap { none, alankh, emerald, medi, lagoon }   // 5 значений
```

У каждого порта есть `localMap`. Миссии используют **мировую карту**, если у портов разный `localMap` или он `none` (`Mission.UseOceanMap`); при `alwaysUseWorldMap` у порта — всегда мировая (см. заметку 15). `lagoon` — отдельная особая локация.

## Порты (`Port`)

Статический реестр `Port.ports[34]` — **максимум 34 порта**, индекс = `portIndex`. Заполняется в `Start()` (`ports[portIndex] = this`).

| Поле | Содержание |
|------|-----------|
| `portIndex` | Индекс в `Port.ports`. |
| `portName` | Название (через `GetPortName()`). |
| `region` (`PortRegion`) | Регион порта. |
| `localMap` (`LocalMap`) | Локальная карта. |
| `hubPort` | «Хаб»-порт (влияет на генерацию миссий). |
| `alwaysUseWorldMap` | Всегда мировая карта для миссий. |
| `island` (`IslandEconomy`) | Экономика/спрос острова. |
| `localMapLocation` / `oceanMapLocation` | Точки на локальной/мировой карте. |
| `producedGoodPrefabs` | Товары, производимые портом. |
| `destinationPorts` | Порты-назначения для миссий. |
| `dude` (`PortDude`) | NPC-портовый (`RegisterDude`/`GetDude`). |

### Миссии порта
- `GetMissions(page, world)` → делегирует в `IslandMissionOffice.GenerateMissions` (активный генератор, заметка 15).
- В `Port` есть и **легаси-генератор** `GenerateMissions` на основе спроса (`island.GetDemand`), с формулой цены:
  ```
  price = distance*3 × (1+0.3*count) × (1+1.2e-5*value) × (1+distance*0.0005) × (1+0.0006*mass) × 0.6
  ```
  где `distance` = мировые единицы / 100.
- **Дедлайн** (`GetDueDay`): база = `2 * (длина дня в часах)`, множители от `requiredRepLevel`: `>1` → ×3.5, `==1` → ×2.5, иначе ×1.5. Сложные (репутационные) миссии дают больше времени. `dueDay = GameState.day + round(distance/1000 / dayFactor) + 1`.
- `SpawnGoods`: инстанцирует товары миссии в порту (с шагом-смещением, пачками по 6).
- `ReduceDemand`/`IncreaseDemand`: принятая миссия снижает спрос в порту назначения, отменённая — возвращает.

### Отладочный телепорт
`teleportPlayer = true` телепортирует игрока на `port.position + up*501` (дебаг).

## Граница мира (`WorldBorder`)

Игровая зона — **прямоугольник в координатах глобуса** (`FloatingOriginManager.GetGlobeCoords`, делитель 9000):

```
globeX ∈ [-12, 32]
globeZ ∈ [26, 46]
```

Поведение при выходе за границы (только когда `GameState.playing && !recovering`):
1. Запускается обратный отсчёт `outTimer = 120` секунд.
2. Показывается текст: `"out of game area\nrecovery in N seconds..."` (**захардкожено**, EN).
3. По истечении → `Recovery.RecoverPlayer(RecoveryReason.worldBorder)` — телепорт в последний порт со штрафом (см. заметку 12).
4. При возврате в зону таймер сбрасывается в 120.

> **Для моддера:** мировые координаты глобуса = `(scenePos - outCurrentOffset - globeOffset) / 9000`. Игровая зона мала по меркам «океана» — это и объясняет плавающее начало координат.

## Практические выводы для мододела

1. **3 региона** (alankh/emerald/medi), индекс региона = индекс валюты и репутации. `gold` — безрегионовая обменная валюта.
2. **34 порта максимум** (`Port.ports[34]`); `portIndex` — стабильный идентификатор (используется в сейвах).
3. **Игровая зона** — прямоугольник глобуса x[-12,32] z[26,46]; выход = автовосстановление через 120 c.
4. **Дедлайны миссий** масштабируются с `requiredRepLevel` (×1.5/×2.5/×3.5 к базовому времени).
5. **Легаси-формула цены** миссии (в `Port.GetTotalPrice`) учитывает дистанцию, число, стоимость, массу — полезна для понимания баланса, хотя активен арбитражный генератор из `IslandMissionOffice`.
6. Координаты: `globeCoords = (pos - outCurrentOffset - globeOffset) / 9000` (см. заметку 11 про floating origin).
