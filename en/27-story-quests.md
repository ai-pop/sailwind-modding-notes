# 27. Story Quests (NPC Dialogues)

Analysis of the story quest system — **separate** from cargo missions (note 15). These are dialogue quests: NPC gives task via dialogue lines, player accepts, brings item, receives reward. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## `Quest` (`[Serializable]`)

Quest description (dialogue data):

| Field | Content |
|------|-----------|
| `questIndex` | Quest index (key in `Quests.currentQuests`). |
| `goldReward` | Gold reward (in **Gold Lions**, currency 3). |
| `questLines[]` | NPC dialogue lines (introductory dialogue, paginated). |
| `playerResponses[]` / `playerAltResponses[]` | Player response options (button). |
| `acceptPrefabIndex` | Item given when accepting quest (index in `PrefabsDirectory.directory`; 0 = none). |
| `deliveredQuestItemIndex` | Index of item to bring for completion. |
| `inProgressLine` / `inProgressResponse` | Line/response "quest still in progress". |
| `playerRepeatQuestion` | Repeat question button. |
| `completionLine` / `playerCompletionResponse` | Line/response on completion. |
| `afterCompletedLine` | Line after completion. |

## Quest States (`Quests.currentQuests[]`)

Singleton `Quests.instance`, `currentQuests` array (saved to save as `quests`). State codes:

| Code | State | `QuestDude` Behavior |
|:--:|-----------|----------------------|
| `0` (and `>=0`) | Available / in dialogue | Shows `questLines[]`, button leads further. |
| `-1` | Accepted, in progress | `inProgressLine` / `inProgressResponse`. |
| `-5` | Completed | `afterCompletedLine`, button `"(bye)"`. |

## Quest-Giver NPC (`QuestDude`, implements `IGPButton`)

### Trigger Entry (`OnTriggerEnter`)
- **Player** (tag `Player`, no item in trigger) → shows `dialogUI`, sound `smallPop`, and by quest state:
  - `== 0` → `ShowDialog(0)` (introduction);
  - `== -1` → `ShowDialog(-1)` (in progress);
  - otherwise (`-5`) → `ShowDialog(-5)` (after completion).
- **Quest item** (`QuestItem` with `questItemIndex == quest.deliveredQuestItemIndex`) → `ShowDialog(-3)` (turn-in).

### `ShowDialog(index)`
- `index >= 0` → `questLines[index]` + button `playerResponses[index]`.
- `-1` → `inProgressLine` + `playerRepeatQuestion`.
- `-2` → `inProgressResponse` + `playerRepeatQuestion`.
- `-3` → `completionLine` + `playerCompletionResponse`.
- `-5` (default) → `afterCompletedLine` + `"(bye)"`.

NPC text is word-wrapped with **45 character** width (`Wrap`).

### Dialogue Progress (`ClickButton`)
1. If quest item is in trigger (`cargoInTrigger`) → **`CompleteQuest`** → `ShowDialog(-5)`.
2. Otherwise if state `-1` → `ShowDialog(-2)` (reminder).
3. Otherwise if state `>= 0` → next line (`questLinesIndex++`); **on reaching last line:**
   - state → `-1` (quest accepted);
   - if `acceptPrefabIndex != 0` → **quest item spawned** (`sold = true`, `RegisterToSave`) near button.
4. Otherwise → close dialogue.

### Completion (`CompleteQuest`)
- Destroys brought item (`DestroyItem`).
- **Reward:** `PlayerGold.currency[3] += quest.goldReward` (Gold Lions), gold sound.
- Quest state → `-5`.

## Quest Item (`QuestItem`)

Component on `ShipItem`: `questItemIndex`. In `Awake()`, forces `sold = true` (quest item is always "purchased"/player-owned). Turn-in occurs when item with matching `questItemIndex` enters `QuestDude` trigger.

## Practical Takeaways for Modding

1. **Quests are data:** `Quest` contains all lines and parameters; logic is universal in `QuestDude`. Adding quests is possible by populating `Quest` objects and `Quests.currentQuests`.
2. **State machine:** `0` (dialogue) → `-1` (accepted) → `-5` (completed). `currentQuests[questIndex]` — single source of truth.
3. **Reward** always in **Gold Lions** (currency 3), not regional money.
4. **Quest item** (`QuestItem.questItemIndex`) is turned in physically (trigger at NPC), same as cargo missions via `PortDude`.
5. **On acceptance** an item may be given (`acceptPrefabIndex`), spawned near button.
6. NPC dialogue lines — string arrays (wrap at 45 characters); response buttons — separate strings. All hardcoded in quest data (translation — via string substitution, see notes 01/08).
7. `Quests.currentQuests` is saved to save (`quests`, note 11).
