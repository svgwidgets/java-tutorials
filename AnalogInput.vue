<template>
  <!--
    AnalogInput — Analog setpoint entry (SVG panel + native <input>)

    Placement: drop anywhere inside an <svg> or <g> element.
    x / y are the CENTER of the widget on the canvas.
    Uses a single <foreignObject> with a bare <input type="number"> —
    no HTML component library dependency, no foreignObject nesting issues.

    Emits:
      update:modelValue  — on every valid keystroke (for v-model binding)
      change             — same as above
      commit             — on Enter or blur, with the final clamped value

    Usage:
      <AnalogInput v-model="pressure" :x="300" :y="200"
        label="PT-001" unit="bar" :min="0" :max="200" :precision="1" />
  -->
  <g :transform="`translate(${x}, ${y})`">

    <!-- Panel shadow -->
    <rect
      :x="-hw + 2" :y="-hh + 3"
      :width="width" :height="height"
      rx="3" fill="#000" opacity="0.3"
    />

    <!-- Panel body -->
    <rect
      :x="-hw" :y="-hh"
      :width="width" :height="height"
      rx="3"
      fill="#111827"
      :stroke="panelStroke"
      stroke-width="1"
    />

    <!-- Top accent bar -->
    <rect
      :x="-hw" :y="-hh"
      :width="width" height="3"
      rx="3"
      :fill="accentColor"
    />

    <!-- Label -->
    <text
      v-if="showLabel && label"
      x="0" :y="-hh + 14"
      text-anchor="middle"
      :font-size="fontSize"
      font-family="'Courier New', Courier, monospace"
      font-weight="700"
      letter-spacing="1"
      fill="#94a3b8"
    >{{ label }}</text>

    <!-- ── Value display row ─────────────────────────────────── -->
    <rect
      :x="-hw + 5" :y="-hh + 19"
      :width="width - 10" :height="displayH"
      rx="2"
      fill="#0f172a"
      stroke="#1e293b"
      stroke-width="1"
    />

    <!-- Numeric value text -->
    <text
      :x="unit ? hw - unitW - 12 : 0"
      :y="-hh + 19 + displayH / 2"
      :text-anchor="unit ? 'end' : 'middle'"
      dominant-baseline="middle"
      :font-size="fontSize + 4"
      font-family="'Courier New', Courier, monospace"
      font-weight="700"
      letter-spacing="1"
      :fill="displayTextColor"
    >{{ displayValue }}</text>

    <!-- Unit label -->
    <text
      v-if="unit"
      :x="hw - 8"
      :y="-hh + 19 + displayH / 2"
      text-anchor="end"
      dominant-baseline="middle"
      :font-size="fontSize - 1"
      font-family="'Courier New', Courier, monospace"
      fill="#475569"
    >{{ unit }}</text>

    <!-- Validation dot -->
    <circle
      :cx="hw - 8" :cy="-hh + 19 + displayH / 2"
      r="3"
      :fill="dotColor"
    />

    <!-- ── Range bar ───────────────────────────────────────────── -->
    <rect
      :x="-hw + 5" :y="-hh + 19 + displayH + 6"
      :width="width - 10" height="3"
      rx="1.5"
      fill="#1f2937"
      stroke="#374151"
      stroke-width="0.5"
    />
    <rect
      :x="-hw + 5" :y="-hh + 19 + displayH + 6"
      :width="rangeFill" height="3"
      rx="1.5"
      :fill="accentColor"
      opacity="0.85"
    />

    <!-- Min / SET PT / Max labels -->
    <text
      :x="-hw + 7" :y="-hh + 19 + displayH + 17"
      text-anchor="start"
      :font-size="fontSize - 2"
      font-family="'Courier New', Courier, monospace"
      fill="#374151"
    >{{ min }}</text>

    <text
      x="0" :y="-hh + 19 + displayH + 17"
      text-anchor="middle"
      :font-size="fontSize - 2"
      font-family="'Courier New', Courier, monospace"
      fill="#4b5563"
    >SET PT</text>

    <text
      :x="hw - 7" :y="-hh + 19 + displayH + 17"
      text-anchor="end"
      :font-size="fontSize - 2"
      font-family="'Courier New', Courier, monospace"
      fill="#374151"
    >{{ max }}</text>

    <!-- ── Native input via foreignObject ─────────────────────── -->
    <!--
      foreignObject is the only viable way to get a real keyboard-driven
      text input inside SVG. We use a bare <input> with no library so
      there are no extra DOM layers or theming conflicts.
      x/y/width/height are top-left positioned (SVG spec).
    -->
    <foreignObject
      :x="-hw + 5"
      :y="hh - inputH - 5"
      :width="width - 10"
      :height="inputH"
    >
      <!--
        xmlns is required — without it browsers won't parse HTML
        inside SVG foreignObject correctly.
      -->
      <body xmlns="http://www.w3.org/1999/xhtml" style="margin:0;padding:0">
        <input
          ref="inputEl"
          type="number"
          :min="min"
          :max="max"
          :step="step"
          :value="pending"
          :disabled="disabled"
          :placeholder="String(modelValue.toFixed(precision))"
          @input="onInput"
          @keydown.enter.prevent="onCommit"
          @keydown.escape.prevent="onCancel"
          @blur="onCommit"
          :style="inputStyles"
          :aria-label="`Set ${label} value (${min}–${max} ${unit})`"
        />
      </body>
    </foreignObject>

    <!-- Disabled overlay -->
    <rect
      v-if="disabled"
      :x="-hw" :y="-hh"
      :width="width" :height="height"
      rx="3"
      fill="#000" opacity="0.45"
      style="cursor: not-allowed"
    />
  </g>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'

