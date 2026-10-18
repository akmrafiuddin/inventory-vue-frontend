<script setup>
import { ref, reactive, computed, watch, onMounted } from 'vue'
import { purchaseOrdersApi } from '@/api/purchaseOrders'
import { suppliersApi } from '@/api/suppliers'
import { warehousesApi } from '@/api/warehouses'
import { productsApi } from '@/api/products'
import { useToastStore } from '@/stores/toasts'
import { useAuthStore } from '@/stores/auth'

const toast = useToastStore()
const auth = useAuthStore()

const items = ref([])
const meta = ref(null)
const page = ref(1)
const search = ref('')
const supplierFilter = ref('')
const statusFilter = ref('')
const loading = ref(false)
const pageError = ref('')

const suppliers = ref([])
const warehouses = ref([])
const allProducts = ref([])
const linkProducts = ref([])

const statusStyles = {
  draft: 'bg-zinc-100 text-zinc-700 ring-1 ring-zinc-300/60',
  received: 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20',
  cancelled: 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20',
  partially_received: 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20',
}

const statusLabels = {
  draft: 'Draft',
  received: 'Received',
  cancelled: 'Cancelled',
  partially_received: 'Partial',
}

function money(value) {
  return `$${Number(value ?? 0).toFixed(2)}`
}

// product options for the line rows: the chosen supplier's linked products
// when available, otherwise every product.
const productOptions = computed(() => {
  const source = linkProducts.value.length ? linkProducts.value : allProducts.value
  return source.map((p) => ({ id: p.id, name: p.name, sku: p.sku, unit_cost: p.unit_cost ?? null }))
})

// ---- create / edit modal ----
const showForm = ref(false)
const editingId = ref(null)
const form = reactive({ supplier_id: '', expected_at: '', notes: '', items: [] })
const errors = ref({})
const saving = ref(false)

// live form subtotal
const formTotal = computed(() => {
  return form.items.reduce((sum, line) => {
    const qty = Number(line.quantity) || 0
    const cost = Number(line.unit_cost) || 0
    return sum + qty * cost
  }, 0)
})

// ---- receive modal ----
const showReceive = ref(false)
const receiveTarget = ref(null)
const receiveForm = reactive({ warehouse_id: '', update_cost: false })
const receiveLines = ref([])
const receiveErrors = ref({})
const savingReceive = ref(false)

const receiveErrorList = computed(() => Object.values(receiveErrors.value).flat())

async function load() {
  loading.value = true
  pageError.value = ''
  try {
    const { data } = await purchaseOrdersApi.list({
      page: page.value,
      search: search.value || undefined,
      supplier_id: supplierFilter.value || undefined,
      status: statusFilter.value || undefined,
    })
    items.value = data.data
    meta.value = data.meta
  } catch {
    pageError.value = 'Could not load purchase orders.'
  } finally {
    loading.value = false
  }
}

async function loadLookups() {
  const [s, w, p] = await Promise.all([suppliersApi.all(), warehousesApi.all(), productsApi.all()])
  suppliers.value = s.data.data
  warehouses.value = w.data.data
  allProducts.value = p.data.data
}

let timer
watch(search, () => {
  clearTimeout(timer)
  timer = setTimeout(() => {
    page.value = 1
    load()
  }, 300)
})

watch([supplierFilter, statusFilter], () => {
  page.value = 1
  load()
})

function go(p) {
  page.value = p
  load()
}

function emptyLine() {
  return { product_id: '', quantity: 1, unit_cost: '' }
}

function resetForm() {
  Object.assign(form, { supplier_id: '', expected_at: '', notes: '', items: [emptyLine()] })
  linkProducts.value = []
  errors.value = {}
}

function onSupplierChange() {
  linkProducts.value = []
  if (!form.supplier_id) return
  suppliersApi.products(form.supplier_id).then(({ data }) => {
    linkProducts.value = data.data
  })
}

function onProductChange(line) {
  const opt = productOptions.value.find((o) => o.id === Number(line.product_id))
  if (opt && opt.unit_cost != null) line.unit_cost = opt.unit_cost
}

function addLine() {
  form.items.push(emptyLine())
}

function removeLine(index) {
  form.items.splice(index, 1)
  if (!form.items.length) form.items.push(emptyLine())
}

async function openCreate() {
  editingId.value = null
  resetForm()
  showForm.value = true
}

