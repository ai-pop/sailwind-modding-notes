# 43. Плавучесть предметов и вода: механика подробно

Подробный технический разбор того, как предметы взаимодействуют с водой: плавучесть, кинематика, высота воды. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 14 (плавучесть судов), 16 (ShipItem), 31 (океан), 33 (спавн/подбор).

## Два GameObject у предмета

Напоминание (заметка 16): у каждого `ShipItem` **два** объекта:
- **visual** — сам `ShipItem` (Renderer, триггер-коллайдеры, логика);
- **`itemRigidbody`** — отдельный runtime-объект (`ItemRigidbody`), создаётся в `ShipItem.CreateRigidbody()`; на нём живут `Rigidbody`, физические коллайдеры и **`SimpleFloatingObject`** (плавучесть).

Плавучесть применяется к **`itemRigidbody`**, не к visual. Visual следует за `itemRigidbody` (или наоборот, в зависимости от состояния).

## `ItemRigidbody`: создание и floater

В `Start()` (на объекте `itemRigidbody`):
```csharp
rigidbody = AddComponent<Rigidbody>();
rigidbody.drag = 1.2f;
rigidbody.angularDrag = item.mass * 0.1f;
rigidbody.isKinematic = true;                 // стартует кинематическим
UpdateMass();                                 // масса = item.mass + содержимое (ящик/бутылка/суп/соль/чай)
StartCoroutine(AddCollider());                // Box/Mesh/Capsule коллайдеры (копия с visual)
floater = AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;    // высота всплытия = ShipItem.floaterHeight (по умолч. 1.6)
gameObject.layer = 2;
ResetPos();                                   // rigidbody.isKinematic = true (ещё раз)
world = FloatingOriginManager.instance.transform;
SetDynamicColTimer();                         // dynamicColTimer = 6
```

- `SimpleFloatingObject` (Crest, базовый класс `FloatingObjectBase`) создаётся **для каждого предмета**.
- `_raiseObject` берётся из `ShipItem.floaterHeight` — целевая высота над водой.
- `FloatingObjectBase.InWater` — флаг «объект в воде».
- Тело **стартует кинематическим** (`isKinematic = true`).

## Кинематика: когда предмет динамический, а когда нет

В `FixedUpdate` вычисляется `flag2` → `rigidbody.isKinematic = flag2`. Предмет **кинематический** (`flag2 = true`), если верно **хотя бы одно**:

| Условие | Комментарий |
|---------|-------------|
| `item.held` | предмет в руках |
| `!item.sold` | не куплен (магазинный) |
| `item.nailed` | прибит/закреплён |
| `attached` | прикреплён (напр. к стене/плите) |
| `currentInventorySlot` или `currentBox` | в инвентаре/ящике |
| `outOfRange` | дальше **600** ед. от `Camera.main` |
| `!GameState.playing \|\| recovering \|\| sleeping \|\| inBed \|\| currentShipyard` | служебные состояния |
| `fixedFramesSinceSpawn < 6` | первые 6 fixed-кадров после спавна |
| `debugForceKinematic` (поле предмета) | принудительно (ставит, напр., `WorldItemSpawner`) |
| `Debugger.kinematicItemsTimer > 0` / `debugForceKinematicBoat` | отладочные флаги |
| `meshCol != null && rigidbody.IsSleeping() && dynamicColTimer <= 0` | mesh-коллайдер «уснул» |

Предмет **динамический** (`isKinematic = false`) только когда: куплен (`sold`), не в руках/инвентаре/ящике, не прибит/прикреплён, **в пределах 600 ед.** от камеры, игра активна, прошло ≥ 6 fixed-кадров, нет принудительной кинематики.

> Кинематическое тело **не реагирует на силы** (в т.ч. на силу плавучести). Поэтому для плавучести тело должно быть динамическим.

## Управление floater: особенность `ToggleCollider`

```csharp
public void ToggleCollider(bool state) {
    ((Behaviour)floater).enabled = false;   // ← безусловное выключение floater
    if (boxCol) boxCol.enabled = state;
    if (meshCol) meshCol.enabled = state;
    if (capsuleCol) capsuleCol.enabled = state;
}
```

- `ToggleCollider` **всегда** выставляет `floater.enabled = false` (независимо от `state`).
- В `FixedUpdate` для предмета вне инвентаря вызывается `ToggleCollider(true)` (включить коллайдеры) — и это **заодно выключает floater**.
- В коде игры **нет** места, которое выставляло бы `floater.enabled = true` обратно.

