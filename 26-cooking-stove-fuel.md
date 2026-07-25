# 26. Кулинария: плита, топливо, варка и копчение

Разбор подсистемы приготовления пищи. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Порча/консервация еды (сушка/засолка) описаны в заметке 23 (`FoodState`); здесь — термическая обработка.

## Цепочка приготовления

```
StoveFuel (топливо, health=200)
   └─ горит: health -= dt*timescale*100; stove.AddHeat(burnRate)
        └─ ShipItemStove.AddHeat → каждый слот: CookableFood.Cook(heat*cookEfficiency, smoker)
             └─ item.amount (прожарка) и currentHeat растут; smoker → foodState.AddSmoked
```

## Плита (`ShipItemStove : ShipItem`)

| Поле | По умолч. | Содержание |
|------|:--:|-----------|
| `cookEfficiency` | 0.4 | КПД плиты (доля тепла, идущая в готовку). |
| `smoker` | false | Режим коптильни (добавляет копчение, не варит суп). |
| `slots` (`StoveCookTrigger[]`) | — | Слоты для еды (несколько блюд одновременно). |
| `currentHeat` | 0 | Текущий нагрев плиты. |
| `fuelTrigger` | — | Приёмник топлива. |

### Взаимодействие (`OnItemClick` по плите)
- Если в руках **топливо** (`StoveFuel`) → заправка в `fuelTrigger`.
- Если в руках **еда** (`CookableFood`) → положить в свободный слот (`GetFreeSlot`). **Коптильня не принимает суп** (`CookableFoodSoup`).

### Нагрев (`AddHeat`)
Для каждого занятого слота: `currentFood.Cook(heat * cookEfficiency, smoker)`. Звук шкварчания (`audio.volume`) растёт с числом готовящихся блюд (до `0.777 * count`).

## Топливо (`StoveFuel`)

| Поле | По умолч. | Содержание |
|------|:--:|-----------|
| `initialFuel` | 200 | Запас топлива → `item.health`. |
| `lit` | false | Горит ли. |
| `inserted` | — | Вставлено ли в плиту. |

- **Вставка:** в `StoveFuelTrigger` (по триггеру или клику).
- **Горение** (`LightUp` → `lit = true`): пока `health > 0`:
  - `health -= deltaTime * Sun.sun.timescale * 100` (расход топлива, зависит от ускорения времени);
  - `cookTrigger.stove.AddHeat(этот же объём)` — передача тепла плите.
- Когда топливо кончилось (`health <= 0`) → `UnregisterBurntFuel()` + `DestroyItem()` (топливо исчезает).

> Топливо — расходуемый предмет: 200 единиц здоровья сгорают со скоростью `100 * timescale` в секунду.

## Готовка еды (`CookableFood`)

| Поле | Содержание |
|------|-----------|
| `currentHeat` (0..2) | Нагрев самого блюда. |
| `cookRate` (1) | Скорость остывания. |
| Материалы | `rawMaterial`, `cookedMaterial`, `burntMaterial`, `spoiledMaterial`, `driedMaterial`. |

### `Cook(addedHeat, smoking)`
```
energyPerBite = itemFood.GetEnergyPerBite()
num = addedHeat * 0.066 / energyPerBite
item.amount  += num * currentHeat * 0.5     // «прожарка» (кап 2.2)
currentHeat  += num * 10                     // нагрев (кап 2)
if smoking → foodState.AddSmoked(num * currentHeat * 0.5)
```
Плотная/калорийная еда (высокий `energyPerBite`) готовится медленнее.

### `item.amount` = степень готовности (визуал и статус)
| `amount` | Состояние | Материал |
|:--:|-----------|----------|
| `< 0.75` | сырое (`raw`) | `rawMaterial` |
| `0.75–1.0` | переход сырое→готовое | lerp raw→cooked |
| `1.0–1.5` | **готовое** (`cooked`) | `cookedMaterial` |
| `1.5–1.75` | переход готовое→горелое | lerp cooked→burnt |
| `>= 1.75` | **сгоревшее** (`burnt`) | `burntMaterial` |
| `dried >= 0.99` | сушёное | `driedMaterial` |
| `spoiled >= 0.9` | испорченное | lerp → `spoiledMaterial` |

Это согласуется с префиксами `FoodState` (заметка 23): `"cooked"` при `amount>=1`, `"burnt"` при `amount>=1.5`.

### Остывание и частицы
- Вне плиты: `currentHeat -= 50 * dt * cookRate * 5e-5` (медленно остывает).
- **Пар** при `currentHeat >= 0.5`: эмиссия lerp 12→24, ветер сносит частицы (`Wind.currentWind * 0.015`).
- **Зелёные частицы гнили** при `spoiled > 0.9` (цвет RGB ≈ 0.27/0.42/0.27).

### Покупная готовка
- `CookInShop()`: `currentHeat = 1.25`, `amount = 1.2` (сразу готовое).
- `SmokeInShop()`: `amount = 1.01` (копчёное).

## Триггеры
- **`StoveCookTrigger`** (слот еды): `smoking`, `stove`, `currentFood`. `OnTriggerEnter` → если слот свободен и плита не в руках → `CookableFood.InsertIntoCookTrigger`.
- **`StoveFuelTrigger`** (приёмник топлива): `InsertFuel(item)`, `UnregisterBurntFuel()`.
- `InsertIntoCookTrigger`: еда крепится к слоту (`attached`, `inStove`, `disableCol`), позиция фиксируется на плите (`UpdatePosition`). `TakeOutOfCooker` — снятие.

## Практические выводы для мододела

1. **Готовка** = топливо (`StoveFuel.health`) → тепло плиты (`AddHeat`) → `CookableFood.Cook`. КПД плиты `cookEfficiency = 0.4`.
2. **Степень готовности** хранится в `ShipItem.amount`: `<0.75` сырое, `1–1.5` готовое, `>1.75` сгоревшее. Меняя `amount`, можно мгновенно «сварить»/«сжечь» еду.
3. **Копчение** — через `smoker`-плиту (`AddSmoked` в `FoodState`); коптильня не варит суп.
4. **Топливо расходуется** со скоростью `100 * timescale`/с из 200 единиц; ускорение времени ускоряет и расход.
5. **Визуал еды** (материалы raw/cooked/burnt/dried/spoiled) и частицы (пар/гниль) управляются `amount`/`currentHeat`/`spoiled`.
6. Состояние готовки (`amount`, `currentHeat` через `health`/`amount`) сохраняется в сейв через `SavePrefabData` (заметка 11); порча/сушка/копчение — через `extraValue0..3`.
