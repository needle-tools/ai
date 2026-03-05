---
name: needle-engine
description: Automatically provides Needle Engine context when working in a Needle Engine web project. Use this skill when editing TypeScript components, Vite config, GLB assets, or anything related to @needle-tools/engine.
reviewed-against: "@needle-tools/engine@4.16.0"
---

# Needle Engine

You are an expert in Needle Engine — a web-first 3D engine built on Three.js with a Unity/Blender-based workflow.

## Key concepts

**Needle Engine** ships 3D scenes from Unity (or Blender) as GLB files and renders them in the browser using Three.js. TypeScript components attached to GameObjects in Unity are serialized into the GLB and re-hydrated at runtime in the browser.

### Embedding in HTML
```html
<!-- The <needle-engine> web component creates and manages a 3D context -->
<needle-engine src="assets/scene.glb"></needle-engine>
```
Access the context programmatically: `document.querySelector("needle-engine").context`

### Component lifecycle (mirrors Unity MonoBehaviour)
```ts
import { Behaviour, serializable, registerType } from "@needle-tools/engine";

@registerType
export class MyComponent extends Behaviour {
  @serializable() myValue: number = 1;

  awake()  {}   // called once when instantiated
  start()  {}   // called once on first frame
  update() {}   // called every frame
  onEnable()  {}
  onDisable() {}
  onDestroy() {}
  onBeforeRender(_frame: XRFrame | null) {}
}
```

### Serialization
- `@registerType` — makes the class discoverable by the GLB deserializer
- `@serializable()` — marks a field for GLB deserialization (primitives)
- `@serializable(Object3D)` — for Three.js object references
- `@serializable(Texture)` — for textures (import Texture from "three")
- `@serializable(RGBAColor)` — for colors

### Accessing the scene
```ts
this.context.scene       // THREE.Scene
this.context.mainCamera  // active camera (THREE.Camera)
this.context.renderer    // THREE.WebGLRenderer
this.context.time.frame  // current frame number
this.context.time.deltaTime  // seconds since last frame
this.gameObject          // the THREE.Object3D this component is on
```

### Finding components
```ts
this.gameObject.getComponent(MyComponent)
this.gameObject.getComponentInChildren(MyComponent)
this.context.scene.getComponentInChildren(MyComponent)

// Global search (import as standalone functions from "@needle-tools/engine")
import { findObjectOfType, findObjectsOfType } from "@needle-tools/engine";
findObjectOfType(MyComponent, this.context)
findObjectsOfType(MyComponent, this.context)
```

### Input handling
```ts
// Polling
if (this.context.input.getPointerDown(0)) { /* pointer pressed */ }
if (this.context.input.getKeyDown("Space")) { /* space pressed */ }

// Event-based (NEPointerEvent works across mouse, touch, and XR controllers)
this.gameObject.addEventListener("pointerdown", (e: NEPointerEvent) => { });
```

### Physics & raycasting
```ts
// Default raycasts hit visible geometry — no colliders needed
// Uses mesh BVH (bounding volume hierarchy) for accelerated raycasting, BVH is generated on a worker
const hits = this.context.physics.raycast();

// Physics-based raycasts (require colliders, uses Rapier physics engine)
const physicsHits = this.context.physics.raycastPhysics();
```

### Networking & multiplayer
Needle Engine has built-in multiplayer. Add a `SyncedRoom` component to enable networking.

- `@syncField()` — automatically syncs a field across all connected clients
- Primitives (string, number, boolean) sync automatically on change
- Complex types (arrays/objects) require reassignment to trigger sync: `this.myArray = this.myArray`
- Key components: `SyncedRoom`, `SyncedTransform`, `PlayerSync`, `Voip`
- Uses WebSockets + optional WebRTC peer-to-peer connections

### WebXR (VR & AR)
Needle Engine has built-in WebXR support for VR and AR across Meta Quest, Apple Vision Pro, and mobile AR.