То есть у предмета, идущего через штатный `ItemRigidbody`, `SimpleFloatingObject` создаётся включённым, но фактически **переводится в `enabled = false`** и игрой обратно не включается. Реагирует ли при этом Crest-плавучесть — зависит от внутренней реализации `SimpleFloatingObject` (класс в выгрузке отсутствует): если она опирается на собственный `enabled`-цикл — плавучесть не применяется; нужна runtime-проверка.

### «Усыпление» физики вдали от игрока
- `outOfRange` = дистанция до `Camera.main` > **600** ед. (проверка раз в 5–8 c, `distanceCheckTimer`).
- При `outOfRange` `FixedUpdate` делает **ранний выход** (физика не управляется), а для купленного предмета не на лодке считает `framesUntilDestroy`; через > 10 кадров — `DestroyItem()` (дальние предметы **уничтожаются**).

## Что реально держит предмет на воде: 4 механизма

| Механизм | Класс | Как работает | Кем используется |
|----------|-------|--------------|------------------|
| **Crest-поплавок** | `SimpleFloatingObject` (`FloatingObjectBase`) | Сила/подъём к поверхности, `_raiseObject`, `InWater` | `ItemRigidbody` (создаёт, но см. `ToggleCollider`), **рабочие** примеры: рыболовный поплавок (`FishingRodFish`/`ShipItemChipLog`), `ChipLogRopeEnd` |
| **Снап по высоте** | `Boyant` | В `Update`: если `ocean.canCheckBuoyancyNow[0]==1`, ставит `position.y = GetWaterHeightAtLocation2(x−choppy, z) + buoyancy` (прямая установка высоты) | Определён, но другими классами **не вызывается** (возможен на префабах в сценах) |
| **Блоб-плавучесть** | `Buoyancy` (заметка 14) | Сетка точек, сила по глубине погружения | Суда, крупные объекты |
| **Океан напрямую** | `Ocean` (Crest) | `GetWaterHeightAtLocation2`, `GetChoppyAtLocation2` | Источник данных для всех выше |

### Рабочие примеры плавающих предметов
- **Рыболовный поплавок** (`FishingRodFish.bobber`, `ShipItemChipLog.bobberBody`): отдельное **динамическое** тело (`!isKinematic`) со **своим включённым** `SimpleFloatingObject` (не через `ItemRigidbody.ToggleCollider`). Поплавок гарантированно плавает — на этом держится вся механика рыбалки (`floater.InWater` проверяется постоянно).
- **Щепка** (`ChipLogRopeEnd`): `GetComponent<SimpleFloatingObject>()` с префаба, читает `InWater`.

Общая черта рабочих случаев: **динамическое тело + включённый `SimpleFloatingObject`, которым никто не управляет через `ToggleCollider`**.

## Высота воды: правильный вызов

| API | Что делает |
|-----|-----------|
| `Ocean.Singleton` | Статический доступ к океану. |
| `ocean.GetWaterHeightAtLocation2(x, z)` | Высота поверхности воды в точке. |
| `ocean.GetChoppyAtLocation2(x, z)` / `GetChoppyAtLocation(x, z)` | Горизонтальное смещение («choppy»); высоту брать в `x − choppy`. |
| `ocean.canCheckBuoyancyNow[0]` | Байт-гейт: `== 1`, когда Crest готов считать высоту (ставлять 0/1 внутри `Ocean`). Запросы высоты имеют смысл только при `== 1`. |
| `ocean.choppy_scale` (2f) | Масштаб choppy. |
| `OceanHeight.GetHeight(SampleHeightHelper, pos[, accuracy])` | Обёртка над Crest `SampleHeightHelper.Init/Sample` (заметки 24/31). |
| `OceanHeight.IsUnderwater(helper, pos)` | `GetHeight(helper, pos) > pos.y`. |

**Плавающее начало координат** (заметка 11): высоту воды запрашивать в **координатах сцены** (как стоит объект); для «реальных»/глобальных расчётов — `FloatingOriginManager.instance.ShiftingPosToRealPos / RealPosToShiftingPos`. `GetWaterHeightAtLocation2` работает с текущими (сценными) координатами.

## Ритуал «предмет плавает в мире» (минимальная последовательность)

После `Instantiate(prefab, scenePos, rot)` (создаётся visual; `ShipItem` сам создаст `itemRigidbody` через `CreateRigidbody`):

