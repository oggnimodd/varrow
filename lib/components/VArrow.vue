<script setup lang="ts">
import { ref, computed, onUnmounted, onMounted } from "vue";

type Point = { x: number; y: number };
type Rect = { x: number; y: number; width: number; height: number };
export type VArrowTarget = HTMLElement | Point;
type Anchor = "top" | "bottom" | "left" | "right" | "center" | "auto";
type PathStyle = "straight" | "curve";
type StrokeStyle = "solid" | "dashed" | "dotted";

export interface VArrowProps {
  start: VArrowTarget;
  end: VArrowTarget;
  path?: PathStyle;
  color?: string;
  strokeWidth?: number;
  strokeStyle?: StrokeStyle;
  dashArray?: string;
  startMarker?: string | null;
  endMarker?: string | null;
  markerSize?: number;
  startAnchor?: Anchor;
  endAnchor?: Anchor;
  startOffset?: Point;
  endOffset?: Point;
}

const props = withDefaults(defineProps<VArrowProps>(), {
  path: "straight",
  color: "#2c3e50",
  strokeWidth: 2,
  strokeStyle: "solid",
  dashArray: "5 5",
  startMarker: null,
  endMarker: "arrow",
  markerSize: 10,
  startAnchor: "auto",
  endAnchor: "auto",
  startOffset: () => ({ x: 0, y: 0 }),
  endOffset: () => ({ x: 0, y: 0 }),
});

const instanceId = Math.random().toString(36).substring(7);
const startRect = ref<Rect | null>(null);
const endRect = ref<Rect | null>(null);
let animationFrameId: number | null = null;

function isElement(val: unknown): val is HTMLElement {
  return val instanceof HTMLElement;
}

function isPoint(val: unknown): val is Point {
  return val != null && typeof val === "object" && "x" in val && "y" in val;
}

function updatePositions() {
  if (isElement(props.start)) {
    startRect.value = props.start.getBoundingClientRect();
  }
  if (isElement(props.end)) {
    endRect.value = props.end.getBoundingClientRect();
  }
}

function startPositionTracker() {
  function track() {
    updatePositions();
    animationFrameId = requestAnimationFrame(track);
  }
  track();
}

onMounted(() => {
  updatePositions();
  startPositionTracker();
});

onUnmounted(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});

const startPos = computed<Rect | null>(() => {
  if (isElement(props.start)) return startRect.value;
  if (isPoint(props.start)) return { ...props.start, width: 0, height: 0 };
  return null;
});

const endPos = computed<Rect | null>(() => {
  if (isElement(props.end)) return endRect.value;
  if (isPoint(props.end)) return { ...props.end, width: 0, height: 0 };
  return null;
});

const anchorPoints = computed(() => {
  if (!startPos.value || !endPos.value) return null;
  const getAnchorPoint = (rect: Rect, anchor: Anchor): Point => {
    switch (anchor) {
      case "top":
        return { x: rect.x + rect.width / 2, y: rect.y };
      case "bottom":
        return { x: rect.x + rect.width / 2, y: rect.y + rect.height };
      case "left":
        return { x: rect.x, y: rect.y + rect.height / 2 };
      case "right":
        return { x: rect.x + rect.width, y: rect.y + rect.height / 2 };
      default:
        return { x: rect.x + rect.width / 2, y: rect.y + rect.height / 2 };
    }
  };
  const getAutoAnchors = (r1: Rect, r2: Rect): [Anchor, Anchor] => {
    const c1 = getAnchorPoint(r1, "center");
    const c2 = getAnchorPoint(r2, "center");
    const angle = (Math.atan2(c2.y - c1.y, c2.x - c1.x) * 180) / Math.PI + 180;
    const getAnchorFromAngle = (angle: number): Anchor => {
      if (angle > 45 && angle <= 135) return "top";
      if (angle > 135 && angle <= 225) return "right";
      if (angle > 225 && angle <= 315) return "bottom";
      return "left";
    };
    return [getAnchorFromAngle(angle), getAnchorFromAngle((angle + 180) % 360)];
  };
  let startAnchor = props.startAnchor;
  let endAnchor = props.endAnchor;
  if (startAnchor === "auto" || endAnchor === "auto") {
    const [autoStart, autoEnd] = getAutoAnchors(startPos.value, endPos.value);
    if (startAnchor === "auto") startAnchor = autoStart;
    if (endAnchor === "auto") endAnchor = autoEnd;
  }
  const p1 = getAnchorPoint(startPos.value, startAnchor);
  const p2 = getAnchorPoint(endPos.value, endAnchor);
  p1.x += props.startOffset.x;
  p1.y += props.startOffset.y;
  p2.x += props.endOffset.x;
  p2.y += props.endOffset.y;
  return { p1, p2, startAnchor, endAnchor };
});

