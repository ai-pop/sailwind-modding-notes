# 18. Время, погода и штормы

Разбор подсистем времени суток, фаз луны, погоды и штормов. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Время суток (`Sun`)

Синглтон `Sun.sun`. Время измеряется в **часах (0..24)**.

| Поле | Содержание |
|------|-----------|
| `globalTime` | Глобальное время в часах (сохраняется в сейв). |
| `localTime` | **Локальное солнечное время** = `globalTime + globeX / 15` (зависит от долготы!). |
| `timescale` | Скорость времени (× `initialTimescale`; **× 9** во время сна — `Sleep.timeskipSleep`). |
| `day` (`GameState.day`) | Номер игрового дня. |

### Смена дня и событие `OnNewDay`
```
globalTime += deltaTime * timescale
if globalTime > 24: globalTime -= 24; GameState.day++; DayLogs.NewDaySheets(); Sun.OnNewDay();
```
**`Sun.OnNewDay` (static event) — центральный «тикер» игры.** На него подписаны:
- `CurrencyMarket.MarketCycle` — изменение курсов валют (заметка 13);
- `IslandEconomy.RandomizeDemand` — спрос островов (заметка 13);
- `BoatDamage.DailyDamage` — износ корпуса и конопатки (заметка 14);
- другие дневные циклы.

> **Для моддера:** подписка на `Sun.OnNewDay += MyHandler` в `OnEnable` (и отписка в `OnDisable`) — штатный способ делать что-то раз в игровой день.

### Локальное время и география
- **Долгота → часовой пояс:** `localTime = globalTime + globeX / 15` (15° долготы = 1 час, как в реальности). Восход на востоке раньше.
- **Широта → длина дня:** `dawnBorder = -0.147 * globeZ + 10.853`, кламп `[6.16, 6.8]` (при `dawnBorderFromLatitude`). День длиннее/короче в зависимости от широты.
- Поворот солнца: `(localTime + 12) * -15°` (15°/час), наклон по широте `z`.
- `nightLerp` / `dawnLerp` (через `sunriseStart`, `dawnBorder`, `dayBorder`) управляют переходами освещения; читаются через `GetNightLerp()` / `GetDawnLerp()`.

### Пауза времени
`IsPaused()` / `SunPaused()` = true, когда: `!GameState.playing`, игрок на верфи (`currentShipyard`), открыт UI торговли (`EconomyUI.uiActive`, кроме debug). Те же условия, что у потребностей и экономики.

`GetRealtimeDayLength() = 24 / timescale` — реальные секунды на игровой день.

### Солнечный компас
При `GameState.holdingSunCompass` меняется `shadowDistance`/`shadowBias` (тени для навигации по солнцу).

## Луна (`Moon`)

Синглтон `Moon.instance`. `currentPhase` (0..1) — фаза луны (сохраняется в сейв как `moonPhase`).

- **Лунный цикл = 28 игровых дней:** `currentPhase += deltaTime * timescale / 24 / 28`.
- Луна вращается на `360 * phase`, повёрнута к солнцу.
- **Яркость лунного света** зависит от фазы: `|0.5 - phase| * 2` (максимум в полнолуние) × высота солнца. Два источника: `moonlightWater`, `moonlightWorld` (тени включаются при intensity > 0.05).
- Цвет луны: `moonriseColor` → `moonDayColor` / `horizonFadeColor` по высоте.

## Погода (`Weather`)

Синглтон `Weather.instance`. Погода = палитра цветов океана/неба + частицы (облака, дождь).

| Поле | Содержание |
|------|-----------|
| `currentRegion` (`Region`) | Текущий регион (определяет набор погод). |
| `targetWeatherSet` / `previousWeatherSet` | Целевой и предыдущий погодные наборы (плавный переход). |
| `currentTargetLerp` / `lerp` | Прогресс перехода (`lerpRate`). |

