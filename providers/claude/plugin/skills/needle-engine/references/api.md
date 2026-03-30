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

Needle Engine networking is layered. The lowest level is `this.context.connection` (WebSocket rooms + messages). Higher-level components build on it.

### Core: `this.context.connection`
```ts
// Connect and join a room — no SyncedRoom needed
this.context.connection.connect();
this.context.connection.joinRoom("my-room");

// Room state
this.context.connection.isConnected      // boolean
this.context.connection.isInRoom         // boolean
this.context.connection.connectionId     // this client's ID
this.context.connection.usersInRoom()    // all user IDs in current room

// Send and receive custom messages
this.context.connection.send("my-event", { data: 42 });
this.context.connection.beginListen("my-event", (msg) => { console.log(msg.data); });
this.context.connection.stopListen("my-event", handler);  // always clean up in onDisable/onDestroy

// Room lifecycle events (import { RoomEvents } from "@needle-tools/engine")
this.context.connection.beginListen(RoomEvents.JoinedRoom, () => { ... });
this.context.connection.beginListen(RoomEvents.LeftRoom, () => { ... });
this.context.connection.beginListen(RoomEvents.UserJoinedRoom, (evt) => { ... });
this.context.connection.beginListen(RoomEvents.UserLeftRoom, (evt) => { ... });
this.context.connection.beginListen(RoomEvents.RoomStateSent, () => { ... }); // all persisted state received
```

### Persistent vs ephemeral messages (guid)
When a message's `data` contains a `guid` field, the server stores it as room state. New users joining later receive all stored state via `RoomStateSent`. Messages without a `guid` are fire-and-forget — only currently connected users see them.

```ts
// Ephemeral — only users currently in the room receive this
this.context.connection.send("chat", { text: "hello", sender: "Alice" });

// Persistent — server stores this by guid; late joiners get it automatically
this.context.connection.send("object-color", { guid: this.guid, color: "#ff0000" });

// Read cached state for a guid (received from server or local sends)
const state = this.context.connection.tryGetState(this.guid);

// Delete persisted state (removes from server so new joiners won't get it)
this.context.connection.sendDeleteRemoteState(this.guid);

// Delete ALL room state (use with caution)
this.context.connection.sendDeleteRemoteStateAll();
```

This is how `@syncField()` and `SyncedTransform` work under the hood — they send messages with the component's `guid`, so state persists for late joiners. Understanding this lets you build custom networking that also persists correctly.

### SyncedRoom (convenience component)
Wraps `context.connection` with auto-join, URL params, random rooms, auto-reconnect, and a join/leave menu button. Add to any object — no code needed for basic room management.
```ts
import { SyncedRoom } from "@needle-tools/engine";

// Add at runtime:
myObject.addComponent(SyncedRoom, { roomName: "my-room" });
// or join a random room:
myObject.addComponent(SyncedRoom, { joinRandomRoom: true });
// or with a prefix (useful for multiple apps on same server):
myObject.addComponent(SyncedRoom, { joinRandomRoom: true, roomPrefix: "myApp_" });
```

Key properties:
| Property | Default | Description |
|---|---|---|
| `roomName` | `""` | Room to join |
| `urlParameterName` | `"room"` | URL param for room name (`?room=xyz`) |
| `joinRandomRoom` | `undefined` | Join random room if no name set |
| `autoRejoin` | `true` | Auto-reconnect on disconnect |
| `requireRoomParameter` | `false` | Only join if URL has room param |
| `createJoinButton` | `true` | Show join/leave button in menu |

### @syncField (auto-sync fields)
```ts
@syncField() score: number = 0;                  // auto-syncs on reassignment

// With change callback:
@syncField(MyClass.prototype.onHealthChange)
health: number = 100;

private onHealthChange(newVal: number, oldVal: number) {
  console.log(`Health changed: ${oldVal} → ${newVal}`);
}

// Complex types — must reassign to trigger sync:
@syncField() items: string[] = [];
this.items.push("sword");
this.items = this.items;   // ← triggers sync
```

### SyncedTransform (sync position/rotation)
Syncs an object's position, rotation, and scale across clients. Ownership is automatic — when a user interacts (e.g. via DragControls), they take ownership.
```ts
import { SyncedTransform } from "@needle-tools/engine";
// Add to any object that should be movable by networked users
myObject.addComponent(SyncedTransform);
```

### PlayerSync + PlayerState (player avatar management)
`PlayerSync` instantiates a prefab for each player joining a room and destroys it on leave. The prefab must have a `PlayerState` component. This is the recommended approach for multiplayer player objects and WebXR avatars.

```ts
import { PlayerSync, PlayerState } from "@needle-tools/engine";

// Runtime setup — load a GLB as the player prefab:
const ps = await PlayerSync.setupFrom("assets/avatar.glb");
scene.add(ps.gameObject);
// The GLB should have a PlayerState component. setupFrom() adds one if missing.

// Events:
ps.onPlayerSpawned  // EventList<Object3D> — fires when any player instance spawns
```

**PlayerState** — attached to each spawned player instance:
```ts
// Static helpers:
PlayerState.isLocalPlayer(obj)   // true if obj belongs to this client
PlayerState.all                  // all PlayerState instances in the scene
PlayerState.local                // only local player's PlayerState instances
PlayerState.getFor(obj)          // find PlayerState for an Object3D or Component

// Instance:
state.isLocalPlayer              // boolean
state.owner                      // connection ID of the owning player

// Events:
PlayerState.addEventListener(PlayerStateEvent.OwnerChanged, (evt) => {
  // evt.detail: { playerState, oldValue, newValue }
});
```

### Voice & Video: Voip and ScreenCapture
Both require an active networked room and HTTPS.

```ts
import { Voip, ScreenCapture } from "@needle-tools/engine";

// Voice chat — auto-connects when joining a room
const voip = myObject.addComponent(Voip, { autoConnect: true, createMenuButton: true });
voip.connect();       // manual start
voip.disconnect();    // manual stop
voip.setMuted(true);  // mute mic

// Screen/camera/microphone sharing
const sc = myObject.addComponent(ScreenCapture);
sc.share({ device: "Screen" });    // "Screen", "Camera", "Microphone", "Canvas"
sc.close();                         // stop sharing
// Receiving clients see the video on a VideoPlayer component on the same object
```

| Voip property | Default | Description |
|---|---|---|
| `autoConnect` | `true` | Start when joining a room |
| `runInBackground` | `true` | Stay connected when tab loses focus |
| `createMenuButton` | `true` | Show mute/unmute button in menu |

### Typical multiplayer setup
1. Add `SyncedRoom` to an object (or call `context.connection.joinRoom()` manually)
2. For player avatars: add `PlayerSync` with a prefab that has `PlayerState`
3. For synced objects: add `SyncedTransform` to movable objects
4. For custom state: use `@syncField()` on component properties
5. For custom events: use `context.connection.send()` / `beginListen()`
6. For voice chat: add `Voip` — for screen sharing: add `ScreenCapture`

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
