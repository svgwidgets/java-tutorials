
<template>
  <g
    class="sensor-display"
    preserve-aspect-ratio="xMidYMid meet"
    @click="handleClick"
    :transform="`translate(${position.x}, ${position.y}) rotate(${rotation || 0})`"
  >
    <!-- Body — rounded rectangle -->
    <rect
      x="0"
      y="0"
      :width="width"
      :height="height"
      :rx="cornerRadius"
      :ry="cornerRadius"
      :fill="COLORS.WHITE"
      :stroke="borderColor"
      :stroke-width="STROKE_WIDTH + 0.5"
      vector-effect="non-scaling-stroke"
    />

    <!-- Alarm stripe at top (thin colored bar showing status) -->
    <rect
      x="1"
      y="1"
      :width="width - 2"
      :height="4"
      :rx="cornerRadius"
      :fill="borderColor"
      opacity="0.8"
    />

    <!-- Tag identifier (e.g. PT, TT, FT) — top area -->
    <text
      v-if="tag"
      :x="width / 2"
      :y="16"
      text-anchor="middle"
      :font-size="8"
      font-weight="bold"
      :fill="COLORS.STROKE"
      class="sensor-text"
    >
      {{ tag }}
    </text>

    <!-- Value — large, centered -->
    <text
      :x="width / 2"
      :y="valueY"
      text-anchor="middle"
      :font-size="ANALOG_DISPLAY_FONTSIZE"
      font-weight="bold"
      :fill="valueColor"
      class="sensor-value"
    >
      {{ formattedValue }}
    </text>

    <!-- Units — below value, smaller -->
    <text
      :x="width / 2"
      :y="unitsY"
      text-anchor="middle"
      :font-size="8"
      :fill="COLORS.TEXT"
      class="sensor-text"
      opacity="0.7"
    >
      {{ units }}
    </text>

    <!-- Label (above component) -->
    <ComponentLabel
      :show="showLabel"
      :text="label"
      :position="{ x: width / 2, y: -8 }"
    />

    <!-- Port indicators -->
    <PortIndicator
      :ports="ports"
      :show="showPorts"
      :highlightedPortId="highlightedPortId"
      @port-click="handlePortClick"
      @port-mouse-enter="handlePortMouseEnter"
      @port-mouse-leave="handlePortMouseLeave"
    />
  </g>
</template>

<script setup lang="ts">
import { ANALOG_DISPLAY_FONTSIZE, COLORS, STROKE_WIDTH } from '@/lib/constants'
import type { AnalogComponentProps, ComponentEvents } from '@/lib/types'
import { getAnalogComponentColor } from '@/lib/utils/colorUtils'
import { computed } from 'vue'
import { ComponentLabel, PortIndicator } from '../../common'
import { usePortEvents } from '@/lib/composables/usePortEvents'

// ═══════════════════════════════════════════════
//  PROPS
// ═══════════════════════════════════════════════

interface AnalogDisplayProps extends AnalogComponentProps {
  highlightedPortId?: string
  tag?: string           // e.g. 'PT', 'TT', 'FT', 'LT'
  precision?: number     // decimal places (default 1)
}

const props = withDefaults(defineProps<AnalogDisplayProps>(), {
  showLabel: true,
  showPorts: false,
  alarm: 'none',
  precision: 1,
})

// ═══════════════════════════════════════════════
//  DIMENSIONS
//
//  Rectangular sensor display.
//  Default 48 x 40 — compact but readable.
//  Ports at left, right, top, bottom centers.
// ═══════════════════════════════════════════════

const width = computed(() => props.dimensions?.width ?? 48)
const height = computed(() => props.dimensions?.height ?? 40)
const cornerRadius = computed(() => Math.min(width.value, height.value) * 0.15)

// ═══════════════════════════════════════════════
//  LAYOUT POSITIONS
// ═══════════════════════════════════════════════

const valueY = computed(() => {
  // If tag is shown, push value down slightly
  return props.tag ? height.value / 2 + 2 : height.value / 2 + 1
})

const unitsY = computed(() => valueY.value + 10)

// ═══════════════════════════════════════════════
//  VALUE FORMATTING
// ═══════════════════════════════════════════════

const formattedValue = computed(() => {
  if (props.value === undefined || props.value === null) return '--'
  return Number(props.value).toFixed(props.precision)
})

// ═══════════════════════════════════════════════
//  COLORS
// ═══════════════════════════════════════════════

const valueColor = computed(() => getAnalogComponentColor(props.alarm))
const borderColor = computed(() => getAnalogComponentColor(props.alarm))

// ═══════════════════════════════════════════════
//  PORTS
//
//  Rectangle-based ports at edge midpoints.
//  These are the actual positions pipes connect to.
// ═══════════════════════════════════════════════

const ports = computed(() => [
  { id: 'left', x: 0, y: height.value / 2 },
  { id: 'right', x: width.value, y: height.value / 2 },
  { id: 'top', x: width.value / 2, y: 0 },
  { id: 'bottom', x: width.value / 2, y: height.value },
])

// ═══════════════════════════════════════════════
//  EVENTS
// ═══════════════════════════════════════════════

const emit = defineEmits<ComponentEvents>()

function handleClick() {
  emit('click')
}

const { handlePortClick, handlePortMouseEnter, handlePortMouseLeave } = usePortEvents(emit)

defineExpose({
  ports: ports,
})
</script>

<style lang="css" scoped>
.sensor-display {
  cursor: pointer;
}

.sensor-display:hover {
  filter: brightness(1.1);
}

.sensor-value {
  font-family: 'IBM Plex Mono', monospace;
  pointer-events: none;
  user-select: none;
}

.sensor-text {
  font-family: 'IBM Plex Sans', sans-serif;
  pointer-events: none;
  user-select: none;
}
</style>
