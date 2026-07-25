# 70. Брызги воды: ванильные ParticleSystem, материалы и точки переиспользования

Разбор ванильных систем брызг Sailwind v0.38: `RefsDirectory`, `BoatSplash`, `BoatSplashCrest`, `AnimSplash` и `WaveSplashZone`. Информация из декомпиляции `Assembly-CSharp.dll`; полезна модам, которым нужен splash без создания собственного неподходящего материала/шейдера.

## Главная runtime-ссылка: `RefsDirectory.seaWaterSpillParticles`

В `RefsDirectory` есть публичное поле:

```csharp
public ParticleSystem seaWaterSpillParticles;
```

Это ссылка, заданная в Unity Inspector, на уже настроенную ванильную particle system морской воды. Она используется игрой, например, при проливании воды из кружки:

```csharp
RefsDirectory.instance.seaWaterSpillParticles.transform.position = spillPos;
RefsDirectory.instance.seaWaterSpillParticles.Play();
```

### Практический вывод

Для визуального эффекта воды мод может получить **реальный скомпилированный ванильный shader/material** через:

```csharp
var ps = RefsDirectory.instance.seaWaterSpillParticles;
var material = ps.GetComponent<ParticleSystemRenderer>().sharedMaterial;
```

Не изменяйте сам singleton particle system: для своего эффекта инстанцируйте prefab/создавайте новый `ParticleSystem` и назначайте копию/`sharedMaterial` renderer-а. Иначе несколько источников брызг будут перезаписывать позицию и emission друг друга.

## `BoatSplashCrest`

`BoatSplashCrest` стоит на boat effect object и содержит:

| Поле | Роль |
|---|---|
| `boatRigidbody` | Rigidbody судна |
| `deltaThreshold` / `deltaMult` | порог и масштаб вертикального изменения |
| `minSize` / `maxSize` | диапазон размера particle system |
| `minSpeed` / `maxSpeed` | диапазон скорости частиц |
| `volumeFadeIn` / `volumeFadeOut` | сглаживание audio |
| `verticalDelta` | текущий вертикальный splash input |

В `FixedUpdate` он сравнивает текущий Y effect object с Y прошлого кадра. При превышении порога включает emission, затем задаёт `particles.startSize`, `particles.startSpeed` и audio volume.

> Это не отдельная вода и не отдельная физика: splash intensity — чисто визуальная функция вертикального движения boat effect transform и скорости Rigidbody.

## `BoatSplash`

Более старый splash-компонент измеряет:

```text
horizontalSpeed = horizontal delta position / deltaTime
verticalSpeed   = difference between local effect delta and parent boat delta / deltaTime
```

Если `verticalSplash == true`, intensity берётся из `verticalSpeed`; иначе из `horizontalSpeed`. Через `size`, `speed`, `lifetime`, `audioVolume` изменяются стартовые параметры particle system.

`UpdatePosition()` использует:

```csharp
Ocean.Singleton.GetWaterHeightAtLocation2(x, z)
```

и плавно перемещает effect object к воде. Этот direct Ocean path требует существующий/готовый `Ocean.Singleton`; его нельзя бездумно вызывать из произвольного mod physics tick.

## `AnimSplash`

`AnimSplash` — ещё один boat effect. В `LateUpdate`:

1. получает высоту через `OceanHeight.GetHeight(new SampleHeightHelper(), position)`;
2. считает изменение расстояния effect object до поверхности;
3. требует одновременно `delta > deltaThreshold` и скорость Rigidbody > `1`;
4. включает основной `ParticleSystem` и `smallSplashes`;
5. устанавливает size и start speed от vertical delta + скорости судна.

`AnimSplash` показывает важный vanilla паттерн: сначала определяют **относительное движение к поверхности**, затем только управляют visual emission. Он не должен быть источником сил плавучести.

## `WaveSplashZone`

`WaveSplashZone` проверяет, покрыла ли вода конкретную boat-zone:

```csharp
float water = OceanHeight.GetHeight(helper, transform.position) + verticalOffset;
if (water >= transform.position.y && !damage.sunk)
{
    damage.Overflow(water - transform.position.y);
    // при двух последовательных wet frame включает particle emission
}
```

То есть water splash zone может участвовать не только в визуале, но и в boat flooding (`BoatDamage.Overflow`). Не добавляйте такой компонент к предметам: он семантически рассчитан на судно.

## Рецепт для моддера

1. Для надежного внешнего вида сначала попробуйте material от `RefsDirectory.seaWaterSpillParticles`.
2. Создайте отдельный world-space `ParticleSystem`; не двигайте singleton spill effect.
3. Intensity считайте из **relative velocity body ↔ surface**, а не просто из world Y velocity.
4. Визуальный splash не должен менять `Rigidbody.velocity`, позицию или `isKinematic`.
5. Если нужен custom shader, он должен быть заранее скомпилирован Unity и загружен из AssetBundle; ShaderLab исходник нельзя превратить в рабочий Unity shader обычным C# runtime-кодом.

## Связанные заметки

- [14 — физика корпуса и плавучесть](14-boat-physics-buoyancy-damage.md)
- [31 — Crest ocean и волны](31-ocean-waves-inertia.md)
- [43 — плавучесть предметов](43-item-buoyancy-water.md)
- [48 — lifecycle высоты океана](48-ocean-height-helper-lifecycle.md)
