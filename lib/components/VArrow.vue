<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";

type Point = { x: number; y: number };
type Rect = { x: number; y: number; width: number; height: number };
export type VArrowTarget = HTMLElement | Point;
type Anchor = "top" | "bottom" | "left" | "right" | "center" | "auto";
type PathStyle = "straight" | "curve";

export interface VArrowProps {
  start: VArrowTarget;
  end: VArrowTarget;
  path?: PathStyle;
  color?: string;
  strokeWidth?: number;
  startMarker?: string | null;
  endMarker?: string | null;
  markerSize?: number;
  startAnchor?: Anchor;
  endAnchor?: Anchor;
  startOffset?: Point;
  endOffset?: Point;
  svgClass?: string;
  curvature?: number;
}

const props = withDefaults(defineProps<VArrowProps>(), {
  path: "straight",
  color: "#2c3e50",
  strokeWidth: 2,
  startMarker: null,
  endMarker: "arrow",
  markerSize: 10,
  startAnchor: "auto",
  endAnchor: "auto",
  startOffset: () => ({ x: 0, y: 0 }),
  endOffset: () => ({ x: 0, y: 0 }),
  svgClass: "",
  curvature: 0.3,
});

const instanceId = Math.random().toString(36).substring(7);
const startRect = ref<Rect | null>(null);
const endRect = ref<Rect | null>(null);
let animationFrameId: number | null = null;

onMounted(() => {
  updatePositions();
  startPositionTracker();
});

onUnmounted(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});

function isElement(val: unknown): val is HTMLElement {
  return val instanceof HTMLElement;
}

function updatePositions() {
  if (isElement(props.start))
    startRect.value = props.start.getBoundingClientRect();
  if (isElement(props.end)) endRect.value = props.end.getBoundingClientRect();
}

function startPositionTracker() {
  const track = () => {
    updatePositions();
    animationFrameId = requestAnimationFrame(track);
  };
  track();
}

const startPos = computed<Rect | null>(() => {
  if (isElement(props.start)) return startRect.value;
  if (props.start && "x" in props.start && "y" in props.start)
    return { ...props.start, width: 0, height: 0 };
  return null;
});

const endPos = computed<Rect | null>(() => {
  if (isElement(props.end)) return endRect.value;
  if (props.end && "x" in props.end && "y" in props.end)
    return { ...props.end, width: 0, height: 0 };
  return null;
});

const pathGeometry = computed(() => {
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

  const getBaseAnchorVec = (anchor: Anchor): Point => {
    if (anchor === "top") return { x: 0, y: -1 };
    if (anchor === "bottom") return { x: 0, y: 1 };
    if (anchor === "left") return { x: -1, y: 0 };
    if (anchor === "right") return { x: 1, y: 0 };
    return { x: 0, y: 0 };
  };

  let startVec = getBaseAnchorVec(startAnchor);
  let endVec = getBaseAnchorVec(endAnchor);

  if (props.path === "curve") {
    const dx = p2.x - p1.x;
    const dy = p2.y - p1.y;
    const angleFactor = 0.75;

    if (startAnchor === "top" || startAnchor === "bottom") {
      startVec.x += (dx / (Math.abs(dy) + 1)) * angleFactor;
    } else if (startAnchor === "left" || startAnchor === "right") {
      startVec.y += (dy / (Math.abs(dx) + 1)) * angleFactor;
    }

    if (endAnchor === "top" || endAnchor === "bottom") {
      endVec.x -= (dx / (Math.abs(dy) + 1)) * angleFactor;
    } else if (endAnchor === "left" || endAnchor === "right") {
      endVec.y -= (dy / (Math.abs(dx) + 1)) * angleFactor;
    }

    const magStart = Math.sqrt(
      startVec.x * startVec.x + startVec.y * startVec.y
    );
    if (magStart > 0) {
      startVec.x /= magStart;
      startVec.y /= magStart;
    }
    const magEnd = Math.sqrt(endVec.x * endVec.x + endVec.y * endVec.y);
    if (magEnd > 0) {
      endVec.x /= magEnd;
      endVec.y /= magEnd;
    }
  }

  const dx = p2.x - p1.x;
  const dy = p2.y - p1.y;
  const dist = Math.sqrt(dx * dx + dy * dy) * props.curvature;

  const cp1 = { x: p1.x + startVec.x * dist, y: p1.y + startVec.y * dist };
  const cp2 = { x: p2.x + endVec.x * dist, y: p2.y + endVec.y * dist };

  return { p1, p2, cp1, cp2, startVec, endVec };
});

