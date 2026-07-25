# 04. Equipment & Tools

> Navigation instruments, tools, fishing gear, lighting, and other equipment.
> Complements notes 28 (Navigation), 23 (Fishing), 17 (Wind/Sails).

---

## Navigation Instruments

### Compasses

| Prefab Index | Name | Special Feature |
|:-----------:|------|-----------------|
| 80 | compass A | Aestrin compass |
| 81 | compass E | Emerald compass |
| 82 | compass M | Medi compass |
| 86 | sun compass A | Sun compass — casts shadow for latitude. Sets `GameState.holdingSunCompass`. Has `chronoLatitude` scroll adjustment. `sharpenShadow=true`. |

**Compass behavior:** Standard compass points north. Sun compass variant uses sundial/gnomon shadow for latitude measurement. `lockX/Y/Z` fields constrain rotation when held. `OnScroll()` adjusts chronometer latitude.

### Chronometers/Clocks

| Prefab Index | Name | Notes |
|:-----------:|------|-------|
| 83 | chronometer A | Aestrin chronometer |
| 84 | chronometer E | Emerald chronometer |
| 85 | chronometer M | Medi chronometer |
| 170 | clock A | Aestrin clock (decorative?) |
| 171 | clock E | Emerald clock — with lid |
| 172 | clock M | Medi clock |

**Clock behavior:** Hands rotate via `Sun.sun.globalTime`. `hourHand` rotates at `time × 2 × 15` degrees. `minuteHand` at `time × 2 × 15 × 12`. Lid toggle on alt-activate (if `lid` exists).

### Quadrant

| Prefab Index | Name |
|:-----------:|------|
| 90 | quadrant |

**Behavior:** Alt-activate toggles inspection mode — rotates `rotatingParent` 90° for angle reading. `dial` shows angle. `lockX/Y/Z` constrain axes.

### Spyglasses

| Prefab Index | Name | Zoom |
|:-----------:|------|:----:|
| 160 | spyglass big | Wide zoom range |
| 161 | spyglass small | Medium zoom |
| 162 | spyglass tiny | Narrow zoom |

**Behavior:** Scroll to zoom (`currentZoom` between `minZoom` and `maxZoom`). `movingParts[]` extend/retract. Has internal `Camera` component for zoom rendering.

### Chip Log (Speedometer)

| Prefab Index | Name |
|:-----------:|------|
| 92 | chip log M |
| 93 | chip log E |

**Behavior:** Throwing mechanic — cast bobber into water, rope unrolls. Measures speed via bobber drag. `maxLength=40` units of rope. `bobberForceMult=10`. Has reel audio, rope effect, and configurable joint for bobber physics.

### Maps & Charting

| Prefab Index | Name | Notes |
|:-----------:|------|-------|
| 115 | map ocean | Ocean overview |
| 116 | map A | Aestrin map |
| 117 | map E | Emerald map |
| 118 | map M | Medi map |
| 119 | map L | Lagoon/Large map |
| 139 | map F | Map variant F |
| 165 | map mirage mountain | Special location map |

Maps are `ShipItemFoldable` with `allowCharting=true`. Unfold to view. `MapChart` component handles drawing. Use `ShipItemInkSet` (index 167) to draw on maps.

### Ink Set

| Prefab Index | Name | Behavior |
|:-----------:|------|----------|
| 167 | ink set | Click on foldable map to draw charts |

---

## Tools

### Repair & Maintenance

| Prefab Index | Name | Function |
|:-----------:|------|----------|
| 159 | hammer | Hull repair / nailing |
| 166 | oakum | Caulking — seals hull leaks |

**Oakum behavior:** Alt-activate near damaged boat. Amount transferred from item to hull: `amount -= num`, `boatDamage.oakum += num`. Rounds to `maxAmount` percentage. Sound: `UISounds.oakum`.

### Cleaning

| Prefab Index | Name | Function |
|:-----------:|------|----------|
| 94 | broom | Sweeps dirt off deck |

**Broom behavior:** Alt-activate starts sweeping (`cleaner.activated = true`). Disables red outline while sweeping. `Cleaner` component manages sweep animation and dirt removal.

### Oars

| Prefab Index | Name |
|:-----------:|------|
| — | oar (prefab index not confirmed) |

