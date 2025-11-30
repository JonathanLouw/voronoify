<script setup>
import { Delaunay } from 'd3-delaunay'
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { Crosshair, Download, MapPinMinus, MapPinPlus, MousePointerClick, Settings, Trash2, Undo2 } from 'lucide-vue-next'

const canvasRef = ref(null)
const sites = ref([])
const nextId = ref(1)
const viewport = ref({ width: window.innerWidth, height: window.innerHeight })
const pan = ref({ x: 0, y: 0 })
const scale = ref(1)
const isPanning = ref(false)
const lastPointer = ref({ x: 0, y: 0 })
const didPan = ref(false)
const activeTool = ref('sites')
const cursorWorld = ref({ x: 0, y: 0 })
const gridSize = ref(64)
const selectedSiteId = ref(null)
const selectedSite = computed(() => sites.value.find((s) => s.id === selectedSiteId.value) || null)
const multiSelectedIds = ref(new Set())
const hoveredSiteId = ref(null)
const isErasing = ref(false)
const isSelectingBox = ref(false)
const selectionStart = ref({ x: 0, y: 0 })
const selectionEnd = ref({ x: 0, y: 0 })
const selectionBackup = ref(new Set())
const suppressNextClick = ref(false)
const imageBitmap = ref(null)
const imageSize = ref({ width: 0, height: 0 })
const showSettings = ref(false)
const exportOpen = ref(false)
const styleOptions = ref({
  gridOpacity: 0.05,
  gridWidth: 1,
  gridColor: '#ffffff',
  axisOpacity: 0.28,
  axisWidth: 1,
  axisColor: '#5eead4',
  voronoiOpacity: 0.2,
  voronoiWidth: 1,
  voronoiColor: '#ffffff',
  siteStrokeWidth: 1.6,
  siteStrokeColor: '#ffffff',
  siteStrokeOpacity: 0.9,
  highlightWidth: 3,
  highlightOpacity: 0.6,
  highlightColor: '#5eead4',
})
const selectionBoxStyle = computed(() => {
  if (!isSelectingBox.value) return null
  const x1 = selectionStart.value.x
  const y1 = selectionStart.value.y
  const x2 = selectionEnd.value.x
  const y2 = selectionEnd.value.y
  const left = Math.min(x1, x2)
  const top = Math.min(y1, y2)
  const width = Math.abs(x2 - x1)
  const height = Math.abs(y2 - y1)
  return {
    left: `${left}px`,
    top: `${top}px`,
    width: `${width}px`,
    height: `${height}px`,
  }
})
const canvasCursor = computed(() => {
  if (isPanning.value) return 'cursor-grabbing'
  if (activeTool.value === 'select') {
    if (isSelectingBox.value) return 'cursor-crosshair'
    if (hoveredSiteId.value) return 'cursor-pointer'
    return 'cursor-default'
  }
  if (activeTool.value === 'paint-delete') return 'cursor-crosshair'
  return 'cursor-crosshair'
})

function resizeCanvas() {
  const canvas = canvasRef.value
  if (!canvas) return

  const dpr = window.devicePixelRatio || 1
  const width = window.innerWidth
  const height = window.innerHeight

  viewport.value = { width, height }
  canvas.width = Math.floor(width * dpr)
  canvas.height = Math.floor(height * dpr)
  canvas.style.width = '100vw'
  canvas.style.height = '100vh'

  const ctx = canvas.getContext('2d')
  if (ctx) {
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  }

  draw()
}

function worldToScreen(point) {
  const originX = pan.value.x
  const originY = pan.value.y
  return {
    x: originX + point.x * scale.value,
    y: originY + point.y * scale.value,
  }
}

function screenToWorld(x, y) {
  const originX = pan.value.x
  const originY = pan.value.y
  return {
    x: (x - originX) / scale.value,
    y: (y - originY) / scale.value,
  }
}

function drawBackground(ctx, width, height) {
  const gradient = ctx.createLinearGradient(0, 0, width, height)
  gradient.addColorStop(0, '#0b1220')
  gradient.addColorStop(1, '#0b1828')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, width, height)

  if (imageBitmap.value) {
    const imgW = imageSize.value.width
    const imgH = imageSize.value.height
    const { x: originX, y: originY } = worldToScreen({ x: 0, y: 0 })
    const drawW = imgW * scale.value
    const drawH = imgH * scale.value
    ctx.globalAlpha = 1
    ctx.drawImage(imageBitmap.value, originX, originY, drawW, drawH)
  }
}

