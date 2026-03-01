<template>
  <!--
    Toggle — Binary control (pure SVG)

    Placement: drop anywhere inside an <svg> or <g> element.
    x / y are the CENTER of the toggle track on the canvas.
    Emits update:modelValue and change so it works with v-model.

    Usage:
      <Toggle v-model="valveOpen" :x="200" :y="150" label="V-001" />
      <Toggle v-model="pumpOn" active-color="#3b82f6" show-state-labels />
      <Toggle v-model="flag" :disabled="!hasPermission" />
  -->
  <g
    :transform="`translate(${x}, ${y})`"
    :style="{ cursor: disabled ? 'not-allowed' : 'pointer' }"
    role="button"
    :aria-label="`${label}: ${modelValue ? onLabel : offLabel}`"
    :aria-pressed="modelValue"
    :aria-disabled="disabled"
    @click="handleClick"
  >
    <!-- Track drop shadow -->
    <rect
      :x="-halfW" :y="-halfH + 2"
      :width="trackWidth" :height="trackHeight"
      :rx="halfH"
      fill="#000" opacity="0.3"
    />

    <!-- Track -->
    <rect
      :x="-halfW" :y="-halfH"
      :width="trackWidth" :height="trackHeight"
      :rx="halfH"
      :fill="modelValue ? activeColor : inactiveColor"
      :stroke="modelValue ? activeBorder : '#1f2937'"
      stroke-width="1"
    />

    <!-- Inner groove line (gives depth) -->
    <rect
      :x="-halfW + 2" :y="-halfH + 2"
      :width="trackWidth - 4" :height="trackHeight - 4"
      :rx="halfH - 2"
      fill="none"
      :stroke="modelValue ? 'rgba(255,255,255,0.25)' : 'rgba(255,255,255,0.06)'"
      stroke-width="1"
    />

    <!-- OFF label (left of track) -->
    <text
      v-if="showStateLabels"
      :x="-halfW - 4" y="0"
      text-anchor="end"
      dominant-baseline="middle"
      :font-size="fontSize - 1"
      font-family="'Courier New', Courier, monospace"
      font-weight="700"
      :fill="!modelValue ? '#9ca3af' : '#374151'"
    >{{ offLabel }}</text>

    <!-- ON label (right of track) -->
    <text
      v-if="showStateLabels"
      :x="halfW + 4" y="0"
      text-anchor="start"
      dominant-baseline="middle"
      :font-size="fontSize - 1"
      font-family="'Courier New', Courier, monospace"
      font-weight="700"
      :fill="modelValue ? activeColor : '#374151'"
    >{{ onLabel }}</text>

    <!-- Thumb drop shadow -->
    <circle
      :cx="thumbCx + 1" cy="2"
      :r="thumbR"
      fill="#000" opacity="0.25"
    />

    <!-- Thumb -->
    <circle
      :cx="thumbCx" cy="0"
      :r="thumbR"
      fill="#e5e7eb"
      stroke="#9ca3af"
      stroke-width="1"
    />

    <!-- Thumb specular -->
    <circle
      :cx="thumbCx - thumbR * 0.28"
      :cy="-thumbR * 0.28"
      :r="thumbR * 0.3"
      fill="white"
      opacity="0.4"
    />

    <!-- Label below -->
    <text
      v-if="showLabel && label"
      x="0"
      :y="halfH + 12"
      text-anchor="middle"
      :font-size="fontSize"
      font-family="'Courier New', Courier, monospace"
      font-weight="600"
      letter-spacing="0.5"
      fill="#9ca3af"
    >{{ label }}</text>

    <!-- Disabled scrim -->
    <rect
      v-if="disabled"
      :x="-halfW" :y="-halfH"
      :width="trackWidth" :height="trackHeight"
      :rx="halfH"
      fill="#000" opacity="0.45"
    />
  </g>
</template>

<script setup lang="ts">
import { computed } from 'vue'

const props = withDefaults(defineProps<{
  /** Current boolean state (use with v-model) */
  modelValue: boolean
  /** Center X on the SVG canvas */
  x?: number
  /** Center Y on the SVG canvas */
  y?: number
  /** Total width of the track */
  trackWidth?: number
  /** Total height of the track */
  trackHeight?: number
  /** Track / thumb color when true */
  activeColor?: string
  /** Track color when false */
  inactiveColor?: string
  /** Label shown below the toggle */
  label?: string
  /** Whether to render the label */
  showLabel?: boolean
  /** Whether to render ON/OFF beside the track */
  showStateLabels?: boolean
  /** State string when true */
  onLabel?: string
  /** State string when false */
  offLabel?: string
  /** Prevent interaction */
  disabled?: boolean
  /** Base font size for labels */
  fontSize?: number
}>(), {
  x: 0,
  y: 0,
  trackWidth: 36,
  trackHeight: 16,
  activeColor: '#22c55e',
  inactiveColor: '#374151',
  label: '',
  showLabel: true,
  showStateLabels: false,
  onLabel: 'ON',
  offLabel: 'OFF',
  disabled: false,
  fontSize: 9,
})

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
  'change': [value: boolean]
}>()

const halfW  = computed(() => props.trackWidth / 2)
const halfH  = computed(() => props.trackHeight / 2)
const thumbR = computed(() => props.trackHeight / 2 - 2)

/** Thumb X: slides between left and right travel limits */
const thumbCx = computed(() => {
  const travel = halfW.value - thumbR.value - 1
  return props.modelValue ? travel : -travel
})

/** Darker border derived from activeColor */
const activeBorder = computed(() => {
  const n = parseInt(props.activeColor.replace('#', ''), 16)
  const clamp = (v: number) => Math.max(0, v - 50)
  const r = clamp(n >> 16)
  const g = clamp((n >> 8) & 0xff)
  const b = clamp(n & 0xff)
  return `#${((r << 16) | (g << 8) | b).toString(16).padStart(6, '0')}`
})

function handleClick() {
  if (props.disabled) return
  const next = !props.modelValue
  emit('update:modelValue', next)
  emit('change', next)
}
</script>
