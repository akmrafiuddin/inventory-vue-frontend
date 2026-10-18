<script setup>
import { ref, reactive, watch, onMounted } from 'vue'
import { warehousesApi } from '@/api/warehouses'
import { useToastStore } from '@/stores/toasts'
import { useAuthStore } from '@/stores/auth'

const toast = useToastStore()
const auth = useAuthStore()

const items = ref([])
const meta = ref(null)
const page = ref(1)
const search = ref('')
const loading = ref(false)
const pageError = ref('')

const showForm = ref(false)
const editingId = ref(null)
const form = reactive({ name: '', address: '', is_active: true })
const errors = ref({})
const saving = ref(false)

async function load() {
  loading.value = true
  pageError.value = ''
  try {
    const { data } = await warehousesApi.list({
      page: page.value,
      search: search.value || undefined,
    })
    items.value = data.data
    meta.value = data.meta
  } catch {
    pageError.value = 'Could not load warehouses.'
  } finally {
    loading.value = false
  }
}

let timer
watch(search, () => {
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

function openCreate() {
  editingId.value = null
  Object.assign(form, { name: '', address: '', is_active: true })
  errors.value = {}
  showForm.value = true
}

function openEdit(w) {
  editingId.value = w.id
  Object.assign(form, {
    name: w.name,
    address: w.address ?? '',
    is_active: w.is_active,
  })
  errors.value = {}
  showForm.value = true
}

async function save() {
  saving.value = true
  errors.value = {}

  // send an empty address as null so the database stays clean
  const payload = {
    name: form.name,
    address: form.address || null,
    is_active: form.is_active,
  }

  try {
    if (editingId.value) {
      await warehousesApi.update(editingId.value, payload)
    } else {
      await warehousesApi.create(payload)
    }
    showForm.value = false
    toast.success(editingId.value ? 'Warehouse updated.' : 'Warehouse created.')
    await load()
  } catch (e) {
    if (e.response?.status === 422) {
      errors.value = e.response.data.errors ?? {}
    } else {
      errors.value = { name: ['Something went wrong. Please try again.'] }
    }
  } finally {
    saving.value = false
  }
}

async function toggleActive(w) {
  try {
    await warehousesApi.update(w.id, {
      name: w.name,
      address: w.address ?? null,
      is_active: !w.is_active,
    })
    await load()
    toast.success(w.is_active ? 'Warehouse deactivated.' : 'Warehouse activated.')
  } catch (e) {
    toast.error(e.response?.data?.message ?? 'Update failed.')
  }
}

async function remove(w) {
  if (!confirm(`Delete "${w.name}"?`)) return
  try {
    await warehousesApi.remove(w.id)
    if (items.value.length === 1 && page.value > 1) page.value--
    await load()
    toast.success('Warehouse deleted.')
  } catch (e) {
    toast.error(e.response?.data?.message ?? 'Delete failed.')
  }
}

onMounted(load)
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h1 class="text-xl font-bold tracking-tight text-zinc-900 sm:text-2xl">Warehouses</h1>
        <p class="mt-0.5 text-xs text-zinc-500 sm:text-sm">
          Manage storage facilities, distribution centers, and stock hubs
        </p>
      </div>
      <div v-if="auth.canManage" class="flex items-center gap-2">
        <button
          @click="openCreate"
          class="inline-flex items-center gap-2 rounded-xl bg-zinc-900 px-3.5 py-2 text-xs font-semibold text-white shadow-2xs hover:bg-zinc-800 transition-colors cursor-pointer"
        >
          <svg class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M12 4v16m8-8H4"
            />
          </svg>
          New warehouse
        </button>
      </div>
    </div>

    <!-- Search Toolbar -->
    <div class="rounded-2xl border border-zinc-200/80 bg-white p-3 shadow-2xs">
      <div class="relative max-w-md">
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
          aria-label="Search warehouses by name or address"
          placeholder="Search by name or address..."
          class="w-full rounded-xl border border-zinc-200 bg-zinc-50/50 pl-9 pr-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 focus:border-zinc-900 focus:bg-white focus:outline-none focus:ring-1 focus:ring-zinc-900 shadow-2xs transition-all"
        />
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
              <th scope="col" class="px-4 py-3 sm:px-6">Warehouse Name</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Location Address</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-center">Status</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-right">Actions</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-zinc-100">
            <tr v-if="loading">
              <td colspan="4" class="px-4 py-12 text-center text-xs text-zinc-400">
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
                  Loading warehouses...
                </div>
              </td>
            </tr>
            <tr v-else-if="!items.length">
              <td colspan="4" class="px-4 py-12 text-center text-xs text-zinc-500">
                <div class="mx-auto flex max-w-xs flex-col items-center">
                  <div
                    class="flex h-10 w-10 items-center justify-center rounded-full bg-zinc-100 text-zinc-400 mb-2"
                  >
                    <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="1.5"
                        d="M19 21V5a2 2 0 00-2-2H7a2 2 0 00-2 2v16m14 0h2m-2 0h-5m-9 0H3m2 0h5M9 7h1m-1 4h1m4-4h1m-1 4h1m-5 10v-5a1 1 0 011-1h2a1 1 0 011 1v5m-4 0h4"
                      />
                    </svg>
                  </div>
                  <span class="font-medium text-zinc-800">No warehouses found</span>
                  <span class="text-zinc-400 mt-0.5"
                    >Try searching with another name or add a new warehouse.</span
                  >
                </div>
              </td>
            </tr>
            <tr v-else v-for="w in items" :key="w.id" class="hover:bg-zinc-50/60 transition-colors">
              <td class="px-4 py-3.5 sm:px-6">
                <div class="flex items-center gap-2.5">
                  <div
                    class="flex h-7 w-7 shrink-0 items-center justify-center rounded-lg bg-zinc-100 text-zinc-600"
                  >
                    <svg class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="1.5"
                        d="M8 14v3m4-3v3m4-3v3M3 21h18M3 10h18M3 7l9-4 9 4M4 10h16v11H4V10z"
                      />
                    </svg>
                  </div>
                  <span class="font-semibold text-zinc-900">{{ w.name }}</span>
                </div>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-zinc-600">{{ w.address ?? '—' }}</td>
              <td class="px-4 py-3.5 sm:px-6 text-center">
                <span
                  class="inline-flex items-center rounded-full px-2.5 py-0.5 text-[11px] font-medium"
                  :class="
                    w.is_active
                      ? 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20'
                      : 'bg-zinc-100 text-zinc-600 ring-1 ring-zinc-300/60'
                  "
                >
                  {{ w.is_active ? 'Active' : 'Inactive' }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-right whitespace-nowrap">
                <div v-if="auth.canManage" class="flex items-center justify-end gap-1.5">
                  <button
                    @click="toggleActive(w)"
                    class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-600 hover:bg-zinc-100 transition-colors cursor-pointer"
                  >
                    {{ w.is_active ? 'Deactivate' : 'Activate' }}
                  </button>
                  <button
                    @click="openEdit(w)"
                    class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-100 transition-colors cursor-pointer"
                  >
                    Edit
                  </button>
                  <button
                    @click="remove(w)"
                    class="rounded-lg border border-rose-200 bg-rose-50/50 px-2.5 py-1 text-xs font-medium text-rose-700 hover:bg-rose-100 transition-colors cursor-pointer"
                  >
                    Delete
                  </button>
                </div>
                <span v-else class="text-zinc-400">—</span>
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
            @click="go(meta.current_page - 1)"
            class="rounded-lg border border-zinc-200 bg-white px-3 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-50 disabled:opacity-40 disabled:cursor-not-allowed shadow-2xs transition-colors cursor-pointer"
          >
            Previous
          </button>
          <button
            :disabled="meta.current_page >= meta.last_page"
            @click="go(meta.current_page + 1)"
            class="rounded-lg border border-zinc-200 bg-white px-3 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-50 disabled:opacity-40 disabled:cursor-not-allowed shadow-2xs transition-colors cursor-pointer"
          >
            Next
          </button>
        </div>
      </div>
    </div>

    <!-- Warehouse Form Modal -->
    <div
      v-if="showForm"
      role="dialog"
      aria-modal="true"
      aria-label="Warehouse form"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <form
        @submit.prevent="save"
        class="flex max-h-[90vh] w-full max-w-md flex-col rounded-2xl border border-zinc-200/80 bg-white shadow-xl"
      >
        <div class="flex items-center justify-between border-b border-zinc-100 px-6 py-4">
          <div>
            <h3 class="text-base font-bold text-zinc-900">
              {{ editingId ? 'Edit Warehouse' : 'New Warehouse' }}
            </h3>
            <p class="text-xs text-zinc-500">
              Facility name, physical location, and operations status
            </p>
          </div>
          <button
            type="button"
            @click="showForm = false"
            class="rounded-lg p-1.5 text-zinc-400 hover:bg-zinc-100 hover:text-zinc-600 transition-colors cursor-pointer"
          >
            <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M6 18L18 6M6 6l12 12"
              />
            </svg>
          </button>
        </div>

        <div class="overflow-y-auto p-6 space-y-4">
          <div>
            <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
              >Warehouse Name <span class="text-rose-500">*</span></label
            >
            <input
              v-model="form.name"
              type="text"
              required
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
            />
            <p v-if="errors.name" class="mt-1 text-xs text-rose-600">{{ errors.name[0] }}</p>
          </div>

          <div>
            <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
              >Physical Location Address</label
            >
            <textarea
              v-model="form.address"
              rows="2"
              placeholder="Facility address, city, country..."
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
            ></textarea>
            <p v-if="errors.address" class="mt-1 text-xs text-rose-600">{{ errors.address[0] }}</p>
          </div>

          <label
            class="flex items-center gap-2.5 pt-1 text-xs sm:text-sm text-zinc-700 cursor-pointer"
          >
            <input
              v-model="form.is_active"
              type="checkbox"
              class="h-4 w-4 rounded border-zinc-300 text-zinc-900 focus:ring-zinc-900"
            />
            <span class="font-medium text-zinc-900">Active Warehouse Facility</span>
          </label>
        </div>

        <div
          class="flex items-center justify-end gap-2 border-t border-zinc-100 bg-zinc-50/50 px-6 py-4 rounded-b-2xl"
        >
          <button
            type="button"
            @click="showForm = false"
            class="rounded-xl border border-zinc-200 bg-white px-4 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 shadow-2xs transition-colors cursor-pointer"
          >
            Cancel
          </button>
          <button
            :disabled="saving"
            class="inline-flex items-center gap-2 rounded-xl bg-zinc-900 px-4 py-2 text-xs font-semibold text-white shadow-2xs hover:bg-zinc-800 disabled:opacity-50 transition-colors cursor-pointer"
          >
            <svg
              v-if="saving"
              class="h-4 w-4 animate-spin text-white"
              fill="none"
              viewBox="0 0 24 24"
            >
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
            {{ saving ? 'Saving...' : 'Save Warehouse' }}
          </button>
        </div>
      </form>
    </div>
  </div>
</template>
