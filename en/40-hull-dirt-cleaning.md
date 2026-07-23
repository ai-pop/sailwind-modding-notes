# 40. Hull Dirt and Cleaning

Analysis of hull fouling/dirt and cleaning mechanics. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to texture saving (note 11, `extraTexture`), daily tick (note 18, `Sun.OnNewDay`) and hull state (note 14, `BoatDamage`).

## How It Works

Boat hull has a **dirt overlay texture** (second material `dirtMaterial` with `mainTexture`). Dirt is not a number, but a **actually painted texture** (128×128) that accumulates and is erased with a "brush".

```
CleanableObject (on hull)
   └─ dirtMaterial.mainTexture  ← dirt texture
        └─ modified via MasterPainter (ApplyCoat / PaintObject)
        └─ saved in SaveableObject.extraTexture (note 11)
Cleaner (scraper/broom item)
   └─ when rubbing against hull, erases dirt (MasterPainter.PaintObject)
```

## `CleanableObject` (`[RequireComponent(HullPlayerCollider)]`)

| Field/Method | Content |
|------------|-----------|
| `dirtCoat` (`Texture2D`) | Dirt brush/mask. |
| `dirtMaterial` | Dirt overlay material on hull. |
| `ApplyDailyDirt()` | **Every day** (`Sun.OnNewDay`): apply thin dirt layer — `MasterPainter.instance.ApplyCoat(this, dirtCoat, 0.02f)`. Hull gradually fouls. |
| `ApplyNewDirtTexture(tex)` | Replace dirt texture + save in `saveable.extraTexture`. |
| `GetCurrentDirtTex()` | Current dirt texture. |
| `LoadTexture(tex)` / `RegisterToSaveable(obj)` | Load from save / bind to `SaveableObject`. |
| `CleanFully()` | Full clean (debug: `MasterPainter.DebugApplyRenderTex`). |

- Subscribed to `Sun.OnNewDay` in `OnEnable` / unsubscribed in `OnDisable`.
- Texture saved as `SaveableObject.extraTexture` (note 11): in save — raw (16 777 216 bytes) or PNG, 128×128.

## `Cleaner` (Cleaning Item)

`Cleaner` is a `ShipItem` (scraper/broom) with animated "bone" (`Rigidbody bone`), two renders (skinned in hand / static on deck).

| Field | Content |
|------|-----------|
| `range` (0.6) | Cleaning radius. |
| `spacing` (1) | Stroke step. |
| `minSpeed` (0.5) | Min movement speed for cleaning. |
| `activated` | Whether currently scrubbing. |
| `uvPos` | Stroke position in hull UV. |

- When item is in hand and player **rubs hull** (movement at speed ≥ `minSpeed`), `Cleaner` erases dirt at contact point via `MasterPainter` (strokes by UV with step `spacing`, radius `range`).
- Sweeping animation (sidesweep), sound, particles.

## `MasterPainter` (Painting Engine)

Singleton `MasterPainter.instance`, paints on hull render-texture:
- `ApplyCoat(cleanable, coatTex, amount)` — apply layer (dirt/paint) with force `amount`.
- `PaintObject(cleanable, uv, ...)` — stroke at UV point (cleaning/painting).
- `DebugApplyRenderTex(cleanable)` — apply/reset (full clean).

## Relation to Other Systems

| System | Connection |
|---------|-------|
| Saving (note 11) | Dirt texture = `SaveableObject.extraTexture` (128×128, raw/PNG). |
| `Sun.OnNewDay` (note 18) | Daily dirt application (`ApplyDailyDirt`). |
| `BoatDamage`/performance (note 14) | Dirty hull likely affects drag/speed (`HullDrag`); cleaning restores performance. |
| `SaveablePaint` (note 22) | Hull painting — same `MasterPainter` mechanism. |

## 👁 Modder's View

| Want | How |
|------|-----|
| Change fouling rate | Patch `CleanableObject.ApplyDailyDirt` (multiplier on `0.02f`) or `MasterPainter.ApplyCoat`. |
| Instantly clean hull | `cleanable.CleanFully()` / `MasterPainter.instance.DebugApplyRenderTex(cleanable)`. |
| Add dirt/paint | `MasterPainter.instance.ApplyCoat(cleanable, tex, amount)`. |
| Read "cleanliness" | `cleanable.GetCurrentDirtTex()` and analyze texture (average alpha/brightness). |
| Auto-clean / "non-fouling" hull | Unsubscribe from `Sun.OnNewDay` (`OnDisable`-patch) or nullify `ApplyCoat`. |
| Visualize cleanliness in HUD | Read dirt texture and display indicator. |

**Pitfalls:**
- Dirt is **a texture**, not a scalar: "cleanliness percentage" must be computed from pixels (e.g., average alpha of `GetCurrentDirtTex()`).
- Texture is 128×128; on save, game distinguishes raw/PNG **by array length** (note 11) — if you replace `extraTexture`, preserve size/format.
- `ApplyDailyDirt` fires on `Sun.OnNewDay` only when `GameState.playing`; code checks `saveable.extraSetting` (boat purchased).
- `MasterPainter` operates on render-texture/UV — for custom painting you need hull UV coordinates.

## Practical Takeaways

1. **Hull dirt = overlay texture** (128×128), accumulates daily (`OnNewDay`, layer 0.02) and erased by `Cleaner` item via `MasterPainter`.
2. **Cleaning** — physical rubbing of hull with scraper (strokes by UV, radius `range`, min speed `minSpeed`).
3. **Texture saved** in `SaveableObject.extraTexture` (raw/PNG) — fouling survives save.
4. Control via `MasterPainter.instance` (`ApplyCoat`/`PaintObject`/`DebugApplyRenderTex`) and `CleanableObject` (`CleanFully`/`GetCurrentDirtTex`).
5. Same mechanic (`MasterPainter` + `SaveablePaint`) used for **hull painting** (note 22).
