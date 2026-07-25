# 69. Извлечение из ящика: handoff `WithdrawItem → PickUpItem`

Точный порядок ванильного извлечения предмета из открытого `CrateInventory` в Sailwind v0.38. Связано с [45](45-crate-cargo-prefabs-filter.md), [47](47-item-holding-pickup-flow.md), [68](68-crate-unseal-timing-and-mass.md).

## Критический порядок вызовов

Кнопка ящика (`CrateInventoryButton.OnActivate`) при пустых руках выполняет **два вызова подряд в одном кадре**:

```csharp
CrateInventoryUI.instance.currentCrate.WithdrawItem(currentItem);
activatingPointer.PickUpItem(currentItem);
```

Нет coroutine, physics tick или EndOfFrame между ними.

```text
CrateInventoryButton.OnActivate
  1. CrateInventory.WithdrawItem(item)
  2. GoPointer.PickUpItem(item)
  3. CrateInventoryUI.RefreshButtons()
  4. позднее: GoPointer.LateUpdate пишет pose предмета в руках
  5. позднее: ItemRigidbody.FixedUpdate синхронизирует twin
```

Это переход между двумя владельцами положения: container/UI → hand/pointer. Мод, который включает свободную физику в середине этого handoff, может оставить twin на старом UI/ящичном pose, пока visual уже перемещается в руки.

## Что именно делает `CrateInventory.WithdrawItem`

```csharp
containedItems.Remove(item);
item.SaveablePrefab.currentCrateId = 0;
item.GetItemRigidbody().attached = false;
item.GetItemRigidbody().disableCol = false;
item.GetItemRigidbody().inStove = false;
item.transform.localScale = Vector3.one;
item.gameObject.layer = 2;
foreach (Transform t in item.GetComponentsInChildren<Transform>(true))
    t.gameObject.layer = 2;
```

Метод **не**:

- не вызывает `ResetPos()`;
- не ставит twin в visual pose;
- не делает Rigidbody dynamic напрямую;
- не перемещает предмет к игроку;
- не уведомляет `ItemRigidbody.UpdateMass()` у родительского ящика.

Он лишь снимает container flags и удаляет ссылку из списка. Положение до следующей стадии ещё может быть позицией ящика или кнопки UI.

## Что сразу делает `GoPointer.PickUpItem`

После `WithdrawItem` vanilla pointer:

```csharp
heldItem = item;
item.gameObject.layer = 2;
heldItem.held = this;
item.OnPickup();
```

Таким образом `held` становится ненулевым в том же call stack. Но окончательная visual pose предмета в руках записывается позднее в pointer `LateUpdate`; twin следует за visual в `ItemRigidbody.FixedUpdate`.

## Почему UI может временно владеть pose предмета

Пока UI ящика открыт, `CrateInventoryUI.LateUpdate()` вызывает для каждой кнопки:

```csharp
button.UpdateItemPos();
```

А `CrateInventoryButton.UpdateItemPos()` пишет:

```csharp
currentItem.transform.position = button.transform.position;
currentItem.transform.rotation = button.transform.rotation * inventoryRotation;
currentItem.itemRigidbodyC.ForceRigidbodyToWalkCol();
```

`RefreshButtons()` после извлечения очищает кнопку, но внешний мод не должен предполагать, что старый UI pose уже исчез до завершения кадра.

## Безопасный рецепт для мода с собственной физикой

При postfix `CrateInventory.WithdrawItem`:

1. считать тело в состоянии `containerTransfer` и **не применять buoyancy/force**;
2. подождать `WaitForEndOfFrame` — vanilla успеет вызвать `PickUpItem` и записать visual pose;
3. подождать первый `WaitForFixedUpdate`;
4. вызвать штатный `shipItem.ResetRigidbody()` или эквивалентно поставить twin в текущую visual pose;
5. только после этого вернуть управление свободной физике — и лишь если `held == null`.

Нельзя вызывать собственную физику между `WithdrawItem` и `PickUpItem`: это именно one-frame race между container pose и hand pose.

## Практические выводы

1. `WithdrawItem()` не является полным физическим release: это только снятие флагов контейнера.
2. `held` устанавливается сразу после него, но pose рук появляется позже; нужен handoff guard минимум до EndOfFrame + следующего fixed tick.
3. Вскрытый ящик и извлекаемый предмет требуют независимых обновлений массы: parent loses mass, child получает свою массу.
4. Если предмет «зависает» в старой позиции ящика/UI, сначала проверяйте пропущенный `ResetRigidbody()` после handoff, а не buoyancy.
