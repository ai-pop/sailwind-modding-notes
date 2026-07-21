# 25. Сон, кровать и ускорение времени

Разбор механики сна и отдыха. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Сон тесно связан с потребностями (заметка 12), временем (заметка 18) и сохранениями (заметка 11).

## `Sleep` (синглтон `Sleep.instance`)

Управляет засыпанием, сном и пробуждением. В `Awake()`: `Time.fixedDeltaTime = 0.02222`, `Time.timeScale = 1`, `sleepTimeStep = fixedDeltaTime * 10`, `sleepTimescale = 16`.

### Ключевые флаги состояния (`GameState`)
| Флаг | Содержание |
|------|-----------|
| `GameState.inBed` (`Transform`) | Кровать, в которой игрок (null = не в кровати). |
| `GameState.sleeping` | Спит ли. |
| `GameState.eyesFullyClosed` | Глаза полностью закрыты ( можно будить). |
| `GameState.justWokeUp` | Только что проснулся (таймер 2 c). |
| `GameState.sleepingInTavern` | Сон в таверне (особый режим). |
| `Sleep.timeskipSleep` (static) | Ускоренный сон (см. ниже). |

## Вход в кровать (`EnterBed`)
1. **Сохранение при сне:** если `Settings.autosaveInterval >= 0 && SaveLoadManager.enableSaveOnSleep` → `SaveGame(compressed: true)`. (Сон — точка автосохранения.)
2. `GameState.inBed = bed`, коллайдер кровати отключается.
3. Отключаются управление игроком, `MouseLook`, `observerMirror`.
4. `sleepCooldown = Random(3, 6)`, `currentBedDuration = 0`.

## Автоматическое засыпание
Находясь в кровати, игрок **автоматически засыпает** (`FallAsleep`), если:
- `PlayerNeeds.sleep < 99` и ещё не спит, и `sleepCooldown <= 0`;
- `PlayerNeeds.water > 10` и `PlayerNeeds.foodDebt > 10` (нельзя уснуть от жажды/голода).

## Засыпание (`FallAsleep`)
1. Закрывает UI потребностей, выключает `MouseLook`, `GameState.sleeping = true`.
2. **Timeskip-сон** (`timeskipSleep = true`) включается, только если **лодка пришвартована** (`CurrentBoatIsMoored`) **или** игрок в таверне. Иначе сон «на месте» без перемотки.
3. Отключает управление, запускает блэкаут `Blackout.FadeTo(1, 3 c)` и корутину `StartSleepTimeWarp`.

### Ускорение времени во сне (`StartSleepTimeWarp`)
Через 3 c после засыпания:
- `Time.fixedDeltaTime = sleepTimeStep` (×10), `Time.timeScale = sleepTimescale` (**×16**);
- `GameState.eyesFullyClosed = true`.

Дополнительно `Sun.Update` при `Sleep.timeskipSleep` ставит `timescale = initialTimescale * 9` — игровой день крутится в ~9 раз быстрее (заметка 18). Итого сон сильно ускоряет время.

## Пробуждение (`WakeUp`)
Возможно только когда `eyesFullyClosed` (иначе «ещё засыпает»). Действия:
- Перевключает рендерер океана, `sleeping = false`, `eyesFullyClosed = false`, `justWokeUp = true` (на 2 c), `sleepingInTavern = false`, `timeskipSleep = false`.
- Блэкаут `Blackout.FadeTo(0, 5.05 c)`, возврат `Time.timeScale = 1`, `fixedDeltaTime = initial`.
- `sleepCooldown = Random(5.56, 7)`.

### Автоматическое пробуждение
- **Обычный сон:** когда `currentSleepDuration > 4.5` (и не восстановление).
- **Сон в таверне:** просыпается между `localTime 7:00–10:00`, если `currentSleepDuration > 3.3` (т.е. спит до утра).
- **По потребностям:** когда `PlayerNeeds.sleep >= 99.99` и `currentSleepDuration > 0.2` → `WakeUp`.

## Выход из кровати (`LeaveBed`)
По нажатию любой кнопки (`AnyButtonDown`: OVR-кнопки или `InputName 8/9`) при `currentBedDuration > 1.01` и не во сне:
- Включает коллайдер кровати, `inBed = null`, `MouseLook`, управление игроком, `observerMirror`; вызывает `WakeUp()`.

## Взаимодействие с другими системами
- **Восстановление** (`Recovery`, заметка 12): обморок вызывает `Sleep.instance.FallAsleep()` и использует `recoveryText` для сообщений.
- **Потребности** (заметка 12): во сне `sleep` растёт (×4 в таверне), восстанавливаются `food/water/protein/vitamins` (в таверне).
- **Сохранения** (заметка 11): `enableSaveOnSleep` → сейв при входе в кровать; автосейв не идёт пока `sleeping`/`recovering`.
- **Время** (заметка 18): `timeskipSleep` ускоряет `Sun.timescale` ×9.

## Практические выводы для мододела

1. **Сон = точка сохранения** (`enableSaveOnSleep`). Хотите сейв по событию — триггерьте `Sleep.instance.EnterBed`/`FallAsleep`.
2. **Timeskip-сон** работает только пришвартованным или в таверне (`timeskipSleep`); в открытом море сон без перемотки времени.
3. **Ускорение времени** во сне: `Time.timeScale = 16` + `Sun.timescale × 9` — учитывайте для своих таймеров.
4. **Нельзя уснуть** при `water <= 10` или `foodDebt <= 10` (жажда/голод).
5. **Пробуждение** по заполнению `sleep >= 99.99`, по длительности (>4.5 c), в таверне — по утреннему времени (7–10).
6. `GameState.eyesFullyClosed` гейтит пробуждение и используется `Recovery`/`ShipItem.ProcessSaveable` (потеря груза, заметка 16).
7. Кнопки выхода из кровати — `InputName 8/9` (основное/альт действие).
