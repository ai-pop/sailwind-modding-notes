# 09. Дополнительно: Рыбалка, Квесты, Части Лодок

> Детали рыбной системы, квестовых предметов, кастомизации лодок и стартовых наборов.
> Дополняет заметки 23 (Рыбалка), 27 (Квесты), 22 (Верфь).

---

## Рыбная система

### OceanFishes

Глобальный синглтон `OceanFishes.instance`.

| Поле | Тип | Описание |
|-------|------|-------------|
| `fishPrefabs[]` | GameObject[] | Префабы рыб по широте |
| `peakLatitude[]` | float[] | Оптимальная широта для каждой рыбы |
| `deviationDistance` | float | Случайное отклонение |
| `localFishesRegions[]` | LocalFishesRegion[] | Региональные переопределения |

**Логика выбора рыбы:**
```
GetFish(pos):
  1. Проверить localFishesRegions — если в радиусе, шанс вернуть локальную рыбу
  2. Иначе: широта + случайное отклонение
  3. Найти рыбу с ближайшей peakLatitude
  4. Вернуть этот префаб
```

### LocalFishesRegion

| Поле | Тип | Умолч. | Описание |
|-------|------|:-----:|-------------|
| `outerRadius` | float | 5000 | Внешний радиус |
| `innerRadius` | float | 2500 | Внутренний радиус |
| `overrideInfluence` | float | 0.75 | Шанс переопределения |
| `localFishPrefabs[]` | GameObject[] | — | Пул локальных рыб |

### Механика FishingRodFish

| Поле | Тип | Умолч. |
|-------|------|:-----:|
| `fishPullForce` | float | 1.0 |
| `pullTensionMult` | float | 1.0 |
| `fishTimer` | float | 6.0 сек |
| `fishEnergy` | float | 1.0 |

**Процесс:**
1. Крючок в воде + длина лески > 1 → таймер
2. Таймер истекает → `CatchFish()`: запрос рыбы из OceanFishes
3. Борьба: tension через `currentTargetTension`
4. Tension > 0.95 → таймер разрыва (3.1 сек макс.)
5. Сбор → `CollectFish()`: спавн ShipItem, `sold=true`, 30% шанс потери крючка

---

## Квестовые предметы

`QuestItem` — MonoBehaviour с авто-пометкой `sold = true`:

| Индекс | Название |
|:-----:|------|
| 330 | quest0 letter |
| 331 | quest0 cargo |

---

## Кастомизация лодок

### BoatPart / BoatPartOption

| Класс | Назначение |
|-------|---------|
| `BoatPart` | Слот части (нос, корма...) |
| `BoatPartOption` | Вариант части |
| `BoatPartsOrder` | Полный заказ опций |

### BoatPartOption

| Свойство | Описание |
|----------|-------------|
| `optionName` | Имя опции |
| `requires[]` | Требуемые опции |
| `requiresDisabled[]` | Требуемые отключённые опции |
| `childMast` | Дочерняя мачта |
| `walkColObject` | Walk collider |

### SaveSailData (сохранение паруса)

| Поле | Описание |
|-------|-------------|
| `prefabIndex` | Индекс паруса |
| `mastIndex` | Индекс мачты |
| `installHeight` | Высота на мачте |
| `minAngle`/`maxAngle` | Углы |
| `health` | Состояние (100=новый) |
| `sailColor` | Индекс цвета |
| `scaleY`/`scaleZ` | Масштаб |

---

## Стартовые наборы (StarterSet)

| Поле | Тип | Описание |
|-------|------|-------------|
| `region` | PortRegion | Регион набора |
| `starterBoat` | Transform | Начальная лодка |

**Поведение:**
- При `GameState.justStarted` + совпадение региона:
  - Активирует дочерние предметы
  - Смещает вниз на 15 ед., помечает `sold = true`, регистрирует в сохранение
- При несовпадении: уничтожает дочерние предметы

---

## Прочие неклассифицированные классы

| Класс | Вероятное назначение |
|-------|---------------------|
| `KiciaAltar` | Алтарь/святилище |
| `ShroomTrigger` | Триггер грибов |
| `OnsenExitTrigger`/`OnsenMusicTrigger` | Горячий источник |
| `Balloon` | Декоративный шар? |
| `Tavern` | Таверна |
| `TavernRumorsDude` | NPC со слухами |

---

*Извлечено из Sailwind v0.38.*
