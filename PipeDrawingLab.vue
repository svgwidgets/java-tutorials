<template>
  <!--
    Pipe Drawing Modes — Interactive Prototype
    ===========================================
    Drop into your project: src/demo/PipeDrawingLab.vue
    
    Three modes to compare:
    1. AUTO — Click port A, click port B, system generates orthogonal route
    2. CLICK — Click port A, click canvas to place waypoints, click port B to finish
    3. ORTHO — Like CLICK but constrained to horizontal/vertical segments only
    
    This is a sandbox to find the right UX before building it into the real editor.
  -->
  <div class="lab">
    <header class="lab-header">
      <span class="lab-title">⬡ Pipe Drawing Lab</span>
      <span class="lab-sep" />
      <span class="lab-subtitle">Compare drawing modes — find what feels right</span>
    </header>

    <div class="lab-body">
      <!-- ── Controls ── -->
      <aside class="controls">
        <div class="ctrl-section">
          <div class="ctrl-heading">Drawing Mode</div>
          <button
            v-for="m in modes"
            :key="m.id"
            class="mode-btn"
            :class="{ active: drawingMode === m.id }"
            @click="drawingMode = m.id; cancelDrawing()"
          >
            <span class="mode-icon">{{ m.icon }}</span>
            <div class="mode-info">
              <span class="mode-name">{{ m.name }}</span>
              <span class="mode-desc">{{ m.desc }}</span>
            </div>
          </button>
        </div>

        <div class="ctrl-section">
          <div class="ctrl-heading">Options</div>
          <label class="ctrl-toggle">
            <input type="checkbox" v-model="snapToGrid" />
            <span>Snap to grid ({{ gridSize }}px)</span>
          </label>
          <label class="ctrl-toggle">
            <input type="checkbox" v-model="showPorts" />
            <span>Show port dots</span>
          </label>
          <label class="ctrl-toggle">
            <input type="checkbox" v-model="showGuides" />
            <span>Show alignment guides</span>
          </label>
        </div>

        <div class="ctrl-section">
          <div class="ctrl-heading">Instructions</div>
          <div class="instructions" v-html="currentInstructions" />
        </div>

        <div class="ctrl-section">
          <div class="ctrl-heading">Connections ({{ connections.length }})</div>
          <div
            v-for="conn in connections"
            :key="conn.id"
            class="conn-item"
            :class="{ selected: selectedConnId === conn.id }"
            @click="selectedConnId = conn.id"
          >
            <span class="conn-label">{{ conn.from.nodeId }}:{{ conn.from.portId }}</span>
            <span class="conn-arrow">→</span>
            <span class="conn-label">{{ conn.to.nodeId }}:{{ conn.to.portId }}</span>
            <span class="conn-wp">{{ conn.waypoints.length }}wp</span>
            <button class="conn-del" @click.stop="deleteConnection(conn.id)" title="Delete">×</button>
          </div>
          <div v-if="connections.length === 0" class="conn-empty">No connections yet. Draw one!</div>
        </div>

        <div class="ctrl-section">
          <div class="ctrl-heading">Actions</div>
          <button class="action-btn" @click="resetAll">Reset All</button>
          <button class="action-btn" @click="logJson">Log JSON</button>
        </div>
      </aside>

      <!-- ── Canvas ── -->
      <div class="canvas-area" ref="canvasArea">
        <svg
          ref="svgEl"
          class="canvas-svg"
          :viewBox="`0 0 ${canvasW} ${canvasH}`"
          @mousemove="handleMouseMove"
          @click="handleCanvasClick"
          @contextmenu.prevent="cancelDrawing"
          @keydown.escape="cancelDrawing"
          tabindex="0"
        >
          <defs>
            <pattern id="lab-grid" :width="gridSize" :height="gridSize" patternUnits="userSpaceOnUse">
              <path :d="`M ${gridSize} 0 L 0 0 0 ${gridSize}`" fill="none" stroke="rgba(255,255,255,0.04)" stroke-width="0.5" />
            </pattern>
            <pattern id="lab-grid-lg" :width="gridSize*5" :height="gridSize*5" patternUnits="userSpaceOnUse">
              <rect :width="gridSize*5" :height="gridSize*5" fill="url(#lab-grid)" />
              <path :d="`M ${gridSize*5} 0 L 0 0 0 ${gridSize*5}`" fill="none" stroke="rgba(255,255,255,0.07)" stroke-width="0.5" />
            </pattern>
          </defs>

          <!-- Grid -->
          <rect x="0" y="0" :width="canvasW" :height="canvasH" fill="url(#lab-grid-lg)" />

          <!-- Alignment guides -->
          <g v-if="showGuides && drawState.isDrawing && mousePos">
            <line :x1="drawState.lastPoint.x" :y1="0" :x2="drawState.lastPoint.x" :y2="canvasH"
                  stroke="rgba(88,166,255,0.12)" stroke-width="0.5" stroke-dasharray="4 4" />
            <line :x1="0" :y1="drawState.lastPoint.y" :x2="canvasW" :y2="drawState.lastPoint.y"
                  stroke="rgba(88,166,255,0.12)" stroke-width="0.5" stroke-dasharray="4 4" />
          </g>

          <!-- ═══ Finished connections ═══ -->
          <g v-for="conn in connections" :key="conn.id" class="conn-group"
             :class="{ 'conn-selected': selectedConnId === conn.id }"
             @click.stop="selectedConnId = conn.id">
            <!-- Hit area -->
            <path :d="buildPath(conn)" fill="none" stroke="transparent" stroke-width="12" />
            <!-- Selection glow -->
            <path v-if="selectedConnId === conn.id" :d="buildPath(conn)"
                  fill="none" stroke="rgba(88,166,255,0.25)" stroke-width="8" stroke-linecap="round" stroke-linejoin="round" />
            <!-- Body -->
            <path :d="buildPath(conn)" fill="none" stroke="#58a6ff" stroke-width="3"
                  stroke-linecap="round" stroke-linejoin="round" />
            <!-- Flow -->
            <path :d="buildPath(conn)" fill="none" stroke="#8dc8ff" stroke-width="2"
                  stroke-dasharray="8 6" stroke-linecap="round" class="flow-anim" />
            <!-- Waypoint handles (when selected) -->
            <g v-if="selectedConnId === conn.id">
              <circle
                v-for="(wp, i) in conn.waypoints"
                :key="i"
                :cx="wp.x" :cy="wp.y" r="4"
                fill="#0e1117" stroke="#58a6ff" stroke-width="1.5"
                class="wp-handle"
                @mousedown.stop="startDragWaypoint(conn.id, i, $event)"
              />
            </g>
          </g>

          <!-- ═══ In-progress pipe (preview) ═══ -->
          <g v-if="drawState.isDrawing">
            <!-- Committed segments -->
            <path v-if="previewCommittedPath" :d="previewCommittedPath"
                  fill="none" stroke="#58a6ff" stroke-width="3" stroke-linecap="round" stroke-linejoin="round" />
            <!-- Rubber-band segment (mouse follow) -->
            <path v-if="previewRubberPath" :d="previewRubberPath"
                  fill="none" stroke="#58a6ff" stroke-width="2" stroke-dasharray="6 4" stroke-linecap="round" stroke-linejoin="round" opacity="0.6" />
            <!-- Committed waypoint dots -->
            <circle v-for="(wp, i) in drawState.waypoints" :key="i"
                    :cx="wp.x" :cy="wp.y" r="3" fill="#58a6ff" opacity="0.8" />
            <!-- Start point indicator -->
            <circle :cx="drawState.startPortWorldPos.x" :cy="drawState.startPortWorldPos.y"
                    r="6" fill="none" stroke="#58a6ff" stroke-width="2" opacity="0.5">
              <animate attributeName="r" from="6" to="12" dur="1.5s" repeatCount="indefinite" />
              <animate attributeName="opacity" from="0.5" to="0" dur="1.5s" repeatCount="indefinite" />
            </circle>
          </g>

          <!-- ═══ Components (static positions) ═══ -->
          <g v-for="node in nodes" :key="node.id"
             :transform="`translate(${node.position.x}, ${node.position.y})`"
             class="node-group">

            <!-- ManualValve -->
            <g v-if="node.typeId === 'manual-valve'">
              <path d="M 0 0 L 20 12 L 0 24 Z" :fill="node.props.state === 'open' ? '#3fb950' : '#f85149'" stroke="#bbb" stroke-width="1.5" />
              <path d="M 40 0 L 20 12 L 40 24 Z" :fill="node.props.state === 'open' ? '#3fb950' : '#f85149'" stroke="#bbb" stroke-width="1.5" />
              <text x="20" y="-7" text-anchor="middle" class="n-label">{{ node.label }}</text>
            </g>

            <!-- Pump -->
            <g v-else-if="node.typeId === 'centrifugal-pump'">
              <circle cx="20" cy="20" r="18" :fill="node.props.state === 'running' ? '#3fb950' : '#555'" stroke="#bbb" stroke-width="1.5" />
              <g :class="{ rotating: node.props.state === 'running' }" style="transform-origin: 20px 20px">
                <line x1="20" y1="5" x2="20" y2="35" stroke="#fff" stroke-width="2" opacity="0.5" />
                <line x1="5" y1="20" x2="35" y2="20" stroke="#fff" stroke-width="2" opacity="0.5" />
              </g>
              <rect x="36" y="14" width="8" height="12" :fill="node.props.state === 'running' ? '#3fb950' : '#555'" stroke="#bbb" stroke-width="1" />
              <text x="22" y="-7" text-anchor="middle" class="n-label">{{ node.label }}</text>
            </g>

            <!-- Tank -->
            <g v-else-if="node.typeId === 'vertical-tank'">
              <rect x="0" y="0" width="60" height="120" rx="4" fill="none" stroke="#bbb" stroke-width="1.5" />
              <rect x="1.5" :y="120 - (node.props.level/100*117) - 1.5" width="57" :height="node.props.level/100*117" rx="2" fill="#2a6db5" opacity="0.6" />
              <text x="30" y="65" text-anchor="middle" class="tank-text">{{ node.props.level }}%</text>
              <text x="30" y="-7" text-anchor="middle" class="n-label">{{ node.label }}</text>
            </g>

            <!-- Sensor -->
            <g v-else-if="node.typeId === 'pressure-sensor'">
              <circle cx="16" cy="16" r="14" fill="#21262d" stroke="#3fb950" stroke-width="2" />
              <text x="16" y="14" text-anchor="middle" class="sensor-val">{{ node.props.value }}</text>
              <text x="16" y="23" text-anchor="middle" class="sensor-unit">{{ node.props.units }}</text>
              <text x="16" y="-7" text-anchor="middle" class="n-label">{{ node.label }}</text>
            </g>

            <!-- Port dots -->
            <g v-if="showPorts">
              <circle
                v-for="port in getNodePorts(node)"
                :key="port.id"
                :cx="port.position.x" :cy="port.position.y"
                r="4"
                :fill="isPortHighlighted(node.id, port.id) ? '#58a6ff' : '#0e1117'"
                :stroke="isPortHighlighted(node.id, port.id) ? '#58a6ff' : '#8b949e'"
                :stroke-width="isPortHighlighted(node.id, port.id) ? 2 : 1.5"
                class="port-dot"
                @click.stop="handlePortClick(node.id, port.id)"
                @mouseenter="hoveredPort = { nodeId: node.id, portId: port.id }"
                @mouseleave="hoveredPort = null"
              />
            </g>
          </g>

          <!-- Port snap indicator (when hovering valid target during drawing) -->
          <g v-if="snapTarget">
            <circle :cx="snapTarget.world.x" :cy="snapTarget.world.y" r="8"
                    fill="none" stroke="#3fb950" stroke-width="2" opacity="0.8">
              <animate attributeName="r" from="8" to="14" dur="1s" repeatCount="indefinite" />
              <animate attributeName="opacity" from="0.8" to="0" dur="1s" repeatCount="indefinite" />
            </circle>
          </g>
        </svg>

        <!-- Mode badge -->
        <div class="mode-badge" :class="drawingMode">
          {{ drawingMode.toUpperCase() }} MODE
        </div>

        <!-- Drawing status -->
        <div v-if="drawState.isDrawing" class="draw-status">
          <span class="draw-status-dot" />
          Drawing from <strong>{{ drawState.fromNodeId }}:{{ drawState.fromPortId }}</strong>
          · {{ drawState.waypoints.length }} waypoints
          · <em>{{ drawingMode === 'click' ? 'Click to place waypoints, click a port to finish' : drawingMode === 'ortho' ? 'Click to place H/V segments, click a port to finish' : 'Click target port' }}</em>
          · <button class="cancel-link" @click="cancelDrawing">Cancel (Esc)</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'

