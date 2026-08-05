# 81. Human NPCs: Behavior, Trading, Quests, Procedural Animations, and the Absence of Navigation

A complete technical analysis of all **human NPC classes** in Sailwind v0.38 (`Assembly-CSharp.dll`). Unlike background vessels (`NPCBoatController`, [Note 77](77-npc-ai-navigation-collision-avoidance-and-waypoint-graphs.md)), human characters in the world of Sailwind are architected completely differently. This note reveals their behavioral architecture, procedural animation system, merchant pricing formulas, quest state machines, and explains **why human NPCs have no active world navigation**.

---

## 1. Human NPC Navigation: Why Characters Don't Walk

In Sailwind v0.38, **all human NPCs are stationary GameObjects** fixed in ports, taverns, and shops (`PortDude`, `QuestDude`, `Shopkeeper`, `TavernRumorsDude`, `CargoTransportDude`).

There is **no `NavMesh`, `NavMeshAgent`, or pathfinding system for human characters anywhere in the game**. The only occurrence of `NavMesh` in the entire `Assembly-CSharp.dll` assembly is within an Oculus VR teleportation sample (`TeleportTargetHandlerNavMesh`).

### 1.1. Abandoned Movement Code: The `Shopkeeper.WalkTo` Coroutine

The sole attempt to implement a walking human NPC in the entire codebase exists as a private coroutine within the `Shopkeeper` class:

```csharp
private IEnumerator WalkTo(Vector3 targetLocalPos, bool walkingHome)
{
    Vector3 targetWorldPos = transform.parent.TransformPoint(targetLocalPos);
    atHome = false;
    walking = true;
    float remainingDistance = Vector3.Distance(transform.position, targetWorldPos);
    transform.LookAt(targetWorldPos, Vector3.up);

    float speedModifier = GameState.justStarted ? 99999f : 1f;

    while (remainingDistance > 0f)
    {
        float step = Time.deltaTime * 0.9f * speedModifier;
        if (step > remainingDistance)
            step = remainingDistance;

        transform.Translate(Vector3.forward * step, Space.Self);
        remainingDistance -= step;
        yield return new WaitForEndOfFrame();
    }
    walking = false;
    if (walkingHome)
        atHome = true;
    else
        transform.rotation = shopRotation;
}
```

#### Analysis of `WalkTo`
1. **Linear Locomotion:** The character simply faces the target (`LookAt`) and translates forward in a straight line at **0.9 m/s** (`Time.deltaTime * 0.9f`), without obstacle avoidance.
2. **Instant Startup Teleport:** If the game just initialized (`GameState.justStarted`), speed is multiplied by `99999f` to snap the NPC instantly to its target destination.
3. **Dead Code:** The `Shopkeeper` class **has no `Update()` method**, and `WalkTo` is **never called** from anywhere in `Assembly-CSharp.dll`. The developer experimented with moving the shopkeeper between home (`homePos`) and the shop counter (`shopLocalPos`) but abandoned the feature, leaving all human NPCs stationary.

---

## 2. Procedural Animation and IK (`NPCAnimations` & `NPCPlayerCol`)

Because human NPCs do not walk, they do not require complex skeletal `Animator` or `SkinnedMeshRenderer` setups. Instead, all characters are animated **procedurally via the `NPCAnimations` component**.

```
        [ Transform: head ]
        ──► Slerp (3 Hz) to Camera.main when playerInRange
                 │
      ┌──────────┴──────────┐
      ▼                     ▼
[ breatheParts[] ]    [ lockParts[] ]
  Z-axis rotation       World-space locking
  (QuadraticInOut)      (feet/hands stay planted)
```

### 2.1. Procedural Breathing and Swaying
Each frame, `currentTime` oscillates between `0` and `breatheDuration`. For each bone in `breatheParts[]`, a Z-axis rotation angle is evaluated:

```csharp
float angle = QuadraticInOut(currentTime, 0f, breatheAngles[i], breatheDuration);
breatheParts[i].localRotation = baseRots[i];
breatheParts[i].Rotate(Vector3.forward, angle, Space.Self);
```

Where `QuadraticInOut` is a quadratic easing curve.

### 2.2. Ground Locking (`lockParts`)
To prevent NPC feet from sliding across tavern floors or shop counters while the torso sways, `lockParts[]` forces limbs back to their initial world coordinates every frame:

```csharp
for (int j = 0; j < lockParts.Length; j++)
{
    Vector3 worldPos = transform.TransformPoint(basePositions[j]);
    lockParts[j].localPosition = lockParts[j].parent.InverseTransformPoint(worldPos);
}
```

### 2.3. Dynamic Head Look IK (`headLookAngle`)
Each human NPC has a child trigger with `NPCPlayerCol`. When the player enters this trigger, it sets `playerInRange = true`.
- If the camera lies within the forward cone defined by `headLookAngle` (default **30°**), the NPC head smoothly rotates toward the player camera at **3 Hz** (`Time.deltaTime * 3f`).
- When the player leaves the sector or trigger, the head returns to `headInitialRot` at **7 Hz** (`Time.deltaTime * 7f`).

---

## 3. Merchant Shopkeeper (`Shopkeeper`)

`Shopkeeper` handles retail commerce in shops (`ShopArea`). It interacts with the player via a collider trigger: bringing a `ShipItem` into the shop trigger evaluates `item.sold` and opens the transaction UI (`BuyItemUI`).

### 3.1. Complete Pricing Table (`GetPrice`)

The `GetPrice(ShipItem item)` method calculates item values in the regional currency (`parentRegion.portRegion`):

| Condition / Item Type | Criteria | Evaluated Price Formula |
|---|---|---|
| **Player Selling Item** (`item.sold == true`) |
| `ShipItemLanternFuel` | Contains oil bottle and is not full | **12** |
| `ShipItemBottle` | Capacity `≥ 5` and `< 30` (empty/partial bottle) | **2** |
| `ShipItemBottle` | Capacity `≥ 30` and `amount == 9f` (full rum/wine cask) | **25** |
| `ShipItemCrate` | Full crate (`amount == maxAmount`) | **80%** of `item.value` (`value * 0.8f`) |
| `ShipItemCrate` | Empty crate (`amount == 0`) | **20** (or **24** if it contained smoked food) |
| `ShipItemSalt` / `ShipItemOakum` | Salt or oakum supplies | `value * (amount / maxAmount) * 0.8f` |
| General Item | Standard retail item | **50%** of `item.value` (`value * 0.5f`) |
| **Player Buying Item** (`item.sold == false`) |
| General Retail Item | Displayed on shop counter | `value * priceMult - (value * priceMult * retailDiscounts[region])` |
| Cookable Food (`CookableFood`) | Displayed on counter (raw/smoked) | Base 75% of `value` (×1.2 if `amount ≥ 1`), multiplied by `priceMult` minus reputation discount |
| Bulk Goods (`IsBulk()`) | Trade cargo barrels/crates | `GetBulkBuyPrice()`: market base + demand markup (`1f + demand * 0.25f`) minus bulk reputation discount |

> **Counter Multiplier (`priceMult`):** The retail price multiplier is retrieved from the `ShopItemSpawner.priceMult` component attached to the parent shop display stand.

---

## 4. Quest Giver (`QuestDude`)

`QuestDude` manages story quests (distinct from cargo missions) using a state machine indexed in `Quests.instance.currentQuests[questIndex]`.

```csharp
// Trigger entry (OnTriggerEnter)
if (other.CompareTag("Player"))
{
    int status = Quests.instance.currentQuests[quest.questIndex];
    if (status == 0)      ShowDialog(0);  // Start dialogue
    else if (status == -1) ShowDialog(-1); // Quest in progress
    else                   ShowDialog(-5); // Quest completed
}
```

