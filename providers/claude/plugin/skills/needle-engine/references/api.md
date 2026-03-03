# Needle Engine — Full API Reference

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

// Input
this.context.input.getPointerDown(index)     // pointer just pressed
this.context.input.getPointerUp(index)       // pointer just released
this.context.input.getPointerPressed(index)  // pointer held
this.context.input.getPointerPosition(index) // {x, y} in screen pixels
this.context.input.getKeyDown(key)           // "Space", "ArrowLeft", "a", etc.
this.context.input.getKeyUp(key)
this.context.input.getKeyPressed(key)

// Physics
this.context.physics.raycast()               // hits visible geometry (no collider needed, uses auto-generated BVH for speed)
this.context.physics.raycastPhysics()        // hits Rapier colliders only
this.context.physics.engine                  // Needle's Rapier wrapper (WASM loaded lazily on first use)
this.context.physics.engine.world            // underlying Rapier world directly

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

## Animation

### Animation (simple clip playback)
```ts
import { Animation } from "@needle-tools/engine";

const anim = this.gameObject.getComponent(Animation);
anim.play();                       // play default clip
anim.play("Idle");                 // play by clip name
anim.stop();
anim.loop = true;                  // loop playback (default: true)
anim.playAutomatically = true;     // auto-play on enable (default: true)
```

### Animator (state machine — Unity Animator Controller)
```ts
import { Animator } from "@needle-tools/engine";

const anim = this.gameObject.getComponent(Animator);

anim.play("Run");                  // play by state name
anim.setFloat("Speed", 1.5);      // Animator parameters (match Unity parameter names)
anim.setBool("IsGrounded", true);
anim.setTrigger("Jump");
anim.speed = 0.5;                  // global playback speed multiplier
```

### PlayableDirector (Timeline)
```ts
import { PlayableDirector } from "@needle-tools/engine";

const director = this.gameObject.getComponent(PlayableDirector);
director.play();                   // start playback
director.pause();
director.stop();
director.time = 2.5;              // scrub to time (seconds)
director.evaluate();              // evaluate at current time (use after setting time)
director.isPlaying                // check playback state
director.isPaused
director.duration                 // total duration in seconds
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

## Networking

`this.context.connection` is the core networking manager. It handles WebSocket connections, room management, and message passing. `SyncedRoom` is a convenience component that auto-joins rooms (via URL params, random names, auto-reconnect, UI buttons) — but is **not required**. You can call `connection.connect()` and `connection.joinRoom()` directly to build custom networking.

### High-level (SyncedRoom + @syncField)
```ts
// SyncedRoom auto-manages room joining via URL params, reconnection, and menu UI
// Just add a SyncedRoom component to your scene, then in your component:
@syncField() score: number = 0;                  // auto-syncs on change

// Complex type — must reassign to trigger sync:
@syncField() items: string[] = [];
this.items.push("sword");
this.items = this.items;   // ← triggers sync
```

### Low-level (connection API directly)
```ts
// Connect and join a room manually — no SyncedRoom needed
this.context.connection.connect();
this.context.connection.joinRoom("my-room");

// Send and receive custom messages
this.context.connection.send("my-event", { data: 42 });
this.context.connection.beginListen("my-event", (msg) => { console.log(msg.data); });
```

---

## WebXR

```ts
import { Behaviour, NeedleXREventArgs, NeedleXRControllerEventArgs, registerType } from "@needle-tools/engine";

@registerType
class MyXRComponent extends Behaviour {

  // Check XR state anytime
  // this.context.xr?.isInXR
  // this.context.xr?.session  // XRSession

  onEnterXR(args: NeedleXREventArgs) {
    console.log("Entered XR, mode:", args.xr.mode);
  }

  onUpdateXR(args: NeedleXREventArgs) {
    const controllers = args.xr.controllers;
    for (const ctrl of controllers) {
      // ctrl.gamepad, ctrl.raycastHit, ctrl.grip, ctrl.hand, etc.
    }
  }

  onLeaveXR(args: NeedleXREventArgs) {
    console.log("Left XR");
  }

  onXRControllerAdded(args: NeedleXRControllerEventArgs) {
    console.log("Controller added:", args.controller);
  }
}
```
