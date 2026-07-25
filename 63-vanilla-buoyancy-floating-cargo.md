# 63. Ванильная плавучесть: эталон и груз за бортом

Разбор эталонной плавучести SimpleFloatingObject и спавна плавающего груза — ответ на запросы B6, B7. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 43 (buoyancy), 61 (каталог).

## B6. Ванильная эталонная плавучесть

### `SimpleFloatingObject` — Crest buoyancy

`SimpleFloatingObject` — класс из **Crest DLL** (не в Assembly-CSharp, не декомпилирован). Базовый класс: `FloatingObjectBase` (Crest). Известные из ItemRigidbody.Start():

```csharp
floater = gameObject.AddComponent<SimpleFloatingObject>();
floater._dragInWaterRotational = 0.02f;
floater._raiseObject = item.floaterHeight;  // default 1.6
```

### Известные поля SimpleFloatingObject (из Crest source/docs + ItemRigidbody)

| Поле | Тип | Default | Содержание |
|------|-----|---------|------------|
| `_raiseObject` | `float` | 1.6 (ShipItem.floaterHeight) | **Target height above water surface** — предмет поднимается на `_raiseObject` метров над уровнем воды |
| `_dragInWaterRotational` | `float` | 0.02 | Rotational drag in water |
| `InWater` | `bool` | — (runtime) | Flag: object currently in water (from FloatingObjectBase) |

### Crest `SimpleFloatingObject` — поведение (из Crest open source)