- Add the `WebXR` component to enable VR/AR sessions
- Use `XRRig` to define the user's starting position — the user is parented to the rig during XR sessions
- Available components: `WebXRImageTracking`, `WebXRPlaneTracking`, `XRControllerModel`, `NeedleXRSession`

## Creating a new project

Use `create-needle` to scaffold a new Needle Engine project:
```bash
npm create needle my-app            # default Vite template
npm create needle my-app -t react   # React template
npm create needle my-app -t vue     # Vue.js template
```

Available templates: `vite` (default), `react`, `vue`, `sveltekit`, `svelte`, `nextjs`, `react-three-fiber`.

Use `npm create needle --list` to see all available templates.

## Vite plugin system

Needle Engine ships a set of Vite plugins via `needlePlugins(command, config, userSettings)`. Custom project plugins go in `vite.config.ts`.

```ts
import { defineConfig } from "vite";
import { needlePlugins } from "@needle-tools/engine/vite";

export default defineConfig(async ({ command }) => ({
  plugins: [
    ...(await needlePlugins(command, {}, {})),
  ],
}));
```

### `makeFilesLocal` — bundling all CDN dependencies

Use the `makeFilesLocal` build option to download and bundle all CDN dependencies (Draco decoder, KTX2 transcoder, fonts, etc.) into the build output. This is essential for playable ads, app store submissions, and offline PWAs where external network requests are not allowed.

```ts
// Auto-detect and download all CDN dependencies at build time:
needlePlugin({ makeFilesLocal: "auto" })

// Fine-grained control over which features to localize:
needlePlugin({ makeFilesLocal: { enabled: true, features: ["draco", "ktx2", "fonts"] } })
```

## Deployment

Projects can be deployed to:
- **Needle Cloud** — official hosting with automatic optimization (`npx needle-cloud deploy`)
- **Vercel** / **Netlify** — standard web hosting
- **itch.io** — for games and interactive experiences
- **Any static host** — Needle Engine projects are standard Vite web apps

From Unity, use built-in deployment components (e.g. `DeployToNeedleCloud`, `DeployToNetlify`).

## Progressive loading (`@needle-tools/gltf-progressive`)

Needle Engine includes `@needle-tools/gltf-progressive` for progressive streaming of 3D models and textures. It creates a tiny initial file with embedded low-quality proxy geometry, then streams higher-quality LODs on demand. Results in ~90% smaller initial downloads with instant display.

Works standalone with any three.js project:
```ts
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { WebGLRenderer } from "three";
import { useNeedleProgressive } from "@needle-tools/gltf-progressive";

const gltfLoader = new GLTFLoader();
const renderer = new WebGLRenderer();

// Register once — progressive loading happens automatically for all subsequent loads
useNeedleProgressive(gltfLoader, renderer);

gltfLoader.load(url, (gltf) => scene.add(gltf.scene));
```

In Needle Engine projects, progressive loading is built in and can be configured via the **Compression & LOD Settings** component in Unity.

## Important URLs
- Docs: https://engine.needle.tools/docs/
- Samples: https://engine.needle.tools/samples/
- GitHub: https://github.com/needle-tools/needle-engine-support
- npm: https://www.npmjs.com/package/@needle-tools/engine

## Searching the documentation

Use the `needle_search` MCP tool to find relevant docs, forum posts, and community answers.

## Common gotchas
- Components must use `@registerType` or they won't be instantiated from GLB (this is handled automatically when exporting from Unity or Blender, but must be added manually for hand-written components)
- GLB assets are in `assets/`, static files in `include/` or `public/`
- `showBalloonMessage` (since 4.16.0) accepts optional `duration`, `once`, and `key` options:
  ```ts
  showBalloonMessage("Hello!", { duration: 3, once: true, key: "my-message" })
  ```
  Use `once` + `key` to show a message only once per session even if called multiple times.
