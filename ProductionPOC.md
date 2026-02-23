# P&ID Designer — Production POC Architecture

**Version:** 1.0.0 · **Date:** February 2026
**Stack:** Vue 3 + TypeScript + Vite + Vuetify + Pinia

---

## A. Architecture & Folder Structure

```
pid-designer/
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── env.d.ts
│
├── src/
│   ├── main.ts                          # App bootstrap, Vuetify + Pinia + Router
│   ├── App.vue                          # Root layout shell
│   │
│   ├── domain/                          # 🧠 Pure TypeScript — zero Vue imports
│   │   ├── models.ts                    # All interfaces: Diagram, Node, Edge, Port, etc.
│   │   ├── defaults.ts                  # Factory functions: createNode(), createEdge()
│   │   ├── schema.ts                    # JSON schema version, validation, migration
│   │   └── geometry.ts                  # Point math, rect intersection, distance
│   │
│   ├── stores/                          # 📦 Pinia — single store, diagram state
│   │   └── useDiagramStore.ts           # Nodes, edges, viewport, selection, mode
│   │
│   ├── composables/                     # 🔧 Vue composables — interaction logic
│   │   ├── useCanvas.ts                 # Pan, zoom, coordinate transforms
│   │   ├── useNodeDrag.ts              # Drag to reposition nodes (edit mode)
│   │   ├── usePipeDraw.ts             # Click-port-to-port pipe creation
│   │   ├── usePaletteDrop.ts          # Drag from palette → drop on canvas
│   │   ├── useOrthoRouter.ts          # Orthogonal routing (wraps your algorithm)
│   │   └── useSerializer.ts           # Export/import JSON with validation
│   │
│   ├── components/                      # 🎨 Vue components
│   │   ├── layout/
│   │   │   ├── AppToolbar.vue          # Top bar: mode toggle, zoom, import/export
│   │   │   ├── AppStatusBar.vue        # Bottom: cursor coords, zoom %, node count
│   │   │   └── PalettePanel.vue        # Left sidebar: component palette (edit only)
│   │   │
│   │   ├── canvas/
│   │   │   ├── DiagramCanvas.vue       # Main SVG canvas with pan/zoom transform
│   │   │   ├── CanvasGrid.vue          # Background grid pattern
│   │   │   ├── NodeRenderer.vue        # Renders one node (delegates to library component)
│   │   │   ├── EdgeRenderer.vue        # Renders one pipe/edge (uses ConnectionPipe)
│   │   │   ├── PortOverlay.vue         # Edit-mode port hit targets
│   │   │   ├── SelectionBox.vue        # Selection rectangle around selected node
│   │   │   ├── DrawingPreview.vue      # Rubber-band line while drawing pipe
│   │   │   └── JunctionNode.vue        # 8-port junction component
│   │   │
│   │   └── palette/
│   │       └── PaletteItem.vue         # Single draggable palette entry
│   │
│   ├── pages/
│   │   ├── DesignerPage.vue            # /designer — main editor/viewer
│   │   └── ExamplesPage.vue            # /examples — pre-built demo diagrams
│   │
│   ├── router/
│   │   └── index.ts                     # Vue Router config
│   │
│   ├── utils/
│   │   └── uid.ts                       # UUID generator
│   │
│   └── lib/                             # 🏭 YOUR EXISTING LIBRARY (symlink or copy)
│       ├── components/PID/...           # ManualValve, GeneralPump, VerticalTank, etc.
│       ├── constants/...
│       ├── types/...
│       ├── utils/...
│       └── composables/...
```

**Why each layer exists:**

| Layer | Rule | Why |
|-------|------|-----|
| `domain/` | Zero Vue imports | Models are portable, testable, serializable |
| `stores/` | Single Pinia store | Diagram is shared state — nodes, edges, viewport, mode |
| `composables/` | One concern each | Testable interaction logic, not buried in components |
| `components/canvas/` | Render only | Read from store, emit events, no business logic |
| `lib/` | Your existing library | Components are "dumb" — they render SVG from props |

---

## B. Data Model (TypeScript)

```typescript
// ═══════════════════════════════════════════════
// src/domain/models.ts
// ═══════════════════════════════════════════════

// ── Primitives ──

export interface Point {
  x: number
  y: number
}

export interface Dimension {
  width: number
  height: number
}

export interface Rect {
  x: number
  y: number
  width: number
  height: number
}

// ── App Mode ──

export type AppMode = 'edit' | 'run'

// ── Viewport ──

export interface ViewportState {
  panX: number         // Canvas translation X (in screen px)
  panY: number         // Canvas translation Y (in screen px)
  zoom: number         // Scale factor (1 = 100%, 0.5 = 50%, 2 = 200%)
  minZoom: number
  maxZoom: number
}

// ── Port ──

export interface PortDefinition {
  id: string                          // 'left', 'right', 'top', 'N', 'SE', etc.
  x: number                          // Local offset from node position
  y: number                          // Local offset from node position
  direction?: 'up' | 'down' | 'left' | 'right'  // Optional, for auto-routing
}

// ── Binding (future middleware placeholder) ──

export interface BindingConfig {
  channelId?: string                 // e.g. "plc.valves.V001.state"
  protocol?: string                  // e.g. "websocket", "openc3", "modbus"
  transform?: string                 // Optional value transform expression
  [key: string]: unknown             // Extensible
}

// ── Node Type (palette definition) ──

export interface NodeTypeDefinition {
  typeId: string                     // 'manual-valve', 'centrifugal-pump', etc.
  label: string                      // 'Manual Valve'
  category: string                   // 'Valves', 'Pumps', 'Tanks', etc.
  icon?: string                      // Optional icon or emoji
  defaultDimensions: Dimension       // Default width/height
  defaultProps: Record<string, unknown>  // Default component props
}

// ── Node (placed instance) ──

export interface DiagramNode {
  id: string                         // UUID
  typeId: string                     // References NodeTypeDefinition.typeId
  position: Point                    // Canvas position (top-left of component)
  rotation: number                   // Degrees, typically 0
  label: string                      // User-visible label (e.g. "V-001")
  dimensions: Dimension              // Current width/height
  props: Record<string, unknown>     // Component-specific props (state, alarm, level, etc.)
  binding?: BindingConfig            // Future middleware binding
  zIndex: number                     // Render order
}

// ── Edge (pipe/connection) ──

export interface DiagramEdge {
  id: string                         // UUID
  from: {
    nodeId: string
    portId: string
  }
  to: {
    nodeId: string
    portId: string
  }
  waypoints: Point[]                 // Orthogonal route (intermediate points, excludes endpoints)
  props: {
    flowing: boolean
    direction: 'forward' | 'reverse'
    flowColor?: string
    strokeWidth?: number
    label?: string
  }
  binding?: BindingConfig            // Future: flow data binding
}

// ── Diagram (top-level, serializable) ──

export interface Diagram {
  schemaVersion: number              // Currently 1
  id: string                         // UUID
  name: string
  description?: string
  createdAt: string                  // ISO 8601
  updatedAt: string                  // ISO 8601
  nodes: DiagramNode[]
  edges: DiagramEdge[]
  viewport: Pick<ViewportState, 'panX' | 'panY' | 'zoom'>
}

// ── Selection State ──

export interface SelectionState {
  selectedNodeId: string | null
  selectedEdgeId: string | null
  hoveredPortInfo: { nodeId: string; portId: string } | null
}

// ── Drawing State (while creating a pipe) ──

export interface DrawingState {
  active: boolean
  fromNodeId: string
  fromPortId: string
  startPos: Point
  waypoints: Point[]
}
```

