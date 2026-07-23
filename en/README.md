# Sailwind — Technical Modding Reference

Documentation on architecture and internals of **Sailwind** (v0.38, Unity 2019.1.10f1), significant for mod development. Contains information impossible to obtain without decompiling `Assembly-CSharp.dll` or empirical runtime analysis: save systems, economy, physics, weather, items, NPCs, and much more.

The reference grew from notes for a translation mod (notes 01–10) into a **complete description of game systems** (notes 11–32). Start with note [30 (global references and singletons)](30-global-refs-singletons.md) and [24 (decompilation coverage)](24-decompilation-coverage-missing-classes.md) if you want a quick orientation.

## Contents by Section

### Text, UI, and Input (from translation mod)
| № | File | Topic |
|---|------|------|
| 01 | [textmesh-not-tmp.md](01-textmesh-not-tmp.md) | Text rendered via TextMesh, not TextMeshPro |
| 02 | [baked-text-bypasses-setter.md](02-baked-text-bypasses-setter.md) | Prefab-baked text bypasses C# setter |
| 03 | [gopointer-input-system.md](03-gopointer-input-system.md) | Custom GoPointer input system instead of EventSystem |
| 04 | [static-font-trap.md](04-static-font-trap.md) | Font loading peculiarities at runtime |
| 05 | [tab-formatted-labels.md](05-tab-formatted-labels.md) | Formatted labels with column separators |
| 06 | [keycode-strings.md](06-keycode-strings.md) | Keycode strings requiring translation exclusion |
| 07 | [sre-false-harmony-works.md](07-sre-false-harmony-works.md) | Mono runtime without SRE — HarmonyX still works |
| 08 | [lookui-hardcoded-strings.md](08-lookui-hardcoded-strings.md) | Complete list of hardcoded UI strings |
| 09 | [google-translate-gotchas.md](09-google-translate-gotchas.md) | Unofficial Google Translate endpoint peculiarities |
| 10 | [class-reference.md](10-class-reference.md) | Quick Assembly-CSharp.dll class reference |

### Core: Saving, State, References
| № | File | Topic |
|---|------|------|
| 11 | [save-system-and-mod-persistence.md](11-save-system-and-mod-persistence.md) | Save system, save format, **`GameState.modData` hook** |
| 30 | [global-refs-singletons.md](30-global-refs-singletons.md) | Global references (Refs/RefsDirectory), singleton map, constants |
| 24 | [decompilation-coverage-missing-classes.md](24-decompilation-coverage-missing-classes.md) | Decompilation coverage: missing classes and API reconstruction |
| 21 | [debugger-cheats-tuning.md](21-debugger-cheats-tuning.md) | Debugger, hidden debug mode (P+N+T), global multipliers |
| 35 | [audio-mixer-ui-ambient.md](35-audio-mixer-ui-ambient.md) | Audio: mixer snapshots, UI sounds, time-of-day ambient |
| 39 | [juicebox-tweens-gamefeel.md](39-juicebox-tweens-gamefeel.md) | Juicebox: reusable tweens, shake, hit-stop, particles (for UI mod) |

### Player and Survival
| № | File | Topic |
|---|------|------|
| 12 | [player-state-needs-recovery.md](12-player-state-needs-recovery.md) | Needs, currencies, reputation, blackout mechanic |
| 25 | [sleep-bed-rest.md](25-sleep-bed-rest.md) | Sleep, bed, timeskip and time acceleration |
| 36 | [player-movement-climb-swim.md](36-player-movement-climb-swim.md) | Player movement: jumping, swimming, crouching, ratlines, anti-stuck |

### Boat: Physics, Sails, Control
| № | File | Topic |
|---|------|------|
| 14 | [boat-physics-buoyancy-damage.md](14-boat-physics-buoyancy-damage.md) | Hull physics: buoyancy, mass, damage, and sinking |
| 17 | [wind-and-sails.md](17-wind-and-sails.md) | Wind model (trade winds, gusts, storms) and sail propulsion |
| 29 | [anchor-mooring-ropes.md](29-anchor-mooring-ropes.md) | Anchor (auto-set/release), mooring, rope controllers |
| 22 | [shipyard-customization-purchase.md](22-shipyard-customization-purchase.md) | Shipyard, boat customization (part dependencies), purchase |
| 38 | [ropes-rigging-steering.md](38-ropes-rigging-steering.md) | Ropes/rigging: RopeController, steering wheel (rudder spring, lock), winches, autopilot |
| 40 | [hull-dirt-cleaning.md](40-hull-dirt-cleaning.md) | Hull dirt (texture, daily fouling) and cleaning via MasterPainter |

