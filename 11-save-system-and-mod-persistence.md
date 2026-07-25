# 11. Система сохранений и хук персистентности модов

Полный разбор подсистемы сериализации игры. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Это самая важная система для любого мода, которому нужно сохранять собственные данные между сессиями.

## Кратко: как моду сохраниться

В игре есть **встроенный официальный хук** для мод-данных. В `GameState` существует статический словарь:

```csharp
// GameState.cs
public static Dictionary<string, string> modData = new Dictionary<string, string>();
```

Этот словарь **автоматически сериализуется в файл сохранения и восстанавливается при загрузке** — без какого-либо кода со стороны мода. Достаточно записать туда свои данные:

```csharp
GameState.modData["MyMod.money"] = "1234";
GameState.modData["MyMod.flags"] = "a,b,c";
```

При следующем сохранении игры значения попадут в сейв, а при загрузке вернутся в тот же словарь. Разработчик оставил под это пустые методы-хуки (см. ниже), но сам механизм переноса `modData` уже полностью работает в релизной версии.

> **Важно:** значения — только `string`. Сложные структуры сериализуйте сами (JSON, base64, `key;key;key`). Ключи конфликтуют между модами глобально — используйте префикс имени мода (`MyMod.*`).

## Архитектура сохранения

```
SaveLoadManager (MonoBehaviour, синглтон .instance)
│
├── currentObjects : SaveableObject[255]   ← «статичные» объекты сцены (лодки, игрок, якоря…)
│       регистрация: SaveableObject.Awake() → AddObject(this) по sceneIndex
│
├── currentPrefabs : List<SaveablePrefab>  ← динамические предметы (ShipItem)
│       регистрация: SaveablePrefab.RegisterToSave() → AddPrefab(this)
│
└── SaveGame(compressed) → DoSaveGame() (корутина)
        собирает SaveContainer → BinaryFormatter → slot{N}.save
```

### Точки входа

| Поле/метод | Тип | Назначение |
|------------|-----|-----------|
| `SaveLoadManager.instance` | `static` | Синглтон менеджера. |
| `SaveLoadManager.readyToSave` | `static bool` | Гейт сохранения: `SaveGame` ничего не делает, пока `false`. |
| `SaveGame(bool compressed)` | `void` | Запуск сохранения (через корутину `DoSaveGame`). |
| `LoadGame(int backupIndex)` | `void` | Загрузка: `0` = текущий сейв, `1..5` = бэкапы. |
| `save` / `saveCompressed` / `load` | `bool` (public) | Флаги-триггеры: выставить в `true` → сработает в `Update()`. |
| `SaveModData()` / `LoadModData()` | `void` | **Пустые хуки** — только `Debug.Log("Saving/Loading mod data...")`. Точки расширения, зарезервированные разработчиком. |

### Условия, при которых сохранение НЕ происходит

`SaveGame` молча отказывает (логирует `"not ready to save"`), если верно любое:

- `busy` (уже идёт сохранение/загрузка);
- игрок в кровати (`GameState.inBed != null`);
- `readyToSave == false`;
- игрок на верфи (`GameState.currentShipyard != null`).

### Автосохранение

В `Update()` менеджера:

- Интервал = `Settings.autosaveInterval` **минут** (`* 60f` секунд). Если `<= 0` — автосейв выключен.
- Автосейв пропускается, когда: `enableAutosave == false`, игрок спит (`GameState.sleeping`), идёт восстановление (`GameState.recovering`), открыт UI ящика (`CrateInventoryUI.instance.showingUI`).
- Автосейв всегда **сжатый** (`compressed: true`) — текстуры чистоты корпуса кодируются в PNG. Ручное сохранение может быть несжатым (raw-текстуры).
- `enableAutosave` принудительно включается вне редактора Unity (`!Application.isEditor`).

## Файлы сохранений

