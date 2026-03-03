---
name: needle-engine
description: Automatically provides Needle Engine context when working in a Needle Engine web project. Use this skill when editing TypeScript components, Vite config, GLB assets, or anything related to @needle-tools/engine.
metadata:
  reviewed-against: "@needle-tools/engine@4.15.0"
  last-reviewed: "2026-03"
---

# Needle Engine

You are an expert in Needle Engine — a web-first 3D engine built on Three.js with a Unity/Blender-based workflow.

## When to Use This Skill

**Use when the user is:**
- Editing TypeScript files that import from `@needle-tools/engine`
- Working on a project with `vite.config.ts` that uses `needlePlugins`
- Loading or debugging `.glb` files exported from Unity or Blender
- Using the Needle Engine Blender addon or Unity package
- Asking about component lifecycle, serialization, XR, networking, or deployment

**Do NOT use for:**
- Pure Three.js projects with no Needle Engine
- Non-web Unity/Blender work with no GLB export

---

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

> **TypeScript config required:** `tsconfig.json` must have `"experimentalDecorators": true` and `"useDefineForClassFields": false` for decorators to work.

---

## Key Concepts

**Needle Engine** ships 3D scenes from Unity or Blender as GLB files and renders them in the browser using Three.js. TypeScript components attached to objects are serialized into the GLB and re-hydrated at runtime.

- **Unity workflow:** C# MonoBehaviours → auto-generated TypeScript stubs → GLB export on play/build
- **Blender workflow:** Components added via the Needle Engine Blender addon → GLB export with component data embedded
- **Embedding:** `<needle-engine src="assets/scene.glb">` web component creates and manages a 3D context
- **Context access:** use `onStart(ctx => { ... })` or `onInitialize(ctx => { ... })` lifecycle hooks (preferred); `document.querySelector("needle-engine").context` works but only from UI event handlers

---

## Unity → Needle Cheat Sheet

| Unity (C#) | Needle Engine (TypeScript) |
|---|---|
| `MonoBehaviour` | `Behaviour` |
| `[SerializeField]` / public field | `@serializable()` (required for all serialized fields) |
| `Instantiate(prefab)` | `instantiate(obj)` |
| `Destroy(obj)` | `destroy(obj)` |
| `GetComponent<T>()` | `this.gameObject.getComponent(T)` |
| `FindObjectOfType<T>()` | `findObjectOfType(T, ctx)` |
| `transform.position` | `this.gameObject.worldPosition` (world) / `this.gameObject.position` (local) |
| `transform.rotation` | `this.gameObject.worldQuaternion` (world) / `this.gameObject.quaternion` (local) |
| `transform.localScale` | `this.gameObject.worldScale` (world) / `this.gameObject.scale` (local) |
| `Resources.Load<T>()` | `@serializable(AssetReference)` |
| `StartCoroutine()` | `this.startCoroutine()` |
| `Time.deltaTime` | `this.context.time.deltaTime` |
| `Camera.main` | `this.context.mainCamera` |
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

Needle Engine extends `Object3D` with component methods (`getComponent`, `addComponent`, `worldPosition`, `worldQuaternion`, `worldScale`, `worldForward`, `worldRight`, `worldUp`, `contains`, etc.). `this.gameObject` is the `Object3D` a component is attached to. The underlying Three.js API still works directly.

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
- `removeComponent()` does NOT call `onDestroy` — use `destroy(obj)` for full cleanup
- Prefer `instantiate()` and `destroy()` functions over `GameObject.instantiate()` / `GameObject.destroy()`

---

## References

For detailed API usage, read these reference files:

- [Full API Reference](references/api.md) — lifecycle, decorators, context API, animation, networking, XR, physics
- [Framework Integration](references/integration.md) — React, Svelte, Vue, vanilla JS examples + SSR patterns
- [Troubleshooting](references/troubleshooting.md) — common errors and fixes
- [Component Template](templates/my-component.ts) — annotated starter component

## Important URLs

- Docs: https://engine.needle.tools/docs/
- Samples: https://engine.needle.tools/samples/
- GitHub: https://github.com/needle-tools/needle-engine-support
- npm: https://www.npmjs.com/package/@needle-tools/engine
