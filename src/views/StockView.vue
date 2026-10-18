<script setup>
import { ref, reactive, watch, onMounted } from 'vue'
import { stocksApi } from '@/api/stocks'
import { stockMovementsApi } from '@/api/stockMovements'
import { productsApi } from '@/api/products'
import { warehousesApi } from '@/api/warehouses'
import { useToastStore } from '@/stores/toasts'
import { downloadResponse } from '@/utils/download'
import BarcodeScanInput from '@/components/BarcodeScanInput.vue'

const toast = useToastStore()

const items = ref([])
const meta = ref(null)
const page = ref(1)
const search = ref('')
const warehouseFilter = ref('')
const lowOnly = ref(false)
const loading = ref(false)
const pageError = ref('')

const warehouses = ref([])

const typeLabels = {
  in: 'Stock in',
  out: 'Stock out',
  adjustment: 'Adjustment',
  transfer_in: 'Transfer in',
  transfer_out: 'Transfer out',
}

// ---- adjust modal ----
const showAdjust = ref(false)
const adjustTarget = ref('')
const adjustForm = reactive({
  product_id: null,
  warehouse_id: '',
  type: 'in',
  quantity: '',
  note: '',
})
const adjustErrors = ref({})
const savingAdjust = ref(false)

// ---- transfer modal ----
const showTransfer = ref(false)
const transferTarget = ref('')
const transferForm = reactive({
  product_id: null,
  from_warehouse_id: '',
  to_warehouse_id: '',
  quantity: '',
  note: '',
})
const transferErrors = ref({})
const savingTransfer = ref(false)

// ---- history panel ----
const showHistory = ref(false)
const historyTitle = ref('')
const historyItems = ref([])
const historyLoading = ref(false)

async function load() {
  loading.value = true
  pageError.value = ''
  try {
    const { data } = await stocksApi.list({
      page: page.value,
      search: search.value || undefined,
      warehouse_id: warehouseFilter.value || undefined,
      low_stock: lowOnly.value ? 1 : undefined,
    })
    items.value = data.data
    meta.value = data.meta
  } catch {
    pageError.value = 'Could not load stock.'
  } finally {
    loading.value = false
  }
}

async function loadWarehouses() {
  const { data } = await warehousesApi.all()
  warehouses.value = data.data
}

let timer
watch(search, () => {
  clearTimeout(timer)
  timer = setTimeout(() => {
    page.value = 1
    load()
  }, 300)
})

watch([warehouseFilter, lowOnly], () => {
  page.value = 1
  load()
})

function go(p) {
  page.value = p
  load()
}

function openAdjust(row) {
  adjustTarget.value = `${row.product?.name} — ${row.product?.sku}`
  Object.assign(adjustForm, {
    product_id: row.product_id,
    warehouse_id: row.warehouse_id,
    type: 'in',
    quantity: '',
    note: '',
  })
  adjustErrors.value = {}
  showAdjust.value = true
}

async function submitAdjust() {
  savingAdjust.value = true
  adjustErrors.value = {}
  try {
    await stockMovementsApi.create({
      product_id: adjustForm.product_id,
      warehouse_id: adjustForm.warehouse_id,
      type: adjustForm.type,
      quantity: Number(adjustForm.quantity),
      note: adjustForm.note || null,
    })
    showAdjust.value = false
    toast.success('Stock adjusted.')
    await load()
  } catch (e) {
    if (e.response?.status === 422) {
      adjustErrors.value = e.response.data.errors ?? {}
    } else {
      adjustErrors.value = { quantity: ['Something went wrong. Please try again.'] }
    }
  } finally {
    savingAdjust.value = false
  }
}

function openTransfer(row) {
  transferTarget.value = row.product?.name
  Object.assign(transferForm, {
    product_id: row.product_id,
    from_warehouse_id: row.warehouse_id,
    to_warehouse_id: '',
    quantity: '',
    note: '',
  })
  transferErrors.value = {}
  showTransfer.value = true
}