function drawGrid(ctx, width, height) {
  const stepPx = gridSize.value * scale.value
  if (stepPx < 4) return

  const originX = pan.value.x
  const originY = pan.value.y

  ctx.strokeStyle = colorWithAlpha(styleOptions.value.gridColor, styleOptions.value.gridOpacity)
  ctx.lineWidth = styleOptions.value.gridWidth

  const startKx = Math.floor(-originX / stepPx) - 1
  for (let k = startKx; ; k++) {
    const x = originX + k * stepPx
    if (x > width) break
    if (x < -stepPx) continue
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, height)
    ctx.stroke()
  }

  const startKy = Math.floor(-originY / stepPx) - 1
  for (let k = startKy; ; k++) {
    const y = originY + k * stepPx
    if (y > height) break
    if (y < -stepPx) continue
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(width, y)
    ctx.stroke()
  }
}

function drawAxes(ctx, width, height) {
  const originX = pan.value.x
  const originY = pan.value.y

  ctx.strokeStyle = colorWithAlpha(styleOptions.value.axisColor, styleOptions.value.axisOpacity)
  ctx.lineWidth = styleOptions.value.axisWidth

  ctx.beginPath()
  ctx.moveTo(0, originY)
  ctx.lineTo(width, originY)
  ctx.stroke()

  ctx.beginPath()
  ctx.moveTo(originX, 0)
  ctx.lineTo(originX, height)
  ctx.stroke()

  ctx.fillStyle = 'rgba(34,211,238,0.45)'
  ctx.beginPath()
  ctx.arc(originX, originY, 5, 0, Math.PI * 2)
  ctx.fill()
  ctx.strokeStyle = 'rgba(255,255,255,0.35)'
  ctx.lineWidth = 1
  ctx.stroke()

  ctx.fillStyle = '#e2f8f0'
  ctx.font = '600 13px "Inter", system-ui, -apple-system, sans-serif'
  ctx.textAlign = 'left'
  ctx.textBaseline = 'top'
  ctx.fillText('0,0', originX + 10, originY + 8)
}

function drawVoronoi(ctx, width, height) {
  if (sites.value.length < 2) return

  const points = sites.value.map((site) => worldToScreen(site))
  const delaunay = Delaunay.from(
    points,
    (point) => point.x,
    (point) => point.y
  )
  const voronoi = delaunay.voronoi([0, 0, width, height])

  ctx.strokeStyle = colorWithAlpha(styleOptions.value.voronoiColor, styleOptions.value.voronoiOpacity)
  ctx.lineWidth = styleOptions.value.voronoiWidth

  for (let i = 0; i < points.length; i++) {
    const polygon = voronoi.cellPolygon(i)
    if (!polygon) continue

    ctx.beginPath()
    ctx.moveTo(polygon[0][0], polygon[0][1])
    for (let j = 1; j < polygon.length; j++) {
      ctx.lineTo(polygon[j][0], polygon[j][1])
    }
    ctx.closePath()
    ctx.stroke()
  }
}

function drawSites(ctx, width, height) {
  ctx.textAlign = 'center'

  sites.value.forEach((site) => {
    const { x, y } = worldToScreen(site)
    const isSelected = site.id === selectedSiteId.value
    const isMultiSelected = multiSelectedIds.value.has(site.id)

    if (x < -20 || x > width + 20 || y < -20 || y > height + 20) {
      return
    }

    ctx.fillStyle = '#10b981'
    ctx.beginPath()
    ctx.arc(x, y, 7, 0, Math.PI * 2)
    ctx.fill()

    ctx.strokeStyle = isSelected || isMultiSelected
      ? colorWithAlpha(styleOptions.value.highlightColor, styleOptions.value.siteStrokeOpacity)
      : colorWithAlpha(styleOptions.value.siteStrokeColor, styleOptions.value.siteStrokeOpacity)
    ctx.lineWidth = styleOptions.value.siteStrokeWidth
    ctx.stroke()

    if (isSelected || isMultiSelected) {
      ctx.strokeStyle = colorWithAlpha(styleOptions.value.highlightColor, styleOptions.value.highlightOpacity)
      ctx.lineWidth = styleOptions.value.highlightWidth
      ctx.beginPath()
      ctx.arc(x, y, 11, 0, Math.PI * 2)
      ctx.stroke()
    }

    ctx.fillStyle = '#0b1220'
    ctx.beginPath()
    ctx.arc(x, y, 3, 0, Math.PI * 2)
    ctx.fill()

    ctx.fillStyle = '#e2f8f0'
    ctx.font = '600 13px "Inter", system-ui, -apple-system, sans-serif'
    ctx.textBaseline = 'bottom'
    ctx.fillText(`#${site.id}`, x, y - 12)

    ctx.font = '500 12px "Inter", system-ui, -apple-system, sans-serif'
    ctx.textBaseline = 'top'
    ctx.fillStyle = 'rgba(226,248,240,0.9)'
    ctx.fillText(`${site.x.toFixed(1)}, ${site.y.toFixed(1)}`, x, y + 10)
  })
}