1. `var item = go.GetComponent<ShipItem>(); item.sold = true;` — иначе `!sold` → кинематика (заметка 33).
2. `go.GetComponent<Good>()?.RegisterAsMissionless();`
3. `go.GetComponent<SaveablePrefab>().RegisterToSave();` — сохранение.
4. **Не ставить** `itemRigidbody.debugForceKinematic = true` (это заморозит тело; так делает `WorldItemSpawner`).
5. Держать предмет в пределах 600 ед. от камеры (иначе ранний выход физики + уничтожение).
6. **Плавучесть** — штатный `SimpleFloatingObject` у предмета через `ItemRigidbody` фактически выключен (`ToggleCollider`, см. выше). Чтобы предмет реально плавал, нужно **одно из**:
   - **(a)** включить и удерживать `floater.enabled = true` (патч `ItemRigidbody.ToggleCollider`, чтобы не трогал floater, либо пере-включение в `LateUpdate` после `ItemRigidbody`); тело при этом должно быть динамическим;
   - **(b)** добавить свой поплавок по образцу `Boyant` — снап `position.y` к `GetWaterHeightAtLocation2` (просто, не конфликтует с `ItemRigidbody`);
   - **(c)** гибрид: оставить `ItemRigidbody` для массы/коллизий, а Y вести самостоятельно.

### Код-заготовка (BepInEx, вариант b — свой снап-поплавок)
```csharp
// свой компонент на visual-объекте предмета
class KeepOnSurface : MonoBehaviour {
    Rigidbody _body; Ocean _ocean;
    public float buoyancy = 0.3f;
    void Start() { _ocean = Ocean.Singleton; _body = GetComponentInChildren<Rigidbody>(); }
    void LateUpdate() {
        if (_ocean == null || _ocean.canCheckBuoyancyNow[0] != 1) return;
        var p = transform.position;
        float chop = _ocean.GetChoppyAtLocation2(p.x, p.z);
        float y = _ocean.GetWaterHeightAtLocation2(p.x - chop, p.z) + buoyancy;
        // мягко тянем к поверхности (или ставим напрямую, как Boyant)
        p.y = Mathf.Lerp(p.y, y, Time.deltaTime * 3f);
        transform.position = p;
    }
}
// спавн:
var go = Object.Instantiate(itemPrefab, scenePos, Quaternion.identity);
var item = go.GetComponent<ShipItem>(); item.sold = true;
go.GetComponent<Good>()?.RegisterAsMissionless();
go.GetComponent<SaveablePrefab>().RegisterToSave();
go.AddComponent<KeepOnSurface>();   // свой поплавок
// ВАЖНО: не ставить debugForceKinematic; держать в радиусе 600 от камеры
```

## Сводка: от чего зависит, плавает ли предмет

1. **Тело динамическое?** (`sold`, не held/nailed/attached, не в инвентаре/ящике, в радиусе 600, игра активна, ≥ 6 кадров, нет `debugForceKinematic`). Кинематика → сил нет → не плавает.
2. **Плавучесть включена?** Штатный `SimpleFloatingObject` у `ItemRigidbody` выключается `ToggleCollider` и игрой не включается → для реальной плавучести нужен включённый floater (патч) ИЛИ свой поплавок (`Boyant`-снап).
3. **Высота воды** берётся из `Ocean.GetWaterHeightAtLocation2(x−choppy, z)` при `canCheckBuoyancyNow[0]==1`, в координатах сцены.
4. **Рабочий референс** — рыболовный поплавок: динамическое тело + свой включённый `SimpleFloatingObject`.

## Truth table: тип предмета → плавает?

(по логике кода v0.38; для предметов через `ItemRigidbody` согласуется с runtime-наблюдением `floater=False` → тонет)

| Тип предмета | floater создан? | floater `enabled`? | Кинематика по умолчанию | Плавает в ваниле? |
|--------------|:--:|:--:|--------------------------|:--:|
| Обычный `ShipItem` (свободный, `sold`) | да | **НЕТ** (выкл. `ToggleCollider`) | стартует кинематиком → динамический, когда свободен/в зоне/playing | фактически **нет** |
| Магазинный (`!sold`) | да | **НЕТ** | **кинематик** (`!sold` → `flag2`) | нет |
| Миссионный товар (`Good`) | да | **НЕТ** | как свободный `sold` | нет |
| World-spawned (`WorldItemSpawner`) | да | **НЕТ** | **кинематик** (`debugForceKinematic=true`) | нет (заморожен на месте) |
| Рыболовный поплавок (`FishingRodFish`/`ShipItemChipLog`) | свой | **ДА** | динамический | **ДА** |
| Щепка (`ChipLogRopeEnd`) | свой | **ДА** | динамический | **ДА** |
| Лодка | `Buoyancy` (блоб), не `SimpleFloatingObject` | — | — | **ДА** (другая система, заметка 14) |

