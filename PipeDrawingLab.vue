<template>
  <!--
    Pipe Drawing Lab v4
    ====================
    Port positions read directly from component instances via defineExpose.
    No portsForType(). No hardcoded radius. No registry sync issues.
    The component IS the source of truth for its own ports.
  -->
  <div class="lab">
    <header class="lab-header">
      <span class="lab-title">⬡ Pipe Drawing Lab</span>
      <span class="lab-sep" />
      <span class="lab-sub">Orthogonal + Free Draw · Port positions from components</span>
    </header>

    <div class="lab-body">
      <!-- ── Controls ── -->
      <aside class="controls">
        <div class="ctrl-section">
          <div class="ctrl-heading">Drawing Mode</div>
          <button v-for="m in modes" :key="m.id" class="mode-btn" :class="{ active: drawingMode === m.id }"
            @click="drawingMode = m.id; cancelDrawing()">
            <span class="mode-icon">{{ m.icon }}</span>
            <div><span class="mode-name">{{ m.name }}</span><br /><span class="mode-desc">{{ m.desc }}</span></div>
          </button>
        </div>

        <div class="ctrl-section">
          <div class="ctrl-heading">Options</div>
          <label class="ctrl-toggle"><input type="checkbox" v-model="snapEnabled" /><span>Snap to grid ({{ gridSize }}px)</span></label>
          <label class="ctrl-toggle"><input type="checkbox" v-model="showPorts" /><span>Show ports</span></label>
        </div>

        <div class="ctrl-section">
          <div class="ctrl-heading">Instructions</div>
          <div class="instructions" v-html="instructions" />
        </div>

        <div class="ctrl-section">
          <div class="ctrl-heading">Connections ({{ connections.length }})</div>
          <div v-for="conn in connections" :key="conn.id" class="conn-item"
            :class="{ selected: selectedConnId === conn.id }" @click="selectedConnId = conn.id">
            <span class="cl">{{ conn.from.nodeId }}:{{ conn.from.portId }}</span>
            <span class="ca">→</span>
            <span class="cl">{{ conn.to.nodeId }}:{{ conn.to.portId }}</span>
            <span class="cw">{{ conn.waypoints.length }}wp</span>
            <button class="cd" @click.stop="deleteConnection(conn.id)">×</button>
          </div>
          <div v-if="!connections.length" class="conn-empty">Click a port to start.</div>
        </div>

        <div class="ctrl-section">
          <button class="action-btn" @click="resetAll">Clear All</button>
          <button class="action-btn" @click="exportJson">Export JSON</button>
        </div>
      </aside>

      <!-- ── Canvas ── -->
      <div class="canvas-area">
        <svg
          ref="svgEl"
          class="canvas-svg"
          :viewBox="`0 0 ${canvasW} ${canvasH}`"
          @mousemove="onMouseMove"
          @click="onCanvasClick"
          @contextmenu.prevent="cancelDrawing"
          @keydown.escape="cancelDrawing"
          tabindex="0"
        >
          <defs>
            <pattern id="sg" :width="gridSize" :height="gridSize" patternUnits="userSpaceOnUse">
              <path :d="`M ${gridSize} 0 L 0 0 0 ${gridSize}`" fill="none" stroke="rgba(255,255,255,0.04)" stroke-width="0.5" />
            </pattern>
            <pattern id="lg" :width="gridSize*5" :height="gridSize*5" patternUnits="userSpaceOnUse">
              <rect :width="gridSize*5" :height="gridSize*5" fill="url(#sg)" />
              <path :d="`M ${gridSize*5} 0 L 0 0 0 ${gridSize*5}`" fill="none" stroke="rgba(255,255,255,0.07)" stroke-width="0.5" />
            </pattern>
          </defs>

          <rect x="0" y="0" :width="canvasW" :height="canvasH" fill="url(#lg)" />

          <!-- Alignment guides while drawing -->
          <g v-if="draw.active && mousePos">
            <line :x1="lastAnchor.x" y1="0" :x2="lastAnchor.x" :y2="canvasH"
                  stroke="rgba(88,166,255,0.08)" stroke-width="0.5" stroke-dasharray="4 4" />
            <line x1="0" :y1="lastAnchor.y" :x2="canvasW" :y2="lastAnchor.y"
                  stroke="rgba(88,166,255,0.08)" stroke-width="0.5" stroke-dasharray="4 4" />
          </g>

          <!-- ═══ FINISHED CONNECTIONS ═══ -->
          <g v-for="conn in connections" :key="conn.id" @click.stop="selectedConnId = conn.id">
            <!-- Selection glow -->
            <path v-if="selectedConnId === conn.id" :d="buildPath(conn)"
              fill="none" stroke="rgba(88,166,255,0.25)" stroke-width="8"
              stroke-linecap="round" stroke-linejoin="round" />
            <!-- Hit area -->
            <path :d="buildPath(conn)" fill="none" stroke="transparent" stroke-width="14"
              style="pointer-events:stroke;cursor:pointer" />
            <!-- YOUR ConnectionPipe -->
            <ConnectionPipe
              :start-position="portWorldPos(conn.from.nodeId, conn.from.portId)"
              :end-position="portWorldPos(conn.to.nodeId, conn.to.portId)"
              :waypoints="conn.waypoints"
              :flowing="conn.props.flowing"
              :direction="conn.props.direction"
            />
            <!-- Waypoint handles when selected -->
            <g v-if="selectedConnId === conn.id">
              <circle v-for="(wp, i) in conn.waypoints" :key="i"
                :cx="wp.x" :cy="wp.y" r="4"
                fill="#0e1117" stroke="#58a6ff" stroke-width="1.5" class="wp-handle"
                @mousedown.stop="startWpDrag(conn.id, i, $event)"
                @dblclick.stop="conn.waypoints.splice(i, 1)" />
            </g>
          </g>

          <!-- ═══ PREVIEW PIPE (while drawing) ═══ -->
          <g v-if="draw.active">
            <ConnectionPipe v-if="draw.waypoints.length > 0"
              :start-position="draw.startPos"
              :end-position="draw.waypoints[draw.waypoints.length - 1]"
              :waypoints="draw.waypoints.slice(0, -1)"
              :flowing="false" />
            <path v-if="rubberBand" :d="rubberBand" fill="none" stroke="#58a6ff" stroke-width="2"
              stroke-dasharray="6 4" stroke-linecap="round" stroke-linejoin="round" opacity="0.5" />
            <circle :cx="draw.startPos.x" :cy="draw.startPos.y" r="6"
              fill="none" stroke="#58a6ff" stroke-width="1.5" class="pulse" />
          </g>

          <!-- ═══ COMPONENTS ═══ -->
          <g v-for="node in nodes" :key="node.id"
            :transform="`translate(${node.position.x}, ${node.position.y})`"
            class="node-group">
            <component
              :is="componentMap[node.typeId]"
              :ref="(el: any) => setCompRef(node.id, el)"
              v-bind="node.props"
              :label="node.label"
              :show-label="true"
              :show-ports="showPorts"
              :highlighted-port-id="getHighlightedPort(node.id)"
              @port-click="(portId: string) => onPortClick(node.id, portId)"
              @port-mouse-enter="(portId: string) => onPortEnter(node.id, portId)"
              @port-mouse-leave="(portId: string) => onPortLeave(node.id, portId)"
              @click="selectedConnId = null"
            />
          </g>

          <!-- Snap target indicator -->
          <circle v-if="snapTarget" :cx="snapTarget.world.x" :cy="snapTarget.world.y"
            r="10" fill="none" stroke="#3fb950" stroke-width="2" class="pulse-green" />
        </svg>

        <div class="mode-badge" :class="drawingMode">{{ drawingMode.toUpperCase() }}</div>

        <div v-if="draw.active" class="draw-status">
          <span class="dot" />
          From <strong>{{ draw.fromNodeId }}:{{ draw.fromPortId }}</strong>
          · {{ draw.waypoints.length }}wp
          · <button class="cancel-link" @click="cancelDrawing">Cancel</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted, type Component, markRaw, nextTick } from 'vue'