function draw() {
  const canvas = canvasRef.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const width = viewport.value.width
  const height = viewport.value.height

  ctx.clearRect(0, 0, width, height)
  drawBackground(ctx, width, height)
  drawGrid(ctx, width, height)
  drawAxes(ctx, width, height)
  drawVoronoi(ctx, width, height)
  drawSites(ctx, width, height)
}

function handleCanvasClick(event) {
  if (suppressNextClick.value) {
    suppressNextClick.value = false
    return
  }
  if (!canvasRef.value || didPan.value || isSelectingBox.value) return

  const rect = canvasRef.value.getBoundingClientRect()
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top

  const world = screenToWorld(x, y)

  if (activeTool.value === 'sites') {
    sites.value.push({
      id: nextId.value++,
      x: world.x,
      y: world.y,
    })
    selectedSiteId.value = null
  } else if (activeTool.value === 'select') {
    selectSiteAt(x, y)
  }

  draw()
}

function popSite() {
  if (!sites.value.length) return
  sites.value = sites.value.slice(0, -1)
  if (selectedSiteId.value && !sites.value.find((s) => s.id === selectedSiteId.value)) {
    selectedSiteId.value = null
  }
  multiSelectedIds.value = new Set()
  draw()
}

function clearSites() {
  sites.value = []
  nextId.value = 1
  selectedSiteId.value = null
  multiSelectedIds.value = new Set()
  draw()
}

function handleWheel(event) {
  if (!canvasRef.value) return

  const rect = canvasRef.value.getBoundingClientRect()
  const cursorX = event.clientX - rect.left
  const cursorY = event.clientY - rect.top

  const worldBefore = screenToWorld(cursorX, cursorY)
  const zoomFactor = Math.exp(-event.deltaY * 0.001)
  const newScale = Math.min(5, Math.max(0.2, scale.value * zoomFactor))

  scale.value = newScale

  const screenAfter = worldToScreen(worldBefore)
  pan.value = {
    x: pan.value.x + (cursorX - screenAfter.x),
    y: pan.value.y + (cursorY - screenAfter.y),
  }

  cursorWorld.value = worldBefore
  event.preventDefault()
  draw()
}

function selectSiteAt(screenX, screenY) {
  const hit = hitTestSite(screenX, screenY)
  selectedSiteId.value = hit
  multiSelectedIds.value = new Set()
}

function hitTestSite(screenX, screenY, radius = 12) {
  let bestId = null
  let bestDist = Infinity

  sites.value.forEach((site) => {
    const { x, y } = worldToScreen(site)
    const dx = screenX - x
    const dy = screenY - y
    const dist = Math.hypot(dx, dy)
    if (dist < radius && dist < bestDist) {
      bestDist = dist
      bestId = site.id
    }
  })

  return bestId
}

function deleteSelectedSite() {
  if (!selectedSite.value) return
  sites.value = sites.value.filter((s) => s.id !== selectedSiteId.value)
  selectedSiteId.value = null
  multiSelectedIds.value = new Set()
  draw()
}

function deleteMultiSelected() {
  if (!multiSelectedIds.value.size) return
  sites.value = sites.value.filter((s) => !multiSelectedIds.value.has(s.id))
  selectedSiteId.value = null
  multiSelectedIds.value = new Set()
  draw()
}

function updateSelectionBox() {
  const x1 = selectionStart.value.x
  const y1 = selectionStart.value.y
  const x2 = selectionEnd.value.x
  const y2 = selectionEnd.value.y
  const minX = Math.min(x1, x2)
  const maxX = Math.max(x1, x2)
  const minY = Math.min(y1, y2)
  const maxY = Math.max(y1, y2)

  const next = new Set()
  sites.value.forEach((site) => {
    const { x, y } = worldToScreen(site)
    if (x >= minX && x <= maxX && y >= minY && y <= maxY) {
      next.add(site.id)
    }
  })
  multiSelectedIds.value = next
  draw()
}

function updateSelected(coord, value) {
  if (!selectedSite.value) return
  const parsed = Number(value)
  if (Number.isNaN(parsed)) return
  selectedSite.value[coord] = parsed
  draw()
}

function deleteSiteAt(screenX, screenY, radius = 12) {
  const before = sites.value.length
  sites.value = sites.value.filter((site) => {
    const { x, y } = worldToScreen(site)
    const dx = screenX - x
    const dy = screenY - y
    return Math.hypot(dx, dy) > radius
  })
  if (sites.value.length !== before) {
    if (selectedSiteId.value && !sites.value.find((s) => s.id === selectedSiteId.value)) {
      selectedSiteId.value = null
    }
    if (multiSelectedIds.value.size) {
      multiSelectedIds.value = new Set([...multiSelectedIds.value].filter((id) => sites.value.some((s) => s.id === id)))
    }
    draw()
  }
}

function resetZoom() {
  scale.value = 1
  draw()
}

