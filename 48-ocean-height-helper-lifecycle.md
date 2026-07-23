# 48. Океан и вода: OceanHeight, SampleHeightHelper, Ocean lifecycle, подводный эффект

Доскональный разбор API высоты воды, lifecycle Ocean, подводный эффект. Декомпиляция `Assembly-CSharp.dll` (Sailwind v0.38). Связано с заметками 31, 43, 46.

## B6. OceanHeight — тела дословно

```csharp
public class OceanHeight : MonoBehaviour
{
    public static float GetHeight(SampleHeightHelper helper, Vector3 worldPos)
    {
        helper.Init(worldPos, 0.2f, false, null);
        float result = 0f;
        if (helper.Sample(ref result)) { return result; }
        return 0f;   // Crest не готов → 0f (не 0.4!)
    }

    public static float GetHeight(SampleHeightHelper helper, Vector3 worldPos, float accuracy)
    {
        helper.Init(worldPos, accuracy, false, null);
        float result = 0f;
        if (helper.Sample(ref result)) { return result; }
        return 0f;
    }

    public static bool IsUnderwater(SampleHeightHelper helper, Vector3 pos)
    {
        return GetHeight(helper, pos) > pos.y;
    }
}
```

**Что внутри:**
- `GetHeight` вызывает `helper.Init(worldPos, accuracy, false, null)`, затем `helper.Sample(ref result)`
- **Init + Sample в одном вызове** — каждый сэмпл требует Init
- Default accuracy = **0.2f** (параметр Crest)
- Если `helper.Sample()` возвращает `false` (Crest не готов) → **возвращается 0f**, НЕ мусорное значение
- `IsUnderwater`: просто `waterHeight > pos.y` — объект ниже воды

**Вывод для моддера:**
- `OceanHeight.GetHeight` — безопасная обёртка; fallback = 0f при неготовности Crest
- Но 0f — это **не правильная высота**; при `canCheckBuoyancyNow[0] != 1` Crest может вернуть false → результат 0f → предмет «под водой» (pos.y > 0 → IsUnderwater = false, pos.y < 0 → true). Проверять `canCheckBuoyancyNow[0]==1` **перед вызовом**, как делает `Boyant`.

## B7. SampleHeightHelper — полная карта

**SampleHeightHelper — класс из Crest (DLL), НЕ декомпилирован в Assembly-CSharp.** В выгрузке отсутствует полное тело класса. Доступны только сигнатуры из использования:

| Метод | Сигнатура (из использования) | Что делает |
|-------|-----|------|
| `Init` | `Init(Vector3 worldPos, float accuracy, bool disabled, Object component)` | Инициализация сэмпла: позиция, точность, отключить displacement?, optional component |
| `Sample` | `bool Sample(ref float result)` | Выполнить сэмпл, вернуть высоту в `result`; false = Crest не готов |

**Конструктор:** `new SampleHeightHelper()` — создаётся как обычный struct/class. В коде игры используется **как поле** на объектах:

```csharp
// AnimSplash
private SampleHeightHelper helper = new SampleHeightHelper();

// BoatSplashCrest  
private SampleHeightHelper helper = new SampleHeightHelper();

// BoatDamage
private SampleHeightHelper helper = new SampleHeightHelper();

// Rain
private SampleHeightHelper helper = new SampleHeightHelper();

// WaveSound
private SampleHeightHelper helper = new SampleHeightHelper();

// OceanHeight.GetHeight — helper приходит как параметр (caller создаёт)
```

**Можно ли создать один инстанс на сессию и звать из чужого FixedUpdate?**

- **Да**, как делает каждый класс в игре: один `helper` на компонент, используется многократно
- `Init()` вызывается **перед каждым сэмплом** — обязательна (переставляет позицию/accuracy)
- Helper — **не потокобезопасный** (Crest не thread-safe; все сэмплы в main thread)
- Не требует «готовности» при создании — просто структурный helper; реальная проверка — в `helper.Sample()` (возвращает false если неготов)

**Вывод для моддера:** создать `private SampleHeightHelper helper = new SampleHeightHelper();` на MonoBehaviour мода, звать `helper.Init(pos, 0.2f, false, null)` + `helper.Sample(ref result)` каждый кадр. Проверять `canCheckBuoyancyNow[0]==1` перед сэмплом (или проверять `result == 0f` как fallback).

## B8. Жизненный цикл объекта Ocean

### Кто/когда создаёт GO с Ocean

`Ocean` — **MonoBehaviour на GameObject в сцене** (не Instantiate runtime). GO «Ocean» — часть scene hierarchy.

