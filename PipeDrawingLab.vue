<template>
  <!--
    Pipe Drawing Lab — Uses Your Real Library Components
    =====================================================
    Drop into: src/demo/PipeDrawingLab.vue
    
    Imports your actual ManualValve, CentrifugalPump, VerticalTank,
    PressureSensor, and ConnectionPipe. Zero changes to library needed.
    
    The editor adds ONLY interaction overlays on top:
      - Invisible port hit-targets (larger click area)
      - Waypoint drag handles
      - Preview rubber-band line
  -->
  <div class="lab">
    <header class="lab-header">
      <span class="lab-title">⬡ Pipe Drawing Lab</span>
      <span class="lab-sep" />
      <span class="lab-sub">Real library components · Three drawing modes</span>
    </header>

    <div class="lab-body">
      <aside class="controls">
        <div class="ctrl-section">
          <div class="ctrl-heading">Drawing Mode</div>
          <button v-for="m in modes" :key="m.id" class="mode-btn" :class="{ active: drawingMode === m.id }"
            @click="drawingMode = m.id; cancelDrawing()">
            <span class="mode-icon">{{ m.icon }}</span>
            <div class="mode-info">
              <span class="mode-name">{{ m.name }}</span>
              <span class="mode-desc">{{ m.desc }}</span>
            </div>
          </button>
        </div>
        <div class="ctrl-section">
          <div class="ctrl-heading">Options</div>
          <label class="ctrl-toggle"><input type="checkbox" v-model="snapToGrid" /><span>Snap to grid</span></label>
          <label class="ctrl-toggle"><input type="checkbox" v-model="showPortDots" /><span>Show ports (PortIndicator)</span></label>
          <label class="ctrl-toggle"><input type="checkbox" v-model="showGuides" /><span>Alignment guides</span></label>
        </div>
        <div class="ctrl-section">
          <div class="ctrl-heading">Instructions</div>
          <div class="instructions" v-html="currentInstructions" />
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
          <div v-if="!connections.length" class="conn-empty">Click a port to start drawing.</div>
        </div>
        <div class="ctrl-section">
          <button class="action-btn" @click="resetAll">Clear All</button>
          <button class="action-btn" @click="exportJson">Export JSON</button>
        </div>
      </aside>

      <div class="canvas-area">
        <svg ref="svgEl" class="canvas-svg" :viewBox="`0 0 ${canvasW} ${canvasH}`"
          @mousemove="handleMouseMove" @click="handleCanvasClick"
          @contextmenu.prevent="cancelDrawing" @keydown.escape="cancelDrawing" tabindex="0">
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

          <!-- Alignment guides -->
          <g v-if="showGuides && drawState.isDrawing && mousePos">
            <line :x1="drawState.lastPoint.x" y1="0" :x2="drawState.lastPoint.x" :y2="canvasH" stroke="rgba(88,166,255,0.1)" stroke-width="0.5" stroke-dasharray="4 4" />
            <line x1="0" :y1="drawState.lastPoint.y" :x2="canvasW" :y2="drawState.lastPoint.y" stroke="rgba(88,166,255,0.1)" stroke-width="0.5" stroke-dasharray="4 4" />
          </g>

          <!-- ═══ FINISHED CONNECTIONS ═══ -->
          <g v-for="conn in connections" :key="conn.id"
            :class="{ 'conn-sel': selectedConnId === conn.id }" @click.stop="selectedConnId = conn.id">
            <!-- Selection glow + hit area (editor overlay) -->
            <path v-if="selectedConnId === conn.id" :d="makePath(conn)" fill="none" stroke="rgba(88,166,255,0.25)" stroke-width="8" stroke-linecap="round" stroke-linejoin="round" />
            <path :d="makePath(conn)" fill="none" stroke="transparent" stroke-width="14" style="pointer-events:stroke;cursor:pointer" />
            <!-- YOUR LIBRARY: ConnectionPipe — used as-is -->
            <ConnectionPipe
              :start-position="getPortWorld(conn.from.nodeId, conn.from.portId)"
              :end-position="getPortWorld(conn.to.nodeId, conn.to.portId)"
              :waypoints="conn.waypoints"
              :flowing="conn.props.flowing"
              :direction="conn.props.direction"
            />
            <!-- Waypoint handles (editor overlay, shown when selected) -->
            <g v-if="selectedConnId === conn.id">
              <circle v-for="(wp, i) in conn.waypoints" :key="i" :cx="wp.x" :cy="wp.y" r="4"
                fill="#0e1117" stroke="#58a6ff" stroke-width="1.5" class="wp-handle"
                @mousedown.stop="startWpDrag(conn.id, i, $event)"
                @dblclick.stop="conn.waypoints.splice(i, 1)" />
            </g>
          </g>

          <!-- ═══ PREVIEW PIPE (drawing in progress) ═══ -->
          <g v-if="drawState.isDrawing">
            <!-- Already committed segments — use your ConnectionPipe -->
            <ConnectionPipe v-if="drawState.waypoints.length > 0"
              :start-position="drawState.startPortWorldPos"
              :end-position="drawState.waypoints[drawState.waypoints.length - 1]"
              :waypoints="drawState.waypoints.slice(0, -1)"
              :flowing="false"
            />
            <!-- Rubber-band to mouse (editor-only, dashed preview) -->
            <path v-if="rubberPath" :d="rubberPath" fill="none" stroke="#58a6ff" stroke-width="2"
              stroke-dasharray="6 4" stroke-linecap="round" stroke-linejoin="round" opacity="0.5" />
            <!-- Start pulse -->
            <circle :cx="drawState.startPortWorldPos.x" :cy="drawState.startPortWorldPos.y" r="6"
              fill="none" stroke="#58a6ff" stroke-width="1.5" class="pulse" />
          </g>

          <!-- ═══ COMPONENTS (your library, rendered as-is) ═══ -->
          <g v-for="node in nodes" :key="node.id">
            <!--
              Wrapper <g> positions the component on canvas.
              Inside: your real component + invisible port hit-targets overlaid on top.
              
              The component handles its own rendering, ports, labels, colors.
              We ONLY add larger invisible circles on top of each port
              so the user has a bigger click target during pipe drawing.
            -->
            <g :transform="`translate(${node.position.x}, ${node.position.y})`" class="node-group">

              <!-- YOUR LIBRARY COMPONENT (rendered by type) -->
              <ManualValve v-if="node.typeId === 'manual-valve'"
                :state="node.props.state" :alarm="node.props.alarm || 'none'"
                :label="node.label" :show-label="true" :show-ports="showPortDots" />

              <CentrifugalPump v-else-if="node.typeId === 'centrifugal-pump'"
                :state="node.props.state" :alarm="node.props.alarm || 'none'"
                :label="node.label" :show-label="true" :show-ports="showPortDots" />

              <VerticalTank v-else-if="node.typeId === 'vertical-tank'"
                :level="node.props.level" :alarm="node.props.alarm || 'none'"
                :label="node.label" :show-label="true" :show-ports="showPortDots" />

              <PressureSensor v-else-if="node.typeId === 'pressure-sensor'"
                :value="node.props.value" :alarm="node.props.alarm || 'none'"
                :label="node.label" :show-label="true" :show-ports="showPortDots" />

              <!-- EDITOR OVERLAY: port hit-targets (invisible, larger radius for easy clicking) -->
              <circle v-for="port in getNodePorts(node)" :key="port.id"
                :cx="port.x" :cy="port.y" r="8"
                fill="transparent" class="port-target"
                :class="{ highlighted: isPortHigh(node.id, port.id), snapping: isSnapTarget(node.id, port.id) }"
                @click.stop="handlePortClick(node.id, port.id)"
                @mouseenter="hovered = { n: node.id, p: port.id }"
                @mouseleave="hovered = null" />
            </g>
          </g>

          <!-- Snap indicator -->
          <circle v-if="snapInfo" :cx="snapInfo.world.x" :cy="snapInfo.world.y" r="10"
            fill="none" stroke="#3fb950" stroke-width="2" class="pulse-green" />
        </svg>

        <div class="mode-badge" :class="drawingMode">{{ drawingMode.toUpperCase() }}</div>
        <div v-if="drawState.isDrawing" class="draw-status">
          <span class="dot" /> Drawing from <strong>{{ drawState.fromNodeId }}:{{ drawState.fromPortId }}</strong>
          · {{ drawState.waypoints.length }}wp
          · <button class="cancel-link" @click="cancelDrawing">Cancel</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'

