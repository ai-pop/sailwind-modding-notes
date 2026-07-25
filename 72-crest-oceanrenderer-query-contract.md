# 72. Crest `OceanRenderer` и контракт CPU-запросов поверхности

Полный разбор core runtime API `Crest.dll` Sailwind v0.38: `OceanRenderer`, collision/flow providers, `SampleHeightHelper`, `SampleFlowHelper` и GPU query pipeline. Это не реконструкция по вызовам игры: классы декомпилированы непосредственно из поставляемого `Crest.dll`.

Связано с [31](31-ocean-waves-inertia.md), [48](48-ocean-height-helper-lifecycle.md), [71](71-crest-simplefloatingobject-exact-model.md).

## `OceanRenderer` — singleton Crest, не `Ocean.Singleton`

В `Crest.dll` главный объект:

```csharp
public class OceanRenderer : MonoBehaviour
{
    public static OceanRenderer Instance { get; private set; }
    public Transform Root { get; private set; }
    public ICollProvider CollisionProvider { get; private set; }
    public IFlowProvider FlowProvider { get; private set; }
    public float SeaLevel => Root.position.y;
}
```

Это отдельная архитектурная сущность от игрового класса `Ocean` из `Assembly-CSharp.dll` (`Ocean.Singleton`). Sailwind содержит оба слоя:

| Слой | DLL / тип | Роль |
|---|---|---|
| Sailwind wrapper | `Ocean`, `OceanHeight` | игровая логика, старый API высоты, weather/visual integration |
| Crest runtime | `Crest.OceanRenderer` | LOD сетка, GPU data, collision/flow providers, wave inputs |

Мод не должен считать присутствие `OceanHeight` доказательством, что `OceanRenderer.Instance`, `CollisionProvider` или GPU readback уже готовы.

## Lifecycle `OceanRenderer`

`OnEnable()`:

1. проверяет `SystemInfo.supportsComputeShaders` и `supports2DArrayTextures`;
2. назначает `OceanRenderer.Instance = this`;
3. создаёт `LodTransform` и LOD data;
4. вызывает `OceanBuilder.GenerateMesh()` для ocean tile root;
5. создаёт subsystems (animated waves всегда; flow/foam/dynamic waves/depth/shadows/clip — по флагам);
6. создаёт `CollisionProvider` и `FlowProvider`;
7. выбирает viewpoint (`_viewpoint`, иначе `Camera.main`).

`LateUpdate()` → `RunUpdate()` только при `Time.timeScale > 0`:

```text
CollisionProvider.UpdateQueries()
FlowProvider.UpdateQueries()
→ global shader params
→ Root follows viewpoint XZ
→ scale/LOD update
→ all LOD data update
→ command buffer build/execute
```

Следствие: CPU query и result могут быть frame-delayed; query, сделанный до `RunUpdate`, не обязан иметь свежий результат в тот же момент.

## Важные `OceanRenderer` параметры

| Поле / property | Default / смысл |
|---|---|
| `_layerName` | `Water`; `OceanBuilder` ищет LayerMask по имени |
| `_gravityMultiplier` | 1; `Gravity = multiplier × Physics.gravity.magnitude` |
| `_minTexelsPerWave` | 3; определяет spatial filtering маленьких волн |
| `_minScale` / `_maxScale` | 8 / 256; масштаб ocean LOD вокруг viewpoint |
| `_lodDataResolution` | 256 |
| `_geometryDownSampleFactor` | 2 |
| `_lodCount` | 7, максимум системы — 15 |
| `_createSeaFloorDepthData` | true |
| `_createFoamSim` | true |
| `_createDynamicWaveSim` | optional, false по default Crest field |
| `_createFlowSim` | optional |
| `_createShadowData` / `_createClipSurfaceData` | optional |

`SeaLevel` — это `Root.position.y`, а не обязательно 0.

## Collision provider: три режима

`SimSettingsAnimatedWaves.CollisionSources`:

```csharp
public enum CollisionSources
{
    None,
    GerstnerWavesCPU,
    ComputeShaderQueries
}
```

| Режим | Provider | Особенность |
|---|---|---|
| `None` | `CollProviderNull` | всегда возвращает нулевой displacement, `Vector3.up` normal, zero velocity; Query status OK |
| `GerstnerWavesCPU` | найденный `ShapeGerstnerBatched` | синхронная CPU аналитика Gerstner волн |
| `ComputeShaderQueries` | `QueryDisplacements` | GPU compute + `AsyncGPUReadback`, актуальные данные могут приходить с задержкой |

Если нужный provider нельзя создать, `CreateCollisionProvider()` возвращает `CollProviderNull`, а не `null`.

## `ICollProvider` / `IFlowProvider`

```csharp
public interface ICollProvider
{
    int Query(int ownerHash, float minSpatialLength, Vector3[] points,
              float[] heights, Vector3[] normals, Vector3[] velocities);
    int Query(int ownerHash, float minSpatialLength, Vector3[] points,
              Vector3[] displacements, Vector3[] normals, Vector3[] velocities);
    bool RetrieveSucceeded(int status);
    void UpdateQueries();
    void CleanUp();
}
```