**Awake:**
```csharp
Singleton = this;     // статический singleton
mat[0] = material; mat[1] = material1; mat[2] = material2;
```

**Start:**
```csharp
canCheckBuoyancyNow = new byte[1];    // создаётся при Start, НЕ при Awake!
Initialize();                          // полный init: FFT, meshes, etc.
```

**Признаки «Ocean готов»:**
1. `canCheckBuoyancyNow[0] == 1` — основной гейт. Выставляется в `calcPhase3()` (после завершения FFT + mesh update). Сбрасывается в `0` в `updNoThreads()` перед `calcComplex`.
2. `Ocean.Singleton != null` — объект существует
3. `canCheckBuoyancyNow != null` — массив создан (только после Start)

**Flow готовности по кадрам:**
```
Start → canCheckBuoyancyNow = new byte[1] → Initialize()
  → calcComplex → FFT → calcPhase3 → canCheckBuoyancyNow[0] = 1  // ГОТОВ после первого Update
```

В `updNoThreads()` каждый кадр:
```
canCheckBuoyancyNow[0] = 0    // перед calcComplex (fr2)
calcComplex + FFT              // fr2 + fr2B
FFT(t_x)                       // fr3  
calcPhase3                      // fr4 → canCheckBuoyancyNow[0] = 1  // ГОТОВ снова
```

**Если `everyXframe = 5`:** гейт = 0 на кадрах fr2 (0), fr2B (1), fr3 (2), fr4 (3) — то есть 0 на 1-3 кадрах из 5, 1 на кадрах 0 и 4. На кадре fr4 (`calcPhase3`) гейт = 1.

**Вывод:** Ocean готов **после первого Update**. До этого — `canCheckBuoyancyNow` может быть null (не Start). В runtime: гейт переключается 0/1 по кадрам; 1 ≈ 2 из 5 кадров при default everyXframe.

## B9. Высота воды для камеры / подводный эффект

### Класс с `cameraWaterHeight`

**Реальное имя:** `PlayerSwimming` — имеет поле `public static float cameraWaterHeight`.

Но в коде `PlayerSwimming.LateUpdate` и `SwimEffects.Update` используется **`UnderwaterEffect.cameraWaterHeight`** — это **другая** статика! Класс `UnderwaterEffect` **НЕ декомпилирован** (отсутствует в Assembly-CSharp выгрузке). Вероятно, это класс из отдельного DLL (плагин Oculus/AmplifyOcclusion/другой).

**Что происходит с `PlayerSwimming.cameraWaterHeight`:**
- Объявлено как `public static float cameraWaterHeight` (строка 37)
- **НЕ читается** в LateUpdate — вместо этого читается `UnderwaterEffect.cameraWaterHeight` (строка 102)
- `PlayerSwimming.cameraWaterHeight` — вероятно, **мёртвое поле** (не используется в LateUpdate; декомпилятор мог создать его из-за конфликтов имен)

**`UnderwaterEffect.cameraWaterHeight`** — реальный источник высоты воды для камеры. Класс `UnderwaterEffect` — **внешний DLL**, не в Assembly-CSharp. Определить точно, как он вычисляет высоту, из декомпиляции невозможно, но из контекста:
- `SwimEffects.Update`: `isUnderwater = Camera.main.transform.position.y < UnderwaterEffect.cameraWaterHeight && flag`
- `PlayerSwimming.LateUpdate`: `waterHeight = Mathf.Lerp(waterHeight, UnderwaterEffect.cameraWaterHeight, deltaTime * lerp)`
- Скорее всего, `UnderwaterEffect` — компонент на камере, сэмплирует высоту воды в позиции камеры и записывает в свою static `cameraWaterHeight`

**Все члены PlayerSwimming (дословно):**

| Поле | Модификатор | Тип | Значение/Назначение |
|------|-------------|-----|---------------------|
| `ovrController` | private | `OVRPlayerController` | VR-контроллер |
| `charController` | private | `CharacterController` | Movement |
| `platform` | [SerializeField] private | Transform | - |
| `width` | [SerializeField] private | float | - |
| `lerp` | [SerializeField] private | float | Скорость Lerp к высоте воды |
| `diveSpeedMult` | public | float | Множитель ныряния |
| `swimHeight` | public | float | Порог глубины для плавания |
| `shallowWaterHeight` | public | float | 0.6f — порог мелководья |
| `surfaceSwimHeight` | public | float | Порог поверхностного плавания |
| `jumpKeyUndiveSpeed` | public | float | Скорость всплытия |
| `swimming` | **public static** | bool | В воде |
| `inShallowWater` | **public static** | bool | В мелкой воде |
| `swimmingOnSurface` | **public static** | bool | На поверхности |
| `observerSwimming` | **public static** | bool | Observer в воде |
| `cameraWaterHeight` | **public static** | float | **Мёртвое поле?** (не используется в LateUpdate) |
| `waterHeight` | private | float | Lerp-跟踪 высоты воды |
| `lastWaterHeight` | private | float | Предыдущая высота (для delta) |
| `dive` | **public static** | float | Нырок (угол камеры * diveSpeedMult) |
| `defaultAccel` | private | float | Дефолтная ускорение OVR |
| `unswimTimer` | private | float | - |
| `debugLog` | [SerializeField] private | bool | - |