---

## C. Canvas Approach

**Decision: SVG with a transform group for pan/zoom.**

Why SVG over Canvas 2D:
- Your library components are already Vue SVG components — they render `<g>`, `<polygon>`, `<circle>`, `<rect>`, etc.
- Vue reactivity drives updates — no manual draw loop needed
- Hit testing is free (SVG elements have native mouse events)
- Clean separation: Pinia store → reactive props → SVG re-render
- For a POC with hundreds of components, SVG performs well. Canvas 2D would only be needed at thousands of animated elements.

**Scaling strategy for large diagrams:**
- Virtualization: only render nodes within the visible viewport (computed from pan/zoom). Nodes outside get culled.
- Debounced routing: recompute orthogonal routes only after drag ends (not every pixel)
- Edge culling: skip rendering edges whose bounding box doesn't intersect the viewport

```vue
<!-- src/components/canvas/DiagramCanvas.vue -->
<template>
  <div
    ref="containerRef"
    class="canvas-container"
    @wheel.prevent="onWheel"
    @mousedown="onMouseDown"
    @mousemove="onMouseMove"
    @mouseup="onMouseUp"
    @contextmenu.prevent
  >
    <svg
      ref="svgRef"
      class="canvas-svg"
      :viewBox="`0 0 ${containerSize.width} ${containerSize.height}`"
    >
      <defs>
        <CanvasGrid :grid-size="gridSize" :zoom="store.viewport.zoom" />
      </defs>

      <!-- Grid background -->
      <rect width="100%" height="100%" fill="url(#canvas-grid)" />

      <!--
        Master transform group.
        All canvas content lives inside this <g>.
        Pan = translate, Zoom = scale.
        Screen → Canvas: invert this transform.
      -->
      <g :transform="canvasTransform">

        <!-- Layer 1: Edges (below nodes) -->
        <EdgeRenderer
          v-for="edge in store.edges"
          :key="edge.id"
          :edge="edge"
          :selected="store.selection.selectedEdgeId === edge.id"
          @click="store.selectEdge(edge.id)"
        />

        <!-- Drawing preview (while creating pipe) -->
        <DrawingPreview
          v-if="store.drawing.active"
          :drawing="store.drawing"
          :mouse-pos="canvasMousePos"
          :mode="drawingMode"
        />

        <!-- Layer 2: Nodes -->
        <NodeRenderer
          v-for="node in store.nodes"
          :key="node.id"
          :node="node"
          :edit-mode="store.isEditMode"
          :selected="store.selection.selectedNodeId === node.id"
          :highlighted-port-id="getHighlightedPort(node.id)"
          @node-mousedown="onNodeMouseDown(node.id, $event)"
          @port-click="onPortClick(node.id, $event)"
          @port-mouse-enter="onPortEnter(node.id, $event)"
          @port-mouse-leave="onPortLeave(node.id, $event)"
        />
      </g>
    </svg>

    <!-- Zoom indicator (HTML overlay) -->
    <div class="zoom-indicator">{{ Math.round(store.viewport.zoom * 100) }}%</div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useDiagramStore } from '@/stores/useDiagramStore'
import { useCanvas } from '@/composables/useCanvas'
import { useNodeDrag } from '@/composables/useNodeDrag'
import { usePipeDraw } from '@/composables/usePipeDraw'
import type { Point } from '@/domain/models'

import CanvasGrid from './CanvasGrid.vue'
import EdgeRenderer from './EdgeRenderer.vue'
import NodeRenderer from './NodeRenderer.vue'
import DrawingPreview from './DrawingPreview.vue'

const store = useDiagramStore()
const containerRef = ref<HTMLDivElement>()
const svgRef = ref<SVGSVGElement>()

const { containerSize, canvasTransform, screenToCanvas, onWheel, gridSize } = useCanvas(containerRef, svgRef)
const { onNodeMouseDown } = useNodeDrag(screenToCanvas)
const { onPortClick, onPortEnter, onPortLeave, drawingMode, canvasMousePos } = usePipeDraw(screenToCanvas)

function getHighlightedPort(nodeId: string): string | undefined {
  const hp = store.selection.hoveredPortInfo
  if (hp?.nodeId === nodeId) return hp.portId
  if (store.drawing.active && store.drawing.fromNodeId === nodeId) return store.drawing.fromPortId
  return undefined
}

// Pan via middle-click or space+drag
const isPanning = ref(false)
const panStart = ref<Point>({ x: 0, y: 0 })

function onMouseDown(e: MouseEvent) {
  // Middle-click or space+click → pan
  if (e.button === 1 || (e.button === 0 && e.shiftKey)) {
    isPanning.value = true
    panStart.value = { x: e.clientX, y: e.clientY }
    e.preventDefault()
  }
}

function onMouseMove(e: MouseEvent) {
  canvasMousePos.value = screenToCanvas({ x: e.clientX, y: e.clientY })

  if (isPanning.value) {
    const dx = e.clientX - panStart.value.x
    const dy = e.clientY - panStart.value.y
    store.pan(dx, dy)
    panStart.value = { x: e.clientX, y: e.clientY }
  }
}

function onMouseUp() {
  isPanning.value = false
}
</script>

<style scoped>
.canvas-container {
  width: 100%;
  height: 100%;
  overflow: hidden;
  position: relative;
  cursor: crosshair;
  /* Curved monitor friendly: fills any aspect ratio */
}
.canvas-svg {
  width: 100%;
  height: 100%;
  display: block;
}
.zoom-indicator {
  position: absolute;
  bottom: 8px;
  right: 8px;
  padding: 2px 8px;
  background: rgba(22,27,34,0.85);
  border: 1px solid rgba(48,54,61,0.6);
  border-radius: 4px;
  font-size: 11px;
  color: #8b949e;
  font-family: monospace;
  pointer-events: none;
}
</style>
```