### World, Time, Ocean
| № | File | Topic |
|---|------|------|
| 18 | [time-weather-storms.md](18-time-weather-storms.md) | Time (`OnNewDay`), moon, regional weather, wandering storms |
| 19 | [world-ports-regions.md](19-world-ports-regions.md) | Regions, ports (34 max), local maps, world border |
| 31 | [ocean-waves-inertia.md](31-ocean-waves-inertia.md) | Crest ocean: wave field inertia, wave height queries |
| 43 | [item-buoyancy-water.md](43-item-buoyancy-water.md) | Item buoyancy and water: ItemRigidbody/SimpleFloatingObject, kinematics, water height, ritual |
| 28 | [navigation-instruments-world-scale.md](28-navigation-instruments-world-scale.md) | Navigation instruments and world scale conclusion: **1 unit = 1 meter** |
| 37 | [maps-charts-drawing.md](37-maps-charts-drawing.md) | Maps/charts: ChartData (lines/points), drawing, GPS track for mod |
| 41 | [cameras-view-modes.md](41-cameras-view-modes.md) | Cameras: BoatCamera (boat overview, InputName 16), CameraFollow, custom cameras |

### Economy, Missions, Items
| № | File | Topic |
|---|------|------|
| 13 | [economy-markets-currency.md](13-economy-markets-currency.md) | Economy: markets, currencies, price model, trader boats |
| 15 | [missions-cargo-mail.md](15-missions-cargo-mail.md) | Missions: generation from arbitrage, cargo and mail delivery |
| 27 | [story-quests.md](27-story-quests.md) | Story quests: QuestDude dialogues, state machine, rewards |
| 16 | [item-framework-shipitem.md](16-item-framework-shipitem.md) | Item framework: ShipItem, physics, inventory, family |
| 44 | [itemrigidbody-field-map-contract.md](44-itemrigidbody-field-map-contract.md) | `ItemRigidbody`: field map, twin contract, freeze/unfreeze, position master |
| 45 | [crate-cargo-prefabs-filter.md](45-crate-cargo-prefabs-filter.md) | Crates/cargo prefabs: `amount`/`containedPrefab` semantics, loot filter |
| 32 | [inventory-cargo-storage.md](32-inventory-cargo-storage.md) | Inventory (5 slots), cargo carriers, transport/storage pricing |
| 23 | [fishing-and-food.md](23-fishing-and-food.md) | Fishing (bite, tension, catch), food spoilage and preservation |
| 26 | [cooking-stove-fuel.md](26-cooking-stove-fuel.md) | Cooking: stove, fuel, boiling/smoking, cooking degree |
| 33 | [item-spawning-pickup.md](33-item-spawning-pickup.md) | Item spawning in world, pickup, buoyancy, crates + modder's view |
| 34 | [worked-example-floating-loot.md](34-worked-example-floating-loot.md) | Worked example: floating loot crates at sea (BepInEx mod recipe) |
| 42 | [worked-example-fast-travel.md](42-worked-example-fast-travel.md) | Worked example: fast travel/teleport (floating coordinates, Refs, Blackout) |

### Population
| № | File | Topic |
|---|------|------|
| 20 | [npcs-world-population.md](20-npcs-world-population.md) | NPCs: waypoint boats, fishermen, PortDude, Shopkeeper, residents |

## Key Facts for Quick Start

- **Mod persistence:** write to `GameState.modData["MyMod.*"]` — saves automatically (note 11).
- **World scale:** 1 Unity unit = 1 meter; mission "km" = 100 m; globe coordinates = meters / 9000 (note 28).
- **Central daily hook:** `Sun.OnNewDay` (note 18).
- **Decompilation incomplete:** `GameInput`, `InputName`, `BoatProbes`, Crest helpers missing — their APIs reconstructed (note 24).
- **Hidden debug mode in release:** hold P+N, press T (note 21).

## Domain

- **Game:** [Sailwind](https://store.steampowered.com/app/1764530/Sailwind/) v0.38
- **Engine:** Unity 2019.1.10f1 (ocean — custom Crest fork)
- **Backend:** Mono (recommended), IL2CPP (experimental)
- **Modding framework:** BepInEx 5.4.23.5 (HarmonyX)
- **Analysis method:** decompilation of `Assembly-CSharp.dll` via ILSpy, runtime logging via BepInEx
- **Source material:** [sailwind-decompiled](https://github.com/ai-pop/sailwind-decompiled) (677 game classes), [sailwind-translator](https://github.com/ai-pop/sailwind-translator) mod

## License

MIT. Materials are free for use in any Sailwind mods.
