---
name: needle-engine
description: Automatically provides Needle Engine context when working in a Needle Engine web project. Use this skill when editing TypeScript components, Vite config, GLB assets, or anything related to @needle-tools/engine.
allowed-tools:
  - needle_search
  - Read
  - Write
  - Edit
  - Bash
---

# Needle Engine

You are an expert in Needle Engine — a web-first 3D engine built on Three.js with a Unity/Blender-based workflow.

## When to Use This Skill

**Use when the user is:**
- Editing TypeScript files that import from `@needle-tools/engine`
- Working on a project with `vite.config.ts` that uses `needlePlugins`
- Loading or debugging `.glb` files in the browser
- Asking about component lifecycle, serialization, XR, networking, or deployment

**Do NOT use for:**
- Pure Three.js projects with no Needle Engine
- Non-web Unity/Blender work with no GLB export

---

## Quick Start

```html
<!-- Embed a scene in any HTML page -->
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

> ⚠️ **TypeScript config required:** `tsconfig.json` must have `"experimentalDecorators": true` and `"useDefineForClassFields": false` for decorators to work.

---

## Key Concepts

**Needle Engine** ships 3D scenes from Unity (or Blender) as GLB files and renders them in the browser using Three.js. TypeScript components attached to GameObjects in Unity are serialized into the GLB and re-hydrated at runtime in the browser.

### Embedding in HTML
```html
<!-- The <needle-engine> web component creates and manages a 3D context -->
<needle-engine src="assets/scene.glb"></needle-engine>
```
Access the context programmatically: `document.querySelector("needle-engine").context`

---

## Component Lifecycle

Mirrors Unity MonoBehaviour exactly:

```ts
import { Behaviour, serializable, registerType } from "@needle-tools/engine";

@registerType
export class MyComponent extends Behaviour {
  @serializable() myValue: number = 1;

  awake()        {}   // once on instantiate (before Start)
  start()        {}   // once on first frame
  update()       {}   // every frame
  lateUpdate()   {}   // every frame, after all update() calls
  fixedUpdate()  {}   // fixed timestep (physics)
  onEnable()     {}   // when component/GO becomes active
  onDisable()    {}   // when component/GO becomes inactive
  onDestroy()    {}   // on removal
  onBeforeRender(_frame: XRFrame | null) {}
}
```

### Physics Callbacks (requires Rapier collider on GameObject)

Add inside your component class:
```ts
onCollisionEnter(col: Collision) {}
onCollisionStay(col: Collision)  {}
onCollisionExit(col: Collision)  {}
onTriggerEnter(col: Collision)   {}
onTriggerStay(col: Collision)    {}
onTriggerExit(col: Collision)    {}
```

### Coroutines

Add inside your component class:
```ts
start() {
  this.startCoroutine(this.myRoutine());
}

*myRoutine() {
  console.log("Start");
  yield;              // wait one frame
  // yield WaitForSeconds(2);  // wait N seconds
  console.log("One frame later");
}
```

---

## Serialization

- `@registerType` — makes the class discoverable by the GLB deserializer
- `@serializable()` — marks a field for GLB deserialization (primitives)
- `@serializable(Object3D)` — for Three.js object references
- `@serializable(Texture)` — for textures (import Texture from "three")
- `@serializable(RGBAColor)` — for colors
- `@serializable(AssetReference)` — for lazily-loaded GLB assets

---

## Accessing the Scene

```ts
this.context.scene       // THREE.Scene
this.context.mainCamera  // active camera (THREE.Camera)
this.context.renderer    // THREE.WebGLRenderer
this.context.time.frame      // current frame number
this.context.time.deltaTime  // seconds since last frame
this.gameObject              // the THREE.Object3D this component is on
```

---

## Finding Components

```ts
this.gameObject.getComponent(MyComponent)
this.gameObject.getComponentInChildren(MyComponent)
this.context.scene.getComponentInChildren(MyComponent)

// Global search
import { findObjectOfType, findObjectsOfType } from "@needle-tools/engine";
findObjectOfType(MyComponent, this.context)
findObjectsOfType(MyComponent, this.context)
```

---

## Instantiate & Destroy

```ts
import { GameObject } from "@needle-tools/engine";

// Clone an object (like Unity Instantiate)
const clone = GameObject.instantiate(this.gameObject);
// With position/rotation:
const clone2 = GameObject.instantiate(prefab, { position: new Vector3(1,0,0) });

// Destroy
GameObject.destroy(clone);         // removes object + calls onDestroy on all components
this.gameObject.removeComponent(comp); // removes component (does NOT call onDestroy)
```

---

## Animation

```ts
import { Animator } from "@needle-tools/engine";

const animator = this.gameObject.getComponent(Animator);
animator?.play("Run");                   // play by state name
animator?.setFloat("Speed", 1.5);       // Animator parameters
animator?.setBool("IsJumping", true);
animator?.setTrigger("Jump");
```

---

## Loading Assets at Runtime

```ts
import { AssetReference } from "@needle-tools/engine";

@registerType
export class LazyLoader extends Behaviour {
  @serializable(AssetReference) sceneRef!: AssetReference;