const props = withDefaults(defineProps<{
  /** Current value (use with v-model) */
  modelValue: number
  /** Center X on the SVG canvas */
  x?: number
  /** Center Y on the SVG canvas */
  y?: number
  /** Widget width in SVG units */
  width?: number
  /** Widget height in SVG units */
  height?: number
  /** Minimum allowed value */
  min?: number
  /** Maximum allowed value */
  max?: number
  /** Step for the native input spinner */
  step?: number
  /** Decimal places shown in the display */
  precision?: number
  /** Unit string shown beside the value (e.g. "bar", "°C", "%") */
  unit?: string
  /** Label shown at the top of the panel */
  label?: string
  /** Whether to render the label */
  showLabel?: boolean
  /** Accent color (top bar + range fill) */
  accentColor?: string
  /** Prevent interaction */
  disabled?: boolean
  /** Base font size */
  fontSize?: number
}>(), {
  x: 0,
  y: 0,
  width: 90,
  height: 105,
  min: 0,
  max: 100,
  step: 1,
  precision: 1,
  unit: '',
  label: '',
  showLabel: true,
  accentColor: '#3b82f6',
  disabled: false,
  fontSize: 9,
})

const emit = defineEmits<{
  'update:modelValue': [value: number]
  'change': [value: number]
  /** Fired on Enter / blur with the final clamped value */
  'commit': [value: number]
}>()

// ── internal state ────────────────────────────────────────────────────────────

const inputEl = ref<HTMLInputElement | null>(null)

/** What the user is currently typing — starts as the current modelValue */
const pending = ref(String(props.modelValue))
/** Whether the current pending string parses to a valid, in-range number */
const isValid = ref(true)

// Keep pending in sync when the prop changes externally
watch(() => props.modelValue, v => {
  pending.value = String(v)
  isValid.value = true
})

// ── computed layout helpers ───────────────────────────────────────────────────

const hw       = computed(() => props.width / 2)
const hh       = computed(() => props.height / 2)
const displayH = computed(() => 22)
const inputH   = computed(() => 18)

/** Approximate char width for the unit label (to push value text left) */
const unitW = computed(() => props.unit.length * 5)

/** Range bar fill width proportional to modelValue */
const rangeFill = computed(() => {
  const pct = Math.min(1, Math.max(0, (props.modelValue - props.min) / (props.max - props.min)))
  return pct * (props.width - 10)
})

// ── derived display ───────────────────────────────────────────────────────────

const displayValue = computed(() => {
  const n = parseFloat(pending.value)
  return isNaN(n) ? props.modelValue.toFixed(props.precision) : n.toFixed(props.precision)
})

const displayTextColor = computed(() => isValid.value ? '#e2e8f0' : '#ef4444')
const dotColor         = computed(() => isValid.value ? '#22c55e' : '#ef4444')
const panelStroke      = computed(() => isValid.value ? '#1e2533' : '#ef4444')

// ── input styles (applied inline so they work inside foreignObject) ────────────
const inputStyles = computed(() => ({
  width: '100%',
  height: '100%',
  background: '#0f172a',
  border: `1px solid ${isValid.value ? '#334155' : '#ef4444'}`,
  borderRadius: '2px',
  color: '#cbd5e1',
  fontFamily: "'Courier New', Courier, monospace",
  fontSize: `${props.fontSize}px`,
  fontWeight: '600',
  textAlign: 'center' as const,
  outline: 'none',
  padding: '0 4px',
  boxSizing: 'border-box' as const,
  // Hide the browser spinner arrows — they take space and look wrong in SVG panels
  MozAppearance: 'textfield' as const,
  WebkitAppearance: 'none' as const,
  appearance: 'none' as const,
}))

// ── event handlers ────────────────────────────────────────────────────────────

function onInput(e: Event) {
  const raw = (e.target as HTMLInputElement).value
  pending.value = raw
  const n = parseFloat(raw)
  if (!isNaN(n) && n >= props.min && n <= props.max) {
    isValid.value = true
    emit('update:modelValue', n)
    emit('change', n)
  } else {
    // Allow intermediate states like "-" or empty without hard-erroring
    isValid.value = raw === '' || raw === '-'
  }
}

function onCommit() {
  const n = parseFloat(pending.value)
  if (!isNaN(n)) {
    const clamped = Math.min(props.max, Math.max(props.min, n))
    pending.value = String(clamped)
    isValid.value = true
    emit('update:modelValue', clamped)
    emit('commit', clamped)
  } else {
    // Revert to last good value
    pending.value = String(props.modelValue)
    isValid.value = true
  }
}

function onCancel() {
  pending.value = String(props.modelValue)
  isValid.value = true
  inputEl.value?.blur()
}
</script>
