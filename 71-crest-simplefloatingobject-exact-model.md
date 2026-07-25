# 71. Crest `SimpleFloatingObject`: точная модель плавучести из `Crest.dll`

Эта заметка исправляет прежнюю неопределённость заметок 43/63: `Crest.dll` из Sailwind v0.38 был декомпилирован, поэтому поведение `Crest.SimpleFloatingObject` известно точно.

## Класс и lifecycle

```text
Crest.FloatingObjectBase : MonoBehaviour
  └─ Crest.SimpleFloatingObject
```

`SimpleFloatingObject.Start()` получает Rigidbody того же GameObject:

```csharp
_rb = GetComponent<Rigidbody>();
if (OceanRenderer.Instance == null)
    enabled = false;
```

То есть при существующем `OceanRenderer.Instance` компонент тикает в `FixedUpdate`; `enabled = false` действительно полностью останавливает его физические силы.

## Параметры

| Поле | Default | Смысл |
|---|---:|---|
| `_raiseObject` | 1 | вертикальный offset к поверхности; `ItemRigidbody.Start()` задаёт из `ShipItem.floaterHeight` (обычно **1.6**) |
| `_buoyancyCoeff` | 3 | коэффициент кубической подъёмной acceleration |
| `_boyancyTorque` | 8 | torque выравнивания к normal воды |
| `_objectWidth` | 3 | `SampleHeightHelper` minimum wavelength/filter width |
| `_forceHeightOffset` | -0.3 | Y-offset точки water drag |
| `_dragInWaterUp/Right/Forward` | 3 / 2 / 1 | directional drag coefficients |
| `_dragInWaterRotational` | 0.2 | rotational drag (ItemRigidbody перезаписывает в **0.02**) |

## Точный `FixedUpdate`

Если Rigidbody динамический и `OceanRenderer.Instance` доступен, Crest:

1. запрашивает displacement, normal и surface velocity:
   ```csharp
   _sampleHeightHelper.Init(transform.position, _objectWidth,
                            allowMultipleCallsPerFrame: true);
   _sampleHeightHelper.Sample(out displacement, out normal, out surfaceVelocity);
   ```
2. добавляет flow из `SampleFlowHelper`;
3. считает относительную скорость `relativeVelocity = rb.velocity - surfaceVelocity`;
4. определяет глубину:
   ```csharp
   depth = displacement.y + OceanRenderer.Instance.SeaLevel
           - transform.position.y + _raiseObject;
   InWater = depth > 0;
   ```
5. если `InWater`, применяет:
   ```csharp
   force = -Physics.gravity.normalized * _buoyancyCoeff * depth * depth * depth;
   rb.AddForce(force, ForceMode.Acceleration);
   ```

### Ключевое свойство: сила кубическая и mass-independent

Crest применяет `ForceMode.Acceleration`, а не `ForceMode.Force`:

```text
acceleration ∝ depth³
```

Следствия:

- масса Rigidbody **не участвует** в response;
- глубоко нырнувший маленький предмет получает резко растущую acceleration;
- `_raiseObject = 1.6` означает, что vanilla item может считаться «в воде» ещё на 1.6 Unity units выше физической поверхности;
- повторное включение этого компонента одновременно с модовой Архимедовой моделью создаёт две независимые системы подъёма и может дать water-trampoline/ejection.

## Drag и torque Crest

Пока `InWater`, Crest применяет `ForceMode.Acceleration` directional drag в точке:

```csharp
position = rb.position + _forceHeightOffset * Vector3.up;
rb.AddForceAtPosition(up      * Dot(up,      -relativeVelocity) * _dragInWaterUp, position, Acceleration);
rb.AddForceAtPosition(right   * Dot(right,   -relativeVelocity) * _dragInWaterRight, position, Acceleration);
rb.AddForceAtPosition(forward * Dot(forward, -relativeVelocity) * _dragInWaterForward, position, Acceleration);
```

Затем:

```csharp
rb.AddTorque(Cross(transform.up, waterNormal) * _boyancyTorque,
             ForceMode.Acceleration);
rb.AddTorque(-_dragInWaterRotational * rb.angularVelocity);
```

Это также не mass-aware водная динамика.

## Почему `ItemRigidbody` — особый случай

В `ItemRigidbody.Start()` ваниль создаёт компонент и пишет:

```csharp
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;
```

Но `ItemRigidbody.ToggleCollider()` безусловно вызывает:

```csharp
floater.enabled = false;
```

Следовательно, жизненный цикл floater зависит от порядка `Start`/`FixedUpdate`/модовых патчей. Мод, который заменяет плавучесть, должен не только рассчитывать свою силу, но и гарантированно удерживать vanilla floater выключенным — одной проверки `enabled` недостаточно при race/повторном включении; надёжная граница владения — Harmony prefix на `SimpleFloatingObject.FixedUpdate` для managed twin.

## `SampleHeightHelper` и готовность Crest

`SampleHeightHelper.Sample()` использует:

```csharp
OceanRenderer.Instance.CollisionProvider.Query(...)
```

Если provider отсутствует или query не succeeded, helper возвращает `false` и sea level. Существование класса/helper instance **не доказывает**, что query pipeline готова. Модовой код должен обрабатывать readiness/fallback отдельно и не выполнять неограниченные queries в горячем physics path.

## Практические выводы

1. Vanilla `SimpleFloatingObject` — это не Архимед: cubic depth acceleration, не зависящая от массы.
2. `floaterHeight=1.6` напрямую превращается в `_raiseObject`; это большой lift offset, не «осадка» предмета.
3. Никогда не запускайте одновременно Crest floater и свой `ForceMode.Force` buoyancy на одном Rigidbody.
4. Для managed item twin надёжно блокируйте Crest `FixedUpdate`, а не только меняйте `Behaviour.enabled`.
5. Для нормального mass-aware поведения нужны собственные mass, displaced volume, water-entry damping и relative-velocity силы.

## Связанные заметки

- [43 — плавучесть предметов](43-item-buoyancy-water.md)
- [48 — lifecycle высоты океана](48-ocean-height-helper-lifecycle.md)
- [63 — плавучесть и floating cargo](63-vanilla-buoyancy-floating-cargo.md)
- [70 — визуальные water splashes](70-water-splash-particle-systems.md)