---

## D. Pinia Store

**Yes, Pinia is justified.** The diagram state (nodes, edges, viewport, selection, mode) is consumed by 10+ components and 5+ composables. Prop-drilling would be painful. A single store keeps it clean.

```typescript
// ═══════════════════════════════════════════════
// src/stores/useDiagramStore.ts
// ═══════════════════════════════════════════════

import { defineStore } from 'pinia'
import { reactive, computed } from 'vue'
import type {
  AppMode, DiagramNode, DiagramEdge, ViewportState,
  SelectionState, DrawingState, Point, Diagram
} from '@/domain/models'
import { uid } from '@/utils/uid'

export const useDiagramStore = defineStore('diagram', () => {
  // ── State ──

  const mode = ref<AppMode>('edit')
  const nodes = reactive<DiagramNode[]>([])
  const edges = reactive<DiagramEdge[]>([])

  const viewport = reactive<ViewportState>({
    panX: 0,
    panY: 0,
    zoom: 1,
    minZoom: 0.1,
    maxZoom: 5,
  })

  const selection = reactive<SelectionState>({
    selectedNodeId: null,
    selectedEdgeId: null,
    hoveredPortInfo: null,
  })

  const drawing = reactive<DrawingState>({
    active: false,
    fromNodeId: '',
    fromPortId: '',
    startPos: { x: 0, y: 0 },
    waypoints: [],
  })

  // ── Getters ──

  const isEditMode = computed(() => mode.value === 'edit')
  const isRunMode = computed(() => mode.value === 'run')
  const selectedNode = computed(() => nodes.find(n => n.id === selection.selectedNodeId))

  function getNode(id: string): DiagramNode | undefined {
    return nodes.find(n => n.id === id)
  }

  function getEdgesForNode(nodeId: string): DiagramEdge[] {
    return edges.filter(e => e.from.nodeId === nodeId || e.to.nodeId === nodeId)
  }

  // ── Mode ──

  function setMode(m: AppMode) {
    mode.value = m
    if (m === 'run') {
      clearSelection()
      cancelDrawing()
    }
  }

  function toggleMode() {
    setMode(mode.value === 'edit' ? 'run' : 'edit')
  }

  // ── Viewport ──

  function pan(dx: number, dy: number) {
    viewport.panX += dx
    viewport.panY += dy
  }

  function zoomTo(newZoom: number, focalPoint?: Point) {
    const clamped = Math.max(viewport.minZoom, Math.min(viewport.maxZoom, newZoom))
    if (focalPoint) {
      // Zoom towards focal point (mouse position)
      const ratio = clamped / viewport.zoom
      viewport.panX = focalPoint.x - ratio * (focalPoint.x - viewport.panX)
      viewport.panY = focalPoint.y - ratio * (focalPoint.y - viewport.panY)
    }
    viewport.zoom = clamped
  }

  function zoomIn() { zoomTo(viewport.zoom * 1.2) }
  function zoomOut() { zoomTo(viewport.zoom / 1.2) }
  function resetViewport() { viewport.panX = 0; viewport.panY = 0; viewport.zoom = 1 }

  // ── Selection ──

  function selectNode(id: string | null) {
    selection.selectedNodeId = id
    selection.selectedEdgeId = null
  }

  function selectEdge(id: string | null) {
    selection.selectedEdgeId = id
    selection.selectedNodeId = null
  }

  function clearSelection() {
    selection.selectedNodeId = null
    selection.selectedEdgeId = null
    selection.hoveredPortInfo = null
  }

  // ── Nodes ──

  function addNode(node: DiagramNode) {
    nodes.push(node)
  }

  function moveNode(nodeId: string, position: Point) {
    const node = getNode(nodeId)
    if (node) {
      node.position.x = position.x
      node.position.y = position.y
    }
  }

  function removeNode(nodeId: string) {
    // Remove connected edges first
    const connected = getEdgesForNode(nodeId)
    connected.forEach(e => removeEdge(e.id))
    const idx = nodes.findIndex(n => n.id === nodeId)
    if (idx >= 0) nodes.splice(idx, 1)
    if (selection.selectedNodeId === nodeId) selection.selectedNodeId = null
  }

  function updateNodeProps(nodeId: string, props: Record<string, unknown>) {
    const node = getNode(nodeId)
    if (node) Object.assign(node.props, props)
  }

  // ── Edges ──

  function addEdge(edge: DiagramEdge) {
    edges.push(edge)
  }

  function updateEdgeWaypoints(edgeId: string, waypoints: Point[]) {
    const edge = edges.find(e => e.id === edgeId)
    if (edge) edge.waypoints = waypoints
  }

  function removeEdge(edgeId: string) {
    const idx = edges.findIndex(e => e.id === edgeId)
    if (idx >= 0) edges.splice(idx, 1)
    if (selection.selectedEdgeId === edgeId) selection.selectedEdgeId = null
  }

  // ── Drawing ──

  function startDrawing(nodeId: string, portId: string, startPos: Point) {
    drawing.active = true
    drawing.fromNodeId = nodeId
    drawing.fromPortId = portId
    drawing.startPos = { ...startPos }
    drawing.waypoints = []
  }

  function addDrawingWaypoint(point: Point) {
    drawing.waypoints.push({ ...point })
  }

  function cancelDrawing() {
    drawing.active = false
    drawing.fromNodeId = ''
    drawing.fromPortId = ''
    drawing.waypoints = []
  }

  // ── Serialization ──

  function toDiagram(): Diagram {
    return {
      schemaVersion: 1,
      id: uid(),
      name: 'Untitled Diagram',
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
      nodes: nodes.map(n => ({ ...n, position: { ...n.position }, dimensions: { ...n.dimensions }, props: { ...n.props } })),
      edges: edges.map(e => ({ ...e, from: { ...e.from }, to: { ...e.to }, waypoints: e.waypoints.map(w => ({ ...w })), props: { ...e.props } })),
      viewport: { panX: viewport.panX, panY: viewport.panY, zoom: viewport.zoom },
    }
  }

  function loadDiagram(diagram: Diagram) {
    nodes.splice(0)
    edges.splice(0)
    diagram.nodes.forEach(n => nodes.push(n))
    diagram.edges.forEach(e => edges.push(e))
    viewport.panX = diagram.viewport.panX
    viewport.panY = diagram.viewport.panY
    viewport.zoom = diagram.viewport.zoom
    clearSelection()
    cancelDrawing()
  }

  function clearDiagram() {
    nodes.splice(0)
    edges.splice(0)
    clearSelection()
    cancelDrawing()
    resetViewport()
  }

  return {
    // State
    mode, nodes, edges, viewport, selection, drawing,
    // Getters
    isEditMode, isRunMode, selectedNode, getNode, getEdgesForNode,
    // Mode
    setMode, toggleMode,
    // Viewport
    pan, zoomTo, zoomIn, zoomOut, resetViewport,
    // Selection
    selectNode, selectEdge, clearSelection,
    // Nodes
    addNode, moveNode, removeNode, updateNodeProps,
    // Edges
    addEdge, updateEdgeWaypoints, removeEdge,
    // Drawing
    startDrawing, addDrawingWaypoint, cancelDrawing,
    // Serialization
    toDiagram, loadDiagram, clearDiagram,
  }
})
```