// ═══════════════════════════════════════════════
//  YOUR LIBRARY IMPORTS
// ═══════════════════════════════════════════════

import ManualValve from '@/lib/components/PID/valves/ManualValve.vue'
import CentrifugalPump from '@/lib/components/PID/pumps/GeneralPump.vue'
import VerticalTank from '@/lib/components/PID/tanks/VerticalTank.vue'
import PressureSensor from '@/lib/components/PID/sensors/PressureSensor.vue'
import ConnectionPipe from '@/lib/components/PID/connectors/ConnectionPipe.vue'

import type { PortDefinition, Position } from '@/lib/types'

// ═══════════════════════════════════════════════
//  COMPONENT MAP
// ═══════════════════════════════════════════════

const componentMap: Record<string, Component> = {
  'manual-valve': markRaw(ManualValve),
  'centrifugal-pump': markRaw(CentrifugalPump),
  'vertical-tank': markRaw(VerticalTank),
  'pressure-sensor': markRaw(PressureSensor),
}

// ═══════════════════════════════════════════════
//  COMPONENT REFS — the single source of truth
//
//  Each component exposes { ports } via defineExpose.
//  We read ports directly from the mounted instance.
//  No registry, no hardcoded dimensions, no sync issues.
// ═══════════════════════════════════════════════