**SwimEffects — все члены:**

| Поле | Тип | Назначение |
|------|-----|------------|
| `swimAudio` | AudioSource | Звук плавания |
| `underwaterAudio` | AudioSource | Звук под водой |
| `surfaceParticles` | ParticleSystem | Всплески на поверхности |
| `playerWake` | GameObject | Wake-эффект |
| `stars` | Renderer | Звёзды (отключаются под водой) |
| `bubbleParticles` | Transform | Bubbles |
| `underwaterCameraEffects` | GameObject[] | Эффекты камеры под водой |
| `controller` | CharacterController | - |
| `swimAudioVolume` | float | 0.2f |
| `isUnderwater` | **public static bool** | Камера под водой |

**Вывод для моддера:**
- `UnderwaterEffect.cameraWaterHeight` — **единственный реальный источник** высоты воды для камеры. Класс недоступен из декомпиляции. Runtime: можно читать `UnderwaterEffect.cameraWaterHeight` (static) — но тип неизвестен. Практика: использовать `PlayerSwimming.cameraWaterHeight` (static) — это **мёртвое поле**, но runtime может быть заполнено модом, OR использовать `Ocean.Singleton.GetWaterHeightAtLocation2(x-chop, z)` + `canCheckBuoyancyNow[0]==1` гейт.

## B10. OceanUpdaterCrest, OceanWavesUpdater, BoatSplashCrest

### OceanUpdaterCrest

Нет готовой высоты волны/нормаль напрямую. Управляет **весами Gerstner-волн** (ветровые и инерционные). Доступные поля:

| Поле | Тип | Назначение |
|------|-----|------------|
| `timeBetweenUpdates` | float | Интервал обновления |
| `inertiaWindScale` | float | Масштаб инерции → волны |
| `windWindScale` | float | Масштаб ветра → волны |
| `lerpRateInertial` | float | Lerp-rate инерционных волн |
| `lerpRateWind` | float | Lerp-rate ветровых волн |
| `smallWavesMult` | [SerializeField] float | Масштаб мелких волн |
| `windSpeedMult` | [SerializeField] float | Масштаб скорости ветра |
| `windWaves` | [SerializeField] ShapeGerstnerBatched | Ветровые волны |
| `inertiaWaves` | [SerializeField] ShapeGerstnerBatched[] | Инерционные волны (DCT 2 штуки) |
| `wavesRotationAngle` | **static float** | Угол поворота волн |

**Для моддера:** `OceanUpdaterCrest` не даёт высоту/нормаль — это updater весов, не sampler. Высоту брать из `Ocean` или `OceanHeight`.

### OceanWavesUpdater

Тоже updater (устаревший, для non-Crest Ocean). Управляет `ocean.scale`, `ocean.choppy_scale`, `ocean.windx`, `ocean.windy`.

| Поле | Тип | Назначение |
|------|-----|------------|
| `wavesRotationAngle` | **static float** | Угол поворота волн (0→lerp→rotationFixer) |
| `ocean` | Ocean (private) | Ссылка на Ocean |
| `oceanScale` | float | Масштаб ocean.scale |
| `choppyFactor` | float | Фактор choppy |

**Для моддера:** `OceanWavesUpdater.wavesRotationAngle` — угол поворота волн (static). Высоту — из Ocean.

### BoatSplashCrest — как игра спавнит брызги у лодки

```csharp
public class BoatSplashCrest : MonoBehaviour
{
    [SerializeField] Rigidbody boatRigidbody;
    [SerializeField] float deltaThreshold;
    [SerializeField] float deltaMult;
    [SerializeField] float minSize, maxSize;
    [SerializeField] float minSpeed, maxSpeed;
    [SerializeField] float volumeFadeIn, volumeFadeOut;
    
    AudioSource audio;
    ParticleSystem particles;
    float verticalDelta;
    float lastDifference;
    SampleHeightHelper helper = new SampleHeightHelper();
}
```

