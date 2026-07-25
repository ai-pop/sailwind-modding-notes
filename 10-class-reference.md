# 10. Справочник классов Assembly-CSharp.dll

Краткий справочник классов, существенных для моддинга. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy 8.2.0.

## Текст и UI

### `LookUI` : MonoBehaviour

Класс, отвечающий за отображение подсказок при наведении на интерактивные объекты.

| Поле/метод | Тип | Описание |
|------------|-----|----------|
| `controlsText` | `TextMesh [SerializeField]` | Управляющие подсказки («pick up», «use», «buy») |
| `hintText` | `TextMesh [SerializeField]` | Описание (`button.description`) |
| `extraText` | `TextMesh [SerializeField]` | Название объекта (`button.lookText`) |
| `ShowLookText(GoPointerButton)` | `void` | Основной метод — устанавливает текст в все TextMesh |
| `ShowHoldText(PickupableItem)` | `void` | Подсказки для предмета в руках |
| `ClearText()` | `void` | Очистка всех TextMesh |

### `StartMenuButton` : GoPointerButton

Кнопка главного меню.

| Поле/метод | Тип | Описание |
|------------|-----|----------|
| `type` | `StartMenuButtonType [SerializeField]` | Тип кнопки (NewGame, Continue, Options, Quit, Slot, LoadBackupSave) |
| `SetButtonText(string)` | `void` | Устанавливает текст кнопки через `GetComponent<TextMesh>().text` в дочерних объектах |

### `GPButtonKeybinding` : GoPointerButton

Кнопка назначения клавиши в настройках управления.

| Поле/метод | Тип | Описание |
|------------|-----|----------|
| `inputName` | `string` | Имя действия (сериализуется в префабе) |
| `input2` | `bool` | Альтернативная клавиша |
| `controllerButton` | `bool` | Кнопка контроллера (OVR) |
| `text` | `TextMesh private` | Дочерний TextMesh для отображения клавиши |
| `UpdateText()` | `void` | Записывает `keyCode.ToString()` в TextMesh |
| `OnActivate()` | `override void` | Переход в режим переназначения: `text.text = "(press a key)"` |
| `Update()` | `void` | Ловит следующую нажатую клавишу при активном переназначении |

## Ввод

### `GoPointer` : MonoBehaviour

Основной raycast-указатель игры.

| Поле/метод | Тип | Описание |
|------------|-----|----------|
| `pointedAtButton` | `GoPointerButton private` | Текущий объект под прицелом |
| `hit` | `RaycastHit private` | Результат последнего raycast |
| `DoRaycast()` | `void private` | Physics.Raycast по центру экрана |
| `MainButtonDown()` | `bool public` | ЛКМ нажата (проверяет `!GameState.inCursorMenu`) |
| `AltButtonDown()` | `bool public` | ПКМ нажата |
| `AltButtonHeld()` | `bool private` | ПКМ удерживается |
| `LateUpdate()` | `void private` | Вызывает `DoRaycast()` |

### `GoPointerButton` : MonoBehaviour

Базовый класс всех интерактивных объектов.

| Поле | Тип | Описание |
|------|-----|----------|
| `lookText` | `string public` | Текст при наведении |
| `description` | `string public` | Описание |
| `isLookedAt` | `bool protected` | Наведён ли курсор |
| `OnActivate()` | `virtual void` | Действие при клике |
| `Look(GoPointer)` | `void public` | Регистрация наведения |

Производные классы: `GPButtonRopeWinch`, `GPButtonSteeringWheel`, `GPButtonSailPusher`, `GPButtonBed`, `GPButtonTrapdoor`, `GPButtonDockMooring`, `GPButtonPurchaseBoat`, `GPButtonInventorySlot`, `GPButtonExtraMenus`, `GPButtonPortMissions`, `GPButtonKeybinding`, `StartMenuButton`, `CurrencySwitchButton`, `BoatDamageWaterButton`, `GPButtonBoatPushCol`, `GPButtonControlToggle`, `GPButtonSettingsCheckbo`, `GPButtonResetKeybindings`.

### `MouseLook` : MonoBehaviour

Управление вращением камеры мышью.

| Метод | Тип | Описание |
|------|-----|----------|
| `ToggleMouseLook(bool)` | `static void` | Включить/выключить вращение камеры |
| `ToggleMouseLookAndCursor(bool)` | `static void` | Полное управление: камера + курсор + `GameState.inCursorMenu` |
| `Update()` | `void private` | Вращение камеры по движению мыши |

### `GameInput`

Статический класс управления клавишами.

| Метод | Тип | Описание |
|------|-----|----------|
| `GetKeyCode(InputName, bool, bool)` | `KeyCode static` | Получить клавишу по имени действия |
| `SetKeyMap(InputName, KeyCode, bool, bool)` | `void static` | Установить клавишу |
| `ResetToDefaults()` | `void static` | Сброс к стандартным клавишам |
| `GetControllerKeyString(InputName)` | `string static` | Строковое имя клавиши контроллера |

## Состояние игры

### `GameState`

Статические поля глобального состояния.

| Поле | Тип | Описание |
|------|-----|----------|
| `inCursorMenu` | `bool public static` | Находится ли игрок в меню с курсором |
| `currentBoat` | `Boat public static` | Текущая лодка игрока |
| `inBed` | `ShipItemBed public static` | Лежит ли игрок в кровати |
| `sleeping` | `bool public static` | Спит ли игрок |
| `lookingWithController` | `bool public static` | Используется ли контроллер |
| `justStarted` | `bool public static` | Флаг недавнего старта |
| `currentlyLoading` | `bool public static` | Идёт загрузка |

### `Refs`

Статические ссылки на ключевые объекты.

| Метод | Тип | Описание |
|------|-----|----------|
| `SetPlayerControl(bool)` | `void static` | Включить/выключить управление игроком (`ovrController.enabled`, `charController.enabled`). **Может бросать NullReferenceException при вызове до инициализации.** |
| `mouseCrosshair` | `GameObject static` | Прицел мыши |

## Перечисления

### `InputName`

Перечисление всех действий управления. Значения используются в `GameInput.GetKeyCode`, `GPButtonKeybinding.inputNameEnum`. Получается через `Enum.Parse(typeof(InputName), inputName)`.

### `StartMenuButtonType`

`NewGame`, `Continue`, `Options`, `Quit`, `Slot`, `LoadBackupSave`.