// ═══════════════════════════════════════════════
//  YOUR LIBRARY IMPORTS
//  Adjust paths to your project structure.
//  ZERO changes to these components are needed.
// ═══════════════════════════════════════════════

import ManualValve from '@/lib/components/PID/valves/ManualValve.vue'
import CentrifugalPump from '@/lib/components/PID/pumps/CentrifugalPump.vue'
import VerticalTank from '@/lib/components/PID/tanks/VerticalTank.vue'
import PressureSensor from '@/lib/components/PID/sensors/PressureSensor.vue'
import ConnectionPipe from '@/lib/components/PID/connectors/ConnectionPipe.vue'

import {
  calculateManualValvePorts,
  calculateTankPorts,
  calculateCirclePorts,
} from '@/lib/utils/positions'

import {
  MANUAL_VALVE_DEFAULT_DIMENSIONS,
  TANK_DEFAULT_DIMENSIONS,
} from '@/lib/constants'

import type { PortDefinition, Position } from '@/lib/types'

// ═══════════════════════════════════════════════
//  PORT LOOKUP
//  Bridges your calculate*Ports() with editor needs.
//  No library change — this lives in the editor.
// ═══════════════════════════════════════════════

// NOTE: If your pump/sensor have their own calculate*Ports functions,
// import and use those instead. These are reasonable defaults
// based on what I saw in your positions.ts.
const PUMP_RADIUS = 18
const SENSOR_RADIUS = 14

