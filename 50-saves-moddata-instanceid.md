# 50. Сохранения: GameState.modData и SaveablePrefab.instanceId

Разбор механизма сохранения modData, стабильность instanceId, триггеры записи. Декомпиляция `Assembly-CSharp.dll` (Sailwind v0.38). Связано с заметкой 11.

## D15. GameState.modData

### Когда словарь реально попадает в сейв

```csharp
// GameState.cs
public static Dictionary<string, string> modData = new Dictionary<string, string>();

// SaveContainer.cs
public Dictionary<string, string> modData = new Dictionary<string, string>();

// SaveLoadManager.cs — при сохранении:
saveContainer.modData = GameState.modData;  // прямое присвоение ссылки!

// SaveLoadManager.cs — при загрузке:
if (saveContainer.modData != null) {
    GameState.modData = saveContainer.modData;  // прямое присвоение ссылки!
}
```

**Триггер записи:** `GameState.modData` попадает в сейв **при каждом явном сохранении** (автосейв, ручной сейв). **Нет отдельного триггера** — dictionary сериализуется как-is.

**Важно:** при сохранении — `saveContainer.modData = GameState.modData` — это **присвоение ссылки**, не копия! Мод может изменять `GameState.modData` в любой момент, и при следующем сохранении — изменения попадут в файл.

При загрузке: `GameState.modData = saveContainer.modData` — присвоение ссылки из десериализованного container. Mод получает **ready dictionary** сразу после загрузки.

**Методы SaveLoadManager (контекст):**
- `SaveModData()` — вызывается **перед** `saveContainer.modData = GameState.modData` — но это абстрактный метод (для модов? или placeholder)
- `LoadModData()` — вызывается **после** `GameState.modData = saveContainer.modData` — тоже абстрактный

### Лимиты размера

`GameState.modData` сериализуется через `BinaryFormatter` (как часть `SaveContainer`). BinaryFormatter:
- **Нет явного лимита** на размер Dictionary<string, string> в BinaryFormatter
- **Практический лимит:** размер файла сохранения (обычно несколько МБ для всей игры)
- **Осторожность:** большие строки (JSON-сериализация целых структур) — безопасны, но увеличивают размер сейва
- **Рекомендация:** ключи — короткие (мод-префикс + id), значения — разумный размер (< 1KB на запись). Не хранить бинарные данные как Base64 строки мегабайтного размера.

### Стоит ли остерегаться больших строк?

**Нет технических ограничений**, но:
- `BinaryFormatter` сериализует `Dictionary<string, string>` эффективно (ключ + значение как строки)
- **Риск:** если мод создаёт тысячи записей → сейв разрастается → загрузка замедляется
- **Рекомендация:** использовать `modData` для конфигурации и лёгких данных, не для логов или истории

### Мод-идентификаторы (best practice)

```csharp
// Рекомендуемый паттерн ключей:
string key = "MyMod_" + itemId;     // префикс мода + уникальный id
GameState.modData[key] = value;      // запись (автоматически при следующем сейве)

// Чтение:
if (GameState.modData.ContainsKey(key)) {
    string value = GameState.modData[key];
}

// Инициализация при отсутствии:
if (!GameState.modData.ContainsKey("MyMod_spawnTimer")) {
    GameState.modData["MyMod_spawnTimer"] = "30";
}
```

**Важно:** мод должен читать `modData` **после загрузки** (GameState.currentlyLoading = false, GameState.playing = true). В момент загрузки `GameState.modData` заполняется из сейва.

## SaveablePrefab.instanceId — стабильность

### Как id назначается

```csharp
private void AssignRandomInstanceId() {
    int num = Random.Range(1, int.MaxValue);
    while (existingInstanceIds.Contains(num)) {
        num = Random.Range(1, int.MaxValue);
    }
    instanceId = num;
}
```

- **Random.Range(1, int.MaxValue)** — случайный id при **первом RegisterToSave**
- `existingInstanceIds` — static List<int>, проверка на уникальность
- Если `instanceId == 0` (не назначен) при `RegisterToSave` → `AssignRandomInstanceId()`

### Стабильность id

**После первого сохранения:** `instanceId` сериализуется в `SavePrefabData.instanceId` и **сохраняется**. При загрузке: `instanceId = data.instanceId` (из сейва).

| Событие | Стабилен ли id? |
|---------|-----------------|
| Перезагрузка сейва | **Да** — id из сейва |
| Покупка (Sell → RegisterToSave) | **Да** — id назначается при первом RegisterToSave |
| Перемещение (в ящик/инвентарь/на лодку) | **Да** — id не меняется |
| Новый Instantiate | **Новый id** — каждый Instantiate → новый ShipItem → новый instanceId при RegisterToSave |
| Destroy + recreate | **Новый id** — старый id удаляется из existingInstanceIds при Unregister |

**Вывод:** `instanceId` **стабилен** для одного физического предмета (один GameObject) между сейвами. Не меняется при перемещении, ящиках, лодке. **Но:** при уничтожении и recreation — новый id.

### Мод-использование instanceId

```csharp
// Получить instanceId предмета
int id = shipItem.GetComponent<SaveablePrefab>().instanceId;

// Сохранить в modData (стабильный между сейвами)
GameState.modData["MyMod_item_" + id] = "state";

// Проверить: предмет зарегистрирован для сохранения?
if (shipItem.GetComponent<SaveablePrefab>() != null) {
    // instanceId назначен при RegisterToSave
}
```

**Риск:** если предмет уничтожен (`DestroyItem`) — его instanceId исчезает из `existingInstanceIds`. Ссылка на id в `modData` станет **мертвой**. Мод должен чистить мёртвые записи при загрузке, или использовать `parentObject` как fallback (SaveablePrefab.GetParentObject() — индекс родительского объекта в сцене).

### currentCrateId — для вложенных предметов

```csharp
public int currentCrateId;  // id ящика, в котором предмет
```

- При помещении предмета в ящик: `currentCrateId` = instanceId ящика
- Сериализуется вместе с предметом
- При загрузке: `LoadAfterDelay` ищет ящик по id и помещает предмет обратно
