# 34. Рабочий пример: плавающие лут-ящики в море

Конкретный рецепт мода, разбираемый как учебный пример: **«по морю случайно спавнятся плавающие ящики с лутом, которые игрок может подобрать»**. Связывает заметки 11, 13, 16, 31, 33. Цель — показать, как из задокументированных «ручек» собирается реальная фича, и какие грабли учесть.

## Что уже даёт игра (не нужно делать)

| Нужно | Берём готовым | Заметка |
|-------|---------------|---------|
| Предмет можно подобрать | `GoPointer.PickUpItem` (клик по raycast) | 33, 10 |
| Предмет плавает | `ItemRigidbody` сам добавит `SimpleFloatingObject` | 33 |
| Предмет сохранится | `SaveablePrefab.RegisterToSave()` → `savedPrefabs` | 11 |
| Высота воды в точке | `OceanHeight.GetHeight(helper, pos)` | 31 |
| Координаты мира | `FloatingOriginManager.instance` (`outCurrentOffset`) | 11 |
| Периодика | `Sun.OnNewDay` или свой таймер в `Update` | 18 |
| Данные мода между сессиями | `GameState.modData` | 11 |

**Делать нужно только:** свой префаб предмета + свою логику спавна (где/когда/сколько).

## Шаг 1. Префаб лут-ящика

Нужен GameObject-префаб с компонентами (минимум):
- `ShipItemCrate` (или `ShipItem`) + `Rigidbody` + `Collider` + `Renderer`;
- `SaveablePrefab` с **уникальным `prefabIndex`**;
- настроенный `containedPrefab` (что внутри) и `amount` (сколько).

**Грабли:**
- `prefabIndex` должен существовать в `PrefabsDirectory.instance.directory`. В рантайме чужой префаб туда не дописать напрямую (массив фиксирован, индекс валидируется в `Start`). **Варианты:** (а) переиспользовать существующий индекс подходящего предмета (например, одного из ящиков) и подменить `containedPrefab`/`amount` после спавна; (б) для «настоящего» нового предмета — AssetBundle + патч `PrefabsDirectory` (отдельная большая тема).
- Для учебного примера проще всего **переиспользовать существующий префаб ящика** из `directory` и менять содержимое.

## Шаг 2. Где спавнить — случайная точка на воде рядом с игроком

```csharp
// позиция игрока в «реальных» координатах
var playerReal = FloatingOriginManager.instance
                    .ShiftingPosToRealPos(Refs.observerMirror.transform.position);

// случайное смещение в кольце 200..800 м от игрока (1 ед. = 1 м, заметка 28)
float ang = Random.Range(0f, Mathf.PI * 2f);
float dist = Random.Range(200f, 800f);
var realPos = playerReal + new Vector3(Mathf.Cos(ang) * dist, 0f, Mathf.Sin(ang) * dist);

// высота воды в этой точке (Crest)
var helper = new SampleHeightHelper();          // класс Crest, см. заметку 24/31
float waterY = OceanHeight.GetHeight(helper, realPos);
realPos.y = waterY;

// обратно в координаты сцены (с учётом плавающего начала координат!)
var scenePos = FloatingOriginManager.instance.RealPosToShiftingPos(realPos);
```

**Грабли:**
- **Плавающее начало координат**: мир сдвигается (`outCurrentOffset`). Спавнить нужно в **координатах сцены** (`RealPosToShiftingPos`), а «думать» о мире — в реальных (`ShiftingPosToRealPos`). Если заспавнить в «реальных» координатах напрямую — предмет уедет на километры после сдвига.
- **Не спавнить слишком далеко**: игра (`WorldItemSpawner`) использует порог 100 м для видимого спавна. Для лута в кольце 200–800 м предмет появится вне экрана — это нормально, но учти, что физика далеко может быть «усыплена».
- **`SampleHeightHelper`** — из отсутствующего Crest-модуля (заметка 24); его нужно взять из сборки. Альтернатива — спавнить чуть выше воды и положиться на `SimpleFloatingObject`, который сам опустит предмет на поверхность.

## Шаг 3. Спавн (по «ритуалу» из заметки 33)

```csharp
void SpawnLootCrate(Vector3 scenePos)
{
    // берём существующий префаб ящика из директории (учебный вариант)
    var prefab = PrefabsDirectory.instance.directory[cratePrefabIndex];

    var go = Object.Instantiate(prefab, scenePos, Quaternion.identity);
    var item = go.GetComponent<ShipItem>();

    // «ритуал» мирового предмета:
    item.sold = true;                                    // принадлежит миру
    go.GetComponent<Good>()?.RegisterAsMissionless();    // не миссия

    // (опционально) подменить содержимое ящика:
    // var crate = go.GetComponent<ShipItemCrate>();
    // crate.amount = Random.Range(1, 5);  // и т.п.

    // сохранение:
    go.GetComponent<SaveablePrefab>().RegisterToSave();
}
```

Подбор, плавание и сохранение **заработают сами** (шаг «что уже даёт игра»).

