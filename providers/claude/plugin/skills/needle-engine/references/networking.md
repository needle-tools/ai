# Needle Engine — Networking Reference

## Table of Contents
- [Core: context.connection](#core-thiscontextconnection)
- [Persistent vs ephemeral messages](#persistent-vs-ephemeral-messages-guid)
- [SyncedRoom](#syncedroom-convenience-component)
- [@syncField](#syncfield-auto-sync-fields)
- [SyncedTransform](#syncedtransform-sync-positionrotation)
- [PlayerSync + PlayerState](#playersync--playerstate-player-avatar-management)
- [Voip and ScreenCapture](#voice--video-voip-and-screencapture)

---

Needle Engine networking is layered. The lowest level is `this.context.connection` (WebSocket rooms + messages). Higher-level components build on it.

## Core: `this.context.connection`
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
this.context.connection.beginListen(RoomEvents.UserJoinedRoom, (evt) => {
  console.log("User joined:", evt.userId);   // evt: { userId: string }
});
this.context.connection.beginListen(RoomEvents.UserLeftRoom, (evt) => {
  console.log("User left:", evt.userId);     // evt: { userId: string }
});
this.context.connection.beginListen(RoomEvents.RoomStateSent, () => { ... }); // all persisted state received

// Always check connection state before sending:
if (this.context.connection.isInRoom) {
  this.context.connection.send("my-event", { data: 42 });
}
```

**Important:** `send()` broadcasts to all users in the room **except yourself** — you won't receive your own messages. Custom messages also do NOT automatically include a sender ID. If you need to know who sent a message, include `connectionId` yourself:
```ts
// Sending — include your own ID
this.context.connection.send("player-position", {
  senderId: this.context.connection.connectionId,
  x: pos.x, y: pos.y, z: pos.z,
});

// Receiving — filter out your own messages
this.context.connection.beginListen("player-position", (msg) => {
  if (msg.senderId === this.context.connection.connectionId) return; // ignore self
  // handle remote player position...
});
```

`userId` is only available in room lifecycle events (`UserJoinedRoom`, `UserLeftRoom`), not in custom messages.

---

## Networked Instantiation and Destruction

For spawning objects that should appear on all clients, use `syncInstantiate` / `instantiateSynced` instead of manually sending custom events.

```ts
import { instantiate, syncInstantiate, syncDestroy, registerPrefabProvider } from "@needle-tools/engine";

// Local clone (only on this client)
const clone = instantiate(prefabObject, { parent: this.gameObject });

// Networked spawn (appears on all connected clients)
const networked = syncInstantiate(prefabObject, {
  parent: this.gameObject,
  position: [x, y, z],
  deleteOnDisconnect: true,    // removed when the spawning user disconnects
});

// Persistent networked spawn (survives disconnects, replayed to late joiners)
const persistent = syncInstantiate(prefabObject, {
  parent: this.gameObject,
  deleteOnDisconnect: false,
});

// Via AssetReference
const synced = await myAssetRef.instantiateSynced({
  parent: this.gameObject,
  deleteOnDisconnect: false,
});

// Networked destroy (removed on all clients)
syncDestroy(obj, this.context.connection, true);
```

### Runtime prefabs with `registerPrefabProvider`
When creating prefabs at runtime (not loaded from GLB), register them so `syncInstantiate` can recreate them on other clients:
```ts
import { registerPrefabProvider, ObjectUtils } from "@needle-tools/engine";

const myPrefab = ObjectUtils.createPrimitive("Cube", { color: 0xff8c00 });
myPrefab.guid = "my-runtime-cube";
myPrefab.removeFromParent(); // don't add to scene — it's a template
registerPrefabProvider("my-runtime-cube", async () => myPrefab);

// Now syncInstantiate will work across clients
syncInstantiate(myPrefab, { parent: ctx.scene, deleteOnDisconnect: false });
```

### World-building pattern (first player seeds, late joiners receive)
```ts
let shouldBuildWorld = false;

connection.beginListen(RoomEvents.JoinedRoom, () => {
  const inRoom = connection.usersInRoom();
  shouldBuildWorld = inRoom.length === 1; // I'm the only one here
});

connection.beginListen(RoomEvents.RoomStateSent, () => {
  // State replay complete — only build if we're first AND no objects exist yet
  if (!shouldBuildWorld) return;

  for (let i = 0; i < 100; i++) {
    syncInstantiate(cookiePrefab, {
      parent: ctx.scene,
      position: [x, 0, z],
      deleteOnDisconnect: false, // persists for late joiners
    });
  }
});
```

---

## Persistent vs ephemeral messages (guid)
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

---

## SyncedRoom (convenience component)
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

---

## @syncField (auto-sync fields)
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

---

## SyncedTransform (sync position/rotation)
Syncs an object's position, rotation, and scale across clients. Ownership is automatic — when a user interacts (e.g. via DragControls), they take ownership.
```ts
import { SyncedTransform } from "@needle-tools/engine";
// Add to any object that should be movable by networked users
myObject.addComponent(SyncedTransform);
```

---

## PlayerSync + PlayerState (player avatar management)
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

---

## Voice & Video: Voip and ScreenCapture
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

---

### Syncing Animations
For syncing Animator state across clients, see the [SyncedAnimator sample](https://github.com/needle-tools/needle-engine-samples/blob/main/package/Runtime/Networking/Scripts/Networking~/Animator/SyncedAnimator.ts) — it listens for Animator parameter changes and broadcasts them via `context.connection`.

---

## Typical multiplayer setup
1. Add `SyncedRoom` to an object (or call `context.connection.joinRoom()` manually)
2. For player avatars: add `PlayerSync` with a prefab that has `PlayerState`
3. For synced objects: add `SyncedTransform` to movable objects
4. For custom state: use `@syncField()` on component properties
5. For custom events: use `context.connection.send()` / `beginListen()`
6. For voice chat: add `Voip` — for screen sharing: add `ScreenCapture`