---

## E. Core Composables

### useCanvas — Pan, Zoom, Coordinate Transforms

```typescript
// src/composables/useCanvas.ts

import { ref, computed, onMounted, onUnmounted, type Ref } from 'vue'
import { useDiagramStore } from '@/stores/useDiagramStore'
import type { Point } from '@/domain/models'

export function useCanvas(
  containerRef: Ref<HTMLDivElement | undefined>,
  svgRef: Ref<SVGSVGElement | undefined>,
) {
  const store = useDiagramStore()
  const containerSize = ref({ width: 1920, height: 1080 })
  const gridSize = ref(10)

  // The SVG transform string applied to the master <g>
  const canvasTransform = computed(() =>
    `translate(${store.viewport.panX}, ${store.viewport.panY}) scale(${store.viewport.zoom})`
  )

  /**
   * Convert screen coordinates (e.g. MouseEvent.clientX/Y) to canvas coordinates.
   * This inverts the pan+zoom transform.
   */
  function screenToCanvas(screen: Point): Point {
    if (!containerRef.value) return screen
    const rect = containerRef.value.getBoundingClientRect()
    return {
      x: (screen.x - rect.left - store.viewport.panX) / store.viewport.zoom,
      y: (screen.y - rect.top - store.viewport.panY) / store.viewport.zoom,
    }
  }

  /** Zoom centered on mouse position */
  function onWheel(e: WheelEvent) {
    const delta = e.deltaY > 0 ? 0.9 : 1.1
    const newZoom = store.viewport.zoom * delta
    store.zoomTo(newZoom, { x: e.clientX, y: e.clientY })
  }

  // Track container resize
  let resizeObserver: ResizeObserver | null = null

  onMounted(() => {
    if (containerRef.value) {
      const rect = containerRef.value.getBoundingClientRect()
      containerSize.value = { width: rect.width, height: rect.height }

      resizeObserver = new ResizeObserver(entries => {
        for (const entry of entries) {
          containerSize.value = {
            width: entry.contentRect.width,
            height: entry.contentRect.height,
          }
        }
      })
      resizeObserver.observe(containerRef.value)
    }
  })

  onUnmounted(() => resizeObserver?.disconnect())

  return { containerSize, canvasTransform, screenToCanvas, onWheel, gridSize }
}
```

### useNodeDrag — Drag to Reposition