const compRefs = ref<Record<string, any>>({})

function setCompRef(nodeId: string, el: any) {
  if (el) {
    compRefs.value[nodeId] = el
  } else {
    delete compRefs.value[nodeId]
  }
}

/**
 * Read port positions from the actual component instance.
 * Falls back gracefully if ref not yet mounted.
 */
function getExposedPorts(nodeId: string): PortDefinition[] {
  const comp = compRefs.value[nodeId]
  if (!comp) return []
  // ports is a computed ref exposed via defineExpose
  // Access .value if it's a ref, otherwise use directly
  const ports = comp.ports
  if (!ports) return []
  return Array.isArray(ports) ? ports : (ports.value ?? [])
}

/**
 * Get a specific port's world position.
 * Port local coords come from the component itself.
 * World = node.position + port local position.
 */
function portWorldPos(nodeId: string, portId: string): Position {
  const node = nodes.find(n => n.id === nodeId)
  if (!node) return { x: 0, y: 0 }

  const ports = getExposedPorts(nodeId)
  const port = ports.find(p => p.id === portId)
  if (!port) return node.position

  return {
    x: node.position.x + port.x,
    y: node.position.y + port.y,
  }
}

// ═══════════════════════════════════════════════
//  TYPES
// ═══════════════════════════════════════════════

interface NodeData {
  id: string; typeId: string; position: Position; label: string
  props: Record<string, any>
}

interface ConnData {
  id: string
  from: { nodeId: string; portId: string }
  to: { nodeId: string; portId: string }
  waypoints: Position[]
  props: { flowing: boolean; direction: string }
}

type DrawMode = 'click' | 'ortho'

// ═══════════════════════════════════════════════
//  STATE
// ═══════════════════════════════════════════════

const canvasW = 900, canvasH = 500, gridSize = ref(10)
const snapEnabled = ref(true)
const showPorts = ref(true)
const selectedConnId = ref<string | null>(null)
const hoveredPort = ref<{ nodeId: string; portId: string } | null>(null)
const mousePos = ref<Position | null>(null)
const svgEl = ref<SVGSVGElement | null>(null)
const drawingMode = ref<DrawMode>('ortho')
let connCtr = 0

const draw = reactive({
  active: false,
  fromNodeId: '',
  fromPortId: '',
  startPos: { x: 0, y: 0 } as Position,
  waypoints: [] as Position[],
})

const wpDrag = reactive({ active: false, connId: '', idx: -1 })