function resetPosition() {
  pan.value = { x: 0, y: 0 }
  draw()
}

function downloadBlob(filename, blob) {
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()
  URL.revokeObjectURL(url)
}

function exportSitesJson() {
  const json = JSON.stringify(sites.value, null, 2)
  const blob = new Blob([json], { type: 'application/json' })
  downloadBlob('sites.json', blob)
  exportOpen.value = false
}

function drawExportOverlay(ctx, width, height) {
  if (sites.value.length >= 2) {
    const points = sites.value.map((site) => ({ x: site.x, y: site.y }))
    const delaunay = Delaunay.from(
      points,
      (p) => p.x,
      (p) => p.y
    )
    const voronoi = delaunay.voronoi([0, 0, width, height])
    ctx.strokeStyle = colorWithAlpha(styleOptions.value.voronoiColor, styleOptions.value.voronoiOpacity)
    ctx.lineWidth = styleOptions.value.voronoiWidth
    for (let i = 0; i < points.length; i++) {
      const poly = voronoi.cellPolygon(i)
      if (!poly) continue
      ctx.beginPath()
      ctx.moveTo(poly[0][0], poly[0][1])
      for (let j = 1; j < poly.length; j++) {
        ctx.lineTo(poly[j][0], poly[j][1])
      }
      ctx.closePath()
      ctx.stroke()
    }
  }

  ctx.textAlign = 'center'
  ctx.font = '600 13px "Inter", system-ui, -apple-system, sans-serif'
  sites.value.forEach((site) => {
    const x = site.x
    const y = site.y
    const selected = selectedSiteId.value === site.id || multiSelectedIds.value.has(site.id)

    ctx.fillStyle = '#10b981'
    ctx.beginPath()
    ctx.arc(x, y, 7, 0, Math.PI * 2)
    ctx.fill()

    ctx.strokeStyle = selected
      ? colorWithAlpha(styleOptions.value.highlightColor, styleOptions.value.siteStrokeOpacity)
      : colorWithAlpha(styleOptions.value.siteStrokeColor, styleOptions.value.siteStrokeOpacity)
    ctx.lineWidth = styleOptions.value.siteStrokeWidth
    ctx.stroke()

    if (selected) {
      ctx.strokeStyle = colorWithAlpha(styleOptions.value.highlightColor, styleOptions.value.highlightOpacity)
      ctx.lineWidth = styleOptions.value.highlightWidth
      ctx.beginPath()
      ctx.arc(x, y, 11, 0, Math.PI * 2)
      ctx.stroke()
    }

    ctx.fillStyle = '#0b1220'
    ctx.beginPath()
    ctx.arc(x, y, 3, 0, Math.PI * 2)
    ctx.fill()

    ctx.fillStyle = '#e2f8f0'
    ctx.textBaseline = 'bottom'
    ctx.fillText(`#${site.id}`, x, y - 12)
    ctx.font = '500 12px "Inter", system-ui, -apple-system, sans-serif'
    ctx.textBaseline = 'top'
    ctx.fillStyle = 'rgba(226,248,240,0.9)'
    ctx.fillText(`${site.x.toFixed(1)}, ${site.y.toFixed(1)}`, x, y + 10)
    ctx.font = '600 13px "Inter", system-ui, -apple-system, sans-serif'
  })
}

async function exportImageFile() {
  if (!imageBitmap.value || !imageSize.value.width || !imageSize.value.height) return
  const { width, height } = imageSize.value
  const off = document.createElement('canvas')
  off.width = width
  off.height = height
  const ctx = off.getContext('2d')
  if (!ctx) return
  ctx.drawImage(imageBitmap.value, 0, 0, width, height)
  drawExportOverlay(ctx, width, height)
  const blob = await new Promise((resolve) => off.toBlob(resolve, 'image/png', 1))
  if (blob) {
    downloadBlob('voronoify-image.png', blob)
  }
  exportOpen.value = false
}

function colorWithAlpha(color, alpha) {
  if (!color) return `rgba(255,255,255,${alpha})`
  if (color.startsWith('#')) {
    let hex = color.slice(1)
    if (hex.length === 3) {
      hex = hex.split('').map((c) => c + c).join('')
    }
    const int = parseInt(hex, 16)
    const r = (int >> 16) & 255
    const g = (int >> 8) & 255
    const b = int & 255
    return `rgba(${r},${g},${b},${alpha})`
  }
  return color
}

async function handleImageUpload(event) {
  const file = event.target?.files?.[0]
  if (!file) return
  try {
    const blob = await file.arrayBuffer()
    const bitmap = await createImageBitmap(new Blob([blob]))
    imageBitmap.value = bitmap
    imageSize.value = { width: bitmap.width, height: bitmap.height }
    draw()
  } catch (err) {
    console.error('Failed to load image', err)
  } finally {
    event.target.value = ''
  }
}

