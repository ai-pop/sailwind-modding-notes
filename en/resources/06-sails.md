# 06. Sails — Complete Registry

> All sail prefabs organized by region, type, and size.
> Complements note 17 (Wind & Sails), note 22 (Shipyard).
> Sail size numbers are area multipliers for wind force calculation.

---

## Sail Naming Convention

```
[obs] SAIL [Region] [Type] [Size/Description]
```

- **obs** = Obsolete (old version kept for save compatibility)
- **Region:** A=Aestrin, M=Medi, E=Emerald, Am=Aestrin-modified, Em=Emerald-modified, L=Large/Lagoon
- **Types:** square, lateen, gaff, jib, genoa, junk, spritsail, topsail, brig

---

## Aestrin (A) Sails

| Index | Name | Type | Size | Status |
|:-----:|------|------|:----:|:------:|
| 0 | SAIL A small square wide | square | wide | Active |
| 1 | obs SAIL A square 3.5 | square | 3.50 | Obsolete |
| 2 | SAIL A small lateen | lateen | small | Active |
| 3 | SAIL A square 2.5 | square | 2.50 | Active |
| 4 | SAIL A cogsize square | square | cog | Active |
| 6 | SAIL A jib small | jib | small | Active |
| 7 | SAIL A jib cutter1 | jib | cutter1 | Active |
| 8 | SAIL A jib cutter2 | jib | cutter2 | Active |
| 9 | SAIL A jib full | jib | full | Active |
| 10 | SAIL A lateen tallmast | lateen | tallmast | Active |
| 11 | obs SAIL A lateen mizzen | lateen | mizzen | Obsolete |
| 12 | SAIL A lateen long | lateen | long | Active |
| 13 | SAIL A gaff cabin short | gaff | short | Active |
| 14 | SAIL A gaff cabin tall | gaff | tall | Active |
| 15 | SAIL A gaff full | gaff | full | Active |
| 16 | SAIL A gaff full small | gaff | full-small | Active |
| 59 | obs SAIL A cogsize-1 square | square | cog-1 | Obsolete |

### Aestrin-Modified (Am) Sails

| Index | Name | Type | Size | Status |
|:-----:|------|------|:----:|:------:|
| 60 | SAIL Am lateen mid | lateen | mid | Active |
| 61 | SAIL Am lateen small | lateen | small | Active |
| 70 | SAIL Am jib 17yd | jib | 17yd | Active |
| 71 | obs SAIL Am jib 14 | jib | 14 | Obsolete |
| 72 | obs SAIL Am jib 3.3 | jib | 3.3 | Obsolete |
| 75 | obs SAIL Am gaff full | gaff | full | Obsolete |
| 76 | obs SAIL Am gaff cabin | gaff | cabin | Obsolete |
| 77 | obs SAIL Am gaff cabin short 2 | gaff | cabin-short | Obsolete |
| 78 | SAIL Am topsail gaff | gaff | topsail | Active |
| 79 | obs SAIL Am topsail gaff 1.35 | gaff | 1.35 | Obsolete |
| 80 | obs SAIL Am square | square | — | Obsolete |
| 81 | obs SAIL Am wide square 1.7 | square | 1.70 | Obsolete |
| 82 | obs SAIL Am wide square 2.5 | square | 2.50 | Obsolete |

---

## Medi (M) Sails

