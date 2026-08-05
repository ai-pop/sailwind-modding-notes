# 78. Dialogues, Rumors, Procedural NPC Animations, and Customs/Contraband Bribery System

A comprehensive technical breakdown of player-NPC interaction systems, procedural character animation, tavern rumor generation, and customs/contraband inspection mechanics in Sailwind v0.38 (`Assembly-CSharp.dll`). This note supplements story quests ([Note 27](27-story-quests.md)) and world population ([Note 20](20-npcs-world-population.md)).

---

## 1. Taverns and Rumor Generation (`TavernRumorsDude` and `PortRumors`)

Every tavern houses an informant NPC (`TavernRumorsDude : MonoBehaviour, IGPButton`) who shares rumors exclusively in exchange for alcohol.

### 1.1. Activation and Beverage Validation (`OnTriggerEnter`)

For the "Offer Drink" button (`drinkButton`) to appear, an item entering the trigger must satisfy strict validation criteria:

```csharp
ShipItemBottle bottle = other.GetComponent<ShipItemBottle>();
if (bottle != null && bottle.amount > 1f && bottle.GetCapacity() < 30f &&
    bottle.sold && bottle.health >= bottle.GetCapacity())
{
    currentDrink = bottle;
    drinkButton.SetActive(true);
}
```

| Condition | Description |
|---|---|
| `bottle.amount > 1f` | The bottle must contain more than 1 liquid unit. |
| `bottle.GetCapacity() < 30f` | Capacity must be under 30 (excluding large water barrels; only wine/rum/ale bottles qualify). |
| `bottle.sold == true` | The bottle must be owned by the player (preventing players from offering unpaid shop items). |
| `bottle.health >= capacity` | The bottle must be undamaged/unbroken. |

### 1.2. Rumor Level and Drink Quality (`ClickDrinkButton`)

When the drink button is clicked, the beverage is **destroyed** (`currentDrink.DestroyItem()`), and the quality of the revealed intelligence depends on the drink's volume/type:

```csharp
int level = 0;
if (currentDrink.amount < 5f)
    level = 2; // Expensive/concentrated beverage -> detailed rumor
```

With a 50% probability (or if `specialRumors` is empty), the port's economic rumor generator is invoked: `PortRumors.GenerateRumorText(level)`. Otherwise, a random flavor string from `specialRumors[]` is displayed.

> **Text Wrapping Width:** `TavernRumorsDude.Wrap(text, 33)` wraps lines at **33 characters**, whereas `QuestDude` dialogs wrap at **45 characters**.

---

## 2. Linking Rumors to Real Economy (`PortRumors`)

Tavern rumors are not static filler; they represent **live economic intelligence** regarding `TraderBoat` movements ([Note 77](77-npc-ai-navigation-collision-avoidance-and-waypoint-graphs.md)).

```csharp
public Rumor GenerateRumor(int level)
{
    List<TraderBoat> boats = GetDepartingBoats();
    if (boats.Count <= 0)
        boats = GetIncomingBoats();
    if (boats.Count <= 0)
        return default(Rumor);

    TraderBoat boat = boats[Random.Range(0, boats.Count)];
    Vector2 topGood = FindHighestGoodCount(boat); // .x = good ID, .y = count
    ...
    result.mainGood = Mathf.RoundToInt(topGood.x);
    result.mainGoodCount = Mathf.RoundToInt(topGood.y);
    result.origin = boat.GetLastMarket();
    result.destination = boat.GetCurrentDestination();
    result.rumorLevel = level;
    return result;
}
```

### Impact of Rumor Level (`level`) on Text
In `GenerateRumorText(level)`, cargo quantities are obscured or revealed based on `level`:

| Rumor Level (`level`) | Quantity Condition | Formatted Rumor Text |
|:--:|---|---|
| `0` (cheap drink) | Any quantity | `"carrying some {GoodName}."` |
| `2` (expensive drink) | `mainGoodCount > 12` | `"carrying a large load of {GoodName}."` |
| `2` (expensive drink) | `mainGoodCount <= 4` | `"carrying some {GoodName}."` |
| `2` (expensive drink) | `5..12` | `"carrying a sizeable load of {GoodName}."` |

---

## 3. Procedural NPC Animation and IK (`NPCAnimations` & `NPCPlayerCol`)

Sailwind NPCs do not use Unity skeletal animation (`Animator` / `SkinnedMeshRenderer`). Character motion is animated procedurally via `NPCAnimations`.

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

### 3.1. Procedural Breathing and Swaying
Each frame, `currentTime` oscillates between `0` and `breatheDuration`. For each bone in `breatheParts[]`, a rotation angle is evaluated:

```csharp
float num = QuadraticInOut(currentTime, 0f, breatheAngles[i], breatheDuration);
breatheParts[i].localRotation = baseRots[i];
breatheParts[i].Rotate(Vector3.forward, num, Space.Self);
```