function handlePointerDown(event) {
  const isMiddle = event.button === 1
  const isLeft = event.button === 0
  const rect = canvasRef.value?.getBoundingClientRect()
  const x = rect ? event.clientX - rect.left : null
  const y = rect ? event.clientY - rect.top : null

  if (activeTool.value === 'paint-delete' && isLeft) {
    isErasing.value = true
    if (rect && x !== null && y !== null) {
      deleteSiteAt(x, y, 12)
    }
    event.preventDefault()
    return
  }

  if (activeTool.value === 'select' && isLeft && event.shiftKey) {
    if (rect && x !== null && y !== null) {
      const hit = hitTestSite(x, y)
      if (hit) {
        const next = new Set(multiSelectedIds.value)
        if (next.has(hit)) next.delete(hit)
        else next.add(hit)
        multiSelectedIds.value = next
        selectedSiteId.value = next.size === 1 ? [...next][0] : null
        draw()
      } else {
        selectionBackup.value = new Set(multiSelectedIds.value)
        isSelectingBox.value = true
        selectionStart.value = { x, y }
        selectionEnd.value = { x, y }
        suppressNextClick.value = true
      }
    }
    event.preventDefault()
    return
  }

  const hitAtPointer = rect && x !== null && y !== null ? hitTestSite(x, y) : null

  const canPan =
    isMiddle ||
    (activeTool.value === 'select' && isLeft && !event.shiftKey && !hitAtPointer) ||
    (activeTool.value === 'sites' && isMiddle) ||
    (activeTool.value === 'paint-delete' && isMiddle)

  if (!canPan) return

  isPanning.value = true
  didPan.value = false
  lastPointer.value = { x: event.clientX, y: event.clientY }
  event.preventDefault()
  event.target?.setPointerCapture?.(event.pointerId)
}

function handlePointerMove(event) {
  const rect = canvasRef.value?.getBoundingClientRect()
  if (rect) {
    const x = event.clientX - rect.left
    const y = event.clientY - rect.top
    cursorWorld.value = screenToWorld(x, y)
    if (!isPanning.value && activeTool.value === 'select') {
      hoveredSiteId.value = hitTestSite(x, y, 14)
    } else if (!isPanning.value) {
      hoveredSiteId.value = null
    }
    if (isSelectingBox.value && activeTool.value === 'select') {
      if (!event.shiftKey) {
        isSelectingBox.value = false
        multiSelectedIds.value = new Set(selectionBackup.value)
        selectionStart.value = { x: 0, y: 0 }
        selectionEnd.value = { x: 0, y: 0 }
        draw()
        return
      }
      selectionEnd.value = { x, y }
      updateSelectionBox()
      return
    }
    if (isErasing.value && activeTool.value === 'paint-delete') {
      deleteSiteAt(x, y, 12)
      return
    }
  }

  if (!isPanning.value) return
  const dx = event.clientX - lastPointer.value.x
  const dy = event.clientY - lastPointer.value.y

  if (Math.abs(dx) > 1 || Math.abs(dy) > 1) {
    didPan.value = true
  }

  pan.value = {
    x: pan.value.x + dx,
    y: pan.value.y + dy,
  }

  event.preventDefault()
  lastPointer.value = { x: event.clientX, y: event.clientY }
  draw()
}

function handlePointerUp(event) {
  hoveredSiteId.value = null
  if (isErasing.value) {
    isErasing.value = false
  }
  if (isSelectingBox.value) {
    isSelectingBox.value = false
    if (multiSelectedIds.value.size === 1) {
      selectedSiteId.value = [...multiSelectedIds.value][0]
    } else {
      selectedSiteId.value = null
    }
    selectionStart.value = { x: 0, y: 0 }
    selectionEnd.value = { x: 0, y: 0 }
    suppressNextClick.value = true
    draw()
  }
  if (isPanning.value) {
    isPanning.value = false
    didPan.value = false
    event.target?.releasePointerCapture?.(event.pointerId)
  }
}

function setTool(tool) {
  activeTool.value = tool
  if (tool === 'sites') {
    selectedSiteId.value = null
    hoveredSiteId.value = null
  }
  if (tool !== 'select') {
    hoveredSiteId.value = null
    multiSelectedIds.value = new Set()
    isSelectingBox.value = false
  }
}

