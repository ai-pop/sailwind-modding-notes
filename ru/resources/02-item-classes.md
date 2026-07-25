# 02. Иерархия Классов ShipItem

> Полный справочник всех подклассов `ShipItem`, их C# полей, поведения и взаимодействий.
> Расширяет заметку 16 (Система Предметов).

---

## Цепочка наследования

```
MonoBehaviour
  └─ GoPointerButton          (интерактивный объект — взгляд, активация, альт-активация)
       └─ PickupableItem       (механика подбора/бросания/удержания)
            └─ ShipItem        (базовый предмет — масса, цена, физика, сохранение)
                 └─ ShipItemHangable  (подвешиваемые — фонари и т.д.)
                      └─ ShipItemLight  (источники света)
                 └─ 30+ прочих подклассов
```

---

## Базовый класс: `ShipItem`

### Сериализованные поля (задаются в инспекторе Unity)

| Поле | Тип | Умолч. | Описание |
|-------|------|:-----:|-------------|
| `wallAttachment` | bool | false | Можно прибить к стене через raycast |
| `delayLook` | bool | false | Задержка обновления текста взгляда |
| `mass` | float | 1.0 | Масса в кг (влияет на BoatMass) |
| `value` | int | — | Базовая цена для экономики |
| `name` | string | — | Отображаемое имя |
| `category` | TransactionCategory | — | Категория транзакции |
| `inventoryScale` | float | 1.0 | Масштаб в слоте инвентаря |
| `inventoryRotation` | float | 0 | Поворот Y в инвентаре |
| `inventoryRotationX` | float | 0 | Поворот X в инвентаре |
| `floaterHeight` | float | 1.6 | Высота подъёма над водой (плавучесть) |
| `itemRigidbody` | Transform | — | Физический двойник (сериализованная ссылка) |

### Рантайм-состояние

| Поле | Тип | Описание |
|-------|------|-------------|
| `sold` | bool | Куплен (false=на полке) |
| `nailed` | bool | Прибит/закреплён |
| `health` | float | Прочность / объём жидкости / укусы еды |
| `amount` | float | Количество / стак / уровень заполнения |
| `daysInStorage` | int | Дней хранения (влияет на порчу) |
| `itemRigidbodyC` | ItemRigidbody | Кешированный физический двойник |
| `currentWalkCol` | Transform | Текущий walk collider лодки |
| `currentActualBoat` | Transform | Текущий родитель-лодка |

### Ключевые виртуальные методы

| Метод | Когда вызывается | Назначение |
|--------|-------------|------------------|
| `OnLoad()` | После Awake + 1 кадр | Инициализация из данных сохранения |
| `OnBuy()` | После покупки | Пост-покупочная настройка |
| `OnPickup()` | Взят игроком | Очистка креплений, инвентаря |
| `OnDrop()` | Брошен игроком | Крепление к стене, возврат в магазин |
| `OnEnterInventory()` | Помещён в слот инвентаря | Отсоединение джойнтов, выход с лодки |
| `OnLeaveInventory()` | Извлечён из инвентаря | Перевключение физики |
| `UpdateLookText()` | Каждый кадр при взгляде | Установка `lookText`/`description` |
| `OnAltActivate()` | ПКМ / Alt | Специальное действие |
| `OnScroll(float)` | Колёсико мыши | Поворот предмета, зум подзорной трубы |
| `ExtraFixedUpdate()` | FixedUpdate | Физика предмета |
| `ExtraLateUpdate()` | LateUpdate | Визуальные обновления |
| `AllowOnItemClick()` | Проверка взаимодействия | Предмет→предмет |
| `OnItemClick()` | Клик с удерживаемым предметом | Взаимодействие предмет→предмет |

---

## Справочник подклассов

### Еда и напитки

#### `ShipItemFood` — Съедобная еда
| Поле | Тип | Описание |
|-------|------|-------------|
| `eatenMeshes[]` | Mesh[] | Меши стадий съедания (3 состояния) |
| `energyPerBite` | float | Энергия за укус (умолч. 10) |
| `rawEnergyMult` | float | Множитель сырой еды (умолч. 0.25) |
| `protein` | float | Содержание белка |
| `vitamins` | float | Содержание витаминов |

