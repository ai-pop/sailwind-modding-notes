# 25. Sleep, Bed, and Time Acceleration

Analysis of sleep and rest mechanics. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Sleep is closely tied to needs (note 12), time (note 18), and saves (note 11).

## `Sleep` (singleton `Sleep.instance`)

Manages falling asleep, sleeping, and waking. In `Awake()`: `Time.fixedDeltaTime = 0.02222`, `Time.timeScale = 1`, `sleepTimeStep = fixedDeltaTime * 10`, `sleepTimescale = 16`.

### Key State Flags (`GameState`)
| Flag | Content |
|------|-----------|
| `GameState.inBed` (`Transform`) | Bed the player is in (null = not in bed). |
| `GameState.sleeping` | Whether sleeping. |
| `GameState.eyesFullyClosed` | Eyes fully closed (can be woken). |
| `GameState.justWokeUp` | Just woke up (timer 2 s). |
| `GameState.sleepingInTavern` | Sleeping in tavern (special mode). |
| `Sleep.timeskipSleep` (static) | Accelerated sleep (see below). |

## Entering Bed (`EnterBed`)
1. **Save on sleep:** if `Settings.autosaveInterval >= 0 && SaveLoadManager.enableSaveOnSleep` → `SaveGame(compressed: true)`. (Sleep is an autosave point.)
2. `GameState.inBed = bed`, bed collider disabled.
3. Player control, `MouseLook`, `observerMirror` disabled.
4. `sleepCooldown = Random(3, 6)`, `currentBedDuration = 0`.

## Automatic Falling Asleep
In bed, player **automatically falls asleep** (`FallAsleep`) if:
- `PlayerNeeds.sleep < 99` and not already sleeping, and `sleepCooldown <= 0`;
- `PlayerNeeds.water > 10` and `PlayerNeeds.foodDebt > 10` (cannot sleep from thirst/hunger).

## Falling Asleep (`FallAsleep`)
1. Closes needs UI, disables `MouseLook`, `GameState.sleeping = true`.
2. **Timeskip sleep** (`timeskipSleep = true`) enabled only if **boat is moored** (`CurrentBoatIsMoored`) **or** player is in tavern. Otherwise sleep "in place" without time warp.
3. Disables controls, starts blackout `Blackout.FadeTo(1, 3 s)` and coroutine `StartSleepTimeWarp`.

### Time Acceleration During Sleep (`StartSleepTimeWarp`)
After 3 s from falling asleep:
- `Time.fixedDeltaTime = sleepTimeStep` (×10), `Time.timeScale = sleepTimescale` (**×16**);
- `GameState.eyesFullyClosed = true`.

Additionally, `Sun.Update` when `Sleep.timeskipSleep` sets `timescale = initialTimescale * 9` — in-game day runs ~9× faster (note 18). Overall, sleep significantly accelerates time.

## Waking Up (`WakeUp`)
Only possible when `eyesFullyClosed` (otherwise "still falling asleep"). Actions:
- Re-enables ocean renderer, `sleeping = false`, `eyesFullyClosed = false`, `justWokeUp = true` (for 2 s), `sleepingInTavern = false`, `timeskipSleep = false`.
- Blackout `Blackout.FadeTo(0, 5.05 s)`, restore `Time.timeScale = 1`, `fixedDeltaTime = initial`.
- `sleepCooldown = Random(5.56, 7)`.

### Automatic Wake-Up
- **Normal sleep:** when `currentSleepDuration > 4.5` (and not recovery).
- **Tavern sleep:** wakes between `localTime 7:00–10:00`, if `currentSleepDuration > 3.3` (sleeps until morning).
- **By needs:** when `PlayerNeeds.sleep >= 99.99` and `currentSleepDuration > 0.2` → `WakeUp`.

## Leaving Bed (`LeaveBed`)
On any button press (`AnyButtonDown`: OVR buttons or `InputName 8/9`) with `currentBedDuration > 1.01` and not sleeping:
- Enables bed collider, `inBed = null`, `MouseLook`, player control, `observerMirror`; calls `WakeUp()`.

## Interaction with Other Systems
- **Recovery** (`Recovery`, note 12): blackout triggers `Sleep.instance.FallAsleep()` and uses `recoveryText` for messages.
- **Needs** (note 12): during sleep, `sleep` grows (×4 in tavern), `food/water/protein/vitamins` restore (in tavern).
- **Saves** (note 11): `enableSaveOnSleep` → save on entering bed; autosave doesn't run while `sleeping`/`recovering`.
- **Time** (note 18): `timeskipSleep` accelerates `Sun.timescale` ×9.

## Practical Takeaways for Modding

1. **Sleep = save point** (`enableSaveOnSleep`). Want a save on event — trigger `Sleep.instance.EnterBed`/`FallAsleep`.
2. **Timeskip sleep** only works when moored or in tavern (`timeskipSleep`); at open sea, sleep without time warp.
3. **Time acceleration** during sleep: `Time.timeScale = 16` + `Sun.timescale × 9` — account for this in your timers.
4. **Cannot fall asleep** when `water <= 10` or `foodDebt <= 10` (thirst/hunger).
5. **Wake-up** when `sleep >= 99.99`, by duration (>4.5 s), in tavern — by morning time (7–10).
6. `GameState.eyesFullyClosed` gates wake-up and is used by `Recovery`/`ShipItem.ProcessSaveable` (cargo loss, note 16).
7. Bed exit buttons — `InputName 8/9` (primary/alt action).
