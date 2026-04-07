# Needle Engine — Built-in Components Reference

## Table of Contents
- [Physics](#physics)
- [Animation](#animation)
- [Audio](#audio)
- [Video](#video)
- [Lighting and Shadows](#lighting-and-shadows)
- [Post-Processing](#post-processing)
- [Camera](#camera)
- [Scene Switching](#scene-switching)
- [Interaction](#interaction)
- [Splines](#splines)
- [Debug Tools](#debug-tools)
- [Utilities](#utilities)

---

## Physics

Needle Engine uses Rapier (WASM) for physics. Rapier is loaded lazily on first use.

### Colliders
```ts
import { BoxCollider, SphereCollider, MeshCollider } from "@needle-tools/engine";

// Add a box collider from code (auto-fits to mesh bounds, optionally adds rigidbody):
BoxCollider.add(myMesh, { rigidbody: true });

// Or add manually:
myObject.addComponent(BoxCollider);
myObject.addComponent(SphereCollider);
```

Collider types: `BoxCollider`, `SphereCollider`, `CapsuleCollider`, `MeshCollider`. Set `isTrigger = true` for trigger volumes.

### Rigidbody
```ts
import { Rigidbody } from "@needle-tools/engine";

const rb = myObject.getComponent(Rigidbody);
rb.useGravity = true;
rb.mass = 2.0;
rb.isKinematic = false;        // true = not affected by forces

// Forces and impulses
rb.applyForce(new Vector3(0, 10, 0));     // continuous force (acceleration, applied over time)
rb.setForce(new Vector3(0, 10, 0));       // reset + apply new force in one call
rb.applyImpulse(new Vector3(5, 0, 0));    // instant velocity change (use for jumps, hits, explosions)

// Velocity — read and write directly (ALWAYS use these instead of accessing internals)
const vel = rb.getVelocity();              // current linear velocity (Vector3)
rb.setVelocity(new Vector3(0, 0, 0));     // set linear velocity directly
rb.setVelocity(0, 0, 0);                  // also accepts x, y, z args
const angVel = rb.getAngularVelocity();    // current angular velocity
rb.setAngularVelocity(new Vector3(0, 0, 0));
rb.smoothedVelocity;                       // averaged over ~10 frames (useful for UI/predictions)

// Stopping / resetting motion
rb.resetVelocities();                      // zero out both linear and angular velocity
rb.resetForces();                          // cancel all applied forces
rb.resetTorques();                         // cancel all applied torques
rb.resetForcesAndTorques();                // cancel both forces and torques

// Positioning
rb.teleport({ x: 0, y: 5, z: 0 });       // move without physics (resets velocities/forces)

// Sleep state
rb.wakeUp();                               // wake a sleeping body
rb.isSleeping;                             // check if body is asleep
```

**Force vs Impulse:** `applyForce()` is for continuous effects (thrusters, wind) — call every frame. `applyImpulse()` is for instant one-shot velocity changes (jumps, hits, button press) — call once.

**Never access `rb._body` or internal Rapier handles directly.** All velocity and force control is available through the public methods above. For example, to brake a rolling ball on key release, use `rb.getVelocity()` + `rb.setVelocity()` — not `(rb as any)._body.linvel()`.

Key properties: `mass`, `autoMass`, `useGravity`, `gravityScale` (multiplier, 0 = no gravity), `drag` (linear damping), `angularDrag`, `isKinematic`, `lockPositionX/Y/Z`, `lockRotationX/Y/Z`, `sleepThreshold`, `dominanceGroup`, `collisionDetectionMode` (Discrete or Continuous).

API reference: https://engine.needle.tools/docs/api/Rigidbody

### Physics callbacks
Defined on components (require a Collider on the same GameObject):
```ts
onCollisionEnter(col: Collision) { /* hit something */ }
onCollisionStay(col: Collision)  { /* still touching */ }
onCollisionExit(col: Collision)  { /* separated */ }
onTriggerEnter(col: Collision)   { /* entered trigger */ }
onTriggerStay(col: Collision)
onTriggerExit(col: Collision)
```

### Raycasting
```ts
// Visual raycast (hits any visible geometry, no collider needed, BVH-accelerated)
const hits = this.context.physics.raycast();

// Physics engine raycast (hits Rapier colliders only)
const hit = this.context.physics.engine?.raycast(origin, direction);
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

## Audio

### AudioSource
```ts
import { AudioSource } from "@needle-tools/engine";

const audio = this.gameObject.getComponent(AudioSource);
audio.clip = "sounds/music.mp3";  // URL to audio file
audio.volume = 0.8;
audio.loop = true;
audio.spatialBlend = 1;           // 0 = 2D, 1 = full 3D positional
audio.play();
audio.pause();
audio.stop();

// Browser autoplay policy: audio won't play until user interaction
AudioSource.registerWaitForAllowAudio(() => {
  audio.play();
});
```

Key properties: `clip` (string/MediaStream), `volume` (0–1), `loop`, `spatialBlend` (0–1), `playOnAwake`, `pitch`, `minDistance`, `maxDistance`, `isPlaying`, `time`, `duration`.

### AudioListener
Represents the "ears" in the scene. Attach to the camera (auto-added to main camera if missing). Only one should be active.

```ts
import { AudioListener } from "@needle-tools/engine";
this.context.mainCamera?.addComponent(AudioListener);
```

---

## Video

### VideoPlayer
```ts
import { VideoPlayer } from "@needle-tools/engine";

const vp = this.gameObject.addComponent(VideoPlayer);
vp.url = "videos/intro.mp4";       // mp4, webm, or m3u8 (HLS)
vp.isLooping = true;
vp.playOnAwake = true;
vp.play();
vp.pause();
vp.stop();
vp.currentTime = 10;                // seek to 10 seconds

// Webcam / screen capture:
vp.setVideo(mediaStream);

// HLS livestreams: just set an m3u8 URL — hls.js loads automatically
vp.url = "https://stream.example.com/live.m3u8";
```

Key properties: `url`, `isLooping`, `playbackSpeed`, `muted`, `playInBackground`, `screenspace`, `isPlaying`, `videoElement`, `videoTexture`.

The video texture is applied to the object's material by default (MaterialOverride render mode). The object needs a `Renderer` component.

---

## Lighting and Shadows

### Light
```ts
import { Light } from "@needle-tools/engine";

const light = this.gameObject.getComponent(Light);
light.intensity = 1.5;
light.color.set(1, 0.95, 0.9);   // warm white
light.shadows = 2;                 // 0=None, 1=Hard, 2=Soft
light.shadowResolution = 2048;
```

Light types (set in Unity/Blender, not changeable at runtime): Directional (1), Point (2), Spot (0). Spot lights have `spotAngle` and `innerSpotAngle`. Point/Spot lights have `range`.

### ContactShadows
Soft ground shadows based on proximity — no lights needed.
```ts
import { ContactShadows } from "@needle-tools/engine";

// Auto-create fitted to scene
const shadows = ContactShadows.auto(this.context);
shadows.opacity = 0.6;
shadows.blur = 5;

// Or via HTML attribute:
// <needle-engine contactshadows="0.7">
```

### ShadowCatcher
Catches real-time shadows from light sources onto a surface. Use for AR ground planes.
```ts
import { ShadowCatcher } from "@needle-tools/engine";
const catcher = obj.addComponent(ShadowCatcher);
catcher.mode = 0;  // 0=ShadowMask, 1=Additive, 2=Occluder
```

ContactShadows = soft ambient-style, no lights needed, better performance. ShadowCatcher = accurate shadows from real lights, higher cost.

### ReflectionProbe
Provides per-object environment reflections using cubemap or HDR textures. Objects can reference a specific probe as their reflection source, producing more accurate localized reflections than a single global environment map.

```ts
import { ReflectionProbe } from "@needle-tools/engine";

// Typically set up in Unity/Blender: add ReflectionProbe to an object, assign a cubemap texture,
// then on Renderer components set the probe as "anchor override"

// Check if a material is using a reflection probe:
ReflectionProbe.isUsingReflectionProbe(material);
```

Debug: `?debugreflectionprobe` URL param. Disable all: `?noreflectionprobe`.

---

## Post-Processing

Needle Engine uses the [pmndrs postprocessing](https://github.com/pmndrs/postprocessing) library. Add and remove effects via `this.context.postprocessing`.

### API (`this.context.postprocessing`)
```ts
import { BloomEffect } from "@needle-tools/engine";

// Add/remove effects
const bloom = new BloomEffect();
bloom.intensity.value = 3;
bloom.threshold.value = 0.5;
this.context.postprocessing.addEffect(bloom);
this.context.postprocessing.removeEffect(bloom);

// Other API
this.context.postprocessing.markDirty();             // force rebuild next frame
this.context.postprocessing.effects;                 // readonly array of active effects
this.context.postprocessing.multisampling = "auto";  // "auto" or number (0 to max)
this.context.postprocessing.adaptiveResolution = true; // reduce DPR when FPS drops
```

### Built-in effect components

All imported from `@needle-tools/engine`. Properties use `VolumeParameter` — set values with `.value`:

```ts
// Bloom — glow on bright areas
const bloom = new BloomEffect();
bloom.threshold.value = 0.9;     // brightness cutoff (default: 0.9)
bloom.intensity.value = 1;       // glow strength (default: 1)
bloom.scatter.value = 0.7;       // spread (default: 0.7)

// Depth of Field — focus blur
import { DepthOfField, DepthOfFieldMode } from "@needle-tools/engine";
const dof = new DepthOfField();
dof.mode = DepthOfFieldMode.Bokeh;  // Off, Gaussian, or Bokeh
dof.focusDistance.value = 1;         // focus distance
dof.focalLength.value = 0.2;        // focus range
dof.aperture.value = 20;            // bokeh scale

// Vignette — darkened edges
const vig = new Vignette();
vig.intensity.value = 0.5;      // darkness (default: 0)
vig.color.value = { r: 0, g: 0, b: 0, a: 1 };

// Color Adjustments — exposure, contrast, hue, saturation
const ca = new ColorAdjustments();
ca.postExposure.value = 1;      // exposure (default: 1)
ca.contrast.value = 0;          // -1 to 1
ca.hueShift.value = 0;          // hue rotation
ca.saturation.value = 0;        // saturation adjustment

// Tonemapping
const tm = new ToneMappingEffect();
tm.setMode("AgX");              // ACES, AgX, Neutral, etc.
tm.exposure.value = 1;

// Chromatic Aberration — color fringing
const chr = new ChromaticAberration();
chr.intensity.value = 0.5;

// Pixelation
const pix = new PixelationEffect();
pix.granularity.value = 10;     // pixel size

// SSAO — ambient occlusion
const ssao = new ScreenSpaceAmbientOcclusion();
ssao.intensity.value = 2;
ssao.samples.value = 9;         // quality vs performance
ssao.falloff.value = 1;
ssao.color.value = new Color(0, 0, 0);

// N8AO — alternative AO (higher quality)
import { ScreenSpaceAmbientOcclusionN8, ScreenSpaceAmbientOcclusionN8QualityMode } from "@needle-tools/engine";
const n8ao = new ScreenSpaceAmbientOcclusionN8();
n8ao.aoRadius.value = 1;        // world-space radius
n8ao.intensity.value = 1;
n8ao.quality = ScreenSpaceAmbientOcclusionN8QualityMode.Medium;

// Antialiasing (SMAA)
const aa = new Antialiasing();
aa.preset.value = 2;            // 0=Low, 1=Medium, 2=High, 3=Ultra

// Tilt Shift — miniature/diorama look
const ts = new TiltShiftEffect();
ts.focusArea.value = 0.4;       // in-focus band size
ts.feather.value = 0.3;         // blur transition
ts.offset.value = 0;            // vertical offset
ts.rotation.value = 0;          // angle

// Sharpening
const sharp = new SharpeningEffect();
sharp.amount = 1;               // strength (direct property, not VolumeParameter)
sharp.radius = 1;               // radius
```

### Runtime parameter changes
```ts
// VolumeParameter values update the underlying shader uniforms immediately
bloom.intensity.value = 5;  // takes effect next frame, no rebuild needed

// Enable/disable individual effects
bloom.enabled = false;       // removes from pipeline
bloom.active = false;        // also removes from pipeline

// Enable/disable entire Volume
volume.enabled = false;      // removes all its effects from core stack
```

### Notes
- Post-processing is disabled during XR sessions.
- Multisampling auto-adjusts: disabled when SMAA is present, scales down on low FPS, scales up when stable.
- Effects are automatically ordered (Bloom before Vignette before ToneMapping, etc.). Custom effects can set `order` to control placement.
- Alpha is preserved through the pipeline.

---

## Camera

```ts
// Access the main camera
this.context.mainCamera                  // THREE.Camera
this.context.mainCameraComponent         // Needle Camera component

// Switch the active camera:
import { Camera } from "@needle-tools/engine";
const cam = targetObject.getComponent(Camera);
this.context.setCurrentCamera(cam);      // make this the active camera

// Camera properties
cam.fieldOfView = 60;
cam.nearClipPlane = 0.1;
cam.farClipPlane = 1000;
cam.orthographic = false;

// Screen to world
const ray = cam.screenPointToRay(screenX, screenY);
```

Key properties: `fieldOfView`, `nearClipPlane`, `farClipPlane`, `backgroundColor`, `orthographic`, `orthographicSize`, `clearFlags`, `targetTexture`.

### Custom camera control (first-person, etc.)
For code-only scenes where you want full camera control (first-person, fly cam, etc.):

1. Use `<needle-engine camera-controls="0">` to prevent auto-added OrbitControls
2. Remove any existing OrbitControls — they override camera rotation every frame:
```ts
import { OrbitControls } from "@needle-tools/engine";

onStart(ctx => {
  // Remove OrbitControls so they don't fight your custom camera logic
  const cam = ctx.mainCamera;
  const orbit = cam?.getComponent(OrbitControls);
  if (orbit) orbit.destroy();
});
```
3. Write a `Behaviour` component for camera control — use `update()` and the engine's input system (`this.context.input`), not raw DOM events or `requestAnimationFrame`
4. See the [FirstPersonCharacter sample](https://github.com/needle-tools/needle-engine-samples/blob/main/package/Runtime/FirstPersonController/Scripts/FirstPersonController~/FirstPersonCharacter.ts) for a working example

---

## Scene Switching

`SceneSwitcher` manages loading/unloading multiple GLB scenes — useful for multi-room apps, configurators, portfolios.

```ts
import { SceneSwitcher } from "@needle-tools/engine";

const switcher = this.gameObject.getComponent(SceneSwitcher);
await switcher.select(0);               // by index
await switcher.select("myScene");        // by name/URI
await switcher.selectNext();
await switcher.selectPrev();

// Add scenes dynamically
switcher.addScene("assets/room2.glb");

// Events
switcher.addEventListener("loadscene-finished", (e) => {
  console.log("Loaded:", e.detail.scene.url);
});
```

Key properties: `scenes` (AssetReference[]), `currentIndex`, `preloadNext`, `preloadPrevious`, `useHistory` (browser back/forward), `useKeyboard` (arrow keys), `useSwipe`, `queryParameterName` (URL param, default `"scene"`).

You can also implement scene switching yourself using `AssetReference` or `loadAsset()`:
```ts
import { AssetReference, loadAsset } from "@needle-tools/engine";

// With AssetReference (caches by URL):
const ref = AssetReference.getOrCreate(baseUrl, "assets/room2.glb");
const instance = await ref.instantiate({ parent: this.context.scene });

// With loadAsset (returns a model wrapper):
const model = await loadAsset("assets/room2.glb");
this.context.scene.add(model.scene);
```

---

## Interaction

### DragControls
Enables dragging objects in 3D. Automatically takes ownership in networked scenes.
```ts
import { DragControls, DragMode } from "@needle-tools/engine";
const drag = obj.addComponent(DragControls);
drag.dragMode = DragMode.XZPlane;       // horizontal plane
// Modes: XZPlane, Attached, HitNormal, DynamicViewAngle (default), SnapToSurfaces, None
```

### Duplicatable
Add alongside `DragControls` — dragging creates a clone instead of moving the original.
```ts
import { Duplicatable } from "@needle-tools/engine";
obj.addComponent(Duplicatable);
```

### DropListener
Enables drag-and-drop of files from the desktop into the 3D scene (GLB, FBX, OBJ, USDZ, VRM, images).
```ts
import { DropListener } from "@needle-tools/engine";
const dl = myObject.addComponent(DropListener);
dl.fitIntoVolume = true;     // auto-scale dropped objects
dl.useNetworking = true;     // sync drops to other clients

// Or load programmatically:
const loaded = await dl.loadFromURL("https://example.com/model.glb");
```

### CharacterController
Capsule collider + rigidbody for character movement. Auto-creates physics components on enable.
```ts
import { CharacterController } from "@needle-tools/engine";

const cc = this.gameObject.getComponent(CharacterController);
cc.move(new Vector3(0, 0, 0.1));  // move forward
cc.isGrounded;                     // true when touching ground

// For jumping, use the rigidbody directly:
if (cc.isGrounded) cc.rigidbody.applyImpulse(new Vector3(0, 5, 0));
```

`CharacterControllerInput` provides a ready-made WASD + Space control scheme with double-jump and animator integration.

For a full first-person controller example, see the [FirstPersonCharacter sample](https://github.com/needle-tools/needle-engine-samples/blob/main/package/Runtime/FirstPersonController/Scripts/FirstPersonController~/FirstPersonCharacter.ts).

For clickable hotspot labels on 3D objects (common in product configurators), see the [Hotspot sample](https://github.com/needle-tools/needle-engine-samples/blob/main/package/Runtime/Hotspots/Scripts/Needle.Hotspots~/Hotspot.ts).

### needle-menu (built-in UI menu)
The `<needle-menu>` web component provides a built-in hamburger menu. Components like `SyncedRoom` and `Voip` add buttons to it automatically. Access via `this.context.menu`.

```ts
// Add a button using ButtonInfo object (recommended)
this.context.menu.appendChild({
  label: "My Action",
  icon: "settings",            // Google Material Icons name
  onClick: () => { /* ... */ },
  priority: 50,                // higher = further right, always visible
});

// Or add a raw HTML button
const button = document.createElement("button");
button.textContent = "Click me";
button.onclick = () => { /* ... */ };
this.context.menu.appendChild(button);

// Control visibility (hiding requires Needle Engine PRO license in production)
this.context.menu.setVisible(false);

// Hide the Needle logo (requires license)
this.context.menu.showNeedleLogo(false);

// Set button priority (controls ordering and which buttons stay visible when space is limited)
NeedleMenu.setElementPriority(button, 90);
```

---

## Splines

### SplineContainer
Defines curves/paths in the scene. Can be created in Unity/Blender or from code.
```ts
import { SplineContainer } from "@needle-tools/engine";
import { Vector3 } from "three";

const spline = obj.addComponent(SplineContainer);
spline.addKnot({ position: new Vector3(0, 0, 0) })
      .addKnot({ position: new Vector3(5, 2, 5) })
      .addKnot({ position: new Vector3(10, 0, 0) });
spline.closed = false;

// Sample the spline (t: 0–1)
const point = spline.getPointAt(0.5);      // world-space position
const tangent = spline.getTangentAt(0.5);  // world-space tangent
```

### SplineWalker
Moves an object along a spline path.
```ts
import { SplineWalker } from "@needle-tools/engine";
const walker = obj.addComponent(SplineWalker);
walker.spline = splineContainer;
walker.duration = 5;        // seconds for full traversal
walker.autoRun = true;
walker.useLookAt = true;    // face movement direction
```

---

## Debug Tools

### Gizmos
Static methods for runtime debug drawing — shapes auto-remove after a duration (0 = one frame).
```ts
import { Gizmos } from "@needle-tools/engine";

Gizmos.DrawLine(start, end, color, duration, depthTest);
Gizmos.DrawWireSphere(center, radius, color, duration);
Gizmos.DrawRay(origin, direction, color, duration);
Gizmos.DrawLabel(position, text, size, duration);
Gizmos.DrawArrow(start, end, color, duration);
Gizmos.DrawWireBox(center, size, color, duration);
```

---

## Utilities

### EventList (Unity Events)
`EventList` is how Unity Events are serialized and invoked at runtime. Declare with `@serializable(EventList)` and call `.invoke()`.
```ts
import { EventList, serializable } from "@needle-tools/engine";

@serializable(EventList) onClick?: EventList;

// Invoke from code:
this.onClick?.invoke();

// Subscribe from code:
const unsub = this.onClick?.addEventListener(() => console.log("Clicked!"));
unsub();  // unsubscribe
```

### Creating Objects from Code
`ObjectUtils` provides convenience methods for creating primitives and text. These are helpers — you can always use standard Three.js objects directly (`new Mesh(geometry, material)`).
```ts
import { ObjectUtils, PrimitiveType } from "@needle-tools/engine";

const cube = ObjectUtils.createPrimitive(PrimitiveType.Cube, {
  color: 0xff0000,
  parent: this.gameObject,
  position: { x: 0, y: 1, z: 0 }
});

const text = ObjectUtils.createText("Hello World");
this.context.scene.add(text);
```

Available primitives: `Cube`, `Sphere`, `Quad`, `Cylinder`. For anything more complex, use Three.js geometry directly or load GLB models.

### ParticleSystem
Full particle system with emission, shape, velocity, color/size over lifetime modules. Currently best configured via Unity/Blender — difficult to set up from code only.
```ts
import { ParticleSystem } from "@needle-tools/engine";
const ps = this.gameObject.getComponent(ParticleSystem);
ps.play();
ps.stop();
ps.pause();
```
