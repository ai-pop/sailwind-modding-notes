# 🚢 Sailwind — Technical Modding Reference

[![Engine](https://img.shields.io/badge/Unity-2019.1.10f1-blue?logo=unity)]()
[![Backend](https://img.shields.io/badge/Mono-primary%20%7C%20IL2CPP-exp.-orange)]()
[![Framework](https://img.shields.io/badge/BepInEx-5.4.23.5_(HarmonyX)-purple)]()
[![Game version](https://img.shields.io/badge/Sailwind-v0.38-green)]()
[![Notes](https://img.shields.io/badge/notes-56-brightgreen)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey)]()

> 🇷🇺 [Русская версия → `../README.md`](../README.md)

Complete documentation on the architecture and internals of **Sailwind** (v0.38), obtained by decompiling `Assembly-CSharp.dll` and runtime analysis. 56 notes cover: save systems, economy, hull physics and ocean, items (twin model), weather, NPCs, UI/text, cameras, coordinate systems, and **full breakdown of the SailwindItemPhysics v4.2 crash** (held-item twin colliding with boat).

---

## 📋 Quick Start

| What you need | Where to find |
|-----------|-----------|
| Mod persistence → `GameState.modData["MyMod.*"]` | [Note 11](11-save-system-and-mod-persistence.md) |
| World scale: **1 Unity unit = 1 meter**, mission km = 100 m | [Note 28](28-navigation-instruments-world-scale.md) |
| Global references and singletons | [Note 30](30-global-refs-singletons.md) |
| Central daily hook: `Sun.OnNewDay` | [Note 18](18-time-weather-storms.md) |
| Hidden debug mode: P+N, press T | [Note 21](21-debugger-cheats-tuning.md) |
| Incomplete decompilation — which classes are missing | [Note 24](24-decompilation-coverage-missing-classes.md) |
| Item framework: ShipItem → twin → ItemRigidbody | [Note 16](16-item-framework-shipitem.md) + [44](44-itemrigidbody-field-map-contract.md) |
| Crash with solid twin collider | [Notes 47–56](#sailwinditemphysics-v42-crash-investigation) |

---

## 📚 Contents

### Text, UI, and Input
| № | File | Topic |
|---|------|------|
| 01 | [textmesh-not-tmp.md](01-textmesh-not-tmp.md) | Text via TextMesh, not TextMeshPro |
| 02 | [baked-text-bypasses-setter.md](02-baked-text-bypasses-setter.md) | Prefab-baked text bypasses C# setter |
| 03 | [gopointer-input-system.md](03-gopointer-input-system.md) | Custom GoPointer input system |
| 04 | [static-font-trap.md](04-static-font-trap.md) | Font loading at runtime |
| 05 | [tab-formatted-labels.md](05-tab-formatted-labels.md) | Formatted labels with tab separators |
| 06 | [keycode-strings.md](06-keycode-strings.md) | Keycode strings — exclude from translation |
| 07 | [sre-false-harmony-works.md](07-sre-false-harmony-works.md) | Mono without SRE — HarmonyX works |
| 08 | [lookui-hardcoded-strings.md](08-lookui-hardcoded-strings.md) | Complete list of hardcoded UI strings |
| 09 | [google-translate-gotchas.md](09-google-translate-gotchas.md) | Google Translate endpoint quirks |
| 10 | [class-reference.md](10-class-reference.md) | Quick Assembly-CSharp.dll class reference |

### Core: Saving, State, References
| № | File | Topic |
|---|------|------|
| 11 | [save-system-and-mod-persistence.md](11-save-system-and-mod-persistence.md) | Save system, `GameState.modData` hook |
| 30 | [global-refs-singletons.md](30-global-refs-singletons.md) | Global references (Refs/RefsDirectory), singletons |
| 24 | [decompilation-coverage-missing-classes.md](24-decompilation-coverage-missing-classes.md) | Decompilation coverage: missing classes |
| 21 | [debugger-cheats-tuning.md](21-debugger-cheats-tuning.md) | Debugger, debug mode (P+N+T), multipliers |
| 35 | [audio-mixer-ui-ambient.md](35-audio-mixer-ambient.md) | Audio: mixer snapshots, UI sounds, ambient |
| 39 | [juicebox-tweens-gamefeel.md](39-juicebox-tweens-gamefeel.md) | Juicebox: tweens, shake, hit-stop, particles |

### Player and Survival
| № | File | Topic |
|---|------|------|
| 12 | [player-state-needs-recovery.md](12-player-state-needs-recovery.md) | Needs, currencies, reputation, blackout |
| 25 | [sleep-bed-rest.md](25-sleep-bed-rest.md) | Sleep, bed, timeskip |
| 36 | [player-movement-climb-swim.md](36-player-movement-climb-swim.md) | Movement: jumps, swimming, ratlines, anti-stuck |

### Boat: Physics, Sails, Control
| № | File | Topic |
|---|------|------|
| 14 | [boat-physics-buoyancy-damage.md](14-boat-physics-buoyancy-damage.md) | Hull physics: buoyancy, mass, damage, sinking |
| 17 | [wind-and-sails.md](17-wind-and-sails.md) | Wind (trade winds, gusts, storms) and sail propulsion |
| 29 | [anchor-mooring-ropes.md](29-anchor-mooring-ropes.md) | Anchor, mooring, ropes |
| 22 | [shipyard-customization-purchase.md](22-shipyard-customization-purchase.md) | Shipyard: boat customization, purchase |
| 38 | [ropes-rigging-steering.md](38-ropes-rigging-steering.md) | Rigging: steering wheel (rudder spring), winches, autopilot |
| 40 | [hull-dirt-cleaning.md](40-hull-dirt-cleaning.md) | Hull dirt and cleaning (MasterPainter) |

### World, Time, Ocean
| № | File | Topic |
|---|------|------|
| 18 | [time-weather-storms.md](18-time-weather-storms.md) | Time (`OnNewDay`), moon, weather, wandering storms |
| 19 | [world-ports-regions.md](19-world-ports-regions.md) | Regions, ports (34 max), world borders |
| 31 | [ocean-waves-inertia.md](31-ocean-waves-inertia.md) | Crest ocean: wave field inertia, height queries |
| 43 | [item-buoyancy-water.md](43-item-buoyancy-water.md) | Item buoyancy: ItemRigidbody/SimpleFloatingObject |
| 28 | [navigation-instruments-world-scale.md](28-navigation-instruments-world-scale.md) | Navigation: **1 unit = 1 m**, coordinates |
| 37 | [maps-charts-drawing.md](37-maps-charts-drawing.md) | Maps/charts: ChartData, drawing, GPS |
| 41 | [cameras-view-modes.md](41-cameras-view-modes.md) | Cameras: BoatCamera, CameraFollow, custom cameras |
| 46 | [water-physics-illusion.md](46-water-physics-illusion.md) | Water physics: volume illusion, SampleHeightHelper lifecycle |

### Economy, Missions, Items
| № | File | Topic |
|---|------|------|
| 13 | [economy-markets-currency.md](13-economy-markets-currency.md) | Economy: markets, currencies, prices |
| 15 | [missions-cargo-mail.md](15-missions-cargo-mail.md) | Missions: generation from arbitrage, delivery |
| 27 | [story-quests.md](27-story-quests.md) | Story: QuestDude, state machine, rewards |
| 16 | [item-framework-shipitem.md](16-item-framework-shipitem.md) | Item framework: ShipItem, physics, inventory |
| 44 | [itemrigidbody-field-map-contract.md](44-itemrigidbody-field-map-contract.md) | `ItemRigidbody`: field map, twin contract |
| 45 | [crate-cargo-prefabs-filter.md](45-crate-cargo-prefabs-filter.md) | Crates: `amount`/`containedPrefab`, loot filter |
| 32 | [inventory-cargo-storage.md](32-inventory-cargo-storage.md) | Inventory (5 slots), cargo carriers |
| 23 | [fishing-and-food.md](23-fishing-and-food.md) | Fishing, food spoilage and preservation |
| 26 | [cooking-stove-fuel.md](26-cooking-stove-fuel.md) | Cooking: stove, fuel, boiling/smoking |
| 33 | [item-spawning-pickup.md](33-item-spawning-pickup.md) | Item spawning, pickup, buoyancy |
| 34 | [worked-example-floating-loot.md](34-worked-example-floating-loot.md) | 🛠 Example: floating loot crates (BepInEx recipe) |
| 42 | [worked-example-fast-travel.md](42-worked-example-fast-travel.md) | 🛠 Example: fast travel/teleport |

### Population
| № | File | Topic |
|---|------|------|
| 20 | [npcs-world-population.md](20-npcs-world-population.md) | NPCs: waypoint boats, fishermen, PortDude, Shopkeeper |

### SailwindItemPhysics v4.2 crash investigation — Round 1 (A1–A5, B6–B12, C13–C14, D15)
| № | File | Topic |
|---|------|------|
| 47 | [item-holding-pickup-flow.md](47-item-holding-pickup-flow.md) | 🔬 Item holding: end-to-end flow (PickUp→FixedUpdate→Drop), twin vs visual, teleport on hold, `held` lifecycle |
| 48 | [ocean-height-helper-lifecycle.md](48-ocean-height-helper-lifecycle.md) | 🔬 Ocean height: `SampleHeightHelper` lifecycle, canCheckBuoyancyNow, batch queries, response delay |
| 49 | [camera-render-shaders.md](49-camera-render-shaders.md) | 🔬 Cameras/shaders: BoatCamera, rendering path, Z-write, depth fog, water surface shader |
| 50 | [saves-moddata-instanceid.md](50-saves-moddata-instanceid.md) | 🔬 Saves/modData: instanceId, SaveablePrefab, modData persistence, save/load cycle, what's lost |

### SailwindItemPhysics v4.2 crash investigation — Round 2 (E1–E6)
> Crash/hang when held-item solid twin collider contacts boat.

| № | File | Topic |
|---|------|------|
| 51 | [boat-collider-topology-dhow.md](51-boat-collider-topology-dhow.md) | 🔬 **E1:** Boat collider topology (dhow): CapsuleCollider (root), BoatEmbarkCollider, walkCollider, HullPlayerCollider, GO hierarchy |
| 52 | [layers-collision-matrix-items-boat-player.md](52-layers-collision-matrix-items-boat-player.md) | 🔬 **E2:** Unity layers (0/2/5/8/12/13/14/23/26), collision matrix, why twin (layer 2) collides with CapsuleCollider but player doesn't |
| 53 | [enterboat-exitboat-flap-mechanism.md](53-enterboat-exitboat-flap-mechanism.md) | 🔬 **E3:** EnterBoat/ExitBoat: trigger callbacks, `frameCounter>1` gate, BoatMass.AddItem idempotent, ≥22 Hz flap scenario |
| 54 | [go-pointer-big-item-decollision.md](54-go-pointer-big-item-decollision.md) | 🔬 **E4:** Big-item decollision in GoPointer: **no while loops**, ComputePenetration single-pass, Boat tag excluded, self-interaction risk |
| 55 | [crash-scenarios-boat-item-collision.md](55-crash-scenarios-boat-item-collision.md) | 🔬 **E5:** Crash scenarios: twin Untagged → BoatDamage.Impact **not filtered** → hullDamage 0.15/s, physics solver loop → frame time explosion |
| 56 | [nailed-wallattachment-deck-placement.md](56-nailed-wallattachment-deck-placement.md) | 🔬 **E6:** nailed (kinematic lock), wallAttachment (raycast+snap+attached), HangableItem (ConfigurableJoint), deck placement vanilla vs mod |

---

## 🔧 Technical Information

### Decompilation

| Parameter | Value |
|----------|----------|
| DLL | `Assembly-CSharp.dll` |
| Game version | Sailwind v0.38 (Steam) |
| Tool | ILSpy 7.x |
| Coverage | ~828 classes (full except `GameInput`, `InputName`, `BoatProbes`, Crest helpers — API reconstructed in note 24) |
| Runtime verification | BepInEx console + Harmony hooks + Debug.Log |

### Item twin model (key concept)

Every `ShipItem` has two GameObjects:
- **Visual GO** (`ShipItem` + Rigidbody kinematic + Collider isTrigger) — what player sees, raycast, interact
- **Twin GO** (`ItemRigidbody` + Rigidbody dynamic + Collider isTrigger in vanilla / non-trigger in mod) — physics, buoyancy, `EnterBoat`/`ExitBoat`

In vanilla twin = isTrigger → "transparent" for boat CapsuleCollider. Mod SailwindItemPhysics v4.2 makes twin non-trigger → twin **physically collides** with CapsuleCollider → crash scenarios 51–55.

### Unity specifics of Sailwind

| System | Details |
|---------|--------|
| Ocean | Custom Crest fork (OceanRenderer, SampleHeightHelper, batch queries) |
| FloatingOrigin | `FloatingOriginManager` — world shift every ~1 km for precision (notes 43, 47) |
| Layers | 0=Default, 2=IgnoreRaycast(twin), 5=UI, 8=Player, 12=HullPlayerCollider(walkCol), 13=Boat, 14=Terrain, 23=Map, 26=Crate (note 52) |
| Raycast mask | `LayerMask.op_Implicit(-604165)` — GoPointer sees most layers, doesn't see 2/5 |
| Fixed timestep | `fixedDeltaTime = 0.022s` (~45.5 Hz physics) |
| Debug mode | P+N → T → hidden multipliers, kinematic items, disableItemUpdate (note 21) |

---

## 🗂 Repository Structure

```
sailwind-modding-notes/
├── README.md                      ← Russian (primary)
├── LICENSE                        ← MIT
├── 01-textmesh-not-tmp.md ... 56-*.md   ← 56 notes (RU)
└── en/
    ├── README.md                  ← English translation
    ├── 01-textmesh-not-tmp.md ... 56-*.md   ← 56 notes (EN)
```

Each note is a standalone Markdown document with:
- 🔬 Header, link to research request (for 47–56)
- Field/type/value tables
- Verbatim bodies of key methods (ILSpy)
- Sequence diagrams and block diagrams
- Practical conclusions "what this means for modders"

---

## 📜 License

MIT — materials are free for use in any Sailwind mods. See [LICENSE](../LICENSE).
