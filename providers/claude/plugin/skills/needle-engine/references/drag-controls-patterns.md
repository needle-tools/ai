# DragControls patterns

Tested setups for common interactive objects. Field reference: [drag-controls.md](drag-controls.md). Each pattern lists only the settings that differ from the defaults; enum names are the TS ones.

**Unity vs. code defaults differ** (see [Defaults](drag-controls.md#defaults-unity-vs-code)). The values below are what the Unity components carry. In code, also set `rotation`, `allowScaling` and `xrHold` explicitly.

Only `DragControls`, `DragTarget` and `Duplicatable` are engine components. Where a setup needs game logic (rules, scoring, a machine reacting to its controls), the patterns describe what that script does with the engine API, so you can write it in a few lines.

## Table of Contents
- [How the pieces combine](#how-the-pieces-combine)
- [Controls: buttons, switches, knobs, levers, sliders](#controls)
- [Doors, lids, drawers, windows](#doors-lids-drawers-windows)
- [Furniture and room decoration](#furniture-and-room-decoration)
- [Handles that drive another object](#handles-that-drive-another-object)
- [Resizing](#resizing)
- [Rotation-only objects](#rotation-only-objects)
- [Containers and carrying contents](#containers-and-carrying-contents)
- [Board games and puzzles](#board-games-and-puzzles)
- [Snapping parts together (bricks, stacking crates)](#snapping-parts-together)
- [Machines: slots + controls + a script](#machines)
- [Script glue](#script-glue)
- [Pitfalls](#pitfalls)

---

## How the pieces combine

- **One object can carry both** a `DragControls` and a `DragTarget`. It's then a movable receiver: a pot that takes vegetables, a crate that takes another crate, a brick that takes bricks, a drawer that holds items.
- **Children with their own `DragControls`** are separate grips. A press goes to the nearest control under the pointer, so a drawer opens without moving its commode and the lid of a chest swings without carrying the chest.
- **`customTransform`** splits grip and moved object. A small icon or handle mesh is what you grab, and another object moves, turns or resizes. Several grips can drive one object.
- **`parentOnDrop`** on a `DragTarget` makes contents ride along with the target: items in a drawer, weights on a balance pan, arrows on a ballista. **`holdOccupants` / `holdAreaOccupants`** keep their rigidbodies kinematic while they're held.
- **In-place modes never enter targets.** Only moving modes (planes, `SnapToSurfaces`, `Attached`, `HitNormal`, `DynamicViewAngle`) can be dropped into a `DragTarget`. A drawer can still *be* a target.
- **`dragTargets = Require`** + a target = "only valid places". A miss returns the object to where it was picked up.
- **Filters on a `DragTarget` combine with AND**: `allowedObjects`, `targetMask` (layers, matched against `object.layers.mask`), `allowedTags`. Put each family of objects on its own layer (vegetables, arrows, bricks, crates) and set `targetMask` to that layer, so pots don't take bricks.

## Controls

All are in-place modes on a child of the device; the device itself has no `DragControls` (or a separate one for moving it).

| Control | Settings |
|---|---|
| **Push button** (springs back) | `Slide`, `axis` = press direction, `axisMax = 0.003` (sign = direction; parent units), `axisSnapMode = None`, `axisEngageThreshold = 0.001`, `returnOnRelease = Rest`. Read "pressed" as `normalizedValue > 0.5`. |
| **Two-position switch** | `Slide`, `axisMax = -0.0045`, `axisSnapMode = Steps`, `axisSteps = 1`. `axisSteps` counts the segments the travel is split into, so `1` stops only at the two ends. |
| **Three-position switch** (AM/FM/tape) | same, `axisSteps = 2`. `normalizedValue` reads `0`, `0.5`, `1`. |
| **Lever with two states** | `HingeRotate`, `axis = Z`, `rangeAngle = -40`, `angleSnapMode = Steps`, `angleSteps = 1`, `angleEngageThreshold = 15`. Read it as on/off: `lever.valueChangedNormalized.addEventListener(v => setOn(v > 0.5))`. |
| **Volume / tuning knob** (limited) | `HingeRotate`, `axis = Z`, `startAngle = 90`, `rangeAngle = -180`. Read `normalizedValue` as `0…1`. |
| **Stove dial with 6 clicks** | `HingeRotate`, `axis = Z`, `rangeAngle = -360` (free spin), `normalizedRangeAngle = 300` (reads `1` at 300°), `angleSnapMode = Steps`, `angleSteps = 6`. |
| **Linear slider** driving a light | `Slide`, `axis = Z`, `axisMin = 1.54`, `axisMax = 0` (min > max is allowed; it only decides which end reads `0`), `axisSnap = 0.2`. `valueChangedNormalized` → `light.intensity`. |
| **Turntable tonearm** | `HingeRotate`, `axis = Y`, `rangeAngle = -38`. The script reads the angle and plays when it's inside a window. |

`axis` is in the **parent's** space. For buttons on a rotated device, pick the parent axis that points into the device, not the world axis.

## Doors, lids, drawers, windows

| Object | Settings |
|---|---|
| **Self-closing door** | `HingeRotate`, `rangeAngle = -130`, `angleEngageThreshold = 11` + `engageThresholdElastic` (stiff at first), `returnOnRelease = Rest`, `returnDuration = 3`, a `returnCurve` that eases in. Optionally add `Rigidbody` + `HingeJoint` (`useLimits`, limits `-130…0`). |
| **Casement / top-hung window sash** | `HingeRotate`, `axis = Z` (side-hung, `rangeAngle = ±98`) or `X` (top-hung, `rangeAngle = 73`). Each sash is its own child. |
| **Oven door** grabbed by its handle only | `HingeRotate`, `rangeAngle = 82`, `angleSnapMode = Ends`, `angleSnap = 10` (falls shut/open near the ends), `grabPointMode = Custom`, `grabPoints = [handle]`, `checkForCollider = PathAndDestination` (doesn't swing through the pulled-out rack). |
| **Oven rack / baking sheet** | `Slide`, `axis = Z`, `axisMax = 0.32`, `checkForCollider = PathAndDestination`. The rack also has a `DragTarget` (`snapWhenPointedAt`, `snapRotation = false`, `parentOnDrop`) for pots. |
| **Drawer that holds things** | `Slide`, `axis = Z`, `axisMax = 0.36` **plus** `DragTarget` `mode = Area`, `areaObject = <child box>`, `areaFit = AutoFit`, `snapWhenPointedAt`, `parentOnDrop`, `holdAreaOccupants`. Items put in ride out with the drawer. |
| **Chest lid** | Child of the movable chest: `HingeRotate`, `rangeAngle = -176`. The chest itself: `SnapToSurfaces` + `DragTarget` Area for its contents. |
| **Cassette lid that latches** | `HingeRotate`, `axis = Y`, `rangeAngle = 72`, `angleSnapMode = Ends`, `angleSnap = 9`. The script sets `lid.enabled = false` when it is closed (locked) and enables the cassette `DragTarget` only while it's open. |

## Furniture and room decoration

| Object | Settings |
|---|---|
| **Sofa / table on the floor** | `SnapToSurfaces`, `surfaceMask = <floor layer>`, `dragTargets = Disallow`, `xrHold = AlwaysSnap`. Turning and resizing with the mouse: `edgeGrab = Rotate`, `cornerGrab = ScaleProportional`, `edgeGrabMargin ≈ 0.25`. For non-box shapes add `edgeGrabBoundsRaycast = true`; for side views `edgeGrabMode = FaceGrid`. |
| **Furniture that must not overlap** | add `checkForCollider = DestinationOnly` (resolve on release) or `DestinationOnlyStrict` (no overlapping pose ever). Colliders on all furniture. |
| **Chair turned by a ring handle** | Chair: `SnapToSurfaces`, floor `surfaceMask`. Child ring mesh: `HingeRotate`, `axis = Y`, `rangeAngle = -360`, `customTransform = chair`. The ring turns the chair, the body moves it. |
| **Picture on a wall** | `XYPlane`, `limitArea = <box on the wall, sibling of the pictures>`, `rotation = Locked`, `checkForCollider = PathAndDestination` (pictures don't overlap), `dragTargets = Disallow`, `xrHold = AlwaysSnap`. |
| **Wall lamp on one specific wall** | `SnapToSurfaces`, `surfaceObjects = [wall mesh]`, `surfaceAnchor = Pivot` (or `Custom` + `customAnchor` = the back plate), `alignToSurfaceNormal`, `rotation = Locked`. `dragStarted`/`dragEnded` toggle a wall highlight object. |
| **Floor-only / wall-only / ceiling-only** | `SnapToSurfaces` + `alignToSurfaceNormal` + a `surfaceNormalFilter` (code) that tests the world normal: floor `n.y > cos(20°)`, wall `abs(n.y) < sin(15°)`, ceiling `n.y < -cos(20°)`. The filter only sees surfaces the drag moves onto, so a drag that *starts* on a forbidden face also needs an `allowDrop` veto on release. For feedback, tint the object in `dragUpdated`. |
| **Small props** (candle holders) | `SnapToSurfaces`, `alignToSurfaceNormal`, `ignorePhysicsWhileDragging`, `Rigidbody` (falls once released). |
| **Spawn copies from a stack** | `Duplicatable` (engine) on the source + `DragControls` on the object: the press makes a clone and drags it. Used for bricks, vegetables, trees. `limitCount` caps clones. |

## Handles that drive another object

The grip is a child sprite or mesh with its own `DragControls` and `customTransform = <object>`.

- **Bigger grip for a slider:** an info icon next to the slider handle with the same `Slide` settings and `customTransform = <slider handle>`. Both drive the same handle and both fire `valueChanged`.
- **Rotation icon that swipes:** `HingeRotate`, `axis = Y`, `rangeAngle = -360`, `customTransform = <rotating object>`, `hingeInput = Swipe` (sideways movement, works at any view angle; `Circular` stops working when the axis is seen edge-on).
- **Move + "scale & turn" icon on a tree:** tree: `XZPlane`. Child icon: `ScaleAndRotate`, `axis = Y`, `rangeAngle = -360`, `customTransform = tree`. As a child, the icon grows with the tree. To keep its size, store `icon.worldScale` in `start` and write it back in `onBeforeRender`.
- **Rug resized from each edge:** see the next section.

## Resizing

| Object | Settings |
|---|---|
| **Uniform resize of a crate** | `Scale` with defaults (`scaleMask = (1,1,1)`, `minScale 0.25`, `maxScale 4` in Unity). |
| **Rug stretched from one edge (one component)** | On the rug: `XZPlane`, `edgeGrab = Stretch`, `cornerGrab = ScaleFree`, `scaleMask = (1,0,1)`, `minScale`/`maxScale`, `limitArea`. The middle moves it; an edge stretches that axis with the opposite edge fixed; a corner resizes both. |
| **Rug stretched from one edge (visible handles)** | Four edge handles in a `Handles` group that is a **sibling** of the rug (not a child, so they don't inherit its scale). Each: `Scale`, `customTransform = rug`, `customPivot = <empty on the opposite edge>` (child of the rug), `scaleMask = (1,0,0)` or `(0,0,1)`. Keep each handle on its edge with a small script: each frame, read `getBoundingBox(rug)` and set the handle's `worldPosition` to the middle of its side, plus a small margin. Moving the rug: a `DragControls` on the rug mesh, `XZPlane`, `customTransform = rug root`, `limitArea`. |
| **Picture resize corner** | child icon: `Scale`, `customTransform = picture`, `scaleMask = (1,0,1)`, `maxScale = 2`, `checkForCollider = DestinationOnly`. |
| **Squash-and-stretch balloon** (keeps volume) | handle: `Scale`, `customTransform = balloon`, `customPivot = <far end>`, `scaleMask = (-0.5,-0.5,1)`. Mask entries are **exponents**: stretching Z by `s` scales X and Y by `s^-0.5`. `returnOnRelease = Rest` springs back. |

## Rotation-only objects

| Object | Settings |
|---|---|
| **Joystick** | `Rotate`, `rotateX` and `rotateY` each `startAngle = ±25`, `rangeAngle = ∓50`, `engageThreshold = 8`; `rotateZ.enabled = false`. `returnOnRelease = Rest` with an overshoot `returnCurve` (springs to the centre). `applyAngularVelocityOnRelease = false`. Read `normalizedAngles` (copy it, it's a live buffer). |
| **Trackball you can flick** | `Rotate`, all three axes free (`rangeAngle = 360`), `Rigidbody` with `useGravity = false` and frozen position. A flick keeps it spinning. |
| **Globe on its axis** | `Rotate`, only `rotateZ` enabled, free. `Rigidbody` with frozen position and the two other rotation axes. |
| **Cannon / turret** | platform: `HingeRotate`, `axis = Y`, free. Barrel (child): `HingeRotate` with a limited pitch range (`startAngle = -14`, `rangeAngle = 41`). |
| **Robot arm** | a chain of `HingeRotate` joints, each a child of the previous one, each with its own `axis` and range (`J1 Y ±170°`, shoulder `Z startAngle 123 rangeAngle -182`, …). An IK handle (`Attached`, `rotation = Free`) is solved by a script that reads `getRestPose()` per joint and pauses while a joint is grabbed (`dragStarted` / `dragEnded`). |

## Containers and carrying contents

| Setup | Settings |
|---|---|
| **Box you drop things into** | `DragTarget` `mode = Area`, `areaObject = <child box marking the inside>`, `areaFit = AutoFit`, `snapWhenPointedAt`, `parentOnDrop`. Objects: any moving mode (`SnapToSurfaces` + `alignToSurfaceNormal` works well). |
| **Cooking pot** (carry it with its contents) | on the pot: `DragControls` `SnapToSurfaces`, `rotation = Free` **and** `DragTarget` Area, `areaFit = AutoFit`, `parentOnDrop`, `holdAreaOccupants`, `targetMask = <vegetables layer>`. |
| **Pan / pot grabbed by its handle** | `SnapToSurfaces`, `grabPointMode = Custom`, `grabPoints = [handle empties]` (each empty's scaled unit cube is the grab area), `surfaceAnchor = Pivot`, `alignToSurfaceNormal`, `ignorePhysicsWhileDragging`. |
| **Shelf with places for pots** | `DragTarget` `Slots`, `slots = [5 empties]`, `snapWhenPointedAt`, `snapRotation = false`, `parentOnDrop`, `targetMask = <pots layer>`. Stove top: same with 4 slots and `snapDistance = 0.05`. |
| **Glass with a lid and fruit inside** | glass: `DragTarget` `Slots` with one lid slot, `allowedObjects = [lid]`, `blockUnallowedObjects = false` (fruit can be aimed *through* it). Inner child: `DragTarget` Area, `capacity = 6`, `areaDropPoint = OnTop`, `adoptChildren = false`. Lid: `Attached`, `attachedDepthRaycast`. |
| **Fruit bowl** | `DragTarget` Area, `areaDropPoint = ThroughTop`, `parentOnDrop`. Fruit: `Rigidbody` + `ignorePhysicsWhileDragging`, `rotation = Free`. |
| **Balance scale** | two pans, each `DragTarget` Area, `capacity = 3`, `snapRotation = false`, `parentOnDrop` (weights tilt with the beam). The script sums `pan.occupants` each frame. |
| **Quiver / arrow box** | `DragTarget` `mode = Stacked`, `slots = [2 empties]`, `stackCount = 4`, `stackDirection = (1,0,0)`, `snapWhenPointedAt`, `parentOnDrop`, `targetMask = <arrows layer>`. Arrows: `dragTargets = Require`, `ignorePhysicsWhileDragging`, kinematic `Rigidbody`. |
| **Record crate** (pull a record out) | `DragTarget` `Slots` (5), `flyAxis = Z`, `flyHeight = -0.18` (records fly in/out along the slot's Z), `engageThreshold = 0.1` + `engageElastic` (resists before letting go). Records: `dragTargets = Require`. Platter: `Slots` with one slot, `flyHeight = 0.08`. |
| **Rug rack that swaps** | rack: `DragTarget` `Slots` (2), `dropScale = 0.38` (rugs shrink while on the rack), `flyHeight`, `swapHoverDistance`. Floor spot: `DragTarget` Single, `blockUnallowedObjects = false`, `allowedObjectDragStarted` / `allowedObjectDragEnded` show and hide a marker. Rugs: `XZPlane`, `dragTargets = Require`, `dragOffset = 0.5`. Dropping on a full slot swaps (`swapOnFullSlot`, default on). |

## Board games and puzzles

| Game | Settings |
|---|---|
| **Chess** | board: `DragTarget` `Grid`, `8 × 8`, `gridSpacing = 0.0625`, `snapOnRelease`, `parentOnDrop`, `flyDuration = 0.24`, `allowedObjects = [32 pieces]`. Pieces (children of the board): `XZPlane`, `limitArea = <board mesh>`, `dragTargets = Require`, `dragOffset = 0.26` (lifted while carried). Rules: `allowDrop` → `e.disallow()` unless `target.currentSlot` is a legal square. A lever (`HingeRotate`, 2 steps) switches rules on and off. |
| **Jigsaw** | tray: `DragTarget` `Slots` (9), `snapWhenPointedAt`, `flyHeight = 0.2`; `beforeDrop` vetoes a tile whose name suffix doesn't match `slots[e.slot]`. Storage: `DragTarget` Area with `snapOnRelease`. Tiles: `SnapToSurfaces`, `dragTargets = Require`. |
| **Shape sorter** | one `DragTarget` Single per hole, `allowedObjects = [its shape]`, `snapDistance = 0.09`, `snapOnRelease`, `flyHeight = 0.15`. Shapes: `SnapToSurfaces`, `dragTargets = Require`. On `objectDropped` the script calls `hole.releaseObject(shape)` and sinks it. |
| **Sliding-block parking puzzle (colliders)** | every block: `Slide` along its row/column (`axis` X or Z, range in cells, `axisSnap = 1`), `checkForCollider = PathAndDestinationStrict`, `BoxCollider`. Blocks stop at each other; no script. |
| **Same puzzle on a grid** | board: `DragTarget` `Grid 6 × 6`, `snapRotation = false`, `swapOnFullSlot = false`, `parentOnDrop`. Blocks: `XZPlane`, `dragTargets = Require`, `snapPoints = [one empty per cell the block covers]`, `rotation = Locked`. |
| **Seed bed** | `DragTarget` `Slots` (8), `flyHeight = 0.09`, `engageThreshold = 0.18` + `engageElastic`. `allowTarget` refuses anything that isn't a spawned seed. A planted seed is locked with `dc.dragMode = DragMode.None`; grown carrots get a `DragControls` (`planeEngageThreshold = 0.18`, elastic) and `allowDrop` refuses releases over the bed. |

## Snapping parts together

**Building bricks** (each brick receives and plugs in):
```ts
import { DragAxis, DragCollisionCheck, DragControls, DragMode, DragTarget, DragTargetMode } from "@needle-tools/engine";

const dc = brick.addComponent(DragControls);
dc.dragMode = DragMode.SnapToSurfaces;
dc.snapPoints = bottomSockets;          // empties under the brick ("Slot.*")
dc.checkForCollider = DragCollisionCheck.DestinationOnly;
dc.ignorePhysicsWhileDragging = true;

const dt = brick.addComponent(DragTarget);
dt.mode = DragTargetMode.Slots;
dt.slots = topStuds;                    // empties on top ("Stud.*")
dt.snapDistance = 0.03;
dt.snapWhenPointedAt = true;
dt.checkForCollider = true;             // landing space must be free
dt.snapRotationSteps = 4; dt.snapRotationAxis = DragAxis.Z;  // fits 4 ways around the stud axis
dt.autoRotate = true;                   // cycles orientations while hovering
dt.swapOnFullSlot = false;
dt.parentOnDrop = true;                 // the stack moves as one
dt.targetMask = bricksLayerMask;
```
A `snapPoints` entry mates with a slot, so a 2×4 brick can land on any stud with any of its sockets. Bricks also have a non-kinematic `Rigidbody` + `BoxCollider` and `Duplicatable` on the spawn copies.

**Stackable crates:** each crate `SnapToSurfaces` + `alignToSurfaceNormal`, and `DragTarget` `Slots` with one empty on its top, `snapDistance = 0.22`, `snapWhenPointedAt`, `parentOnDrop`, `swapOnFullSlot = false`, `targetMask = <crates layer>`.

**Sword in a stone:** `Attached`, `attachedDepthRaycast`, `planeEngageThreshold = 0.45`, `engageThresholdElastic`, `engageThresholdStiffness = 0.47`. It stretches toward the hand and only comes free past 0.45 units.

## Machines

A machine is a few `DragControls`/`DragTarget` components plus one script that reads them. The script holds references (`@serializable(DragControls)`, `@serializable(DragTarget)`) and only reads values or listens to events. It never does its own raycasting.

| Machine | Parts |
|---|---|
| **Cassette player** | 9 controls (buttons `Slide` + return, switches `Slide` + `Steps`, volume/tune `HingeRotate`, lid `HingeRotate`). A `DragTarget` `Slots` for the tape with `allowedTags = ["Cassette"]`, `flyAxis = Z`, disabled until the lid opens. The tape: `Attached`. The script reads `normalizedValue` every frame. |
| **Stove + oven** | 5 dials (6-step hinge), oven door, rack, stove-top `Slots`. The script maps each dial's `normalizedValue` to a plate glow and cooks whatever `plates.getOccupant(slot)` holds. |
| **Ballista** | base `HingeRotate` Y (aim). Arm `HingeRotate` (pitch) + `DragTarget` `Slots` with the arrow slot, `flyHeight = -0.92`, `targetMask`. Draw string: `Slide`, `axis = Z`, `axisMax = 0.53`, `returnOnRelease = Rest`, `returnDuration = 0.3`, overshoot `returnCurve`. On `dragEnded`: read `normalizedValue` (before the return resets it), `loadTarget.releaseObject(arrow)`, wait one frame, then apply the impulse. |
| **Cannon** | barrel `HingeRotate` + `DragTarget` (cannonballs in `allowedObjects`); `objectDropped` triggers the fire animation. |
| **Juice mixer** | button `Slide` (`axisMax = -0.01`, return). Mixes when the inner Area is full (`occupants.length >= capacity`) and the lid slot is occupied, then `releaseObject` + destroy each fruit. |

## Script glue

The calls a game-logic script needs. Everything else is component settings.

```ts
// Values (in-place modes)
dc.normalizedValue;                         // 0…1 across the range — buttons, dials, switches, string draw
dc.currentValue;                            // units / degrees from rest
dc.normalizedAngles;                        // Rotate: Vector3 0…1 per axis (live buffer)
dc.valueChangedNormalized.addEventListener(v => light.intensity = v * 5);

// Drag lifecycle
dc.dragStarted.addEventListener(e => {});  // pause IK, show highlights for legal squares
dc.dragEnded.addEventListener(e => { if (e.returning) return; /* harvest, fire */ });
dc.allowDrop.addEventListener(e => {       // every frame + once on release
  if (e.dropTarget !== board || !legal.has((e.dropTarget as DragTarget).currentSlot)) e.disallow();
});
dc.pointerState != null;                    // "being dragged right now" (hover/drag highlight)
dc.enabled = false;                         // lock a control (latched lid, planted seed)
dc.dragMode = DragMode.None;                // same, but keeps the component active
dc.getRestPose(obj);                        // authored pose that ranges are measured from

// Targets
dt.occupants;                               // live array — copy before mutating: [...dt.occupants]
dt.getOccupant(slot); dt.currentSlot;       // what's in a slot; slot under the dragged object
dt.objectDropped.addEventListener(e => e.object /* , e.slot */);
dt.objectRemoved.addEventListener(e => {});
dt.allowTarget.addEventListener(e => { if (!ok(e.object)) e.disallow(); });   // once per drag
dt.beforeDrop.addEventListener(e => { if (!fits(e.object, dt.slots[e.slot])) e.disallow(); });
dt.allowedObjectDragStarted.addEventListener(e => marker.visible = true);    // light up valid targets
dt.releaseObject(obj);                      // take it out before you fly, sink or destroy it
dt.enabled = open;                          // a slot that only accepts while a lid is open

// Spawn-and-drag in one press (seed pouch, Duplicatable does the same)
onPointerDown(args: PointerEventData) {
  const clone = instantiate(template, { position: args.point, keepWorldPosition: true, visible: true });
  clone.getComponent(DragControls)?.onPointerDown(args);   // forward onPointerUp the same way
}
```

## Pitfalls

- **Impulses on release get overwritten.** `DragControls` writes its own release velocity onto the dragged rigidbodies *after* `dragEnded`. Call `releaseObject` first, then `await delayForFrames(1)`, then apply the force.
- **`resetToRest` / `returnOnRelease` resets `normalizedValue`.** Read the value at the start of `dragEnded`.
- **Handles parented to the object they resize grow with it.** Put them in a sibling group and set their positions from the target's bounds (`getBoundingBox`) every frame.
- **`limitArea` must not be a child of the dragged object.** Chess pieces are children of the board target and use the board *mesh* (a sibling) as `limitArea`.
- **Settings that do nothing on their own:**
  - `customAnchor` without `surfaceAnchor = Custom`.
  - `edgeGrabMargin` without `edgeGrab` / `cornerGrab`.
  - `snapGridSpace = Local` without `snapGridResolution > 0`.
  - `snapDistance` on `Area` targets.
- **A container target blocks aiming at what's inside it.** Set `blockUnallowedObjects = false` on the outer target (a glass with fruit inside, a floor marker under furniture).
- **Names lose their dots at runtime.** `PuzzleTile.006` arrives as `PuzzleTile006`; match name suffixes with `/\.?(\d{3})$/`.
- **`occupants` and event args are reused buffers.** Spread them before iterating while you release or destroy.
