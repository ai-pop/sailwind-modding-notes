# 22. Shipyard, Customization, and Boat Purchase

Analysis of the boat modification subsystem. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## Customization Framework (`BoatCustomParts` / `BoatPart` / `BoatPartOption`)

A boat consists of **customizable parts** (masts, hull elements, etc.), each part having a set of mutually exclusive options.

### `BoatCustomParts` (on boat)
- `availableParts` (`List<BoatPart>`) — all customizable parts.
- `RefreshParts()` / `RefreshPartsWithOrder(order)` — enables the active option for each part, disables others.
- `GetActiveOption(partIndex)` — current option of a part.
- `ForceEnableOption(partIndex, optionIndex)` — switches option (deactivates old GameObject + `walkColObject`, activates new).

### `BoatPart` (`[Serializable]`)
| Field | Content |
|------|-----------|
| `activeOption` | Active option index. |
| `partOptions` (`List<BoatPartOption>`) | Available options. |
| `category` | Part category (int). |

`SetOptionEnabled(i, state)` activates/deactivates option's GameObject + its `childOptions` + `walkColObject` (walkable collider).

### `BoatPartOption` (MonoBehaviour)
| Field | Content |
|------|-----------|
| `optionName` | Option name. |
| `basePrice` / `installCost` | Base price / installation cost. |
| `mass` | Mass (affects `BoatMass`, note 14). |
| `requires` (`List<BoatPartOption>`) | **Dependencies:** which options must be installed. |
| `requiresDisabled` (`List<BoatPartOption>`) | Which options must be **disabled** (conflicts). |
| `walkColObject` | Passage collider. |
| `childOptions` / `childMast` | Child objects / child mast. |

### Dependency System
- **`CanInstall(part, option)`:** option can be installed if all `requires` are active **and** all `requiresDisabled` are inactive. Otherwise returns string `"requires: ..."` / `"no ..."`.
- **`CanUninstall(part, option)`:** cannot remove if another active option depends on it (`requires.Contains`), **or** if it's a mast with attached sails (`Mast.sails.Count > 0`, including `childMast`).

## Boat Purchase (`PurchasableBoat`)

| Field/Method | Content |
|------------|-----------|
| `price` | Boat price. |
| `region` | Region (price currency). |
| `PurchaseBoat()` | **Purchase flag = `saveable.extraSetting = true`**; deducts `price` from `PlayerGold.currency[3]` (**Gold Lions**), logs to `DayLogs[3]`. |
| `isPurchased()` | `return saveable.extraSetting`. |
| `LoadAsPurchased()` | On load, if `extraSetting` — state "purchased". |

> **`SaveableObject.extraSetting` for a boat = "purchased"** (same field used by `Recovery.FindClosestBoat` to find player-owned boats and in saves, see notes 11/12). Boats are purchased with **gold** (currency 3), not regional money.

## Shipyard (`Shipyard`)

Stationary workshop for installing sails, replacing parts, repairing and cleaning hull.

| Field | Content |
|------|-----------|
| `region` | Shipyard region (determines transaction currency). |
| `availableSailColors` / `sailPrefabs` | Available sail colors and prefabs. |
| `editedShipPosition` / `shipReleasePosition` | Boat position during editing / on release. |
| `sailInstaller` / `partsInstaller` | Sail / parts installers. |

### Key Mechanics
- **Shipyard fee** (`GetShipyardFee`) = `PurchasableBoat.price × 3.5` — **350% of base boat price**.
- **Order currency:** prices converted by region rate — `GetCurrencyRate() = CurrencyMarket.currentPrices[region]`. Sail installation cost = `GetSailPrice() * currencyRate`.
- **Order** (`UpdateOrder`): sums sail installation costs (per mast), subtracts returns (`ShipyardRefund`), adds parts costs (`partsInstaller`). Forms `currentOrderText`.
- **Order options:** `currentOrderIncludesCleaning` (hull cleaning), `currentOrderIncludesRepair` (repair).
- **Cancel** (`CancelOrder`): restores `originalData` snapshot (`SaveBoatCustomizationData`) — all changes rolled back.
- **Installation errors:** `installError`, `installedSailObstructedError` (sail obstructs others).
- While player is at shipyard, `GameState.currentShipyard != null` — this **blocks saving, needs ticking, and time progression** (see notes 11/12/18).

## Sail Data (`SaveSailData`)

`[Serializable]`, full configuration of installed sail:

| Field | Content |
|------|-----------|
| `prefabIndex` | Index in `PrefabsDirectory.sails`. |
| `mastIndex` | Which mast. |
| `installHeight` | Installation height. |
| `minAngle` / `maxAngle` | Angle range. |
| `health` | Sail condition. |
| `sailColor` | Color (from `availableSailColors`). |
| `scaleY` / `scaleZ` | Canvas scale (see note 17). |

## Customization Persistence (`SaveBoatCustomizationData`)

`[Serializable]`, saved to save as `SaveableObject.customization` (note 11):

| Field | Content |
|------|-----------|
| `masts[30]` (`bool[]`) | Which masts are installed (up to 30 slots). |
| `sails` (`List<SaveSailData>`) | All installed sails with configuration. |
| `partActiveOptions` (`List<int>`) | Active options for each part. |

`SaveablePaint` — marker component (`[RequireComponent(SaveableObject)]`): hull paint saved separately (clean/dirt texture — `extraTexture` in `SaveableObject`, note 11).

## Practical Takeaways for Modding

1. **Customization** = tree of parts with options and dependencies (`requires`/`requiresDisabled`); cannot remove mast with sails.
2. **Boat purchase** with Gold Lions (currency 3); ownership flag = `SaveableObject.extraSetting`.
3. **Shipyard fee** = 3.5× boat price; payment in regional currency by `CurrencyMarket` rate.
4. **Order cancellation** rolls back to `SaveBoatCustomizationData` snapshot — safe to experiment.
5. **Sails** save full configuration (height, angles, scale, color, health) in `SaveSailData`.
6. At shipyard (`GameState.currentShipyard`) time/needs/saving are paused.