Where `QuadraticInOut` is a quadratic easing curve.

### 3.2. Ground Locking (`lockParts`)
To prevent NPC feet from sliding across the ground while the torso sways, the component caches initial world positions and forces them back every frame:

```csharp
for (int j = 0; j < lockParts.Length; j++)
{
    Vector3 worldPos = transform.TransformPoint(basePositions[j]);
    lockParts[j].localPosition = lockParts[j].parent.InverseTransformPoint(worldPos);
}
```

### 3.3. Dynamic Head Look IK (`headLookAngle`)
A child trigger `NPCPlayerCol` sets `playerInRange = true` when the player approaches. If the camera lies within the forward cone defined by `headLookAngle` (default **30°**):

```csharp
Quaternion targetLook = Quaternion.LookRotation(transform.position - Camera.main.transform.position, Vector3.up);
if (Quaternion.Angle(transform.rotation, targetLook) > 180f - headLookAngle)
    head.rotation = Quaternion.Slerp(head.rotation, Quaternion.LookRotation(Camera.main.transform.position - head.position, Vector3.up), Time.deltaTime * 3f);
else
    head.localRotation = Quaternion.Slerp(head.localRotation, headInitialRot, Time.deltaTime * 7f);
```

The head smoothly rotates toward the player camera at `3 Hz`, returning to `headInitialRot` at `7 Hz` when the player exits the field of view.

---

## 4. Customs, Contraband, and Bribery (`QuestItemDetector`)

Ports feature an inspection trigger (`QuestItemDetector`) that checks for contraband cargo.

### 4.1. Patrol Time Windows
The customs collider is enabled only during designated local solar hours:

```csharp
public void Update()
{
    col.enabled = (Sun.sun.localTime >= activeFrom && Sun.sun.localTime <= activeUntil);
}
```

### 4.2. Seizure and Bribery Waterfall (`OnTriggerEnter`)
If an item matching `item.GetPrefabIndex() == itemPrefabIndex` (contraband) enters the active trigger, it is immediately destroyed (`DestroyItem()`), triggering a bribery/penalty waterfall:

```
[Contraband Detected: DestroyItem()]
                 │
                 ▼
     Player has >= 5 Gold Lions?
     ├── YES ──► Deduct 5 Gold Lions
     │           "You paid a 5 Gold Lions bribe..."
     ▼ NO
 Player has >= 200 Al'Ankh Lions?
     ├── YES ──► Deduct 200 Al'Ankh Lions
     │           "You paid a 200 Al'Ankh Lions bribe..."
     ▼ NO
 [Reputation Reset in Al'Ankh]
 PlayerReputation.ChangeReputation(-99999999, PortRegion.alankh)
 "Failed to bribe the guards. Your reputation in Al'Ankh has been reset."
```

---

## 5. Time-Gated Quest Drop-Offs (`QuestWaypoint`)

Quest delivery waypoints (`QuestWaypoint`) can restrict deliveries during day or night windows:

```csharp
if (Sun.sun.localTime >= unavailableFrom && Sun.sun.localTime <= unavailableTo)
{
    NotificationUi.instance.ShowNotification("Come back at night!");
    return;
}
```

Upon valid delivery (`questItemIndex` matches):
1. The delivered item is destroyed.
2. A 9-second timer begins (`timer = 9f`).
3. After **9 seconds**, a reward or subsequent quest item is instantiated from `PrefabsDirectory.instance.directory[spawnedItemPrefab]` with `sold = true` and registered with the save system (`RegisterToSave()`).

---

## 6. Practical Modding Conclusions

1. **Creating Custom Taverns/Rumors:** Use `TavernRumorsDude` and `PortRumors` to add custom informants. Ensure drinks have `amount < 5f` to trigger `level = 2` (revealing exact cargo volumes); otherwise, only generic quantities are reported.
2. **Procedural Animation for Modded NPCs:** Avoid complex rigged FBX models requiring `SkinnedMeshRenderer`. Use simple Transform hierarchies with `NPCAnimations`, assigning body parts to `breatheParts` (for oscillation) and `lockParts` (to keep feet planted).
3. **Bypassing Customs:** Contraband mods can exploit the "night window" when `Sun.sun.localTime` falls outside `[activeFrom, activeUntil]`, leaving `QuestItemDetector.col.enabled == false`.
4. **Hardcoded Bribery Currencies:** Fines in `QuestItemDetector` are hardcoded to **Gold Lions (5)** or **Al'Ankh Lions (200)**. Lacking both resets player reputation in Al'Ankh (`-99999999`) regardless of which region the port is located in.
5. **Reward Spawn Delay:** Account for the 9-second delay (`timer = 9f`) in `QuestWaypoint`; reward items do not spawn instantly upon turning in a quest item.