const svgBox = computed(() => {
  if (!pathGeometry.value) return { x: 0, y: 0, width: 0, height: 0 };
  const { p1, p2, cp1, cp2 } = pathGeometry.value;
  const pad = props.strokeWidth + props.markerSize * 2;
  const points = [p1, p2];
  if (props.path === "curve") {
    points.push(cp1, cp2);
  }
  const minX = Math.min(...points.map((p) => p.x));
  const minY = Math.min(...points.map((p) => p.y));
  const maxX = Math.max(...points.map((p) => p.x));
  const maxY = Math.max(...points.map((p) => p.y));
  return {
    x: minX - pad,
    y: minY - pad,
    width: maxX - minX + pad * 2,
    height: maxY - minY + pad * 2,
  };
});

const pathD = computed(() => {
  if (!pathGeometry.value) return "";
  const { p1, p2, cp1, cp2 } = pathGeometry.value;
  if (props.path === "curve") {
    return `M ${p1.x} ${p1.y} C ${cp1.x} ${cp1.y}, ${cp2.x} ${cp2.y}, ${p2.x} ${p2.y}`;
  }
  return `M ${p1.x} ${p1.y} L ${p2.x} ${p2.y}`;
});

const startMarkerOrient = computed(() => {
  if (props.path !== "curve" || !pathGeometry.value)
    return "auto-start-reverse";
  const { startVec } = pathGeometry.value;
  return `${Math.atan2(startVec.y, startVec.x) * (180 / Math.PI)}`;
});

const endMarkerOrient = computed(() => {
  if (props.path !== "curve" || !pathGeometry.value)
    return "auto-start-reverse";
  const { endVec } = pathGeometry.value;
  return `${Math.atan2(-endVec.y, -endVec.x) * (180 / Math.PI)}`;
});
</script>

<template>
  <svg
    v-if="pathGeometry"
    :class="svgClass"
    :style="{
      position: 'absolute',
      left: `${svgBox.x}px`,
      top: `${svgBox.y}px`,
      pointerEvents: 'none',
      overflow: 'visible',
      zIndex: -1,
    }"
    :width="svgBox.width"
    :height="svgBox.height"
  >
    <defs>
      <marker
        :id="`arrow-marker-start-${instanceId}`"
        viewBox="0 0 10 10"
        refX="8"
        refY="5"
        :markerWidth="markerSize"
        :markerHeight="markerSize"
        :orient="startMarkerOrient"
      >
        <path d="M 0 2.5 L 8 5 L 0 7.5 z" :fill="color" />
      </marker>
      <marker
        :id="`arrow-marker-end-${instanceId}`"
        viewBox="0 0 10 10"
        refX="8"
        refY="5"
        :markerWidth="markerSize"
        :markerHeight="markerSize"
        :orient="endMarkerOrient"
      >
        <path d="M 0 2.5 L 8 5 L 0 7.5 z" :fill="color" />
      </marker>

      <marker
        :id="`circle-marker-start-${instanceId}`"
        viewBox="0 0 10 10"
        refX="5"
        refY="5"
        :markerWidth="markerSize"
        :markerHeight="markerSize"
      >
        <circle cx="5" cy="5" r="4" :fill="color" />
      </marker>
      <marker
        :id="`circle-marker-end-${instanceId}`"
        viewBox="0 0 10 10"
        refX="5"
        refY="5"
        :markerWidth="markerSize"
        :markerHeight="markerSize"
      >
        <circle cx="5" cy="5" r="4" :fill="color" />
      </marker>
    </defs>
    <path
      :d="pathD"
      :stroke="color"
      :stroke-width="strokeWidth"
      stroke-linecap="round"
      fill="none"
      :transform="`translate(${-svgBox.x}, ${-svgBox.y})`"
      :marker-start="
        startMarker
          ? `url(#${startMarker}-marker-start-${instanceId})`
          : undefined
      "
      :marker-end="
        endMarker ? `url(#${endMarker}-marker-end-${instanceId})` : undefined
      "
    />
  </svg>
</template>