// ═══════════════════════════════════════════
//  TYPES
// ═══════════════════════════════════════════

interface Position { x: number; y: number }
interface PortDef { id: string; position: Position; direction: string }
interface NodeData {
  id: string; typeId: string; position: Position; label: string; props: Record<string, any>
}
interface Connection {
  id: string
  from: { nodeId: string; portId: string }
  to: { nodeId: string; portId: string }
  waypoints: Position[]
}

// ═══════════════════════════════════════════
//  PORT REGISTRY
// ═══════════════════════════════════════════

const PORT_DEFS: Record<string, PortDef[]> = {
  'manual-valve': [
    { id: 'inlet', position: { x: 0, y: 12 }, direction: 'left' },
    { id: 'outlet', position: { x: 40, y: 12 }, direction: 'right' },
  ],
  'centrifugal-pump': [
    { id: 'inlet', position: { x: 0, y: 20 }, direction: 'left' },
    { id: 'outlet', position: { x: 44, y: 20 }, direction: 'right' },
  ],
  'vertical-tank': [
    { id: 'top', position: { x: 30, y: 0 }, direction: 'up' },
    { id: 'bottom', position: { x: 30, y: 120 }, direction: 'down' },
    { id: 'side', position: { x: 60, y: 60 }, direction: 'right' },
  ],
  'pressure-sensor': [
    { id: 'bottom', position: { x: 16, y: 32 }, direction: 'down' },
  ],
}

