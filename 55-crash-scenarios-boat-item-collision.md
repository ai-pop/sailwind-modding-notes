# 55. Краш-сценарии: Collision listeners на лодке и предмете

Разбор всех collision-listeners, которые могут быть задеты при столкновении held-item twin с лодкой — ответ на запрос E5. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy. Связано с заметками 51–54.

## Collision-listeners на ItemRigidbody (twin)

### `ItemRigidbody.OnCollisionEnter(Collision collision)` — вербатим

```csharp
private void OnCollisionEnter(Collision collision)
{
    if (!collision.collider.isTrigger)
    {
        Vector3 relativeVelocity = collision.relativeVelocity;
        if (relativeVelocity.sqrMagnitude >= Debugger.instance.debugItemColAudioThreshold
            && rigidbody.mass > 1f)
        {
            ItemCollisionSoundPlayer instance = ItemCollisionSoundPlayer.instance;
            Vector3 position = item.transform.position;
            float mass = rigidbody.mass;
            instance.PlayWoodColSound(position, mass, relativeVelocity.sqrMagnitude);
        }
    }
}
```

- **Фильтр:** `!collision.collider.isTrigger` → только non-trigger контакты.
- **Порог скорости:** `sqrMagnitude >= debugItemColAudioThreshold` (0.1) и `mass > 1`.
- **Действие:** только звук (`PlayWoodColSound`). Никаких NullRef-циклов. ItemCollisionSoundPlayer имеет cooldown 0.12 с и AudioSource pool → **не крашит, не спамит бесконечно**.
- **Нет доступа к Rigidbody лодки**, нет AddForce, нет изменений состояния предмета.

> `OnCollisionEnter` на twin — **только звук**, безопасно.

## Collision-listeners на лодке

### `BoatImpactSoundsCollider.OnCollisionEnter(Collision collision)`

```csharp
// BoatImpactSoundsCollider.cs — вербатим
private void OnCollisionEnter(Collision collision)
{
    if (!collision.collider.isTrigger)
    {
        sounds.Impact(collision);
    }
}
```

- Фильтр: `!collision.collider.isTrigger` → twin (non-trigger) **пройдёт фильтр**.
- Действие: вызывает `BoatImpactSounds.Impact(collision)`.

### `BoatImpactSounds.Impact(Collision collision)` — вербатим (ключевые строки)

```csharp
public void Impact(Collision collision)
{
    // Только для активной лодки (GameState.lastBoat)
    if (GameState.lastBoat != transform.parent) return;

    // Порог скорости
    if (collision.relativeVelocity.magnitude < 0.2f) return;

    // Слабый контакт (< strongImpactThreshold = 3) → scrape sound + coroutine
    if (collision.relativeVelocity.magnitude < strongImpactThreshold)
    {
        if (!scrapeImpact.isPlaying)
        {
            // scrape sound position + PlayScrapeImpact coroutine — безопасно
        }
        return;
    }

    // Сильный контакт (>= 3): BoatDamage.Impact + wakeUp + impact audio
    BoatDamage.Impact(collision.collider, collision.relativeVelocity.magnitude);
    if (GameState.sleeping) { Sleep.instance.WakeUp(); }
    // impact audio — safe (pool, cooldown)
}
```

### `BoatDamage.Impact(Collider collider, float force)` — критичная точка

Из заметки 14: `Impact()` применяет hullDamage, если:
- нет cooldown (`impactTimer <= 0`, cooldown = 1 c);
- игрок не спит + лодка не пришвартована;
- не `OceanBottom` тег;
- скорость лодки >= minimumImpactVelocity (1.5);
- не `ShipItem` тег.

**Ключевой фильтр:** `!collision.collider.CompareTag("ShipItem")` → **если twin-коллайдер предмета имеет тег `ShipItem`**, Impact **НЕ начисляет урон**.

> Но: тег на twin GO — какой? twin GO создается как `new GameObject()` в `CreateRigidbody()` → **тег не наследуется автоматически**. Тег на visual GO может быть `ShipItem`/`Good`/`ItemSubcollider`/`EmbarkCol`, а twin GO → **Untagged** (default) → тег `ShipItem` **НЕ на twin** → **BoatDamage.Impact НЕ фильтрует twin-предмет по тегу!**

