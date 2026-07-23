# 34. Worked Example: Floating Loot Crates at Sea

A concrete mod recipe, analyzed as a tutorial example: **"randomly spawn floating loot crates at sea that the player can pick up"**. Links notes 11, 13, 16, 31, 33. Goal — show how documented "handles" assemble into a real feature, and what pitfalls to watch for.

## What the Game Already Provides (no need to implement)

| Need | Take as Ready | Note |
|-------|---------------|---------|
| Item can be picked up | `GoPointer.PickUpItem` (click via raycast) | 33, 10 |
| Item floats | `ItemRigidbody` adds `SimpleFloatingObject` automatically | 33 |
| Item persists | `SaveablePrefab.RegisterToSave()` → `savedPrefabs` | 11 |
| Water height at point | `OceanHeight.GetHeight(helper, pos)` | 31 |
| World coordinates | `FloatingOriginManager.instance` (`outCurrentOffset`) | 11 |
| Periodicity | `Sun.OnNewDay` or custom timer in `Update` | 18 |
| Mod data between sessions | `GameState.modData` | 11 |

**You only need to implement:** your item prefab + your spawn logic (where/when/how many).

## Step 1. Loot Crate Prefab

Need a GameObject prefab with components (minimum):
- `ShipItemCrate` (or `ShipItem`) + `Rigidbody` + `Collider` + `Renderer`;
- `SaveablePrefab` with **unique `prefabIndex`**;
- configured `containedPrefab` (what's inside) and `amount` (how many).

**Pitfalls:**
- `prefabIndex` must exist in `PrefabsDirectory.instance.directory`. You can't directly write a foreign prefab there at runtime (array is fixed, index validated in `Start`). **Options:** (a) reuse an existing index of an appropriate item (e.g., one of the crates) and override `containedPrefab`/`amount` after spawning; (b) for a "real" new item — AssetBundle + patch `PrefabsDirectory` (separate big topic).
- For a tutorial example, easiest to **reuse an existing crate prefab** from `directory` and modify contents.

## Step 2. Where to Spawn — Random Point on Water Near Player

```csharp
// player position in "real" coordinates
var playerReal = FloatingOriginManager.instance
                    .ShiftingPosToRealPos(Refs.observerMirror.transform.position);

// random offset in 200..800 m ring around player (1 unit = 1 m, note 28)
float ang = Random.Range(0f, Mathf.PI * 2f);
float dist = Random.Range(200f, 800f);
var realPos = playerReal + new Vector3(Mathf.Cos(ang) * dist, 0f, Mathf.Sin(ang) * dist);

// water height at this point (Crest)
var helper = new SampleHeightHelper();          // Crest class, see note 24/31
float waterY = OceanHeight.GetHeight(helper, realPos);
realPos.y = waterY;

// back to scene coordinates (accounting for floating origin!)
var scenePos = FloatingOriginManager.instance.RealPosToShiftingPos(realPos);
```

**Pitfalls:**
- **Floating origin**: world shifts (`outCurrentOffset`). Spawn in **scene coordinates** (`RealPosToShiftingPos`), but "think" about the world in real coordinates (`ShiftingPosToRealPos`). If you spawn directly in "real" coordinates — item will drift kilometers after shift.
- **Don't spawn too far**: game (`WorldItemSpawner`) uses 100 m threshold for visible spawning. For loot in 200–800 m ring, item appears off-screen — that's fine, but physics far away may be "sleeping".
- **`SampleHeightHelper`** — from missing Crest module (note 24); need to get it from assembly. Alternative — spawn slightly above water and rely on `SimpleFloatingObject` to lower item to surface.

## Step 3. Spawning (by "ritual" from note 33)

```csharp
void SpawnLootCrate(Vector3 scenePos)
{
    // take existing crate prefab from directory (tutorial variant)
    var prefab = PrefabsDirectory.instance.directory[cratePrefabIndex];

    var go = Object.Instantiate(prefab, scenePos, Quaternion.identity);
    var item = go.GetComponent<ShipItem>();

    // "ritual" of a world item:
    item.sold = true;                                    // world-owned
    go.GetComponent<Good>()?.RegisterAsMissionless();    // not a mission

    // (optionally) override crate contents:
    // var crate = go.GetComponent<ShipItemCrate>();
    // crate.amount = Random.Range(1, 5);  // etc.

    // saving:
    go.GetComponent<SaveablePrefab>().RegisterToSave();
}
```

Pickup, floating, and saving **work automatically** (from "what the game already provides" step).

**How the player collects loot** (crate mechanic, note 33): player picks up floating crate → RMB on sealed crate shows `CrateSealUI` (unseal confirmation) → on unsealing, contents **transfer into crate inventory** (crate stays as intact container, `amount → 0`) → crate is then opened as a container and items extracted. So "loot drop" here is specifically a floating sealed crate, not scattered items on water (more reliable: nothing sinks or gets lost).

## Step 4. When to Spawn — BepInEx Plugin

```csharp
[BepInPlugin(GUID, "FloatingLoot", "1.0.0")]
public class FloatingLootPlugin : BaseUnityPlugin
{
    public const string GUID = "com.example.floatingloot";
    private float _timer;
    private Harmony _harmony;

    private void Awake()
    {
        _harmony = new Harmony(GUID);
        // patch SaveModData/LoadModData for own data, if needed (note 11)
        _harmony.PatchAll();
        Logger.LogInfo("FloatingLoot loaded");
    }

    private void Update()
    {
        if (!GameState.playing || GameState.currentlyLoading) return;

        _timer -= Time.deltaTime;
        if (_timer <= 0f)
        {
            _timer = 300f;                      // every 5 minutes of real time
            if (ActiveCrates < MaxCrates)        // own tracking (see pitfalls)
                SpawnLootCrate(RandomOceanPoint());
        }
    }
}
```

**Pitfalls and Solutions:**
- **Tracking active crates**: game doesn't keep a registry of "your" items. Maintain own list (`List<SaveablePrefab>`/instanceId), clean on `OnDestroy`/`Unregister`, otherwise counter drifts.
- **Persistence between sessions**: spawned crates save normally (`RegisterToSave` → `savedPrefabs`), but **your spawner state** (timer, counter) store in `GameState.modData` (note 11) or via patching `SaveModData/LoadModData`.
- **Time pause**: `Update` runs in real time; for "game" time multiply by `Sun.sun.timescale` (note 18) or subscribe to `Sun.OnNewDay`.
- **Don't spawn during sleep/shipyard/recovery**: check `GameState.sleeping`, `GameState.currentShipyard`, `GameState.recovering`.
- **Item limit**: too many live `Rigidbody` hurts performance; keep `MaxCrates` small and despawn distant ones (by `Vector3.Distance` to player).

## Step 5. Despawn Distant/Old Loot

```csharp
// periodic check of own list:
if (Vector3.Distance(crate.position, Refs.observerMirror.position) > 3000f)
    crate.GetComponent<ShipItem>().DestroyItem();   // removes from save + Destroy (note 16)
```
`DestroyItem()` correctly unregisters item from saving (`SaveablePrefab.Unregister`).

## Feature Checklist

- [x] Item prefab with `ShipItem`+`SaveablePrefab` (+ valid `prefabIndex`)
- [x] Random point on water: real coordinates → `OceanHeight.GetHeight` → scene coordinates (`RealPosToShiftingPos`)
- [x] Spawn ritual: `Instantiate` → `sold=true` → `RegisterAsMissionless` → `RegisterToSave`
- [x] Pickup/floating/saving — out of the box
- [x] Timer/limits in `Update` (or `Sun.OnNewDay`), despawn distant via `DestroyItem`
- [x] Spawner state — in `GameState.modData`

## Generalization: Recipe for "Any World Item"

Any "item in the world" feature is built the same way:
1. **Prefab** `ShipItem` + `SaveablePrefab` (+ valid index; for crates — filter in note 45).
2. **Position**: think in real coordinates, spawn in scene coordinates (`±outCurrentOffset`), Y — canonical water height snippet from note 43.
3. **Ritual**: `sold=true` (+ `RegisterAsMissionless`), `RegisterToSave`; **wait for twin** (note 44).
4. **Pickup, saving, UI tooltip** provided by the game. **Floating is NOT**: built-in floater is disabled (`ToggleCollider`), need **your own float** (Boyant-snap) or patch — **fully covered in note 43** (truth table, sequence, checklist, antipatterns); twin/visual contract — in note 44.

> **Critical for "floats and is clickable":** put physics/float only on the **twin** (`GetItemRigidbody()`), not visual (otherwise pickup breaks); keep body non-kinematic (`debugForceKinematic=false` on `ItemRigidbody`, `sold=true`). End-to-end checklist — in note 43.

This same skeleton covers: floating barrels, debris with loot, "treasures" by coordinates, drifting items, item-markers, etc.