Вывод: у предметов, идущих через штатный `ItemRigidbody`, floater создан, но **выключен** и телом часто управляет кинематика → ванильной плавучести нет. Гарантированно плавают объекты со **своим включённым** `SimpleFloatingObject` на динамическом теле (поплавок, щепка) и лодки (блоб-`Buoyancy`).

## Sequence: spawn → twin → floater → pickup → drop → float

```
Object.Instantiate(prefab, scenePos, rot)            // создаётся VISUAL (ShipItem)
  └─ ShipItem.Awake()
       └─ StartCoroutine(LoadAfterDelay())
            ├─ CreateRigidbody()                     // TWIN: new GO + ItemRigidbody
            │     ├─ rigidbody (isKinematic = true)
            │     └─ floater = AddComponent<SimpleFloatingObject>()  // ENABLED, _raiseObject=floaterHeight
            ├─ AddCollisionChecker() / AddLODGroup()
            ├─ [WAIT WaitForEndOfFrame]              // ← twin появляется НЕ синхронно с Instantiate
            └─ OnLoad()
  ── (мод) sold = true; Good?.RegisterAsMissionless(); SaveablePrefab.RegisterToSave()
  ── ItemRigidbody.FixedUpdate() [каждый fixed-кадр]
       ├─ outOfRange (>600 от камеры)? → ранний выход (+ culling)
       ├─ ToggleCollider(true)  →  floater.enabled = false      // ← floater ВЫКЛЮЧЕН
       └─ rigidbody.isKinematic = flag2                          // dynamic, если sold/свободен/в зоне/playing
  ── игрок: GoPointer raycast по visual-триггеру → клик → PickUpItem()   // в руки (twin следует за visual)
  ── drop → предмет в мире, TWIN = master по позиции (visual следует за twin)
  ── плавучесть: штатный floater выкл → для плавания нужен СВОЙ поплавок (Boyant-снап) или патч (ниже)
```

## Канонический сниппет высоты воды (+ floating origin)

Один блок вместо разрозненных заметок 11/31/34:

```csharp
// Мгновенная высота поверхности воды в ТЕКУЩЕЙ позиции объекта.
// Паттерн из Boyant/Buoyancy (v0.38). Вход — SCENE-координаты объекта.
public static bool TryGetWaterY(Vector3 scenePos, out float y)
{
    var o = Ocean.Singleton;
    y = scenePos.y;                                   // fallback: держать текущий Y
    if (o == null || o.canCheckBuoyancyNow == null || o.canCheckBuoyancyNow[0] != 1)
        return false;                                 // Crest не готов → НЕ сэмплировать
    float chop = o.GetChoppyAtLocation2(scenePos.x, scenePos.z);
    y = o.GetWaterHeightAtLocation2(scenePos.x - chop, scenePos.z);   // вычесть choppy!
    return true;
}
```

- **Вход = scene xz** (текущая позиция объекта). `GetWaterHeightAtLocation2` **сама тайлит** волновую сетку (`sizeQX/sizeQZ` + дробная часть), поэтому для мгновенного лукапа ручной перевод real↔scene **не нужен**.
- **Мусорный сэмпл (~0.4)** возникает при: (а) сэмпл, когда `canCheckBuoyancyNow[0] != 1` (данные Crest не готовы), или (б) не вычтен `chop` (`GetChoppyAtLocation2`). Гейт + вычет choppy убирают проблему.
- **Floating origin** важен для **хранения/сравнения** позиций во времени (спавн-точки, дистанция despawn): хранить в **real** (`ShiftingPosToRealPos`), применять в **scene** (`RealPosToShiftingPos`) — заметка 11. Для мгновенной высоты в текущей позиции — scene напрямую.
- **Fallback**, если `Ocean`/Crest недоступен: высоту взять неоткуда — держать текущий `y` или фиксированный уровень; снап без высоты невозможен.

## End-to-end чек-лист: «предмет плавает в мире и кликается»