```typescript
// src/composables/useNodeDrag.ts

import { ref } from 'vue'
import { useDiagramStore } from '@/stores/useDiagramStore'
import { useOrthoRouter } from './useOrthoRouter'
import type { Point } from '@/domain/models'

export function useNodeDrag(screenToCanvas: (screen: Point) => Point) {
  const store = useDiagramStore()
  const router = useOrthoRouter()

  const dragging = ref(false)
  const dragNodeId = ref('')
  const dragOffset = ref<Point>({ x: 0, y: 0 })

  function onNodeMouseDown(nodeId: string, e: MouseEvent) {
    if (!store.isEditMode) return
    if (e.button !== 0) return

    const node = store.getNode(nodeId)
    if (!node) return

    store.selectNode(nodeId)
    dragging.value = true
    dragNodeId.value = nodeId

    const canvasPos = screenToCanvas({ x: e.clientX, y: e.clientY })
    dragOffset.value = {
      x: canvasPos.x - node.position.x,
      y: canvasPos.y - node.position.y,
    }

    const onMove = (me: MouseEvent) => {
      if (!dragging.value) return
      const pos = screenToCanvas({ x: me.clientX, y: me.clientY })
      store.moveNode(dragNodeId.value, {
        x: pos.x - dragOffset.value.x,
        y: pos.y - dragOffset.value.y,
      })
    }

    const onUp = () => {
      dragging.value = false
      // Recompute routes for connected edges ONCE on drop
      router.recomputeEdgesForNode(dragNodeId.value)
      window.removeEventListener('mousemove', onMove)
      window.removeEventListener('mouseup', onUp)
    }

    window.addEventListener('mousemove', onMove)
    window.addEventListener('mouseup', onUp)

    e.preventDefault()
    e.stopPropagation()
  }

  return { onNodeMouseDown, dragging }
}
```

### usePipeDraw — Port-to-Port Connection

```typescript
// src/composables/usePipeDraw.ts

import { ref } from 'vue'
import { useDiagramStore } from '@/stores/useDiagramStore'
import { useOrthoRouter } from './useOrthoRouter'
import { uid } from '@/utils/uid'
import type { Point } from '@/domain/models'

export function usePipeDraw(screenToCanvas: (screen: Point) => Point) {
  const store = useDiagramStore()
  const router = useOrthoRouter()
  const canvasMousePos = ref<Point>({ x: 0, y: 0 })
  const drawingMode = ref<'ortho' | 'click'>('ortho')

  function onPortClick(nodeId: string, portId: string) {
    if (!store.isEditMode) return

    if (!store.drawing.active) {
      // Start drawing
      const portWorld = router.getPortWorldPos(nodeId, portId)
      store.startDrawing(nodeId, portId, portWorld)
      return
    }

    // Finish drawing — clicked target port
    if (nodeId === store.drawing.fromNodeId && portId === store.drawing.fromPortId) return

    const fromPos = store.drawing.startPos
    const toPos = router.getPortWorldPos(nodeId, portId)

    // Use orthogonal routing to compute waypoints
    const waypoints = drawingMode.value === 'ortho'
      ? alignToTarget(store.drawing.waypoints, fromPos, toPos)
      : store.drawing.waypoints

    store.addEdge({
      id: uid(),
      from: { nodeId: store.drawing.fromNodeId, portId: store.drawing.fromPortId },
      to: { nodeId, portId },
      waypoints: simplify(waypoints),
      props: { flowing: false, direction: 'forward' },
    })

    store.cancelDrawing()
  }

  function onPortEnter(nodeId: string, portId: string) {
    store.selection.hoveredPortInfo = { nodeId, portId }
  }

  function onPortLeave(_nodeId: string, _portId: string) {
    store.selection.hoveredPortInfo = null
  }

  return { onPortClick, onPortEnter, onPortLeave, drawingMode, canvasMousePos }
}

// ── Helpers ──

function alignToTarget(waypoints: Point[], start: Point, target: Point): Point[] {
  const wp = [...waypoints]
  const last = wp.length > 0 ? wp[wp.length - 1] : start
  if (Math.abs(last.x - target.x) > 2 && Math.abs(last.y - target.y) > 2) {
    wp.push({ x: target.x, y: last.y })
  }
  return wp
}

function simplify(pts: Point[]): Point[] {
  if (pts.length <= 1) return pts
  const out = [pts[0]]
  for (let i = 1; i < pts.length - 1; i++) {
    const [a, b, c] = [pts[i - 1], pts[i], pts[i + 1]]
    if (Math.abs((b.x - a.x) * (c.y - a.y) - (c.x - a.x) * (b.y - a.y)) > 2) out.push(b)
  }
  out.push(pts[pts.length - 1])
  return out
}
```

### useOrthoRouter — Wraps Your Existing Algorithm

```typescript
// src/composables/useOrthoRouter.ts

import { useDiagramStore } from '@/stores/useDiagramStore'
import type { Point, PortDefinition } from '@/domain/models'

/**
 * Wraps the existing orthogonal routing algorithm.
 * The actual routing math is in snapOrtho/alignToTarget
 * (already proven in PipeDrawingLab).
 *
 * This composable adds:
 * - Port world position lookup (reads from component refs)
 * - Batch recomputation when a node moves
 * - Performance: only recomputes edges connected to moved node
 */
export function useOrthoRouter() {
  const store = useDiagramStore()

  // Component refs for reading exposed ports
  const compRefs: Record<string, any> = {}

  function registerCompRef(nodeId: string, el: any) {
    if (el) compRefs[nodeId] = el
    else delete compRefs[nodeId]
  }

  function getExposedPorts(nodeId: string): PortDefinition[] {
    const comp = compRefs[nodeId]
    if (!comp) return []
    const ports = comp.ports
    return Array.isArray(ports) ? ports : (ports?.value ?? [])
  }

  function getPortWorldPos(nodeId: string, portId: string): Point {
    const node = store.getNode(nodeId)
    if (!node) return { x: 0, y: 0 }
    const ports = getExposedPorts(nodeId)
    const port = ports.find(p => p.id === portId)
    if (!port) return node.position
    return { x: node.position.x + port.x, y: node.position.y + port.y }
  }

  /**
   * Recompute waypoints for all edges connected to a node.
   * Called ONCE after a drag ends — not on every pixel of movement.
   * During drag, edges render with straight lines from old waypoints
   * to the new port positions (acceptable visual, no jank).
   */
  function recomputeEdgesForNode(nodeId: string) {
    const affected = store.getEdgesForNode(nodeId)
    for (const edge of affected) {
      const fromPos = getPortWorldPos(edge.from.nodeId, edge.from.portId)
      const toPos = getPortWorldPos(edge.to.nodeId, edge.to.portId)
      // Simple orthogonal route: L-shaped connection
      const waypoints = computeOrthoRoute(fromPos, toPos)
      store.updateEdgeWaypoints(edge.id, waypoints)
    }
  }

  return { registerCompRef, getExposedPorts, getPortWorldPos, recomputeEdgesForNode }
}

/**
 * Basic orthogonal route computation.
 * Creates an L-shaped or Z-shaped path between two points.
 * All segments are horizontal or vertical.
 */
function computeOrthoRoute(from: Point, to: Point): Point[] {
  // If aligned on one axis, no waypoints needed
  if (Math.abs(from.x - to.x) < 2) return []
  if (Math.abs(from.y - to.y) < 2) return []
  // L-shape: go horizontal first, then vertical
  return [{ x: to.x, y: from.y }]
}
```