const modes = [
  { id: 'ortho' as DrawMode, icon: '⊾', name: 'Ortho Draw', desc: 'H/V segments only.' },
  { id: 'click' as DrawMode, icon: '✎', name: 'Free Draw', desc: 'Waypoints at any angle.' },
]

const nodes = reactive<NodeData[]>([
  { id: 'T-001', typeId: 'vertical-tank', position: { x: 50, y: 80 }, label: 'T-001', props: { level: 65, alarm: 'none' } },
  { id: 'V-001', typeId: 'manual-valve', position: { x: 240, y: 208 }, label: 'V-001', props: { state: 'open', alarm: 'none' } },
  { id: 'P-001', typeId: 'centrifugal-pump', position: { x: 380, y: 200 }, label: 'P-001', props: { state: 'running', alarm: 'none' } },
  { id: 'T-002', typeId: 'vertical-tank', position: { x: 560, y: 80 }, label: 'T-002', props: { level: 40, alarm: 'none' } },
  { id: 'V-002', typeId: 'manual-valve', position: { x: 720, y: 208 }, label: 'V-002', props: { state: 'closed', alarm: 'none' } },
  { id: 'PT-001', typeId: 'pressure-sensor', position: { x: 460, y: 50 }, label: 'PT-001', props: { value: 125.5, units: 'PSI', alarm: 'none' } },
])

const connections = reactive<ConnData[]>([])

// ═══════════════════════════════════════════════
//  PORT HIGHLIGHTING
// ═══════════════════════════════════════════════

function getHighlightedPort(nodeId: string): string | undefined {
  if (hoveredPort.value?.nodeId === nodeId) return hoveredPort.value.portId
  if (draw.active && draw.fromNodeId === nodeId) return draw.fromPortId
  if (snapTarget.value?.nodeId === nodeId) return snapTarget.value.portId
  return undefined
}

// ═══════════════════════════════════════════════
//  SNAP LOGIC
//
//  Port positions are NEVER snapped.
//  Only intermediate waypoints snap to grid.
//  In ortho mode, first waypoint aligns to port axis
//  so the segment is always perfectly H or V.
// ═══════════════════════════════════════════════

function snapGrid(p: Position): Position {
  if (!snapEnabled.value) return p
  const g = gridSize.value
  return { x: Math.round(p.x / g) * g, y: Math.round(p.y / g) * g }
}

function snapOrtho(raw: Position, anchor: Position): { corner: Position | null; point: Position } {
  const dx = Math.abs(raw.x - anchor.x)
  const dy = Math.abs(raw.y - anchor.y)

  if (dx < 5) {
    const y = snapEnabled.value ? Math.round(raw.y / gridSize.value) * gridSize.value : raw.y
    return { corner: null, point: { x: anchor.x, y } }
  }
  if (dy < 5) {
    const x = snapEnabled.value ? Math.round(raw.x / gridSize.value) * gridSize.value : raw.x
    return { corner: null, point: { x, y: anchor.y } }
  }

  const sx = snapEnabled.value ? Math.round(raw.x / gridSize.value) * gridSize.value : raw.x
  const sy = snapEnabled.value ? Math.round(raw.y / gridSize.value) * gridSize.value : raw.y
  return { corner: { x: sx, y: anchor.y }, point: { x: sx, y: sy } }
}

function alignToTargetPort(waypoints: Position[], startPos: Position, targetPos: Position): Position[] {
  const wp = [...waypoints]
  const last = wp.length > 0 ? wp[wp.length - 1] : startPos
  if (Math.abs(last.x - targetPos.x) < 2 || Math.abs(last.y - targetPos.y) < 2) return wp
  wp.push({ x: targetPos.x, y: last.y })
  return wp
}

// ═══════════════════════════════════════════════
//  SNAP TARGET (port proximity during drawing)
// ═══════════════════════════════════════════════

