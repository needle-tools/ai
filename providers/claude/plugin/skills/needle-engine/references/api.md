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
  update()       // every frame
  lateUpdate()   // every frame, after all update() runs
  fixedUpdate()  // fixed timestep (default 50Hz, used for physics)
  onBeforeRender(frame: XRFrame | null)  // just before Three.js renders

  // Deactivation / cleanup
  onDisable()    // when component/GO becomes inactive
  onDestroy()    // called by GameObject.destroy() — NOT by removeComponent()

  // Physics (requires Rapier Collider component on same GameObject)
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
| `@requireComponent(Type)` | Ensure a component of Type exists on the same GO |

### Serializable Types (import from correct source)

```ts
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
this.context.time.deltaTime   // seconds since last frame
this.context.time.time        // total elapsed seconds
this.context.time.realtimeSinceStartup

// Input
this.context.input.getPointerDown(index)     // pointer just pressed
this.context.input.getPointerUp(index)       // pointer just released
this.context.input.getPointerPressed(index)  // pointer held
this.context.input.getPointerPosition(index) // {x, y} in screen pixels
this.context.input.getKeyDown(key)           // "Space", "ArrowLeft", "a", etc.
this.context.input.getKeyUp(key)
this.context.input.getKeyPressed(key)

// Physics
this.context.physics.raycast()               // hits visible geometry (no collider needed)
this.context.physics.raycastPhysics()        // hits Rapier colliders only
this.context.physics.engine                  // access Rapier world directly

// Network
this.context.connection                      // active network connection (if SyncedRoom present)
```

---

## GameObject Utilities

```ts
import { GameObject } from "@needle-tools/engine";

// Component access
go.getComponent(Type)
go.getComponentInChildren(Type)
go.getComponentInParent(Type)
go.getComponents(Type)         // all matching on same GO
go.getComponentsInChildren(Type)

// Lifecycle
GameObject.instantiate(source, opts?)   // clone; opts: { position, rotation, parent }
GameObject.destroy(obj)                  // destroys GO + calls onDestroy on components
obj.removeComponent(comp)               // removes without calling onDestroy

// Active state
go.visible = false           // hides in scene (still ticks)
GameObject.setActive(go, false)  // disables lifecycle callbacks

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
import { WaitForSeconds } from "@needle-tools/engine";

start() {
  this.startCoroutine(this.flashLight());
}

*flashLight() {
  while (true) {
    this.light.visible = !this.light.visible;
    yield WaitForSeconds(0.5);   // wait 0.5 seconds
    // yield;                    // wait exactly one frame
  }
}

// Stop all coroutines on this component:
this.stopAllCoroutines();
```

---

## Animation

```ts
import { Animator } from "@needle-tools/engine";

const anim = this.gameObject.getComponent(Animator);

anim.play("Run");                  // play by state name
anim.setFloat("Speed", 1.5);      // Animator parameters (match Unity parameter names)
anim.setBool("IsGrounded", true);
anim.setTrigger("Jump");
anim.speed = 0.5;                  // global playback speed multiplier
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

---

## Networking

```ts
import { SyncedRoom, SyncedTransform } from "@needle-tools/engine";

// In a component with a SyncedRoom in the scene:
@syncField() score: number = 0;                  // auto-syncs on change

// Complex type — must reassign to trigger sync:
@syncField() items: string[] = [];
this.items.push("sword");
this.items = this.items;   // ← triggers sync

// Low-level RPC:
this.context.connection.send("my-event", { data: 42 });
this.context.connection.beginListen("my-event", (msg) => { console.log(msg.data); });
```

---

## WebXR

```ts
import { WebXR, XRRig, NeedleXRSession } from "@needle-tools/engine";

// In code: check XR state
if (this.context.xr?.isInXR) { ... }
this.context.xr?.session   // XRSession

// XR controller input
onBeforeRender(frame: XRFrame | null) {
  if (!frame) return;
  const controllers = this.context.xr?.controllers;
  // controllers[0].gamepad, controllers[0].raycastHit, etc.
}
```
