# 55. Crash scenarios: Collision listeners on boat and item

Breakdown of all collision-listeners that could be triggered when held-item twin collides with the boat — answer to request E5. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy. Related to notes 51–54.

## Collision-listeners on ItemRigidbody (twin)

### `ItemRigidbody.OnCollisionEnter(Collision collision)` — verbatim

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

- **Filter:** `!collision.collider.isTrigger` → only non-trigger contacts.
- **Velocity threshold:** `sqrMagnitude >= debugItemColAudioThreshold` (0.1) and `mass > 1`.
- **Action:** only sound (`PlayWoodColSound`). No NullRef loops. ItemCollisionSoundPlayer has cooldown 0.12 s and AudioSource pool → **doesn't crash, doesn't spam infinitely**.
- **No access to boat's Rigidbody**, no AddForce, no item state changes.

> `OnCollisionEnter` on twin — **only sound**, safe.

## Collision-listeners on the boat

### `BoatImpactSoundsCollider.OnCollisionEnter(Collision collision)`

```csharp
// BoatImpactSoundsCollider.cs — verbatim
private void OnCollisionEnter(Collision collision)
{
    if (!collision.collider.isTrigger)
    {
        sounds.Impact(collision);
    }
}
```

- Filter: `!collision.collider.isTrigger` → twin (non-trigger) **passes the filter**.
- Action: calls `BoatImpactSounds.Impact(collision)`.

### `BoatImpactSounds.Impact(Collision collision)` — verbatim (key lines)

```csharp
public void Impact(Collision collision)
{
    // Only for active boat (GameState.lastBoat)
    if (GameState.lastBoat != transform.parent) return;

    // Velocity threshold
    if (collision.relativeVelocity.magnitude < 0.2f) return;

    // Weak contact (< strongImpactThreshold = 3) → scrape sound + coroutine
    if (collision.relativeVelocity.magnitude < strongImpactThreshold)
    {
        if (!scrapeImpact.isPlaying)
        {
            // scrape sound position + PlayScrapeImpact coroutine — safe
        }
        return;
    }

    // Strong contact (>= 3): BoatDamage.Impact + wakeUp + impact audio
    BoatDamage.Impact(collision.collider, collision.relativeVelocity.magnitude);
    if (GameState.sleeping) { Sleep.instance.WakeUp(); }
    // impact audio — safe (pool, cooldown)
}
```

### `BoatDamage.Impact(Collider collider, float force)` — critical point

From note 14: `Impact()` applies hullDamage if:
- no cooldown (`impactTimer <= 0`, cooldown = 1 s);
- player not sleeping + boat not moored;
- not `OceanBottom` tag;
- boat velocity >= minimumImpactVelocity (1.5);
- not `ShipItem` tag.

**Key filter:** `!collision.collider.CompareTag("ShipItem")` → **if twin-collider has tag `ShipItem`**, Impact **does not apply damage**.

> But: tag on twin GO — what? twin GO is created as `new GameObject()` in `CreateRigidbody()` → **tag doesn't inherit automatically**. Tag on visual GO can be `ShipItem`/`Good`/`ItemSubcollider`/`EmbarkCol`, but twin GO → **Untagged** (default) → tag `ShipItem` is **NOT on twin** → **BoatDamage.Impact does NOT filter twin-item by tag!**

### Crash scenario #1: hullDamage on every OnCollisionEnter

If twin (non-trigger, Untagged) collides with BoatImpactSoundsCollider:
- `BoatImpactSoundsCollider.OnCollisionEnter` → `BoatImpactSounds.Impact` → `BoatDamage.Impact`.
- `Impact` doesn't filter by tag (twin = Untagged, not ShipItem).
- But `impactTimer` cooldown = 1 s → hullDamage is applied **once per second** during constant contact.
- `maxDamagePerImpact = 0.15` → **every 1 s: hullDamage += 0.15** → boat destroyed in ~7 s!

> **This is the real crash scenario for the mod:** held item with solid twin-collider, constantly contacting boat → BoatDamage damage once per second → hullDamage grows → boat sinks.

### Crash scenario #2: NullRef on OnCollisionEnter

`ItemCollisionSoundPlayer.instance` — singleton. On early access (before Awake) → null. But `OnCollisionEnter` is called by physics engine, only if both Rigidbodies are active — singleton should be ready.

`BoatImpactSounds.Impact` → `BoatDamage.Impact(collider, magnitude)` → collider always non-null (from physics). BoatDamage checks cooldown + sleeping + moored + OceanBottom + velocity + ShipItem tag — all fields initialized in Awake.

> **No NullRef loops visible** — all key references (damage, sounds, collision.collider) are initialized in Awake or guaranteed by physics engine.

### Crash scenario #3: Infinite physics loop (not NullRef, but deadlock)

If twin-item "spring" mod pulls item inside boat collider → physics pushes out → spring pulls back → **OnCollisionEnter every physics step** → `BoatImpactSounds.Impact` → `BoatDamage.Impact` (cooldown 1s, but OnCollisionEnter fires every step) → **not deadlock, but Impact spam at every physics sub-step**.

Unity physics sub-stepping: with `CollisionDetectionMode = Discrete` (mod forces Discrete on twin) → OnCollisionEnter fires **once per physics step** (FixedUpdate rate). Not an infinite loop within one frame.

> **Deadlock/hang is not from collision listeners.** It's from **physics solver loop** — if mod spring constantly conflicts with collider push-out, physics solver can do many iterations per frame → **frame time explosion** (but not infinite loop).

## Other listeners

| Class | OnCollisionEnter | Action | Risk |
|-------|-----------------|----------|------|
| `ItemRigidbody` | yes | sound | safe |
| `BoatImpactSoundsCollider` | yes | Impact→Damage+WakeUp+sound | **hullDamage on contact with Untagged twin** |
| `Anchor` | (in FixedUpdate, not OnCollision) | anchor set/release | not related to held item |
| `BoatDamage` | (no OnCollisionEnter on damage GO) | Impact called by BoatImpactSoundsCollider | see above |

## Practical conclusions

1. **Crash/hang most likely from physics solver loop**, not from NullRef in collision listeners.
2. **BoatDamage.Impact doesn't filter twin-item** (twin = Untagged) → held big-item with solid twin can cause hullDamage to boat during constant contact (0.15 every 1 s).
3. **ItemCollisionSoundPlayer.PlayWoodColSound** — safe (cooldown, pool).
4. **Solutions:** (a) give twin GO tag `ShipItem` → BoatDamage.Impact filters; (b) Harmony-prefix on `BoatImpactSoundsCollider.OnCollisionEnter` → skip when held item; (c) `Physics.IgnoreLayerCollision` twin↔boat while held; (d) patch `BoatDamage.Impact` → check held-status.
5. Most complete solution — **ignore twin↔boat collisions while held** via `Physics.IgnoreLayerCollision` or Harmony.
