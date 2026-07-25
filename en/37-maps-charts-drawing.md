# 37. Maps and Charts: Drawing, Data, Ruler

Analysis of sea chart subsystem — physical map items that the player draws routes on. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Coordinates — in note 28, chart saving — in notes 11/22, map item (`ShipItemFoldable`) — in note 16.

## Chart Data Model (`ChartData`)

`[Serializable]` — what a map stores (and what's saved to save as `chartData`, note 11):

```csharp
public class ChartData {
    public ChartLine tempLine;        // line currently being drawn
    public List<ChartLine> lines;     // drawn lines
    public List<ChartPoint> points;   // points/marks
}

public class ChartLine  { float startX, startY, endX, endY; int color; }  // segment
public class ChartPoint { float posX, posY; }                              // point
```

Line/point coordinates — **local coordinates of the map surface** (not world). `color` — pen color index (`0` = none/eraser).

## Map Item (`MapChart`)

Component on a foldable map (`ShipItemFoldable`). Fields: `chartData`, `chartRenderer` (`MeshRenderer` with `RenderTexture`), `rulerScale`, `useLargeRuler`.

- **Rendering:** `MapChartTextureRenderer.instance.UpdateTexture(chartData, renderTexture)` draws `lines`/`points` into map texture. Called after each stroke (`UpdateTexture()`).
- **Drawing:**
  - `OnActivate(localPos)` — start line (`currentLine`, start = `localPos`, color = `MapTableCamera.currentLineColor`) or finish (add to `chartData.lines`, if `color > 0`).
  - `OnLook(localPos)` — draw line (update `endX/endY`) + update ruler.
  - `InputName 9` (alt action) — confirm/cancel line.
  - Writing sound `UISounds.write` on stroke.
- **Ruler**: measures distance between two points (`UpdateRulerPositions(worldStart, worldEnd)`, `UpdateRulerScale`).
- When map is open — `MapChart` attaches to map table camera (`MapTableCamera.instance.cam`), layer 23, collider active.

## Map Table (`MapTableCamera`, singleton)

Separate camera for viewing/drawing on the map:
- `currentMap` (`ShipItem`) — which map is open; `cam` — render camera; `ruler`, `quill` (pen), `prot` (protractor), `kit` (drafting tool set).
- `currentLineColor` — selected pen color.
- `ToggleRuler`, `UpdateRulerScale`, `UpdateRulerPositions`, `RulerToWorldPos`, `DisableMapCam`.
- Map scrolling via drag/controller, pointer.

## Relation to Other Systems

| System | Connection |
|---------|-------|
| `ShipItemFoldable` (note 16) | Map item; `allowCharting`, `mapChart` (`MapChart`). |
| Saving (notes 11/22) | `ChartData` serialized in `SavePrefabData.chartData` / `SaveSailData`. Drawings survive save. |
| Coordinates (note 28) | World/globe coordinates need conversion to map local coordinates (`MapChart.transform.TransformPoint/InverseTransformPoint`). |
| `LocalMap` (note 19) | Which local map (`none/alankh/emerald/medi/lagoon`). |

## 👁 Modder's View: "I want GPS track / auto-map / custom markers"

**Add a point/line to player's map:**
```csharp
var map = /* currently open map: MapTableCamera.instance.currentMap */;
var chart = map.GetComponent<ShipItemFoldable>().mapChart;   // MapChart
var data  = chart.chartData;

// mark (in map local coordinates!):
data.points.Add(new ChartPoint { posX = localX, posY = localY });

// route segment:
data.lines.Add(new ChartLine { startX = ax, startY = ay, endX = bx, endY = by, color = 1 });

chart.UpdateTexture();   // redraw
```

**Convert world position to map coordinates:**
```csharp
// world (accounting for floating origin) → map local
Vector3 world = FloatingOriginManager.instance.RealPosToShiftingPos(realPos);
Vector3 local = chart.transform.InverseTransformPoint(world);
// local.x / local.y → posX/posY on chart
```

**Mod ideas on this base:**
- **GPS track:** on timer/`Sun.OnNewDay` add `ChartLine` segments between last player positions → auto-draw traveled path.
- **Destination markers:** add `ChartPoint` at destination port of active mission (`Mission.destinationPort`, note 15) — "goal mark" on map.
- **Auto-charting islands:** outline shores (points along land boundary) on approach.
- **Chart export/import:** `ChartData` — simple serializable set of lines/points; can save/share routes.

**Pitfalls:**
- `ChartLine/ChartPoint` coordinates — **local map coordinates**, not world. Without `InverseTransformPoint`, marks drift away.
- `color = 0` line won't be added (treated as "eraser"/empty).
- `UpdateTexture()` must be called after changing `chartData`, otherwise drawing won't update until next stroke.
- `currentMap` in `MapTableCamera.instance` — only when map is actually open; for "background" recording, hold reference to `ShipItemFoldable.mapChart` directly.
- Drawings save automatically (`chartData` in save).

## Practical Takeaways

1. **Map = `ChartData`** (lists of `lines`/`points` in local map coordinates) + `MapChart` (rendering to `RenderTexture` via `MapChartTextureRenderer`).
2. **Drawing** — pen via `MapTableCamera` (color `currentLineColor`), ruler/protractor for measurements.
3. **Chart data is saved** to save — drawings are persistent.
4. For **custom marks/tracks**: write to `chartData.lines/points` (local coordinates via `InverseTransformPoint`) + `UpdateTexture()`.
5. This is a ready foundation for GPS tracks, auto-maps, destination markers, and route export.