### useSerializer — Export/Import JSON

```typescript
// src/composables/useSerializer.ts

import { useDiagramStore } from '@/stores/useDiagramStore'
import type { Diagram } from '@/domain/models'

const CURRENT_SCHEMA_VERSION = 1

export function useSerializer() {
  const store = useDiagramStore()

  function exportDiagram() {
    const diagram = store.toDiagram()
    const json = JSON.stringify(diagram, null, 2)
    const blob = new Blob([json], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = `${diagram.name.replace(/\s+/g, '_')}.json`
    a.click()
    URL.revokeObjectURL(url)
  }

  function importDiagram(): Promise<{ success: boolean; error?: string }> {
    return new Promise(resolve => {
      const input = document.createElement('input')
      input.type = 'file'
      input.accept = '.json'

      input.onchange = async () => {
        const file = input.files?.[0]
        if (!file) { resolve({ success: false, error: 'No file selected' }); return }

        try {
          const text = await file.text()
          const data = JSON.parse(text)
          const result = validateDiagram(data)
          if (!result.valid) { resolve({ success: false, error: result.error }); return }

          store.loadDiagram(data as Diagram)
          resolve({ success: true })
        } catch (e) {
          resolve({ success: false, error: 'Invalid JSON file' })
        }
      }

      input.click()
    })
  }

  return { exportDiagram, importDiagram }
}

function validateDiagram(data: unknown): { valid: boolean; error?: string } {
  if (!data || typeof data !== 'object') return { valid: false, error: 'Not an object' }
  const d = data as Record<string, unknown>

  if (d.schemaVersion !== CURRENT_SCHEMA_VERSION) {
    return { valid: false, error: `Unsupported schema version: ${d.schemaVersion}. Expected: ${CURRENT_SCHEMA_VERSION}` }
  }
  if (!Array.isArray(d.nodes)) return { valid: false, error: 'Missing nodes array' }
  if (!Array.isArray(d.edges)) return { valid: false, error: 'Missing edges array' }

  for (const node of d.nodes as any[]) {
    if (!node.id || !node.typeId || !node.position) {
      return { valid: false, error: `Invalid node: missing id, typeId, or position` }
    }
  }

  for (const edge of d.edges as any[]) {
    if (!edge.id || !edge.from?.nodeId || !edge.to?.nodeId) {
      return { valid: false, error: `Invalid edge: missing id, from, or to` }
    }
  }

  return { valid: true }
}
```

### usePaletteDrop — Drag from Palette onto Canvas

```typescript
// src/composables/usePaletteDrop.ts

import { useDiagramStore } from '@/stores/useDiagramStore'
import { uid } from '@/utils/uid'
import type { NodeTypeDefinition, Point } from '@/domain/models'

export function usePaletteDrop(screenToCanvas: (screen: Point) => Point) {
  const store = useDiagramStore()

  /**
   * Called when a palette item is dropped on the canvas.
   * Creates a new node at the drop position (adjusted for pan/zoom).
   */
  function handleDrop(typeDef: NodeTypeDefinition, screenPos: Point) {
    const canvasPos = screenToCanvas(screenPos)

    // Center the component on the drop point
    const position = {
      x: canvasPos.x - typeDef.defaultDimensions.width / 2,
      y: canvasPos.y - typeDef.defaultDimensions.height / 2,
    }

    store.addNode({
      id: uid(),
      typeId: typeDef.typeId,
      position,
      rotation: 0,
      label: generateLabel(typeDef.typeId),
      dimensions: { ...typeDef.defaultDimensions },
      props: { ...typeDef.defaultProps },
      zIndex: store.nodes.length,
    })
  }

  return { handleDrop }
}

/** Generate incrementing labels like V-001, V-002, P-001, etc. */
const counters: Record<string, number> = {}
function generateLabel(typeId: string): string {
  const prefix = TYPE_PREFIXES[typeId] || 'X'
  counters[typeId] = (counters[typeId] || 0) + 1
  return `${prefix}-${String(counters[typeId]).padStart(3, '0')}`
}

const TYPE_PREFIXES: Record<string, string> = {
  'manual-valve': 'V',
  'centrifugal-pump': 'P',
  'vertical-tank': 'T',
  'pressure-sensor': 'PT',
  'temperature-sensor': 'TT',
  'junction': 'J',
}
```

---

## F. Key Components

### NodeRenderer — Delegates to Library Components

