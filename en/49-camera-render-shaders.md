# 49. Camera and rendering: game camera, shaders

Breakdown of camera hierarchy, tags, additional cameras, shaders in build. From decompilation of `Assembly-CSharp.dll` (Sailwind v0.38).

## C13. Game camera

### Game camera GO hierarchy

Main camera — GO with tag **`"MainCamera"`** (confirmed: `BoatEmbarkTrigger` uses `CompareTag("MainCamera")`). `Camera.main` returns this camera.

**Camera classes in the game:**

| Class | GO / context | When `Camera.main` ≠ this camera |
|-------|-------------|----------------------------------|
| `BoatCamera` | Additional camera (top/external boat view) | `BoatCamera.on == true` — switches to boat camera |
| `MapTableCamera` | Chart table camera | When using the map table |
| `CameraFollow` | Follow camera (NPC?) | — |
| `CameraHeightSmoothing` | Smooth height | — |
| `CharacterCameraConstraint` | Character camera constraint | — |
| `MenuCameraMovement` | Main menu | In menu (GameState.playing = false) |
| `ShipyardCameraRotator` | Shipyard | GameState.currentShipyard != null |

**BoatCamera:**
- When `BoatCamera.on == true` — player sees boat from outside/above
- In `GoPointer.DoRaycast()`: raycast doesn't work when `BoatCamera.on` (early return)
- In `GoPointer.LateUpdate()`: pickup/drop don't work when `BoatCamera.on`
- Main MainCamera camera still exists, but `BoatCamera` is an **additional Camera** on a separate GO (separate render)

**MapTableCamera:**
- Activated when using the chart table
- `Camera.main` still points to the main camera (MainCamera tag)

**Sleep screen:**
- When `GameState.sleeping / GameState.inBed` — screen is closed (`eyesFullyClosed`), raycast blocked
- Camera.main continues to exist

**Conclusion for modder:**
- `Camera.main` — **stably** points to the main camera (MainCamera tag) in active gameplay
- `BoatCamera.on` — gate: when true, raycast/pickup blocked
- GameState.sleeping/inBed — raycast blocked
- No "blind" periods where Camera.main ≠ eye camera in active gameplay

### AmplifyOcclusionEffect

`Camera.main.GetComponent<AmplifyOcclusionEffect>()` — used in `GPButtonSettingsCheckbo` (toggle post-processing). This is a component from AmplifyOcclusion DLL — **Built-in RP post-processing**.

## C14. Shaders in the build

### Built-in RP — confirmation

Sailwind uses **Built-in Render Pipeline** (not URP/HDRP):

1. `Ocean.oceanShader` — ocean shader (custom, with reflection/refraction, LOD levels)
2. `AmplifyOcclusionEffect` — postprocess from Amplify (Built-in only)
3. `Material.SetVector/SetColor/SetFloat` — Built-in Material API
4. `RenderTexture` / `Camera.Render()` — Built-in reflection/refraction pipeline
5. `Skybox` component — Built-in
6. `MeshRenderer` / `LODGroup` — Built-in

**Shader.Find at runtime:** all **Unity Built-in RP built-in shaders** are available:
- `Standard`, `Standard (Specular setup)`
- `Unlit/Color`, `Unlit/Texture`, `Unlit/Transparent`
- `Sprites/Default`
- `Legacy Shaders/Diffuse`, `Legacy Shaders/Transparent/Cutout`, etc.
- `UI/Default` (for UI)
- `Particle/Alpha Blended`, `Particle/Additive`

**Also available in game build:**
- Ocean shader (custom, `oceanShader` — stored on Ocean component)
- AmplifyOcclusion shaders (from DLL)
- Wake material (`_Strength` property) — Custom shader

**For `splash.bundle` compatibility:** use **Built-in RP** shaders:
- `Standard` — primary PBR
- `Unlit/Transparent` — for particle/splash effects
- `Particle/Alpha Blended` — for particle systems

**DO NOT use:**
- URP/HDRP shaders (`.shadergraph`, `Shader.Graph`)
- `Universal Render Pipeline/Lit` — not in Built-in build

**Shader.Find:** works at runtime for shaders included in the build. `Standard` and `Unlit` are always included. Game custom shaders — **not via Shader.Find**, they're on specific Material components.
