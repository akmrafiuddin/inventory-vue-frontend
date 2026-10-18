<script setup>
import { ref, watch, onMounted } from 'vue'
import { activityLogsApi } from '@/api/activityLogs'

const items = ref([])
const meta = ref(null)
const page = ref(1)
const search = ref('')
const actionFilter = ref('')
const loading = ref(false)
const pageError = ref('')

const actionStyles = {
  created: 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20',
  updated: 'bg-blue-50 text-blue-700 ring-1 ring-blue-600/20',
  deleted: 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20',
  force_deleted: 'bg-rose-100 text-rose-800 ring-1 ring-rose-600/30',
  restored: 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20',
}

const actionLabels = {
  created: 'Created',
  updated: 'Updated',
  deleted: 'Deleted',
  force_deleted: 'Deleted forever',
  restored: 'Restored',
}

function formatDate(value) {
  if (!value) return '—'
  const date = new Date(value)
  return date.toLocaleString()
}

async function load() {
  loading.value = true
  pageError.value = ''
  try {
    const { data } = await activityLogsApi.list({
      page: page.value,
      search: search.value || undefined,
      action: actionFilter.value || undefined,
    })
    items.value = data.data
    meta.value = data.meta
  } catch {
    pageError.value = 'Could not load the activity log.'
  } finally {
    loading.value = false
  }
}

let timer
watch([search, actionFilter], () => {
  clearTimeout(timer)
  timer = setTimeout(() => {
    page.value = 1
    load()
  }, 300)
})

function go(p) {
  page.value = p
  load()
}

