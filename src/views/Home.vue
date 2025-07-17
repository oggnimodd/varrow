<script setup lang="ts">
import { ref } from "vue";
import { useDraggable } from "@vueuse/core";
import { VArrow } from "varrow";

// --- Create a ref and a draggable instance for EVERY box ---

const box1 = ref<HTMLElement | null>(null);
const { style: box1Style } = useDraggable(box1, {
  initialValue: { x: 50, y: 150 },
});

const box2 = ref<HTMLElement | null>(null);
const { style: box2Style } = useDraggable(box2, {
  initialValue: { x: 300, y: 250 },
});

const box3 = ref<HTMLElement | null>(null);
const { style: box3Style } = useDraggable(box3, {
  initialValue: { x: 500, y: 150 },
});

const box4 = ref<HTMLElement | null>(null);
const { style: box4Style } = useDraggable(box4, {
  initialValue: { x: 750, y: 250 },
});

const box5 = ref<HTMLElement | null>(null);
const { style: box5Style } = useDraggable(box5, {
  initialValue: { x: 50, y: 500 },
});

const box6 = ref<HTMLElement | null>(null);
const { style: box6Style } = useDraggable(box6, {
  initialValue: { x: 300, y: 600 },
});

const box7 = ref<HTMLElement | null>(null);
const { style: box7Style } = useDraggable(box7, {
  initialValue: { x: 500, y: 500 },
});

const box8 = ref<HTMLElement | null>(null);
const { style: box8Style } = useDraggable(box8, {
  initialValue: { x: 750, y: 600 },
});
</script>

<template>
  <div class="showcase-container">
    <header class="showcase-header">
      <h1>VArrow Library Showcase</h1>
      <p>All boxes are draggable. Move them around to see the arrows react!</p>
    </header>

    <!-- The Draggable Boxes -->
    <div ref="box1" class="box" :style="box1Style">Box 1</div>
    <div ref="box2" class="box" :style="box2Style">Box 2</div>
    <div ref="box3" class="box" :style="box3Style">Box 3</div>
    <div ref="box4" class="box" :style="box4Style">Box 4</div>
    <div ref="box5" class="box" :style="box5Style">Box 5</div>
    <div ref="box6" class="box" :style="box6Style">Box 6</div>
    <div ref="box7" class="box" :style="box7Style">Box 7</div>
    <div ref="box8" class="box" :style="box8Style">Box 8</div>

    <!-- The VArrow Components -->
    <template
      v-if="box1 && box2 && box3 && box4 && box5 && box6 && box7 && box8"
    >
      <!-- Example 1: Straight vs. Curved Path -->
      <VArrow :start="box1" :end="box2" />
      <VArrow :start="box2" :end="box1" path="curve" color="#3b82f6" />

      <!-- Example 2: Styled & Dashed/Dotted Lines -->
      <VArrow
        :start="box3"
        :end="box4"
        color="crimson"
        :stroke-width="4"
        stroke-style="dashed"
      />
      <VArrow
        :start="box4"
        :end="box3"
        color="rebeccapurple"
        stroke-style="dotted"
        start-marker="circle"
        end-marker="circle"
        :marker-size="8"
      />

      <!-- Example 3: Explicit Anchors & Offsets -->
      <VArrow
        :start="box5"
        :end="box6"
        color="darkorange"
        path="curve"
        :stroke-width="5"
        start-anchor="bottom"
        end-anchor="top"
        :start-offset="{ x: -20, y: 10 }"
        :end-offset="{ x: 20, y: -10 }"
      />

      <VArrow
        :start="box4"
        :end="box8"
        color="darkorange"
        path="curve"
        :stroke-width="5"
        start-anchor="bottom"
        end-anchor="top"
        :start-offset="{ x: -20, y: 10 }"
        :end-offset="{ x: 20, y: -10 }"
      />

      <VArrow
        :start="box5"
        :end="box8"
        color="darkorange"
        path="curve"
        :stroke-width="5"
        start-anchor="bottom"
        end-anchor="top"
        :start-offset="{ x: -20, y: 10 }"
        :end-offset="{ x: 20, y: -10 }"
      />

      <!-- Example 4: Mixed Targets (Element to Coordinate) -->
      <VArrow
        :start="box7"
        :end="{ x: 600, y: 750 }"
        color="#007acc"
        start-marker="circle"
        :end-marker="null"
      />
      <VArrow
        :start="{ x: 850, y: 450 }"
        :end="box8"
        color="green"
        :stroke-width="3"
      />
    </template>

    <!-- On-screen labels for clarity -->
    <div class="pod-label" style="top: 100px; left: 150px">
      <h3>1. Basic Paths</h3>
      <p>Straight vs. Curved</p>
    </div>
    <div class="pod-label" style="top: 100px; left: 600px">
      <h3>2. Styling</h3>
      <p>Dashed, Dotted & Markers</p>
    </div>
    <div class="pod-label" style="top: 450px; left: 150px">
      <h3>3. Anchors & Offsets</h3>
      <p>Fine-tuned connections</p>
    </div>
    <div class="pod-label" style="top: 450px; left: 600px">
      <h3>4. Mixed Targets</h3>
      <p>Element to/from Coordinate</p>
    </div>
  </div>
</template>

<style>
:root {
  --border-color: #e2e8f0;
  --bg-color: #f8fafc;
  --text-color: #0f172a;
  --box-bg: white;
  --box-border: #94a3b8;
  --box-shadow: rgba(0, 0, 0, 0.05);
}

body {
  font-family:
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    Roboto,
    Oxygen,
    Ubuntu,
    Cantarell,
    "Open Sans",
    "Helvetica Neue",
    sans-serif;
  background-color: var(--bg-color);
  color: var(--text-color);
  margin: 0;
}

.showcase-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.showcase-header {
  padding: 1.5rem 2rem;
  border-bottom: 1px solid var(--border-color);
  position: relative;
  z-index: 20;
  background-color: var(--bg-color);
}

.showcase-header h1 {
  margin: 0 0 0.25rem 0;
}

.showcase-header p {
  margin: 0;
  color: #64748b;
}

.box {
  position: fixed; /* useDraggable works best with fixed positioning */
  width: 180px;
  height: 90px;
  border: 2px solid var(--box-border);
  background-color: var(--box-bg);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  box-shadow:
    0 4px 6px -1px var(--box-shadow),
    0 2px 4px -2px var(--box-shadow);
  transition:
    box-shadow 0.2s ease,
    transform 0.2s ease;
  cursor: move;
  z-index: 10;
}

.box:hover {
  transform: scale(1.05);
  box-shadow:
    0 10px 15px -3px var(--box-shadow),
    0 4px 6px -4px var(--box-shadow);
}

.pod-label {
  position: absolute;
  pointer-events: none;
  z-index: 5;
  text-align: center;
}

.pod-label h3 {
  margin: 0 0 0.25rem 0;
  color: #334155;
}

.pod-label p {
  margin: 0;
  font-size: 14px;
  color: #64748b;
}
</style>