- `ChangeWeather(newSet)`: снимает снапшот текущего, ставит цель, `lerp` 0→1.
- Каждый кадр: смешивает `previous → target` палитры, затем с `nightPalette` по `Sun.GetNightLerp()` (ночное затемнение). Применяет через `OceanColorBlender`.
- **`GameState.rainIntensity = finalParticles.rainDensity`** — глобальная интенсивность дождя. Используется, например, в `BoatDamage` (набор воды в трюм от дождя, заметка 14).
- Частицы: `cloudDensity`, `rainDensity` → эмиссия (`rain ×75`, `outerRain ×125`, `rainSplash ×250`, `upperClouds ×2`).

### `WeatherSet` (погодный набор)
Содержит `dayPalette`, `dawnPalette` (`OceanColorPalette`) и `particles` (`WeatherParticlesSettings`). `GetPalette` смешивает день↔рассвет по `GetDawnLerp()`. `BlendSets` смешивает два набора.

## Регионы (`Region`)

Мир разбит на **триггерные объёмы-регионы**. `Region` (`OnTriggerEnter` по тегу `Player` → `RegionBlender.SwitchRegion`):

| Поле | Содержание |
|------|-----------|
| `portRegion` (`PortRegion`) | Идентификатор региона. |
| `clearWeather` / `cloudyWeather` / `rainWeather` / `stormWeather` | 4 погодных набора региона. |
| `windChaos` / `windDirChaos` | Variability ветра (читается `Wind`, заметка 17). |
| `stormRange` (1000) | Дальность влияния штормов. |
| `stormCount` (1) | Число штормов в регионе. |

## Штормы (`WeatherStorms`, `WanderingStorm`)

Погода **управляется близостью к блуждающим штормам** (`WanderingStorm[]`), а не случайностью.

`WeatherStorms.instance` каждые 0.05 c ищет ближайший активный шторм и применяет погоду по **нормированной дистанции** (`GetNormalizedDistance = (distance - stormRadius) / stormRange`, кламп 0..1):

| Норм. дистанция | Погода |
|-----------------|--------|
| `>= 1` | `clearWeather` (ясно) |
| `> cloudyBorder (0.4)` | смешение `clear → cloudy` |
| `> rainBorder (0.1)` | смешение `cloudy → rain` |
| `<= rainBorder` | смешение `rain → storm` (в центре — шторм) |

- `WeatherStorms.currentStormDistance` (static) — дистанция до ближайшего шторма; читается `Wind` для усиления ветра (заметка 17).
- `WeatherStorms.totemAttraction` (static) — «притяжение» штормов; управляется `WindTotemOrb` (тотем ветра может притягивать/отталкивать шторм).
- `WanderingStorm` — сам шторм: движется по миру, имеет радиус, молнии (`WanderingStormLightning`).

## Практические выводы для мододела

1. **`Sun.OnNewDay`** — главный дневной хук. Подписывайтесь для «ежедневных» эффектов; учитывайте, что он не firing при паузе времени (сон ускоряет время ×9, но день всё равно наступает).
2. **Время** = часы 0..24; локальное время зависит от долготы (`globeX/15`), длина дня — от широты. `Sun.sun.globalTime` можно менять для перемотки.
3. **`GameState.rainIntensity`** — удобный глобальный индикатор дождя (0..~1) для ваших эффектов.
4. **Погода региональна** и определяется **близостью к шторму**: хотите локальную бурю — работайте с `WanderingStorm`/`WeatherStorms`, хотите сменить погоду региона — `Weather.instance.ChangeWeather(set)`.
5. **Фаза луны** — 28-дневный цикл, `Moon.instance.currentPhase` (0..1, 0.5 = полнолуние).
6. **Ветер региона** (`windChaos`/`windDirChaos`) и ветер от шторма связаны через `Region` и `WeatherStorms.currentStormDistance`.
7. Время, фаза луны и день сохраняются в сейв (`time`, `moonPhase`, `day` — заметка 11).