**Поведение:** Alt-активация при удержании — съесть один укус. Каждый укус уменьшает `health` на 1 (всего 3 укуса). Состояния cooked/raw/salted/smoked/dried влияют на энергию. Использует `FoodState` для порчи/консервации. Можно посолить кликом `ShipItemSalt`.

#### `ShipItemSoup` — Суп/котёл
| Поле | Тип | Описание |
|-------|------|-------------|
| `capacity` | float | Макс. объём воды (умолч. 20) |
| `currentWater` | float | Текущий объём воды |
| `currentEnergy` | float | Готовая энергия |
| `currentUncookedEnergy` | float | Оставшаяся сырая энергия |
| `currentSpoiled` | float | Количество порчи |
| `currentVitamins` | float | Витамины |
| `currentProtein` | float | Белок |
| `currentSalted` | float | Уровень соли |

**Сохранение:** Использует `extraValue0–4` для состояния вне health/amount.

#### `ShipItemKettle` — Чайник
| Поле | Тип | Описание |
|-------|------|-------------|
| `capacity` | float | Макс. воды (умолч. 10) |
| `brewSpeed` | float | Скорость заваривания |
| `currentWater` | float | Объём воды |
| `currentTeaAmount` | float | Добавлено чайных листьев |
| `currentCookedTeaAmount` | float | Крепость заваренного чая |
| `currentTeaType` | LiquidType | Тип чая |

**Поведение:** Принимает клики `ShipItemTea` для добавления чая. Готовка заваривает чай. Питьё восстанавливает гидратацию + энергию в зависимости от типа чая.

#### `ShipItemBottle` — Ёмкость для жидкости
| Поле | Тип | Описание |
|-------|------|-------------|
| `capacity` | float | Макс. объём (9=ведро, <5=кружка, <30=бутылка, ≥30=бочка) |

**Поведение:** `health` = уровень заполнения. `amount` = индекс типа жидкости. ПКМ чтобы пить. Можно переливать между bottle/soup/kettle. Название: `capacity==9` → «bucket», `<5` → «mug», `<30` → «bottle», `≥30` → «barrel».

#### `ShipItemTea` — Чайные листья
| Поле | Тип | Описание |
|-------|------|-------------|
| `teaType` | LiquidType | Тип чая |

**Поведение:** Клик на чайник для добавления. `amount * 0.1` добавляется к массе.

#### `ShipItemSalt` — Соль
| Поле | Тип | Описание |
|-------|------|-------------|
| `maxSalt` | float | Макс. количество соли (умолч. 100) |

**Поведение:** Клик на еду чтобы посолить. `amount / maxSalt * 100` = % соли. Предотвращает порчу.

---

### Готовка

#### `ShipItemStove` — Печь
Хранит топливо. Имеет позиции готовки (`StoveCookTrigger`).

#### `ShipItemStoveFuel` — Топливо для печи
Время горения зависит от `health`.

#### `ShipItemKnife` — Нож для нарезки
Нарезает еду в нарезанные варианты. `KnifeCollider` обрабатывает разрезание.

---

### Инструменты

#### `ShipItemHammer` — Молоток
Ремонт корпуса (прибивание досок) и общее строительство.

#### `ShipItemBroom` — Метла
| Поле | Тип | Описание |
|-------|------|-------------|
| `cleaner` | Cleaner | Ссылка на компонент уборки |

**Поведение:** Alt-активация подметает. `cleaner.activated = true`. Отключает красный контур при подметании.

#### `ShipItemOakum` — Пакля (конопатка)
| Поле | Тип | Описание |
|-------|------|-------------|
| `maxAmount` | float | Макс. количество пакли |

**Поведение:** Alt-активация у повреждённой лодки. `amount -= num`, `boatDamage.oakum += num`.

#### `ShipItemOar` — Весло
Ручное движение. Без специальных полей в декомпиляции.