function getPortsForType(typeId: string): PortDefinition[] {
  switch (typeId) {
    case 'manual-valve':
      return calculateManualValvePorts(MANUAL_VALVE_DEFAULT_DIMENSIONS.width, MANUAL_VALVE_DEFAULT_DIMENSIONS.height)
    case 'vertical-tank':
      return calculateTankPorts(TANK_DEFAULT_DIMENSIONS.width, TANK_DEFAULT_DIMENSIONS.height)
    case 'centrifugal-pump':
      return calculateCirclePorts(PUMP_RADIUS)
    case 'pressure-sensor':
      return calculateCirclePorts(SENSOR_RADIUS)
    default:
      return []
  }
}

// ═══════════════════════════════════════════════
//  TYPES & STATE
// ═══════════════════════════════════════════════

interface NodeData { id: string; typeId: string; position: Position; label: string; props: Record<string, any> }
interface ConnData {
  id: string; from: { nodeId: string; portId: string }; to: { nodeId: string; portId: string }
  waypoints: Position[]; props: { flowing: boolean; direction: string }
}
type DrawMode = 'auto' | 'click' | 'ortho'

const canvasW = 900, canvasH = 500, gridSize = ref(10)
const snapToGrid = ref(true), showPortDots = ref(true), showGuides = ref(true)
const selectedConnId = ref<string | null>(null)
const hovered = ref<{ n: string; p: string } | null>(null)
const mousePos = ref<Position | null>(null)
const svgEl = ref<SVGSVGElement | null>(null)
const drawingMode = ref<DrawMode>('click')
let connCtr = 0

const drawState = reactive({
  isDrawing: false, fromNodeId: '', fromPortId: '',
  startPortWorldPos: { x: 0, y: 0 } as Position,
  waypoints: [] as Position[], lastPoint: { x: 0, y: 0 } as Position,
})
const wpDrag = reactive({ active: false, connId: '', idx: -1 })

const modes = [
  { id: 'auto' as DrawMode, icon: '⚡', name: 'Auto Route', desc: 'Port → Port. System picks route.' },
  { id: 'click' as DrawMode, icon: '✎', name: 'Free Draw', desc: 'Click to place waypoints freely.' },
  { id: 'ortho' as DrawMode, icon: '⊾', name: 'Ortho Draw', desc: 'Horizontal/Vertical segments only.' },
]

