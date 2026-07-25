# 03. Кастомная система ввода: GoPointer

## Суть

Ввод в Sailwind построен на собственной системе `GoPointer` (raycast по коллайдерам), а не на Unity EventSystem / Canvas raycaster. Это критично для модов, которым нужно изолировать или перехватить ввод (например, для внутриигрового UI).

## Архитектура

```
GoPointer (MonoBehaviour)
├── DoRaycast()              — Physics.Raycast по центру экрана
│   ├── hit.collider → GetComponent<GoPointerButton>()
│   └── button.Look(this)    — наведение на объект
├── MainButtonDown()         — ЛКМ (InputName.8, проверяет !inCursorMenu)
├── AltButtonDown()          — ПКМ (InputName.9, проверяет !inCursorMenu)
├── AltButtonHeld()          — удержание ПКМ
└── LateUpdate()             — вызывает DoRaycast каждый кадр
```

`GoPointerButton` — базовый класс всех интерактивных объектов (293+ экземпляров `GPButtonRopeWinch` в типичной сцене). Содержит `public string lookText` и `public string description`, которые `LookUI` копирует в TextMesh при наведении.

## Изоляция ввода

### Метод 1: штатный флаг (частичный)

```csharp
MouseLook.ToggleMouseLookAndCursor(false);
// Устанавливает GameState.inCursorMenu = true
// Освобождает курсор (Cursor.visible = true, lockState = None)
```

**Ограничение:** флаг `inCursorMenu` проверяют `MainButtonDown`, `AltButtonDown`, `MouseButtonPointer`, `GPButtonBed`. Но **`DoRaycast` и `MouseLook.Update` проверяют его непоследовательно** — наведение и вращение камеры могут продолжать работать.

### Метод 2: Harmony-патчи (полный)

Гарантированная изоляция — prefix-патчи на ключевые методы:

```csharp
// GoPointer.DoRaycast → return false при открытом UI
[Prefix] static bool SkipDoRaycast() => !ModUI.IsVisible;

// GoPointer.MainButtonDown/AltButtonDown/AltButtonHeld → return false
[Prefix] static bool SkipButton(ref bool __result) {
    if (ModUI.IsVisible) { __result = false; return false; }
    return true;
}

// MouseLook.Update → return false (камера не следует за мышью)
[Prefix] static bool SkipMouseLook() => !ModUI.IsVisible;
```

Патчи устанавливаются через reflection (`AccessTools.Method`), поскольку типы `GoPointer` и `MouseLook` находятся в `Assembly-CSharp.dll`, не в референсах плагина.

## Курсор

Штатный метод `MouseLook.ToggleMouseLookAndCursor(bool newState)`:
- `true` → `Cursor.lockState = Locked`, `Cursor.visible = false`, `inCursorMenu = false` (геймплей)
- `false` → `Cursor.visible = true`, `Cursor.lockState = None`, `inCursorMenu = true` (меню)

При использовании необходимо **сохранять предыдущее состояние** `GameState.inCursorMenu` перед释放 курсора и восстанавливать при закрытии UI — иначе курсор скроется в игровом меню, где он должен оставаться видимым.

## Предостережение

`Refs.SetPlayerControl(bool)` может вызывать `NullReferenceException`, если вызван до полной инициализации игры (наблюдалось в `EconomyUI.Awake()` и `CurrencyExchangeUI.Awake()`). Не использовать в `Awake`/`Start` плагина.
