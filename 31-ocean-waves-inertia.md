# 31. Океан и волны: инерция волнового поля

Разбор волновой модели. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Океан — кастомный форк **Crest**; плавучесть судов описана в заметке 14, ветер — в заметке 17.

## Инерция волн (`WavesInertia`)

Волновое поле **не следует за ветром мгновенно** — у него есть инерция (как у реального моря: волны медленно разворачиваются и набирают высоту).

| Поле | Содержание |
|------|-----------|
| `wind` | Ссылка на `Wind`. |
| `directionChangeRate` | Скорость разворота волн к ветру. |
| `inertiaChangeRate` | Скорость изменения инерции. |
| `magnitudeChangeRate` | Скорость изменения высоты волн. |
| `currentInertia` | Текущая инерция (мин. 1, при росте кап 70). |
| `currentMagnitude` | Текущая высота волн (мин. 1). |

### Разворот волн (`Update`)
```
factor = max(0.25, 10 / currentInertia)
transform.rotation = Slerp(transform.rotation, wind.transform.rotation, dt * directionChangeRate * factor)
```
Чем **выше инерция**, тем **медленнее** волны разворачиваются к ветру (factor меньше).

### Изменение инерции (`UpdateCurrentInertia`)
Зависит от угла между направлением ветра и волн (`Quaternion.Angle`):
- **Угол ≤ 45°** (волны почти по ветру) → инерция **растёт** (до 70): волны «успокаиваются», стабилизируются по ветру.
- **Угол > 90°** (волны поперёк ветра) → инерция **падает**: поле быстрее перестраивается.
- Кап: минимум `1`.

### Высота волн (`currentMagnitude`)
- Плавно догоняет силу ветра: если `currentMagnitude < Wind.currentWind.magnitude` → растёт, иначе падает, со скоростью `magnitudeChangeRate`. Минимум `1`.
- Т.е. **высота волн ≈ сила ветра**, но с запаздыванием.

### Персистентность
`LoadInertia(rotation, inertia, magnitude)` восстанавливает из сейва; сохраняются `wavesRotation`, `wavesInertia`, `wavesMagnitude` (заметка 11). Состояние волн переживает загрузку — море не «сбрасывается».

## Запрос высоты волны (`OceanHeight`, статический)

Канонический способ получить высоту воды в точке (через Crest `SampleHeightHelper`, отсутствующий в репо — заметка 24):

```csharp
// точность по умолчанию 0.2
static float GetHeight(SampleHeightHelper helper, Vector3 worldPos);
static float GetHeight(SampleHeightHelper helper, Vector3 worldPos, float accuracy);

static bool IsUnderwater(SampleHeightHelper helper, Vector3 pos)
    => GetHeight(helper, pos) > pos.y;
```

`SampleHeightHelper.Init(worldPos, accuracy, false, null)` → `Sample(ref result)`. Используется повсеместно для проверки «под водой ли объект» и высоты волны.

## Связь с другими системами

| Система | Как использует волны |
|---------|---------------------|
| `Buoyancy` (заметка 14) | Сэмплирует высоту через `Ocean.GetWaterHeightAtLocation2` + `GetChoppyAtLocation` для силы плавучести. |
| `Wind` (заметка 17) | Источник направления/силы; волны догоняют ветер через `WavesInertia`. |
| `Ocean` (Crest) | `Ocean.Singleton`, `GetWaterHeightAtLocation2`, `GetChoppyAtLocation`, рендеринг (`RefsDirectory.oceanRenderer`). |
| `BoatDamage.Overflow` | Захлёстывающие волны добавляют воду в трюм. |
| `Fourier` / `Fourier2` | FFT-генерация волнового спектра (Gerstner, Crest). |

## Практические выводы для мододела

1. **Волны инерционны:** направление разворачивается к ветру со скоростью `~ directionChangeRate * (10/currentInertia)`, высота догоняет силу ветра. Резкая смена ветра не мгновенно меняет волны.
2. **Высота волн ≈ `Wind.currentWind.magnitude`** (с запаздыванием, мин. 1). Хотите штиль/шторм — меняйте ветер, волны подтянутся.
3. **Высота воды в точке:** `OceanHeight.GetHeight(helper, worldPos)` / `IsUnderwater(...)` (нужен `SampleHeightHelper` из Crest).
4. Состояние волн (`wavesRotation/Inertia/Magnitude`) сохраняется в сейв.
5. Для своей плавучести используйте `Ocean.Singleton.GetWaterHeightAtLocation2(x, z)` (учитывая `outCurrentOffset` плавающего начала координат).