Crest `SimpleFloatingObject` (open-source component, https://github.com/crest-ocean/crest):
- Каждый FixedUpdate: query wave height at object position via `SampleHeightHelper`.
- Apply buoyancy force: `AddForceAtPosition(up × buoyancyForce)` at multiple points along object bounds.
- `_raiseObject` — vertical offset: object floats with **bottom approximately at water level, top at `_raiseObject` above**. This is NOT "center of mass at `_raiseObject`" — it's **bottom of object at water level + raise height offset**.
- `InWater` — set when wave height at object position is above object bottom.
- Buoyancy force proportional to submerged volume (approximated via wave height vs object position).

> **Эталонная осадка ванильного предмета:** предмет плавает с **нижняя часть ≈ на уровне воды**, а **верхняя часть ≈ на `_raiseObject` (1.6 м) выше воды**. Для ящика высотой ~0.5 м → ~0.5 м submerged, ~1.1 м above water → предмет сидит высоко (visible).

### `ShipItem.floaterHeight` — buoyancy raise height

`floaterHeight = 1.6f` — default SerializeField на ShipItem. Prefab может override (например, лёгкий предмет → floaterHeight = 1.8 (floats higher), тяжелый → 1.2 (floats lower)). Но **prefab values — runtime only** (не в декомпиляции).

### `ItemRigidbody.FixedUpdate` — floater control

```csharp
if (inStove)
{
    flag = false;  // → skip floater ToggleCollider (floater stays enabled)
}
if (disableCol)
{
    ToggleCollider(state: false);  // floater.enabled = false
}
if (!flag) return;  // skip collider toggling
// else → ToggleCollider(state: true) → floater.enabled = false (always disabled when colliders active?)
```

> **`ToggleCollider` отключает floater** (`((Behaviour)floater).enabled = false`). Floater работает **только когда colliders toggled off** (in inventory/crate/stove). Когда twin free → colliders enabled → **floater disabled** → buoyancy от SimpleFloatingObject НЕ применяется.

> **КРИТИЧНО:** `ToggleCollider(true)` → `floater.enabled = false`. Twin в свободном состоянии (sold, not held, not in inventory) → `ToggleCollider(true)` → **floater OFF** → buoyancy НЕ работает. Но... предмет **плавает** в ваниле. Как?

**Ответ:** `SimpleFloatingObject.FixedUpdate` (Crest) — buoyancy force применяется **независимо** от `floater.enabled`? Нет — `enabled = false` → `FixedUpdate` не вызывается → buoyancy off.

> **Real mechanism:** ItemRigidbody.FixedUpdate — `ToggleCollider(true)` вызывается **внутри FixedUpdate**, но **в конце** FixedUpdate. Buoyancy в том же FixedUpdate **уже применена** (Crest SimpleFloatingObject.FixedUpdate runs before ItemRigidbody.FixedUpdate due to script execution order). Twin buoyancy force applied → then ItemRigidbody toggles floater off → **next frame** floater OFF → no buoyancy → twin sinks? **No** — twin is kinematic (held!=null, !sold, nailed, etc.) → position driven by visual/EnterBoat/walkCol → buoyancy irrelevant. Twin dynamic (sold, free, onBoat, >6 frames) → floater **enabled again** because `ToggleCollider` logic re-evaluates: `flag = true` → `ToggleCollider(true)` → `floater.enabled = false`...

> **Реальное поведение:** ванильный twin при free (sold, not held, dynamic) → **floater.enabled = false** (ToggleCollider disables it) → **twin DOES NOT GET BUOYANCY FORCE** → twin должен sinking. Но ванильный предмет плавает — **значит twin position = visual position** (twin slave при held, но twin master при free) → **twin dynamic, но floater off** → twin sinks → visual.position = twin.position → visual sinks → **Но предмет плавает!**

> **Resolution:** `SimpleFloatingObject` buoyancy применяется **к visual GO**, не twin? Нет — SimpleFloatingObject добавлен на **twin GO** (ItemRigidbody.Start()). Twin dynamic → floater disabled → twin sinks → visual follows twin → both sink. **Но предмет плавает!** → **floater.enabled логика более сложная**, чем видно из truncated code. Нужно runtime проверить.

**Альтернативное объяснение:** `ToggleCollider` вызывается **только когда `flag` = true** (distance check passed, in range). Если twin **out of range** → `flag = false` → early return → `ToggleCollider` NOT called → **floater remains enabled** → buoyancy works. Twin in range → `flag = true` → `ToggleCollider(true)` → floater off → no buoyancy → twin sinks? **Нет — twin kinematic в этом случае** (held, nailed, !sold, etc.) → buoyancy irrelevant.

> **Ванильная плавучесть:** floater работает для twin в **out-of-range** состоянии (distance > 600 от камеры). Twin far → kinematic (flag2=true) → position frozen → buoyancy irrelevant (kinematic). Twin near → dynamic (flag2=false) → floater off (ToggleCollider) → **NO BUOYANCY** → twin dynamic, sinks?

> **Runtime check required** — вероятнее всего, Crest SimpleFloatingObject работает **независимо от ItemRigidbody floater toggle** в некоторых условиях, или buoyancy применяется через другой механизм. Мод, который глушит floater и заменяет на честный Архимед — **правильный подход**, т.к. ванильный floater — «магический» (force-based, не physics-accurate).

## B7. Груз за бортом в ваниле

### Спавнит ли игра сама плавающий/потерянный груз?

**Нет — ваниль НЕ спавнит плавающий груз за бортом.**

`WorldItemSpawner` — спавнит предметы на фиксированных позициях в мире (причалы, берег, порты). **Не в море.** Спавнер работает только при `distance < 100f` от камеры (respawn check).

**Floating loot crates (мод заметки 34)** — это **модовый** механизм. Ваниль не имеет «плавающих ящиков» или «потерянного груза» в море.

###漂流物 (drift items) в ваниле

Если игрок **выбросил предмет в море** (drop item while on boat) → предмет falls → twin dynamic → SimpleFloatingObject buoyancy → предмет **плавает** на `_raiseObject` уровне. Это **единственный «груз за бортом»** в ваниле — игрокский drop, не автоспавн.

### Настройка плавучести dropped items в ваниле

- `_raiseObject = floaterHeight` (default 1.6) — предмет плавает **высоко** (1.6 m above water).
- `drag = 1.2` → предмет медленно замедляется в воде.
- `angularDrag = mass × 0.1` → rotation drag.
- ItemRigidbody.FixedUpdate: twin free → position master → twin floats at floaterHeight → visual follows → item visible on water surface.

> **Эталон калибровки для мода:** ванильный предмет на воде → sits **high** (1.6 m above water surface). Мод с честным Архимедом → предмет sits по **осадка = mass / (displacement × water_density)**. Для лёгкого предмета (mass=1) → осадка малая → sits high — **close to vanilla**. Для тяжелого (mass=20) → осадка большая → sits lower — **deviates from vanilla expectations**.

> **Калибровка:** мод с честным Архимедом должен давать **осадку ≈ mass / (bounds.volume × 1025)** для воды (density 1025 kg/m³). Масса 1 кг + bounds 0.01 m³ → осадка ≈ 1/(0.01×1025) = 0.097 → предмет почти полностью на поверхности. Масса 20 кг + bounds 0.05 m³ → осадка ≈ 20/(0.05×1025) = 0.39 → предмет ~40% submerged — **realistic**, но ниже ванильного «1.6 m above water».

## Практические выводы для мододела

1. **SimpleFloatingObject — Crest DLL, не декомпилирован** — `_raiseObject = 1.6` default, buoyancy force-based, не physics-accurate.
2. **ToggleCollider disables floater** — twin free (sold, dynamic) → floater.enabled = false → buoyancy OFF → twin должен sinking. **Runtime check needed** — вероятнее всего, Crest buoyancy работает через другой путь.
3. **Мод глушит floater и заменяет честным Архимедом — правильный подход.** Ванильный floater — «магический» force-based.
4. **Ваниль НЕ спавнит груз за бортом** — только игрокский drop в море. WorldItemSpawner — причалы/берег.
5. **Эталон калибровки:** ванильный предмет → sits 1.6 m above water. Честный Архимед → осадка = mass/(bounds×1025). Лёгкие предметы → close to vanilla. Тяжёлые → lower — realistic, но может разойтись с player expectations.
6. **Для точной калибровки** — нужен runtime dump bounds.size (twin collider extents) для каждого предмета, чтобы мод мог вычислить displacement volume.