Пути строит `SaveSlots` от `Application.persistentDataPath` (на Windows: `%USERPROFILE%\AppData\LocalLow\<Company>\<Product>\`):

| Файл | Назначение |
|------|-----------|
| `slot{N}.save` | Текущее сохранение слота N. |
| `slot{N}_backup{1..5}.save` | Кольцевые бэкапы: перед каждым сохранением текущий сейв сдвигается в `backup1`, прежний `backup1` → `backup2`, …, `backup5` удаляется. |

- **6 слотов** (индексы `0..5`). `SaveSlots.currentSlot` — активный.
- `SaveSlots.slotsActive[6]` / `activeSlotsCount` — какие слоты заняты (определяется по наличию файла в `Awake`).
- Формат — **`BinaryFormatter`** (legacy .NET binary serialization) поверх `[Serializable]`-класса `SaveContainer`. Это важно: формат бинарный, привязан к именам полей/сборки; версионирование через поле `gameVersion` (текущее = `1`).

> **Для моддера:** читать/писать сейвы напрямую из мода не нужно и опасно (BinaryFormatter). Используйте `GameState.modData`. Внешним инструментам для разбора сейва понадобится та же структура `SaveContainer`.

## SaveContainer — что именно сохраняется

`SaveContainer` (`[Serializable]`) — плоский объект со всеми данными мира. Основные группы полей:

### Игрок и экономика
| Поле | Тип | Содержание |
|------|-----|-----------|
| `playerCurrency` | `int[]` | Кошелёк по валютам (`PlayerGold.currency`). |
| `currentCurrency` | `int` | Активная валюта. |
| `currencyRates` | `float[]` | Текущие курсы валют (`CurrencyMarket.instance.currentPrices`). |
| `playerReputation` | `int[]` | Репутация (`PlayerReputation.GetSaveData()`). |
| `playerKnownPrices` | `PriceReport[]` | Известные игроку цены. |
| `tradeReceipts` | `TradeReceipt[7]` | Квитанции о сделках. |
| `charts` | `List<ChartData>` | Нарисованные морские карты. |
| `currencyDayLogs` | `DaySheet[][]` | История доходов по дням/валютам. |

### Время и мир
| Поле | Тип | Содержание |
|------|-----|-----------|
| `time` | `float` | Глобальное время (`Sun.sun.globalTime`). |
| `moonPhase` | `float` | Фаза луны (`Moon.instance.currentPhase`). |
| `day` | `int` | Номер игрового дня (`GameState.day`). |
| `wind` | `SerializableVector3` | Базовый ветер (`Wind.currentBaseWind`). |
| `wavesRotation` / `wavesInertia` / `wavesMagnitude` | `Quaternion`/`float` | Состояние волн (`WavesInertia`). |
| `lastVisitedPort` | `int` | Индекс последнего порта (`-1` = нет). |

### Потребности и привычки
`food`, `foodDebt`, `water`, `sleep`, `sleepDebt`, `vitamins`, `protein`, `alcohol` (float) — из `PlayerNeeds`; `tobaccoWhite/Green/Black/Brown` (float) — из `PlayerTobacco`.

### Мир (порты, рынки, миссии, NPC)
| Поле | Тип | Содержание |
|------|-----|-----------|
| `marketsSupply` | `float[][]` | Запасы рынков по портам (`IslandMarket.currentSupply`). |
| `marketsKnownPrices` | `PriceReport[][]` | Известные цены по портам. |
| `portDemands` | `int[][]` | Потребности островов (`port.island.GetDemandSaveData()`). |
| `missionWarehouses` | `int[][]` | Склады миссий (`IslandMissionOffice.GetData()`). |
| `quests` | `int[]` | Активные квесты (`Quests.instance.currentQuests`). |
| `itemSpawnerCooldowns` | `float[]` | Кулдауны спавнеров предметов. |
| `traderBoatData` | `TraderBoatData[]` | Состояние торговых лодок. |
| `npcBoatData` | `NPCBoatData[]` | Состояние NPC-лодок. |
| `savedMissions` | `SaveMissionData[]` | Активные миссии (`PlayerMissions.missions`). |
| `loggedMissions` | — | Журнал миссий (`MissionLog`). |

### Объекты и предметы
| Поле | Тип | Содержание |
|------|-----|-----------|
| `savedObjects` | `List<SaveObjectData>` | Все `SaveableObject` сцены. |
| `savedPrefabs` | `List<SavePrefabData>` | Все динамические предметы (`SaveablePrefab`). |
| `modData` | `Dictionary<string,string>` | **Данные модов** (`GameState.modData`). |
| `gameVersion` | `int` | Версия формата (=`1`). |

## SaveableObject — «статичные» объекты сцены

Лодки, игрок, якоря, швартовы и т.п. — объекты с фиксированным `sceneIndex`.

- Массив на **255** элементов в `SaveLoadManager.currentObjects`.
- Регистрация в `Awake()`: `SaveLoadManager.instance.AddObject(this)`. При коллизии индексов — `LogError("scene index conflict")`.
- **`sceneIndex == 0` — это игрок** (при загрузке у него двигают `PlayerControllerMirror.controller`).
- Позиция сохраняется **относительно плавающего начала координат**: `transform.position - FloatingOriginManager.instance.outCurrentOffset`.
- Универсальные «доп. поля» переиспользуются под разные цели:

| Поле | Для лодки (BoatDamage) | Для якоря (Anchor) |
|------|------------------------|--------------------|
| `extraSetting` | — | Якорь поставлен (`IsSet()`) |
| `extraValue` | `waterLevel` | Длина троса (`GetRopeLength()`) |
| `extraValue1` | `hullDamage` | — |
| `extraValue2` | `oakum` (конопатка) | — |
| `extraTexture` | `Texture2D` текстура грязи/чистоты корпуса (128×128) | — |
| `customization` | `SaveBoatCustomizationData` (кастомизация лодки) | — |

- `sinkRotation` — ориентация затонувшей лодки (`BoatDamage.GetSinkRotation()`).
- Текстура чистоты: при сохранении несжатая = `GetRawTextureData()` (ровно `16777216` байт = 128×128×RGBA32... фактически 128×128×4×... см. ниже), сжатая = `EncodeToPNG()`. При загрузке игра различает форматы **по длине массива**: `== 16777216` → `LoadRawTextureData`, иначе → `ImageConversion.LoadImage` (PNG) с ресайзом до 128×128.
- Локальные предметы лодки (`BoatLocalItems`) сохраняются как отдельные `SavePrefabData` с привязкой к родителю.

## SaveablePrefab — динамические предметы (ShipItem)

Все подбираемые/покупаемые предметы (`[RequireComponent(typeof(ShipItem))]`).

- `prefabIndex` — индекс в `PrefabsDirectory.directory[]`. При загрузке предмет **инстанцируется заново** из префаба и наполняется данными.
- `instanceId` — случайный уникальный int (для различения экземпляров; хранится в статическом `existingInstanceIds`).
- Позиция: относительно родительского объекта (`itemParentObject > 0` → `InverseTransformPoint`), иначе относительно плавающего начала координат.
- **`inventorySlot`**: `< 100` → слот инвентаря (`GPButtonInventorySlot.inventorySlots[slot]`); `>= 100` → грузовой носильщик (`CargoCarrier.carriers[slot-100]`).
- Универсальные `extraValue0..4` переиспользуются:

| Компонент | extraValue0 | extraValue1 | extraValue2 | extraValue3 | extraValue4 |
|-----------|-------------|-------------|-------------|-------------|-------------|
| `FoodState` | dried | smoked | salted | spoiled | — |
| `ShipItemSoup` | uncookedEnergy | spoiled | salted | vitamins | protein |
| `ShipItemKettle` | teaAmount | cookedTeaAmount | — | — | — |

  (у супа/чайника `itemHealth`=water, `itemAmount`=energy/teaType)
- `chartData` — данные морской карты для раскладных предметов (`ShipItemFoldable.allowCharting`).
- Связь с миссией: `itemMissionIndex` → `PlayerMissions.missions[idx].RegisterGood(...)`, иначе `RegisterAsMissionless()`.

## PrefabsDirectory — реестр префабов

`PrefabsDirectory.instance` держит массивы:

| Массив | Содержание |
|--------|-----------|
| `directory[]` | Все предметные префабы, индекс = `prefabIndex`. Индекс `0` = null. |
| `sails[]` | Префабы парусов. |
| `sailColors[]` | Палитра цветов парусов. |
| `shipItems[]` | Кеш `ShipItem` по каждому префабу. |

**Маппинг товар↔предмет** (асимметричный, критично для модов экономики):

```csharp
GoodToItemIndex(goodIndex): goodIndex > 30 ? goodIndex + 170 : goodIndex
ItemToGoodIndex(itemIndex): itemIndex > 200 ? itemIndex - 170 : itemIndex
```

Т.е. товары (goods) `0..30` совпадают с индексами предметов `0..30`, а товары `31+` лежат в предметах начиная с индекса `201`.

## FloatingOriginManager — плавающее начало координат

Мир Sailwind огромен, поэтому используется **floating origin** (игрок держится около начала координат сцены, мир сдвигается). Океан — Crest.

| Член | Назначение |
|------|-----------|
| `instance` | Синглтон. |
| `outCurrentOffset` (`Vector3`) | Текущий сдвиг мира. **Все сейвы хранят позиции за вычетом этого offsets.** |
| `ShiftingPosToRealPos(pos)` | `pos - outCurrentOffset` (сцена → «реальные» координаты). |
| `RealPosToShiftingPos(pos)` | `pos + outCurrentOffset` («реальные» → сцена). |
| `GetGlobeCoords(obj)` | `(pos - outCurrentOffset - globeOffset) / 9000f` — координаты глобуса (делитель **9000**). |
| `shiftDistance` | Порог смещения игрока, после которого мир сдвигается. |
| `ShiftingThisFrame` / `CurrentShiftVector` | Статические флаги текущего кадра сдвига. |
| `shiftingRigidbodies` | Список `ShiftingRigidbody`, которые нужно скорректировать при сдвиге. |

> **Для моддера:** любые сохраняемые или передаваемые по сети координаты должны быть в «реальной» системе (за вычетом `outCurrentOffset`), иначе после сдвига мира объект уедет. Делитель глобуса `9000f` связывает мировые единицы с координатами на глобусе/карте.

## Вспомогательные сериализуемые типы

`SerializableVector3` и `SerializableQuaternion` — `[Serializable]`-структуры с неявными операторами конвертации в/из `UnityEngine.Vector3`/`Quaternion`. Используются потому, что `BinaryFormatter` не умеет напрямую сериализовать типы Unity.

## Практические выводы для мододела

1. **Персистентность:** пишите в `GameState.modData["MyMod.*"]` — сохранится само. Не трогайте `BinaryFormatter` и файлы сейвов.
2. **Тайминг:** если нужно что-то сделать строго до/после сохранения — патчьте `SaveLoadManager.SaveModData()` / `LoadModData()` (они пустые и стабильные). Альтернатива — Harmony-патч `DoSaveGame`/`LoadGame`.
3. **Координаты:** всегда учитывайте `FloatingOriginManager.instance.outCurrentOffset`.
4. **Не сохраняйтесь** в кровати/на верфи — игра сама блокирует сейв в эти моменты; учитывайте это в своей логике.
5. **Индексы:** `sceneIndex` (0..254, 0=игрок), `prefabIndex` (реестр предметов), goods/items маппинг `>30 → +170`.
