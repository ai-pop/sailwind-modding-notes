# 39. Juicebox: твины и «гейм-фил» (переиспользуемый)

В игре есть **встроенная библиотека анимаций/эффектов** `Juicebox`, которой можно пользоваться из мода напрямую — свои твины писать не нужно. Информация получена декомпиляцией `Assembly-CSharp.dll` (Sailwind v0.38) через ILSpy.

## `Juicebox` (синглтон `Juicebox.juice`)

Все методы — через `Juicebox.juice.<метод>(...)`. Твины работают через добавляемый компонент `JuiceboxHelper` с кривой easing. `DefaultTween = linear`.

### Твины трансформов и цвета
```csharp
Juicebox.juice.TweenScale(obj, targetScale, duration, tween, returnToOrigin);
Juicebox.juice.TweenPosition(obj, targetPos, duration, tween, returnToOrigin);
Juicebox.juice.TweenLocalPosition(obj, targetLocalPos, duration, tween);
Juicebox.juice.TweenColor(obj, targetColor /*Color32*/, duration, tween, returnToOrigin);
Juicebox.juice.FadeIn(obj, duration, tween, target);    // появление (альфа)
Juicebox.juice.FadeOut(obj, duration, tween, target);   // исчезновение
```
- `returnToOrigin = true` — после твина вернуться в исходное состояние (для «поп»-эффектов кнопок).
- `duration <= 0` — применить мгновенно.
- Перегрузки без параметров используют значения по умолчанию (1 c, `linear`).

### «Гейм-фил»
```csharp
Juicebox.juice.FreezeGame(duration);              // хит-стоп (заморозка на мгновение)
Juicebox.juice.ChangeGameSpeed(speed, duration);  // временной варп
Juicebox.juice.ShakeScreen(speed, decay, direction);          // тряска камеры
Juicebox.juice.ShakeObject(obj, speed, decay, direction);     // тряска объекта
```

### Эффекты
```csharp
Juicebox.juice.AttachObject(attachTo, attachedObject /*или resourceName*/, duration);
Juicebox.juice.EmitParticles(position, rotation, particlesPrefab /*или resourceName*/, duration);
Juicebox.juice.PlaySoundAt(name, position, ...);   // звук в точке (напр. "lock unlock" у штурвала)
```

## Кривые easing (`JuiceboxTween`)

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
17 кривых: линейная, квадратичная, экспоненциальная, bounce, overshoot, elastic (каждая in/out/inOut).

## Как это использует игра (примеры)

- **Поп кнопки** диалога квеста: `TweenScale(dialogUI, Vector3.one * 0.9f, 0.1f, bounceInOut, returnToOrigin: true)` (заметка 27).
- **Звук в точке**: `Juicebox.juice.PlaySoundAt("lock unlock", pos, 0, 0.66, 0.88)` при фиксации штурвала (заметка 38).
- Фейды UI, тряска при ударах и т.п.

## 👁 Взгляд моддера

| Хочу | Как |
|------|-----|
| Плавное появление/исчезновение своего UI | `FadeIn/FadeOut(myObj, duration, tween)` |
| «Поп»/пульсация кнопки | `TweenScale(obj, scale, 0.1f, bounceInOut, returnToOrigin: true)` |
| Анимация перемещения HUD-элемента | `TweenPosition / TweenLocalPosition` |
| Смена цвета (урон, предупреждение) | `TweenColor(obj, color32, duration, tween)` |
| Тряска камеры при событии | `ShakeScreen(speed, decay)` |
| Хит-стоп / замедление при важном моменте | `FreezeGame(duration)` / `ChangeGameSpeed(speed, duration)` |
| Частицы/звук в точке мира | `EmitParticles(...)` / `PlaySoundAt(...)` |

**Грабли:**
- `Juicebox.juice` доступен только когда объект `Juicebox` в сцене (есть в игровой сцене); в главном меню может не быть — проверять на null.
- `TweenColor` работает с `Color32` и через `Renderer`/графику объекта; для `TextMesh` цвет может потребовать своего подхода.
- Твины добавляют компонент `JuiceboxHelper` на объект — несколько одновременных твинов одного свойства на одном объекте могут конфликтовать.
- `ChangeGameSpeed`/`FreezeGame` влияют на `Time.timeScale` глобально — осторожно, заденет всю игру (и таймеры других модов).

## Практические выводы

1. **`Juicebox.juice`** — готовый tween-движок игры: scale/position/color/fade + screen/object shake + hit-stop + частицы + звук в точке.
2. **17 кривых easing** (`JuiceboxTween`) — от linear до elastic/bounce.
3. Для полировки своего UI/эффектов **используйте `Juicebox.juice`**, а не пишите корутины-лерпы вручную.
4. `returnToOrigin` — для «поп»-эффектов; `duration <= 0` — мгновенно.
5. Осторожно с `ChangeGameSpeed`/`FreezeGame` (глобальный `Time.timeScale`).
