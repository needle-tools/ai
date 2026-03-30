---
name: needle-engine
description: >
  Provides Needle Engine context for web-based 3D projects built on Three.js
  with the @needle-tools/engine component system. Use this skill whenever the user
  is working with Needle Engine components, GLB files exported from Unity or Blender,
  Vite configs with needlePlugins, TypeScript classes extending Behaviour, or anything
  involving @needle-tools/engine imports. Also trigger when the user mentions
  "needle engine", "needle tools", serializable decorators (@serializable, @syncField,
  @registerType), the <needle-engine> web component, 3D web apps using a component
  system on Three.js, or 3D scenes loaded from GLB in a web context — even if they
  don't explicitly name the engine.
compatibility:
  - optional: needle_search MCP tool (search Needle Engine docs, forum posts, and community answers)
---

# Needle Engine

You are an expert in Needle Engine — a web-first 3D engine built on Three.js with a component system and Unity/Blender-based workflow.

## Quick Start

```html
<needle-engine src="assets/scene.glb"></needle-engine>
<script type="module">
  import "@needle-tools/engine";
</script>
```

Minimal TypeScript component:
```ts
import { Behaviour, serializable, registerType } from "@needle-tools/engine";

@registerType
export class HelloWorld extends Behaviour {
  @serializable() message: string = "Hello!";

  start() {
    console.log(this.message);
  }
}
```

> ⚠️ **TypeScript config required:** `tsconfig.json` must have `"experimentalDecorators": true` and `"useDefineForClassFields": false` for decorators to work. Without `useDefineForClassFields: false`, TypeScript overwrites `@serializable()` properties with their default values *after* the decorator runs, silently breaking deserialization.

---

## Key Concepts

**Needle Engine** is a web-first 3D engine built on Three.js. All code is TypeScript — Unity and Blender are optional visual editors, not required. There are three ways to work:

### Workflows

**Code-only (no Unity/Blender):**
Scaffold a project with `npm create needle`, write TypeScript components, and build scenes entirely from code. Use `onStart`, `onUpdate`, and other lifecycle hooks to set up scenes, or create components extending `Behaviour`. This is a fully supported first-class workflow.

**Unity or Blender as visual editors:**
Unity and Blender act as scene editors — they manage a local Vite dev server, export scenes as GLB files into the web project's `assets/` folder (configured via `needle.config.json`), and serialize component data into glTF extensions. At runtime the engine deserializes this data and creates the corresponding TypeScript components with their configured values. The editors also run a **component compiler** (`@needle-tools/needle-component-compiler`) that watches your `src/scripts/` directory and auto-generates C# stubs (for Unity) or JSON files (for Blender, which the addon loads to generate UI) so your custom TypeScript components appear as editable components in the editor's inspector — this is a convenience feature for visual editing, not a requirement.

Everything exported from Unity/Blender is accessible from code afterwards. The editors are tools for visual scene setup; the runtime is pure web/TypeScript.

### Accessing the engine from code

**Lifecycle hooks** — standalone functions that work outside of any component class:
```ts
import { onStart, onUpdate, onBeforeRender, onDestroy } from "@needle-tools/engine";

// Each returns an unsubscribe function
const unsub = onStart(ctx => {
  console.log("Scene ready:", ctx.scene);
  // Access components, create objects, set up logic here
});

onUpdate(ctx => {
  // Runs every frame
});

// For SSR frameworks (Next.js, SvelteKit, Nuxt), use dynamic import:
import("@needle-tools/engine").then(({ onStart }) => {
  onStart(ctx => { /* ... */ });
});
```

Available hooks: `onInitialized`, `onStart`, `onUpdate`, `onBeforeRender`, `onAfterRender`, `onClear`, `onDestroy`

**From the `<needle-engine>` HTML element:**
```ts
// Synchronous (may be undefined if not yet loaded)
const ctx = document.querySelector("needle-engine")?.context;

// Async (waits for loading to finish)
const ctx = await document.querySelector("needle-engine")?.getContext();

// Event-based
document.querySelector("needle-engine")?.addEventListener("loadfinished", (ev) => {
  const ctx = ev.detail.context;
});
```

**From a framework component (React, Svelte, Vue):**
Use lifecycle hooks with dynamic imports to avoid SSR issues — see [Framework Integration](references/integration.md) for patterns.