  async loadIt() {
    const instance = await this.sceneRef.instantiate({ parent: this.gameObject });
  }
}
```

Or load a GLB URL directly using Three.js:
```ts
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
const gltf = await loader.loadAsync("assets/extra.glb");
this.context.scene.add(gltf.scene);
```

---

## Input Handling

```ts
// Polling
if (this.context.input.getPointerDown(0)) { /* pointer pressed */ }
if (this.context.input.getKeyDown("Space")) { /* space pressed */ }

// Event-based (NEPointerEvent works across mouse, touch, and XR controllers)
this.gameObject.addEventListener("pointerdown", (e: NEPointerEvent) => { });
```

### Custom Events Between Components
```ts
// Dispatch
this.gameObject.dispatchEvent(new CustomEvent("myEvent", { detail: { score: 42 } }));

// Listen from another component
other.gameObject.addEventListener("myEvent", (e: CustomEvent) => {
  console.log(e.detail.score);
});
```

---

## Physics & Raycasting

```ts
// Default raycasts hit visible geometry — no colliders needed
const hits = this.context.physics.raycast();

// Physics-based raycasts (require colliders, uses Rapier physics engine)
const physicsHits = this.context.physics.raycastPhysics();
```

---

## Networking & Multiplayer

Needle Engine has built-in multiplayer. Add a `SyncedRoom` component to enable networking.

- `@syncField()` — automatically syncs a field across all connected clients
- Primitives (string, number, boolean) sync automatically on change
- Complex types (arrays/objects) require reassignment to trigger sync: `this.myArray = this.myArray`
- Key components: `SyncedRoom`, `SyncedTransform`, `PlayerSync`, `Voip`
- Uses WebSockets + optional WebRTC peer-to-peer connections

---

## WebXR (VR & AR)

Needle Engine has built-in WebXR support for VR and AR across Meta Quest, Apple Vision Pro, and mobile AR.

- Add the `WebXR` component to enable VR/AR sessions
- Use `XRRig` to define the user's starting position — the user is parented to the rig during XR sessions
- Available components: `WebXRImageTracking`, `WebXRPlaneTracking`, `XRControllerModel`, `NeedleXRSession`

---

## Unity → Needle Cheat Sheet

| Unity (C#) | Needle Engine (TypeScript) |
|---|---|
| `MonoBehaviour` | `Behaviour` |
| `[SerializeField]` | `@serializable()` |
| `[RequireComponent]` | `@requireComponent(Type)` |
| `Instantiate(prefab)` | `GameObject.instantiate(obj)` |
| `Destroy(obj)` | `GameObject.destroy(obj)` |
| `GetComponent<T>()` | `this.gameObject.getComponent(T)` |
| `FindObjectOfType<T>()` | `findObjectOfType(T, ctx)` |
| `transform.position` | `this.gameObject.position` |
| `transform.rotation` | `this.gameObject.quaternion` |
| `Resources.Load<T>()` | `@serializable(AssetReference)` |
| `StartCoroutine()` | `this.startCoroutine()` |
| `Time.deltaTime` | `this.context.time.deltaTime` |
| `Camera.main` | `this.context.mainCamera` |
| `Debug.Log()` | `console.log()` |
| `OnCollisionEnter()` | `onCollisionEnter(col: Collision)` |
| `OnTriggerEnter()` | `onTriggerEnter(col: Collision)` |

---

## Creating a New Project

```bash
npm create needle my-app            # default Vite template
npm create needle my-app -t react   # React template
npm create needle my-app -t vue     # Vue.js template
```

Available templates: `vite` (default), `react`, `vue`, `sveltekit`, `svelte`, `nextjs`, `react-three-fiber`.

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

## Deployment

- **Needle Cloud** — `npx needle-cloud deploy`
- **Vercel / Netlify** — standard Vite web app
- **itch.io** — for games
- **Any static host** — `npm run build` produces a standard dist folder

From Unity, use built-in deployment components (e.g. `DeployToNeedleCloud`, `DeployToNetlify`).

---

## Progressive Loading (`@needle-tools/gltf-progressive`)

```ts
import { useNeedleProgressive } from "@needle-tools/gltf-progressive";
useNeedleProgressive(gltfLoader, renderer);
gltfLoader.load(url, (gltf) => scene.add(gltf.scene));
```

In Needle Engine projects this is built in — configure via **Compression & LOD Settings** in Unity.

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

- `@registerType` is required or the component won't be instantiated from GLB (Unity/Blender export adds this automatically, but hand-written components need it)
- GLB assets go in `assets/`, static files (fonts, images) in `public/`
- `useDefineForClassFields: false` must be set in `tsconfig.json` — otherwise decorators silently break field initialization
- `@syncField()` only triggers on reassignment — mutating an array/object in place won't sync; do `this.arr = this.arr`
- Physics callbacks (`onCollisionEnter` etc.) require a Rapier `Collider` component on the GameObject
- `removeComponent()` does NOT call `onDestroy` — use `GameObject.destroy()` for full cleanup

---

## References

- 📖 [Full API Reference](references/api.md) — complete lifecycle, decorators, and context API
- 🐛 [Troubleshooting](references/troubleshooting.md) — common errors and fixes
- 🧩 [Component Template](templates/my-component.ts) — annotated starter component

## Important URLs

- Docs: https://engine.needle.tools/docs/
- Samples: https://engine.needle.tools/samples/
- GitHub: https://github.com/needle-tools/needle-engine-support
- npm: https://www.npmjs.com/package/@needle-tools/engine
