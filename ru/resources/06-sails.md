# 06. Паруса — Полный Реестр

> Все префабы парусов по регионам, типам и размерам.
> Дополняет заметку 17 (Ветер и Паруса), заметку 22 (Верфь).
> Числа размеров — множители площади для расчёта силы ветра.

---

## Соглашение об именах

```
[obs] SAIL [Регион] [Тип] [Размер/Описание]
```

- **obs** = устаревший (для совместимости сохранений)
- **Регион:** A=Aestrin, M=Medi, E=Emerald, Am=Aestrin-модиф., Em=Emerald-модиф., L=Large

---

## Aestrin (A)

| Индекс | Имя | Тип | Размер |
|:-----:|------|------|:----:|
| 0 | SAIL A small square wide | square | wide |
| 1 | obs SAIL A square 3.5 | square | 3.50 |
| 2 | SAIL A small lateen | lateen | small |
| 3 | SAIL A square 2.5 | square | 2.50 |
| 4 | SAIL A cogsize square | square | cog |
| 6 | SAIL A jib small | jib | small |
| 7 | SAIL A jib cutter1 | jib | cutter1 |
| 8 | SAIL A jib cutter2 | jib | cutter2 |
| 9 | SAIL A jib full | jib | full |
| 10 | SAIL A lateen tallmast | lateen | tallmast |
| 11 | obs SAIL A lateen mizzen | lateen | mizzen |
| 12 | SAIL A lateen long | lateen | long |
| 13 | SAIL A gaff cabin short | gaff | short |
| 14 | SAIL A gaff cabin tall | gaff | tall |
| 15 | SAIL A gaff full | gaff | full |
| 16 | SAIL A gaff full small | gaff | full-small |

### Aestrin-модиф. (Am)

| Индекс | Имя | Тип | Размер |
|:-----:|------|------|:----:|
| 60 | SAIL Am lateen mid | lateen | mid |
| 61 | SAIL Am lateen small | lateen | small |
| 70 | SAIL Am jib 17yd | jib | 17yd |
| 71 | obs SAIL Am jib 14 | jib | 14 |
| 72 | obs SAIL Am jib 3.3 | jib | 3.3 |
| 78 | SAIL Am topsail gaff | gaff | topsail |

---

## Medi (M)

| Индекс | Имя | Тип | Размер |
|:-----:|------|------|:----:|
| 5 | SAIL M tiny gaff | gaff | tiny |
| 30 | SAIL M small gaff | gaff | small |
| 31 | SAIL M cog jib | jib | cog |
| 32 | SAIL M cog square | square | cog |
| 33 | SAIL M cog lateen | lateen | cog |
| 34 | SAIL M cog genoa | genoa | cog |
| 36 | SAIL M gaff 3 | gaff | 3 |
| 37 | SAIL M gaff huge | gaff | huge |
| 38 | SAIL M gaff narrow short | gaff | narrow-short |
| 39 | SAIL M gaff narrow tall | gaff | narrow-tall |
| 40 | SAIL M narrow genoa | genoa | narrow |
| 41 | SAIL M narrow jib | jib | narrow |
| 46 | SAIL M square spritsail | square | spritsail |
| 50 | SAIL M tapered square | square | tapered |

### Medi Brig

| Индекс | Имя | Тип | Размер |
|:-----:|------|------|:----:|
| 110 | SAIL M brig jib | jib | brig |
| 115 | SAIL M brig square 1 | square | 1.00 |
| 116 | SAIL M brig square upper | square | upper |
| 117 | SAIL M gaff brig short 0,95 | gaff | 0.95 |
| 119 | SAIL M gaff brig 0.95 | gaff | 0.95 |

---

## Emerald (E)

| Индекс | Имя | Тип | Размер |
|:-----:|------|------|:----:|
| 18 | SAIL E gaff junk mizzen | gaff | junk-mizzen |
| 21 | SAIL E small junk 7 | junk | 7.00 |
| 23 | SAIL E gaff straight mizzen | gaff | straight-mizzen |
| 24 | SAIL E genoa kakam | genoa | kakam |
| 25 | SAIL E jib kakam | jib | kakam |
| 26 | SAIL E square tiny | square | tiny |

### Emerald-модиф. (Em)

| Индекс | Имя | Тип | Размер |
|:-----:|------|------|:----:|
| 90 | SAIL Em junk | junk | — |
| 91 | SAIL Em junk smol | junk | small |
| 101 | SAIL Em square junk 1,0 | square | 1.00 |

---

## Large/Lagoon (L)

| Индекс | Имя | Тип | Размер |
|:-----:|------|------|:----:|
| 128 | SAIL L junklateen | junklateen | — |

---

## Связанные классы кода

| Класс | Назначение |
|-------|---------|
| `Sail` | Базовый компонент паруса |
| `SailConnections` | Точки крепления верёвок |
| `SailAreaPrefabSetter` | Установка площади из префаба |
| `SailCategory` | Категория типа паруса |
| `BoatWind` | Сила ветра на паруса |
| `RopeControllerSailAngle` | Управление углом паруса |
| `RopeControllerSailReef` | Управление рифлением |
| `ReefEffect*` | Визуальные эффекты рифления |
| `SailFlapAudio` | Звук хлопанья паруса |

### SaveSailData

| Поле | Тип | Описание |
|-------|------|-------------|
| `prefabIndex` | int | Индекс паруса |
| `mastIndex` | int | Индекс мачты |
| `installHeight` | float | Высота установки |
| `minAngle`/`maxAngle` | float | Углы паруса |
| `health` | float | Состояние (100=новый) |
| `sailColor` | int | Индекс цвета |
| `scaleY`/`scaleZ` | float | Масштаб |

### Масса паруса (GetSailMass)
```
junk/gaff:    sailPower × 40
staysail:     sailPower × 20
остальные:    sailPower × 30
```

---

## Типы парусов по регионам

| Регион | Square | Lateen | Gaff | Jib | Genoa | Junk |
|:------:|:------:|:------:|:----:|:---:|:-----:|:----:|
| Aestrin | ✓ | ✓ | ✓ | ✓ | — | — |
| Medi | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Emerald | ✓ | — | ✓ | ✓ | ✓ | ✓ |
| Large | — | — | — | — | — | ✓ |

---

*Извлечено из Sailwind v0.38.*