onMounted(load)
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="flex flex-col gap-1 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h1 class="text-xl font-bold tracking-tight text-zinc-900 sm:text-2xl">Activity Log</h1>
        <p class="mt-0.5 text-xs text-zinc-500 sm:text-sm">
          Audit trail of system modifications, stock changes, and operator actions
        </p>
      </div>
    </div>

    <!-- Filters & Search Toolbar -->
    <div class="rounded-2xl border border-zinc-200/80 bg-white p-3 shadow-2xs">
      <div class="flex flex-col gap-3 sm:flex-row sm:items-center">
        <div class="relative flex-1">
          <svg
            class="pointer-events-none absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-zinc-400"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
            />
          </svg>
          <input
            v-model="search"
            type="text"
            aria-label="Search the activity description"
            placeholder="Search activity description..."
            class="w-full rounded-xl border border-zinc-200 bg-zinc-50/50 pl-9 pr-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 focus:border-zinc-900 focus:bg-white focus:outline-none focus:ring-1 focus:ring-zinc-900 shadow-2xs transition-all"
          />
        </div>
        <div class="flex items-center gap-2 sm:gap-3">
          <select
            v-model="actionFilter"
            aria-label="Filter by action"
            class="rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs sm:text-sm text-zinc-700 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          >
            <option value="">All actions</option>
            <option value="created">Created</option>
            <option value="updated">Updated</option>
            <option value="deleted">Deleted</option>
            <option value="restored">Restored</option>
            <option value="force_deleted">Deleted forever</option>
          </select>
        </div>
      </div>
    </div>

    <!-- Error Alert -->
    <div
      v-if="pageError"
      class="rounded-xl border border-rose-200 bg-rose-50/80 p-4 text-xs sm:text-sm text-rose-800 flex items-center justify-between"
    >
      <div class="flex items-center gap-2">
        <svg
          class="h-4 w-4 shrink-0 text-rose-600"
          fill="none"
          viewBox="0 0 24 24"
          stroke="currentColor"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
          />
        </svg>
        <span>{{ pageError }}</span>
      </div>
      <button
        @click="load"
        class="text-xs font-semibold underline hover:no-underline text-rose-800"
      >
        Retry
      </button>
    </div>

    <!-- Data Table Card -->
    <div class="overflow-hidden rounded-2xl border border-zinc-200/80 bg-white shadow-2xs">
      <div class="overflow-x-auto">
        <table class="w-full text-left text-xs sm:text-sm">
          <thead
            class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
          >
            <tr>
              <th scope="col" class="px-4 py-3 sm:px-6">Timestamp</th>
              <th scope="col" class="px-4 py-3 sm:px-6">User</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-center">Action</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Description</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-right font-mono">IP Address</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-zinc-100">
            <tr v-if="loading">
              <td colspan="5" class="px-4 py-12 text-center text-xs text-zinc-400">
                <div class="inline-flex items-center gap-2">
                  <svg class="h-4 w-4 animate-spin text-zinc-400" fill="none" viewBox="0 0 24 24">
                    <circle
                      class="opacity-25"
                      cx="12"
                      cy="12"
                      r="10"
                      stroke="currentColor"
                      stroke-width="4"
                    ></circle>
                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                  </svg>
                  Loading activity logs...
                </div>
              </td>
            </tr>
            <tr v-else-if="!items.length">
              <td colspan="5" class="px-4 py-12 text-center text-xs text-zinc-500">
                <div class="mx-auto flex max-w-xs flex-col items-center">
                  <div
                    class="flex h-10 w-10 items-center justify-center rounded-full bg-zinc-100 text-zinc-400 mb-2"
                  >
                    <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="1.5"
                        d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"
                      />
                    </svg>
                  </div>
                  <span class="font-medium text-zinc-800">No activity recorded</span>
                  <span class="text-zinc-400 mt-0.5"
                    >Events and audit updates will appear here automatically.</span
                  >
                </div>
              </td>
            </tr>
            <tr
              v-else
              v-for="log in items"
              :key="log.id"
              class="hover:bg-zinc-50/60 transition-colors"
            >
              <td class="whitespace-nowrap px-4 py-3.5 sm:px-6 text-xs text-zinc-500 font-medium">
                {{ formatDate(log.created_at) }}
              </td>
              <td class="px-4 py-3.5 sm:px-6">
                <div class="flex items-center gap-2">
                  <div
                    class="flex h-6 w-6 shrink-0 items-center justify-center rounded-full bg-zinc-100 text-[10px] font-semibold text-zinc-700"
                  >
                    {{ (log.user_name || 'U').charAt(0).toUpperCase() }}
                  </div>
                  <span class="font-semibold text-zinc-900">{{ log.user_name }}</span>
                </div>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-center">
                <span
                  class="inline-flex items-center rounded-full px-2.5 py-0.5 text-[11px] font-medium"
                  :class="actionStyles[log.action] ?? 'bg-zinc-100 text-zinc-700'"
                >
                  {{ actionLabels[log.action] ?? log.action }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-zinc-700">{{ log.description }}</td>
              <td class="px-4 py-3.5 sm:px-6 text-right font-mono text-xs text-zinc-400">
                {{ log.ip_address ?? '—' }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Pagination -->
      <div
        v-if="meta && meta.last_page > 1"
        class="flex items-center justify-between border-t border-zinc-200/60 bg-zinc-50/50 px-4 py-3 sm:px-6 text-xs sm:text-sm"
      >
        <span class="text-zinc-500">
          Page <span class="font-medium text-zinc-800">{{ meta.current_page }}</span> of
          <span class="font-medium text-zinc-800">{{ meta.last_page }}</span> ({{ meta.total }}
          total)
        </span>
        <div class="flex items-center gap-1.5">
          <button
            :disabled="meta.current_page <= 1"
            class="rounded-lg border border-zinc-200 bg-white px-3 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-50 disabled:opacity-40 disabled:cursor-not-allowed shadow-2xs transition-colors cursor-pointer"
            @click="go(meta.current_page - 1)"
          >
            Previous
          </button>
          <button
            :disabled="meta.current_page >= meta.last_page"
            class="rounded-lg border border-zinc-200 bg-white px-3 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-50 disabled:opacity-40 disabled:cursor-not-allowed shadow-2xs transition-colors cursor-pointer"
            @click="go(meta.current_page + 1)"
          >
            Next
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