function getNodePorts(node: NodeData): PortDef[] {
  return PORT_DEFS[node.typeId] || []
}

function getPortWorld(nodeId: string, portId: string): Position {
  const node = nodes.find(n => n.id === nodeId)
  if (!node) return { x: 0, y: 0 }
  const port = (PORT_DEFS[node.typeId] || []).find(p => p.id === portId)
  if (!port) return node.position
  return { x: node.position.x + port.position.x, y: node.position.y + port.position.y }
}

function getPortDirection(nodeId: string, portId: string): string {
  const node = nodes.find(n => n.id === nodeId)
  if (!node) return 'right'
  const port = (PORT_DEFS[node.typeId] || []).find(p => p.id === portId)
  return port?.direction || 'right'
}

// ═══════════════════════════════════════════
//  STATIC NODES (test layout)
// ═══════════════════════════════════════════

const nodes = reactive<NodeData[]>([
  { id: 'T-001', typeId: 'vertical-tank', position: { x: 80, y: 120 }, label: 'T-001', props: { level: 65 } },
  { id: 'V-001', typeId: 'manual-valve', position: { x: 240, y: 228 }, label: 'V-001', props: { state: 'open' } },
  { id: 'P-001', typeId: 'centrifugal-pump', position: { x: 380, y: 220 }, label: 'P-001', props: { state: 'running' } },
  { id: 'T-002', typeId: 'vertical-tank', position: { x: 560, y: 120 }, label: 'T-002', props: { level: 40 } },
  { id: 'V-002', typeId: 'manual-valve', position: { x: 700, y: 228 }, label: 'V-002', props: { state: 'closed' } },
  { id: 'PT-001', typeId: 'pressure-sensor', position: { x: 465, y: 80 }, label: 'PT-001', props: { value: 125.5, units: 'PSI' } },
])