```csharp
public interface IFlowProvider
{
    int Query(int ownerHash, float minSpatialLength, Vector3[] points,
              Vector3[] flows);
    bool RetrieveSucceeded(int status);
    void UpdateQueries();
    void CleanUp();
}
```

### `ownerHash` — не декоративный параметр

`QueryBase` использует `ownerHash` как key регистрации segment в кольцевом буфере. Повторно используйте **стабильный** hash одного query owner; не генерируйте случайный ID каждый кадр.

Ограничения Compute query:

| Лимит | Значение |
|---|---:|
| default max query count | 4096 (`SimSettingsAnimatedWaves._maxQueryCount`) |
| max registered GUID/owner | 1024 |
| ring buffer segment pool | 7 |
| устаревшие регистрации | сохраняются максимум ~10 frames при acquire нового segment |
| finite-difference normal stencil | 0.1 world unit |

Переполнение кольцевого буфера логирует:

```text
Query ring buffer exhausted. Please report this to developers.
```

## Query status bits

`QueryBase.QueryStatus`:

| Bit | Enum | Значение |
|---:|---|---|
| 0 | `OK` | success |
| 1 | `RetrieveFailed` | latest GPU/readback result для owner недоступен |
| 2 | `PostFailed` | query points не были успешно поставлены |
| 4 | `NotEnoughDataForVels` | нет двух snapshots для velocity |
| 8 | `VelocityDataInvalidated` | registration segment изменился |
| 16 | `InvalidDtForVelocity` | interval snapshots слишком мал |

`RetrieveSucceeded(status)` проверяет только `(status & 1) == 0`. Поэтому caller, которому нужна surface velocity, должен отдельно учитывать bits 4/8/16: высота может быть валидна, а velocity — нет.

## `SampleHeightHelper`: точная семантика

`SampleHeightHelper` держит массивы длины 1 и имеет четыре overload `Sample`.

```csharp
helper.Init(worldPosition, minSpatialLength, allowMultipleCallsPerFrame, context);
bool ok = helper.Sample(out float height);
```

При success:

```text
height = displacement.y + OceanRenderer.Instance.SeaLevel
```

При `CollisionProvider == null`:

```text
returns false
height = 0
normal = up
velocity = zero
```

При provider query, но result ещё не извлечён:

```text
returns false
height = SeaLevel
normal = up
velocity = zero
```

**Нельзя игнорировать bool result.** Возвращённый 0/SeaLevel может быть fallback, а не измеренной волной.

## `OceanHeight.GetHeight` Sailwind wrapper: скрытая ловушка

Игровой `OceanHeight.GetHeight(helper, pos[, accuracy])`:

```csharp
helper.Init(worldPos, accuracy, false, null);
float result = 0f;
if (helper.Sample(ref result)) return result;
return 0f;
```

Он возвращает `0f` при неудаче и не сообщает caller-у success/failure. Это объясняет, почему один лишь `float` от `OceanHeight` нельзя считать доказательством существующей поверхности.

> Для безопасного мода предпочтительнее напрямую использовать `SampleHeightHelper.Sample(...)` и bool либо держать собственный fallback state/diagnostics.

## `SampleFlowHelper`

```csharp
helper.Init(position, minSpatialLength);
bool ok = helper.Sample(out Vector2 flow);
```

Flow query возвращает XZ из `Vector3` result:

```csharp
flow.x = result.x;
flow.y = result.z;
```

При отсутствующем/неудачном provider — `false` и `Vector2.zero`.

Если `_createFlowSim` выключен, `OceanRenderer` создаёт `FlowProviderNull`, который возвращает zero flow с success status.

## Безопасный шаблон query для мода

```csharp
var helper = new Crest.SampleHeightHelper(); // хранить на body/service, не создавать каждый frame
helper.Init(scenePosition, minSpatialLength: objectWidth,
            allowMultipleCallsPerFrame: true);

float height;
Vector3 normal, surfaceVelocity;
bool exact = helper.Sample(out height, out normal, out surfaceVelocity);
if (!exact)
{
    // Не делайте из 0f «точную воду».
    // Используйте последний valid sample / собственный fallback / suspend.
}
```

### Выбор `minSpatialLength`

Crest преобразует его в min grid size:

```text
minGridSize = minSpatialLength / 2 / OceanRenderer.MinTexelsPerWave
```

Большое значение фильтрует короткие волны и снижает detail; маленькое требует мелких LOD и даёт более шумный ответ. Для физического body разумная отправная точка — характерная ширина корпуса, не 0 и не размер мира.

## Практические выводы

1. `OceanRenderer.Instance`, `CollisionProvider` и готовый readback — три разные readiness стадии.
2. `OceanHeight.GetHeight()` теряет bool failure; возвращённый 0 может быть fallback.
3. Compute query имеет лимиты: 4096 points по default, 1024 owner hash, ring buffer 7 сегментов.
4. Для normal Crest реально отправляет 3 query point на один normal (центр, +X 0.1, +Z 0.1).
5. CPU Gerstner provider (`ShapeGerstnerBatched`) — отдельный безопасный путь без GPU readback; его нужно рассматривать отдельно (заметка 73).
6. Не вызывайте query из бесконтрольного количества body каждый `FixedUpdate`; batch/cache/query service обязателен для массовых mod объектов.