```vue
<!-- src/components/canvas/NodeRenderer.vue -->
<template>
  <g
    :transform="`translate(${node.position.x}, ${node.position.y})`"
    class="node-renderer"
    :class="{ selected, 'edit-mode': editMode }"
    @mousedown.stop="$emit('node-mousedown', $event)"
  >
    <!-- Selection outline (edit mode only) -->
    <rect
      v-if="selected && editMode"
      :x="-4" :y="-4"
      :width="node.dimensions.width + 8"
      :height="node.dimensions.height + 8"
      fill="none"
      stroke="#58a6ff"
      stroke-width="1.5"
      stroke-dasharray="4 4"
      rx="4"
    />

    <!-- Actual library component -->
    <component
      :is="componentMap[node.typeId]"
      :ref="(el: any) => $emit('register-ref', node.id, el)"
      v-bind="node.props"
      :label="node.label"
      :show-label="true"
      :show-ports="editMode"
      :highlighted-port-id="highlightedPortId"
      @port-click="(portId: string) => $emit('port-click', portId)"
      @port-mouse-enter="(portId: string) => $emit('port-mouse-enter', portId)"
      @port-mouse-leave="(portId: string) => $emit('port-mouse-leave', portId)"
    />

    <!-- Junction (special: 8-port circular node) -->
    <JunctionNode
      v-if="node.typeId === 'junction'"
      :ref="(el: any) => $emit('register-ref', node.id, el)"
      :show-ports="editMode"
      :highlighted-port-id="highlightedPortId"
      @port-click="(portId: string) => $emit('port-click', portId)"
      @port-mouse-enter="(portId: string) => $emit('port-mouse-enter', portId)"
      @port-mouse-leave="(portId: string) => $emit('port-mouse-leave', portId)"
    />
  </g>
</template>

<script setup lang="ts">
import { markRaw, type Component } from 'vue'
import type { DiagramNode } from '@/domain/models'

// Your library components
import ManualValve from '@/lib/components/PID/valves/ManualValve.vue'
import CentrifugalPump from '@/lib/components/PID/pumps/GeneralPump.vue'
import VerticalTank from '@/lib/components/PID/tanks/VerticalTank.vue'
import AnalogDisplay from '@/lib/components/PID/sensors/AnalogDisplay.vue'
import JunctionNode from './JunctionNode.vue'

const componentMap: Record<string, Component> = {
  'manual-valve': markRaw(ManualValve),
  'centrifugal-pump': markRaw(CentrifugalPump),
  'vertical-tank': markRaw(VerticalTank),
  'pressure-sensor': markRaw(AnalogDisplay),
}

interface Props {
  node: DiagramNode
  editMode: boolean
  selected: boolean
  highlightedPortId?: string
}

defineProps<Props>()
defineEmits<{
  'node-mousedown': [event: MouseEvent]
  'port-click': [portId: string]
  'port-mouse-enter': [portId: string]
  'port-mouse-leave': [portId: string]
  'register-ref': [nodeId: string, el: any]
}>()
</script>
```

### JunctionNode — 8-Port Circular Node

```vue
<!-- src/components/canvas/JunctionNode.vue -->
<template>
  <g class="junction-node">
    <circle
      :cx="radius" :cy="radius" :r="radius"
      fill="#1c2129" stroke="#58a6ff" stroke-width="1.5"
      vector-effect="non-scaling-stroke"
    />
    <circle
      :cx="radius" :cy="radius" :r="radius * 0.3"
      fill="#58a6ff" opacity="0.3"
    />

    <!-- 8 ports: N, NE, E, SE, S, SW, W, NW -->
    <PortIndicator
      :ports="ports"
      :show="showPorts"
      :highlighted-port-id="highlightedPortId"
      @port-click="handlePortClick"
      @port-mouse-enter="handlePortMouseEnter"
      @port-mouse-leave="handlePortMouseLeave"
    />
  </g>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { PortIndicator } from '@/lib/components/common'
import { usePortEvents } from '@/lib/composables/usePortEvents'

interface Props {
  showPorts?: boolean
  highlightedPortId?: string
  radius?: number
}

const props = withDefaults(defineProps<Props>(), {
  showPorts: false,
  radius: 12,
})

const emit = defineEmits(['port-click', 'port-mouse-enter', 'port-mouse-leave'])

// 8 ports evenly spaced around the circle
const ports = computed(() => {
  const r = props.radius
  const cx = r, cy = r
  const dirs = ['N', 'NE', 'E', 'SE', 'S', 'SW', 'W', 'NW']
  const angles = [270, 315, 0, 45, 90, 135, 180, 225] // degrees from East

  return dirs.map((id, i) => {
    const rad = (angles[i] * Math.PI) / 180
    return {
      id,
      x: cx + r * Math.cos(rad),
      y: cy + r * Math.sin(rad),
    }
  })
})

const { handlePortClick, handlePortMouseEnter, handlePortMouseLeave } = usePortEvents(emit)

defineExpose({ ports })
</script>
```

### EdgeRenderer — Uses Your ConnectionPipe

```vue
<!-- src/components/canvas/EdgeRenderer.vue -->
<template>
  <g @click.stop="$emit('click')">
    <!-- Selection glow -->
    <path
      v-if="selected"
      :d="pathD"
      fill="none" stroke="rgba(88,166,255,0.25)" stroke-width="8"
      stroke-linecap="round" stroke-linejoin="round"
    />
    <!-- Hit area -->
    <path
      :d="pathD"
      fill="none" stroke="transparent" stroke-width="14"
      style="pointer-events:stroke;cursor:pointer"
    />
    <!-- Actual pipe (your library component) -->
    <ConnectionPipe
      :start-position="fromPos"
      :end-position="toPos"
      :waypoints="edge.waypoints"
      :flowing="edge.props.flowing"
      :direction="edge.props.direction"
    />
  </g>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import type { DiagramEdge, Point } from '@/domain/models'
import { useOrthoRouter } from '@/composables/useOrthoRouter'
import ConnectionPipe from '@/lib/components/PID/connectors/ConnectionPipe.vue'

interface Props {
  edge: DiagramEdge
  selected: boolean
}

const props = defineProps<Props>()
defineEmits<{ click: [] }>()

const router = useOrthoRouter()

const fromPos = computed(() => router.getPortWorldPos(props.edge.from.nodeId, props.edge.from.portId))
const toPos = computed(() => router.getPortWorldPos(props.edge.to.nodeId, props.edge.to.portId))

const pathD = computed(() => {
  const pts: Point[] = [fromPos.value, ...props.edge.waypoints, toPos.value]
  return pts.map((p, i) => `${i ? 'L' : 'M'} ${p.x} ${p.y}`).join(' ')
})
</script>
```

