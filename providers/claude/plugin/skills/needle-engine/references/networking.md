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
this.context.connection.beginListen(RoomEvents.UserJoinedRoom, (evt) => { ... });
this.context.connection.beginListen(RoomEvents.UserLeftRoom, (evt) => { ... });
this.context.connection.beginListen(RoomEvents.RoomStateSent, () => { ... }); // all persisted state received
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