// ═══════════════════════════════════════════
//  STATE
// ═══════════════════════════════════════════

const canvasW = 900
const canvasH = 500
const gridSize = ref(10)
const snapToGrid = ref(true)
const showPorts = ref(true)
const showGuides = ref(true)
const selectedConnId = ref<string | null>(null)
const hoveredPort = ref<{ nodeId: string; portId: string } | null>(null)
const mousePos = ref<Position | null>(null)
const svgEl = ref<SVGSVGElement | null>(null)
const canvasArea = ref<HTMLDivElement | null>(null)

const connections = reactive<Connection[]>([])

type DrawMode = 'auto' | 'click' | 'ortho'
const drawingMode = ref<DrawMode>('click')

const drawState = reactive({
  isDrawing: false,
  fromNodeId: '',
  fromPortId: '',
  startPortWorldPos: { x: 0, y: 0 } as Position,
  waypoints: [] as Position[],
  lastPoint: { x: 0, y: 0 } as Position,
})

// Waypoint dragging
const wpDrag = reactive({
  active: false,
  connId: '',
  wpIndex: -1,
  offset: { x: 0, y: 0 },
})

const modes = [
  { id: 'auto' as DrawMode, icon: '⚡', name: 'Auto Route', desc: 'Click port A → port B. System picks the route.' },
  { id: 'click' as DrawMode, icon: '✎', name: 'Click to Place', desc: 'Click canvas to drop waypoints freely.' },
  { id: 'ortho' as DrawMode, icon: '⊾', name: 'Ortho Draw', desc: 'Like Click, but locked to H/V segments.' },
]

let connCounter = 0
function nextConnId(): string { return `conn-${++connCounter}` }

// ═══════════════════════════════════════════
//  COMPUTED
// ═══════════════════════════════════════════

const currentInstructions = computed(() => {
  if (drawState.isDrawing) {
    if (drawingMode.value === 'auto') return '<b>Click a target port</b> to complete the connection.<br/>Right-click or Esc to cancel.'
    if (drawingMode.value === 'click') return '<b>Click on canvas</b> to place waypoints at any angle.<br/><b>Click a port</b> to finish.<br/>Right-click or Esc to cancel.'
    return '<b>Click on canvas</b> to place orthogonal waypoints (H/V only).<br/><b>Click a port</b> to finish.<br/>Right-click or Esc to cancel.'
  }
  return '<b>Click any port</b> (blue/yellow dot) to start drawing a pipe.'
})

