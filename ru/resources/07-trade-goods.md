# 07. Торговые Товары и Грузы

> Полный каталог всех торговых товаров (компонент Good), грузов и их регионального происхождения.
> Дополняет заметки 13 (Экономика), 15 (Миссии), 45 (Ящики/Грузы).

---

## Система Good

Компонент `Good` на торгуемых ShipItem:

| Поле | Тип | Описание |
|-------|------|-------------|
| `nativeRegion` | PortRegion | Регион происхождения |
| `requiredRepLevel` | int | Мин. репутация для покупки |
| `sizeDescription` | string | Описание размера |
| `missionIndex` | int | Назначенная миссия (-1 если нет) |

**Всего товаров:** 65 (`Refs.goodCount = 65`)

### Маппинг индексов
```
Good 0–30  → Предметы 0–30   (общие индексы)
Good 31–64 → Предметы 201–234 (смещение +170)
```

---

## Товары 0–30: Потребительские

### Еда и рыба (ящики)

| Good | Предм. | Название | Регион |
|:----:|:----:|------|:------:|
| 0 | 0 | salmon | Emerald |
| 1 | 1 | crate salmon | Emerald |
| 2 | 2 | crate dates | — |
| 3 | 3 | crate coconuts | — |
| 4 | 4 | crate lamb | — |
| 7 | 7 | crate cheese | — |
| 8 | 8 | crate goat cheese | — |
| 9 | 9 | crate sunspot fish | Aestrin |
| 14 | 14 | crate north fish | Medi |
| 15 | 15 | crate sausages | — |
| 16 | 16 | crate pork | — |
| 17 | 17 | crate bananas | — |
| 18 | 18 | crate trout | Medi |
| 19 | 19 | crate eel | Emerald |
| 27 | 27 | crate seafood | — |
| 30 | 30 | crate books | — |

### Жидкости (бочки)

| Good | Предм. | Название | Содержимое |
|:----:|:----:|------|----------|
| 10 | 10 | barrel water | вода |
| 11 | 11 | barrel rum | ром |
| 12 | 12 | barrel beer | пиво |
| 13 | 13 | barrel wine | вино |
| 24 | 24 | barrel spices | специи |

### Материалы

| Good | Предм. | Название |
|:----:|:----:|------|
| 20 | 20 | crate gems |
| 21 | 21 | crate iron |
| 22 | 22 | crate gold |
| 23 | 23 | crate copper |
| 25 | 25 | crate grain |
| 26 | 26 | crate medicine |
| 28 | 28 | crate silk |
| 29 | 29 | crate generic goods |

---

## Товары 31–64: Оптовые (предметы 201–234)

| Good | Предм. | Название | Категория |
|:----:|:----:|------|----------|
| 31 | 201 | crate venison | bulkFood |
| 32 | 202 | crate truffles | bulkFood |
| 33 | 203 | tools | bulkGood |
| 34 | 204 | crate sculptures | bulkGood |
| 35 | 205 | logs | bulkGood |
| 36 | 206 | barrel mead | bulkAlco |
| 37 | 207 | tobacco white (big) | bulkGood |
| 38 | 208 | tobacco green (big) | bulkGood |
| 39 | 209 | tobacco black (big) | bulkGood |
| 40 | 210 | tobacco brown (big) | bulkGood |
| 41 | 211 | tobacco blue (big) | bulkGood |
| 42 | 212 | crate rice | bulkFood |
| 43 | 213 | crate oranges | bulkFood |
| 44 | 214 | crate forest mushrooms | bulkFood |
| 46 | 216 | crate cave mushrooms | bulkFood |
| 47 | 217 | lumber | bulkGood |
| 48 | 218 | nails | bulkGood |
| 49 | 219 | leather | bulkGood |
| 50 | 220 | rabbit furs | bulkGood |
| 51 | 221 | mail | mission |
| 52 | 222 | wool | bulkGood |
| 53 | 223 | olive oil | bulkFood |
| 54 | 224 | apples | bulkFood |
| 55 | 225 | marble | bulkGood |
| 56 | 226 | silver | bulkGood |
| 57 | 227 | sulfur | bulkGood |
| 58 | 228 | barrel cider | bulkAlco |
| 59 | 229 | hemp | bulkGood |
| 60 | 230 | dyes | bulkGood |
| 61 | 231 | rubber | bulkGood |
| 62 | 232 | coffee | bulkFood |
| 63 | 233 | salt | bulkGood |
| 64 | 234 | saffron | bulkGood |

---

## Грузовые предметы (не-Good)

| Индекс | Название |
|:-----:|------|
| 104 | crate of fishing hooks |
| 108 | crate of firewood |
| 131 | lantern candle crate |
| 311–319 | crates of tobacco (white/green/black/brown/blue) |
| 385 | salt barrel |
| 386 | coffee barrel |

---

## Система ящиков

### ShipItemCrate

| Свойство | Описание |
|----------|-------------|
| `containedPrefab` | Префаб содержимого |
| `smokedFood` | Содержимое копчёное |
| `amount` | Количество внутри |
| `mass` | База ящика + `containedPrefab.mass × amount` |

### Распечатывание
```
UnsealCrate():
  for i in 0..amount:
    Instantiate(containedPrefab, pos + up*0.5, rot)
    RegisterToSave()
    если CookableFood: опционально коптить
  DestroyItem()
```

### CrateInventory
Управляет UI запечатанного ящика. `CrateSealUI` — взаимодействие с печатью.

---

## Опт vs Розница

`ShipItem.IsBulk()` = true для: `bulkAlco`, `bulkFood`, `bulkWater`, `bulkGood`.

Оптовые товары идут в грузовой отсек (cargo carrier), не в личный инвентарь.

---

## Миссионные товары

`Good.RegisterToMission(assignedMission, dueDay)`:
- `missionIndex` установлен
- `SaveablePrefab.RegisterToSave()`
- `ShipItem.RegisterMissionGood()` → `sold = true`, включает обводку
- `UpdateLookText()`: `"{name}\nto {port}\ndue: {dueText}"`

Доставка: `Deliver()` → снять тег, уведомить миссию, `DestroyItem()`.

Почта (good 51 / item 221) — специальный миссионный товар.

---

## Региональное происхождение

- **Aestrin:** sunspot fish, templefish, tuna
- **Emerald:** salmon, eel, shimmertail, bananas
- **Medi:** trout, north fish, lamb, cheese, goat cheese
- **Болото:** snapper, bubbler
- **Универсальные:** tools, generic goods

---

*Извлечено из Sailwind v0.38.*