| Index | Name | Type | Size | Status |
|:-----:|------|------|:----:|:------:|
| 5 | SAIL M tiny gaff | gaff | tiny | Active |
| 30 | SAIL M small gaff | gaff | small | Active |
| 31 | SAIL M cog jib | jib | cog | Active |
| 32 | SAIL M cog square | square | cog | Active |
| 33 | SAIL M cog lateen | lateen | cog | Active |
| 34 | SAIL M cog genoa | genoa | cog | Active |
| 36 | SAIL M gaff 3 | gaff | 3 | Active |
| 37 | SAIL M gaff huge | gaff | huge | Active |
| 38 | SAIL M gaff narrow short | gaff | narrow-short | Active |
| 39 | SAIL M gaff narrow tall | gaff | narrow-tall | Active |
| 40 | SAIL M narrow genoa | genoa | narrow | Active |
| 41 | SAIL M narrow jib | jib | narrow | Active |
| 42 | obs SAIL M tiny jib | jib | tiny | Obsolete |
| 43 | obs SAIL M huge lateen | lateen | huge | Obsolete |
| 44 | obs SAIL M mizzen lateen | lateen | mizzen | Obsolete |
| 45 | obs SAIL M mizzen lateen small | lateen | mizzen-small | Obsolete |
| 46 | SAIL M square spritsail | square | spritsail | Active |
| 47 | obs SAIL M square small | square | small | Obsolete |
| 48 | obs SAIL M cog square mizzen | square | cog-mizzen | Obsolete |
| 49 | obs SAIL M cog square topsail | square | cog-topsail | Obsolete |
| 50 | SAIL M tapered square | square | tapered | Active |
| 51 | obs SAIL M tapered square mizzen | square | tapered-mizzen | Obsolete |
| 52 | obs SAIL M tapered square topsail | square | tapered-topsail | Obsolete |

### Medi Brig Sails

| Index | Name | Type | Size | Status |
|:-----:|------|------|:----:|:------:|
| 110 | SAIL M brig jib | jib | brig | Active |
| 111 | obs SAIL M brig jib 1.35 | jib | 1.35 | Obsolete |
| 113 | obs SAIL M brig square upper .65 | square | 0.65 | Obsolete |
| 114 | obs SAIL M brig square 1.15 | square | 1.15 | Obsolete |
| 115 | SAIL M brig square 1 | square | 1.00 | Active |
| 116 | SAIL M brig square upper | square | upper | Active |
| 117 | SAIL M gaff brig short 0,95 | gaff | 0.95 | Active |
| 118 | obs SAIL M gaff brig short 1.3 | gaff | 1.30 | Obsolete |
| 119 | SAIL M gaff brig 0.95 | gaff | 0.95 | Active |
| 120 | obs SAIL M gaff brig 1.3 | gaff | 1.30 | Obsolete |
| 121 | obs SAIL M lateen BIG 2.1 | lateen | 2.10 | Obsolete |

---

## Emerald (E) Sails

| Index | Name | Type | Size | Status |
|:-----:|------|------|:----:|:------:|
| 17 | obs SAIL E gaff junk main | gaff | junk-main | Obsolete |
| 18 | SAIL E gaff junk mizzen | gaff | junk-mizzen | Active |
| 19 | obs SAIL E gaff straight main | gaff | straight-main | Obsolete |
| 20 | obs SAIL E small junk 9 | junk | 9.00 | Obsolete |
| 21 | SAIL E small junk 7 | junk | 7.00 | Active |
| 22 | obs SAIL E smaller junk 5 | junk | 5.00 | Obsolete |
| 23 | SAIL E gaff straight mizzen | gaff | straight-mizzen | Active |
| 24 | SAIL E genoa kakam | genoa | kakam | Active |
| 25 | SAIL E jib kakam | jib | kakam | Active |
| 26 | SAIL E square tiny | square | tiny | Active |
| 27 | obs SAIL E square junk 6 | square | 6.00 | Obsolete |
| 28 | obs SAIL E square junk 7 | square | 7.00 | Obsolete |
| 29 | obs SAIL E square junk 8 | square | 8.00 | Obsolete |
| 53 | obs SAIL E jib kakam small | jib | kakam-small | Obsolete |

### Emerald-Modified (Em) Sails

