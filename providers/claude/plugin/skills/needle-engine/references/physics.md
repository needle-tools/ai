# Needle Engine — Physics Reference

Needle Engine uses Rapier (WASM) for physics. Rapier is a lazy-loaded async module — it is NOT automatically imported when you import `Rigidbody`, `BoxCollider`, or other physics component classes.

**In code-only projects (no GLB with colliders), you MUST explicitly load Rapier before creating any physics objects:**
```ts
import { NEEDLE_ENGINE_MODULES } from "@needle-tools/engine";

onStart(async ctx => {
    // Load Rapier WASM — without this, physics silently does nothing in code-only projects
    await NEEDLE_ENGINE_MODULES.RAPIER_PHYSICS.load();

    // NOW create rigidbodies, colliders, apply forces, etc.
    const rb = myObject.addComponent(Rigidbody);
    myObject.addComponent(SphereCollider);
    rb.applyImpulse(new Vector3(0, 5, 0));
});
```

In GLB-based projects (Unity/Blender export), Rapier loads automatically when the deserializer creates collider components. But in code-only projects, nothing triggers the load — `addComponent(Rigidbody)` and `addComponent(SphereCollider)` just silently fail to create physics bodies, `applyImpulse`/`applyForce` do nothing, and there are no errors. This is the #1 cause of "physics don't work on deploy" bugs.

## Colliders

Pick the shape that best fits the object — don't default to BoxCollider for everything.

```ts
import { BoxCollider, SphereCollider, CapsuleCollider, MeshCollider } from "@needle-tools/engine";

// Quick setup — auto-fits to mesh bounds, optionally adds rigidbody:
BoxCollider.add(myMesh, { rigidbody: true });
SphereCollider.add(myMesh, { rigidbody: true });

// Or add manually and configure:
const box = myObject.addComponent(BoxCollider);
// box.size, box.center

const sphere = myObject.addComponent(SphereCollider);
// sphere.radius (default: 0.5), sphere.center

const capsule = myObject.addComponent(CapsuleCollider);
// capsule.radius (default: 0.5), capsule.height (default: 2) — use for characters, poles, bottles

const mesh = myObject.addComponent(MeshCollider);
// mesh.convex = true for dynamic objects (required with Rigidbody)
// mesh.convex = false for static concave geometry (walls, terrain)
```

Use `SphereCollider` for balls, `CapsuleCollider` for characters/cylinders, `MeshCollider` for complex static geometry. Set `isTrigger = true` for trigger volumes.

## Rigidbody
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

## Physics callbacks
Defined on components (require a Collider on the same GameObject):
```ts
onCollisionEnter(col: Collision) { /* hit something */ }
onCollisionStay(col: Collision)  { /* still touching */ }
onCollisionExit(col: Collision)  { /* separated */ }
onTriggerEnter(col: Collision)   { /* entered trigger */ }
onTriggerStay(col: Collision)
onTriggerExit(col: Collision)
```

## Raycasting
```ts
// Visual raycast (hits any visible geometry, no collider needed, BVH-accelerated)
const hits = this.context.physics.raycast();

// Physics engine raycast (hits Rapier colliders only)
const hit = this.context.physics.engine?.raycast(origin, direction);
```
