# Needle Engine — DragControls & DragTarget

`DragControls` was rebuilt with in-place modes (slide, hinge, rotate, scale), `DragTarget` drop slots, return-on-release, collision checks, edge/corner gestures, and typed events. Older engine versions only have `XZPlane`, `Attached`, `HitNormal`, `DynamicViewAngle`, `SnapToSurfaces`, `None` plus `keepRotation`/`keepScale`. **Before using anything below, check the installed package has it** — e.g. `node <skill-path>/scripts/lookup-api.mjs <project-path> DragTarget` or look for `Slide` in `DragMode` in `node_modules/@needle-tools/engine`.

API reference: https://engine.needle.tools/docs/api/DragControls · https://engine.needle.tools/docs/api/DragTarget

**Building something specific** (a button, a dial, a drawer that holds things, chess, bricks, a machine with several controls)? Start with [drag-controls-patterns.md](drag-controls-patterns.md). It has working settings for each of these and the script calls that tie them together.

## Table of Contents
- [Setup](#setup)
- [Defaults: Unity vs. code](#defaults-unity-vs-code)
- [Drag modes](#drag-modes)
- [Which settings apply to which mode](#which-settings-apply-to-which-mode)
- [Combining settings](#combining-settings)
- [Recipes](#recipes)
- [Reading the value of a control](#reading-the-value-of-a-control)
- [Snapping and engage thresholds](#snapping-and-engage-thresholds)
- [Return on release](#return-on-release)
- [Grab point, anchor, pivot, snap points](#grab-point-anchor-pivot-snap-points)
- [Rotating and scaling while moving](#rotating-and-scaling-while-moving)
- [Edge and corner grabs](#edge-and-corner-grabs)
- [Collisions and physics](#collisions-and-physics)
- [XR](#xr)
- [Events and live state](#events-and-live-state)
- [Custom constraints and modes](#custom-constraints-and-modes)
- [DragTarget (drop slots, grids, stacks, areas)](#dragtarget)
- [Deprecated fields](#deprecated-fields)
- [Gotchas](#gotchas)

---

## Setup

```ts
import { DragControls, DragMode } from "@needle-tools/engine";
const drag = obj.addComponent(DragControls);
drag.dragMode = DragMode.XZPlane;
```

- The object (or a child) needs **visible mesh geometry**. The `EventSystem` is created automatically; DragControls no longer adds an `ObjectRaycaster`, and none is needed.
- A `Rigidbody` on the dragged object is made kinematic for the drag and restored afterwards.
- `customTransform` (every mode) moves a different object while this object's geometry stays the handle — several handles with different modes can drive one object (a lid with a grip at each end, a machine with a slider and a knob).
- Settings are grouped: `snapping` (grid, axis and angle snap, snap points), `engage` (thresholds), `rim` (edge and corner grabs), `grab` (grab points), `surface` (surface filter and anchor), `scaling` (mask, min/max) and `returning` (return on release). Set fields on the group (`drag.snapping.gridResolution = 0.25`); don't replace the group object.
- Networking: add `SyncedTransform`; ownership is taken automatically when a drag starts.
- Debug: `?debugdrag` (drag helpers) and `?debugdragtarget` (drop target logging/gizmos).

## Defaults: Unity vs. code

A component added in the Unity inspector and one created with `addComponent` start from **different defaults**. In code, `rotation`, `allowScaling` and `xrHold` stay unset and fall back to the legacy fields, so older scripts keep their behaviour; Unity stamps better defaults on new components and converts older ones from their legacy values:

| Setting | New Unity component | Older Unity component (migrated) | `addComponent(DragControls)` |
|---------|---------------------|----------------------------------|------------------------------|
| `rotation` | `YawOnly` | `Free` (`Locked` only if both legacy keep-rotation flags were on) | screen `Locked` (`keepRotation`), XR `Free` (`xrKeepRotation`) |
| `allowScaling` | `false` | `!keepScale` (usually `true`) | **`true`** (`!keepScale`) — pinch scales |
| `scaling.min` / `scaling.max` | `0.25` / `4` | `0.0001` / `1000000` (unbounded, as before) | `0.0001` / `1000000` |
| `xrHold` | `FreeThenSnap` | `Free` if `xrDragMode` was `Attached`, else `AlwaysSnap` | `Free` (from `xrDragMode = Attached`) |

In code, set these explicitly instead of relying on defaults.

## Drag modes

`dragMode` (default `DynamicViewAngle`). **Moving** modes carry the object through the scene and can use drop targets; **in-place** modes work the object where it stands and never enter a `DragTarget`.

| Mode | Kind | Use |
|------|------|-----|
| `XZPlane` / `XYPlane` / `YZPlane` | moving | Floor/table, wall, other wall. Planes are in the **parent's** local space. Limit with `limitArea`. |
| `DynamicViewAngle` | moving | XZ from above, screen plane from low angles (default). |
| `Attached` | moving | Follows the pointer at its grab depth (screen plane / XR hand). |
| `HitNormal` | moving | Along the initial hit normal. |
| `SnapToSurfaces` | moving | Places on scene surfaces; filter with `surface.mask` / `surface.objects`. |
| `Slide` | in-place | Along `axis` between `axisMin`/`axisMax`. Drawers, sliders, levers. |
| `HingeRotate` | in-place | Around `axis` at `customPivot`, `startAngle` + `rangeAngle`. Doors, lids, valves. |
| `Rotate` | in-place | Several axes at once via `rotateX`/`rotateY`/`rotateZ`. Joysticks, trackballs, gimbals. |
| `Scale` | in-place | Resize along `scaling.mask`. Handles, resizable furniture. |
| `ScaleAndRotate` | in-place | Turns about `axis` (bearing) and resizes by distance from it. Knobs that grow. |
| `None` | — | Not draggable; every other setting is inactive. |

`axis` (`DragAxis.X/Y/Z`) is always in the **parent's** local space, so a drawer in a rotated cabinet needs no special handling. Ranges are measured from the **rest pose** (the authored transform captured before the first drag), not from where the current drag started.

## Which settings apply to which mode

Each mode reads only its own settings; the rest are silently ignored. This mirrors what the Unity inspector shows per mode.

| Setting | Planes | SnapToSurfaces | Attached / HitNormal / DynamicViewAngle | Slide | HingeRotate | Rotate | Scale | ScaleAndRotate |
|---|---|---|---|---|---|---|---|---|
| `customTransform`, `returning.mode`, `checkForCollider`, `ignorePhysicsWhileDragging`, `dragStarted/Updated/Ended`, `allowDrop` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `dragTargets`, `snapping.points` | ✓ | ✓ | ✓ | | | | | |
| `limitArea`, `limitAreaFit` | ✓ | | | | | | | |
| `surface.mask/objects/normalFilter`, `surface.anchor`, `surface.customAnchor`, `surface.alignToNormal`, `surface.settle` | | ✓ | | | | | | |
| `snapping.gridResolution`, `snapping.gridSpace`, `engage.planeThreshold` | ✓ | ✓ | ✓ | | | | | |
| `dragOffset` | ✓ | ✓ | not `Attached` | | | | | |
| `rotation`, `allowScaling`, edge/corner grab | ✓ | ✓ | ✓ | | | | | |
| `grab.mode`, `grab.points` | ✓ | ✓ | ✓ | handles only | handles only | | | |
| `grab.alignToPointer` | ✓ | ✓ | ✓ | | | | | |
| `xrHold` | ✓ | ✓ | not `Attached` | | | | | |
| `xrDistanceDragFactor` | ✓ | ✓ | ✓ | | | | | |
| `attachedDepthRaycast` | | | `Attached` only | | | | | |
| `axis` | | | | ✓ | ✓ | | | ✓ |
| `customPivot` | | | | | ✓ | ✓ | ✓ | ✓ |
| `axisMin/Max`, `snapping.axisSnap/axisSteps/axisSnapMode`, `engage.axisThreshold` | | | | ✓ | | | | |
| `startAngle`, `rangeAngle`, `normalizedRangeAngle`, `snapping.angleSnap`, `engage.angleThreshold` | | | | | ✓ | | | ✓ |
| `snapping.angleSteps`, `snapping.angleSnapMode`, `hingeInput` | | | | | ✓ | | | |
| `engage.mode` | | | | ✓ | ✓ | | | |
| `rotateX/Y/Z` | | | | | | ✓ | | |
| `scaling.mask`, `scaling.min/max`, `engage.scaleThreshold` | pinch / rim resize | pinch / rim resize | pinch / rim resize | | | | ✓ | ✓ |
| `applyAngularVelocityOnRelease` | | | | | | ✓ | | |
| `valueChanged`, `valueChangedNormalized` | | | | ✓ | ✓ | first axis | factor | angle |

## Combining settings

**Prerequisites — a setting does nothing unless:**
- `rim.edge` / `rim.corner` → `rim.margin > 0`. Their `Rotate` option also needs `rotation` other than `Locked`; `Stretch` and the scale options do not need `allowScaling`.
- `rim.inXR` / `rim.inTouch` → `rim.edge` or `rim.corner` set.
- `scaling.min` / `scaling.max` in a moving mode → `allowScaling = true` (pinch), `rim.edge = Stretch` or a `rim.corner` scale option.
- `limitAreaFit` → `limitArea` is set. **`limitArea` must not be a child of the dragged object** — it would move with it and never constrain anything. Use a sibling or a scene object.
- `surface.customAnchor` → `surface.anchor = SurfaceAnchor.Custom` (falls back to the bounding box while empty).
- `grab.points` / `grab.alignToPointer` → `grab.mode = GrabPoint.Custom` (empty list falls back to the hit point). Each grab point needs a non-zero scale on every axis (lossy scale counts) — its scaled unit cube is the grab area.
- `engage.mode` → `engage.axisThreshold` / `engage.angleThreshold` > 0.
- `engage.elastic` / `engage.stiffness` → some engage threshold > 0. They are **one** shared setting for every threshold on the component.
- `snapping.gridSpace` → `snapping.gridResolution > 0`.
- `returning.delay` / `returning.duration` / `returning.curve` → `returning.mode` ≠ `None`. `returning.curve` is unused while `returning.duration = 0`.
- `snapping.points` → `dragTargets` ≠ `Disallow` and a `DragTarget` in the scene.
- `applyAngularVelocityOnRelease` → a `Rigidbody` on the object (Unity: "Add Rotation Physics Component").

**No effect / mode limits:**
- `GrabPoint.Pivot` has no effect in `Slide`/`HingeRotate` (the object is never re-hung on the pointer); use `Custom` there to restrict where it can be grabbed.
- `ScaleAndRotate` has only plain `snapping.angleSnap` spacing — no `snapping.angleSteps`/`snapping.angleSnapMode`, no `engage.mode`, no `hingeInput` (it always reads the bearing around the axis).
- `Rotate` with all three axes enabled and **any** limit, snap or engage threshold runs as a gimbal: the middle axis only reaches ±90°. With no limits it is a free trackball, but the reported angles are one of several equivalent readings. For meaningful angles, disable an axis or drop the limits.
- `Scale` / `ScaleAndRotate` with `scaling.mask = (0, 0, 0)` never resize (`Scale` then does nothing at all).
- `rangeAngle` ≥ 360 (abs) is free rotation, not a limit; the value still sweeps once per revolution. A range that doesn't contain 0° (the rest pose) is allowed, but the first drag carries the object in and it can't come back out — usually a range measured against the model instead of the pivot.
- `customPivot` set to the dragged object itself is the same as leaving it empty.
- `xrHold` is ignored by all in-place modes (they always hold) and has nothing to settle into with `Attached`.
- Edge/corner grabs are mouse and pen only unless `rim.inTouch = true` (touch, handheld AR included) or `rim.inXR = true`; by default fingers and XR hands carry by the edge, and pinch to turn and scale.
- `surface.settle = false` lets a `SnapToSurfaces` object leave the surface edge and be dropped in mid-air.

**Precedence when settings compete for where the object ends up:**
1. `DragControls.allowDrop` refusal or `DragTarget.beforeDrop` refusal → back to where this drag picked it up.
2. A `DragTarget` that accepts the object → it stays in the target (beats `returning.mode`).
3. `dragTargets = Require` with no target → back to the pickup pose.
4. `returning.mode` (`Rest` / `DragStart`).
5. `checkForCollider = DestinationOnly` release correction, XR `FreeThenSnap` settle.

**`scaling.min` / `scaling.max` mean different things:** in `Scale`/`ScaleAndRotate` and for edge/corner resizes they are multiples of the **authored** size (same limits across drags); for pinch they are multiples of the **drag-start** scale (`scaling.relativeToStart`, code-only, default `true`; `false` = absolute world scale per axis).

**Good pairs:** `ignorePhysicsWhileDragging` + `checkForCollider` (place by testing colliders, nothing gets scattered) · `Slide`/`HingeRotate` + `engage.axisThreshold`/`engage.angleThreshold` + `engage.mode = FromStart` (sticks shut, free once open) · `Rotate` + `returning.mode = Rest` + `returning.curve` overshoot (springy joystick) · `SnapToSurfaces` + `surface.objects` + `surface.normalFilter` (floor only, no walls) · moving mode + `dragTargets = Require` + `DragTarget` (board games).

## Recipes

```ts
import {
  DragControls, DragMode, DragAxis, DragReturn, DragRotationMode,
} from "@needle-tools/engine";

// Furniture on the floor, inside a rug, on a 0.25 grid, turnable upright
const chair = chairObj.addComponent(DragControls);
chair.dragMode = DragMode.XZPlane;
chair.limitArea = rugObj;            // its scaled unit cube is the region; not a child of chairObj
chair.snapping.gridResolution = 0.25;
chair.rotation = DragRotationMode.YawOnly;
chair.allowScaling = false;          // code default would allow pinch scaling

// Drawer: 0.4 units out along parent X
const drawer = drawerObj.addComponent(DragControls);
drawer.dragMode = DragMode.Slide;
drawer.axis = DragAxis.X;
drawer.axisMin = 0;                  // closed = authored position
drawer.axisMax = 0.4;                // may be negative for the other direction

// Door: 90° around an empty at the hinge edge
const door = doorObj.addComponent(DragControls);
door.dragMode = DragMode.HingeRotate;
door.customPivot = hingeEmpty;       // only its position is read, once per drag
door.axis = DragAxis.Y;
door.startAngle = 0;                 // degrees from the authored pose
door.rangeAngle = 90;                // signed; ≥360 = free spin

// Two-axis joystick that springs back to center
const stick = stickObj.addComponent(DragControls);
stick.dragMode = DragMode.Rotate;
stick.rotateY.enabled = false;       // two axes: no gimbal flip
stick.rotateX.startAngle = -30; stick.rotateX.rangeAngle = 60;
stick.rotateZ.startAngle = -15; stick.rotateZ.rangeAngle = 30;
stick.returning.mode = DragReturn.Rest;

// Height-only resize handle
const handle = boxObj.addComponent(DragControls);
handle.dragMode = DragMode.Scale;
handle.scaling.mask.set(0, 1, 0);       // exponents per axis; 0 = never written
handle.scaling.min = 0.5; handle.scaling.max = 3;   // multiples of the authored size

// Place props on the floor mesh only
const prop = propObj.addComponent(DragControls);
prop.dragMode = DragMode.SnapToSurfaces;
prop.surface.objects = [floorObj];    // descendants count; ANDed with surface.mask (layers)
prop.surface.normalFilter = n => n.y > 0.8;  // code-only: refuse walls/steep faces
```

### Several controls on one object

DragControls are built to be combined: each child with its own `DragControls` is a separate grip, and a press goes to the nearest one under the pointer. So the parent's drag doesn't fire when you grab a child.

```ts
// Movable cabinet with drawers: the cabinet slides on the floor, each drawer opens on its own
const cab = cabinet.addComponent(DragControls);
cab.dragMode = DragMode.XZPlane;
cab.allowScaling = false;
for (const drawerObj of drawers) {             // children of `cabinet`, origin on the front face
  const d = drawerObj.addComponent(DragControls);
  d.dragMode = DragMode.Slide;
  d.axis = DragAxis.Z;                          // parent (cabinet) space, so it follows the cabinet's yaw
  d.axisMin = 0; d.axisMax = 0.38;
}
```

### Rug that stretches from each edge (one side only)

One `DragControls` on the rug does it: the middle moves it, an edge stretches it along that axis (the opposite edge stays put), a corner resizes both axes.

```ts
const d = rugObj.addComponent(DragControls);
d.dragMode = DragMode.XZPlane;
d.rotation = DragRotationMode.YawOnly;
d.rim.edge = EdgeGrabAction.Stretch;
d.rim.corner = CornerGrabAction.ScaleFree;   // or ScaleProportional; None = corners stretch too
d.scaling.mask.set(1, 0, 1);
d.scaling.min = 0.3; d.scaling.max = 3;            // multiples of the authored size
d.limitArea = floorArea;                     // optional
```

For visible grips instead of an invisible rim, use four edge handles, each with its own `DragControls` that drives the same rug through `customTransform`. `Scale` resizes about `customPivot`, so a pivot on the **opposite** edge keeps that edge still while the grabbed one follows the pointer. With `scaling.mask` the handle only stretches its own axis, not the whole rug.

```ts
// rug: root of the carpet at unit scale; its geometry spans -0.5..0.5 on X and Z
const edges = [
  { axis: "x", side:  1 }, { axis: "x", side: -1 },
  { axis: "z", side:  1 }, { axis: "z", side: -1 },
] as const;
for (const { axis, side } of edges) {
  const handle = makeHandleMesh(axis);          // thin bar along the edge, child of `rug`
  handle.position[axis] = 0.5 * side;
  rug.add(handle);

  const pivot = new Object3D();                 // empty on the opposite edge
  pivot.position[axis] = -0.5 * side;
  rug.add(pivot);

  const d = handle.addComponent(DragControls);
  d.customTransform = rug;                      // the handle is the grip, the rug is what resizes
  d.dragMode = DragMode.Scale;
  d.customPivot = pivot;                        // read once per drag, position only
  d.scaling.mask.set(axis === "x" ? 1 : 0, 0, axis === "z" ? 1 : 0);
  d.scaling.min = 0.3; d.scaling.max = 3;             // multiples of the authored size
}
// Moving the whole rug: one more DragControls on `rug` itself (XZPlane). The handles win the press on the edges.
```
Handles parented to the rug inherit its scale, so a bar along the X edge gets wider when the rug is stretched along X. If that matters, keep the bars thin across their own axis, or put the handles in a sibling group that you re-place after each drag (`dragEnded`). See [patterns: Resizing](drag-controls-patterns.md#resizing) for the sibling-group setup.

### Turning furniture with the mouse

A single pointer moving furniture never rotates it. Let the rim of the object turn it instead, and the middle keeps moving it:

```ts
const chair = chairObj.addComponent(DragControls);
chair.dragMode = DragMode.XZPlane;
chair.allowScaling = false;
chair.rotation = DragRotationMode.YawOnly;     // edge turn is off while rotation is Locked (code default)
chair.rim.edge = EdgeGrabAction.Rotate;
chair.rim.margin = 0.25;                    // fraction of the shortest half-extent
```
Two-finger twist on touch and the XR wrist also turn it once `rotation` isn't `Locked`. For a fixed-step turn from a key or button, rotate about the up axis (`obj.rotateY(step)`) rather than adding to `rotation.y` (see [Gotchas](#gotchas)).

## Reading the value of a control

| Property | Meaning |
|----------|---------|
| `currentValue` | Slide: parent units from rest. Hinge/ScaleAndRotate: degrees from rest. Scale: factor. Rotate: first enabled axis. Moving modes: `0`. |
| `normalizedValue` | `0…1` across the authored range (`axisMin`→`axisMax`, `startAngle`→end of `rangeAngle`, Scale: `scaling.min`→`scaling.max`). |
| `currentAngles` / `normalizedAngles` | `Rotate`: per-axis degrees / `0…1` as a `Vector3`. Live buffer — copy it. |
| `currentScale` / `normalizedScale` | `ScaleAndRotate`: scale factor / `0…1` between `scaling.min` and `scaling.max`. |

Events fire only on change:
```ts
door.valueChanged.addEventListener(deg => console.log(deg));
drawer.valueChangedNormalized.addEventListener(t => light.intensity = t * 5);
```

`normalizedRangeAngle` moves where a hinge reports `1` (e.g. a 360° free dial that reads `1` at 340°, or a 90° door that counts as open at 80°) without changing the swing.

## Snapping and engage thresholds

- **Grid (moving modes):** `snapping.gridResolution` (0 = off), `snapping.gridSpace` = world grid or parent space.
- **Slide:** `snapping.axisSnapMode` picks which field is read: `snapping.axisSteps` (count of even stops) or `snapping.axisSnap` (spacing / end capture distance).
- **Hinge:** `snapping.angleSnapMode` + `snapping.angleSteps` / `snapping.angleSnap`. `ScaleAndRotate`: `snapping.angleSnap` spacing only. `Rotate`: per-axis `rotateX.angleSnap` etc.
- `SnapMode`: `None = 0`, `Steps = 1` (reads `*Steps`), `Incremental = 2` (default; reads `*Snap` as spacing), `Ends = 3` (free travel; let go within `*Snap` of an end and it pulls shut/open).

Engage thresholds are dead zones before the object starts following:
- `engage.planeThreshold` (moving modes, world units), `engage.axisThreshold` (Slide), `engage.angleThreshold` (hinge, ScaleAndRotate, edge/corner turn), `engage.scaleThreshold` (Scale, ScaleAndRotate, edge stretch, corner resize), `rotateX.engageThreshold` (per axis — keeps a joystick pushed forward from drifting sideways).
- `engage.mode` (Slide/Hinge only): `FromStart = 0`, `FromEnd = 1`, `Always = 2` (default; with snapping also holds at each stop). A drawer that sticks shut but slides freely once open: `engage.axisThreshold = 0.05`, `engage.mode = 0`.
- `engage.elastic` + `engage.stiffness` (0…1): stretch toward the pointer inside the dead zone instead of standing still; releasing before the threshold returns the object. Stiffness `1` = same as off.

> `SnapMode`, `EngageMode` and `SnapGridSpace` are **not exported** from the `@needle-tools/engine` root at the time of writing — assign the numeric values shown above (or set them in Unity).

## Return on release

```ts
lever.returning.mode = DragReturn.Rest;   // None (default) | Rest (authored pose) | DragStart (undo this drag)
lever.returning.delay = 0.5;                   // seconds held before returning; regrab cancels
lever.returning.duration = 0.25;               // 0 snaps
lever.returning.curve = myAnimationCurve;      // progress→progress; >1 overshoots (springy)
```
Works in every mode. Value events follow the object home. An accepting drop target wins over the return. From code: `drag.resetToRest(obj?, animate = true)` and `drag.getRestPose(obj?)` (parent-space pose; the zero every range is measured from).

## Grab point, anchor, pivot, snap points

Four separate ideas — don't mix them up:

| Field | Answers |
|-------|---------|
| `grab.mode` (`GrabPoint.Hit` default / `Pivot` / `Custom`) + `grab.points` | What the **pointer** holds. `Custom` = only the scaled unit cube of each `grab.points` transform can be grabbed (a cup by its handle). `grab.alignToPointer` hangs the object by the handle's frame. |
| `surface.anchor` (`SurfaceAnchor.BoundsBottom` default / `Pivot` / `Custom`) + `surface.customAnchor` | What touches the **surface** in `SnapToSurfaces` (height only). |
| `snapping.points` | What mates with a `DragTarget` **slot** (connectors, plugs). Empty = object origin. |
| `customPivot` | What `HingeRotate`/`Rotate`/`Scale`/`ScaleAndRotate` turn and scale **about**. |

In `Slide`/`HingeRotate`, `grab.points` only restrict where the object can be taken hold of; the object is not re-hung on them.

## Rotating and scaling while moving

- `rotation: DragRotationMode` — `Locked = 0`, `YawOnly = 1` (upright, turnable — usually what you want), `Free = 2`. Applies to screen and XR. Single-pointer screen drags never rotate; rotation comes from two-finger twist, XR wrist, edge grabs, or drop-target previews.
- `allowScaling` — two-pointer pinch / two-hand scaling, bounded by `scaling.min`/`scaling.max`.
- `surface.alignToNormal` (`SnapToSurfaces`) is independent of `rotation`.
- `dragOffset` — visual stand-off along the plane/surface normal while dragging (lifts the object slightly); eases away on release.

## Edge and corner grabs

Moving modes only: the middle always moves the object; the sides and corners do what you pick.

| Setting | Options |
|---------|---------|
| `rim.edge: EdgeGrabAction` | `None` (default) · `Rotate` (turn in place about the mode's up axis or `customPivot`) · `Stretch` (resize along that side's axis; opposite side fixed, grabbed side follows the pointer) |
| `rim.corner: CornerGrabAction` | `None` (default; corners do what the edges do) · `ScaleFree` (both face axes independently) · `ScaleProportional` (uniform; grows up from the floor) · `Rotate` |

```ts
drag.rotation = DragRotationMode.YawOnly;   // Rotate is off while rotation is Locked
drag.rim.edge = EdgeGrabAction.Rotate;
drag.rim.corner = CornerGrabAction.ScaleProportional;
drag.rim.margin = 0.25;                 // fraction of the shortest half-extent; 0 disables
drag.rim.mode = EdgeGrabMode.SideView;  // TopDown (default) | SideView | FaceGrid
drag.rim.boundsRaycast = true;          // hit the bounding box, for non-box shapes
drag.rim.inXR = false;                  // true: XR hands/controllers use the rim too
drag.rim.inTouch = false;               // true: touch (and handheld AR) uses the rim too
```
Every resize keeps the opposite side or corner where it is, reads `scaling.mask` (`0` locks an axis), `scaling.min`/`scaling.max` and `engage.scaleThreshold`, and respects `checkForCollider` and (plane modes) `limitArea` — the grown side stops at the area's edge. It does not need `allowScaling`. Turns use `engage.angleThreshold`. Ask before a drag what a spot does — for cursors and highlights:
```ts
const g = drag.getGestureAt(hit.point);   // GrabGesture.Move | Turn | Resize | Stretch | null
```
`getGrabZoneAt(point)` returns `GrabZone.Body/Edge/Corner`; `getGrabRim(out)` gives the rim geometry for drawing. Call `invalidateBounds()` after swapping geometry.

## Collisions and physics

- `checkForCollider: DragCollisionCheck` (every mode; needs colliders on obstacles):
  `None = 0` · `DestinationOnly = 1` (overlap allowed while dragging, resolved on release) · `PathAndDestination = 2` (path and pose clear, slides along obstacles) · `DestinationOnlyStrict = 3` (every pose clear, path not checked) · `PathAndDestinationStrict = 4` (no reacquiring past obstacles). `drag.collisionBlocked` reports a live block.
- `ignorePhysicsWhileDragging` — disables the object's colliders and makes its bodies kinematic for the drag, so carrying a crate through a stack doesn't scatter it.
- `attachedDepthRaycast` — `Attached` + screen pointer: pull the object toward the camera instead of into geometry.
- `applyAngularVelocityOnRelease` (default `true`) — a `Rotate` flick spins a `Rigidbody` on release. Freeze the body's position and the rotation axes the drag leaves disabled, or physics drifts the pose.
- `HingeJoint` honours `useLimits` / `limits` (degrees from the authored pose). In Unity, "Add Hinge Physics Component" on a `HingeRotate` DragControls sets up the body, a kinematic frame and the joint limits from the hinge range (unlimited beyond ±180°, where the drag keeps the range itself).

## XR

- `xrHold: XRHoldBehaviour` — what happens to the mode while a controller/hand holds a moving-mode object: `AlwaysSnap = 0` (constraints stay, furniture stays on the floor), `FreeThenSnap = 1` (loose in the hand, settles back into the mode on release), `Free = 2` (stays where let go).
- `xrDistanceDragFactor` — push/pull multiplier for far grabs.
- `screenProfile` / `xrProfile` expose the resolved settings per input type (`dragMode`, `hold`, `rotation`, `allowScaling`).

## Events and live state

```ts
drag.dragStarted.addEventListener(e => {});   // DragStartedEventArgs
drag.dragUpdated.addEventListener(e => {      // every frame
  e.dropTarget;     // IDropTargetHandle | null — what would catch it now
  e.dropAllowed;    // advisory allowDrop answer
});
drag.dragEnded.addEventListener(e => {
  e.dropped; e.dropTarget; e.allowed; e.returning;
});
// Veto a release (every mode). Returns the object to where this drag picked it up.
drag.allowDrop.addEventListener(e => {
  if (!e.isRelease) return;                   // per-frame advisory call vs. binding release call
  if (e.dropTarget?.gameObject !== board) e.disallow();
});
```
All args carry `target`, `object`, `mode` (the mode actually running — an XR hold can differ from `dragMode`), `pointerCount`, `isXR`, `startPosition`/`startQuaternion`/`startScale`, `value`, `normalizedValue`. Vectors are the component's **own buffers** — copy what you keep. Use `e.object` in `dragEnded`; `draggedObject` is already `null` there.

`drag.pointerState` (`null` when not dragging) describes the **pointer** for overlays: `point`, `plane`, `origin`, `axis`, `grabPoint`, `aimPosition`, `gesture`, `ray`. `getModeOrigin(out)` / `getModeAxis(out)` give origin/axis without a drag (to draw a control's range at rest).

Statics: `DragControls.CurrentlySelected`, `DragControls.HasAnySelected`.

## Custom constraints and modes

`drag.constraints` runs after the mode's own constraints every frame (single and two-pointer). Built-ins include `BoundsConstraint`, `LimitAreaConstraint`, `AxisLineConstraint`, `FixedPositionConstraint`, `GridSnapConstraint`, `KeepScaleConstraint`, `ScaleLimitConstraint`, `CollisionBlockConstraint`:
```ts
import { BoundsConstraint } from "@needle-tools/engine";
const bounds = new BoundsConstraint();
bounds.min.set(-5, 0, -5); bounds.max.set(5, 0, 5);
bounds.referenceSpace = room;   // null = world
bounds.clampBounds = true;      // whole box, not just the pivot
drag.constraints.push(bounds);
```
A custom one is any object with `apply(followObject)` (modify the follow object in place) and optional `init(ctx)`. A whole new mode: `registerDragStrategy(100 as DragMode, () => new MyStrategy())`. A custom drop system: implement `IDropTargetSystem` and `registerDropTargetSystem(...)`.

---

## DragTarget

Receiver for objects dragged in a **moving** mode. Add `DragControls` to each movable object and `DragTarget` to the receiver.

`mode: DragTargetMode`:
| Mode | Slots |
|------|-------|
| `Single` (default) | One slot at this transform. |
| `Slots` | One per transform in `slots` (empty list = accepts nothing). |
| `Grid` | `grid.columns` × `grid.rows` with `grid.spacing`, local XZ, centred. |
| `Area` | A region (this object's unit cube, or `area.region`'s); objects keep their drop position. `area.capacity` limits it (0 = unlimited). |
| `Stacked` | One pile per `slots` transform along its up axis (or `stack.direction`); `stack.count` < 2 = unlimited. |

```ts
import { DragControls, DragMode, DragTarget, DragTargetMode, DropTargetMode } from "@needle-tools/engine";

const piece = pieceObj.addComponent(DragControls);
piece.dragMode = DragMode.XZPlane;
piece.dragTargets = DropTargetMode.Require;   // Allow (default) | Require (return if missed) | Disallow

const board = boardObj.addComponent(DragTarget);
board.mode = DragTargetMode.Grid;
board.grid.columns = 8; board.grid.rows = 8; board.grid.spacing = 1;
board.objectDropped.addEventListener(e => console.log("square", e.slot));
```

Settings live in groups, like `DragControls`: `grid`, `area`, `stack`, `snapping`, `swap`, `holding`, `fly`, `engage` and `filter`. `mode`, `slots` and the events stay on the component.

Key settings:
- **`snapping`, offering:** `distance` (world units, default 0.2), `whenPointedAt` (offer what the pointer aims at, at any distance; empty slot transforms and grid cells get a hit disc of radius `distance`, an Area its region box), `onRelease` (don't pull in live; snap on release), `checkForCollider` (boolean — landing space must be empty).
- **`snapping`, orientation:** `rotation` (default `true`), `rotationSteps` + `rotationAxis` (symmetric objects fitting several ways), `autoRotate` / `autoRotateInterval` (cycle orientations while hovering).
- **Full slots:** `stack.count`, `swap.onFullSlot` (default `true` — dropping on an occupied slot swaps), `swap.hoverAxis` / `swap.hoverDistance` (preview stand-off). While a drag hovers a slot that would swap, `DragHandleVisualizer` shows a swap icon over the occupant (`showSwap`, `swapIcon`).
- **`holding`:** `parentOnDrop` (+ `adoptChildren` claims pre-placed children at start), `occupants` (default `true`, kinematic while in a slot), `areaOccupants` (Area, default `false`), `dropScale` (uniform world scale while held).
- **`area`:** `region`, `fit` (`DragTargetAreaFit.None` / `AutoFitKeepRotation` / `AutoFit`), `dropPoint` (`DragTargetAreaDropPoint.None` / `Surface` / `Interior` / `ThroughTop` / `OnTop`), `capacity`.
- **`fly` and `engage`, in/out:** `fly.height` (distance, `0` = no flight), `fly.axis` (`DragTargetAxis`, signed: `PosX`/`NegX`/`PosY`/`NegY`/`PosZ`/`NegZ`, default `PosY`; the slot-local side objects enter, leave and are pulled out by), `fly.duration`; `engage.threshold` (+ `engage.elastic`, `engage.stiffness`) — pull-out distance before an object comes free (a sword from a stone).
- **`filter`:** `objects` (descendants count), `mask` (layers), `tags` (any of), `blockUnallowed`. All filters combine with AND.

**DragTarget combinations:**
- `Area` ignores `snapping.distance`, `snapping.rotation*`, `snapping.autoRotate*`, `stack`, `swap`, `holding.occupants`, `fly` and `engage`; it uses `area.capacity`, `area.fit`, `area.dropPoint`, `holding.areaOccupants` instead. `snapping.onRelease` in Area only matters with `area.fit` ≠ `None` (fit on release instead of live).
- Slot modes ignore `area` and `holding.areaOccupants`.
- `area.dropPoint` ≠ `None` also enables pointed drops without `snapping.whenPointedAt`; changes apply when the target is next enabled.
- `snapping.rotationSteps` / `snapping.rotationAxis` need `snapping.rotation`. `snapping.autoRotate` needs `snapping.rotation`, `snapping.rotationSteps` ≥ 2, and `snapping.onRelease = false`.
- `swap.onFullSlot` only works at `stack.count = 1`; above that a full stack refuses. `swap.hoverAxis`/`swap.hoverDistance` need `swap.onFullSlot` (zero axis = no stand-off).
- `holding.adoptChildren` needs `holding.parentOnDrop`; children with in-place modes (drawers, lids) are skipped.
- `fly.duration` needs `fly.height` > 0; `engage.elastic`/`engage.stiffness` need `engage.threshold` > 0. `fly.axis` serves both and sets the engage side with `fly.height` at `0`.
- `filter.blockUnallowed = false` lets refused objects be aimed through this target (a carpet spot on the floor that must not block furniture).
- Drag Targets only receive objects whose DragControls uses a moving mode and `dragTargets` ≠ `Disallow`.

Events (`EventList<…>`): `targetEntered`, `targetExited`, `objectDropped`, `objectRemoved`, `allowedObjectDragStarted` / `allowedObjectDragEnded` (light up valid targets as soon as a piece is picked up), `allowTarget` (veto once per drag: `e.disallow()`), `beforeDrop` (final veto with the resolved `e.position`/`e.quaternion`/`e.snapPoint`).

Queries: `occupants`, `getOccupant(slot)`, `getOccupants(slot)`, `getSlotLoad(slot)`, `slotCount`, `slotCapacity`, `currentSlot`, `draggedObject`, `getSlotWorldPosition(slot)`, `getSlotWorldQuaternion(slot)`, `accepts(obj)`, `releaseObject(obj)`. All enabled targets: `DragTarget.instances`.

Three different vetoes: `DragControls.allowDrop` asks "may this release happen here" (every mode, every frame + release); `DragTarget.allowTarget` asks "will this target take this object at all" (once per drag); `DragTarget.beforeDrop` checks the exact landing pose. All must decide synchronously.

## Deprecated fields

Still honoured when the replacement is unset, so old scenes keep working. Use the replacements in new code.

| Deprecated | Use |
|------------|-----|
| `keepRotation`, `xrKeepRotation` | `rotation: DragRotationMode` |
| `keepScale`, `xrKeepScale` | `allowScaling` (inverted) |
| `xrDragMode` | `xrHold: XRHoldBehaviour` |
| `snapGridResolution` | `snapping.gridResolution` |

## Gotchas

- Code-created DragControls default to **rotation locked on screen and pinch scaling on** — see [Defaults](#defaults-unity-vs-code).
- `rangeAngle` ≥ 360 means **no limit**, not a 360° range. Negative ranges sweep the other way.
- Slide/Hinge/Rotate/Scale/ScaleAndRotate **never** enter drop targets — `DragTarget` only works with moving modes.
- `customPivot` is read once at drag start, position only, in the parent's space.
- `valueChanged` only reports in-place modes; moving modes report through `dragUpdated`.
- Event args and `pointerState` are live buffers reused every frame — copy before storing.
- `surface.normalFilter` and `scaling.relativeToStart` are code-only (not in the Unity inspector).
- Don't build your own drag, rotate or resize handles out of pointer events and a `Raycaster` before trying the modes and recipes above. Handles that drive another object (`customTransform`), pivots (`customPivot`), per-axis resize (`scaling.mask`) and child controls inside a draggable parent cover most furniture and editor-style interactions.
- Read yaw from the quaternion, not `rotation.y`. Once an object's quaternion has been written (by a drag, for example), its Euler angles can come back as `(π, 0, π)` instead of `(0, π, 0)`, and adding to `rotation.y` then turns it the wrong way. Use `obj.rotateY(delta)` or premultiply a Y-axis quaternion.
