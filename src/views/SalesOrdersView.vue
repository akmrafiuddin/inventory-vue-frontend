<script setup>
import { ref, reactive, computed, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { salesOrdersApi } from '@/api/salesOrders'
import { customersApi } from '@/api/customers'
import { warehousesApi } from '@/api/warehouses'
import { productsApi } from '@/api/products'
import { useToastStore } from '@/stores/toasts'
import { useAuthStore } from '@/stores/auth'

const router = useRouter()
const toast = useToastStore()
const auth = useAuthStore()

const items = ref([])
const meta = ref(null)
const page = ref(1)
const search = ref('')
const customerFilter = ref('')
const statusFilter = ref('')
const loading = ref(false)
const pageError = ref('')

const customers = ref([])
const warehouses = ref([])
const allProducts = ref([])

const statusStyles = {
  draft: 'bg-zinc-100 text-zinc-700 ring-1 ring-zinc-300/60',
  confirmed: 'bg-blue-50 text-blue-700 ring-1 ring-blue-600/20',
  partially_shipped: 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20',
  shipped: 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20',
  cancelled: 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20',
}

const statusLabels = {
  draft: 'Draft',
  confirmed: 'Confirmed',
  partially_shipped: 'Partial',
  shipped: 'Shipped',
  cancelled: 'Cancelled',
}

function money(value) {
  return `$${Number(value ?? 0).toFixed(2)}`
}

// ---- create / edit modal ----
const showForm = ref(false)
const editingId = ref(null)
const form = reactive({
  customer_id: '',
  order_date: '',
  expected_ship_date: '',
  shipping_address: '',
  notes: '',
  discount: '',
  tax: '',
  items: [],
})
const errors = ref({})
const saving = ref(false)

// live form subtotal & total
const formSubtotal = computed(() => {
  return form.items.reduce((sum, line) => {
    const qty = Number(line.quantity) || 0
    const price = Number(line.unit_price) || 0
    return sum + qty * price
  }, 0)
})

const formTotal = computed(() => {
  const discount = Number(form.discount) || 0
  const tax = Number(form.tax) || 0
  return Math.max(0, formSubtotal.value - discount + tax)
})

// ---- ship / dispatch modal ----
const showShip = ref(false)
const shipTarget = ref(null)
const shipForm = reactive({ warehouse_id: '' })
const shipLines = ref([])
const shipErrors = ref({})
const savingShip = ref(false)

const shipErrorList = computed(() => Object.values(shipErrors.value).flat())

async function load() {
  loading.value = true
  pageError.value = ''
  try {
    const { data } = await salesOrdersApi.list({
      page: page.value,
      search: search.value || undefined,
      customer_id: customerFilter.value || undefined,
      status: statusFilter.value || undefined,
    })
    items.value = data.data
    meta.value = data.meta
  } catch {
    pageError.value = 'Could not load sales orders.'
  } finally {
    loading.value = false
  }
}

async function loadLookups() {
  const [c, w, p] = await Promise.all([customersApi.all(), warehousesApi.all(), productsApi.all()])
  customers.value = c.data.data
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

watch([customerFilter, statusFilter], () => {
  page.value = 1
  load()
})

function go(p) {
  page.value = p
  load()
}

function emptyLine() {
  return { product_id: '', quantity: 1, unit_price: '' }
}

function resetForm() {
  const today = new Date().toISOString().substring(0, 10)
  Object.assign(form, {
    customer_id: '',
    order_date: today,
    expected_ship_date: '',
    shipping_address: '',
    notes: '',
    discount: '',
    tax: '',
    items: [emptyLine()],
  })
  errors.value = {}
}

function onProductChange(line) {
  const opt = allProducts.value.find((o) => o.id === Number(line.product_id))
  if (opt && opt.sell_price != null) {
    line.unit_price = opt.sell_price
  }
}

function addLine() {
  form.items.push(emptyLine())
}

function removeLine(index) {
  form.items.splice(index, 1)
  if (!form.items.length) form.items.push(emptyLine())
}

function openCreate() {
  editingId.value = null
  resetForm()
  showForm.value = true
}

async function openEdit(so) {
  editingId.value = so.id
  resetForm()
  const { data } = await salesOrdersApi.show(so.id)
  const d = data.data
  form.customer_id = d.customer_id
  form.order_date = d.order_date ? d.order_date.substring(0, 10) : ''
  form.expected_ship_date = d.expected_ship_date ? d.expected_ship_date.substring(0, 10) : ''
  form.shipping_address = d.shipping_address ?? ''
  form.notes = d.notes ?? ''
  form.discount = d.discount ? Number(d.discount) : ''
  form.tax = d.tax ? Number(d.tax) : ''
  form.items = (d.items ?? []).map((i) => ({
    product_id: i.product_id,
    quantity: i.quantity,
    unit_price: i.unit_price,
  }))
  showForm.value = true
}

async function save() {
  saving.value = true
  errors.value = {}

  const payload = {
    customer_id: Number(form.customer_id),
    order_date: form.order_date || null,
    expected_ship_date: form.expected_ship_date || null,
    shipping_address: form.shipping_address || null,
    notes: form.notes || null,
    discount: form.discount === '' ? 0 : Number(form.discount),
    tax: form.tax === '' ? 0 : Number(form.tax),
    items: form.items
      .filter((l) => l.product_id)
      .map((l) => ({
        product_id: Number(l.product_id),
        quantity: Number(l.quantity),
        unit_price: l.unit_price === '' ? 0 : Number(l.unit_price),
      })),
  }

  try {
    if (editingId.value) {
      await salesOrdersApi.update(editingId.value, payload)
    } else {
      await salesOrdersApi.create(payload)
    }
    showForm.value = false
    toast.success(editingId.value ? 'Sales order updated.' : 'Sales order created.')
    await load()
  } catch (e) {
    if (e.response?.status === 422) {
      errors.value = e.response.data.errors ?? {}
    } else {
      errors.value = { customer_id: ['Something went wrong. Please try again.'] }
    }
  } finally {
    saving.value = false
  }
}

async function confirmOrder(so) {
  try {
    await salesOrdersApi.confirm(so.id)
    toast.success('Sales order confirmed.')
    await load()
  } catch (e) {
    toast.error(e.response?.data?.message ?? 'Confirm failed.')
  }
}

async function openShip(so) {
  shipTarget.value = so
  shipForm.warehouse_id = ''
  shipLines.value = []
  shipErrors.value = {}
  showShip.value = true

  try {
    const { data } = await salesOrdersApi.show(so.id)
    shipLines.value = (data.data.items ?? []).map((i) => ({
      id: i.id,
      product_name: i.product_name,
      product_sku: i.product_sku,
      ordered: i.quantity,
      shipped: i.quantity_shipped,
      remaining: i.quantity_remaining,
      quantity: i.quantity_remaining,
    }))
  } catch {
    shipErrors.value = { warehouse_id: ['Could not load the order lines.'] }
  }
}

function shipAll() {
  shipLines.value.forEach((line) => {
    line.quantity = line.remaining
  })
}

async function submitShip() {
  savingShip.value = true
  shipErrors.value = {}
  try {
    await salesOrdersApi.ship(shipTarget.value.id, {
      warehouse_id: Number(shipForm.warehouse_id),
      items: shipLines.value.map((l) => ({ id: l.id, quantity: Number(l.quantity) || 0 })),
    })
    showShip.value = false
    toast.success('Stock dispatched successfully.')
    await load()
  } catch (e) {
    shipErrors.value = { warehouse_id: ['Something went wrong. Please try again.'] }

    if (e.response?.status === 422) {
      shipErrors.value = e.response.data.errors ?? {}
      if (!Object.keys(shipErrors.value).length && e.response.data.message) {
        shipErrors.value = { items: [e.response.data.message] }
      }
    }
  } finally {
    savingShip.value = false
  }
}

async function cancelOrder(so) {
  if (!confirm(`Cancel ${so.number}?`)) return
  try {
    await salesOrdersApi.cancel(so.id)
    toast.success('Sales order cancelled.')
    await load()
  } catch (e) {
    toast.error(e.response?.data?.message ?? 'Cancel failed.')
  }
}

async function removeOrder(so) {
  if (!confirm(`Delete draft ${so.number}?`)) return
  try {
    await salesOrdersApi.remove(so.id)
    if (items.value.length === 1 && page.value > 1) page.value--
    await load()
    toast.success('Sales order deleted.')
  } catch (e) {
    toast.error(e.response?.data?.message ?? 'Delete failed.')
  }
}

function printOrder(so) {
  router.push({ name: 'sales-order-print', params: { id: so.id } })
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
        <h1 class="text-xl font-bold tracking-tight text-zinc-900 sm:text-2xl">Sales Orders</h1>
        <p class="mt-0.5 text-xs text-zinc-500 sm:text-sm">
          Manage client orders, fulfillment status, and shipments
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
          New sales order
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
            aria-label="Search by order number"
            placeholder="Search SO number..."
            class="w-full rounded-xl border border-zinc-200 bg-zinc-50/50 pl-9 pr-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 focus:border-zinc-900 focus:bg-white focus:outline-none focus:ring-1 focus:ring-zinc-900 shadow-2xs transition-all"
          />
        </div>
        <div class="flex flex-wrap items-center gap-2 sm:gap-3">
          <select
            v-model="customerFilter"
            aria-label="Filter by customer"
            class="rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs sm:text-sm text-zinc-700 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          >
            <option value="">All customers</option>
            <option v-for="c in customers" :key="c.id" :value="c.id">
              {{ c.name }}{{ c.company_name ? ` (${c.company_name})` : '' }}
            </option>
          </select>
          <select
            v-model="statusFilter"
            aria-label="Filter by status"
            class="rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs sm:text-sm text-zinc-700 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          >
            <option value="">All statuses</option>
            <option value="draft">Draft</option>
            <option value="confirmed">Confirmed</option>
            <option value="partially_shipped">Partially shipped</option>
            <option value="shipped">Shipped</option>
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
              <th scope="col" class="px-4 py-3 sm:px-6">Order #</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Customer</th>
              <th scope="col" class="px-4 py-3 sm:px-6">Order Date</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-center">Lines</th>
              <th scope="col" class="px-4 py-3 sm:px-6 text-right">Total</th>
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
                  Loading sales orders...
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
                        d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"
                      />
                    </svg>
                  </div>
                  <span class="font-medium text-zinc-800">No sales orders found</span>
                  <span class="text-zinc-400 mt-0.5"
                    >Try adjusting your search criteria or create a new order.</span
                  >
                </div>
              </td>
            </tr>
            <tr
              v-else
              v-for="so in items"
              :key="so.id"
              class="hover:bg-zinc-50/60 transition-colors"
            >
              <td class="px-4 py-3.5 sm:px-6">
                <span class="font-mono text-xs font-semibold text-zinc-900">{{ so.number }}</span>
              </td>
              <td class="px-4 py-3.5 sm:px-6">
                <div class="font-medium text-zinc-900">{{ so.customer?.name ?? '—' }}</div>
                <div v-if="so.customer?.company_name" class="text-[11px] text-zinc-500">
                  {{ so.customer.company_name }}
                </div>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-zinc-600">
                {{ so.order_date ? so.order_date.substring(0, 10) : '—' }}
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-center text-zinc-600">
                <span
                  class="inline-flex items-center rounded-md bg-zinc-100 px-2 py-0.5 text-xs font-medium text-zinc-700"
                >
                  {{ so.items_count }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-right font-medium text-zinc-900">
                {{ money(so.total) }}
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-center">
                <span
                  class="inline-flex items-center rounded-full px-2.5 py-0.5 text-[11px] font-medium"
                  :class="statusStyles[so.status] || 'bg-zinc-100 text-zinc-700'"
                >
                  {{ statusLabels[so.status] ?? so.status }}
                </span>
              </td>
              <td class="px-4 py-3.5 sm:px-6 text-right whitespace-nowrap">
                <div class="flex items-center justify-end gap-1.5">
                  <button
                    @click="printOrder(so)"
                    title="Print order"
                    class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-600 hover:bg-zinc-100 hover:text-zinc-900 transition-colors cursor-pointer"
                  >
                    Print
                  </button>

                  <template v-if="so.status === 'draft' && auth.canManage">
                    <button
                      @click="confirmOrder(so)"
                      class="rounded-lg border border-blue-200 bg-blue-50 px-2.5 py-1 text-xs font-semibold text-blue-700 hover:bg-blue-100 transition-colors cursor-pointer"
                    >
                      Confirm
                    </button>
                  </template>

                  <template v-if="['draft', 'confirmed', 'partially_shipped'].includes(so.status)">
                    <button
                      @click="openShip(so)"
                      class="rounded-lg border border-emerald-200 bg-emerald-50 px-2.5 py-1 text-xs font-semibold text-emerald-700 hover:bg-emerald-100 transition-colors cursor-pointer"
                    >
                      Ship
                    </button>
                  </template>

                  <template v-if="so.status === 'draft' && auth.canManage">
                    <button
                      @click="openEdit(so)"
                      class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-700 hover:bg-zinc-100 transition-colors cursor-pointer"
                    >
                      Edit
                    </button>
                    <button
                      @click="removeOrder(so)"
                      class="rounded-lg border border-rose-200 bg-rose-50/50 px-2.5 py-1 text-xs font-medium text-rose-700 hover:bg-rose-100 transition-colors cursor-pointer"
                    >
                      Delete
                    </button>
                  </template>

                  <template v-if="['draft', 'confirmed'].includes(so.status) && auth.canManage">
                    <button
                      @click="cancelOrder(so)"
                      class="rounded-lg border border-zinc-200 px-2.5 py-1 text-xs font-medium text-zinc-500 hover:bg-rose-50 hover:text-rose-700 hover:border-rose-200 transition-colors cursor-pointer"
                    >
                      Cancel
                    </button>
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
      aria-label="Sales order form"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <form
        @submit.prevent="save"
        class="flex max-h-[90vh] w-full max-w-3xl flex-col rounded-2xl border border-zinc-200/80 bg-white shadow-xl"
      >
        <div class="flex items-center justify-between border-b border-zinc-100 px-6 py-4">
          <div>
            <h3 class="text-base font-bold text-zinc-900">
              {{ editingId ? 'Edit Sales Order' : 'New Sales Order' }}
            </h3>
            <p class="text-xs text-zinc-500">
              Fill in client information, fulfillment schedule, and products
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
          <div class="grid grid-cols-1 sm:grid-cols-3 gap-4">
            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
                >Customer <span class="text-rose-500">*</span></label
              >
              <select
                v-model="form.customer_id"
                required
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              >
                <option value="">Select customer</option>
                <option v-for="c in customers" :key="c.id" :value="c.id">
                  {{ c.name }}{{ c.company_name ? ` (${c.company_name})` : '' }}
                </option>
              </select>
              <p v-if="errors.customer_id" class="mt-1 text-xs text-rose-600">
                {{ errors.customer_id[0] }}
              </p>
            </div>

            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Order Date</label>
              <input
                v-model="form.order_date"
                type="date"
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              />
              <p v-if="errors.order_date" class="mt-1 text-xs text-rose-600">
                {{ errors.order_date[0] }}
              </p>
            </div>

            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
                >Expected Ship Date</label
              >
              <input
                v-model="form.expected_ship_date"
                type="date"
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              />
            </div>
          </div>

          <div>
            <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Shipping Address</label>
            <input
              v-model="form.shipping_address"
              type="text"
              placeholder="Delivery destination address..."
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
            />
          </div>

          <!-- Line Items Section -->
          <div class="rounded-xl border border-zinc-200/80 bg-zinc-50/40 p-4">
            <div class="mb-3 flex items-center justify-between">
              <div>
                <label class="text-xs font-bold uppercase tracking-wider text-zinc-700"
                  >Line Items</label
                >
                <p class="text-[11px] text-zinc-500">Products requested in this sales order</p>
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
                Add item
              </button>
            </div>

            <p v-if="errors.items" class="mb-2 text-xs text-rose-600">{{ errors.items[0] }}</p>

            <div class="overflow-x-auto rounded-xl border border-zinc-200 bg-white">
              <table class="w-full text-left text-xs">
                <thead
                  class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
                >
                  <tr>
                    <th scope="col" class="px-3 py-2">Product</th>
                    <th scope="col" class="w-24 px-3 py-2">Qty</th>
                    <th scope="col" class="w-28 px-3 py-2">Unit price ($)</th>
                    <th scope="col" class="w-28 px-3 py-2 text-right">Line total</th>
                    <th scope="col" class="w-10 px-2 py-2"></th>
                  </tr>
                </thead>
                <tbody class="divide-y divide-zinc-100">
                  <tr v-for="(line, idx) in form.items" :key="idx" class="hover:bg-zinc-50/50">
                    <td class="px-3 py-2">
                      <select
                        v-model="line.product_id"
                        @change="onProductChange(line)"
                        required
                        class="w-full rounded-lg border border-zinc-200 bg-white px-2 py-1.5 text-xs text-zinc-800 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                      >
                        <option value="">Select product</option>
                        <option v-for="p in allProducts" :key="p.id" :value="p.id">
                          {{ p.name }} ({{ p.sku }}) — {{ money(p.sell_price) }}
                        </option>
                      </select>
                    </td>
                    <td class="px-3 py-2">
                      <input
                        v-model="line.quantity"
                        type="number"
                        min="1"
                        required
                        class="w-full rounded-lg border border-zinc-200 px-2 py-1.5 text-xs text-zinc-800 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                      />
                    </td>
                    <td class="px-3 py-2">
                      <input
                        v-model="line.unit_price"
                        type="number"
                        step="0.01"
                        min="0"
                        class="w-full rounded-lg border border-zinc-200 px-2 py-1.5 text-xs text-zinc-800 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                      />
                    </td>
                    <td class="px-3 py-2 text-right font-medium text-zinc-900">
                      {{ money((Number(line.quantity) || 0) * (Number(line.unit_price) || 0)) }}
                    </td>
                    <td class="px-2 py-2 text-center">
                      <button
                        type="button"
                        @click="removeLine(idx)"
                        title="Remove row"
                        class="rounded p-1 text-zinc-400 hover:bg-rose-50 hover:text-rose-600 transition-colors cursor-pointer"
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
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>

          <!-- Bottom Summary & Notes -->
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 pt-1">
            <div>
              <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Internal Notes</label>
              <textarea
                v-model="form.notes"
                rows="3"
                placeholder="Optional customer instructions or internal memo..."
                class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
              ></textarea>
            </div>

            <div
              class="rounded-xl border border-zinc-200/80 bg-zinc-50/50 p-4 space-y-2.5 text-xs sm:text-sm"
            >
              <div class="flex items-center justify-between text-zinc-600">
                <span>Subtotal:</span>
                <span class="font-medium text-zinc-900">{{ money(formSubtotal) }}</span>
              </div>
              <div class="flex items-center justify-between gap-3 text-zinc-600">
                <span>Discount ($):</span>
                <input
                  v-model="form.discount"
                  type="number"
                  step="0.01"
                  min="0"
                  placeholder="0.00"
                  class="w-24 rounded-lg border border-zinc-200 bg-white px-2 py-1 text-right text-xs text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                />
              </div>
              <div class="flex items-center justify-between gap-3 text-zinc-600">
                <span>Tax ($):</span>
                <input
                  v-model="form.tax"
                  type="number"
                  step="0.01"
                  min="0"
                  placeholder="0.00"
                  class="w-24 rounded-lg border border-zinc-200 bg-white px-2 py-1 text-right text-xs text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                />
              </div>
              <div
                class="flex items-center justify-between border-t border-zinc-200 pt-2 text-sm sm:text-base font-bold text-zinc-900"
              >
                <span>Grand Total:</span>
                <span class="text-zinc-900">{{ money(formTotal) }}</span>
              </div>
            </div>
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
            {{ saving ? 'Saving...' : 'Save Order' }}
          </button>
        </div>
      </form>
    </div>

    <!-- Ship / Dispatch Modal -->
    <div
      v-if="showShip"
      role="dialog"
      aria-modal="true"
      aria-label="Ship sales order"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <div
        class="flex max-h-[85vh] w-full max-w-2xl flex-col rounded-2xl border border-zinc-200/80 bg-white shadow-xl"
      >
        <div class="flex items-center justify-between border-b border-zinc-100 px-6 py-4">
          <div>
            <h3 class="text-base font-bold text-zinc-900">Dispatch & Ship Order</h3>
            <p class="text-xs text-zinc-500">
              {{ shipTarget?.number }} &bull; {{ shipTarget?.customer?.name }}
            </p>
          </div>
          <button
            type="button"
            @click="showShip = false"
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
          <div
            v-if="shipErrorList.length"
            class="rounded-xl border border-rose-200 bg-rose-50/80 p-3 text-xs text-rose-700"
          >
            <p v-for="(err, i) in shipErrorList" :key="i">{{ err }}</p>
          </div>

          <div>
            <label class="mb-1.5 block text-xs font-semibold text-zinc-700"
              >Source Warehouse <span class="text-rose-500">*</span></label
            >
            <select
              v-model="shipForm.warehouse_id"
              required
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
            >
              <option value="">Select fulfillment warehouse</option>
              <option v-for="w in warehouses" :key="w.id" :value="w.id">{{ w.name }}</option>
            </select>
          </div>

          <div>
            <div class="mb-2.5 flex items-center justify-between">
              <span class="text-xs font-bold uppercase tracking-wider text-zinc-700"
                >Quantities to Dispatch</span
              >
              <button
                type="button"
                @click="shipAll"
                class="text-xs font-semibold text-zinc-900 hover:underline cursor-pointer"
              >
                Ship all remaining
              </button>
            </div>

            <div class="overflow-x-auto rounded-xl border border-zinc-200">
              <table class="w-full text-left text-xs">
                <thead
                  class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
                >
                  <tr>
                    <th scope="col" class="px-3.5 py-2.5">Product</th>
                    <th scope="col" class="px-3 py-2.5 text-right">Ordered</th>
                    <th scope="col" class="px-3 py-2.5 text-right">Shipped</th>
                    <th scope="col" class="px-3 py-2.5 text-right">Remaining</th>
                    <th scope="col" class="w-28 px-3.5 py-2.5 text-right">Ship Now</th>
                  </tr>
                </thead>
                <tbody class="divide-y divide-zinc-100">
                  <tr v-for="line in shipLines" :key="line.id" class="hover:bg-zinc-50/50">
                    <td class="px-3.5 py-2.5">
                      <span class="font-medium text-zinc-900">{{ line.product_name }}</span>
                      <span class="block font-mono text-[11px] text-zinc-400">{{
                        line.product_sku
                      }}</span>
                    </td>
                    <td class="px-3 py-2.5 text-right text-zinc-600">{{ line.ordered }}</td>
                    <td class="px-3 py-2.5 text-right text-zinc-500">{{ line.shipped }}</td>
                    <td class="px-3 py-2.5 text-right font-medium text-zinc-900">
                      {{ line.remaining }}
                    </td>
                    <td class="px-3.5 py-2.5 text-right">
                      <input
                        v-model="line.quantity"
                        type="number"
                        min="0"
                        :max="line.remaining"
                        class="w-20 rounded-lg border border-zinc-200 px-2 py-1 text-right text-xs text-zinc-900 focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
                      />
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>

        <div
          class="flex items-center justify-end gap-2 border-t border-zinc-100 bg-zinc-50/50 px-6 py-4 rounded-b-2xl"
        >
          <button
            type="button"
            @click="showShip = false"
            class="rounded-xl border border-zinc-200 bg-white px-4 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 shadow-2xs transition-colors cursor-pointer"
          >
            Cancel
          </button>
          <button
            :disabled="savingShip || !shipForm.warehouse_id"
            @click="submitShip"
            class="inline-flex items-center gap-2 rounded-xl bg-zinc-900 px-4 py-2 text-xs font-semibold text-white shadow-2xs hover:bg-zinc-800 disabled:opacity-50 transition-colors cursor-pointer"
          >
            <svg
              v-if="savingShip"
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
            {{ savingShip ? 'Dispatching...' : 'Dispatch Stock' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