**Механика брызг:**
- `FixedUpdate → UpdateDeltas + UpdateIntensity`
- **UpdateDeltas:** `verticalDelta = lastDifference - transform.position.y` — разница высот текущего и прошлого кадра
- Если `verticalDelta < deltaThreshold` → delta = 0 (нет брызг)
- **UpdateIntensity:** `num = verticalDelta * deltaMult * boatRigidbody.velocity.magnitude`
  - `startSize = Lerp(minSize, maxSize, num)` — размер частиц
  - `startSpeed = Lerp(minSpeed, maxSpeed, num)` — скорость частиц
  - Audio volume = Lerp к `num` с fade-in/fade-out
  - Если volume ≤ 0.01 → audio OFF; иначе → Play

**Для моддера:** образец для единообразных splash-эффектов. Использовать `verticalDelta` (разница Y-позиций) + `velocity.magnitude` → масштаб частиц + громкость. `SampleHeightHelper` — не используется в BoatSplashCrest для высоты (вычисляет delta через lastDifference, не через Ocean).

## B11. Boyant — тело класса (подтверждение снап высоты)

```csharp
public class Boyant : MonoBehaviour
{
    private Ocean ocean;
    public float buoyancy;
    private bool hasChoppy;
    private Vector3 oldPos;

    void Start() {
        ocean = Ocean.Singleton;
        hasChoppy = ocean.choppy_scale > 0f;
        oldPos = transform.position;
    }

    void Update() {
        if (ocean.canCheckBuoyancyNow[0] == 1) {
            float num = 0f;
            if (hasChoppy) {
                num = ocean.GetChoppyAtLocation2(transform.position.x, transform.position.z);
            }
            float num2 = ocean.GetWaterHeightAtLocation2(
                transform.position.x - num, transform.position.z) + buoyancy;
            transform.position = new Vector3(transform.position.x, num2, transform.position.z);
            oldPos = transform.position;
        } else {
            transform.position = oldPos;  // держать старую позицию при неготовности
        }
    }
}
```

**Подтверждено:** Boyant — **простой снап по высоте** (заметка 43 верна). Дополнение:
- **hasChoppy** проверяется при Start (choppy_scale > 0) — не каждый кадр
- **oldPos** — fallback при неготовности Ocean (`canCheckBuoyancyNow[0] != 1`)
- X/Z не меняются — только Y snap
- **buoyancy** — публичное поле, смещение вверх от воды (по умолч. на префабах)

## B12. Звук брызг (предмет в воде)

**Класс:** `ItemCollisionSoundPlayer` — singleton на объекте в сцене.

```csharp
public class ItemCollisionSoundPlayer : MonoBehaviour
{
    public float speedMult;
    public float minPitchMass = 50f;
    public float maxPitchMass = 1f;
    public float basePitch = 0.5f;
    public float pitchMult = 1f;
    public float volumeReductionFromPitch = 0.5f;
    public AudioSource[] audios;
    public static ItemCollisionSoundPlayer instance;
    private float globalCooldown;
}
```

**Метод:** `PlayWoodColSound(Vector3 position, float mass, float speed)`
- Проигрывает звук столкновения предмета (wood collision)
- Вызывается из `ItemRigidbody.OnCollisionEnter` (не-trigger collision, relativeVelocity.sqrMagnitude > threshold, mass > 1)
- Pitch = `basePitch + InverseLerp(minPitchMass, maxPitchMass, mass) * pitchMult`
- Volume = `speed * speedMult - pitch * volumeReductionFromPitch`
- Audio source позиционируется в точке столкновения
- Cooldown = 0.12s (global)

**Для предмета в воде:** `ItemCollisionSoundPlayer` не имеет splash-звука для предметов. Звук столкновения — wood collision (предмет → предмет/лодка). Splash-звуки для предметов в воде **не реализованы** в ваниле.

**Splash-звуки лодки:**
- `AnimSplash` (на лодке) — AudioSource + ParticleSystem, управляется delta высоты + скорость
- `BoatSplashCrest` (на лодке) — AudioSource + ParticleSystem, управляется verticalDelta + скорость лодки

**UISoundPlayer (для UI-звук):**
```csharp
UISoundPlayer.instance.PlayUISound(UISounds.itemPickup, 0.45f, pitch);
UISoundPlayer.instance.PlaySmallItemDropSound();
```
- `UISounds.itemPickup`, `UISounds.itemInventoryIn`, `UISounds.winchClick` — enum
- Pitch/volume настраивается

**Вывод для моддера:** для splash-звука предмета в воде — нужно создавать свой. Ваниль не имеет. Образец: `AnimSplash` / `BoatSplashCrest` для лодки (deltaY + velocity → масштаб/громкость).
