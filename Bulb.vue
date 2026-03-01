<template>
  <!--
    Bulb — Binary state indicator (pure SVG)

    Placement: drop anywhere inside an <svg> or <g> element.
    x / y are the CENTER of the bulb on the canvas.

    Usage:
      <Bulb :value="pumpRunning" :x="120" :y="80" label="P-001" />
      <Bulb :value="alarmActive" active-color="#ef4444" :pulse="true" label="ALM" show-state />
  -->
  <g
    :transform="`translate(${x}, ${y})`"
    role="img"
    :aria-label="`${label}: ${value ? onLabel : offLabel}`"
  >
    <!-- Housing ring -->
    <circle :r="outerR" fill="#1e2530" stroke="#111827" stroke-width="1.5" />

    <!-- Pulsing glow halo (active only) -->
    <circle v-if="value && pulse" :r="outerR - 1" :fill="activeColor" opacity="0.25">
      <animate
        attributeName="opacity"
        values="0.1;0.4;0.1"
        :dur="`${pulseDuration}s`"
        repeatCount="indefinite"
      />
    </circle>

    <!-- Lens -->
    <circle
      :r="lensR"
      :fill="value ? activeColor : inactiveColor"
      :stroke="value ? activeBorder : '#1f2937'"
      stroke-width="1"
    />

    <!-- Specular highlight -->
    <ellipse
      :rx="lensR * 0.22"
      :ry="lensR * 0.15"
      :cx="-lensR * 0.15"
      :cy="-lensR * 0.22"
      fill="white"
      :opacity="value ? 0.65 : 0.15"
    />

    <!-- Label -->
    <text
      v-if="showLabel && label"
      x="0"
      :y="outerR + 12"
      text-anchor="middle"
      :font-size="fontSize"
      font-family="'Courier New', Courier, monospace"
      font-weight="600"
      letter-spacing="0.5"
      fill="#9ca3af"
    >{{ label }}</text>

    <!-- State text (ON / OFF) -->
    <text
      v-if="showState"
      x="0"
      :y="outerR + 12 + (showLabel && label ? fontSize + 2 : 0)"
      text-anchor="middle"
      :font-size="fontSize - 1"
      font-family="'Courier New', Courier, monospace"
      :fill="value ? activeColor : '#6b7280'"
    >{{ value ? onLabel : offLabel }}</text>
  </g>
</template>

<script setup lang="ts">
import { computed } from 'vue'

const props = withDefaults(defineProps<{
  /** Current boolean state */
  value: boolean
  /** Center X on the SVG canvas */
  x?: number
  /** Center Y on the SVG canvas */
  y?: number
  /** Lens diameter in SVG units */
  size?: number
  /** Lens color when value is true */
  activeColor?: string
  /** Lens color when value is false */
  inactiveColor?: string
  /** Text shown below the bulb */
  label?: string
  /** Whether to render the label */
  showLabel?: boolean
  /** State string when true */
  onLabel?: string
  /** State string when false */
  offLabel?: string
  /** Whether to show onLabel / offLabel text */
  showState?: boolean
  /** Animate a glow pulse when active */
  pulse?: boolean
  /** Pulse animation duration in seconds */
  pulseDuration?: number
  /** Base font size for label text */
  fontSize?: number
}>(), {
  x: 0,
  y: 0,
  size: 16,
  activeColor: '#22c55e',
  inactiveColor: '#374151',
  label: '',
  showLabel: true,
  onLabel: 'ON',
  offLabel: 'OFF',
  showState: false,
  pulse: true,
  pulseDuration: 1.8,
  fontSize: 9,
})

const lensR  = computed(() => props.size / 2)
const outerR = computed(() => props.size / 2 + 3)

/** Darken the activeColor slightly for the lens stroke */
const activeBorder = computed(() => {
  const n = parseInt(props.activeColor.replace('#', ''), 16)
  const clamp = (v: number) => Math.max(0, v - 60)
  const r = clamp(n >> 16)
  const g = clamp((n >> 8) & 0xff)
  const b = clamp(n & 0xff)
  return `#${((r << 16) | (g << 8) | b).toString(16).padStart(6, '0')}`
})
</script>