const snapTarget = computed<{ nodeId: string; portId: string; world: Position } | null>(() => {
  if (!drawState.isDrawing || !mousePos.value) return null
  const SNAP_DIST = 15
  for (const node of nodes) {
    for (const port of getNodePorts(node)) {
      // Don't snap to start port
      if (node.id === drawState.fromNodeId && port.id === drawState.fromPortId) continue
      const world = { x: node.position.x + port.position.x, y: node.position.y + port.position.y }
      const dx = world.x - mousePos.value.x
      const dy = world.y - mousePos.value.y
      if (Math.sqrt(dx * dx + dy * dy) < SNAP_DIST) {
        return { nodeId: node.id, portId: port.id, world }
      }
    }
  }
  return null
})

const previewCommittedPath = computed(() => {
  if (!drawState.isDrawing) return ''
  const pts = [drawState.startPortWorldPos, ...drawState.waypoints]
  if (pts.length < 2) return ''
  return ptsToSvgPath(pts)
})

const previewRubberPath = computed(() => {
  if (!drawState.isDrawing || !mousePos.value) return ''
  const from = drawState.waypoints.length > 0
    ? drawState.waypoints[drawState.waypoints.length - 1]
    : drawState.startPortWorldPos
  const to = snapTarget.value ? snapTarget.value.world : mousePos.value

  if (drawingMode.value === 'ortho') {
    // Orthogonal: show L-shape preview (horizontal then vertical)
    const mid = { x: to.x, y: from.y }
    return ptsToSvgPath([from, mid, to])
  }
  return ptsToSvgPath([from, to])
})

// ═══════════════════════════════════════════
//  AUTO-ROUTING ALGORITHM
// ═══════════════════════════════════════════

function generateAutoRoute(
  startPos: Position, startDir: string,
  endPos: Position, endDir: string
): Position[] {
  const MIN_STUB = 25 // minimum extension from port before turning

  // Compute stub exit points (extend from port in its direction)
  const stubStart = extendInDirection(startPos, startDir, MIN_STUB)
  const stubEnd = extendInDirection(endPos, endDir, MIN_STUB)

  // If stubs are roughly aligned horizontally, simple Z-route
  if (Math.abs(stubStart.y - stubEnd.y) < 3) {
    // Already on same horizontal — straight through
    return maybeStubs(startPos, startDir, endPos, endDir, MIN_STUB, [])
  }

  // General case: route stub → horizontal jog → vertical jog → stub
  const midX = (stubStart.x + stubEnd.x) / 2
  const waypoints = [
    stubStart,
    { x: midX, y: stubStart.y },
    { x: midX, y: stubEnd.y },
    stubEnd,
  ]

  // Clean up: remove redundant collinear points
  return simplifyWaypoints(waypoints)
}

function extendInDirection(pos: Position, dir: string, dist: number): Position {
  switch (dir) {
    case 'right': return { x: pos.x + dist, y: pos.y }
    case 'left':  return { x: pos.x - dist, y: pos.y }
    case 'down':  return { x: pos.x, y: pos.y + dist }
    case 'up':    return { x: pos.x, y: pos.y - dist }
    default:      return { x: pos.x + dist, y: pos.y }
  }
}

function maybeStubs(
  startPos: Position, startDir: string,
  endPos: Position, endDir: string,
  stubLen: number, middle: Position[]
): Position[] {
  const pts: Position[] = []
  // Only add stubs if direction doesn't point toward the target naturally
  const dx = endPos.x - startPos.x
  if ((startDir === 'right' && dx < stubLen) || (startDir === 'left' && dx > -stubLen) ||
      startDir === 'up' || startDir === 'down') {
    pts.push(extendInDirection(startPos, startDir, stubLen))
  }
  pts.push(...middle)
  if ((endDir === 'left' && dx < stubLen) || (endDir === 'right' && dx > -stubLen) ||
      endDir === 'up' || endDir === 'down') {
    pts.push(extendInDirection(endPos, endDir, stubLen))
  }
  return simplifyWaypoints(pts)
}

function simplifyWaypoints(pts: Position[]): Position[] {
  if (pts.length <= 2) return pts
  const result = [pts[0]]
  for (let i = 1; i < pts.length - 1; i++) {
    if (!isCollinear(pts[i - 1], pts[i], pts[i + 1])) {
      result.push(pts[i])
    }
  }
  result.push(pts[pts.length - 1])
  return result
}

function isCollinear(a: Position, b: Position, c: Position): boolean {
  // Cross product ≈ 0 means collinear
  return Math.abs((b.x - a.x) * (c.y - a.y) - (c.x - a.x) * (b.y - a.y)) < 2
}

// ═══════════════════════════════════════════
//  COORDINATE HELPERS
// ═══════════════════════════════════════════

