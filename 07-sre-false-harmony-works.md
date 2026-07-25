# 07. SRE=False — HarmonyX тем не менее работает

## Суть

BepInEx 5.4.23.5 в логе сообщает `Supports SRE: False`. Это означает, что Mono runtime игры скомпилирован без `System.Reflection.Emit` (SRE). Тем не менее, HarmonyX (поставляемый с BepInEx) функционирует корректно.

## Ожидание vs реальность

| | Ожидание | Реальность |
|---|---------|------------|
| `PatchAll()` | Exception: SRE не доступен | Патчи применяются успешно |
| Prefix/Postfix | Не работают | Работают |
| `ref`-параметры | Не поддерживаются | Поддерживаются |

## Причина

HarmonyX (форк Harmony от BepInEx-команды) содержит альтернативный бэкенд для окружений без SRE. Вместо `DynamicMethod` + `ILGenerator` используется `Mono.Cecil` для манипуляции IL на уровне сборки. Это медленнее при установке патчей, но функционально эквивалентно.

## Подтверждение

Существующие моды для Sailwind на Harmony: NANDFixes, TowableBoats, SailInfo, ChronoCompassGPS — все работают на том же runtime с `SRE: False`. Простой prefix-патч с `ref string value` на `TextMesh.text` setter применяется и выполняется без ошибок.

## Практическое значение

Не следует избегать Harmony при разработке модов для Sailwind из-за `SRE: False`. Достаточно обернуть `PatchAll` в `try/catch` и логировать результат для диагностики.

```csharp
try {
    harmony.PatchAll(typeof(MyPatcher));
    Log.LogInfo("Harmony patches applied.");
} catch (Exception ex) {
    Log.LogError($"Harmony PatchAll failed: {ex}");
}
```

## Ограничение

Транспиляторы (IL-level manipulation через `CodeInstruction`) могут работать нестабильно без SRE. Prefix и Postfix — надёжны. Transpiler — на свой риск.
