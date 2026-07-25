# 23. Рыбалка и сохранение еды

Разбор подсистем рыбалки и порчи/консервации пищи. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Выбор рыбы (`OceanFishes`, `LocalFishesRegion`)

Тип пойманной рыбы определяется **географией** (широтой), с локальными переопределениями.

`OceanFishes.instance.GetFish(pos)`:
1. **Локальные регионы** (`LocalFishesRegion[]`): для каждого — влияние = `InverseLerp(outerRadius, innerRadius, distance) * overrideInfluence`. Если `Random(0,1) <= влияние` → случайная рыба из `localFishPrefabs` региона.
   - По умолчанию: `outerRadius = 5000`, `innerRadius = 2500`, `overrideInfluence = 0.75`.
2. **Иначе по широте:** `latitude = globeCoords.z + weightedDeviation` (случайное отклонение, усреднённое по 6 итерациям, амплитуда `deviationDistance`). Выбирается рыба с ближайшей `peakLatitude[i]`.

Каждый префаб рыбы имеет «пиковую широту» обитания (`peakLatitude[]`). Отладка: клавиша `j` логирует, какая рыба поймается (`DebugFishCatch`).

## Мини-игра рыбалки (`FishingRodFish`)

### Поклёвка
При заброшенном поплавке (`bobber` в воде, леска выпущена, `linearLimit > 1`), удочка в руках, рыбы ещё нет:
- `fishTimer` тикает раз в секунду.
- **Шанс поклёвки** = `InverseLerp(3, 20, distance) * 2.5 + 0.5` % в секунду, где `distance` — дистанция до удочки. Чем **дальше заброс**, тем выше шанс (от ~0.5% на 3 м до ~3% на 20 м).
- При успехе → `CatchFish()`: выбирается рыба (`OceanFishes.GetFish`), меш поплавка заменяется на модель рыбы, `fishEnergy = 1`.

### Борьба (натяжение лески)
В `FixedUpdate`:
- Рыба тянет (`fishPullForce`), пока в воде и есть энергия.
- `pullForce` = текущая сила соединения (`ConfigurableJoint.currentForce`) — натяжение лески.
- `currentTargetTension` растёт, когда `pullForce >= lowerForceThreshold` (игрок тянет против рыбы), с учётом подмотки (`reelBendMult`, длина лески).
- **Обрыв:** если `tension > 0.95` — растёт `snapTimer`, удочка трясётся, играет звук натяжения; **при `snapTimer > 3.1 c` → `ReleaseFish()`** (леска рвётся, рыба потеряна).
- Натяжение спадает, когда не тянешь. `rod.SetRodTension(tension)` визуально гнёт удочку.

### Подсечка и улов (`CollectFish`)
- Инстанцирует пойманную рыбу как `ShipItem` (`sold = true`, `RegisterToSave`).
- **31% шанс потерять крючок:** `if Random(0,100) > 69 → rod.DetachHook()`.
- Если поплавок вне воды — рыба «мертва» (`fishDead`), борьба прекращается.

## Сохранение еды (`FoodState`)

Еда портится; её можно **консервировать** сушкой, копчением, засолкой или варкой.

### Поля (все 0..1, кроме указанных)
| Поле | Содержание |
|------|-----------|
| `spoilDuration` | Базовая длительность до порчи. |
| `dried` | Степень сушки. |
| `smoked` | Степень копчения. |
| `salted` | Степень засолки. |
| `spoiled` | Степень порчи (`> 0.9` = «rotten»). |
| `slicePrefabIndex` / `slicesCount` | Нарезка (кусочки). |
| `inWater` | В воде (размокает). |

### Сушка (`dried`)
```
rate = deltaTime * timescale / (30 * energyPerBite)
rate *= Lerp(1, 10, salted)          // соль ускоряет сушку
if на сушилке (DryingRackCol): rate *= 2
if слой 26 (в инвентаре): rate = 0
if inWater: rate = -100 * deltaTime * timescale   // в воде размокает
dried += rate   (кламп 0..1)
+ копчение: если плита в режиме smoking → dried += currentHeat
```

### Порча (`spoiled`)
```
rate = deltaTime * timescale / spoilDuration
rate *= InverseLerp(1, 0, currentHeat)   // нагрев (варка) останавливает порчу
rate *= (1 - dried)                       // сушёное не портится
rate *= (1 - smoked)                      // копчёное не портится
spoiled += rate
```
**Вывод:** сушка и копчение полностью предотвращают порчу; варка (нагрев) — пока горячее; соль ускоряет сушку.

### Засолка
`GetRequiredSalt() = energyPerBite * 0.33 * (1 - salted)` — сколько соли нужно для засолки. `AddSmoked(amount)` добавляет копчение (кап 1).

### Текстовые префиксы состояния (захардкожены, EN)
Порядок проверки в `UpdateLookText` (первый совпавший):

| Условие | Префикс |
|---------|---------|
| `spoiled > 0.9` | `"rotten "` |
| `amount >= 1.5` | `"burnt "` (пережарено) |
| `salted >= 0.99 && smoked >= 0.99` | `"salted smoked "` |
| `salted >= 0.99` | `"salted "` |
| `smoked >= 0.99` | `"smoked "` |
| `dried >= 0.99` | `"dried "` |
| `amount >= 1` | `"cooked "` |
| `raw && spoiled < 0.2` | `"fresh raw "` |
| `raw` | `"raw "` |
| `spoiled < 0.2` | `"fresh "` |

Итог: `description = префикс + name`. Сушилка детектится триггером `DryingRackCol`.

## Жидкости (`LiquidType`)

```csharp
public enum LiquidType {
  none, water, rum, wine, cocoWine, mead, honeyBeer, riceBeer,
  cider, seaWater, coffee, blackTea, greenTea, whiteTea
}
```
14 типов жидкостей (для чайника `ShipItemKettle`, бутылок, напитков). Чай: black/green/white; алкоголь: rum/wine/cocoWine/mead/honeyBeer/riceBeer/cider; безалкогольное: water/coffee/seaWater.

## Практические выводы для мододела

1. **Рыба по широте:** тип определяется `globeCoords.z` (+ случайность) и `peakLatitude[]` префабов; локальные регионы (`LocalFishesRegion`) переопределяют улов в радиусе.
2. **Клёв** зависит от дальности заброса (3–20 м → 0.5–3%/с).
3. **Мини-игра** = управление натяжением: тяни, но не держи `tension > 0.95` дольше ~3.1 c, иначе обрыв. 31% шанс потерять крючок при подсечке.
4. **Консервация еды:** `dried`/`smoked` полностью блокируют порчу; `salted` ускоряет сушку; нагрев (варка) останавливает порчу, пока горячее.
5. **Префиксы еды** (`"rotten"`, `"smoked"`, `"cooked"`…) захардкожены на английском в `FoodState.UpdateLookText` (см. заметку 08).
6. Состояние еды (`dried/smoked/salted/spoiled`) сохраняется через `SavePrefabData.extraValue0..3` (заметка 11).
