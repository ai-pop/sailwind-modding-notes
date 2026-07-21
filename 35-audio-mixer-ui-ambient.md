# 35. Звук: микшер, UI-звуки, амбиент

Разбор звуковой подсистемы. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Контекстный микшер (`AudioMixers`)

Синглтон `AudioMixers.instance` держит **снапшоты** `AudioMixer`, переключающие общий микс звука по контексту:

| Снапшот | Когда |
|---------|-------|
| `outdoorSnapshot` | На улице. |
| `indoorSnapshot` | В помещении. |
| `semiIndoorSnapshot` | Полузакрытое пространство. |
| `gameActiveSnapshot` | Активная игра. |
| `gameSleepSnapshot` | Сон (приглушение). |

> **Для мода:** можно дёргать `AudioMixers.instance.indoorSnapshot.TransitionTo(...)` для своих сцен/помещений.

## UI-звуки (`UISoundPlayer`, `UISounds`)

`UISoundPlayer.instance` — проигрыватель интерфейсных звуков с **пулом `AudioSource`** (`GetAudioSource` берёт первый незанятый источник).

### `UISounds` (enum, 17 значений)
`buttonClick, buttonHover, buttonBack, itemPickup, itemInventoryIn, itemInventoryOut, smallPop, crateOpen, crateClose, crateSealBreak, winchClick, winchUnclick, itemDrop, unused, oakum, liquid, write`

### Основной метод
```csharp
UISoundPlayer.instance.PlayUISound(UISounds sound, float volume, float pitch);
```
Плюс удобные обёртки: `PlayUIClickSound()`, `PlaySmallItemDropSound()`, `PlayBigItemDropSound()`, `PlayParchmentSound()`, `PlayWritingSound()`, `PlayLiquidPourSound()`, `PlayGoldSound()`, `PlayOpenSound()`, `PlayCloseSound()`.

Клипы лежат в массивах: `uiSounds[]` (индексируется `UISounds`), `parchmentSounds[]`, `writingSounds[]`, отдельные `gold`, `liquidPour`, `parchmentOpen/Close`.

> **Для мода:** существующие UI-звуки играются одной строкой `PlayUISound(...)`. Для **своего** звука проще завести собственный `AudioSource` (игровой пул ограничен) и грузить клип через AssetBundle.

## Амбиент по времени суток (`AmbientSound`)

Компонент на `AudioSource`, играющий звук **только в окне времени** (по `Sun.sun.localTime`):

- `startHour` / `endHour` — границы (случайный джиттер при старте: start `+[-0.1, 1]`, end `+[-0.1, 0.1]` — чтобы не все точки звучали синхронно).
- `CheckIfWithinPlayHours()`: если `localTime` в окне → играть; иначе — гасить. Поддерживает переход через полночь (wrap).
- Плавное нарастание/затухание громкости (`± dt * volume * 0.12`).

Применение: птицы днём, сверчки ночью, прибой и т.п.

> **Для мода:** паттерн «звук в окне времени» = `AmbientSound`-компонент + `startHour/endHour`. Свой амбиент, привязанный к погоде/шторму, строится аналогично (условие вместо времени).

## Рассинхронизация зацикленных звуков (`AudioRandomDelay`)

Простой трюк: в `Awake()` останавливает источник и стартует со случайной задержкой `Random(0, clip.length)`, чтобы одинаковые зацикленные амбиенты **не стартовали в фазе** (иначе слышен «хор»). Полезно копировать для своих петлевых звуков.

## Глобальная громкость (`Settings`)

`Settings.ApplyVolume()` (заметка 12):
```
AudioListener.volume = muteSound || currentlyLoading ? 0 : soundVolume / 20f
```
`Settings.soundVolume` (0..20), `muteSound`. Общая громкость игры — через `AudioListener.volume`.

## 👁 Взгляд моддера: «хочу добавить свои звуки»

| Задача | Как |
|--------|-----|
| Проиграть штатный UI-звук | `UISoundPlayer.instance.PlayUISound(UISounds.X, vol, pitch)` |
| Свой звук (разовый) | Свой `GameObject` + `AudioSource`, клип из AssetBundle, `PlayOneShot` |
| Свой амбиент по времени | Компонент по образцу `AmbientSound` (окно `startHour/endHour` + `Sun.localTime`) |
| Свой амбиент по событию | Подписка на `Sun.OnNewDay`/свою логику + `AudioSource` |
| Петли без «хора» | Рассинхронизация по образцу `AudioRandomDelay` (случайный старт) |
| Смена микса (помещение/сон) | `AudioMixers.instance.<snapshot>.TransitionTo(seconds)` |
| Уважать громкость игры | Не глушить `AudioListener.volume`; для своей музыки — отдельный канал/`AudioMixerGroup` |

## Практические выводы

1. **UI-звуки** централизованы в `UISoundPlayer` (пул источников, 17 типов `UISounds`); играются одной строкой.
2. **Контекст** (улица/помещение/сон) переключается снапшотами `AudioMixers`.
3. **Амбиент** привязан к `Sun.localTime` через `AmbientSound` (окно часов + фейд); зацикленные звуки рассинхронизируются `AudioRandomDelay`.
4. **Громкость** игры = `AudioListener.volume = soundVolume/20` (`Settings.ApplyVolume`).
5. Для своих звуков — свой `AudioSource` + AssetBundle-клип; штатные — через `PlayUISound`.