1. `Object.Instantiate(prefab, scenePos, rot)` (visual; `ShipItem` сам создаст twin).
2. `sold = true`; `GetComponent<Good>()?.RegisterAsMissionless()`; `GetComponent<SaveablePrefab>().RegisterToSave()`.
3. **Дождаться twin**: `while (itemRigidbodyC == null) yield return null;` (или 1–2 `WaitForEndOfFrame`).
4. **Плавучесть** — свой поплавок на **twin** (Boyant-снап через `TryGetWaterY`) ИЛИ патч `ToggleCollider` + держать `floater.enabled = true`. **Не на visual.**
5. **Не кинематик**: `GetItemRigidbody().debugForceKinematic = false`; предмет `sold`, не held/attached/nailed, в зоне 600, `playing`.
6. **Pickup/unseal работают**: на visual ничего не вешать (триггер-коллайдеры visual = raycast подбора); ящик вскрывается штатно (`CrateSealUI` → `CrateInventory`).
7. **Save/load**: сам предмет сохраняется через `RegisterToSave` (`savedPrefabs`); состояние своего спавнера (таймер, список instanceId) — в `GameState.modData`.

## Сводные `GameState`-гейты для тикающего/спавнящего мода

| Флаг | Когда | Что гейтить |
|------|-------|-------------|
| `GameState.playing` | активная игра | весь тик мода |
| `GameState.sleeping` | сон | не спавнить/не двигать |
| `GameState.recovering` | обморок/восстановление | не спавнить |
| `GameState.currentShipyard != null` | на верфи | не спавнить |
| `GameState.currentlyLoading` | загрузка сцены | не спавнить, ждать |
| `Refs.observerMirror == null` | нет позиции игрока | не спавнить |
| StartMenu / загрузочная анимация | главное меню | не тикать (покрывается `playing == false`) |

Единый гейт:
```csharp
bool CanTick() =>
    GameState.playing && !GameState.sleeping && !GameState.recovering &&
    GameState.currentShipyard == null && !GameState.currentlyLoading &&
    Refs.observerMirror != null;
```

## Антипаттерны (на чем ломаются)

- ❌ **`AddComponent<SimpleFloatingObject>`/`Rigidbody` на VISUAL** → ломает подбор: для свободного предмета master = twin (перезаписывает visual), а триггер-коллайдеры visual используются для raycast. **Физику/поплавок — только на twin.**
- ❌ **Хранить spawn-позицию в scene-координатах** и применять после сдвига мира → уезжает. Хранить **real**, применять **scene** (заметка 11).
- ❌ **Сэмпл `GetWaterHeightAtLocation2` при `canCheckBuoyancyNow[0] != 1`** → мусор (~0.4).
- ❌ **Не вычесть `chop`** (`GetChoppyAtLocation2`) → высота смещена.
- ❌ **`timer = 0` после пустого `modData`** → нулевой/бесконечный интервал спавна. При отсутствии ключа — инициализировать дефолтом.
- ❌ **`debugForceKinematic = true`** когда нужно плавание (это заморозка; так делает `WorldItemSpawner`).
- ❌ **`shipItem.debugForceKinematic`** → CS1061: поле на **`ItemRigidbody`** (`GetItemRigidbody().debugForceKinematic`).
- ❌ **`UISoundPlayer.instance` на раннем буте** без guard → null. Проверять `instance != null` / try.
- ❌ **`GetComponent<ShipItemCrate>()` как фильтр «грузовых ящиков»** → ловит `tobacco big`, `apples` (заметка 45).
- ❌ **Доступ к twin сразу после `Instantiate`** → `itemRigidbodyC == null` (twin создаётся в корутине из `Awake`).

## Практические выводы

- `SimpleFloatingObject` создаётся на каждом предмете (`_raiseObject = floaterHeight`), но штатный путь `ItemRigidbody.ToggleCollider` переводит его в `enabled = false` без обратного включения в коде игры.
- Кинематика гейтится длинным списком условий (таблица выше); вдали (>600) физика отключается, а купленные предметы вне лодки уничтожаются.
- Гарантированно плавают объекты с **динамическим телом и включённым `SimpleFloatingObject` вне управления `ToggleCollider`** (поплавок, щепка).
- `Boyant` — готовый простой снап-поплавок (`GetWaterHeightAtLocation2 + buoyancy`), удобный образец для своего решения.
- Запросы высоты воды — только при `canCheckBuoyancyNow[0]==1`, в координатах сцены, с учётом choppy-смещения.
