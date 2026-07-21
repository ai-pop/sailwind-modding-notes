# 37. Карты и чарты: рисование, данные, линейка

Разбор подсистемы морских карт — физических карт-предметов, на которых игрок рисует маршруты. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Координаты — в заметке 28, сохранение чартов — в заметках 11/22, предмет-карта (`ShipItemFoldable`) — в заметке 16.

## Модель данных чарта (`ChartData`)

`[Serializable]` — то, что хранит карта (и что сохраняется в сейв как `chartData`, заметка 11):

```csharp
public class ChartData {
    public ChartLine tempLine;        // линия, которую рисуют прямо сейчас
    public List<ChartLine> lines;     // нарисованные линии
    public List<ChartPoint> points;   // точки/метки
}

public class ChartLine  { float startX, startY, endX, endY; int color; }  // отрезок
public class ChartPoint { float posX, posY; }                              // точка
```

Координаты линий/точек — **локальные координаты поверхности карты** (не мировые). `color` — индекс цвета пера (`0` = нет/ластик).

## Карта-предмет (`MapChart`)

Компонент на раскладной карте (`ShipItemFoldable`). Поля: `chartData`, `chartRenderer` (`MeshRenderer` с `RenderTexture`), `rulerScale`, `useLargeRuler`.

- **Рендер:** `MapChartTextureRenderer.instance.UpdateTexture(chartData, renderTexture)` рисует `lines`/`points` в текстуру карты. Вызывается после каждого штриха (`UpdateTexture()`).
- **Рисование:**
  - `OnActivate(localPos)` — начать линию (`currentLine`, старт = `localPos`, цвет = `MapTableCamera.currentLineColor`) или завершить (добавить в `chartData.lines`, если `color > 0`).
  - `OnLook(localPos)` — вести линию (обновлять `endX/endY`) + обновлять линейку.
  - `InputName 9` (альт-действие) — подтвердить/отменить линию.
  - Звук письма `UISounds.write` при штрихе.
- **Линейка** (ruler): измеряет расстояние между двумя точками (`UpdateRulerPositions(worldStart, worldEnd)`, `UpdateRulerScale`).
- Когда карта открыта — `MapChart` прикрепляется к камере стола (`MapTableCamera.instance.cam`), слой 23, коллайдер активен.

## Стол с картой (`MapTableCamera`, синглтон)

Отдельная камера, через которую рассматривается/рисуется карта:
- `currentMap` (`ShipItem`) — какая карта открыта; `cam` — камера рендера; `ruler`, `quill` (перо), `prot` (транспортир), `kit` (набор чертёжных инструментов).
- `currentLineColor` — выбранный цвет пера.
- `ToggleRuler`, `UpdateRulerScale`, `UpdateRulerPositions`, `RulerToWorldPos`, `DisableMapCam`.
- Прокрутка карты драгом/контроллером, указатель.

## Связь с другими системами

| Система | Связь |
|---------|-------|
| `ShipItemFoldable` (заметка 16) | Предмет-карта; `allowCharting`, `mapChart` (`MapChart`). |
| Сохранение (заметки 11/22) | `ChartData` сериализуется в `SavePrefabData.chartData` / `SaveSailData`. Нарисованное переживает сейв. |
| Координаты (заметка 28) | Мировые/globe-координаты нужно переводить в локальные координаты карты (`MapChart.transform.TransformPoint/InverseTransformPoint`). |
| `LocalMap` (заметка 19) | Какая локальная карта (`none/alankh/emerald/medi/lagoon`). |

## 👁 Взгляд моддера: «хочу GPS-трек / авто-карту / свои метки»

**Добавить точку/линию на карту игрока:**
```csharp
var map = /* текущая открытая карта: MapTableCamera.instance.currentMap */;
var chart = map.GetComponent<ShipItemFoldable>().mapChart;   // MapChart
var data  = chart.chartData;

// метка (локальные координаты карты!):
data.points.Add(new ChartPoint { posX = localX, posY = localY });

// отрезок маршрута:
data.lines.Add(new ChartLine { startX = ax, startY = ay, endX = bx, endY = by, color = 1 });

chart.UpdateTexture();   // перерисовать
```

**Перевод мировой позиции в координаты карты:**
```csharp
// мировая (с учётом плавающего начала координат) → локальная карты
Vector3 world = FloatingOriginManager.instance.RealPosToShiftingPos(realPos);
Vector3 local = chart.transform.InverseTransformPoint(world);
// local.x / local.y → posX/posY на чарте
```

**Идеи модов на этой базе:**
- **GPS-трек:** по таймеру/`Sun.OnNewDay` добавлять `ChartLine`-сегменты между последними позициями игрока → авто-отрисовка пройденного пути.
- **Маркеры целей:** добавлять `ChartPoint` в порт назначения активной миссии (`Mission.destinationPort`, заметка 15) — «метка цели» на карте.
- **Авто-чартинг островов:** обводить берега (точки по границе суши) при приближении.
- **Экспорт/импорт чарта:** `ChartData` — простой сериализуемый набор линий/точек; можно сохранять/делиться маршрутами.

**Грабли:**
- Координаты `ChartLine/ChartPoint` — **локальные координаты карты**, не мировые. Без `InverseTransformPoint` метки уедут.
- `color = 0` линия не добавляется (считается «ластиком»/пустой).
- `UpdateTexture()` нужно вызывать после изменения `chartData`, иначе рисунок не обновится до следующего штриха.
- `currentMap` в `MapTableCamera.instance` — только когда карта реально открыта; для записи «в фоне» держите ссылку на `ShipItemFoldable.mapChart` напрямую.
- Нарисованное сохраняется автоматически (`chartData` в сейве).

## Практические выводы

1. **Карта = `ChartData`** (списки `lines`/`points` в локальных координатах карты) + `MapChart` (рендер в `RenderTexture` через `MapChartTextureRenderer`).
2. **Рисование** — пером через `MapTableCamera` (цвет `currentLineColor`), линейка/транспортир для измерений.
3. **Данные чарта сохраняются** в сейв — нарисованное персистентно.
4. Для **своих меток/треков**: писать в `chartData.lines/points` (локальные координаты через `InverseTransformPoint`) + `UpdateTexture()`.
5. Это готовая основа для GPS-трека, авто-карты, маркеров целей и экспорта маршрутов.