const snapTarget = computed<{ nodeId: string; portId: string; world: Position } | null>(() => {
  if (!draw.active || !mousePos.value) return null
  const DIST = 18
  for (const node of nodes) {
    const ports = getExposedPorts(node.id)
    for (const port of ports) {
      if (node.id === draw.fromNodeId && port.id === draw.fromPortId) continue
      const w = { x: node.position.x + port.x, y: node.position.y + port.y }
      if (Math.hypot(w.x - mousePos.value.x, w.y - mousePos.value.y) < DIST) {
        return { nodeId: node.id, portId: port.id, world: w }
      }
    }
  }
  return null
})

// ═══════════════════════════════════════════════
//  RUBBER BAND
// ═══════════════════════════════════════════════

const lastAnchor = computed(() => {
  if (!draw.active) return { x: 0, y: 0 }
  return draw.waypoints.length > 0 ? draw.waypoints[draw.waypoints.length - 1] : draw.startPos
})

const rubberBand = computed(() => {
  if (!draw.active || !mousePos.value) return ''
  const anchor = lastAnchor.value
  const target = snapTarget.value ? snapTarget.value.world : mousePos.value

  if (drawingMode.value === 'ortho') {
    const { corner, point } = snapOrtho(target, anchor)
    const dest = snapTarget.value ? snapTarget.value.world : point
    if (corner) return `M ${anchor.x} ${anchor.y} L ${corner.x} ${corner.y} L ${dest.x} ${dest.y}`
    return `M ${anchor.x} ${anchor.y} L ${dest.x} ${dest.y}`
  }

  return `M ${anchor.x} ${anchor.y} L ${target.x} ${target.y}`
})

const instructions = computed(() => {
  if (!draw.active) return '<b>Click any port</b> to start drawing a pipe.'
  if (drawingMode.value === 'ortho') return '<b>Click</b> = H/V segment. <b>Click port</b> = finish. <b>Esc</b> = cancel.'
  return '<b>Click</b> = waypoint. <b>Click port</b> = finish. <b>Esc</b> = cancel.'
})

// ═══════════════════════════════════════════════
//  PATH BUILDER
// ═══════════════════════════════════════════════

function buildPath(conn: ConnData): string {
  const s = portWorldPos(conn.from.nodeId, conn.from.portId)
  const e = portWorldPos(conn.to.nodeId, conn.to.portId)
  return [s, ...conn.waypoints, e].map((p, i) => `${i ? 'L' : 'M'} ${p.x} ${p.y}`).join(' ')
}

// ═══════════════════════════════════════════════
//  SVG COORDINATES
// ═══════════════════════════════════════════════

function toSvg(e: MouseEvent): Position {
  if (!svgEl.value) return { x: 0, y: 0 }
  const pt = svgEl.value.createSVGPoint()
  pt.x = e.clientX; pt.y = e.clientY
  const ctm = svgEl.value.getScreenCTM()
  if (!ctm) return { x: 0, y: 0 }
  const s = pt.matrixTransform(ctm.inverse())
  return { x: s.x, y: s.y }
}

// ═══════════════════════════════════════════════
//  EVENT HANDLERS
// ═══════════════════════════════════════════════

function onPortClick(nodeId: string, portId: string) {
  if (!draw.active) {
    const pos = portWorldPos(nodeId, portId)
    draw.active = true
    draw.fromNodeId = nodeId
    draw.fromPortId = portId
    draw.startPos = { ...pos }
    draw.waypoints = []
    selectedConnId.value = null
    return
  }

  if (nodeId === draw.fromNodeId && portId === draw.fromPortId) return

  let wp = [...draw.waypoints]
  const targetPos = portWorldPos(nodeId, portId)

  if (drawingMode.value === 'ortho') {
    wp = alignToTargetPort(wp, draw.startPos, targetPos)
  }

  connections.push({
    id: `conn-${++connCtr}`,
    from: { nodeId: draw.fromNodeId, portId: draw.fromPortId },
    to: { nodeId, portId },
    waypoints: simplify(wp),
    props: { flowing: true, direction: 'forward' },
  })
  selectedConnId.value = `conn-${connCtr}`
  cancelDrawing()
}