async function openEdit(po) {
  editingId.value = po.id
  resetForm()
  const { data } = await purchaseOrdersApi.show(po.id)
  const d = data.data
  form.supplier_id = d.supplier_id
  form.expected_at = d.expected_at ? d.expected_at.substring(0, 10) : ''
  form.notes = d.notes ?? ''
  form.items = (d.items ?? []).map((i) => ({
    product_id: i.product_id,
    quantity: i.quantity,
    unit_cost: i.unit_cost,
  }))
  await onSupplierChange()
  showForm.value = true
}

async function save() {
  saving.value = true
  errors.value = {}

  const payload = {
    supplier_id: Number(form.supplier_id),
    expected_at: form.expected_at || null,
    notes: form.notes || null,
    items: form.items
      .filter((l) => l.product_id)
      .map((l) => ({
        product_id: Number(l.product_id),
        quantity: Number(l.quantity),
        unit_cost: l.unit_cost === '' ? 0 : Number(l.unit_cost),
      })),
  }

  try {
    if (editingId.value) {
      await purchaseOrdersApi.update(editingId.value, payload)
    } else {
      await purchaseOrdersApi.create(payload)
    }
    showForm.value = false
    toast.success(editingId.value ? 'Purchase order updated.' : 'Purchase order created.')
    await load()
  } catch (e) {
    if (e.response?.status === 422) {
      errors.value = e.response.data.errors ?? {}
    } else {
      errors.value = { supplier_id: ['Something went wrong. Please try again.'] }
    }
  } finally {
    saving.value = false
  }
}

async function openReceive(po) {
  receiveTarget.value = po
  receiveForm.warehouse_id = ''
  receiveForm.update_cost = false
  receiveLines.value = []
  receiveErrors.value = {}
  showReceive.value = true

  try {
    const { data } = await purchaseOrdersApi.show(po.id)
    receiveLines.value = (data.data.items ?? []).map((i) => ({
      id: i.id,
      product_name: i.product_name,
      product_sku: i.product_sku,
      ordered: i.quantity,
      received: i.quantity_received,
      remaining: i.quantity_remaining,
      quantity: i.quantity_remaining,
    }))
  } catch {
    receiveErrors.value = { warehouse_id: ['Could not load the order lines.'] }
  }
}

function receiveAll() {
  receiveLines.value.forEach((line) => {
    line.quantity = line.remaining
  })
}

async function submitReceive() {
  savingReceive.value = true
  receiveErrors.value = {}
  try {
    await purchaseOrdersApi.receive(receiveTarget.value.id, {
      warehouse_id: Number(receiveForm.warehouse_id),
      update_cost: receiveForm.update_cost,
      items: receiveLines.value.map((l) => ({ id: l.id, quantity: Number(l.quantity) || 0 })),
    })
    showReceive.value = false
    toast.success('Stock received.')
    await load()
  } catch (e) {
    receiveErrors.value = { warehouse_id: ['Something went wrong. Please try again.'] }

    if (e.response?.status === 422) {
      receiveErrors.value = e.response.data.errors ?? {}
      // the backend reports quantity problems under a custom "message" key
      if (!Object.keys(receiveErrors.value).length && e.response.data.message) {
        receiveErrors.value = { items: [e.response.data.message] }
      }
    }
  } finally {
    savingReceive.value = false
  }
}

async function cancelPo(po) {
  if (!confirm(`Cancel ${po.number}?`)) return
  try {
    await purchaseOrdersApi.cancel(po.id)
    await load()
    toast.success(`${po.number} cancelled.`)
  } catch (e) {
    toast.error(e.response?.data?.message ?? 'Cancel failed.')
  }
}

async function remove(po) {
  if (!confirm(`Delete ${po.number}?`)) return
  try {
    await purchaseOrdersApi.remove(po.id)
    if (items.value.length === 1 && page.value > 1) page.value--
    await load()
    toast.success(`${po.number} deleted.`)
  } catch (e) {
    toast.error(e.response?.data?.message ?? 'Delete failed.')
  }
}

