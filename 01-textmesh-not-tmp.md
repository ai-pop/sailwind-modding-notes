# 01. TextMesh, не TextMeshPro

## Суть

Sailwind рендерит весь внутриигровой текст через `UnityEngine.TextMesh` (legacy 3D-text component). TextMeshPro и `UnityEngine.UI.Text` в сцене отсутствуют — ноль экземпляров каждого.

## Подтверждение

Runtime-сканер сцены (`FindObjectsOfType<TMP_Text>()` и `FindObjectsOfType<UnityEngine.UI.Text>()`) возвращает 0 для обоих типов при любом состоянии игры (главное меню, настройки, игровой процесс).

Декомпиляция `Assembly-CSharp.dll` подтверждает: все классы, работающие с текстом, используют `TextMesh` напрямую:

- `LookUI` — поля `[SerializeField] private TextMesh controlsText; hintText; extraText;`
- `StartMenuButton.SetButtonText(string)` — `GetComponent<TextMesh>().text = text;`
- `GPButtonKeybinding.UpdateText()` — `text.text = keyCode.ToString();`

## Следствия для моддинга

**Патчить `TextMesh.text` setter, не `TMP_Text.text`.** Harmony-патч:

```csharp
[HarmonyPatch(typeof(TextMesh), nameof(TextMesh.text), MethodType.Setter)]
[HarmonyPrefix]
public static void Prefix(ref string value) { /* ... */ }
```

Патч на `TMP_Text.text` или `UI.Text.text` не сработает — таких компонентов не существует в сцене.

**Шрифты — `UnityEngine.Font`, не `TMP_FontAsset`.** `TMP_FontAsset.CreateFontAsset()` неприменим. Управление кириллицей — через `Font` и `Renderer.sharedMaterial` (см. [04-static-font-trap.md](04-static-font-trap.md)).

**XUnity.AutoTranslator неэффективен.** По умолчанию он патчит `TMP_Text` и `UI.Text` — ни одного такого объекта нет. Плагин сидит в памяти, ничего не делая (`Skipping plugin scan because no plugin-specific translations has been registered`).

## Замечание

TextMesh — deprecated-компонент из Unity 4.x era. Разработчик Sailwind выбрал его осознанно (вероятно, для VR-совместимости и world-space 3D-текста). Переход игры на TMP в будущих версиях сделал бы этот раздел неактуальным — но на v0.38 это ключевое архитектурное ограничение.