**Как игрок заберёт лут** (механика ящика, заметка 33): игрок подбирает плавающий ящик → ПКМ по запечатанному ящику показывает `CrateSealUI` (подтверждение вскрытия) → при вскрытии содержимое **перекладывается в инвентарь ящика** (ящик остаётся целым контейнером, `amount → 0`) → дальше ящик открывается как контейнер и из него достаются предметы. То есть «лут-дроп» здесь — это именно плавающий запечатанный ящик, а не рассыпанные по воде предметы (так надёжнее: ничего не тонет и не теряется).

## Шаг 4. Когда спавнить — плагин BepInEx

```csharp
[BepInPlugin(GUID, "FloatingLoot", "1.0.0")]
public class FloatingLootPlugin : BaseUnityPlugin
{
    public const string GUID = "com.example.floatingloot";
    private float _timer;
    private Harmony _harmony;

    private void Awake()
    {
        _harmony = new Harmony(GUID);
        // патч SaveModData/LoadModData для своих данных, если нужно (заметка 11)
        _harmony.PatchAll();
        Logger.LogInfo("FloatingLoot loaded");
    }

    private void Update()
    {
        if (!GameState.playing || GameState.currentlyLoading) return;

        _timer -= Time.deltaTime;
        if (_timer <= 0f)
        {
            _timer = 300f;                      // раз в 5 минут реального времени
            if (ActiveCrates < MaxCrates)        // свой учёт (см. грабли)
                SpawnLootCrate(RandomOceanPoint());
        }
    }
}
```

**Грабли и решения:**
- **Учёт активных ящиков**: игра не ведёт реестр «моих» предметов. Свой список (`List<SaveablePrefab>`/instanceId) чистить по `OnDestroy`/`Unregister`, иначе счётчик уплывёт.
- **Персистентность между сессиями**: заспавненные ящики сохраняются штатно (`RegisterToSave` → `savedPrefabs`), но **список точек/состояния своего спавнера** (таймер, счётчик) храните в `GameState.modData` (заметка 11) или через патч `SaveModData/LoadModData`.
- **Пауза времени**: `Update` идёт в реальном времени; если хотите «игровое» время — умножайте на `Sun.sun.timescale` (заметка 18) или подписывайтесь на `Sun.OnNewDay`.
- **Не спавнить во сне/на верфи/в восстановлении**: проверять `GameState.sleeping`, `GameState.currentShipyard`, `GameState.recovering`.
- **Лимит предметов**: слишком много живых `Rigidbody` бьёт по производительности; держите `MaxCrates` небольшим и despawnите дальние (по `Vector3.Distance` до игрока).

## Шаг 5. Despawn далёкого/старого лута

```csharp
// периодический проход по своему списку:
if (Vector3.Distance(crate.position, Refs.observerMirror.position) > 3000f)
    crate.GetComponent<ShipItem>().DestroyItem();   // снимает с сейва + Destroy (заметка 16)
```
`DestroyItem()` корректно разрегистрирует предмет из сохранения (`SaveablePrefab.Unregister`).

## Чек-лист фичи

- [x] Префаб предмета с `ShipItem`+`SaveablePrefab` (+ валидный `prefabIndex`)
- [x] Случайная точка на воде: реальные координаты → `OceanHeight.GetHeight` → координаты сцены (`RealPosToShiftingPos`)
- [x] Спавн-ритуал: `Instantiate` → `sold=true` → `RegisterAsMissionless` → `RegisterToSave`
- [x] Подбор/плавание/сохранение — из коробки
- [x] Таймер/лимиты в `Update` (или `Sun.OnNewDay`), despawn дальних через `DestroyItem`
- [x] Состояние спавнера — в `GameState.modData`

## Обобщение: рецепт «любой мировой предмет»

Любая фича «предмет в мире» строится одинаково:
1. **Префаб** `ShipItem` + `SaveablePrefab` (+ валидный индекс; для ящиков — фильтр в заметке 45).
2. **Позиция**: думать в реальных координатах, спавнить в координатах сцены (`±outCurrentOffset`), Y — канонический сниппет высоты воды в заметке 43.
3. **Ритуал**: `sold=true` (+ `RegisterAsMissionless`), `RegisterToSave`; **дождаться twin** (заметка 44).
4. **Подбор, сохранение, UI-подсказка** даёт игра. **Плавание — нет**: штатный floater выключен (`ToggleCollider`), нужен **свой поплавок** (Boyant-снап) или патч — **полностью в заметке 43** (truth table, sequence, чек-лист, антипаттерны); twin/visual-контракт — в заметке 44.

> **Критично для «плавает и кликается»:** физику/поплавок вешать только на **twin** (`GetItemRigidbody()`), не на visual (иначе ломается подбор); тело держать не-кинематическим (`debugForceKinematic=false` на `ItemRigidbody`, `sold=true`). Чек-лист end-to-end — в заметке 43.

Этот же скелет покрывает: плавающие бочки, обломки с лутом, «сокровища» по координатам, дрейфующие предметы, маркеры-предметы и т.п.