### 4.1. Dialogue Formatting and Wrapping Width
NPC dialogue text (`questLines[]`) is wrapped automatically by `Wrap(string v, int size)` at a line width of **45 characters**.

### 4.2. Quest Acceptance and Turn-In
- **Acceptance:** Clicking through all dialogue lines transitions the state to `-1`. If the quest specifies `acceptPrefabIndex != 0`, a starter quest item spawns near the dialogue button with `sold = true`.
- **Turn-In:** When the player brings an item matching `questItemIndex == quest.deliveredQuestItemIndex` into the `QuestDude` trigger, the NPC shows the turn-in dialogue (`ShowDialog(-3)`). Confirming destroys the item (`DestroyItem()`), awards **Gold Lions** (`PlayerGold.currency[3] += quest.goldReward`), and sets state to `-5` (completed).

---

## 5. Port Mission Clerk and Porter (`PortDude` & `CargoTransportDude`)

### 5.1. `PortDude` (Port Clerk)
Performs two key roles at the mission desk:
1. **Cargo Acceptance (`OnTriggerEnter`):** Checks colliders tagged `"Good"`. If the item belongs to a mission where `destinationPort == port`, `good.Deliver()` is invoked. If the port does not match, a notification reads: `"You are at the wrong port!"`.
2. **UI Access (`ActivateMissionListUI`):**
   - When `openEconomyUI == false`, opens the port mission list (`MissionListUI`).
   - When `openEconomyUI == true`, verifies player reputation (`PlayerReputation.GetRepLevel(port.region) >= 1`). Lacking level 1 reputation blocks market access with `"Not enough reputation"`.

### 5.2. `CargoTransportDude` (Porter)
A stationary NPC for renting cargo carriers. Entering their trigger invokes `CargoCarrierUI.instance.ShowUI(transform, carrierIndex)`.

---

## 6. Tavern Rumor Informant (`TavernRumorsDude`)

`TavernRumorsDude` generates dynamic rumors about trade ships and prices ([Note 78](78-dialogues-rumors-bribery-and-contraband-customs.md)).

- **Billboarding:** The tavern speech bubble rotates to face the player's camera mirror in `Update()`: `speechUI.transform.rotation = Refs.observerMirror.transform.rotation`.
- **Beverage Validation:** Requires a player-owned alcoholic bottle (`amount > 1f && capacity < 30f && sold && health >= capacity`).
- **Text Wrapping Width:** Unlike `QuestDude` (45 characters), `TavernRumorsDude` wraps rumor text at **33 characters** (`Wrap(str, 33)`).

---

## 7. Practical Modding Conclusions

1. **Creating Walking NPCs from Scratch:** Do not look for an existing NavMesh or human walking AI in the game—none exists. To add walking townsfolk or patrols, you must:
   - Generate your own NavMesh or waypoint graph.
   - Implement custom movement (or resurrect the abandoned `Shopkeeper.WalkTo` coroutine via Harmony/reflection and call it from `Update`).
2. **Animating Modded NPCs:** Do not import heavy rigged FBX meshes requiring `SkinnedMeshRenderer`. Split the NPC model into separate Transform parts (head, torso, shoulders, feet), add `NPCAnimations`, and assign `breatheParts` (for oscillation) and `lockParts` (to keep feet planted).
3. **Creating Custom Merchants:** To create a custom shop, place a `Shopkeeper` and link it to a `Region`, `IslandEconomy`, and `ShopArea`. To override pricing or make specific loot sell for more, patch `Shopkeeper.GetPrice(ShipItem)`.
4. **Data-Driven Quests:** New dialogue quests can be added without custom logic by populating `Quest` serializable data and adding it to `Quests.instance.currentQuests`, since `QuestDude` universally handles the `0 -> -1 -> -5` state flow.
5. **Dialogue Box Widths:** When translating or modifying text, account for hardcoded string wrapping limits: **45 characters** for `QuestDude` and **33 characters** for `TavernRumorsDude`.