### How data flows

1. **Scene setup** — either in Unity/Blender (visual) or in code (programmatic)
2. **Export** (if using editors) — scene → GLB with component data in glTF extensions → `assets/` folder
3. **Runtime** — `<needle-engine src="scene.glb">` loads the GLB, deserializes components, and starts the frame loop
4. **Code access** — hooks, `context` property, or components' lifecycle methods (`start`, `update`, etc.)

### `<needle-engine>` Attributes

Boolean attributes can be disabled with `="0"` (e.g. `camera-controls="0"`).

```html
<needle-engine
  src="assets/scene.glb"
  camera-controls
  auto-rotate
  autoplay
  background-color="#222"
  environment-image="studio"
  contactshadows
></needle-engine>
```

| Attribute | Description |
|---|---|
| `src` | GLB/glTF file path(s) — string, array, or comma-separated |
| `camera-controls` | Adds default OrbitControls with auto-fit if no `OrbitControls`/`ICameraController` exists in the root GLB. Disable with `="0"` for fully custom camera. To tweak defaults, get `OrbitControls` from the main camera in `onStart` |
| `auto-rotate` | Auto-rotate the camera (requires `camera-controls`) |
| `autoplay` | Auto-play animations in the loaded scene |
| `background-color` | Hex or RGB background color (e.g. `#ff0000`) |
| `background-image` | Skybox URL or preset: `studio`, `blurred-skybox`, `quicklook`, `quicklook-ar` |
| `background-blurriness` | Blur intensity for background (0–1) |
| `environment-image` | Environment lighting image URL or preset (same presets as `background-image`) |
| `contactshadows` | Enable contact shadows |
| `tone-mapping` | `none`, `linear`, `neutral`, `agx` |
| `poster` | Placeholder image URL shown while loading |
| `loadstart` / `progress` / `loadfinished` | Callback functions for loading lifecycle |

HTML attributes on `<needle-engine>` **override** the equivalent settings from the scene/Camera component. For example, `background-color="#222"` overrides whatever `Camera.backgroundColor` is set to in Unity/Blender. Remove the attribute to let the scene settings take effect.

---

## Unity → Needle Cheat Sheet