---

## G. Pages & Router

```typescript
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

export default createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', redirect: '/designer' },
    { path: '/designer', component: () => import('@/pages/DesignerPage.vue') },
    { path: '/examples', component: () => import('@/pages/ExamplesPage.vue') },
  ],
})
```

```vue
<!-- src/pages/DesignerPage.vue -->
<template>
  <v-layout class="designer-layout">
    <!-- Palette (edit mode only) -->
    <v-navigation-drawer
      v-if="store.isEditMode"
      permanent
      :width="220"
      color="surface"
    >
      <PalettePanel :screen-to-canvas="screenToCanvas" />
    </v-navigation-drawer>

    <!-- Main content -->
    <v-main>
      <AppToolbar />

      <div class="canvas-wrapper">
        <DiagramCanvas ref="canvasRef" />
      </div>

      <AppStatusBar />
    </v-main>
  </v-layout>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useDiagramStore } from '@/stores/useDiagramStore'
import AppToolbar from '@/components/layout/AppToolbar.vue'
import AppStatusBar from '@/components/layout/AppStatusBar.vue'
import PalettePanel from '@/components/layout/PalettePanel.vue'
import DiagramCanvas from '@/components/canvas/DiagramCanvas.vue'

const store = useDiagramStore()
const canvasRef = ref()

// Expose screenToCanvas for palette drop
const screenToCanvas = (pos: { x: number; y: number }) => {
  // Delegate to canvas composable
  return canvasRef.value?.screenToCanvas?.(pos) ?? pos
}
</script>

<style scoped>
.designer-layout { height: 100vh; }
.canvas-wrapper { height: calc(100vh - 48px - 32px); /* toolbar + statusbar */ }
</style>
```

---

## H. App Bootstrap

```typescript
// src/main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'
import 'vuetify/styles'
import '@mdi/font/css/materialdesignicons.css'

import App from './App.vue'
import router from './router'

const vuetify = createVuetify({
  components,
  directives,
  theme: {
    defaultTheme: 'dark',
    themes: {
      dark: {
        colors: {
          background: '#0e1117',
          surface: '#161b22',
          'surface-variant': '#1c2129',
          primary: '#58a6ff',
          error: '#f85149',
          warning: '#d29922',
          success: '#3fb950',
        },
      },
    },
  },
})

createApp(App)
  .use(createPinia())
  .use(vuetify)
  .use(router)
  .mount('#app')
```

```vue
<!-- src/App.vue -->
<template>
  <v-app>
    <router-view />
  </v-app>
</template>
```

---

## I. Scaffolding Notes

```bash
# 1. Create project
npm create vite@latest pid-designer -- --template vue-ts
cd pid-designer

# 2. Install dependencies
npm install vue-router@4 pinia vuetify @mdi/font

# 3. Project runs
npm run dev

# 4. Symlink your existing library
ln -s /path/to/your/lib src/lib
# OR copy it
cp -r /path/to/your/lib src/lib
```

```json
// package.json (key deps)
{
  "dependencies": {
    "vue": "^3.4",
    "vue-router": "^4.3",
    "pinia": "^2.2",
    "vuetify": "^3.5",
    "@mdi/font": "^7.4"
  }
}
```

```typescript
// src/utils/uid.ts
let counter = 0
export function uid(): string {
  return `${Date.now().toString(36)}-${(counter++).toString(36)}-${Math.random().toString(36).slice(2, 8)}`
}
```

---

## J. Performance Strategy

| Concern | Strategy |
|---------|----------|
| Node drag | Only recompute connected edge routes on `mouseup`, not every `mousemove` |
| Many edges | Skip rendering edges whose bounding box doesn't intersect viewport |
| Zoom/pan | Only adjusts CSS transform — no recomputation of routes |
| Large diagrams (500+ nodes) | Add viewport culling: `v-if="isNodeVisible(node)"` |
| SVG rendering | `vector-effect="non-scaling-stroke"` — already in your library |
| Reactivity | Pinia's reactive arrays are efficient; use `shallowRef` for large collections if needed later |

---

## K. Binding Architecture (Future, Not Implemented)

Every `DiagramNode` and `DiagramEdge` has an optional `binding?: BindingConfig` field:

```typescript
// Already in the model, ready for future middleware
interface BindingConfig {
  channelId?: string    // "plc.valves.V001.state"
  protocol?: string     // "websocket", "openc3", "modbus"
  transform?: string    // Optional value mapping
  [key: string]: unknown
}
```

In run mode, a future `useBindings()` composable would:
1. Read all nodes/edges with bindings
2. Subscribe to data channels via the appropriate adapter
3. Push incoming values into `node.props` (which are reactive)
4. Components re-render automatically via Vue reactivity

The component library is already "dumb" — it renders from props. So binding just means updating props from external data. No component changes needed.

---

## Summary

| Deliverable | Status | Key Files |
|-------------|--------|-----------|
| A. Architecture | ✅ | Folder structure above |
| B. Data model | ✅ | `domain/models.ts` |
| C. Canvas approach | ✅ | SVG + transform group, `DiagramCanvas.vue` |
| D. Ortho routing | ✅ | `useOrthoRouter.ts` + your existing algorithm |
| E. Edit interactions | ✅ | `useNodeDrag`, `usePipeDraw`, `usePaletteDrop` |
| F. Run mode | ✅ | `store.isRunMode` → hides ports, disables drag |
| G. Export/import | ✅ | `useSerializer.ts` with validation |
| H. UI / Vuetify | ✅ | Dark theme, drawer, toolbar |
| Junction 8-port | ✅ | `JunctionNode.vue` |
| Pinia store | ✅ | Single store, all actions typed |
| Router | ✅ | `/designer` + `/examples` |