onMounted(() => {
  load()
  loadLookups()
})
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h1 class="text-xl font-bold tracking-tight text-zinc-900 sm:text-2xl">Purchase Orders</h1>
        <p class="mt-0.5 text-xs text-zinc-500 sm:text-sm">
          Manage vendor procurement, expected shipments, and inventory intake
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
          New purchase order
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
            aria-label="Search purchase orders by number"
            placeholder="Search by PO number..."
            class="w-full rounded-xl border border-zinc-200 bg-zinc-50/50 pl-9 pr-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 focus:border-zinc-900 focus:bg-white focus:outline-none focus:ring-1 focus:ring-zinc-900 shadow-2xs transition-all"
          />
        </div>
        <div class="flex flex-wrap items-center gap-2 sm:gap-3">
          <select
            v-model="supplierFilter"
            aria-label="Filter by supplier"
            class="rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs sm:text-sm text-zinc-700 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          >
            <option value="">All suppliers</option>
            <option v-for="s in suppliers" :key="s.id" :value="s.id">{{ s.name }}</option>
          </select>
          <select
            v-model="statusFilter"
            aria-label="Filter by status"
            class="rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs sm:text-sm text-zinc-700 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          >
            <option value="">All statuses</option>
            <option value="draft">Draft</option>
            <option value="partially_received">Partial</option>
            <option value="received">Received</option>
            <option value="cancelled">Cancelled</option>
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
              <th scope="col" class="px-4 py-3 sm:px-6">Number</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Supplier</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-center">Status</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-center">Items</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-right">Total</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Expected</th>
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
                  Loading purchase orders...
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
                        d="M16 11V7a4 4 0 00-8 0v4M5 9h14l1 12H4L5 9z"
                      />
                    </svg>
                  </div>
                  <span class="font-medium text-zinc-800">No purchase orders found</span>
                  <span class="text-zinc-400 mt-0.5"
                    >Try adjusting your filters or create a purchase order.</span
                  >
                </div>
              </td>
            </tr>
            <tr
              v-else
              v-for="po in items"
              :key="po.id"
              class="hover:bg-zinc-50/60 transition-colors"
            >
              <td class="px-4 py-3.5 sm:px-6">
                <span class="font-mono text-xs font-semibold text-zinc-900">{{ po.number }}</span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 font-medium text-zinc-900">
                {{ po.supplier?.name ?? '—' }}
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-center">
                <span
                  class="inline-flex items-center rounded-full px-2.5 py-0.5 text-[11px] font-medium"
                  :class="statusStyles[po.status] || 'bg-zinc-100 text-zinc-700'"
                >
                  {{ statusLabels[po.status] ?? po.status }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-center text-zinc-600">
                <span
                  class="inline-flex items-center rounded-md bg-zinc-100 px-2 py-0.5 text-xs font-medium text-zinc-700"
                >
                  {{ po.items_count }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-right font-medium text-zinc-900">
                {{ money(po.total) }}
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-zinc-600">
                {{ po.expected_at ? po.expected_at.substring(0, 10) : '—' }}
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-right whitespace-nowrap">
                <div class="flex items-center justify-end gap-1.5">
                  <RouterLink
                    :to="{ name: 'purchase-order-print', params: { id: po.id } }"
                    target="_blank"
                    class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-600 hover:bg-zinc-100 hover:text-zinc-900 transition-colors cursor-pointer"
                  >
                    Print
                  </RouterLink>

                  <button
                    v-if="po.status === 'draft' || po.status === 'partially_received'"
                    @click="openReceive(po)"
                    class="rounded-lg border border-emerald-200 bg-emerald-50 px-2.5 py-1 text-xs font-semibold text-emerald-700 hover:bg-emerald-100 transition-colors cursor-pointer"
                  >
                    Receive
                  </button>

                  <template v-if="auth.canManage">
                    <button
                      v-if="po.status === 'draft' || po.status === 'partially_received'"
                      @click="cancelPo(po)"
                      class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-500 hover:bg-rose-50 hover:text-rose-700 hover:border-rose-200 transition-colors cursor-pointer"
                    >
                      Cancel
                    </button>
                    <template v-if="po.status === 'draft'">
                      <button
                        @click="openEdit(po)"
                        class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-100 transition-colors cursor-pointer"
                      >
                        Edit
                      </button>
                      <button
                        @click="remove(po)"
                        class="rounded-lg border border-rose-200 bg-rose-50/50 px-2.5 py-1 text-xs font-medium text-rose-700 hover:bg-rose-100 transition-colors cursor-pointer"
                      >
                        Delete
                      </button>
                    </template>
                  </template>
                </div>
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

    <!-- Create / Edit Modal -->
    <div
      v-if="showForm"
      role="dialog"
      aria-modal="true"
      aria-label="Purchase order form"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <form
        @submit.prevent="save"
        class="flex max-h-[90vh] w-full max-w-3xl flex-col rounded-2xl border border-zinc-200/80 bg-white shadow-xl"
      >
        <div class="flex items-center justify-between border-b border-zinc-100 px-6 py-4">
          <div>
            <h3 class="text-base font-bold text-zinc-900">
              {{ editingId ? 'Edit Purchase Order' : 'New Purchase Order' }}
            </h3>
            <p class="text-xs text-zinc-500">
              Select supplier, expected delivery date, and order items
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

        <div class="overflow-y-auto p-6 space-y-5">
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
                >Supplier <span class="text-rose-500">*</span></label
              >
              <select
                v-model="form.supplier_id"
                @change="onSupplierChange"
                required
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              >
                <option value="">Select supplier</option>
                <option v-for="s in suppliers" :key="s.id" :value="s.id">{{ s.name }}</option>
              </select>
              <p v-if="errors.supplier_id" class="mt-1 text-xs text-rose-600">
                {{ errors.supplier_id[0] }}
              </p>
            </div>
            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
                >Expected Delivery Date</label
              >
              <input
                v-model="form.expected_at"
                type="date"
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              />
              <p v-if="errors.expected_at" class="mt-1 text-xs text-rose-600">
                {{ errors.expected_at[0] }}
              </p>
            </div>
          </div>

          <!-- Items section -->
          <div class="rounded-xl border border-zinc-200/80 bg-zinc-50/40 p-4">
            <div class="mb-3 flex items-center justify-between">
              <div>
                <label class="text-xs font-bold uppercase tracking-wider text-zinc-700"
                  >Line Items</label
                >
                <p class="text-[11px] text-zinc-500">Products and agreed purchase costs</p>
              </div>
              <button
                type="button"
                @click="addLine"
                class="inline-flex items-center gap-1 rounded-lg border border-zinc-200 bg-white px-2.5 py-1 text-xs font-semibold text-zinc-700 hover:bg-zinc-100 shadow-2xs transition-colors cursor-pointer"
              >
                <svg class="h-3.5 w-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M12 4v16m8-8H4"
                  />
                </svg>
                Add line
              </button>
            </div>

            <p v-if="errors.items" class="mb-2 text-xs text-rose-600">{{ errors.items[0] }}</p>

            <div class="space-y-2">
              <div
                v-for="(line, index) in form.items"
                :key="index"
                class="flex flex-col sm:flex-row sm:items-center gap-2 rounded-xl border border-zinc-200 bg-white p-2.5 shadow-2xs"
              >
                <div class="flex-1">
                  <select
                    v-model="line.product_id"
                    aria-label="Line product"
                    @change="onProductChange(line)"
                    class="w-full rounded-lg border border-zinc-200 bg-white px-2.5 py-1.5 text-xs text-zinc-800 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                  >
                    <option value="">Select product</option>
                    <option v-for="o in productOptions" :key="o.id" :value="o.id">
                      {{ o.name }}{{ o.sku ? ` (${o.sku})` : '' }}
                    </option>
                  </select>
                </div>
                <div class="flex items-center gap-2">
                  <input
                    v-model="line.quantity"
                    type="number"
                    min="1"
                    aria-label="Line quantity"
                    placeholder="Qty"
                    class="w-20 rounded-lg border border-zinc-200 px-2.5 py-1.5 text-xs text-zinc-800 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                  />
                  <input
                    v-model="line.unit_cost"
                    type="number"
                    step="0.01"
                    min="0"
                    aria-label="Line unit cost"
                    placeholder="Unit cost"
                    class="w-24 rounded-lg border border-zinc-200 px-2.5 py-1.5 text-xs text-zinc-800 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                  />
                  <div class="w-20 text-right text-xs font-semibold text-zinc-900">
                    {{ money((Number(line.quantity) || 0) * (Number(line.unit_cost) || 0)) }}
                  </div>
                  <button
                    type="button"
                    aria-label="Remove line"
                    class="rounded p-1 text-zinc-400 hover:bg-rose-50 hover:text-rose-600 transition-colors cursor-pointer"
                    @click="removeLine(index)"
                  >
                    <svg class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                      <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
                      />
                    </svg>
                  </button>
                </div>
              </div>
            </div>

            <div
              class="mt-3 flex items-center justify-end border-t border-zinc-200/60 pt-2 text-xs font-bold text-zinc-900"
            >
              <span class="text-zinc-500 mr-2">Estimated Total:</span>
              <span>{{ money(formTotal) }}</span>
            </div>
          </div>

          <div>
            <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Internal Notes</label>
            <textarea
              v-model="form.notes"
              rows="2"
              placeholder="Vendor notes, reference numbers, or delivery notes..."
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
            ></textarea>
          </div>
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
            {{ saving ? 'Saving...' : 'Save Purchase Order' }}
          </button>
        </div>
      </form>
    </div>

    <!-- Receive Modal -->
    <div
      v-if="showReceive"
      role="dialog"
      aria-modal="true"
      aria-label="Receive purchase order"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <form
        @submit.prevent="submitReceive"
        class="flex max-h-[85vh] w-full max-w-lg flex-col rounded-2xl border border-zinc-200/80 bg-white shadow-xl"
      >
        <div class="flex items-center justify-between border-b border-zinc-100 px-6 py-4">
          <div>
            <h3 class="text-base font-bold text-zinc-900">Receive Purchase Order</h3>
            <p class="text-xs text-zinc-500">
              Receive items for
              <span class="font-mono font-semibold">{{ receiveTarget?.number }}</span> into
              inventory
            </p>
          </div>
          <button
            type="button"
            @click="showReceive = false"
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
              >Destination Warehouse <span class="text-rose-500">*</span></label
            >
            <select
              v-model="receiveForm.warehouse_id"
              required
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
            >
              <option value="">Select destination warehouse</option>
              <option v-for="w in warehouses" :key="w.id" :value="w.id">{{ w.name }}</option>
            </select>
          </div>

          <div>
            <div class="mb-2.5 flex items-center justify-between">
              <span class="text-xs font-bold uppercase tracking-wider text-zinc-700"
                >Items to Receive</span
              >
              <button
                type="button"
                @click="receiveAll"
                class="text-xs font-semibold text-zinc-900 hover:underline cursor-pointer"
              >
                Receive all remaining
              </button>
            </div>

            <div class="space-y-2">
              <div
                v-for="line in receiveLines"
                :key="line.id"
                class="flex items-center gap-3 rounded-xl border border-zinc-200/80 bg-zinc-50/40 p-3 text-xs sm:text-sm"
              >
                <div class="flex-1">
                  <div class="font-semibold text-zinc-900">{{ line.product_name }}</div>
                  <div class="mt-0.5 text-xs text-zinc-500">
                    Ordered <span class="font-medium text-zinc-700">{{ line.ordered }}</span> &bull;
                    Received
                    <span class="font-medium text-zinc-700">{{ line.received }}</span> &bull;
                    <span class="font-medium text-zinc-900">{{ line.remaining }} remaining</span>
                  </div>
                </div>
                <input
                  v-model.number="line.quantity"
                  type="number"
                  min="0"
                  :max="line.remaining"
                  :disabled="line.remaining === 0"
                  class="w-20 rounded-lg border border-zinc-200 px-2 py-1 text-right text-xs sm:text-sm text-zinc-900 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 disabled:bg-zinc-100 disabled:text-zinc-400"
                />
              </div>
            </div>
          </div>

          <label
            class="flex items-center gap-2.5 pt-1 text-xs sm:text-sm text-zinc-700 cursor-pointer"
          >
            <input
              v-model="receiveForm.update_cost"
              type="checkbox"
              class="h-4 w-4 rounded border-zinc-300 text-zinc-900 focus:ring-zinc-900"
            />
            <span>Update product cost prices to the ordered unit cost</span>
          </label>

          <div
            v-if="receiveErrorList.length"
            class="rounded-xl border border-rose-200 bg-rose-50/80 p-3 text-xs text-rose-700 space-y-1"
          >
            <p v-for="(msg, index) in receiveErrorList" :key="index">{{ msg }}</p>
          </div>
        </div>

        <div
          class="flex items-center justify-end gap-2 border-t border-zinc-100 bg-zinc-50/50 px-6 py-4 rounded-b-2xl"
        >
          <button
            type="button"
            @click="showReceive = false"
            class="rounded-xl border border-zinc-200 bg-white px-4 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 shadow-2xs transition-colors cursor-pointer"
          >
            Cancel
          </button>
          <button
            :disabled="savingReceive || !receiveForm.warehouse_id"
            class="inline-flex items-center gap-2 rounded-xl bg-zinc-900 px-4 py-2 text-xs font-semibold text-white shadow-2xs hover:bg-zinc-800 disabled:opacity-50 transition-colors cursor-pointer"
          >
            <svg
              v-if="savingReceive"
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
            {{ savingReceive ? 'Receiving...' : 'Receive Stock' }}
          </button>
        </div>
      </form>
    </div>
  </div>
</template>