`ShipItemOar` class exists in decompilation. Manual rowing propulsion.

### Slicing Knives

| Prefab Index | Name | Region |
|:-----------:|------|:------:|
| 370 | slicing knife A | Aestrin |
| 371 | slicing knife E | Emerald |
| 372 | slicing knife M | Medi |

**Behavior:** Click on food to slice. Generates sliced variant based on `FoodState.slicePrefabIndex`. `KnifeCollider` component handles the cutting detection.

---

## Fishing Equipment

### Fishing Rods

| Prefab Index | Name |
|:-----------:|------|
| 95 | fishing rod 1 |

**Behavior:** `ShipItemFishingRod` — accepts hook (`amount <= 0` = needs hook). Casting mechanic with rope/bobber similar to chip log. Has line physics.

### Fishing Hooks

| Prefab Index | Name | Notes |
|:-----------:|------|-------|
| 99 | fishing hook | Single hook |
| 104 | crate of fishing hooks | Bulk hooks |

**Behavior:** Click on rod to attach. Used as bait. `ShipItemFishingHook.AllowOnItemClick()` checks for `ShipItemFishingRod` with `amount <= 0`.

---

## Lighting

### Lanterns

| Prefab Index | Name | Fuel | Color |
|:-----------:|------|:----:|:-----:|
| 110 | lantern A | Oil | Warm (Aestrin) |
| 111 | lantern E yellow | Oil | Yellow (Emerald) |
| 112 | lantern E red | Oil | Red (Emerald) |
| 113 | lantern E green | Oil | Green (Emerald) |
| 114 | lantern M | Oil | Warm (Medi) |
| 133 | lantern M big | Oil | Large (Medi) |
| 134 | lantern E blu | Oil | Blue (Emerald) |

### Candle Lanterns

| Prefab Index | Name | Fuel |
|:-----------:|------|:----:|
| 130 | lantern candle | Candle |
| 131 | lantern candle crate(N | Bulk candles |

### Lantern Fuel

| Prefab Index | Name | Type |
|:-----------:|------|------|
| 132 | lantern oil bottle | Oil refill |

**Fuel behavior:** `ShipItemLanternFuel.oilBottle` distinguishes oil from candles. `RequestOil(amountRequested)` transfers fuel: reduces own `health`, returns amount transferred. `RequestCandle()` destroys the candle item.

### Lamp Hooks

| Prefab Index | Name |
|:-----------:|------|
| 79 | lamp hanger generic |

**Behavior:** `ShipItemLampHook` — click with hangable item (lantern) to attach. Creates `ConfigurableJoint`. `occupied` flag prevents multiple attachments. Joint copied from hook's existing joint to `itemRigidbody`.

---

## Item Interaction Rules

### AllowOnItemClick Matrix

| Item A (held) ↓ / Item B (looked at) → | Bottle | Soup | Kettle | Food | Salt | Light | LanternFuel | Rod | Hook | Foldable | LampHook |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Bottle** | ✓ | ✓ | ✓ | — | — | — | — | — | — | — | — |
| **Soup** | ✓ | — | — | — | — | — | — | — | — | — | — |
| **Kettle** | ✓ | — | — | — | — | — | — | — | — | — | — |
| **Food** | — | — | — | — | — | — | — | — | — | — | — |
| **Salt** | — | — | — | ✓ | — | — | — | — | — | — | — |
| **Tea** | — | — | ✓ | — | — | — | — | — | — | — | — |
| **LanternFuel** | — | — | — | — | — | ✓ | — | — | — | — | — |
| **FishingHook** | — | — | — | — | — | — | — | ✓ | — | — | — |
| **InkSet** | — | — | — | — | — | — | — | — | — | ✓ | — |
| **Lantern (hangable)** | — | — | — | — | — | — | — | — | — | — | ✓ |
| **Knife** | — | — | — | ✓ (slice) | — | — | — | — | — | — | — |

---

## Spyglass Zoom Mechanics

```
Scroll → OnScroll(float input)
  currentZoom = clamp(currentZoom + scrollDelta, minZoom, maxZoom)
  UpdateCam() — adjusts camera FOV
  UpdateMovingParts() — extends/retracts tube meshes
  UpdateCapsuleCol() — adjusts capsule collider height/center
```

---

*Extracted from Sailwind v0.38 decompilation.*