function svgPoint(e: MouseEvent): Position {
  if (!svgEl.value) return { x: 0, y: 0 }
  const pt = svgEl.value.createSVGPoint()
  pt.x = e.clientX
  pt.y = e.clientY
  const ctm = svgEl.value.getScreenCTM()
  if (!ctm) return { x: 0, y: 0 }
  const svgPt = pt.matrixTransform(ctm.inverse())
  return { x: svgPt.x, y: svgPt.y }
}

function snap(pos: Position): Position {
  if (!snapToGrid.value) return pos
  return {
    x: Math.round(pos.x / gridSize.value) * gridSize.value,
    y: Math.round(pos.y / gridSize.value) * gridSize.value,
  }
}

function ptsToSvgPath(pts: Position[]): string {
  if (pts.length < 2) return ''
  return `M ${pts[0].x} ${pts[0].y} ` + pts.slice(1).map(p => `L ${p.x} ${p.y}`).join(' ')
}

function buildPath(conn: Connection): string {
  const start = getPortWorld(conn.from.nodeId, conn.from.portId)
  const end = getPortWorld(conn.to.nodeId, conn.to.portId)
  return ptsToSvgPath([start, ...conn.waypoints, end])
}

// ═══════════════════════════════════════════
//  PORT / CONNECTION HELPERS
// ═══════════════════════════════════════════

function isPortHighlighted(nodeId: string, portId: string): boolean {
  if (hoveredPort.value?.nodeId === nodeId && hoveredPort.value?.portId === portId) return true
  if (drawState.isDrawing && drawState.fromNodeId === nodeId && drawState.fromPortId === portId) return true
  if (snapTarget.value?.nodeId === nodeId && snapTarget.value?.portId === portId) return true
  return false
}

function isPortOccupied(nodeId: string, portId: string): boolean {
  return connections.some(
    c => (c.from.nodeId === nodeId && c.from.portId === portId) ||
         (c.to.nodeId === nodeId && c.to.portId === portId)
  )
}

// ═══════════════════════════════════════════
//  DRAWING STATE MACHINE
// ═══════════════════════════════════════════

function handlePortClick(nodeId: string, portId: string) {
  if (!drawState.isDrawing) {
    // ── Start drawing ──
    const worldPos = getPortWorld(nodeId, portId)
    drawState.isDrawing = true
    drawState.fromNodeId = nodeId
    drawState.fromPortId = portId
    drawState.startPortWorldPos = { ...worldPos }
    drawState.lastPoint = { ...worldPos }
    drawState.waypoints = []
    selectedConnId.value = null
    return
  }

  // ── Finish drawing (clicked a target port) ──
  // Don't allow connecting to start port
  if (nodeId === drawState.fromNodeId && portId === drawState.fromPortId) return

  let waypoints = [...drawState.waypoints]

  if (drawingMode.value === 'auto') {
    // Auto mode: generate route now
    const startDir = getPortDirection(drawState.fromNodeId, drawState.fromPortId)
    const endDir = getPortDirection(nodeId, portId)
    waypoints = generateAutoRoute(
      drawState.startPortWorldPos, startDir,
      getPortWorld(nodeId, portId), endDir
    )
  } else if (drawingMode.value === 'ortho') {
    // Ortho: add final L-segment to reach port
    const lastPt = waypoints.length > 0 ? waypoints[waypoints.length - 1] : drawState.startPortWorldPos
    const endPt = getPortWorld(nodeId, portId)
    if (Math.abs(lastPt.x - endPt.x) > 2 && Math.abs(lastPt.y - endPt.y) > 2) {
      // Need a corner: horizontal then vertical
      waypoints.push({ x: endPt.x, y: lastPt.y })
    }
  }

  // Create connection
  const conn: Connection = {
    id: nextConnId(),
    from: { nodeId: drawState.fromNodeId, portId: drawState.fromPortId },
    to: { nodeId, portId },
    waypoints: simplifyWaypoints(waypoints),
  }
  connections.push(conn)
  selectedConnId.value = conn.id
  cancelDrawing()
}

