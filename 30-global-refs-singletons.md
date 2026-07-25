# 30. Глобальные ссылки и синглтоны (быстрый справочник)

Сводный справочник точек доступа к ключевым объектам игры. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Держите под рукой при написании модов.

## `Refs` (статические ссылки на игрока и мир)

```csharp
public class Refs {
    public static OVRPlayerController   ovrController;          // контроллер игрока (движение)
    public static Transform             ovrCameraRig;           // риг камеры
    public static CharacterController   charController;         // коллайдер-контроллер персонажа
    public static PlayerControllerMirror observerMirror;        // «зеркало» наблюдателя (эталон позиции)
    public static Transform             controllerCameraRotMirror;
    public static Transform             shiftingWorld;          // мир плавающего начала координат
    public static MouthCol              playerMouthCol;         // «рот» игрока (питьё/курение)
    public static GameObject            mouseCrosshair;         // прицел мыши
    public static Material              mouseCrosshairMaterial;
    public static Material              emptyMaterial;
    public static Transform[]           islands;                // все острова

    public const int goodCount = 65;     // максимум товаров
    public const int portCount = 34;     // максимум портов

    public static void SetPlayerControl(bool state);  // вкл/выкл ovrController + charController
}
```

### Практическое применение `Refs`
- **Позиция игрока/наблюдателя:** `Refs.observerMirror.transform.position` — часто используется как «позиция игрока» (восстановление, ветер, рыба, пассаты).
- **Управление:** `Refs.SetPlayerControl(false/true)` — штатно отключить/включить управление (используется сном, восстановлением, UI). **Может бросить NullReferenceException до инициализации** (см. заметку 10).
- **Бег/движение:** `Refs.ovrController.IsRunning()`, `Refs.ovrController.outLastMovement` (см. заметку 12).
- **Мир:** `Refs.shiftingWorld` = корень плавающего начала координат (родитель предметов, `FloatingOriginManager`, заметка 11).

## `RefsDirectory` (синглтон `RefsDirectory.instance` — реестр ассетов)

```csharp
public class RefsDirectory : MonoBehaviour {
    public Material       emptyMat;
    public LODGroup       LODtemplateItems;          // шаблон LOD для предметов (заметка 16)
    public OceanRenderer  oceanRenderer;             // Crest-рендерер океана
    public GameObject     depthFixerPrefab;
    public GameObject     clothRopePrefab;           // тканевые тросы
    public GameObject     clothRopeSheetPrefab;
    public GameObject     clothRopeJibSheetPrefab;
    public GameObject     sailSnapSoundPrefab;       // звук хлопка паруса
    public ParticleSystem seaWaterSpillParticles;    // брызги морской воды
    public Color          damageColor;               // цвет урона корпуса
    public Color          caulkColor;                // цвет конопатки
    public static RefsDirectory instance;
}
```
Используется для инстанцирования общих префабов (тросы, частицы, звуки) и получения шаблонов (LOD предметов).

## Карта синглтонов игры

Большинство менеджеров — синглтоны через `public static X instance`, устанавливаемый в `Awake`/`Start`. Основные:

| Синглтон | Доступ | Назначение |
|----------|--------|-----------|
| `SaveLoadManager.instance` | заметка 11 | Сохранения, `modData`. |
| `CurrencyMarket.instance` | заметка 13 | Курсы валют. |
| `DebugMarketTracker.instance` | заметка 13 | Баланс-хаб экономики/миссий. |
| `PrefabsDirectory.instance` | заметки 11/16 | Реестр префабов предметов/парусов. |
| `FloatingOriginManager.instance` | заметка 11 | Плавающее начало координат. |
| `PlayerNeeds.instance` | заметка 12 | Потребности (`godMode`). |
| `PlayerTobacco.instance` | заметка 12 | Табак. |
| `Sun.sun` | заметка 18 | Время (`globalTime`, `OnNewDay`). |
| `Moon.instance` | заметка 18 | Фаза луны. |
| `Weather.instance` | заметка 18 | Погода (`currentRegion`). |
| `WeatherStorms.instance` | заметка 18 | Штормы (`currentStormDistance`). |
| `Sleep.instance` | заметка 25 | Сон. |
| `Quests.instance` | заметка 27 | Состояния квестов. |
| `OceanFishes.instance` | заметка 23 | Выбор рыбы. |
| `Wind.instance` | заметка 17 | Ветер (`ForceNewWind`). |
| `RefsDirectory.instance` | выше | Реестр ассетов. |
| `ItemSpawners.instance` | заметка 11 | Спавнеры предметов (кулдауны). |
| `MissionLog.instance` | заметка 15 | Журнал миссий. |
| `DayLogs.instance` | заметки 12/13 | Журнал доходов по дням. |

### Статические классы (без instance)
`GameState`, `PlayerGold`, `PlayerReputation`, `PlayerMissions`, `Settings`, `Refs`, `GameInput` (отсутствует, заметка 24), `Recovery`, `MouseLook`, `Wind` (частично), `WeatherStorms` (частично).

## Практические выводы для мододела

1. **Инициализация:** синглтоны ставятся в `Awake`/`Start` — не обращайтесь к `X.instance` слишком рано (в `Awake` другого компонента); лучше в `Start` или позже. `Refs.SetPlayerControl` может NPE до инициализации.
2. **«Позиция игрока»** чаще всего = `Refs.observerMirror.transform.position`, не `Camera.main`.
3. **Константы:** товаров — **65**, портов — **34** (`Refs.goodCount`/`portCount`).
4. **Плавающее начало координат:** все мировые позиции считайте относительно `FloatingOriginManager.instance.outCurrentOffset` (заметка 11), корень мира — `Refs.shiftingWorld`.
5. **Отключение управления:** `Refs.SetPlayerControl(false)` + `MouseLook.ToggleMouseLookAndCursor(...)` для своих UI/катсцен.
6. **Общие ассеты** (тросы, частицы, LOD-шаблон) берите из `RefsDirectory.instance`, а не ищите по сцене.
