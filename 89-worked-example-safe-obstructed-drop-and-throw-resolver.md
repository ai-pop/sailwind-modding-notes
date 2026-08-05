# 89. Пример кода: безопасный сброс при перекрытии и резолвер броска для модов на физику

Практическое руководство и эталонный C#-код (рецепт для BepInEx-модов, таких как SailwindItemPhysics) для устранения взрывного отлетания предметов при сбросе с перекрытием (красный контур, «shoot away») и корректной обработки бросков в открытом пространстве.

Основано на выводах из [84](84-pickupable-item-collision-checker-and-decollision.md), [86](86-obstructed-drop-ejection-why-items-shoot-away.md) и [88](88-anchor-physics-exclusion-why-anchors-require-separate-handling.md).

---

## 1. Проблема: почему требуется единый резолвер

Моды на физику предметов часто перехватывают освобождение предмета из рук (`OnDrop` / `DropItem`), чтобы передать динамическому телу скорость руки игрока (`ThrowImpulse`). Без проверки на перекрытие с палубой или другими ящиками это приводит к двум багам:
1. При сбросе в тесную стопку ящик улетает в море из-за перекрытия твердых коллайдеров PhysX.
2. При попытке бросить якорь (`Anchor`) рвется ограничение троса `ConfigurableJoint`.

---

## 2. Полный код компонента-резолвера (`SafeDropAndThrowResolver.cs`)

```csharp
using System.Collections;
using UnityEngine;

namespace SailwindModdingRecipes
{
    /// <summary>
    /// Единый обработчик освобождения предметов для модов на физику.
    /// Гарантирует ванильное мягкое «втискивание» (reshuffle) при сбросе в тесных
    /// местах и честный баллистический бросок на открытом пространстве.
    /// </summary>
    public static class SafeDropAndThrowResolver
    {
        // Порог безопасности для выявления перекрытия геометрии (в метрах)
        private const float ObstructedThreshold = 0.045f;

        /// <summary>
        /// Вызывается сразу после ванильного drop/release предмета из рук.
        /// </summary>
        public static void ResolveRelease(ShipItem item, Rigidbody rb, Vector3 handVelocity)
        {
            if (item == null || rb == null)
                return;

            // 1. Исключение для якоря (Anchor : PickupableItem)
            if (item is Anchor anchor)
            {
                ResolveAnchorThrowSafe(anchor, rb, handVelocity);
                return;
            }

            // 2. Проверяем, находится ли предмет в состоянии перекрытия
            bool isObstructed = IsItemObstructed(item);

            if (isObstructed)
            {
                // РЕЖИМ ВТИСКИВАНИЯ (Vanilla Reshuffle Mode):
                // Отменяем импульс броска и гасим скорости, чтобы сработал ComputePenetration
                rb.velocity = Vector3.zero;
                rb.angularVelocity = Vector3.zero;
                item.StartCoroutine(SoftReshuffleCoroutine(item, rb));
            }
            else
            {
                // ОТКРЫТОЕ ПРОСТРАНСТВО:
                // Применяем скорость руки игрока (импульс броска)
                ApplyThrowMomentum(rb, handVelocity);
            }
        }

        private static bool IsItemObstructed(ShipItem item)
        {
            var checker = item.GetComponent<PickupableItemCollisionChecker>();
            if (checker == null || checker.collisions == 0)
                return false;

            // Если горит красный контур (!allowObstructedDropping или глубина > 4.5 см)
            return !checker.allowObstructedDropping || checker.collisions > 1;
        }

        /// <summary>
        /// Временно удерживает twin-коллайдер в режиме триггера на 10 кадров PhysX,
        /// позволяя геометрической деколлизии вытолкнуть предмет без взрыва.
        /// </summary>
        private static IEnumerator SoftReshuffleCoroutine(ShipItem item, Rigidbody rb)
        {
            Collider col = rb.GetComponent<Collider>();
            if (col == null)
                yield break;

            bool wasTrigger = col.isTrigger;
            col.isTrigger = true; // Выключаем твердый контакт PhysX

            for (int i = 0; i < 10; i++)
            {
                rb.velocity = Vector3.zero; // Гасим паразитные импульсы
                yield return new WaitForFixedUpdate();
            }

            // Возвращаем коллайдер в твердое состояние после завершения сдвига
            col.isTrigger = wasTrigger;
            rb.WakeUp();
        }

        private static void ApplyThrowMomentum(Rigidbody rb, Vector3 handVelocity)
        {
            if (handVelocity.magnitude < 0.8f)
                return; // Слишком слабое движение — обычный сброс к ногам

            rb.velocity = handVelocity;
            rb.useGravity = true;
        }

        private static void ResolveAnchorThrowSafe(Anchor anchor, Rigidbody rb, Vector3 handVelocity)
        {
            ConfigurableJoint joint = anchor.GetComponent<ConfigurableJoint>();
            if (anchor.IsSet() || joint == null || joint.linearLimit.limit < 2f)
                return;

            if (rb.mass < 10f)
                rb.mass = 100f; // Восстанавливаем реальную массу якоря

            float maxSafeSpeed = Mathf.Min(6f, joint.linearLimit.limit * 0.7f);
            Vector3 safeLaunch = Vector3.ClampMagnitude(handVelocity, maxSafeSpeed);

            rb.AddForce(safeLaunch, ForceMode.VelocityChange);
        }
    }
}
```

---

## 3. Как внедрить резолвер в Harmony-патч мода

Подключите вызов `SafeDropAndThrowResolver.ResolveRelease` в postfix-патче освобождения предмета (или в системе мониторинга `CorrectManager`):

```csharp
[HarmonyPatch(typeof(PickupableItem), nameof(PickupableItem.DropItem))]
public static class DropItem_Patch
{
    public static void Postfix(PickupableItem __instance)
    {
        ShipItem item = __instance as ShipItem;
        if (item == null)
            return;

        Rigidbody twinRb = GetItemRigidbody(item);
        Vector3 handVel = GetMeasuredHandVelocity(item);

        SafeDropAndThrowResolver.ResolveRelease(item, twinRb, handVel);
    }
}
```

### 3.1. Ключевые преимущества патча
1. **Исключает баг «shoot away»:** перекрытые ящики плавно «втискиваются» в стопки благодаря 10 кадрам `SoftReshuffleCoroutine` в режиме `isTrigger = true`.
2. **Безопасно для швартовки:** бросок якоря ограничивается длиной троса и использует `ForceMode.VelocityChange` без разрыва `ConfigurableJoint`.
3. **Сохраняет честный бросок:** в открытом пространстве предметы реалистично летят по траектории движения руки игрока.
