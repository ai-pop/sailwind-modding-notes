# 76. `DockPushCol`: слой dock collider и физика рядом с водой

Разбор `DockPushCol` и `GPButtonBoatPushCol` из Sailwind v0.38. Эта заметка важна для модов на предметы/груз: dock collider может использовать тот же Unity layer, что и physics twin предмета.

Связано с [52](52-layers-collision-matrix-items-boat-player.md) и [54](54-go-pointer-big-item-decollision.md).

## `DockPushCol : GoPointerButton`

`DockPushCol` — интерактивный dock-side collider, которым игрок может толкать пришвартованную лодку. В `Awake()` он сохраняет prefab layer и collider:

```csharp
initialLayer = gameObject.layer;
col = GetComponent<Collider>();
```

### Ключевой слой-переключатель

В каждом `ExtraFixedUpdate()`:

```csharp
if (GameState.currentBoat != null)
    gameObject.layer = initialLayer;

if (GameState.currentBoat == null)
    gameObject.layer = 2;
```

Итого:

| Состояние игрока | Layer `DockPushCol` |
|---|---:|
| Игрок на лодке (`GameState.currentBoat != null`) | исходный prefab layer |
| Игрок не на лодке | **2** |

Layer 2 в Sailwind используется и physics twin предметов (`ItemRigidbody.Start()`). `IgnoreRaycast` в Unity не означает "не участвует в физике": это только raycast convention. Реальная collision matrix определяется Physics Settings.

> Поэтому static collider dock может оказаться на том же layer 2, что и solid item twin. Большой crate, пересекающий dock geometry около ватерлинии, может попасть в обычный PhysX penetration solve даже если мод ожидает, что layer 2 будет только "невидим для raycast".

## Толкание лодки

`DockPushCol` не толкает предметы. Он действует только при `isClicked` и только когда `GameState.currentBoat` существует:

```csharp
Rigidbody boat = GameState.currentBoat.parent.GetComponent<Rigidbody>();
Vector3 push = (pushForceMult * boat.mass * pointer.forward
              + upForceMult * boat.mass * Vector3.up) / max(boat.velocity.magnitude, 1f);
boat.AddForceAtPosition(push, pointer.position + Vector3.up * verticalOffset);
```

Поля по умолчанию в C#:

| Поле | Значение |
|---|---:|
| `pushForceMult` | -0.55 |
| `upForceMult` | 0 (не serialized default) |
| `verticalOffset` | 0 (не serialized default) |

Если collider dock физически сталкивается с crate, это не скриптовая dock-force логика `DockPushCol`; это Unity contact/penetration между collider-ами.

## `GPButtonBoatPushCol` — другой случай

`GPButtonBoatPushCol` расположен на лодке и управляет collider состоянием иначе:

```csharp
if (col.enabled && player is on this boat)
    col.enabled = false;
if (!col.enabled && player is not on this boat)
    col.enabled = true;
if (col.enabled && (PlayerSwimming.swimming || !Refs.charController.isGrounded))
    col.enabled = false;
```

То есть boat push collider отключается, когда игрок плавает/не стоит на земле. `DockPushCol` такого отключения не делает — он просто меняет layer по `GameState.currentBoat`.

## Практические выводы для моддера

1. Не определяйте "безопасность" collider только по layer 2: dock collider и item twin могут совпасть на этом layer.
2. `DockPushCol` не применяет силу к предмету; catastrophic item launch рядом с dock нужно искать в PhysX solver/contact state, не в `DockPushCol.AddForceAtPosition`.
3. При собственной cargo physics фиксируйте в diagnostics: collider name, tag, layer, `attachedRigidbody`, contact normal, relative velocity и penetration depth.
4. Контакт dock с большим crate может зависеть от того, находится ли игрок на лодке: `DockPushCol` возвращает исходный layer только при `GameState.currentBoat != null`.
5. Для совместимости не выключайте dock collider глобально: он нужен vanilla interaction/push flow. Правильный путь — per-body contact policy или предсказуемая depenetration логика.