const nodes = reactive<NodeData[]>([
  { id: 'T-001', typeId: 'vertical-tank', position: { x: 50, y: 80 }, label: 'T-001', props: { level: 65, alarm: 'none' } },
  { id: 'V-001', typeId: 'manual-valve', position: { x: 230, y: 198 }, label: 'V-001', props: { state: 'open', alarm: 'none' } },
  { id: 'P-001', typeId: 'centrifugal-pump', position: { x: 370, y: 192 }, label: 'P-001', props: { state: 'running', alarm: 'none' } },
  { id: 'T-002', typeId: 'vertical-tank', position: { x: 540, y: 80 }, label: 'T-002', props: { level: 40, alarm: 'none' } },
  { id: 'V-002', typeId: 'manual-valve', position: { x: 710, y: 198 }, label: 'V-002', props: { state: 'closed', alarm: 'none' } },
  { id: 'PT-001', typeId: 'pressure-sensor', position: { x: 460, y: 50 }, label: 'PT-001', props: { value: 125.5, units: 'PSI', alarm: 'none' } },
])

const connections = reactive<ConnData[]>([])

// ═══════════════════════════════════════════════
//  PORT HELPERS
// ═══════════════════════════════════════════════

function getNodePorts(node: NodeData): PortDefinition[] { return getPortsForType(node.typeId) }

function getPortWorld(nodeId: string, portId: string): Position {
  const node = nodes.find(n => n.id === nodeId)
  if (!node) return { x: 0, y: 0 }
  const port = getPortsForType(node.typeId).find(p => p.id === portId)
  if (!port) return node.position
  return { x: node.position.x + port.x, y: node.position.y + port.y }
}

function getPortDir(nodeId: string, portId: string): string {
  if (portId.includes('left')) return 'left'
  if (portId.includes('right')) return 'right'
  if (portId.includes('top')) return 'up'
  if (portId.includes('bottom')) return 'down'
  return 'right'
}

function isPortHigh(nid: string, pid: string): boolean {
  return (hovered.value?.n === nid && hovered.value?.p === pid) ||
    (drawState.isDrawing && drawState.fromNodeId === nid && drawState.fromPortId === pid)
}
function isSnapTarget(nid: string, pid: string): boolean {
  return snapInfo.value?.nodeId === nid && snapInfo.value?.portId === pid
}

// ═══════════════════════════════════════════════
//  COMPUTED
// ═══════════════════════════════════════════════

const currentInstructions = computed(() => {
  if (!drawState.isDrawing) return '<b>Click any port</b> to start drawing a pipe.'
  if (drawingMode.value === 'auto') return '<b>Click target port</b> to complete. Esc to cancel.'
  if (drawingMode.value === 'click') return '<b>Click canvas</b> = waypoint. <b>Click port</b> = finish. Esc/right-click = cancel.'
  return '<b>Click canvas</b> = H/V waypoint. <b>Click port</b> = finish. Esc = cancel.'
})

const snapInfo = computed<{ nodeId: string; portId: string; world: Position } | null>(() => {
  if (!drawState.isDrawing || !mousePos.value) return null
  for (const node of nodes) {
    for (const port of getNodePorts(node)) {
      if (node.id === drawState.fromNodeId && port.id === drawState.fromPortId) continue
      const w = { x: node.position.x + port.x, y: node.position.y + port.y }
      if (Math.hypot(w.x - mousePos.value.x, w.y - mousePos.value.y) < 18) return { nodeId: node.id, portId: port.id, world: w }
    }
  }
  return null
})

const rubberPath = computed(() => {
  if (!drawState.isDrawing || !mousePos.value) return ''
  const from = drawState.waypoints.length > 0 ? drawState.waypoints[drawState.waypoints.length - 1] : drawState.startPortWorldPos
  const to = snapInfo.value ? snapInfo.value.world : mousePos.value
  if (drawingMode.value === 'ortho') return `M ${from.x} ${from.y} L ${to.x} ${from.y} L ${to.x} ${to.y}`
  return `M ${from.x} ${from.y} L ${to.x} ${to.y}`
})

function makePath(conn: ConnData): string {
  const s = getPortWorld(conn.from.nodeId, conn.from.portId), e = getPortWorld(conn.to.nodeId, conn.to.portId)
  return [s, ...conn.waypoints, e].map((p, i) => `${i ? 'L' : 'M'} ${p.x} ${p.y}`).join(' ')
}

