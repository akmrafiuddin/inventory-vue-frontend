<script setup>
import { ref, reactive, watch, onMounted } from 'vue'
import { customersApi } from '@/api/customers'
import { useToastStore } from '@/stores/toasts'
import { useAuthStore } from '@/stores/auth'

const toast = useToastStore()
const auth = useAuthStore()

const items = ref([])
const meta = ref(null)
const page = ref(1)
const search = ref('')
const activeFilter = ref('')
const loading = ref(false)
const pageError = ref('')

const showForm = ref(false)
const editingId = ref(null)
const form = reactive({
  name: '',
  company_name: '',
  email: '',
  phone: '',
  address: '',
  tax_number: '',
  is_active: true,
})
const errors = ref({})
const saving = ref(false)

async function load() {
  loading.value = true
  pageError.value = ''
  try {
    const { data } = await customersApi.list({
      page: page.value,
      search: search.value || undefined,
      is_active: activeFilter.value === '' ? undefined : activeFilter.value,
    })
    items.value = data.data
    meta.value = data.meta
  } catch {
    pageError.value = 'Could not load customers.'
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

watch(activeFilter, () => {
  page.value = 1
  load()
})

function go(p) {
  page.value = p
  load()
}

function resetForm() {
  Object.assign(form, {
    name: '',
    company_name: '',
    email: '',
    phone: '',
    address: '',
    tax_number: '',
    is_active: true,
  })
  errors.value = {}
}

function openCreate() {
  editingId.value = null
  resetForm()
  showForm.value = true
}

function openEdit(c) {
  editingId.value = c.id
  resetForm()
  Object.assign(form, {
    name: c.name,
    company_name: c.company_name ?? '',
    email: c.email ?? '',
    phone: c.phone ?? '',
    address: c.address ?? '',
    tax_number: c.tax_number ?? '',
    is_active: c.is_active,
  })
  showForm.value = true
}

async function save() {
  saving.value = true
  errors.value = {}

  const payload = {
    name: form.name,
    company_name: form.company_name || null,
    email: form.email || null,
    phone: form.phone || null,
    address: form.address || null,
    tax_number: form.tax_number || null,
    is_active: form.is_active,
  }

  try {
    if (editingId.value) {
      await customersApi.update(editingId.value, payload)
    } else {
      await customersApi.create(payload)
    }
    showForm.value = false
    toast.success(editingId.value ? 'Customer updated.' : 'Customer created.')
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

async function remove(c) {
  if (!confirm(`Delete "${c.name}"?`)) return
  try {
    await customersApi.remove(c.id)
    if (items.value.length === 1 && page.value > 1) page.value--
    await load()
    toast.success('Customer deleted.')
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
        <h1 class="text-xl font-bold tracking-tight text-zinc-900 sm:text-2xl">Customers</h1>
        <p class="mt-0.5 text-xs text-zinc-500 sm:text-sm">
          Manage customer profiles, billing addresses, and order histories
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
          New customer
        </button>
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
            aria-label="Search customers"
            placeholder="Search by name, company, email or phone..."
            class="w-full rounded-xl border border-zinc-200 bg-zinc-50/50 pl-9 pr-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 focus:border-zinc-900 focus:bg-white focus:outline-none focus:ring-1 focus:ring-zinc-900 shadow-2xs transition-all"
          />
        </div>
        <div class="flex items-center gap-2 sm:gap-3">
          <select
            v-model="activeFilter"
            aria-label="Filter by status"
            class="rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs sm:text-sm text-zinc-700 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          >
            <option value="">All statuses</option>
            <option value="1">Active</option>
            <option value="0">Inactive</option>
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
              <th scope="col" class="px-4 py-3 sm:px-6">Customer</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Company</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Email</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Phone</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-center">Orders</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-center">Status</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-right">Actions</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-zinc-100">
            <tr v-if="loading">
              <td colspan="7" class="px-4 py-12 text-center text-xs text-zinc-400">
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
                  Loading customers...
                </div>
              </td>
            </tr>
            <tr v-else-if="!items.length">
              <td colspan="7" class="px-4 py-12 text-center text-xs text-zinc-500">
                <div class="mx-auto flex max-w-xs flex-col items-center">
                  <div
                    class="flex h-10 w-10 items-center justify-center rounded-full bg-zinc-100 text-zinc-400 mb-2"
                  >
                    <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="1.5"
                        d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0M15 7a3 3 0 11-6 0 3 3 0 016 0zm6 3a2 2 0 11-4 0 2 2 0 014 0zM7 10a2 2 0 11-4 0 2 2 0 014 0z"
                      />
                    </svg>
                  </div>
                  <span class="font-medium text-zinc-800">No customers found</span>
                  <span class="text-zinc-400 mt-0.5"
                    >Try searching with a different keyword or create one.</span
                  >
                </div>
              </td>
            </tr>
            <tr v-else v-for="c in items" :key="c.id" class="hover:bg-zinc-50/60 transition-colors">
              <td class="px-4 py-3.5 sm:px-6">
                <div class="flex items-center gap-2.5">
                  <div
                    class="flex h-7 w-7 shrink-0 items-center justify-center rounded-full bg-zinc-100 text-xs font-semibold text-zinc-700"
                  >
                    {{ (c.name || 'C').charAt(0).toUpperCase() }}
                  </div>
                  <span class="font-semibold text-zinc-900">{{ c.name }}</span>
                </div>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-zinc-600">{{ c.company_name ?? '—' }}</td>
              <td class="px-4 py-3.5 sm:px-6 text-zinc-600">{{ c.email ?? '—' }}</td>
              <td class="px-4 py-3.5 sm:px-6 text-zinc-600 font-mono text-xs">
                {{ c.phone ?? '—' }}
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-center text-zinc-600">
                <span
                  class="inline-flex items-center rounded-md bg-zinc-100 px-2 py-0.5 text-xs font-medium text-zinc-700"
                >
                  {{ c.sales_orders_count ?? 0 }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-center">
                <span
                  class="inline-flex items-center rounded-full px-2.5 py-0.5 text-[11px] font-medium"
                  :class="
                    c.is_active
                      ? 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20'
                      : 'bg-zinc-100 text-zinc-600 ring-1 ring-zinc-300/60'
                  "
                >
                  {{ c.is_active ? 'Active' : 'Inactive' }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-right whitespace-nowrap">
                <div v-if="auth.canManage" class="flex items-center justify-end gap-1.5">
                  <button
                    @click="openEdit(c)"
                    class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-100 transition-colors cursor-pointer"
                  >
                    Edit
                  </button>
                  <button
                    @click="remove(c)"
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

    <!-- Customer Form Modal -->
    <div
      v-if="showForm"
      role="dialog"
      aria-modal="true"
      aria-label="Customer form"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <form
        @submit.prevent="save"
        class="flex max-h-[90vh] w-full max-w-lg flex-col rounded-2xl border border-zinc-200/80 bg-white shadow-xl"
      >
        <div class="flex items-center justify-between border-b border-zinc-100 px-6 py-4">
          <div>
            <h3 class="text-base font-bold text-zinc-900">
              {{ editingId ? 'Edit Customer' : 'New Customer' }}
            </h3>
            <p class="text-xs text-zinc-500">
              Enter customer details, contact info, and tax registration
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
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
                >Full Name <span class="text-rose-500">*</span></label
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
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Company Name</label>
              <input
                v-model="form.company_name"
                type="text"
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              />
              <p v-if="errors.company_name" class="mt-1 text-xs text-rose-600">
                {{ errors.company_name[0] }}
              </p>
            </div>
          </div>

          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Email Address</label>
              <input
                v-model="form.email"
                type="email"
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              />
              <p v-if="errors.email" class="mt-1 text-xs text-rose-600">{{ errors.email[0] }}</p>
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Phone</label>
              <input
                v-model="form.phone"
                type="text"
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              />
              <p v-if="errors.phone" class="mt-1 text-xs text-rose-600">{{ errors.phone[0] }}</p>
            </div>
          </div>

          <div>
            <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Tax / VAT ID</label>
            <input
              v-model="form.tax_number"
              type="text"
              placeholder="e.g. VAT-9281734"
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
            />
            <p v-if="errors.tax_number" class="mt-1 text-xs text-rose-600">
              {{ errors.tax_number[0] }}
            </p>
          </div>

          <div>
            <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
              >Billing / Shipping Address</label
            >
            <textarea
              v-model="form.address"
              rows="2"
              placeholder="Street address, city, state, postal code..."
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
            <span class="font-medium text-zinc-900">Active Customer</span>
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
            {{ saving ? 'Saving...' : 'Save Customer' }}
          </button>
        </div>
      </form>
    </div>
  </div>
</template>