function onPortEnter(nodeId: string, portId: string) {
  hoveredPort.value = { nodeId, portId }
}
function onPortLeave(_nodeId: string, _portId: string) {
  hoveredPort.value = null
}

function onCanvasClick(e: MouseEvent) {
  if (!draw.active) { selectedConnId.value = null; return }

  if (snapTarget.value) {
    onPortClick(snapTarget.value.nodeId, snapTarget.value.portId)
    return
  }

  const raw = toSvg(e)
  const anchor = lastAnchor.value

  if (drawingMode.value === 'ortho') {
    const { corner, point } = snapOrtho(raw, anchor)
    if (corner) draw.waypoints.push(corner)
    draw.waypoints.push(point)
  } else {
    draw.waypoints.push(snapGrid(raw))
  }
}

function onMouseMove(e: MouseEvent) {
  mousePos.value = toSvg(e)
  if (wpDrag.active) {
    const conn = connections.find(c => c.id === wpDrag.connId)
    if (conn?.waypoints[wpDrag.idx]) {
      conn.waypoints[wpDrag.idx] = snapGrid(mousePos.value)
    }
  }
}

function cancelDrawing() {
  draw.active = false
  draw.fromNodeId = ''
  draw.fromPortId = ''
  draw.waypoints = []
}

// ═══════════════════════════════════════════════
//  POST-DRAW EDITING
// ═══════════════════════════════════════════════

function startWpDrag(connId: string, idx: number, e: MouseEvent) {
  wpDrag.active = true; wpDrag.connId = connId; wpDrag.idx = idx
  e.preventDefault()
}

function onGlobalMouseUp() { wpDrag.active = false }

onMounted(() => {
  window.addEventListener('mouseup', onGlobalMouseUp)
  nextTick(() => svgEl.value?.focus())
})
onUnmounted(() => { window.removeEventListener('mouseup', onGlobalMouseUp) })

// ═══════════════════════════════════════════════
//  UTILITIES
// ═══════════════════════════════════════════════

function simplify(pts: Position[]): Position[] {
  if (pts.length <= 1) return pts
  const out = [pts[0]]
  for (let i = 1; i < pts.length - 1; i++) {
    const [a, b, c] = [pts[i - 1], pts[i], pts[i + 1]]
    if (Math.abs((b.x - a.x) * (c.y - a.y) - (c.x - a.x) * (b.y - a.y)) > 2) out.push(b)
  }
  out.push(pts[pts.length - 1])
  return out
}

function deleteConnection(id: string) {
  const i = connections.findIndex(c => c.id === id)
  if (i >= 0) connections.splice(i, 1)
  if (selectedConnId.value === id) selectedConnId.value = null
}

function resetAll() { connections.splice(0); selectedConnId.value = null; cancelDrawing() }