async function submitTransfer() {
  savingTransfer.value = true
  transferErrors.value = {}
  try {
    await stockMovementsApi.transfer({
      product_id: transferForm.product_id,
      from_warehouse_id: transferForm.from_warehouse_id,
      to_warehouse_id: transferForm.to_warehouse_id,
      quantity: Number(transferForm.quantity),
      note: transferForm.note || null,
    })
    showTransfer.value = false
    toast.success('Stock transferred.')
    await load()
  } catch (e) {
    if (e.response?.status === 422) {
      transferErrors.value = e.response.data.errors ?? {}
    } else {
      transferErrors.value = { quantity: ['Something went wrong. Please try again.'] }
    }
  } finally {
    savingTransfer.value = false
  }
}

async function openHistory(row) {
  historyTitle.value = `${row.product?.name} — ${row.warehouse?.name}`
  historyItems.value = []
  showHistory.value = true
  historyLoading.value = true
  try {
    const { data } = await stockMovementsApi.list({
      product_id: row.product_id,
      warehouse_id: row.warehouse_id,
      per_page: 50,
    })
    historyItems.value = data.data
  } finally {
    historyLoading.value = false
  }
}

async function exportMovements() {
  try {
    const response = await stockMovementsApi.exportCsv({
      warehouse_id: warehouseFilter.value || undefined,
    })
    downloadResponse(response.data, 'stock-movements.csv')
    toast.success('Export ready.')
  } catch {
    toast.error('Export failed.')
  }
}

// a barcode scanner types the code and hits Enter — open the adjust modal for it
async function handleScan(code) {
  try {
    const { data } = await productsApi.lookup(code)
    const product = data.data

    const preferred =
      warehouses.value.find((w) => String(w.id) === String(warehouseFilter.value)) ??
      warehouses.value[0]

    adjustTarget.value = `${product.name} — ${product.sku}`
    Object.assign(adjustForm, {
      product_id: product.id,
      warehouse_id: preferred ? preferred.id : '',
      type: 'in',
      quantity: '',
      note: '',
    })
    adjustErrors.value = {}
    showAdjust.value = true
  } catch (e) {
    if (e.response?.status === 404) {
      toast.error(`No product matches "${code}".`)
    } else {
      toast.error('Barcode lookup failed.')
    }
  }
}

