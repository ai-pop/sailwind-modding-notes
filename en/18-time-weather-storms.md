# 18. Time, Weather, and Storms

Analysis of day-night cycle, moon phases, weather, and storms subsystem. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## Day-Night Cycle (`Sun`)

Singleton `Sun.sun`. Time is measured in **hours (0..24)**.

| Field | Content |
|------|-----------|
| `globalTime` | Global time in hours (saved to save). |
| `localTime` | **Local solar time** = `globalTime + globeX / 15` (depends on longitude!). |
| `timescale` | Time speed (× `initialTimescale`; **× 9** during sleep — `Sleep.timeskipSleep`). |
| `day` (`GameState.day`) | In-game day number. |

### Day Change and `OnNewDay` Event
```
globalTime += deltaTime * timescale
if globalTime > 24: globalTime -= 24; GameState.day++; DayLogs.NewDaySheets(); Sun.OnNewDay();
```
**`Sun.OnNewDay` (static event) — the game's central "ticker".** Subscribers include:
- `CurrencyMarket.MarketCycle` — currency rate changes (note 13);
- `IslandEconomy.RandomizeDemand` — island demand (note 13);
- `BoatDamage.DailyDamage` — hull wear and caulking decay (note 14);
- other daily cycles.

> **For modders:** subscribing to `Sun.OnNewDay += MyHandler` in `OnEnable` (and unsubscribing in `OnDisable`) is the standard way to do something once per in-game day.

### Local Time and Geography
- **Longitude → timezone:** `localTime = globalTime + globeX / 15` (15° longitude = 1 hour, as in reality). Sunrise earlier in the east.
- **Latitude → day length:** `dawnBorder = -0.147 * globeZ + 10.853`, clamped `[6.16, 6.8]` (with `dawnBorderFromLatitude`). Day is longer/shorter depending on latitude.
- Sun rotation: `(localTime + 12) * -15°` (15°/hour), tilt by latitude `z`.
- `nightLerp` / `dawnLerp` (via `sunriseStart`, `dawnBorder`, `dayBorder`) govern lighting transitions; read via `GetNightLerp()` / `GetDawnLerp()`.

### Time Pause
`IsPaused()` / `SunPaused()` = true when: `!GameState.playing`, player at shipyard (`currentShipyard`), trade UI open (`EconomyUI.uiActive`, except debug). Same conditions as needs and economy.

`GetRealtimeDayLength() = 24 / timescale` — real seconds per in-game day.

### Sun Compass
When `GameState.holdingSunCompass`, `shadowDistance`/`shadowBias` changes (shadows for sun navigation).

## Moon (`Moon`)

Singleton `Moon.instance`. `currentPhase` (0..1) — moon phase (saved to save as `moonPhase`).

- **Lunar cycle = 28 in-game days:** `currentPhase += deltaTime * timescale / 24 / 28`.
- Moon rotates by `360 * phase`, oriented toward the sun.
- **Moonlight brightness** depends on phase: `|0.5 - phase| * 2` (maximum at full moon) × sun height. Two sources: `moonlightWater`, `moonlightWorld` (shadows enabled at intensity > 0.05).
- Moon color: `moonriseColor` → `moonDayColor` / `horizonFadeColor` by height.

## Weather (`Weather`)

Singleton `Weather.instance`. Weather = ocean/sky color palettes + particles (clouds, rain).

| Field | Content |
|------|-----------|
| `currentRegion` (`Region`) | Current region (determines weather set). |
| `targetWeatherSet` / `previousWeatherSet` | Target and previous weather sets (smooth transition). |
| `currentTargetLerp` / `lerp` | Transition progress (`lerpRate`). |

- `ChangeWeather(newSet)`: snapshots current, sets target, `lerp` 0→1.
- Every frame: blends `previous → target` palettes, then with `nightPalette` by `Sun.GetNightLerp()` (night dimming). Applies via `OceanColorBlender`.
- **`GameState.rainIntensity = finalParticles.rainDensity`** — global rain intensity. Used, e.g., in `BoatDamage` (bilge water intake from rain, note 14).
- Particles: `cloudDensity`, `rainDensity` → emission (`rain ×75`, `outerRain ×125`, `rainSplash ×250`, `upperClouds ×2`).

### `WeatherSet` (Weather Set)
Contains `dayPalette`, `dawnPalette` (`OceanColorPalette`) and `particles` (`WeatherParticlesSettings`). `GetPalette` blends day↔dawn by `GetDawnLerp()`. `BlendSets` blends two sets.

## Regions (`Region`)

The world is divided into **trigger-volume regions**. `Region` (`OnTriggerEnter` by tag `Player` → `RegionBlender.SwitchRegion`):

| Field | Content |
|------|-----------|
| `portRegion` (`PortRegion`) | Region identifier. |
| `clearWeather` / `cloudyWeather` / `rainWeather` / `stormWeather` | 4 weather sets for the region. |
| `windChaos` / `windDirChaos` | Wind variability (read by `Wind`, note 17). |
| `stormRange` (1000) | Storm influence range. |
| `stormCount` (1) | Number of storms in region. |

## Storms (`WeatherStorms`, `WanderingStorm`)

Weather is **driven by proximity to wandering storms** (`WanderingStorm[]`), not randomness.

`WeatherStorms.instance` every 0.05 s finds nearest active storm and applies weather by **normalized distance** (`GetNormalizedDistance = (distance - stormRadius) / stormRange`, clamped 0..1):

| Normalized distance | Weather |
|-----------------|--------|
| `>= 1` | `clearWeather` (clear) |
| `> cloudyBorder (0.4)` | blend `clear → cloudy` |
| `> rainBorder (0.1)` | blend `cloudy → rain` |
| `<= rainBorder` | blend `rain → storm` (storm at center) |

- `WeatherStorms.currentStormDistance` (static) — distance to nearest storm; read by `Wind` for amplification (note 17).
- `WeatherStorms.totemAttraction` (static) — storm "attraction"; controlled by `WindTotemOrb` (wind totem can attract/repel storms).
- `WanderingStorm` — the storm itself: moves through the world, has radius, lightning (`WanderingStormLightning`).

## Practical Takeaways for Modding

1. **`Sun.OnNewDay`** — main daily hook. Subscribe for "daily" effects; note it doesn't fire during time pause (sleep speeds time ×9, but day still arrives).
2. **Time** = hours 0..24; local time depends on longitude (`globeX/15`), day length on latitude. `Sun.sun.globalTime` can be changed for fast-forward.
3. **`GameState.rainIntensity`** — convenient global rain indicator (0..~1) for your effects.
4. **Weather is regional** and determined by **proximity to storm**: want a local storm — work with `WanderingStorm`/`WeatherStorms`, want to change region weather — `Weather.instance.ChangeWeather(set)`.
5. **Moon phase** — 28-day cycle, `Moon.instance.currentPhase` (0..1, 0.5 = full moon).
6. **Regional wind** (`windChaos`/`windDirChaos`) and storm wind are linked via `Region` and `WeatherStorms.currentStormDistance`.
7. Time, moon phase, and day are saved to save (`time`, `moonPhase`, `day` — note 11).
