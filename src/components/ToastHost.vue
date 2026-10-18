<script setup>
import { useToastStore } from '@/stores/toasts'

const toasts = useToastStore()

const badgeStyles = {
  success: 'bg-emerald-500',
  error: 'bg-rose-500',
  info: 'bg-zinc-800',
}
</script>

<template>
  <div
    aria-live="polite"
    aria-atomic="false"
    class="pointer-events-none fixed right-4 top-4 z-50 flex w-80 max-w-[calc(100vw-2rem)] flex-col gap-2.5"
  >
    <div
      v-for="t in toasts.items"
      :key="t.id"
      :role="t.type === 'error' ? 'alert' : 'status'"
      class="pointer-events-auto flex items-center justify-between gap-3 rounded-xl border border-zinc-200/90 bg-white px-3.5 py-3 text-sm text-zinc-800 shadow-lg shadow-zinc-900/5 transition-all"
    >
      <div class="flex items-center gap-2.5 min-w-0">
        <span
          class="h-2 w-2 shrink-0 rounded-full"
          :class="badgeStyles[t.type] ?? badgeStyles.info"
        ></span>
        <span class="truncate font-medium text-xs sm:text-sm text-zinc-900">{{ t.message }}</span>
      </div>
      <button
        type="button"
        aria-label="Dismiss notification"
        class="shrink-0 rounded-md p-1 text-zinc-400 hover:bg-zinc-100 hover:text-zinc-700 transition-colors"
        @click="toasts.remove(t.id)"
      >
        <svg
          class="h-3.5 w-3.5"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2.2"
          stroke-linecap="round"
        >
          <line x1="18" y1="6" x2="6" y2="18"></line>
          <line x1="6" y1="6" x2="18" y2="18"></line>
        </svg>
      </button>
    </div>
  </div>
</template>