onMounted(() => {
  resizeCanvas()
  window.addEventListener('resize', resizeCanvas)
  window.addEventListener('keydown', handleKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', resizeCanvas)
  window.removeEventListener('keydown', handleKeydown)
})

watch(
  () => sites.value.length,
  () => draw()
)

watch(
  () => scale.value,
  () => draw()
)

watch(
  styleOptions,
  () => draw(),
  { deep: true }
)

function handleKeydown(event) {
  const tag = event.target?.tagName
  const inInput = tag && ['INPUT', 'TEXTAREA'].includes(tag)

  if (event.ctrlKey && event.key.toLowerCase() === 'z' && !inInput) {
    event.preventDefault()
    popSite()
    return
  }

  if (activeTool.value !== 'select') return
  if (event.key === 'Delete' && (selectedSite.value || multiSelectedIds.value.size)) {
    if (inInput) return
    event.preventDefault()
    if (selectedSite.value) deleteSelectedSite()
    else deleteMultiSelected()
  }
}
</script>

<template>
  <main class="relative h-screen w-screen overflow-hidden bg-slate-950 text-white">
    <canvas
      ref="canvasRef"
      class="absolute inset-0 h-full w-full"
      :class="canvasCursor"
      @click="handleCanvasClick"
      @contextmenu.prevent
      @wheel.prevent="handleWheel"
      @pointerdown="handlePointerDown"
      @pointermove="handlePointerMove"
      @pointerup="handlePointerUp"
      @pointercancel="handlePointerUp"
      @pointerleave="handlePointerUp"
    />

    <div v-if="exportOpen || showSettings" class="pointer-events-auto absolute inset-0" @click="exportOpen = false; showSettings = false"></div>

    <div class="pointer-events-none absolute inset-0">
      <div class="absolute top-4 left-4 pointer-events-auto">
        <div class="relative">
          <button
            type="button"
            class="inline-flex items-center gap-2 rounded-full border border-white/10 bg-slate-900/80 px-3 py-2 text-sm font-semibold text-slate-100 shadow-lg backdrop-blur transition hover:-translate-y-[1px]"
            @click="exportOpen = !exportOpen"
            title="Export"
          >
            <Download class="h-5 w-5" />
            Export
          </button>
          <div
            v-if="exportOpen"
            class="absolute left-0 top-full mt-2 w-52 rounded-2xl border border-white/10 bg-slate-900/95 p-2 text-sm text-slate-100 shadow-xl backdrop-blur"
          >
            <button
              type="button"
              class="flex w-full items-center gap-2 rounded-lg px-3 py-2 text-left transition hover:bg-white/10"
              @click="exportSitesJson"
            >
              Export sites (JSON)
            </button>
            <button
              type="button"
              class="flex w-full items-center gap-2 rounded-lg px-3 py-2 text-left transition hover:bg-white/10 disabled:opacity-40 disabled:hover:bg-transparent"
              :disabled="!imageBitmap"
              @click="exportImageFile"
            >
              Export image (PNG)
            </button>
          </div>
        </div>
      </div>
      <div class="absolute top-4 right-4 pointer-events-auto">
        <button
          type="button"
          class="inline-flex items-center gap-2 rounded-full border border-white/10 bg-slate-900/80 px-3 py-2 text-sm font-semibold text-slate-100 shadow-lg backdrop-blur transition hover:-translate-y-[1px]"
          @click="showSettings = true"
          title="Open options"
        >
          <Settings class="h-5 w-5" />
          Options
        </button>
      </div>
      <div class="absolute left-1/2 bottom-6 -translate-x-1/2">
        <div class="pointer-events-auto flex items-center gap-3 rounded-full border border-white/10 bg-slate-900/80 px-4 py-2 shadow-2xl shadow-emerald-500/10 backdrop-blur">
          <div class="flex rounded-full border border-white/10 bg-white/5">
            <button
              type="button"
              class="flex items-center gap-2 rounded-l-full px-3 py-1.5 text-sm font-semibold transition hover:translate-y-[-1px]"
              :class="activeTool === 'sites' ? 'bg-emerald-500/90 text-slate-950 shadow-lg shadow-emerald-500/40' : 'text-slate-100'"
              :aria-pressed="activeTool === 'sites'"
              @click="setTool('sites')"
              title="Sites tool (add points)"
            >
              <MapPinPlus class="h-5 w-5" />
            </button>
            <button
              type="button"
              class="flex items-center gap-2 px-3 py-1.5 text-sm font-semibold transition hover:translate-y-[-1px]"
              :class="activeTool === 'paint-delete' ? 'bg-emerald-500/90 text-slate-950 shadow-lg shadow-emerald-500/40' : 'text-slate-100'"
              :aria-pressed="activeTool === 'paint-delete'"
              @click="setTool('paint-delete')"
              title="Paint delete tool"
            >
              <MapPinMinus class="h-5 w-5" />
            </button>
            <button
              type="button"
              class="flex items-center gap-2 rounded-r-full px-3 py-1.5 text-sm font-semibold transition hover:translate-y-[-1px]"
              :class="activeTool === 'select' ? 'bg-emerald-500/90 text-slate-950 shadow-lg shadow-emerald-500/40' : 'text-slate-100'"
              :aria-pressed="activeTool === 'select'"
              @click="setTool('select')"
              title="Select / multi-select (Shift-drag) tool"
            >
              <MousePointerClick class="h-5 w-5" />
            </button>
          </div>
          <span class="text-sm text-slate-100/90">{{ sites.length || 'No' }} site{{ sites.length === 1 ? '' : 's' }}</span>
          <label class="flex items-center gap-2 rounded-full border border-white/10 bg-white/5 px-3 py-1.5 text-sm font-semibold text-slate-100 shadow-none transition hover:translate-y-[-1px] hover:bg-white/10 cursor-pointer">
            <input type="file" accept="image/*" class="hidden" @change="handleImageUpload" />
            Upload image
          </label>
          <div class="flex items-center gap-1">
            <button
              type="button"
              class="rounded-full border border-white/10 bg-white/5 p-2 text-slate-100 transition hover:border-white/30 hover:bg-white/10"
              @click="popSite"
              aria-label="Undo last site"
              title="Undo last site"
            >
              <Undo2 class="h-4 w-4" />
            </button>
            <button
              type="button"
              class="rounded-full border border-white/10 bg-white/5 p-2 text-rose-100 transition hover:border-rose-300/60 hover:bg-rose-400/15"
              @click="clearSites"
              aria-label="Clear all sites"
              title="Clear all sites"
            >
              <Trash2 class="h-4 w-4" />
            </button>
          </div>
        </div>
      </div>
    </div>
    <div v-if="selectedSite && multiSelectedIds.size <= 1" class="pointer-events-none absolute inset-y-0 right-0 flex items-center pr-4">
      <div class="pointer-events-auto w-64 max-w-sm rounded-2xl border border-white/10 bg-slate-900/85 p-4 text-sm text-slate-100 shadow-2xl backdrop-blur">
        <div class="flex items-center justify-between mb-3">
          <div class="font-semibold text-emerald-200">Site #{{ selectedSite.id }}</div>
          <button
            type="button"
            class="rounded-full border border-white/10 bg-white/5 p-2 text-rose-100 transition hover:border-rose-300/60 hover:bg-rose-400/15"
            @click="deleteSelectedSite"
          >
            <Trash2 class="h-4 w-4" />
          </button>
        </div>
        <div class="space-y-2">
          <label class="block text-xs uppercase tracking-[0.15em] text-slate-400">X coordinate</label>
          <input
            type="number"
            step="0.1"
            :value="selectedSite.x"
            class="w-full rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1 text-slate-100 outline-none focus:border-emerald-400/60"
            @input="updateSelected('x', $event.target.value)"
          />
          <label class="block text-xs uppercase tracking-[0.15em] text-slate-400">Y coordinate</label>
          <input
            type="number"
            step="0.1"
            :value="selectedSite.y"
            class="w-full rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1 text-slate-100 outline-none focus:border-emerald-400/60"
            @input="updateSelected('y', $event.target.value)"
          />
        </div>
      </div>
    </div>
    <div class="pointer-events-none absolute bottom-6 right-6">
      <div class="pointer-events-auto flex items-center gap-2 rounded-xl border border-white/10 bg-slate-900/80 px-3 py-2 text-sm text-slate-100 shadow-lg backdrop-blur">
        <div>{{ cursorWorld.x.toFixed(1) }}, {{ cursorWorld.y.toFixed(1) }}</div>
        <div class="flex rounded-full border border-white/10 bg-white/5">
          <button
            type="button"
            class="flex items-center gap-2 rounded-full px-2 py-1 text-xs font-semibold text-slate-100 transition hover:translate-y-[-1px] hover:bg-white/10"
            @click="resetPosition"
            title="Reset position to origin"
          >
            <Crosshair class="h-4 w-4" />
          </button>
        </div>
      </div>
    </div>
    <div class="pointer-events-none absolute bottom-6 left-6">
      <div class="pointer-events-auto flex items-center gap-2 rounded-xl border border-white/10 bg-slate-900/80 px-3 py-2 text-sm text-slate-100 shadow-lg backdrop-blur">
        <span class="font-semibold text-emerald-200">Zoom {{ scale.toFixed(2) }}x</span>
        <button
          v-if="Math.abs(scale - 1) > 0.001"
          type="button"
          class="rounded-full border border-white/10 bg-white/5 px-2 py-1 text-xs font-semibold text-slate-100 transition hover:border-white/30 hover:bg-white/10"
          @click="resetZoom"
        >
          Reset
        </button>
      </div>
    </div>
    <div v-if="selectionBoxStyle" class="pointer-events-none absolute inset-0">
      <div
        class="absolute rounded-md border-2 border-emerald-300/60 bg-emerald-300/10 shadow-[0_0_0_1px_rgba(16,185,129,0.25)]"
        :style="selectionBoxStyle"
      ></div>
    </div>
    <div v-if="showSettings" class="pointer-events-none absolute inset-0">
      <div class="absolute top-4 right-4 pointer-events-auto">
        <div class="origin-top-right translate-y-2 rounded-2xl border border-white/10 bg-slate-900/95 p-4 text-sm text-slate-100 shadow-2xl backdrop-blur min-w-[360px] max-w-[440px]">
          <div class="flex items-center justify-between mb-3">
            <h2 class="text-base font-semibold text-white">Canvas options</h2>
            <button
              type="button"
              class="rounded-full border border-white/10 bg-white/5 px-3 py-1 text-xs font-semibold text-slate-100 transition hover:border-white/30 hover:bg-white/10"
              @click="showSettings = false"
            >
              Close
            </button>
          </div>
          <div class="grid grid-cols-1 gap-3">
            <div class="space-y-2 rounded-xl border border-white/10 bg-white/5 p-3">
              <div class="flex items-center justify-between">
                <span class="font-semibold text-emerald-200">Grid</span>
              </div>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Opacity</span>
                <input type="range" min="0" max="1" step="0.01" v-model.number="styleOptions.gridOpacity" class="w-32" />
                <input type="number" step="0.01" min="0" max="1" v-model.number="styleOptions.gridOpacity" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Thickness</span>
                <input type="range" min="0.5" max="4" step="0.1" v-model.number="styleOptions.gridWidth" class="w-32" />
                <input type="number" step="0.1" min="0.1" max="10" v-model.number="styleOptions.gridWidth" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Color</span>
                <input type="color" v-model="styleOptions.gridColor" class="h-8 w-12 rounded border border-white/10 bg-slate-800" />
              </label>
            </div>
            <div class="space-y-2 rounded-xl border border-white/10 bg-white/5 p-3">
              <div class="flex items-center justify-between">
                <span class="font-semibold text-emerald-200">Axes</span>
              </div>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Opacity</span>
                <input type="range" min="0" max="1" step="0.01" v-model.number="styleOptions.axisOpacity" class="w-32" />
                <input type="number" step="0.01" min="0" max="1" v-model.number="styleOptions.axisOpacity" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Thickness</span>
                <input type="range" min="0.5" max="4" step="0.1" v-model.number="styleOptions.axisWidth" class="w-32" />
                <input type="number" step="0.1" min="0.1" max="10" v-model.number="styleOptions.axisWidth" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Color</span>
                <input type="color" v-model="styleOptions.axisColor" class="h-8 w-12 rounded border border-white/10 bg-slate-800" />
              </label>
            </div>
            <div class="space-y-2 rounded-xl border border-white/10 bg-white/5 p-3">
              <div class="flex items-center justify-between">
                <span class="font-semibold text-emerald-200">Voronoi</span>
              </div>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Opacity</span>
                <input type="range" min="0" max="1" step="0.01" v-model.number="styleOptions.voronoiOpacity" class="w-32" />
                <input type="number" step="0.01" min="0" max="1" v-model.number="styleOptions.voronoiOpacity" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Thickness</span>
                <input type="range" min="0.5" max="4" step="0.1" v-model.number="styleOptions.voronoiWidth" class="w-32" />
                <input type="number" step="0.1" min="0.1" max="10" v-model.number="styleOptions.voronoiWidth" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Color</span>
                <input type="color" v-model="styleOptions.voronoiColor" class="h-8 w-12 rounded border border-white/10 bg-slate-800" />
              </label>
            </div>
            <div class="space-y-2 rounded-xl border border-white/10 bg-white/5 p-3">
              <div class="flex items-center justify-between">
                <span class="font-semibold text-emerald-200">Sites</span>
              </div>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Stroke width</span>
                <input type="range" min="0.5" max="4" step="0.1" v-model.number="styleOptions.siteStrokeWidth" class="w-32" />
                <input type="number" step="0.1" min="0.1" max="10" v-model.number="styleOptions.siteStrokeWidth" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Highlight width</span>
                <input type="range" min="1" max="8" step="0.1" v-model.number="styleOptions.highlightWidth" class="w-32" />
                <input type="number" step="0.1" min="0.1" max="10" v-model.number="styleOptions.highlightWidth" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Highlight opacity</span>
                <input type="range" min="0" max="1" step="0.01" v-model.number="styleOptions.highlightOpacity" class="w-32" />
                <input type="number" step="0.01" min="0" max="1" v-model.number="styleOptions.highlightOpacity" class="w-16 rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Stroke color</span>
                <input type="color" v-model="styleOptions.siteStrokeColor" class="h-8 w-12 rounded border border-white/10 bg-slate-800" />
              </label>
              <label class="flex items-center justify-between gap-3">
                <span class="text-xs text-slate-300">Highlight color</span>
                <input type="color" v-model="styleOptions.highlightColor" class="h-8 w-12 rounded border border-white/10 bg-slate-800" />
              </label>
            </div>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>