const svgBox = computed(() => {
  if (!anchorPoints.value) return { x: 0, y: 0, width: 0, height: 0 };
  const { p1, p2 } = anchorPoints.value;
  const pad = props.strokeWidth + props.markerSize * 2;
  const minX = Math.min(p1.x, p2.x);
  const minY = Math.min(p1.y, p2.y);
  const maxX = Math.max(p1.x, p2.x);
  const maxY = Math.max(p1.y, p2.y);
  return {
    x: minX - pad,
    y: minY - pad,
    width: maxX - minX + pad * 2,
    height: maxY - minY + pad * 2,
  };
});

const pathD = computed(() => {
  if (!anchorPoints.value) return "";
  const { p1, p2, startAnchor, endAnchor } = anchorPoints.value;
  if (props.path === "curve") {
    const VEC_UP: Point = { x: 0, y: -1 };
    const VEC_DOWN: Point = { x: 0, y: 1 };
    const VEC_LEFT: Point = { x: -1, y: 0 };
    const VEC_RIGHT: Point = { x: 1, y: 0 };
    const curveFactor = 0.3;
    const dx = p2.x - p1.x;
    const dy = p2.y - p1.y;
    const getControlVec = (anchor: Anchor): Point => {
      if (anchor === "top") return VEC_UP;
      if (anchor === "bottom") return VEC_DOWN;
      if (anchor === "left") return VEC_LEFT;
      if (anchor === "right") return VEC_RIGHT;
      return { x: 0, y: 0 };
    };
    const startVec = getControlVec(startAnchor);
    const endVec = getControlVec(endAnchor);
    const dist = Math.sqrt(dx * dx + dy * dy) * curveFactor;
    const cp1 = { x: p1.x + startVec.x * dist, y: p1.y + startVec.y * dist };
    const cp2 = { x: p2.x + endVec.x * dist, y: p2.y + endVec.y * dist };
    return `M ${p1.x} ${p1.y} C ${cp1.x} ${cp1.y}, ${cp2.x} ${cp2.y}, ${p2.x} ${p2.y}`;
  }
  return `M ${p1.x} ${p1.y} L ${p2.x} ${p2.y}`;
});

const strokeDasharray = computed(() => {
  if (props.strokeStyle === "dashed") return props.dashArray;
  if (props.strokeStyle === "dotted") return `0 ${props.strokeWidth * 2}`;
  return "none";
});
</script>

<template>
  <svg
    v-if="anchorPoints"
    :style="{
      position: 'absolute',
      left: `${svgBox.x}px`,
      top: `${svgBox.y}px`,
      pointerEvents: 'none',
      overflow: 'visible',
    }"
    :width="svgBox.width"
    :height="svgBox.height"
  >
    <defs>
      <marker
        :id="`arrow-marker-${instanceId}`"
        viewBox="0 0 10 10"
        refX="8"
        refY="5"
        :markerWidth="markerSize"
        :markerHeight="markerSize"
        orient="auto-start-reverse"
      >
        <path d="M 0 0 L 10 5 L 0 10 z" :fill="color" />
      </marker>
      <marker
        :id="`circle-marker-${instanceId}`"
        viewBox="0 0 10 10"
        refX="5"
        refY="5"
        :markerWidth="markerSize"
        :markerHeight="markerSize"
        orient="auto-start-reverse"
      >
        <circle cx="5" cy="5" r="5" :fill="color" />
      </marker>
    </defs>
    <path
      :d="pathD"
      :stroke="color"
      :stroke-width="strokeWidth"
      :stroke-dasharray="strokeDasharray"
      stroke-linecap="round"
      fill="none"
      :transform="`translate(${-svgBox.x}, ${-svgBox.y})`"
      :marker-start="
        startMarker ? `url(#${startMarker}-marker-${instanceId})` : undefined
      "
      :marker-end="
        endMarker ? `url(#${endMarker}-marker-${instanceId})` : undefined
      "
    />
  </svg>
</template>
