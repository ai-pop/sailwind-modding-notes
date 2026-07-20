# 22. Верфь, кастомизация и покупка лодок

Разбор подсистемы модификации судов. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## Каркас кастомизации (`BoatCustomParts` / `BoatPart` / `BoatPartOption`)

Лодка состоит из **настраиваемых частей** (мачты, элементы корпуса и т.п.), у каждой части — набор взаимоисключающих вариантов.

### `BoatCustomParts` (на лодке)
- `availableParts` (`List<BoatPart>`) — все настраиваемые части.
- `RefreshParts()` / `RefreshPartsWithOrder(order)` — включает активный вариант каждой части, выключает остальные.
- `GetActiveOption(partIndex)` — текущий вариант части.
- `ForceEnableOption(partIndex, optionIndex)` — смена варианта (деактивирует старый GameObject + `walkColObject`, активирует новый).

### `BoatPart` (`[Serializable]`)
| Поле | Содержание |
|------|-----------|
| `activeOption` | Индекс активного варианта. |
| `partOptions` (`List<BoatPartOption>`) | Доступные варианты. |
| `category` | Категория части (int). |

`SetOptionEnabled(i, state)` активирует/деактивирует GameObject варианта + его `childOptions` + `walkColObject` (коллайдер, по которому можно ходить).

### `BoatPartOption` (MonoBehaviour)
| Поле | Содержание |
|------|-----------|
| `optionName` | Название варианта. |
| `basePrice` / `installCost` | Базовая цена / стоимость установки. |
| `mass` | Масса (влияет на `BoatMass`, заметка 14). |
| `requires` (`List<BoatPartOption>`) | **Зависимости:** какие варианты должны быть установлены. |
| `requiresDisabled` (`List<BoatPartOption>`) | Какие варианты должны быть **отключены** (конфликты). |
| `walkColObject` | Коллайдер прохода. |
| `childOptions` / `childMast` | Дочерние объекты / дочерняя мачта. |

### Система зависимостей
- **`CanInstall(part, option)`:** вариант можно ставить, если все `requires` активны **и** все `requiresDisabled` неактивны. Иначе возвращает строку `"requires: ..."` / `"no ..."`.
- **`CanUninstall(part, option)`:** нельзя снять, если другой активный вариант зависит от этого (`requires.Contains`), **или** если это мачта с прикреплёнными парусами (`Mast.sails.Count > 0`, в т.ч. `childMast`).

## Покупка лодки (`PurchasableBoat`)

| Поле/метод | Содержание |
|------------|-----------|
| `price` | Цена лодки. |
| `region` | Регион (валюта цены). |
| `PurchaseBoat()` | **Флаг покупки = `saveable.extraSetting = true`**; списывает `price` из `PlayerGold.currency[3]` (**Gold Lions**), логирует в `DayLogs[3]`. |
| `isPurchased()` | `return saveable.extraSetting`. |
| `LoadAsPurchased()` | При загрузке, если `extraSetting` — состояние «куплена». |

> **`SaveableObject.extraSetting` для лодки = «куплена»** (это же поле используется `Recovery.FindClosestBoat` для поиска принадлежащих игроку лодок и в сейве, см. заметки 11/12). Лодки покупаются за **золото** (валюта 3), а не за региональные деньги.

## Верфь (`Shipyard`)

Стационарная мастерская для установки парусов, замены частей, ремонта и чистки корпуса.

| Поле | Содержание |
|------|-----------|
| `region` | Регион верфи (определяет валюту расчёта). |
| `availableSailColors` / `sailPrefabs` | Доступные цвета и префабы парусов. |
| `editedShipPosition` / `shipReleasePosition` | Позиция лодки на editing / на выпуске. |
| `sailInstaller` / `partsInstaller` | Установщики парусов / частей. |

### Ключевые механики
- **Плата верфи** (`GetShipyardFee`) = `PurchasableBoat.price × 3.5` — **350% от базовой цены лодки**.
- **Валюта заказа:** цены пересчитываются по курсу региона — `GetCurrencyRate() = CurrencyMarket.currentPrices[region]`. Стоимость установки паруса = `GetSailPrice() * currencyRate`.
- **Заказ** (`UpdateOrder`): суммирует стоимость установки парусов (по мачтам), вычитает возвраты (`ShipyardRefund`), добавляет стоимость частей (`partsInstaller`). Формирует текст `currentOrderText`.
- **Опции заказа:** `currentOrderIncludesCleaning` (чистка корпуса), `currentOrderIncludesRepair` (ремонт).
- **Отмена** (`CancelOrder`): восстанавливает снапшот `originalData` (`SaveBoatCustomizationData`) — все изменения откатываются.
- **Ошибки установки:** `installError`, `installedSailObstructedError` (парус мешает другим).
- Пока игрок на верфи, `GameState.currentShipyard != null` — это **блокирует сохранение, тик потребностей и ход времени** (см. заметки 11/12/18).

## Данные паруса (`SaveSailData`)

`[Serializable]`, полная конфигурация установленного паруса:

| Поле | Содержание |
|------|-----------|
| `prefabIndex` | Индекс в `PrefabsDirectory.sails`. |
| `mastIndex` | На какой мачте. |
| `installHeight` | Высота установки. |
| `minAngle` / `maxAngle` | Диапазон углов. |
| `health` | Состояние паруса. |
| `sailColor` | Цвет (из `availableSailColors`). |
| `scaleY` / `scaleZ` | Масштаб полотна (см. заметку 17). |

## Персистентность кастомизации (`SaveBoatCustomizationData`)

`[Serializable]`, сохраняется в сейв как `SaveableObject.customization` (заметка 11):

| Поле | Содержание |
|------|-----------|
| `masts[30]` (`bool[]`) | Какие мачты установлены (до 30 слотов). |
| `sails` (`List<SaveSailData>`) | Все установленные паруса с конфигурацией. |
| `partActiveOptions` (`List<int>`) | Активные варианты каждой части. |

`SaveablePaint` — компонент-маркер (`[RequireComponent(SaveableObject)]`): краска корпуса сохраняется отдельно (текстура чистоты/грязи — `extraTexture` в `SaveableObject`, заметка 11).

## Практические выводы для мододела

1. **Кастомизация** = дерево частей с вариантами и зависимостями (`requires`/`requiresDisabled`); нельзя снять мачту с парусами.
2. **Покупка лодки** за Gold Lions (валюта 3); флаг владения = `SaveableObject.extraSetting`.
3. **Плата верфи** = 3.5× цены лодки; расчёт в региональной валюте по курсу `CurrencyMarket`.
4. **Отмена заказа** откатывает к снапшоту `SaveBoatCustomizationData` — безопасно экспериментировать.
5. **Паруса** сохраняют полную конфигурацию (высота, углы, масштаб, цвет, прочность) в `SaveSailData`.
6. На верфи (`GameState.currentShipyard`) время/потребности/сохранение приостановлены.