// ═══════════════════════════════════════════════
//  AUTO ROUTING
// ═══════════════════════════════════════════════

function autoRoute(sp: Position, sd: string, ep: Position, ed: string): Position[] {
  const S = 25
  const s = ext(sp, sd, S), e = ext(ep, ed, S), mx = (s.x + e.x) / 2
  return simplify([s, { x: mx, y: s.y }, { x: mx, y: e.y }, e])
}
function ext(p: Position, d: string, n: number): Position {
  return d === 'right' ? { x: p.x + n, y: p.y } : d === 'left' ? { x: p.x - n, y: p.y } : d === 'down' ? { x: p.x, y: p.y + n } : { x: p.x, y: p.y - n }
}
function simplify(pts: Position[]): Position[] {
  if (pts.length <= 2) return pts
  const o = [pts[0]]
  for (let i = 1; i < pts.length - 1; i++) {
    const [a, b, c] = [pts[i - 1], pts[i], pts[i + 1]]
    if (Math.abs((b.x - a.x) * (c.y - a.y) - (c.x - a.x) * (b.y - a.y)) > 2) o.push(b)
  }
  o.push(pts[pts.length - 1]); return o
}

// ═══════════════════════════════════════════════
//  COORDINATE HELPERS
// ═══════════════════════════════════════════════

function toSvg(e: MouseEvent): Position {
  if (!svgEl.value) return { x: 0, y: 0 }
  const pt = svgEl.value.createSVGPoint(); pt.x = e.clientX; pt.y = e.clientY
  const ctm = svgEl.value.getScreenCTM(); if (!ctm) return { x: 0, y: 0 }
  const s = pt.matrixTransform(ctm.inverse()); return { x: s.x, y: s.y }
}
function snap(p: Position): Position {
  if (!snapToGrid.value) return p
  return { x: Math.round(p.x / gridSize.value) * gridSize.value, y: Math.round(p.y / gridSize.value) * gridSize.value }
}

// ═══════════════════════════════════════════════
//  DRAWING STATE MACHINE
// ═══════════════════════════════════════════════

function handlePortClick(nid: string, pid: string) {
  if (!drawState.isDrawing) {
    const w = getPortWorld(nid, pid)
    Object.assign(drawState, { isDrawing: true, fromNodeId: nid, fromPortId: pid, startPortWorldPos: { ...w }, lastPoint: { ...w }, waypoints: [] })
    selectedConnId.value = null; return
  }
  if (nid === drawState.fromNodeId && pid === drawState.fromPortId) return
  let wp = [...drawState.waypoints]
  if (drawingMode.value === 'auto') {
    wp = autoRoute(drawState.startPortWorldPos, getPortDir(drawState.fromNodeId, drawState.fromPortId), getPortWorld(nid, pid), getPortDir(nid, pid))
  } else if (drawingMode.value === 'ortho') {
    const last = wp.length ? wp[wp.length - 1] : drawState.startPortWorldPos, end = getPortWorld(nid, pid)
    if (Math.abs(last.x - end.x) > 2 && Math.abs(last.y - end.y) > 2) wp.push({ x: end.x, y: last.y })
  }
  const id = `conn-${++connCtr}`
  connections.push({ id, from: { nodeId: drawState.fromNodeId, portId: drawState.fromPortId }, to: { nodeId: nid, portId: pid }, waypoints: simplify(wp), props: { flowing: true, direction: 'forward' } })
  selectedConnId.value = id; cancelDrawing()
}

function handleCanvasClick(e: MouseEvent) {
  if (!drawState.isDrawing) return
  if (drawingMode.value === 'auto') return
  if (snapInfo.value) { handlePortClick(snapInfo.value.nodeId, snapInfo.value.portId); return }
  const pt = snap(toSvg(e))
  if (drawingMode.value === 'ortho') {
    const last = drawState.waypoints.length ? drawState.waypoints[drawState.waypoints.length - 1] : drawState.startPortWorldPos
    if (Math.abs(last.x - pt.x) > 2 && Math.abs(last.y - pt.y) > 2) drawState.waypoints.push({ x: pt.x, y: last.y })
  }
  drawState.waypoints.push({ ...pt }); drawState.lastPoint = { ...pt }
}