onMounted(() => {
  load()
  loadWarehouses()
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h2 class="text-xl sm:text-2xl font-bold tracking-tight text-zinc-900">Stock</h2>
        <p class="text-xs text-zinc-500 mt-0.5">
          Real-time inventory levels, reorder quantities, and multi-warehouse transfers
        </p>
      </div>
      <button
        @click="exportMovements"
        class="rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 shadow-2xs transition-colors cursor-pointer"
      >
        Export movements CSV
      </button>
    </div>

    <div class="flex flex-wrap items-center gap-2.5">
      <BarcodeScanInput class="w-full max-w-xs" @scan="handleScan" />
      <input
        v-model="search"
        type="text"
        aria-label="Search stock by product name or SKU"
        placeholder="Search by product name or SKU..."
        class="w-full max-w-xs rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs sm:text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs transition-colors focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
      />
      <select
        v-model="warehouseFilter"
        aria-label="Filter by warehouse"
        class="rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs sm:text-sm text-zinc-800 shadow-2xs transition-colors focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
      >
        <option value="">All warehouses</option>
        <option v-for="w in warehouses" :key="w.id" :value="w.id">{{ w.name }}</option>
      </select>
      <label class="flex items-center gap-2 text-xs sm:text-sm text-zinc-600 cursor-pointer">
        <input
          v-model="lowOnly"
          type="checkbox"
          class="h-4 w-4 rounded border-zinc-300 text-zinc-900 focus:ring-zinc-900"
        />
        Low stock only
      </label>
    </div>

    <p
      v-if="pageError"
      class="rounded-xl bg-rose-50 border border-rose-200/70 px-4 py-3 text-xs sm:text-sm text-rose-700"
    >
      {{ pageError }}
    </p>

    <div class="overflow-hidden rounded-2xl bg-white border border-zinc-200/80 shadow-2xs">
      <div class="overflow-x-auto">
        <table class="w-full text-left text-xs sm:text-sm">
          <thead
            class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
          >
            <tr>
              <th scope="col" class="px-4 py-3">Product</th>
              <th scope="col" class="px-4 py-3">SKU</th>
              <th scope="col" class="px-4 py-3">Warehouse</th>
              <th scope="col" class="px-4 py-3">Quantity</th>
              <th scope="col" class="px-4 py-3 text-right">Actions</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-zinc-100">
            <tr v-if="loading">
              <td colspan="5" class="px-4 py-8 text-center text-xs text-zinc-400">
                Loading stock...
              </td>
            </tr>
            <tr v-else-if="!items.length">
              <td colspan="5" class="px-4 py-8 text-center text-xs text-zinc-400">
                No stock records found.
              </td>
            </tr>
            <tr v-else v-for="s in items" :key="s.id" class="hover:bg-zinc-50/50 transition-colors">
              <td class="px-4 py-3 font-medium text-zinc-900">{{ s.product?.name ?? '-' }}</td>
              <td class="px-4 py-3 font-mono text-xs text-zinc-500">{{ s.product?.sku ?? '-' }}</td>
              <td class="px-4 py-3 text-zinc-600">{{ s.warehouse?.name ?? '-' }}</td>
              <td class="px-4 py-3 font-semibold text-zinc-900">{{ s.quantity }}</td>
              <td class="px-4 py-3 text-right">
                <button
                  @click="openAdjust(s)"
                  class="mr-3 font-medium text-zinc-900 hover:text-zinc-600 transition-colors cursor-pointer"
                >
                  Adjust
                </button>
                <button
                  @click="openTransfer(s)"
                  class="mr-3 font-medium text-zinc-700 hover:text-zinc-900 transition-colors cursor-pointer"
                >
                  Transfer
                </button>
                <button
                  @click="openHistory(s)"
                  class="font-medium text-zinc-500 hover:text-zinc-800 transition-colors cursor-pointer"
                >
                  History
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div
      v-if="meta && meta.last_page > 1"
      class="flex items-center justify-between text-xs text-zinc-500 pt-1"
    >
      <span>Page {{ meta.current_page }} of {{ meta.last_page }} ({{ meta.total }} total)</span>
      <div class="flex gap-2">
        <button
          :disabled="meta.current_page <= 1"
          @click="go(meta.current_page - 1)"
          class="rounded-xl border border-zinc-200 bg-white px-3 py-1.5 font-medium text-zinc-700 hover:bg-zinc-50 disabled:opacity-40 transition-colors shadow-2xs cursor-pointer"
        >
          Previous
        </button>
        <button
          :disabled="meta.current_page >= meta.last_page"
          @click="go(meta.current_page + 1)"
          class="rounded-xl border border-zinc-200 bg-white px-3 py-1.5 font-medium text-zinc-700 hover:bg-zinc-50 disabled:opacity-40 transition-colors shadow-2xs cursor-pointer"
        >
          Next
        </button>
      </div>
    </div>

    <!-- adjust modal -->
    <div
      v-if="showAdjust"
      role="dialog"
      aria-modal="true"
      aria-label="Adjust stock"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <form
        @submit.prevent="submitAdjust"
        class="w-full max-w-md space-y-4 rounded-2xl bg-white p-6 sm:p-7 shadow-xl border border-zinc-200/80"
      >
        <div class="border-b border-zinc-100 pb-3">
          <h3 class="text-base font-bold text-zinc-900">Adjust stock</h3>
          <p class="text-xs text-zinc-500 mt-0.5">{{ adjustTarget }}</p>
        </div>

        <div>
          <label class="mb-1 block text-xs font-semibold text-zinc-700">Warehouse</label>
          <select
            v-model="adjustForm.warehouse_id"
            class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-sm text-zinc-800 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
          >
            <option value="">Select warehouse</option>
            <option v-for="w in warehouses" :key="w.id" :value="w.id">{{ w.name }}</option>
          </select>
          <p v-if="adjustErrors.warehouse_id" class="mt-1 text-xs text-rose-600">
            {{ adjustErrors.warehouse_id[0] }}
          </p>
        </div>

        <div>
          <label class="mb-1 block text-xs font-semibold text-zinc-700">Type</label>
          <select
            v-model="adjustForm.type"
            class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-sm text-zinc-800 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
          >
            <option value="in">Stock in</option>
            <option value="out">Stock out</option>
            <option value="adjustment">Adjustment (+/-)</option>
          </select>
          <p v-if="adjustErrors.type" class="mt-1 text-xs text-rose-600">
            {{ adjustErrors.type[0] }}
          </p>
        </div>

        <div>
          <label class="mb-1 block text-xs font-semibold text-zinc-700">
            Quantity
            <span v-if="adjustForm.type === 'adjustment'" class="text-zinc-400 font-normal"
              >(signed, e.g. -3)</span
            >
          </label>
          <input
            v-model="adjustForm.quantity"
            type="number"
            class="w-full rounded-xl border border-zinc-200 px-3.5 py-2 text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
          />
          <p v-if="adjustErrors.quantity" class="mt-1 text-xs text-rose-600">
            {{ adjustErrors.quantity[0] }}
          </p>
        </div>

        <div>
          <label class="mb-1 block text-xs font-semibold text-zinc-700">
            Note
            <span v-if="adjustForm.type === 'adjustment'" class="text-rose-500">*</span>
          </label>
          <textarea
            v-model="adjustForm.note"
            rows="2"
            class="w-full rounded-xl border border-zinc-200 px-3.5 py-2 text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
          ></textarea>
          <p v-if="adjustErrors.note" class="mt-1 text-xs text-rose-600">
            {{ adjustErrors.note[0] }}
          </p>
        </div>

        <div class="flex justify-end gap-2 pt-2 border-t border-zinc-100">
          <button
            type="button"
            @click="showAdjust = false"
            class="rounded-xl border border-zinc-200 bg-white px-4 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 cursor-pointer shadow-2xs"
          >
            Cancel
          </button>
          <button
            :disabled="savingAdjust"
            class="rounded-xl bg-zinc-900 px-4 py-2 text-xs font-semibold text-white hover:bg-zinc-800 disabled:opacity-50 transition-colors shadow-2xs cursor-pointer"
          >
            {{ savingAdjust ? 'Saving...' : 'Save' }}
          </button>
        </div>
      </form>
    </div>

    <!-- transfer modal -->
    <div
      v-if="showTransfer"
      role="dialog"
      aria-modal="true"
      aria-label="Transfer stock"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <form
        @submit.prevent="submitTransfer"
        class="w-full max-w-md space-y-4 rounded-2xl bg-white p-6 sm:p-7 shadow-xl border border-zinc-200/80"
      >
        <div class="border-b border-zinc-100 pb-3">
          <h3 class="text-base font-bold text-zinc-900">Transfer stock</h3>
          <p class="text-xs text-zinc-500 mt-0.5">{{ transferTarget }}</p>
        </div>

        <div class="grid grid-cols-2 gap-3 sm:gap-4">
          <div>
            <label class="mb-1 block text-xs font-semibold text-zinc-700">From</label>
            <select
              v-model="transferForm.from_warehouse_id"
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-sm text-zinc-800 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
            >
              <option v-for="w in warehouses" :key="w.id" :value="w.id">{{ w.name }}</option>
            </select>
          </div>
          <div>
            <label class="mb-1 block text-xs font-semibold text-zinc-700">To</label>
            <select
              v-model="transferForm.to_warehouse_id"
              class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-sm text-zinc-800 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
            >
              <option value="">Select warehouse</option>
              <option v-for="w in warehouses" :key="w.id" :value="w.id">{{ w.name }}</option>
            </select>
            <p v-if="transferErrors.to_warehouse_id" class="mt-1 text-xs text-rose-600">
              {{ transferErrors.to_warehouse_id[0] }}
            </p>
          </div>
        </div>

        <div>
          <label class="mb-1 block text-xs font-semibold text-zinc-700">Quantity</label>
          <input
            v-model="transferForm.quantity"
            type="number"
            min="1"
            class="w-full rounded-xl border border-zinc-200 px-3.5 py-2 text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
          />
          <p v-if="transferErrors.quantity" class="mt-1 text-xs text-rose-600">
            {{ transferErrors.quantity[0] }}
          </p>
        </div>

        <div>
          <label class="mb-1 block text-xs font-semibold text-zinc-700">Note</label>
          <textarea
            v-model="transferForm.note"
            rows="2"
            class="w-full rounded-xl border border-zinc-200 px-3.5 py-2 text-sm text-zinc-900 shadow-2xs focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900 transition-colors"
          ></textarea>
        </div>

        <div class="flex justify-end gap-2 pt-2 border-t border-zinc-100">
          <button
            type="button"
            @click="showTransfer = false"
            class="rounded-xl border border-zinc-200 bg-white px-4 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 cursor-pointer shadow-2xs"
          >
            Cancel
          </button>
          <button
            :disabled="savingTransfer"
            class="rounded-xl bg-zinc-900 px-4 py-2 text-xs font-semibold text-white hover:bg-zinc-800 disabled:opacity-50 transition-colors shadow-2xs cursor-pointer"
          >
            {{ savingTransfer ? 'Transferring...' : 'Transfer' }}
          </button>
        </div>
      </form>
    </div>

    <!-- history panel -->
    <div
      v-if="showHistory"
      role="dialog"
      aria-modal="true"
      aria-label="Stock movement history"
      class="fixed inset-0 z-50 flex items-center justify-center bg-zinc-950/40 backdrop-blur-xs p-4"
    >
      <div
        class="flex max-h-[80vh] w-full max-w-2xl flex-col rounded-2xl bg-white p-6 sm:p-7 shadow-xl border border-zinc-200/80"
      >
        <div class="mb-4 flex items-center justify-between border-b border-zinc-100 pb-3.5">
          <div>
            <h3 class="text-base font-bold text-zinc-900">Movement history</h3>
            <p class="text-xs text-zinc-500 mt-0.5">{{ historyTitle }}</p>
          </div>
          <button
            @click="showHistory = false"
            class="rounded-xl border border-zinc-200 bg-white px-3 py-1.5 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 cursor-pointer shadow-2xs"
          >
            Close
          </button>
        </div>

        <div class="overflow-y-auto rounded-xl border border-zinc-200/80">
          <table class="w-full text-left text-xs sm:text-sm">
            <thead
              class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
            >
              <tr>
                <th scope="col" class="px-3.5 py-2.5">Date</th>
                <th scope="col" class="px-3.5 py-2.5">Type</th>
                <th scope="col" class="px-3.5 py-2.5">Change</th>
                <th scope="col" class="px-3.5 py-2.5">Balance</th>
                <th scope="col" class="px-3.5 py-2.5">By</th>
                <th scope="col" class="px-3.5 py-2.5">Note</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-zinc-100">
              <tr v-if="historyLoading">
                <td colspan="6" class="px-3.5 py-6 text-center text-xs text-zinc-400">
                  Loading history...
                </td>
              </tr>
              <tr v-else-if="!historyItems.length">
                <td colspan="6" class="px-3.5 py-6 text-center text-xs text-zinc-400">
                  No movements yet.
                </td>
              </tr>
              <tr
                v-else
                v-for="m in historyItems"
                :key="m.id"
                class="hover:bg-zinc-50/50 transition-colors"
              >
                <td class="px-3.5 py-2.5 text-zinc-500 text-xs font-mono">
                  {{ new Date(m.created_at).toLocaleString() }}
                </td>
                <td class="px-3.5 py-2.5 font-medium text-zinc-800">
                  {{ typeLabels[m.type] ?? m.type }}
                </td>
                <td
                  class="px-3.5 py-2.5 font-semibold text-xs"
                  :class="m.quantity < 0 ? 'text-rose-600' : 'text-emerald-600'"
                >
                  {{ m.quantity > 0 ? '+' : '' }}{{ m.quantity }}
                </td>
                <td class="px-3.5 py-2.5 font-mono text-zinc-700 font-medium">
                  {{ m.balance_after }}
                </td>
                <td class="px-3.5 py-2.5 text-zinc-600">{{ m.user_name ?? '-' }}</td>
                <td class="px-3.5 py-2.5 text-zinc-500 text-xs">{{ m.note ?? '-' }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>
