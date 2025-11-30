<template>
  <div class="relative">
    <button
      type="button"
      class="inline-flex items-center gap-2 rounded-full border border-white/10 bg-slate-900/80 px-3 py-2 text-sm font-semibold text-slate-100 shadow-lg backdrop-blur transition hover:-translate-y-[1px]"
      @click="$emit('toggle')"
      title="Export"
    >
      <slot name="icon" />
      Export
    </button>
    <div
      v-if="open"
      class="absolute left-0 top-full mt-2 w-52 rounded-2xl border border-white/10 bg-slate-900/95 p-2 text-sm text-slate-100 shadow-xl backdrop-blur"
    >
      <button
        type="button"
        class="flex w-full items-center gap-2 rounded-lg px-3 py-2 text-left transition hover:bg-white/10"
        @click="$emit('export-json')"
      >
        Export sites (JSON)
      </button>
      <button
        type="button"
        class="flex w-full items-center gap-2 rounded-lg px-3 py-2 text-left transition hover:bg-white/10 disabled:opacity-40 disabled:hover:bg-transparent"
        :disabled="!imageAvailable"
        @click="$emit('export-image')"
      >
        Export image (PNG)
      </button>
    </div>
  </div>
</template>

<script setup>
defineProps({
  open: Boolean,
  imageAvailable: Boolean,
})

defineEmits(['toggle', 'export-json', 'export-image'])
</script>
