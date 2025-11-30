<template>
  <div
    v-if="open"
    class="origin-top-right translate-y-2 rounded-2xl border border-white/10 bg-slate-900/95 p-4 text-sm text-slate-100 shadow-2xl backdrop-blur min-w-[360px] max-w-[440px]"
  >
    <div class="flex items-center justify-between mb-3">
      <h2 class="text-base font-semibold text-white">Canvas options</h2>
      <button
        type="button"
        class="rounded-full border border-white/10 bg-white/5 px-3 py-1 text-xs font-semibold text-slate-100 transition hover:border-white/30 hover:bg-white/10"
        @click="$emit('close')"
      >
        Close
      </button>
    </div>
    <div class="grid grid-cols-1 gap-3">
      <slot />
    </div>
  </div>
</template>

<script setup>
defineProps({
  open: Boolean,
})
defineEmits(['close'])
</script>