function handleCanvasClick(e: MouseEvent) {
  if (!drawState.isDrawing) return
  if (drawingMode.value === 'auto') return // auto mode: only port clicks finish

  // If snapping to a port, treat as port click
  if (snapTarget.value) {
    handlePortClick(snapTarget.value.nodeId, snapTarget.value.portId)
    return
  }

  // Place a waypoint
  const raw = svgPoint(e)
  const pt = snap(raw)

  if (drawingMode.value === 'ortho') {
    // Orthogonal: insert corner point so segments are H/V only
    const lastPt = drawState.waypoints.length > 0
      ? drawState.waypoints[drawState.waypoints.length - 1]
      : drawState.startPortWorldPos

    // If not aligned, add a corner (horizontal first, then vertical)
    if (Math.abs(lastPt.x - pt.x) > 2 && Math.abs(lastPt.y - pt.y) > 2) {
      drawState.waypoints.push({ x: pt.x, y: lastPt.y })
    }
    drawState.waypoints.push({ ...pt })
    drawState.lastPoint = { ...pt }
  } else {
    // Free mode: place waypoint anywhere
    drawState.waypoints.push({ ...pt })
    drawState.lastPoint = { ...pt }
  }
}

function handleMouseMove(e: MouseEvent) {
  const pt = svgPoint(e)
  mousePos.value = snap(pt)

  // Waypoint dragging
  if (wpDrag.active) {
    const conn = connections.find(c => c.id === wpDrag.connId)
    if (conn && conn.waypoints[wpDrag.wpIndex]) {
      const snapped = snap(pt)
      conn.waypoints[wpDrag.wpIndex] = { x: snapped.x, y: snapped.y }
    }
  }
}

function cancelDrawing() {
  drawState.isDrawing = false
  drawState.fromNodeId = ''
  drawState.fromPortId = ''
  drawState.waypoints = []
}

// ═══════════════════════════════════════════
//  WAYPOINT DRAGGING (post-draw editing)
// ═══════════════════════════════════════════

function startDragWaypoint(connId: string, wpIndex: number, e: MouseEvent) {
  wpDrag.active = true
  wpDrag.connId = connId
  wpDrag.wpIndex = wpIndex
  e.preventDefault()
}

function handleGlobalMouseUp() {
  wpDrag.active = false
}

onMounted(() => {
  window.addEventListener('mouseup', handleGlobalMouseUp)
  // Focus SVG for keyboard events
  svgEl.value?.focus()
})
onUnmounted(() => {
  window.removeEventListener('mouseup', handleGlobalMouseUp)
})

// ═══════════════════════════════════════════
//  ACTIONS
// ═══════════════════════════════════════════

function deleteConnection(id: string) {
  const idx = connections.findIndex(c => c.id === id)
  if (idx >= 0) connections.splice(idx, 1)
  if (selectedConnId.value === id) selectedConnId.value = null
}

function resetAll() {
  connections.splice(0, connections.length)
  selectedConnId.value = null
  cancelDrawing()
}

function logJson() {
  console.log(JSON.stringify({
    nodes: nodes.map(n => ({ id: n.id, typeId: n.typeId, position: n.position })),
    connections: connections.map(c => ({
      id: c.id, from: c.from, to: c.to,
      waypoints: c.waypoints,
    })),
  }, null, 2))
  alert('Logged to console — open DevTools to see.')
}
</script>

