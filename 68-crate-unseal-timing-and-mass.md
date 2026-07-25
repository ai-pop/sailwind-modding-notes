# 68. Вскрытие ящика: порядок кадров, `CrateInventory` и масса

Точный разбор `ShipItemCrate.UnsealCrate()`, `CrateInventory.InsertItem()` и `ItemRigidbody.UpdateMass()` в Sailwind v0.38. Дополняет заметки [45](45-crate-cargo-prefabs-filter.md), [61](61-item-catalog-mass-table-units.md), [65](65-item-mass-buoyancy.md).

## Главный факт

У запечатанного ящика масса считается по полям `containedPrefab × amount`. После вскрытия ваниль превращает эти данные в живые объекты `CrateInventory.containedItems`, но **не пересчитывает массу ящика из нового списка**.

Поэтому вскрытый ящик может быть полон предметов, а его vanilla `Rigidbody.mass` уже равна только массе оболочки.

Это не предположение: порядок вызовов в декомпилированном коде однозначен.

## Последовательность `UnsealCrate()`

Для каждого из `int amount` предметов метод делает:

```csharp
GameObject go = Instantiate(
    containedPrefab,
    transform.position + new Vector3(0f, 100.5f, 0f),
    transform.rotation);

amount -= 1f;
go.GetComponent<SaveablePrefab>().RegisterToSave();
StartCoroutine(InsertItem(go.GetComponent<ShipItem>()));
```

`InsertItem` откладывает фактическое помещение предмета:

```csharp
private IEnumerator InsertItem(ShipItem item)
{
    yield return new WaitForEndOfFrame();
    item.sold = true;
    crateInventory.InsertItem(item);
}
```

После запуска всех coroutine, но **до их `WaitForEndOfFrame`**, `UnsealCrate()` сразу вызывает:

```csharp
UpdateLookText();
itemRigidbodyC.UpdateMass();
StartCoroutine(OpenAfterDelay());
```

К этому моменту `amount` уже равен `0`, а `containedItems` ещё пуст. Поэтому `ItemRigidbody.UpdateMass()` делает:

```csharp
rigidbody.mass = item.mass;
rigidbody.mass += containedPrefab.mass * amount; // amount == 0
```

Итог: Rigidbody получает **массу пустой оболочки**.

### Вскрытие не уничтожает контейнер

В теле `ShipItemCrate.UnsealCrate()` нет вызова `DestroyItem()`, `Destroy(gameObject)` или `SaveablePrefab.Unregister()`. Ванильный результат вскрытия — тот же crate GameObject, но с `amount == 0` и компонентом `CrateInventory`; он остаётся контейнером. Если ящик исчезает именно в момент вскрытия, это не штатное действие `UnsealCrate`, а внешний cleanup/мод либо последующая логика дальности/сохранения.

## Когда предметы реально появляются в ящике

На следующем `EndOfFrame` каждая coroutine вызывает:

```csharp
crateInventory.InsertItem(item);
```

`CrateInventory.InsertItem()`:

1. добавляет `ShipItem` в `public List<ShipItem> containedItems`;
2. записывает `SaveablePrefab.currentCrateId`;
3. выставляет у предмета `ItemRigidbody.attached = true`, `disableCol = true`, `inStove = true`;
4. уменьшает scale до `inventoryScale × 0.33`;
5. переводит visual и детей на layer `26`.

`CrateInventory.LateUpdate()` затем удерживает каждый contained item в позиции и повороте ящика.

### Важное отсутствие вызова

Ни `CrateInventory.InsertItem()`, ни `CrateInventory.WithdrawItem()` **не вызывают** `crate.itemRigidbodyC.UpdateMass()`.

Следовательно, после того как список стал непустым, масса ящика не получает автоматической поправки. Ванильная формула знает только старую sealed-пару `containedPrefab`/`amount`, а не `containedItems`.

## Полная временная шкала

```text
frame N, UnsealCrate()
  amount: N → 0
  spawned goods: y = crate.y + 100.5
  InsertItem coroutines scheduled
  crate ItemRigidbody.UpdateMass()
    → shell mass + containedPrefab.mass × 0
    → shell mass

EndOfFrame N
  each InsertItem coroutine:
    item.sold = true
    CrateInventory.InsertItem(item)
    → containedItems gains real item
    → no crate UpdateMass call

EndOfFrame N+1
  OpenAfterDelay() → CrateInventory.OpenCrate()
```

## Практические выводы для моддера

1. `amount == 0` не доказывает, что ящик пуст: после вскрытия содержимое находится в `CrateInventory.containedItems`.
2. Для физики, веса, грузоподъёмности или UI вскрытого ящика считайте:
   ```text
   crate ShellMass + Σ(item.mass for item in CrateInventory.containedItems)
   ```
   а не `containedPrefab.mass × amount`.
3. При обработке `UnsealCrate()` нельзя пересчитывать массу только в его postfix: живые предметы появятся лишь после EndOfFrame. Надёжные точки — postfix `CrateInventory.InsertItem` и `WithdrawItem`, либо отложенное чтение списка.
4. Не используйте положение spawned goods в момент `Instantiate`: они специально создаются на `+100.5Y` до помещения в контейнер.
5. При извлечении предмета `WithdrawItem()` удаляет его из списка и снимает `attached`/`disableCol`/`inStove`; агрегированная масса контейнера снова должна быть пересчитана.