---

### Навигация

#### `ShipItemCompass` — Компас (включая солнечный)
| Поле | Тип | Описание |
|-------|------|-------------|
| `sunCompassSundial` | Transform | Гномон (вариант солнечного компаса) |
| `chronoLatitude` | ChronometerLatitude | Хронометр широты |
| `lockX/Y/Z` | bool | Блокировка осей вращения |
| `sharpenShadow` | bool | Режим резкой тени |

#### `ShipItemClock` — Часы/хронометр
| Поле | Тип | Описание |
|-------|------|-------------|
| `minuteHand` | Transform | Минутная стрелка |
| `hourHand` | Transform | Часовая стрелка |
| `lid` | Transform | Крышка (опционально) |

**Поведение:** Стрелки вращаются по `Sun.sun.globalTime`. Alt-активация открывает/закрывает крышку.

#### `ShipItemQuadrant` — Квадрант
| Поле | Тип | Описание |
|-------|------|-------------|
| `dial` | Transform | Циферблат угла |
| `rotatingParent` | Transform | Вращающаяся часть |
| `lockX/Y/Z` | bool | Блокировка осей |

#### `ShipItemSpyglass` — Подзорная труба
| Поле | Тип | Описание |
|-------|------|-------------|
| `currentZoom` | float | Текущий зум |
| `movingParts[]` | Transform[] | Выдвижные части |
| `cam` | Camera | Камера зума |
| `minZoom` / `maxZoom` | float | Диапазон зума |

#### `ShipItemChipLog` — Чип-лаг (измерение скорости)
| Поле | Тип | Описание |
|-------|------|-------------|
| `bobberJoint` | ConfigurableJoint | Соединение поплавка |
| `maxLength` | float | Макс. длина верёвки (умолч. 40) |
| `bobberForceMult` | float | Множитель силы поплавка |

---

### Освещение

#### `ShipItemLight : ShipItemHangable` — Фонарь
| Поле | Тип | Описание |
|-------|------|-------------|
| `on` | bool | Состояние света |
| `usesOil` | bool | Использует масло (а не свечу) |
| `fuelConsumptionRate` | float | Скорость расхода топлива |
| `light` | Light | Unity Light |
| `particles` | ParticleSystem | Частицы пламени |

**Поведение:** `amount >= 1` → свет включён. Топливо из `ShipItemLanternFuel`. Подвешивается на `ShipItemLampHook`.

#### `ShipItemLanternFuel` — Топливо для фонаря
| Поле | Тип | Описание |
|-------|------|-------------|
| `oilBottle` | bool | Бутылка масла (а не свеча) |
| `initialHealth` | float | Начальное количество топлива |

#### `ShipItemLampHook` — Крюк для фонаря
| Поле | Тип | Описание |
|-------|------|-------------|
| `occupied` | bool | Занят фонарём |

#### `ShipItemHangable` — Подвешиваемый
Базовый класс для предметов на крюках.

---

### Отдых и привычки

#### `ShipItemBed` — Кровать
| Поле | Тип | Описание |
|-------|------|-------------|
| `sleepPos` | Transform | Позиция сна |
| `wakePos` | Transform | Позиция пробуждения |

#### `ShipItemPipe` — Трубка
Используется с `ShipItemTobacco`.

#### `ShipItemTobacco` — Табак
Разновидности: white/green/black/brown/blue.

#### `ShipItemElixir` — Эликсир
| Поле | Тип | Описание |
|-------|------|-------------|
| `addedSleep` | float | Восстановление сна |
| `addedWater` | float | Восстановление жажды |
| `addedFood` | float | Восстановление голода |

#### `ShipItemRandomElixir` — Случайный эликсир
Случайные эффекты.

---

### Хранение и грузы

#### `ShipItemCrate` — Ящик
| Поле | Тип | Описание |
|-------|------|-------------|
| `containedPrefab` | GameObject | Префаб содержимого |
| `smokedFood` | bool | Содержимое предварительно копчёное |