| Unity (C#) | Needle Engine (TypeScript) |
|---|---|
| `MonoBehaviour` | `Behaviour` |
| `[SerializeField]` / public field | `@serializable()` (required for all serialized fields) |
| `Instantiate(prefab)` | `instantiate(obj)` |
| `Destroy(obj)` | `destroy(obj)` |
| `GetComponent<T>()` | `this.gameObject.getComponent(T)` |
| `AddComponent<T>()` | `this.gameObject.addComponent(T)` |
| `FindObjectOfType<T>()` | `findObjectOfType(T, ctx)` |
| `transform.position` | `this.gameObject.worldPosition` (world) / `this.gameObject.position` (local) |
| `transform.rotation` | `this.gameObject.worldQuaternion` (world) / `this.gameObject.quaternion` (local) |
| `transform.localScale` | `this.gameObject.worldScale` (world) / `this.gameObject.scale` (local) |
| `Resources.Load<T>()` | No direct equivalent — use `@serializable(AssetReference)` to assign refs in editor, then `.instantiate()` or `.asset` at runtime |
| `StartCoroutine()` | `this.startCoroutine()` (in a component; unlike Unity, coroutines stop when the component is disabled) |
| `Time.deltaTime` | `this.context.time.deltaTime` |
| `Camera.main` | `this.context.mainCamera` (THREE.Camera) / `this.context.mainCameraComponent` (Needle Camera component) |
| `Debug.Log()` | `console.log()` |
| `OnCollisionEnter()` | `onCollisionEnter(col: Collision)` |
| `OnTriggerEnter()` | `onTriggerEnter(col: Collision)` |

---

## Three.js → Needle Cheat Sheet

| Three.js | Needle Engine |
|---|---|
| `new Mesh(geo, mat)` | Created in Unity/Blender, exported as GLB; access via `Renderer.sharedMesh` / `Renderer.sharedMaterials` |
| `scene.add(obj)` | `this.gameObject.add(obj)` or `instantiate(prefab)` |
| `scene.remove(obj)` | `obj.removeFromParent()` (re-parent) or `destroy(obj)` (permanent) |
| `obj.position` | `obj.position` (local) / `obj.worldPosition` (world — Needle extension) |
| `obj.quaternion` | `obj.quaternion` (local) / `obj.worldQuaternion` (world — Needle extension) |
| `obj.scale` | `obj.scale` (local) / `obj.worldScale` (world — Needle extension) |
| `obj.getWorldPosition(v)` | `obj.worldPosition` (getter, no temp vec needed) |
| `obj.traverse(cb)` | `obj.traverse(cb)` (same — it's Three.js underneath) |
| `obj.children` | `obj.children` (same) |
| `obj.parent` | `obj.parent` (same) |
| `raycaster.intersectObjects()` | `this.context.physics.raycast()` (auto BVH, faster) |
| `renderer.setAnimationLoop(cb)` | `update() {}` in a component, or `onUpdate(cb)` hook |
| `clock.getDelta()` | `this.context.time.deltaTime` |
| `new GLTFLoader().load(url)` | `AssetReference.getOrCreate(base, url)` then `.instantiate()`, or `loadAsset(url)` |

Needle Engine patches `Object3D.prototype` with component methods and world-space transforms. `this.gameObject` is the `Object3D` a component is attached to. The underlying Three.js API still works directly.

**Object3D extensions:** `getComponent`, `addComponent`, `worldPosition` (get/set), `worldQuaternion` (get/set), `worldScale` (get/set), `worldForward` (get/set), `worldRight`, `worldUp`, `contains`, `activeSelf`. World transform setters must be assigned (`obj.worldPosition = vec`) — mutating the returned vector won't apply.

**Materials & Renderer:**
```ts
// Option 1: Renderer component (available on objects exported from Unity/Blender, or add manually)
const renderer = obj.getComponent(Renderer);
renderer.sharedMaterial;           // first material
renderer.sharedMaterials[0] = mat; // assign by index

// Option 2: Direct Three.js access (always works)
const mesh = obj as THREE.Mesh;
mesh.material = new MeshStandardMaterial({ color: 0xff0000 });

// Per-object overrides without cloning materials:
const block = MaterialPropertyBlock.get(mesh);
block.setOverride("color", new Color(1, 0, 0));
```

---

## Creating a New Project

```bash
npm create needle my-app                  # Vite (default)
npm create needle my-app -t react         # React + Vite
npm create needle my-app -t vue           # Vue + Vite
npm create needle my-app -t sveltekit     # SvelteKit
npm create needle my-app -t nextjs        # Next.js
npm create needle my-app -t react-three-fiber  # R3F
```

---

## Vite Plugin System

```ts
import { defineConfig } from "vite";
import { needlePlugins } from "@needle-tools/engine/vite";

export default defineConfig(async ({ command }) => ({
  plugins: [
    ...(await needlePlugins(command, {}, {})),
  ],
}));
```

---

## `needle.config.json`

Lives in the web project root. Configures asset paths and build output for the Vite plugin and Unity/Blender integration.

```json
{
  "assetsDirectory": "assets",       // where GLB files are exported to (default: "assets")
  "buildDirectory": "dist",          // build output (default: "dist")
  "scriptsDirectory": "src/scripts", // where user components live
  "codegenDirectory": "src/generated" // auto-generated code from export
}
```

## Deployment

All Needle Engine projects are standard Vite web apps — `npm run build` produces a `dist` folder deployable anywhere.

**Needle Cloud** (recommended):
```bash
# CLI deployment
# Auth: run `npx needle-cloud login`, or set NEEDLE_CLOUD_TOKEN env var
# For CI/CD: create an access token at https://cloud.needle.tools/team (read/write permissions)
npx needle-cloud deploy dist                          # deploy the dist folder
npx needle-cloud deploy dist --name my-project        # with a project name
npx needle-cloud deploy dist --team my-team-name      # deploy to a specific team
npx needle-cloud deploy dist --token                  # prompts to paste an access token interactively
npx needle-cloud deploy dist --token <token>          # pass token directly (CI/CD scripts)

# GitHub Actions for continuous deployment:
# uses: needle-tools/deploy-to-needle-cloud-action@v1
# with: { token: ${{ secrets.NEEDLE_CLOUD_TOKEN }}, dir: ./dist, name: my-project }
```
Needle Cloud provides instant deployment, built-in networking server, automatic HTTPS, and version management.

**Other platforms:** Vercel, Netlify, GitHub Pages, itch.io, FTP — all work as standard static site deployments. From Unity/Blender, built-in deployment components provide one-click deploy to these platforms.

See the [deployment docs](https://engine.needle.tools/docs/how-to-guides/deployment/) for platform-specific guides.

---

## Networking

Needle Engine networking has three layers — use the highest-level one that fits:

| Layer | Component | Purpose |
|---|---|---|
| Low-level | `context.connection` | WebSocket rooms, send/listen custom messages, guid-based persistence |
| Convenience | `SyncedRoom` | Auto-join rooms via URL params, reconnect, join/leave UI button |
| Player management | `PlayerSync` + `PlayerState` | Auto-spawn/destroy player prefabs on join/leave (used for avatars) |

Additional networking components: `SyncedTransform` (sync position/rotation), `@syncField()` (sync custom state), `Voip` (voice chat), `ScreenCapture` (screen/camera sharing).

**Key concept — guid persistence:** Messages with a `guid` field are stored on the server as room state and sent to late joiners. Messages without `guid` are ephemeral (fire-and-forget). This is how `@syncField` and `SyncedTransform` work under the hood.

For full networking API, code examples, and details on each layer, read [references/networking.md](references/networking.md).

---

## Built-in Components (Quick Reference)

These are commonly used components — all imported from `@needle-tools/engine`. See [api.md](references/api.md) for full details.

| Component | Purpose |
|---|---|
| `Animation` / `Animator` | Play animation clips or state machines |
| `AudioSource` / `AudioListener` | Spatial audio playback (use `registerWaitForAllowAudio` for autoplay policy) |
| `VideoPlayer` | Video on 3D objects (mp4, webm, HLS) |
| `Light` | Directional, Point, Spot lights with shadows |
| `ContactShadows` | Soft ground shadows without lights |
| `Volume` | Post-processing (Bloom, SSAO, DoF, Vignette, etc.) |
| `Camera` | Camera control, field of view, switching active camera |
| `SceneSwitcher` | Load/unload multiple GLB scenes |
| `DragControls` | Drag objects in 3D (auto-ownership in multiplayer) |
| `Duplicatable` | Drag to clone objects |
| `DropListener` | Drag-and-drop files from desktop into scene |
| `SplineContainer` / `SplineWalker` | Paths and motion along curves |
| `ParticleSystem` | Particle effects (best configured via Unity/Blender) |
| `USDZExporter` | iOS AR Quick Look export |
| `Gizmos` | Debug drawing (lines, spheres, labels) |
| `ObjectUtils` | Create primitives and text from code |
| `BoxCollider` / `SphereCollider` | Physics colliders (`BoxCollider.add(mesh, { rigidbody: true })` for quick setup) |
| `Rigidbody` | Physics body (forces, impulses, gravity, kinematic mode) |
| `CharacterController` | Capsule collider + rigidbody for character movement |
| `EventList` | Unity Events — `@serializable(EventList)` + `.invoke()` |

Three.js objects work directly alongside these — `ObjectUtils.createPrimitive()` is a convenience, not a requirement. Use `new THREE.Mesh(geometry, material)` anytime.

---

## Built-in Extensions

These ship with Needle Engine and work automatically — no setup needed:

**[@needle-tools/gltf-progressive](https://github.com/needle-tools/gltf-progressive)** — Progressive LOD loading for meshes and textures. Stores multiple LOD levels inside GLB files, automatically swaps based on screen coverage at runtime. Configured in Unity/Blender via Compression & LOD Settings. Debug: `?debugprogressive`

**[@needle-tools/three-animation-pointer](https://github.com/needle-tools/three-animation-pointer)** — Implements the `KHR_animation_pointer` glTF extension. Allows animating any property (material colors, light intensity, camera FOV, custom component properties) not just transforms and morph targets.

**[@needle-tools/materialx](https://www.npmjs.com/package/@needle-tools/materialx)** — MaterialX material support via WASM. Loaded on-demand only when a GLB contains MaterialX materials. Can also load `.mtlx` files directly:
```ts
import { MaterialX } from "@needle-tools/engine";
const material = await MaterialX.loadFromUrl("materials/wood.mtlx");
```

**FastHDR / Environment maps** — Needle Engine supports ultra-fast preprocessed PMREM environment textures (KTX2-based FastHDR). Free HDRIs available at https://cloud.needle.tools/hdris
```ts
import { loadPMREM } from "@needle-tools/engine";

// Load and apply as environment lighting
const envTex = await loadPMREM("https://cloud.needle.tools/hdris/studio.ktx2", this.context.renderer);
if (envTex) this.context.scene.environment = envTex;
```
Or set directly via HTML: `<needle-engine environment-image="https://cloud.needle.tools/hdris/studio.ktx2">`

---

## Searching the Documentation

Use the `needle_search` MCP tool to find relevant docs, forum posts, and community answers:

```
needle_search("how to play animation clip from code")
needle_search("SyncedTransform multiplayer")
needle_search("deploy to Needle Cloud CI")
```

Use this *before* guessing at API details — the docs are the source of truth.

---

## Common Gotchas

- `@registerType` is required or the component won't be instantiated from GLB. Unity/Blender export adds this automatically via codegen; hand-written components need it explicitly.
- GLB assets go in `assets/`, static files (fonts, images, videos) in `public/` (configurable via `needle.config.json`)
- `useDefineForClassFields: false` in `tsconfig.json` — see the warning in Quick Start above
- `@syncField()` only triggers on reassignment — mutating an array/object in place won't sync. Do `this.arr = this.arr` to force a sync event.
- Physics callbacks (`onCollisionEnter` etc.) require a Needle `Collider` component (BoxCollider, SphereCollider ...) on the GameObject — they won't fire on mesh-only objects
- `removeComponent()` does NOT call `onDestroy` — any cleanup logic in `onDestroy` (event listeners, timers, allocated resources) will be skipped. Use `destroy(obj)` for full cleanup.
- `PlayerSync` prefab must have a `PlayerState` component — without it, the spawned instance will be immediately destroyed with an error. In Unity/Blender, add PlayerState to the prefab root.
- Prefer the standalone `instantiate()` and `destroy()` functions over `GameObject.instantiate()` / `GameObject.destroy()` — the standalone versions are the current API
- `loadAsset()` returns a model wrapper (not an Object3D) — use `.scene` to get the root Object3D
- `AssetReference.getOrCreateFromUrl()` caches by URL — loading the same URL twice returns the same Object3D. Use `.instantiate()` or `loadAsset()` with `{ context }` for multiple independent copies
- Never use `setInterval` to poll for `context` — use `onStart(ctx => { ... })` or `await element.getContext()` instead. Polling is fragile and may access partially initialized state
- There is NO `menu` attribute on `<needle-engine>` — to hide the menu, use `context.menu.setVisible(false)` from code (requires PRO license in production)

---

## References

Read these **only when needed** — don't load them all upfront:

- 📖 [Core API](references/api.md) — lifecycle, decorators, context (input, physics, time), gameobject, coroutines, asset loading, renderer/materials
- 🧩 [Components](references/components.md) — physics, animation, audio, video, lighting, post-processing, camera, scene switching, interaction, splines, particles, debug tools, utilities
- 🌐 [Networking](references/networking.md) — connection API, SyncedRoom, PlayerSync, @syncField, SyncedTransform, Voip, ScreenCapture, guid persistence
- 🥽 [WebXR](references/xr.md) — VR/AR sessions, XRRig, controllers, pointer events in XR, image tracking, depth sensing, camera access, mesh detection, DOM overlay, iOS AR, multiplayer avatars
- 🔗 [Framework Integration](references/integration.md) — React, Svelte, Vue, Next.js, SvelteKit patterns
- 💡 [Component Examples](references/examples.md) — practical examples: click handling, runtime loading, networking, materials, code-only scenes, input, coroutines
- 🐛 [Troubleshooting](references/troubleshooting.md) — error messages, unexpected behavior, build failures
- 🧩 [Component Template](templates/my-component.ts) — annotated starting point for new components

## Important URLs

- Docs: https://engine.needle.tools/docs/
- Samples: https://engine.needle.tools/samples/
- Samples index (all official samples with source): https://github.com/needle-tools/needle-engine-samples/blob/main/samples.json
- GitHub: https://github.com/needle-tools/needle-engine-support
- npm: https://www.npmjs.com/package/@needle-tools/engine
