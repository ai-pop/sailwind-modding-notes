# 49. Камера и рендер: игровая камера, шейдеры

Разбор иерархии камер, теги, дополнительные камеры, шейдеры в билде. Декомпиляция `Assembly-CSharp.dll` (Sailwind v0.38).

## C13. Игровая камера

### Иерархия GO игровой камеры

Основная камера — GO с тегом **`"MainCamera"`** (подтверждено: `BoatEmbarkTrigger` использует `CompareTag("MainCamera")`). `Camera.main` возвращает эту камеру.

**Классы камер в игре:**

| Класс | GO / контекст | Когда `Camera.main` ≠ эта камера |
|-------|---------------|----------------------------------|
| `BoatCamera` | Дополнительная камера (вид лодки сверху/снаружи) | `BoatCamera.on == true` — переключение на лодочную камеру |
| `MapTableCamera` | Камера планшета (картографический стол) | При использовании карты |
| `CameraFollow` | Следящая камера (NPC?) | - |
| `CameraHeightSmoothing` | Плавная высота | - |
| `CharacterCameraConstraint` | Ограничение камеры персонажа | - |
| `MenuCameraMovement` | Главное меню | В меню (GameState.playing = false) |
| `ShipyardCameraRotator` | Верфь | GameState.currentShipyard != null |

**BoatCamera:**
- Когда `BoatCamera.on == true` — игрок видит лодку снаружи/ сверху
- В `GoPointer.DoRaycast()`: raycast не работает при `BoatCamera.on` (early return)
- В `GoPointer.LateUpdate()`: подбор/дроп не работают при `BoatCamera.on`
- Основная MainCamera камера при этом всё ещё существует, но `BoatCamera` — **дополнительная Camera** на отдельном GO (отдельный рендер)

**MapTableCamera:**
- Активируется при использовании картографического стола
- `Camera.main` всё равно указывает на основную камеру (MainCamera tag)

**Спальный экран (сон):**
- При `GameState.sleeping / GameState.inBed` — экран закрыт (`eyesFullyClosed`), raycast заблокирован
- Camera.main продолжает существовать

**Вывод для моддера:**
- `Camera.main` — **стабильно** указывает на главную камеру (MainCamera tag) в активной игре
- `BoatCamera.on` — гейт: при true raycast/подбор заблокирован
- GameState.sleeping/inBed — raycast заблокирован
- Нет «слепых» периодов где Camera.main ≠ eye camera в активной игре

### AmplifyOcclusionEffect

`Camera.main.GetComponent<AmplifyOcclusionEffect>()` — используется в `GPButtonSettingsCheckbo` (toggle post-processing). Это компонент из AmplifyOcclusion DLL — **Built-in RP post-processing**.

## C14. Шейдеры в билде

### Built-in RP — подтверждение

Sailwind использует **Built-in Render Pipeline** (не URP/HDRP):

1. `Ocean.oceanShader` — шейдер океана (custom, с reflection/refraction, LOD levels)
2. `AmplifyOcclusionEffect` — постпроцесс от Amplify (Built-in only)
3. `Material.SetVector/SetColor/SetFloat` — API Built-in Material
4. `RenderTexture` / `Camera.Render()` — Built-in reflection/refraction pipeline
5. `Skybox` компонент — Built-in
6. `MeshRenderer` / `LODGroup` — Built-in

**Shader.Find в runtime:** доступны **все встроенные шейдеры Unity Built-in RP**:
- `Standard`, `Standard (Specular setup)`
- `Unlit/Color`, `Unlit/Texture`, `Unlit/Transparent`
- `Sprites/Default`
- `Legacy Shaders/Diffuse`, `Legacy Shaders/Transparent/Cutout`, etc.
- `UI/Default` (для UI)
- `Particle/Alpha Blended`, `Particle/Additive`

**В билде игры также доступны:**
- Океан шейдер (custom, `oceanShader` — stored on Ocean component)
- `AmplifyOcclusion` шейдеры (из DLL)
- Wake material (`_Strength` property) — Custom shader

**Для совместимости `splash.bundle`:** использовать **Built-in RP** шейдеры:
- `Standard` — основной PBR
- `Unlit/Transparent` — для particle/spash effects
- `Particle/Alpha Blended` — для particle systems

**НЕ использовать:**
- URP/HDRP шейдеры (`.shadergraph`, `Shader.Graph`)
- `Universal Render Pipeline/Lit` — нет в Built-in билде

**Shader.Find:** работает в runtime для шейдеров, включённых в билд. `Standard` и `Unlit` включены всегда. Custom шейдеры игры — **не через Shader.Find**, они на конкретных Material компонентах.
