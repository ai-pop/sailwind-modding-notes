# 35. Audio: Mixer, UI Sounds, Ambient

Analysis of the audio subsystem. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## Contextual Mixer (`AudioMixers`)

Singleton `AudioMixers.instance` holds **snapshots** of `AudioMixer`, switching the overall audio mix by context:

| Snapshot | When |
|---------|-------|
| `outdoorSnapshot` | Outside. |
| `indoorSnapshot` | Inside building. |
| `semiIndoorSnapshot` | Semi-enclosed space. |
| `gameActiveSnapshot` | Active gameplay. |
| `gameSleepSnapshot` | Sleep (muted). |

> **For modding:** you can call `AudioMixers.instance.indoorSnapshot.TransitionTo(...)` for your own scenes/rooms.

## UI Sounds (`UISoundPlayer`, `UISounds`)

`UISoundPlayer.instance` — UI sound player with **`AudioSource` pool** (`GetAudioSource` takes first free source).

### `UISounds` (enum, 17 values)
`buttonClick, buttonHover, buttonBack, itemPickup, itemInventoryIn, itemInventoryOut, smallPop, crateOpen, crateClose, crateSealBreak, winchClick, winchUnclick, itemDrop, unused, oakum, liquid, write`

### Main Method
```csharp
UISoundPlayer.instance.PlayUISound(UISounds sound, float volume, float pitch);
```
Plus convenience wrappers: `PlayUIClickSound()`, `PlaySmallItemDropSound()`, `PlayBigItemDropSound()`, `PlayParchmentSound()`, `PlayWritingSound()`, `PlayLiquidPourSound()`, `PlayGoldSound()`, `PlayOpenSound()`, `PlayCloseSound()`.

Clips stored in arrays: `uiSounds[]` (indexed by `UISounds`), `parchmentSounds[]`, `writingSounds[]`, separate `gold`, `liquidPour`, `parchmentOpen/Close`.

> **For modding:** existing UI sounds are played with a single `PlayUISound(...)` call. For **your own** sound, easiest to create your own `AudioSource` (game pool is limited) and load clip via AssetBundle.

## Time-of-Day Ambient (`AmbientSound`)

Component on `AudioSource`, playing sound **only within a time window** (by `Sun.sun.localTime`):

- `startHour` / `endHour` — boundaries (random jitter on start: start `+[-0.1, 1]`, end `+[-0.1, 0.1]` — so not all points sound synchronized).
- `CheckIfWithinPlayHours()`: if `localTime` in window → play; otherwise — fade out. Supports midnight wrap.
- Smooth volume fade in/out (`± dt * volume * 0.12`).

Application: birds during day, crickets at night, surf, etc.

> **For modding:** pattern "sound in time window" = `AmbientSound` component + `startHour/endHour`. Custom ambient tied to weather/storm built similarly (condition instead of time).

## Loop Sound Desynchronization (`AudioRandomDelay`)

Simple trick: in `Awake()` stops source and starts with random delay `Random(0, clip.length)`, so identical looped ambients **don't start in phase** (otherwise you hear a "chorus"). Useful to copy for your own looped sounds.

## Global Volume (`Settings`)

`Settings.ApplyVolume()` (note 12):
```
AudioListener.volume = muteSound || currentlyLoading ? 0 : soundVolume / 20f
```
`Settings.soundVolume` (0..20), `muteSound`. Overall game volume — via `AudioListener.volume`.

## 👁 Modder's View: "I want to add my own sounds"

| Task | How |
|--------|-----|
| Play stock UI sound | `UISoundPlayer.instance.PlayUISound(UISounds.X, vol, pitch)` |
| Custom sound (one-shot) | Own `GameObject` + `AudioSource`, clip from AssetBundle, `PlayOneShot` |
| Custom time-based ambient | Component modeled on `AmbientSound` (window `startHour/endHour` + `Sun.localTime`) |
| Custom event-based ambient | Subscribe to `Sun.OnNewDay`/own logic + `AudioSource` |
| Loops without "chorus" | Desynchronization modeled on `AudioRandomDelay` (random start) |
| Switch mix (room/sleep) | `AudioMixers.instance.<snapshot>.TransitionTo(seconds)` |
| Respect game volume | Don't override `AudioListener.volume`; for your music — separate channel/`AudioMixerGroup` |

## Practical Takeaways

1. **UI sounds** centralized in `UISoundPlayer` (source pool, 17 `UISounds` types); played with one call.
2. **Context** (outdoor/indoor/sleep) switches via `AudioMixers` snapshots.
3. **Ambient** tied to `Sun.localTime` via `AmbientSound` (hour window + fade); looped sounds desynchronized by `AudioRandomDelay`.
4. **Game volume** = `AudioListener.volume = soundVolume/20` (`Settings.ApplyVolume`).
5. For custom sounds — own `AudioSource` + AssetBundle clip; stock sounds — via `PlayUISound`.
