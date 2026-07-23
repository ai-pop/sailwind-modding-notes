# 39. Juicebox: Tweens and "Game Feel" (Reusable)

The game has a **built-in animation/effects library** `Juicebox` that you can use directly from your mod — no need to write your own tweens. Information obtained by decompiling `Assembly-CSharp.dll` (Sailwind v0.38) via ILSpy.

## `Juicebox` (singleton `Juicebox.juice`)

All methods via `Juicebox.juice.<method>(...)`. Tweens work via added `JuiceboxHelper` component with easing curve. `DefaultTween = linear`.

### Transform and Color Tweens
```csharp
Juicebox.juice.TweenScale(obj, targetScale, duration, tween, returnToOrigin);
Juicebox.juice.TweenPosition(obj, targetPos, duration, tween, returnToOrigin);
Juicebox.juice.TweenLocalPosition(obj, targetLocalPos, duration, tween);
Juicebox.juice.TweenColor(obj, targetColor /*Color32*/, duration, tween, returnToOrigin);
Juicebox.juice.FadeIn(obj, duration, tween, target);    // fade in (alpha)
Juicebox.juice.FadeOut(obj, duration, tween, target);   // fade out
```
- `returnToOrigin = true` — return to initial state after tween (for button "pop" effects).
- `duration <= 0` — apply instantly.
- Overloads without parameters use defaults (1 s, `linear`).

### "Game Feel"
```csharp
Juicebox.juice.FreezeGame(duration);              // hit-stop (brief freeze)
Juicebox.juice.ChangeGameSpeed(speed, duration);  // time warp
Juicebox.juice.ShakeScreen(speed, decay, direction);          // camera shake
Juicebox.juice.ShakeObject(obj, speed, decay, direction);     // object shake
```

### Effects
```csharp
Juicebox.juice.AttachObject(attachTo, attachedObject /*or resourceName*/, duration);
Juicebox.juice.EmitParticles(position, rotation, particlesPrefab /*or resourceName*/, duration);
Juicebox.juice.PlaySoundAt(name, position, ...);   // sound at point (e.g. "lock unlock" at steering wheel)
```

## Easing Curves (`JuiceboxTween`)

```csharp
public enum JuiceboxTween {
    simple, linear,
    quadraticIn, quadraticOut, quadraticInOut,
    exponentialIn, exponentialOut, exponentialInOut,
    bounceIn, bounceOut, bounceInOut,
    overshootIn, overshootOut, overshootInOut,
    elasticIn, elasticOut, elasticInOut
}
```
17 curves: linear, quadratic, exponential, bounce, overshoot, elastic (each in/out/inOut).

## How the Game Uses It (Examples)

- **Quest dialogue button pop**: `TweenScale(dialogUI, Vector3.one * 0.9f, 0.1f, bounceInOut, returnToOrigin: true)` (note 27).
- **Sound at point**: `Juicebox.juice.PlaySoundAt("lock unlock", pos, 0, 0.66, 0.88)` on steering wheel lock (note 38).
- UI fades, shake on impacts, etc.

## 👁 Modder's View

| Want | How |
|------|-----|
| Smooth appearance/disappearance of custom UI | `FadeIn/FadeOut(myObj, duration, tween)` |
| Button "pop"/pulse | `TweenScale(obj, scale, 0.1f, bounceInOut, returnToOrigin: true)` |
| Animated HUD element movement | `TweenPosition / TweenLocalPosition` |
| Color change (damage, warning) | `TweenColor(obj, color32, duration, tween)` |
| Camera shake on event | `ShakeScreen(speed, decay)` |
| Hit-stop / slowdown at key moment | `FreezeGame(duration)` / `ChangeGameSpeed(speed, duration)` |
| Particles/sound at world point | `EmitParticles(...)` / `PlaySoundAt(...)` |

**Pitfalls:**
- `Juicebox.juice` only available when `Juicebox` object is in scene (present in game scene); may not exist in main menu — check for null.
- `TweenColor` works with `Color32` via `Renderer`/object graphics; for `TextMesh` color may need a different approach.
- Tweens add `JuiceboxHelper` component to object — multiple simultaneous tweens of same property on same object may conflict.
- `ChangeGameSpeed`/`FreezeGame` affect `Time.timeScale` globally — be careful, affects entire game (and timers of other mods).

## Practical Takeaways

1. **`Juicebox.juice`** — game's ready tween engine: scale/position/color/fade + screen/object shake + hit-stop + particles + sound at point.
2. **17 easing curves** (`JuiceboxTween`) — from linear to elastic/bounce.
3. For polishing your UI/effects **use `Juicebox.juice`**, don't write coroutine-lerps manually.
4. `returnToOrigin` — for "pop" effects; `duration <= 0` — instant.
5. Be careful with `ChangeGameSpeed`/`FreezeGame` (global `Time.timeScale`).
