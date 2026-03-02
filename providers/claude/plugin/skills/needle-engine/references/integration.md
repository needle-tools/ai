# Needle Engine — Framework Integration

## React

### Listen for 3D events in a component
```tsx
import { useEffect, useState } from "react";

function ScoreDisplay() {
  const [score, setScore] = useState(0);

  useEffect(() => {
    const ne = document.querySelector("needle-engine");
    const handler = (e: CustomEvent) => setScore(e.detail.score);
    ne?.addEventListener("score-changed", handler as EventListener);
    return () => ne?.removeEventListener("score-changed", handler as EventListener);
  }, []);

  return <div>Score: {score}</div>;
}
```

### Call into the 3D scene from React
```tsx
function GameControls() {
  const addScore = () => {
    const ctx = (document.querySelector("needle-engine") as any)?.context;
    ctx?.scene.getComponentInChildren(GameManager)?.addScore(10);
  };
  return <button onClick={addScore}>+10</button>;
}
```

### Wait for scene ready before accessing components
```tsx
useEffect(() => {
  const ne = document.querySelector("needle-engine");
  const onReady = () => {
    const ctx = (ne as any).context;
    // safe to access components here
  };
  ne?.addEventListener("loadingfinished", onReady);
  return () => ne?.removeEventListener("loadingfinished", onReady);
}, []);
```

---

## Svelte / SvelteKit

```svelte
<script>
  import { onMount } from "svelte";
  let score = 0;

  onMount(() => {
    const ne = document.querySelector("needle-engine");
    const handler = (e) => (score = e.detail.score);
    ne?.addEventListener("score-changed", handler);
    return () => ne?.removeEventListener("score-changed", handler);
  });

  function addScore() {
    const ctx = document.querySelector("needle-engine")?.context;
    ctx?.scene.getComponentInChildren(GameManager)?.addScore(10);
  }
</script>

<p>Score: {score}</p>
<button on:click={addScore}>+10</button>
<needle-engine src="assets/scene.glb" />
```

---

## Vue

```vue
<template>
  <div>Score: {{ score }}</div>
  <needle-engine src="assets/scene.glb" ref="ne" />
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";

const score = ref(0);
const ne = ref(null);

function onScore(e) { score.value = e.detail.score; }

onMounted(() => ne.value?.addEventListener("score-changed", onScore));
onUnmounted(() => ne.value?.removeEventListener("score-changed", onScore));
</script>
```

---

## Vanilla JS / No Framework

```html
<needle-engine src="assets/scene.glb" id="scene"></needle-engine>

<script type="module">
  import "@needle-tools/engine";
  import { onStart, onUpdate } from "@needle-tools/engine";

  const ne = document.getElementById("scene");

  // Wait for scene to finish loading
  ne.addEventListener("loadingfinished", () => {
    const ctx = ne.context;
    console.log("Scene ready:", ctx.scene);
  });

  // Standalone hooks (no component class needed)
  onStart((ctx) => {
    console.log("Engine started");
  });
</script>
```

---

## Engine Hooks Reference

These standalone functions from `@needle-tools/engine` mirror the component lifecycle but work outside of a class:

| Hook | When it fires |
|---|---|
| `onStart(cb)` | Once when the context/scene is ready |
| `onUpdate(cb)` | Every frame (before rendering) |
| `onBeforeRender(cb)` | Just before Three.js renders |
| `onDestroy(cb)` | When the context is torn down |

All callbacks receive `(ctx: Context)` as their argument.

```ts
import { onStart, onUpdate, onBeforeRender, onDestroy } from "@needle-tools/engine";

onStart((ctx) => { /* setup */ });
onUpdate((ctx) => { /* per-frame logic */ });
onDestroy((ctx) => { /* cleanup */ });
```
