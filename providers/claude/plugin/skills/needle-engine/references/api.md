# Needle Engine — Core API Reference

## Table of Contents
- [Lifecycle Methods](#lifecycle-methods-complete)
- [Decorators](#decorators)
- [Context API](#context-api-thiscontext)
- [GameObject Utilities](#gameobject-utilities)
- [Finding Objects](#finding-objects)
- [Coroutines](#coroutines)
- [Asset Loading at Runtime](#asset-loading-at-runtime)
- [Renderer and Materials](#renderer-and-materials)

---

## Lifecycle Methods (complete)

All methods are optional — only implement what you need.

```ts
class MyComponent extends Behaviour {
  // Initialization
  awake()        // first, before Start, even if disabled
  onEnable()     // whenever component/GO becomes active
  start()        // once, on first enabled frame

  // Per-frame
  earlyUpdate()  // every frame, before update()
  update()       // every frame
  lateUpdate()   // every frame, after all update() runs
  onBeforeRender(frame: XRFrame | null)  // just before Three.js renders
  onAfterRender()                        // just after Three.js renders

  // Deactivation / cleanup
  onDisable()    // when component/GO becomes inactive
  onDestroy()    // called by GameObject.destroy() — NOT by removeComponent()

  // Pointer events (requires an EventSystem + Raycaster in the scene)
  onPointerEnter?(args: PointerEventData)   // pointer enters this object
  onPointerMove?(args: PointerEventData)    // pointer moves over this object
  onPointerExit?(args: PointerEventData)    // pointer leaves this object
  onPointerDown?(args: PointerEventData)    // pointer button pressed on this object
  onPointerUp?(args: PointerEventData)      // pointer button released
  onPointerClick?(args: PointerEventData)   // full click on this object

  // XR events
  supportsXR?(mode: XRSessionMode): boolean            // filter which XR modes this component handles
  onBeforeXR?(mode: XRSessionMode, args: XRSessionInit) // modify session init params
  onEnterXR?(args: NeedleXREventArgs)                  // joined an XR session
  onUpdateXR?(args: NeedleXREventArgs)                 // per-frame during XR
  onLeaveXR?(args: NeedleXREventArgs)                  // left the XR session
  onXRControllerAdded?(args: NeedleXRControllerEventArgs)   // controller connected
  onXRControllerRemoved?(args: NeedleXRControllerEventArgs) // controller disconnected

  // Physics (requires Needle Collider component on same GameObject)
  onCollisionEnter(col: Collision)
  onCollisionStay(col: Collision)
  onCollisionExit(col: Collision)
  onTriggerEnter(col: Collision)
  onTriggerStay(col: Collision)
  onTriggerExit(col: Collision)
}
```

---

## Decorators

| Decorator | Purpose |
|---|---|
| `@registerType` | Required on every component — registers the class for GLB deserialization |
| `@serializable()` | Serialize/deserialize a primitive (number, string, boolean) |
| `@serializable(Type)` | Serialize/deserialize a typed field (Object3D, Texture, Color, etc.) |
| `@syncField()` | Auto-sync field over the network in a SyncedRoom |
| `@syncField(onChange)` | Sync + call a callback when value changes remotely |


### Serializable Types

```ts
// Primitives — no type argument needed
@serializable() myNumber!: number;
@serializable() myString!: string;
@serializable() myBool!: boolean;

// Complex types — pass the constructor
import { RGBAColor, AssetReference } from "@needle-tools/engine";
import { Object3D, Texture, Vector2, Vector3, Color } from "three";

@serializable(Object3D)      myRef!: Object3D;
@serializable(Texture)       tex!: Texture;
@serializable(RGBAColor)     col!: RGBAColor;
@serializable(AssetReference) asset!: AssetReference;
@serializable(Vector3)       pos!: Vector3;
```

---

## Context API (`this.context`)

```ts
this.context.scene          // THREE.Scene
this.context.mainCamera     // THREE.Camera (currently active)
this.context.renderer       // THREE.WebGLRenderer
this.context.domElement     // <needle-engine> HTML element

// Time
this.context.time.frame       // frame counter (number)
this.context.time.deltaTime   // seconds since last frame (affected by timeScale)
this.context.time.time        // total elapsed seconds
this.context.time.realtimeSinceStartup
this.context.time.timeScale   // default 1; affects deltaTime, animation, and audio

// Input — polling API (check in update())
this.context.input.getPointerDown(index)     // pointer just pressed this frame
this.context.input.getPointerUp(index)       // pointer just released this frame
this.context.input.getPointerPressed(index)  // pointer currently held
this.context.input.getPointerPosition(index) // {x, y} in screen pixels
this.context.input.getPointerPositionDelta(index)  // movement since last frame
this.context.input.getPointerPressedCount()  // how many pointers are pressed
this.context.input.mousePosition             // shortcut for pointer 0 position
this.context.input.getKeyDown(key)           // "Space", "ArrowLeft", "a", etc.
this.context.input.getKeyUp(key)
this.context.input.getKeyPressed(key)

// Input — event-based API (subscribe/unsubscribe)
this.context.input.addEventListener("pointerdown", (evt) => { /* NEPointerEvent */ });
this.context.input.addEventListener("pointerup", callback);
this.context.input.addEventListener("pointermove", callback);
this.context.input.addEventListener("keydown", callback);
this.context.input.removeEventListener("pointerdown", callback);

// Component pointer callbacks (require EventSystem + Raycaster in the scene):
// onPointerEnter, onPointerMove, onPointerExit, onPointerDown, onPointerUp, onPointerClick
// These fire on the specific object the pointer interacts with (see Lifecycle Methods)

// Physics — two raycast systems for different purposes:
// 1. Visual raycast: hits rendered geometry (no collider needed)
//    Automatically builds MeshBVH (three-mesh-bvh) on web workers on first raycast per object
//    Use for: UI interaction, picking visible objects, click detection
this.context.physics.raycast()               // from mouse position by default
this.context.physics.raycast({ screenPoint, maxDistance, layerMask, ignore })

// 2. Physics engine raycast: hits Rapier colliders only
//    Use for: ground detection, line-of-sight, physics-based queries
this.context.physics.engine?.raycast(origin, direction, { maxDistance, solid })
this.context.physics.engine?.raycastAndGetNormal(origin, direction)
this.context.physics.engine?.sphereOverlap(position, radius)

// Access Rapier world directly for advanced queries:
this.context.physics.engine.world            // underlying Rapier world

// Network
this.context.connection                      // core networking manager (usable directly or via SyncedRoom)
```

---

## GameObject Utilities

```ts
import { instantiate, destroy, GameObject } from "@needle-tools/engine";

// Component access
go.getComponent(Type)
go.getComponentInChildren(Type)
go.getComponentInParent(Type)
go.getComponents(Type)         // all matching on same GO
go.getComponentsInChildren(Type)

// Lifecycle
instantiate(source, opts?)              // preferred — clone; opts: { position, rotation, parent }
destroy(obj)                            // destroys GO + calls onDestroy on components
obj.removeComponent(comp)               // removes without calling onDestroy

// Active state
go.visible = false           // hides in scene (still ticks)
GameObject.setActive(go, false)  // disables lifecycle callbacks

// Hierarchy
go.contains(otherObj)        // true if otherObj is a descendant (Needle extension on Object3D)

// World-space properties (Needle extensions on Object3D)
go.worldPosition             // get/set world position (Vector3)
go.worldQuaternion           // get/set world rotation (Quaternion)
go.worldScale                // get/set world scale (Vector3)
go.worldForward              // forward direction in world space (Vector3)
go.worldRight                // right direction in world space (Vector3)
go.worldUp                   // up direction in world space (Vector3)

// Tag / name
go.name          // string
go.userData.tags // string[] (set from Unity via Tag component)
```

---

## Finding Objects

```ts
import { findObjectOfType, findObjectsOfType } from "@needle-tools/engine";

findObjectOfType(MyComponent, ctx)          // first match in scene
findObjectsOfType(MyComponent, ctx)         // all matches
ctx.scene.getObjectByName("Player")         // by name (Three.js built-in)
```

---

## Coroutines

Generator functions that can yield across frames:

```ts
import { WaitForSeconds, WaitForFrames, delayForFrames } from "@needle-tools/engine";

start() {
  this.startCoroutine(this.flashLight());
}

*flashLight() {
  while (true) {
    this.light.visible = !this.light.visible;
    yield WaitForSeconds(0.5);   // wait 0.5 seconds
    // yield;                    // wait exactly one frame
    // yield WaitForFrames(10);  // wait N frames
  }
}

// Stop all coroutines on this component:
this.stopAllCoroutines();

// Async alternative (returns a Promise):
await delayForFrames(5);
```

---

## Asset Loading at Runtime

```ts
import { AssetReference } from "@needle-tools/engine";

// Declare in component (set in Unity Inspector)
@serializable(AssetReference) prefab!: AssetReference;

async start() {
  // Load and instantiate
  const instance = await this.prefab.instantiate({ parent: this.gameObject });

  // Or just load the GLTF (no instantiate)
  const gltf = await this.prefab.loadAssetAsync();
}
```

Load a GLB by URL at runtime:
```ts
import { AssetReference } from "@needle-tools/engine";

const ref = AssetReference.getOrCreate(this.context.domElement.baseURI, "assets/extra.glb");
const instance = await ref.instantiate({ parent: this.gameObject });
```

Load any asset directly (without AssetReference):
```ts
import { loadAsset } from "@needle-tools/engine";

const model = await loadAsset("assets/model.glb");
const obj = model.scene; // ← Object3D is on .scene, not the return value itself
obj.traverse(n => { /* ... */ });
```

> **`loadAsset()` returns a model wrapper** (with `.scene`, `.animations`, etc.) — not an Object3D directly. The wrapper type is universal regardless of format (GLB, FBX, OBJ, USDZ). Use `model.scene` to get the root Object3D.

> **Caching:** `AssetReference.getOrCreateFromUrl()` caches by URL and returns the **same Object3D** on repeated calls. Adding a cached object to the scene again just moves it. Use `.instantiate()` or call `loadAsset()` with `{ context }` for independent copies.

> **Note:** Needle Engine automatically handles KTX, Draco, and meshopt decompression — no loader setup needed.

---

## Renderer and Materials

### Accessing meshes and materials
The `Renderer` component wraps Three.js meshes/materials. It's present on objects exported from Unity/Blender, but not automatically created for code-only objects — add it manually with `addComponent(Renderer)`, or access materials directly via Three.js (`(obj as Mesh).material`).

```ts
import { Renderer } from "@needle-tools/engine";

const renderer = this.gameObject.getComponent(Renderer);

// Materials
renderer.sharedMaterial            // first material (read/write)
renderer.sharedMaterials           // all materials (array, index-assignable)
renderer.sharedMaterials[0] = mat; // replace a material by index

// Meshes
renderer.sharedMesh                // first Mesh/SkinnedMesh Object3D
renderer.sharedMeshes              // all mesh Object3Ds (for multi-material groups)

// GPU Instancing — draws identical meshes in a single draw call for performance
// In Unity/Blender: enable on the material or via the Needle UI on the object
// In code:
Renderer.setInstanced(obj, true);  // enable instancing (also creates Renderer if missing)

// Visibility (without affecting hierarchy or component state)
Renderer.setVisible(obj, false);
```

### MaterialPropertyBlock — per-object material overrides
Overrides material properties (color, texture, roughness, etc.) on a **per-object** basis without creating new material instances. Multiple objects can share the same material but look different. Overrides are applied in `onBeforeRender` and restored in `onAfterRender`.

```ts
import { MaterialPropertyBlock } from "@needle-tools/engine";
import { Color, Texture } from "three";

// Get or create a property block for an object (never use the constructor directly)
const block = MaterialPropertyBlock.get(myMesh);

// Override properties
block.setOverride("color", new Color(1, 0, 0));
block.setOverride("roughness", 0.2);
block.setOverride("map", myTexture);

// Override with UV transform (e.g. for lightmaps)
block.setOverride("lightMap", lightmapTex, {
  offset: new Vector2(0.5, 0.5),
  repeat: new Vector2(2, 2)
});

// Read back
const color = block.getOverride("color")?.value;

// Remove overrides
block.removeOveride("color");     // remove one
block.clearAllOverrides();         // remove all
block.dispose();                   // remove the entire property block

// Check if an object has overrides
MaterialPropertyBlock.hasOverrides(myMesh);
```

Overrides are registered on the **Object3D**, not on the material — if you swap the material, overrides still apply to the new one. Use `dispose()` or `clearAllOverrides()` to remove them.

Common use cases: per-object colors/tinting, lightmaps, reflection probes, see-through/x-ray effects.

