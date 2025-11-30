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

    <div v-if="exportOpen || showSettings" class="pointer-events-auto absolute inset-0" @click="closePopovers"></div>

    <div class="pointer-events-none absolute inset-0">
      <div class="absolute top-4 left-4 pointer-events-auto">
        <ExportMenu
          :open="exportOpen"
          :image-available="!!imageBitmap"
          @toggle="exportOpen = !exportOpen"
          @export-json="exportSitesJson"
          @export-image="exportImageFile"
        >
          <template #icon>
            <Download class="h-5 w-5" />
          </template>
        </ExportMenu>
      </div>

      <div class="absolute top-4 right-4 pointer-events-auto">
        <div class="relative">
          <button
            type="button"
            class="inline-flex items-center gap-2 rounded-full border border-white/10 bg-slate-900/80 px-3 py-2 text-sm font-semibold text-slate-100 shadow-lg backdrop-blur transition hover:-translate-y-[1px]"
            @click="showSettings = !showSettings"
            title="Open options"
          >
            <Settings class="h-5 w-5" />
            Options
          </button>
          <OptionsPopover :open="showSettings" @close="showSettings = false">
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
          </OptionsPopover>
        </div>
      </div>

      <div class="absolute left-1/2 bottom-6 -translate-x-1/2">
        <BottomToolbar
          :active-tool="activeTool"
          :site-count="sites.length"
          @set-tool="setTool"
          @upload="handleImageUpload"
          @undo="popSite"
          @clear="clearSites"
        >
          <template #site-icon>
            <MapPinPlus class="h-5 w-5" />
          </template>
          <template #delete-icon>
            <MapPinMinus class="h-5 w-5" />
          </template>
          <template #move-icon>
            <Move class="h-5 w-5" />
          </template>
          <template #select-icon>
            <MousePointerClick class="h-5 w-5" />
          </template>
          <template #undo-icon>
            <Undo2 class="h-4 w-4" />
          </template>
          <template #clear-icon>
            <Trash2 class="h-4 w-4" />
          </template>
        </BottomToolbar>
      </div>
    </div>

    <SelectionPanel
      :selected-site="selectedSite"
      :multi-selected-count="multiSelectedIds.size"
      @delete="deleteSelectedSite"
      @update="updateSelected"
    >
      <template #trash-icon>
        <Trash2 class="h-4 w-4" />
      </template>
    </SelectionPanel>

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
  </main>
</template>

<script setup>
import { Crosshair, Download, MapPinMinus, MapPinPlus, MousePointerClick, Move, Settings, Trash2, Undo2 } from 'lucide-vue-next'
import BottomToolbar from './components/BottomToolbar.vue'
import ExportMenu from './components/ExportMenu.vue'
import OptionsPopover from './components/OptionsPopover.vue'
import SelectionPanel from './components/SelectionPanel.vue'
import { useVoronoify } from './composables/useVoronoify'

const {
  canvasRef,
  canvasCursor,
  activeTool,
  sites,
  cursorWorld,
  scale,
  selectionBoxStyle,
  selectedSite,
  multiSelectedIds,
  showSettings,
  exportOpen,
  imageBitmap,
  styleOptions,
  handleCanvasClick,
  handleWheel,
  handlePointerDown,
  handlePointerMove,
  handlePointerUp,
  setTool,
  handleImageUpload,
  popSite,
  clearSites,
  deleteSelectedSite,
  updateSelected,
  resetPosition,
  resetZoom,
  exportSitesJson,
  exportImageFile,
  closePopovers,
  movingSiteId,
  moveOffset,
} = useVoronoify()
</script>
