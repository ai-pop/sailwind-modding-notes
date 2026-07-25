# 70. Water splashes: vanilla ParticleSystems, materials, and reuse points

An examination of Sailwind v0.38's vanilla splash systems: `RefsDirectory`, `BoatSplash`, `BoatSplashCrest`, `AnimSplash`, and `WaveSplashZone`. Derived from `Assembly-CSharp.dll`; useful to mods that need water visuals without inventing an unsuitable material or shader.

## Primary runtime reference: `RefsDirectory.seaWaterSpillParticles`

`RefsDirectory` exposes this public Inspector-populated field:

```csharp
public ParticleSystem seaWaterSpillParticles;
```

It points to an already configured vanilla sea-water particle system. The game uses it, for example, when water is spilled from a mug:

```csharp
RefsDirectory.instance.seaWaterSpillParticles.transform.position = spillPos;
RefsDirectory.instance.seaWaterSpillParticles.Play();
```

### Practical consequence

A mod can obtain a **real compiled vanilla water shader/material** via:

```csharp
var ps = RefsDirectory.instance.seaWaterSpillParticles;
var material = ps.GetComponent<ParticleSystemRenderer>().sharedMaterial;
```

Do not mutate the singleton particle system itself. Instantiate a prefab or create a separate `ParticleSystem` and assign a copy/its renderer material; otherwise multiple splash sources overwrite one another's position and emission state.

## `BoatSplashCrest`

`BoatSplashCrest` is attached to a boat effect object and contains:

| Field | Role |
|---|---|
| `boatRigidbody` | boat Rigidbody |
| `deltaThreshold` / `deltaMult` | vertical-change threshold and scale |
| `minSize` / `maxSize` | particle-system size range |
| `minSpeed` / `maxSpeed` | particle-speed range |
| `volumeFadeIn` / `volumeFadeOut` | audio smoothing |
| `verticalDelta` | current vertical splash input |

In `FixedUpdate`, it compares the effect object's current Y with its previous-frame Y. Once the threshold is exceeded, it enables emission and writes `particles.startSize`, `particles.startSpeed`, and audio volume.

> It is not another water simulation or a physics system: splash intensity is purely a visual function of boat-effect transform motion and Rigidbody speed.

## `BoatSplash`

The older splash component measures:

```text
horizontalSpeed = horizontal position delta / deltaTime
verticalSpeed   = difference between local-effect delta and parent-boat delta / deltaTime
```

When `verticalSplash == true`, intensity comes from `verticalSpeed`; otherwise it uses `horizontalSpeed`. `size`, `speed`, `lifetime`, and `audioVolume` drive particle-system startup values.

`UpdatePosition()` calls:

```csharp
Ocean.Singleton.GetWaterHeightAtLocation2(x, z)
```

then smoothly moves the effect object toward the water. This direct Ocean path requires a live/ready `Ocean.Singleton`; it must not be called blindly from arbitrary mod physics ticks.

## `AnimSplash`

`AnimSplash` is another boat effect. In `LateUpdate` it:

1. reads height with `OceanHeight.GetHeight(new SampleHeightHelper(), position)`;
2. measures change in distance from effect object to surface;
3. requires both `delta > deltaThreshold` and Rigidbody speed > `1`;
4. enables a primary `ParticleSystem` and `smallSplashes`;
5. sets size and start speed from vertical delta plus boat speed.

It shows a useful vanilla pattern: calculate **relative motion against the surface** first, then drive visual emission only. It must not be used as a buoyancy-force source.

## `WaveSplashZone`

`WaveSplashZone` tests whether water covers a specific boat zone:

```csharp
float water = OceanHeight.GetHeight(helper, transform.position) + verticalOffset;
if (water >= transform.position.y && !damage.sunk)
{
    damage.Overflow(water - transform.position.y);
    // after two consecutive wet frames, enables particle emission
}
```

This means a wave splash zone can affect both visuals and boat flooding through `BoatDamage.Overflow`. Do not add it to items: it is semantically boat-specific.

## Modder recipe

1. For reliable visuals, first reuse the material from `RefsDirectory.seaWaterSpillParticles`.
2. Create a separate world-space `ParticleSystem`; do not move the singleton spill effect.
3. Derive intensity from **relative body-to-surface velocity**, not just world Y velocity.
4. A visual splash must not change `Rigidbody.velocity`, position, or `isKinematic`.
5. A custom shader must be compiled in Unity ahead of time and loaded from an AssetBundle; normal C# runtime code cannot turn ShaderLab source into a usable Unity shader.

## Related notes

- [14 — hull physics and buoyancy](../14-boat-physics-buoyancy-damage.md)
- [31 — Crest ocean and waves](../31-ocean-waves-inertia.md)
- [43 — item buoyancy](../43-item-buoyancy-water.md)
- [48 — ocean-height lifecycle](48-ocean-height-helper-lifecycle.md)