| Index | Name | Type | Size | Status |
|:-----:|------|------|:----:|:------:|
| 90 | SAIL Em junk | junk | — | Active |
| 91 | SAIL Em junk smol | junk | small | Active |
| 92 | obs SAIL Em junk 1.15 | junk | 1.15 | Obsolete |
| 93 | obs SAIL Em junk 1.33 | junk | 1.33 | Obsolete |
| 95 | obs SAIL Em jib 2,3 | jib | 2.3 | Obsolete |
| 96 | obs SAIL Em jib 2,0 | jib | 2.0 | Obsolete |
| 97 | obs SAIL Em jib 1,6 | jib | 1.6 | Obsolete |
| 98 | obs SAIL Em jib 1,3 | jib | 1.3 | Obsolete |
| 100 | obs SAIL Em square junk 0,85 | square | 0.85 | Obsolete |
| 101 | SAIL Em square junk 1,0 | square | 1.00 | Active |
| 102 | obs SAIL Em square junk 1.45 | square | 1.45 | Obsolete |
| 103 | obs SAIL Em square junk 1.66 | square | 1.66 | Obsolete |
| 106 | obs SAIL Em gaff straight 1.6 | gaff | 1.60 | Obsolete |
| 107 | obs SAIL Em gaff junk 2.0 | gaff | 2.00 | Obsolete |

---

## Large/Lagoon (L) Sails

| Index | Name | Type | Size | Status |
|:-----:|------|------|:----:|:------:|
| 127 | obs SAIL L junklateen 1_3 | junklateen | 1.3 | Obsolete |
| 128 | SAIL L junklateen | junklateen | — | Active |
| 129 | obs SAIL L junklateen 1_6 | junklateen | 1.6 | Obsolete |
| 130 | obs SAIL L junklateen 2_0 | junklateen | 2.0 | Obsolete |

---

## Sail-Related Code Components

### Key Classes

| Class | Purpose |
|-------|---------|
| `Sail` | Base sail component on sail prefabs |
| `SailConnections` | Rope/attachment points |
| `SailAreaPrefabSetter` | Sets sail area from prefab |
| `SailCategory` | Sail type category enum |
| `BoatWind` | Wind force applied to sails |
| `RopeControllerSailAngle` | Sail angle adjustment |
| `RopeControllerSailAngleJib` | Jib-specific angle control |
| `RopeControllerSailAngleSquare` | Square sail angle control |
| `RopeControllerSailReef` | Reefing control |
| `ReefEffect*` | Visual reef effects (Jib, JunkSail, ScaleUniversal, SquareBoom) |
| `SailFlapAudio` | Sail flapping sound |
| `SailHingeAudio` | Hinge/yard rotation sound |
| `SailShadowCol` | Sail shadow collider |

### SailAreaPrefabSetter

This component sets the sail area based on prefab reference, connecting the `PrefabsDirectory.sails[]` array to runtime sail physics.

---

## Sail Color System

`PrefabsDirectory.sailColors[]` — array of available sail colors. Applied at shipyard via `ShipyardSailInstaller`.

---

## Sail Types by Region Summary

| Region | Square | Lateen | Gaff | Jib | Genoa | Junk | Other |
|:------:|:------:|:------:|:----:|:---:|:-----:|:----:|:-----:|
| **Aestrin (A)** | ✓ | ✓ | ✓ | ✓ | — | — | — |
| **Aestrin-mod (Am)** | ✓ | ✓ | ✓ | ✓ | — | — | topsail |
| **Medi (M)** | ✓ | ✓ | ✓ | ✓ | ✓ | — | spritsail |
| **Medi Brig** | ✓ | ✓ | ✓ | ✓ | — | — | — |
| **Emerald (E)** | ✓ | — | ✓ | ✓ | ✓ | ✓ | — |
| **Emerald-mod (Em)** | ✓ | — | ✓ | ✓ | — | ✓ | — |
| **Large (L)** | — | — | — | — | — | ✓ | junklateen |

---

*Extracted from Sailwind v0.38 asset analysis. Obsolete sails are retained in PrefabsDirectory for backward save compatibility.*
