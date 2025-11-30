<template>
  <div v-if="selectedSite && multiSelectedCount <= 1" class="pointer-events-none absolute inset-y-0 right-0 flex items-center pr-4">
    <div class="pointer-events-auto w-64 max-w-sm rounded-2xl border border-white/10 bg-slate-900/85 p-4 text-sm text-slate-100 shadow-2xl backdrop-blur">
      <div class="flex items-center justify-between mb-3">
        <div class="font-semibold text-emerald-200">Site #{{ selectedSite.id }}</div>
        <button
          type="button"
          class="rounded-full border border-white/10 bg-white/5 p-2 text-rose-100 transition hover:border-rose-300/60 hover:bg-rose-400/15"
          @click="$emit('delete')"
        >
          <slot name="trash-icon" />
        </button>
      </div>
      <div class="space-y-2">
        <label class="block text-xs uppercase tracking-[0.15em] text-slate-400">X coordinate</label>
        <input
          type="number"
          step="0.1"
          :value="selectedSite.x"
          class="w-full rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1 text-slate-100 outline-none focus:border-emerald-400/60"
          @input="$emit('update', 'x', $event.target.value)"
        />
        <label class="block text-xs uppercase tracking-[0.15em] text-slate-400">Y coordinate</label>
        <input
          type="number"
          step="0.1"
          :value="selectedSite.y"
          class="w-full rounded-lg border border-white/10 bg-slate-950/60 px-2 py-1 text-slate-100 outline-none focus:border-emerald-400/60"
          @input="$emit('update', 'y', $event.target.value)"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  selectedSite: Object,
  multiSelectedCount: {
    type: Number,
    default: 0,
  },
})

defineEmits(['delete', 'update'])
</script>
