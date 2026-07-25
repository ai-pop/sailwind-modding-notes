# 04. Загрузка шрифтов в runtime: статические vs динамические

## Суть

В Unity 2019.1 существуют два способа загрузки шрифта из `.ttf`-файла в runtime, и они ведут себя принципиально по-разному при подмене у `TextMesh`. Неправильный выбор приводит к **полному исчезновению текста**.

## `new Font(path)` — статический шрифт

```csharp
Font font = new Font("C:/path/to/font.ttf");
```

Создаёт **статический** шрифт: атлас глифов печётся один раз при загрузке. Если в момент создания в атласе нет нужных глифов (например, кириллицы) — они не появятся. Подмена такого шрифта у TextMesh:

- `target.font = staticFont;`
- `target.GetComponent<Renderer>().sharedMaterial = staticFont.material;`

**Результат:** текст пропадает полностью. В логе — спам `Font size and style overrides are only supported for dynamic fonts`.

## `Font.CreateDynamicFontFromOSFont(name, size)` — динамический шрифт

```csharp
Font font = Font.CreateDynamicFontFromOSFont("Arial", 16);
```

Создаёт **динамический** шрифт: глифы пекутся в текстуру по запросу при рендере. Поддерживает любой размер, любой набор символов, работает с TextMesh корректно.

**Ограничение:** ищет шрифт **по имени семейства среди установленных в ОС**. Loose `.ttf`-файл из папки плагина не будет найден, если имя не зарегистрировано в системе.

## Корректная стратегия для loose `.ttf`

1. **Парсинг имени семейства из TTF.** Чтение таблицы `name` (nameID=1, platformID=3, encoding UTF-16BE):

```csharp
// Упрощённо: чтение offset table → поиск тега 'name' → nameID=1
string familyName = ReadFontFamilyName(path);
Font font = Font.CreateDynamicFontFromOSFont(familyName, 16);
```

2. **Проверка кириллицы** через `GetCharacterInfo` после `RequestCharactersInTexture`:

```csharp
font.RequestCharactersInTexture("Ая", 32);
CharacterInfo info;
bool hasCyrillic = font.GetCharacterInfo('А', out info, 32);
```

3. **Fallback на системный Arial** если динамический шрифт по имени не создан или не содержит кириллицы.

## Ложные негативы `GetCharacterInfo`

Для свежесозданного динамического шрифта текстура глифов **перестраивается асинхронно**. Вызов `GetCharacterInfo` в том же кадре может вернуть `false` для валидного глифа. Это не означает отсутствие глифа — означает, что текстура ещё не обновилась.

**Следствие:** нельзя отвергать шрифт solely на основе `GetCharacterInfo == false` в первом кадре. Безопасная стратегия — принимать шрифт, если файл валиден, и позволить Unity дозапечь глифы при первом рендере.

## Сводка

| Метод | Тип шрифта | Кириллица | Loose .ttf | TextMesh |
|-------|-----------|-----------|------------|----------|
| `new Font(path)` | Статический | Только предпечённые глифы | Да | **Ломает рендер** |
| `CreateDynamicFontFromOSFont(name)` | Динамический | По запросу | Только установленные в ОС | Корректно |
| `CreateDynamicFontFromOSFont(familyName из TTF)` | Динамический | По запросу | Да (через парсинг имени) | Корректно |