### Краш-сценарий №1: hullDamage при каждом OnCollisionEnter

Если twin (non-trigger, Untagged) сталкивается с BoatImpactSoundsCollider:
- `BoatImpactSoundsCollider.OnCollisionEnter` → `BoatImpactSounds.Impact` → `BoatDamage.Impact`.
- `Impact` не фильтрует по тегу (twin = Untagged, не ShipItem).
- Но `impactTimer` cooldown = 1 с → hullDamage начисляется **раз в секунду** при постоянном контакте.
- `maxDamagePerImpact = 0.15` → **каждый 1 с: hullDamage += 0.15** → лодка разрушается за ~7 с!

> **Это — реальный краш-сценарий для мода:** held предмет с твёрдым twin-коллайдером, постоянно контактирующий с лодкой → BoatDamage урон раз в секунду → hullDamage растёт → лодка тонет.

### Краш-сценарий №2: NullRef при OnCollisionEnter

`ItemCollisionSoundPlayer.instance` — singleton. При раннем доступе (до Awake) → null. Но `OnCollisionEnter` вызывается physics engine, только если оба Rigidbody активны — singleton должен быть готов.

`BoatImpactSounds.Impact` → `BoatDamage.Impact(collider, magnitude)` → collider всегда non-null (от physics). BoatDamage сам проверяет cooldown + sleeping + moored + OceanBottom + velocity + ShipItem tag — все поля инициализированы в Awake.

> **NullRef-циклов не видно** — все ключевые ссылки (damage, sounds, collision.collider) инициализированы при Awake или гарантированы physics engine.

### Краш-сценарий №3: Infinite physics loop (не NullRef, но deadlock)

Если twin-предмет «пружина» мода тянет предмет внутрь лодочного коллайдера → physics выталкивает → пружина тянет обратно → **OnCollisionEnter каждый physics step** → `BoatImpactSounds.Impact` → `BoatDamage.Impact` (cooldown 1с, но OnCollisionEnter fires каждый step) → **не deadlock, но спам Impact при каждом physics sub-step**.

Unity physics sub-stepping: при `CollisionDetectionMode = Discrete` (мод принудительно ставит Discrete на twin) → OnCollisionEnter fires **once per physics step** (FixedUpdate rate). Не бесконечный цикл внутри одного frame.

> **Deadlock/зависание не от collision listeners.** Это от **physics solver loop** — если пружина мода постоянно конфликтует с collider push-out, physics solver может делать много iterations per frame → **frame time explosion** (но не infinite loop).

## Другие listeners

| Класс | OnCollisionEnter | Действие | Риск |
|-------|-----------------|----------|------|
| `ItemRigidbody` | да | звук | безопасно |
| `BoatImpactSoundsCollider` | да | Impact→Damage+WakeUp+sound | **hullDamage при контакте с Untagged twin** |
| `Anchor` | (в FixedUpdate, не OnCollision) | anchor set/release | не связан с held item |
| `BoatDamage` | (нет OnCollisionEnter на damage GO) | Impact вызывается BoatImpactSoundsCollider | см. выше |

## Практические выводы

1. **Краш/зависание наиболее вероятно от physics solver loop**, не от NullRef в collision listeners.
2. **BoatDamage.Impact не фильтрует twin-предмет** (twin = Untagged) → held big-item с твёрдым twin может наносить hullDamage лодке при постоянном контакте (0.15 каждые 1 с).
3. **ItemCollisionSoundPlayer.PlayWoodColSound** — безопасно (cooldown, pool).
4. **Решение:** (а) дать twin GO тег `ShipItem` → BoatDamage.Impact фильтрует; (б) Harmony-prefix на `BoatImpactSoundsCollider.OnCollisionEnter` → пропускать при held item; (в) `Physics.IgnoreLayerCollision` twin↔boat при холде; (д) патч `BoatDamage.Impact` → проверять held-status.
5. Наиболее полное решение — **игнорировать коллизии twin↔лодка при холде** через `Physics.IgnoreLayerCollision` или Harmony.