<style scoped>
.lab {
  display: flex; flex-direction: column; height: 100vh; width: 100vw;
  background: #0e1117; color: #e6edf3;
  font-family: 'IBM Plex Sans', -apple-system, BlinkMacSystemFont, sans-serif; font-size: 13px;
}
.lab-header {
  display: flex; align-items: center; gap: 10px;
  padding: 0 16px; height: 42px; background: #161b22; border-bottom: 1px solid #30363d;
}
.lab-title { font-weight: 600; font-size: 13px; }
.lab-sep { width: 1px; height: 18px; background: #30363d; }
.lab-subtitle { font-size: 12px; color: #8b949e; }
.lab-body { display: flex; flex: 1; overflow: hidden; }

/* ── Controls ── */
.controls {
  width: 240px; background: #161b22; border-right: 1px solid #30363d;
  overflow-y: auto; padding: 12px; flex-shrink: 0;
}
.ctrl-section { margin-bottom: 16px; }
.ctrl-heading {
  font-size: 10px; font-weight: 600; text-transform: uppercase;
  letter-spacing: 0.07em; color: #656d76; margin-bottom: 6px;
}

.mode-btn {
  display: flex; align-items: flex-start; gap: 8px; width: 100%;
  padding: 8px 10px; margin-bottom: 4px;
  background: #1c2129; border: 1px solid #30363d; border-radius: 6px;
  color: #e6edf3; cursor: pointer; transition: all 0.12s; text-align: left;
  font-family: inherit;
}
.mode-btn:hover { border-color: #58a6ff; background: #21262d; }
.mode-btn.active { border-color: #58a6ff; background: rgba(88,166,255,0.1); box-shadow: 0 0 0 1px #58a6ff; }
.mode-icon { font-size: 16px; line-height: 1; margin-top: 1px; }
.mode-info { display: flex; flex-direction: column; gap: 2px; }
.mode-name { font-size: 12px; font-weight: 600; }
.mode-desc { font-size: 10px; color: #8b949e; line-height: 1.3; }

.ctrl-toggle {
  display: flex; align-items: center; gap: 6px; padding: 4px 0; cursor: pointer;
  font-size: 12px; color: #8b949e;
}
.ctrl-toggle input { accent-color: #58a6ff; }

.instructions {
  font-size: 11px; color: #8b949e; line-height: 1.6;
  padding: 8px; background: #1c2129; border: 1px solid #30363d; border-radius: 6px;
}
.instructions :deep(b) { color: #e6edf3; }
.instructions :deep(br) { display: block; margin: 4px 0; }

.conn-item {
  display: flex; align-items: center; gap: 4px;
  padding: 4px 6px; margin-bottom: 2px; border-radius: 4px;
  font-size: 10px; font-family: 'IBM Plex Mono', monospace; cursor: pointer;
  background: #1c2129; border: 1px solid transparent;
}
.conn-item:hover { border-color: #30363d; }
.conn-item.selected { border-color: #58a6ff; background: rgba(88,166,255,0.08); }
.conn-label { color: #8b949e; }
.conn-arrow { color: #656d76; }
.conn-wp { color: #656d76; margin-left: auto; }
.conn-del {
  background: none; border: none; color: #656d76; font-size: 14px; cursor: pointer;
  padding: 0 2px; line-height: 1;
}
.conn-del:hover { color: #f85149; }
.conn-empty { font-size: 11px; color: #656d76; padding: 8px 0; }

.action-btn {
  width: 100%; padding: 6px 10px; margin-bottom: 4px;
  background: #21262d; border: 1px solid #30363d; color: #e6edf3;
  font-size: 11px; font-family: inherit; border-radius: 4px; cursor: pointer;
}
.action-btn:hover { background: #282e36; border-color: #3d444d; }

/* ── Canvas ── */
.canvas-area { flex: 1; position: relative; overflow: hidden; }
.canvas-svg { width: 100%; height: 100%; display: block; outline: none; cursor: crosshair; }

.n-label { font-size: 10px; font-weight: 600; fill: #e6edf3; font-family: 'IBM Plex Sans', sans-serif; }
.tank-text { font-size: 13px; font-weight: 700; fill: #e6edf3; font-family: 'IBM Plex Mono', monospace; }
.sensor-val { font-size: 9px; font-weight: 600; fill: #e6edf3; font-family: 'IBM Plex Mono', monospace; }
.sensor-unit { font-size: 7px; fill: #8b949e; font-family: 'IBM Plex Mono', monospace; }

.port-dot { cursor: pointer; transition: all 0.12s; }
.port-dot:hover { r: 6; }

.node-group { cursor: pointer; }
.node-group:hover { filter: brightness(1.1); }

.wp-handle { cursor: grab; transition: all 0.1s; }
.wp-handle:hover { r: 6; stroke-width: 2; }

.flow-anim { animation: flowAnim 1.2s linear infinite; }
@keyframes flowAnim { from { stroke-dashoffset: 0; } to { stroke-dashoffset: -28; } }

.rotating { animation: rotate 2s linear infinite; }
@keyframes rotate { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }

.mode-badge {
  position: absolute; top: 10px; left: 10px;
  padding: 4px 10px; border-radius: 4px; font-size: 10px; font-weight: 600;
  letter-spacing: 0.06em; pointer-events: none;
}
.mode-badge.auto { background: rgba(188,140,255,0.15); color: #bc8cff; border: 1px solid rgba(188,140,255,0.3); }
.mode-badge.click { background: rgba(88,166,255,0.15); color: #58a6ff; border: 1px solid rgba(88,166,255,0.3); }
.mode-badge.ortho { background: rgba(63,185,80,0.15); color: #3fb950; border: 1px solid rgba(63,185,80,0.3); }

.draw-status {
  position: absolute; bottom: 10px; left: 10px; right: 10px;
  padding: 6px 12px; background: #161b22; border: 1px solid #30363d;
  border-radius: 6px; font-size: 11px; color: #8b949e;
  display: flex; align-items: center; gap: 6px;
}
.draw-status strong { color: #e6edf3; font-weight: 500; }
.draw-status em { color: #656d76; font-style: normal; }
.draw-status-dot {
  width: 6px; height: 6px; background: #58a6ff; border-radius: 50%;
  animation: pulse 1.5s infinite;
}
@keyframes pulse { 0%,100% { opacity: 1; } 50% { opacity: 0.3; } }

.cancel-link {
  background: none; border: none; color: #f85149; font-family: inherit;
  font-size: 11px; cursor: pointer; padding: 0; text-decoration: underline;
}
</style>