function handleMouseMove(e: MouseEvent) {
  mousePos.value = snap(toSvg(e))
  if (wpDrag.active) {
    const c = connections.find(c => c.id === wpDrag.connId)
    if (c?.waypoints[wpDrag.idx]) c.waypoints[wpDrag.idx] = { ...mousePos.value }
  }
}

function cancelDrawing() { Object.assign(drawState, { isDrawing: false, fromNodeId: '', fromPortId: '', waypoints: [] }) }

function startWpDrag(cid: string, idx: number, e: MouseEvent) { Object.assign(wpDrag, { active: true, connId: cid, idx }); e.preventDefault() }
function globalUp() { wpDrag.active = false }
onMounted(() => { window.addEventListener('mouseup', globalUp); svgEl.value?.focus() })
onUnmounted(() => { window.removeEventListener('mouseup', globalUp) })

function deleteConnection(id: string) { const i = connections.findIndex(c => c.id === id); if (i >= 0) connections.splice(i, 1); if (selectedConnId.value === id) selectedConnId.value = null }
function resetAll() { connections.splice(0); selectedConnId.value = null; cancelDrawing() }
function exportJson() {
  const d = { version: 1, nodes: nodes.map(n => ({ ...n })), connections: connections.map(c => ({ ...c })) }
  const b = new Blob([JSON.stringify(d, null, 2)], { type: 'application/json' })
  const u = URL.createObjectURL(b); const a = document.createElement('a'); a.href = u; a.download = 'diagram.json'; a.click(); URL.revokeObjectURL(u)
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
.mode-btn:hover { border-color: #58a6ff; }
.mode-btn.active { border-color: #58a6ff; background: rgba(88,166,255,0.1); box-shadow: 0 0 0 1px #58a6ff; }
.mode-icon { font-size: 16px; } .mode-info { display: flex; flex-direction: column; gap: 2px; } .mode-name { font-size: 12px; font-weight: 600; } .mode-desc { font-size: 10px; color: #8b949e; }
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
.port-target { cursor: pointer; transition: all 0.15s; } .port-target:hover, .port-target.highlighted { fill: rgba(88,166,255,0.2); stroke: #58a6ff; stroke-width: 2; } .port-target.snapping { fill: rgba(63,185,80,0.25); stroke: #3fb950; stroke-width: 2; }
.wp-handle { cursor: grab; transition: all 0.1s; } .wp-handle:hover { r: 6; }
.pulse { animation: p1 1.5s ease-out infinite; } .pulse-green { animation: p2 1s ease-out infinite; }
@keyframes p1 { 0% { r: 6; opacity: 0.6; } 100% { r: 14; opacity: 0; } }
@keyframes p2 { 0% { r: 10; opacity: 0.7; } 100% { r: 18; opacity: 0; } }
.mode-badge { position: absolute; top: 10px; left: 10px; padding: 4px 10px; border-radius: 4px; font-size: 10px; font-weight: 600; letter-spacing: 0.06em; pointer-events: none; }
.mode-badge.auto { background: rgba(188,140,255,0.15); color: #bc8cff; border: 1px solid rgba(188,140,255,0.3); }
.mode-badge.click { background: rgba(88,166,255,0.15); color: #58a6ff; border: 1px solid rgba(88,166,255,0.3); }
.mode-badge.ortho { background: rgba(63,185,80,0.15); color: #3fb950; border: 1px solid rgba(63,185,80,0.3); }
.draw-status { position: absolute; bottom: 10px; left: 10px; right: 10px; padding: 6px 12px; background: #161b22; border: 1px solid #30363d; border-radius: 6px; font-size: 11px; color: #8b949e; display: flex; align-items: center; gap: 6px; }
.draw-status strong { color: #e6edf3; }
.dot { width: 6px; height: 6px; background: #58a6ff; border-radius: 50%; animation: blink 1.5s infinite; }
@keyframes blink { 0%,100% { opacity: 1; } 50% { opacity: 0.3; } }
.cancel-link { background: none; border: none; color: #f85149; font-family: inherit; font-size: 11px; cursor: pointer; text-decoration: underline; }
</style>