#### `ShipItemFoldable` — Складываемое (карты)
| Поле | Тип | Описание |
|-------|------|-------------|
| `allowCharting` | bool | Можно чертить карты |
| `foldedMesh` / `unfoldedMesh` | Mesh | Меши свёрнутого/развёрнутого |
| `mapChart` | MapChart | Компонент черчения |

---

### Рыбалка

#### `ShipItemFishingRod` — Удочка
Держит крючок. `amount` = прикреплён ли крючок.

#### `ShipItemFishingHook` — Крючок
Клик на удочку для прикрепления.

---

### Разное

#### `ShipItemScroll` — Свиток
| Поле | Тип | Описание |
|-------|------|-------------|
| `directory` | ScrollDirectory | Каталог содержимого свитков |
| `page` | Renderer | Рендерер страницы |

**Поведение:** `amount` = индекс типа свитка (туториал ≤2, прочие >2). Туториалы: value=30, прочие: value=120.

#### `ShipItemTotem` — Тотем погоды
| Поле | Тип | Описание |
|-------|------|-------------|
| `castParticles` | ParticleSystem | Эффект каста |
| `stormAttraction` | float | Сила притяжения шторма |

**Поведение:** Alt-удержание для каста. После: `health = -1`.

#### `ShipItemInkSet` — Чернила
Клик на `ShipItemFoldable` (с `allowCharting`) для черчения карт.

---

## Полный список подклассов (35+)

| # | Класс | Базовый | Категория |
|:--|-------|------|----------|
| 1 | `ShipItem` | PickupableItem | база |
| 2 | `ShipItemBed` | ShipItem | мебель |
| 3 | `ShipItemBottle` | ShipItem | напитки |
| 4 | `ShipItemBroom` | ShipItem | инструмент |
| 5 | `ShipItemChipLog` | ShipItem | навигация |
| 6 | `ShipItemClock` | ShipItem | навигация |
| 7 | `ShipItemCompass` | ShipItem | навигация |
| 8 | `ShipItemCrate` | ShipItem | груз |
| 9 | `ShipItemElixir` | ShipItem | расходник |
| 10 | `ShipItemFishingHook` | ShipItem | рыбалка |
| 11 | `ShipItemFishingRod` | ShipItem | рыбалка |
| 12 | `ShipItemFoldable` | ShipItem | разное |
| 13 | `ShipItemFood` | ShipItem | еда |
| 14 | `ShipItemHammer` | ShipItem | инструмент |
| 15 | `ShipItemHangable` | ShipItem | база-подвес |
| 16 | `ShipItemInkSet` | ShipItem | инструмент |
| 17 | `ShipItemKettle` | ShipItem | готовка |
| 18 | `ShipItemKnife` | ShipItem | инструмент |
| 19 | `ShipItemLampHook` | ShipItem | освещение |
| 20 | `ShipItemLanternFuel` | ShipItem | топливо |
| 21 | `ShipItemLight` | ShipItemHangable | освещение |
| 22 | `ShipItemOakum` | ShipItem | инструмент |
| 23 | `ShipItemOar` | ShipItem | инструмент |
| 24 | `ShipItemPipe` | ShipItem | расходник |
| 25 | `ShipItemQuadrant` | ShipItem | навигация |
| 26 | `ShipItemRandomElixir` | ShipItem | расходник |
| 27 | `ShipItemSalt` | ShipItem | готовка |
| 28 | `ShipItemScroll` | ShipItem | разное |
| 29 | `ShipItemSoup` | ShipItem | еда |
| 30 | `ShipItemSpyglass` | ShipItem | навигация |
| 31 | `ShipItemStove` | ShipItem | готовка |
| 32 | `ShipItemStoveFuel` | ShipItem | топливо |
| 33 | `ShipItemTea` | ShipItem | напитки |
| 34 | `ShipItemTobacco` | ShipItem | расходник |
| 35 | `ShipItemTotem` | ShipItem | магия |

---

*Извлечено из Assembly-CSharp.dll (Sailwind v0.38).*
