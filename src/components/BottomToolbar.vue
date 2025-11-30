<template>
  <div class="pointer-events-auto flex items-center gap-2.5 rounded-full border border-white/10 bg-slate-900/80 p-2 shadow-2xl shadow-emerald-500/10 backdrop-blur">
    <div class="flex h-10 overflow-hidden rounded-full border border-white/10 bg-white/5">
      <button
        type="button"
        class="flex h-full items-center gap-2 rounded-l-full px-3 text-sm font-semibold transition hover:translate-y-[-1px]"
        :class="activeTool === 'sites' ? 'bg-emerald-500/90 text-slate-950 shadow-lg shadow-emerald-500/40' : 'text-slate-100'"
        :aria-pressed="activeTool === 'sites'"
        @click="$emit('set-tool', 'sites')"
        title="Sites tool (add points)"
      >
        <slot name="site-icon" />
      </button>
      <button
        type="button"
        class="flex h-full items-center gap-2 px-3 text-sm font-semibold transition hover:translate-y-[-1px]"
        :class="activeTool === 'paint-delete' ? 'bg-emerald-500/90 text-slate-950 shadow-lg shadow-emerald-500/40' : 'text-slate-100'"
        :aria-pressed="activeTool === 'paint-delete'"
        @click="$emit('set-tool', 'paint-delete')"
        title="Paint delete tool"
      >
        <slot name="delete-icon" />
      </button>
      <button
        type="button"
        class="flex h-full items-center gap-2 px-3 text-sm font-semibold transition hover:translate-y-[-1px]"
        :class="activeTool === 'move' ? 'bg-emerald-500/90 text-slate-950 shadow-lg shadow-emerald-500/40' : 'text-slate-100'"
        :aria-pressed="activeTool === 'move'"
        @click="$emit('set-tool', 'move')"
        title="Move tool"
      >
        <slot name="move-icon" />
      </button>
      <button
        type="button"
        class="flex h-full items-center gap-2 rounded-r-full px-3 text-sm font-semibold transition hover:translate-y-[-1px]"
        :class="activeTool === 'select' ? 'bg-emerald-500/90 text-slate-950 shadow-lg shadow-emerald-500/40' : 'text-slate-100'"
        :aria-pressed="activeTool === 'select'"
        @click="$emit('set-tool', 'select')"
        title="Select / multi-select (Shift-drag) tool"
      >
        <slot name="select-icon" />
      </button>
    </div>
    <span class="text-sm text-slate-100/90">{{ sitesLabel }}</span>
    <label class="flex h-10 items-center gap-2 rounded-full border border-white/10 bg-white/5 px-3 text-sm font-semibold text-slate-100 shadow-none transition hover:translate-y-[-1px] hover:bg-white/10 cursor-pointer">
      <input type="file" accept="image/*" class="hidden" @change="$emit('upload', $event)" />
      Upload image
    </label>
    <div class="flex items-center gap-1">
      <button
        type="button"
        class="flex h-10 w-10 items-center justify-center rounded-full border border-white/10 bg-white/5 text-slate-100 transition hover:border-white/30 hover:bg-white/10"
        @click="$emit('undo')"
        aria-label="Undo last site"
        title="Undo last site"
      >
        <slot name="undo-icon" />
      </button>
      <button
        type="button"
        class="flex h-10 w-10 items-center justify-center rounded-full border border-white/10 bg-white/5 text-rose-100 transition hover:border-rose-300/60 hover:bg-rose-400/15"
        @click="$emit('clear')"
        aria-label="Clear all sites"
        title="Clear all sites"
      >
        <slot name="clear-icon" />
      </button>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  activeTool: {
    type: String,
    required: true,
  },
  siteCount: {
    type: Number,
    required: true,
  },
})

defineEmits(['set-tool', 'upload', 'undo', 'clear'])

const sitesLabel = computed(() => {
  const c = props.siteCount
  if (!c) return 'No sites'
  if (c === 1) return '1 site'
  return `${c} sites`
})
</script>