function exportJson() {
  const d = { version: 1, nodes: nodes.map(n => ({ ...n })), connections: connections.map(c => ({ ...c })) }
  const blob = new Blob([JSON.stringify(d, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a'); a.href = url; a.download = 'diagram.json'; a.click()
  URL.revokeObjectURL(url)
}
</script>

<style scoped>
.lab { display: flex; flex-direction: column; height: 100vh; background: #0e1117; color: #e6edf3; font-family: 'IBM Plex Sans', -apple-system, sans-serif; font-size: 13px; }
.lab-header { display: flex; align-items: center; gap: 10px; padding: 0 16px; height: 42px; background: #161b22; border-bottom: 1px solid #30363d; }
.lab-title { font-weight: 600; } .lab-sep { width: 1px; height: 18px; background: #30363d; } .lab-sub { font-size: 12px; color: #8b949e; }
.lab-body { display: flex; flex: 1; overflow: hidden; }
.controls { width: 240px; background: #161b22; border-right: 1px solid #30363d; overflow-y: auto; padding: 12px; flex-shrink: 0; }
.ctrl-section { margin-bottom: 14px; }
.ctrl-heading { font-size: 10px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.07em; color: #656d76; margin-bottom: 6px; }
.mode-btn { display: flex; align-items: flex-start; gap: 8px; width: 100%; padding: 8px 10px; margin-bottom: 4px; background: #1c2129; border: 1px solid #30363d; border-radius: 6px; color: #e6edf3; cursor: pointer; text-align: left; font-family: inherit; transition: all 0.12s; }
.mode-btn:hover { border-color: #58a6ff; } .mode-btn.active { border-color: #58a6ff; background: rgba(88,166,255,0.1); box-shadow: 0 0 0 1px #58a6ff; }
.mode-icon { font-size: 16px; } .mode-name { font-size: 12px; font-weight: 600; } .mode-desc { font-size: 10px; color: #8b949e; }
.ctrl-toggle { display: flex; align-items: center; gap: 6px; padding: 3px 0; cursor: pointer; font-size: 12px; color: #8b949e; }
.ctrl-toggle input { accent-color: #58a6ff; }
.instructions { font-size: 11px; color: #8b949e; line-height: 1.6; padding: 8px; background: #1c2129; border: 1px solid #30363d; border-radius: 6px; }
.instructions :deep(b) { color: #e6edf3; }
.conn-item { display: flex; align-items: center; gap: 4px; padding: 4px 6px; margin-bottom: 2px; border-radius: 4px; font-size: 10px; font-family: monospace; cursor: pointer; background: #1c2129; border: 1px solid transparent; }
.conn-item:hover { border-color: #30363d; } .conn-item.selected { border-color: #58a6ff; background: rgba(88,166,255,0.08); }
.cl { color: #8b949e; } .ca { color: #656d76; } .cw { color: #656d76; margin-left: auto; }
.cd { background: none; border: none; color: #656d76; font-size: 14px; cursor: pointer; padding: 0 2px; } .cd:hover { color: #f85149; }
.conn-empty { font-size: 11px; color: #656d76; padding: 6px 0; }
.action-btn { width: 100%; padding: 6px 10px; margin-bottom: 4px; background: #21262d; border: 1px solid #30363d; color: #e6edf3; font-size: 11px; font-family: inherit; border-radius: 4px; cursor: pointer; } .action-btn:hover { background: #282e36; }
.canvas-area { flex: 1; position: relative; overflow: hidden; }
.canvas-svg { width: 100%; height: 100%; display: block; outline: none; cursor: crosshair; }
.node-group { cursor: pointer; }
.wp-handle { cursor: grab; transition: all 0.1s; } .wp-handle:hover { r: 6; }
.pulse { animation: p1 1.5s ease-out infinite; } .pulse-green { animation: p2 1s ease-out infinite; }
@keyframes p1 { 0% { r: 6; opacity: 0.6; } 100% { r: 14; opacity: 0; } }
@keyframes p2 { 0% { r: 10; opacity: 0.7; } 100% { r: 18; opacity: 0; } }
.mode-badge { position: absolute; top: 10px; left: 10px; padding: 4px 10px; border-radius: 4px; font-size: 10px; font-weight: 600; letter-spacing: 0.06em; pointer-events: none; }
.mode-badge.click { background: rgba(88,166,255,0.15); color: #58a6ff; border: 1px solid rgba(88,166,255,0.3); }
.mode-badge.ortho { background: rgba(63,185,80,0.15); color: #3fb950; border: 1px solid rgba(63,185,80,0.3); }
.draw-status { position: absolute; bottom: 10px; left: 10px; right: 10px; padding: 6px 12px; background: #161b22; border: 1px solid #30363d; border-radius: 6px; font-size: 11px; color: #8b949e; display: flex; align-items: center; gap: 6px; }
.draw-status strong { color: #e6edf3; }
.dot { width: 6px; height: 6px; background: #58a6ff; border-radius: 50%; animation: blink 1.5s infinite; }
@keyframes blink { 0%,100% { opacity: 1; } 50% { opacity: 0.3; } }
.cancel-link { background: none; border: none; color: #f85149; font-family: inherit; font-size: 11px; cursor: pointer; text-decoration: underline; }
</style>
