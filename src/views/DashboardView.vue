<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'
import { stocksApi } from '@/api/stocks'
import { productsApi } from '@/api/products'
import { useAuthStore } from '@/stores/auth'
import BarcodeScanInput from '@/components/BarcodeScanInput.vue'
import ProductDetailModal from '@/components/ProductDetailModal.vue'

const auth = useAuthStore()

// State
const loading = ref(true)
const refreshing = ref(false)
const error = ref('')
const data = ref(null)

// Active tab in recent operations stream ('sales' | 'purchases' | 'movements')
const activeTab = ref('sales')

// Barcode scanner modal state
const showScanModal = ref(false)
const scanInputRef = ref(null)
const scanLoading = ref(false)
const scanError = ref('')
const scannedProduct = ref(null)

// Detailed product modal for quick inspection
const inspectionProduct = ref(null)

// Current date formatted nicely
const currentDateFormatted = computed(() => {
  return new Date().toLocaleDateString(undefined, {
    weekday: 'long',
    month: 'short',
    day: 'numeric',
    year: 'numeric',
  })
})

// Permissions
const canCreateSalesOrder = computed(() => ['admin', 'manager'].includes(auth.user?.role))
const canCreatePurchaseOrder = computed(() => ['admin', 'manager'].includes(auth.user?.role))
const canViewActivity = computed(() => ['admin', 'manager'].includes(auth.user?.role))

// Format currency
function money(val) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(val ?? 0)
}

// Relative time helper (e.g. "5m ago", "2h ago")
function timeAgo(dateString) {
  if (!dateString) return '-'
  const date = new Date(dateString)
  const now = new Date()
  const seconds = Math.floor((now - date) / 1000)

  if (seconds < 60) return 'Just now'
  const minutes = Math.floor(seconds / 60)
  if (minutes < 60) return `${minutes}m ago`
  const hours = Math.floor(minutes / 60)
  if (hours < 24) return `${hours}h ago`
  const days = Math.floor(hours / 24)
  if (days < 30) return `${days}d ago`
  return date.toLocaleDateString()
}

// Format short date (e.g. "Oct 18, 2026")
function formatDate(dateString) {
  if (!dateString) return '-'
  const date = new Date(dateString)
  return date.toLocaleDateString(undefined, {
    month: 'short',
    day: 'numeric',
    year: 'numeric',
  })
}

// Status styling for sales orders
const salesStatusStyles = {
  draft: 'bg-zinc-100 text-zinc-700 ring-1 ring-zinc-300/60',
  confirmed: 'bg-blue-50 text-blue-700 ring-1 ring-blue-600/20',
  partially_shipped: 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20',
  shipped: 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20',
  cancelled: 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20',
}

// Status styling for purchase orders
const purchaseStatusStyles = {
  draft: 'bg-zinc-100 text-zinc-700 ring-1 ring-zinc-300/60',
  partially_received: 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20',
  received: 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20',
  cancelled: 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20',
}

// Movement type badges
const movementStyles = {
  in: 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20',
  out: 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20',
  adjustment: 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20',
  transfer_in: 'bg-zinc-100 text-zinc-800 ring-1 ring-zinc-300/60',
  transfer_out: 'bg-zinc-100 text-zinc-800 ring-1 ring-zinc-300/60',
}

// Audit log action badges
const activityStyles = {
  created: 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20',
  updated: 'bg-zinc-100 text-zinc-800 ring-1 ring-zinc-300/60',
  deleted: 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20',
  restored: 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20',
  shipped: 'bg-blue-50 text-blue-700 ring-1 ring-blue-600/20',
  received: 'bg-teal-50 text-teal-700 ring-1 ring-teal-600/20',
}

// Fetch dashboard data
async function loadDashboard(isRefresh = false) {
  if (isRefresh) {
    refreshing.value = true
  } else {
    loading.value = true
  }
  error.value = ''

  try {
    const { data: res } = await stocksApi.dashboard()
    data.value = res
  } catch {
    error.value = 'Failed to load dashboard statistics. Please check your connection and retry.'
  } finally {
    loading.value = false
    refreshing.value = false
  }
}

// Open Quick Scanner modal
function openScanModal() {
  scannedProduct.value = null
  scanError.value = ''
  showScanModal.value = true
  nextTick(() => {
    scanInputRef.value?.focus()
  })
}

// Handle barcode or SKU scan
async function handleBarcodeScan(code) {
  scanLoading.value = true
  scanError.value = ''
  scannedProduct.value = null

  try {
    const { data: res } = await productsApi.lookup(code)
    scannedProduct.value = res.data
  } catch (err) {
    if (err.response?.status === 404) {
      scanError.value = `No item found for SKU / Barcode "${code}".`
    } else {
      scanError.value = 'Lookup failed. Please verify the code and retry.'
    }
  } finally {
    scanLoading.value = false
  }
}

// Inspect a product
function openProductDetail(product) {
  inspectionProduct.value = product
}

function inspectScannedProduct() {
  if (scannedProduct.value) {
    inspectionProduct.value = scannedProduct.value
    showScanModal.value = false
  }
}

// Keyboard shortcut (press '/' to open scan modal)
function handleKeydown(e) {
  if (e.key === '/' && !['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName)) {
    e.preventDefault()
    openScanModal()
  } else if (e.key === 'Escape' && showScanModal.value) {
    showScanModal.value = false
  }
}

onMounted(() => {
  loadDashboard()
  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
})
</script>

<template>
  <div class="space-y-6">
    <!-- ============================================================ -->
    <!-- HEADER BAR: Welcome greeting, status, date & quick buttons    -->
    <!-- ============================================================ -->
    <div
      class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between rounded-2xl bg-white p-5 sm:p-6 shadow-2xs border border-zinc-200/80"
    >
      <div>
        <div class="flex items-center gap-2.5">
          <h1 class="text-xl sm:text-2xl font-bold tracking-tight text-zinc-900">
            Operations Dashboard
          </h1>
          <span
            class="inline-flex items-center gap-1.5 rounded-full px-2.5 py-0.5 text-xs font-medium capitalize"
            :class="
              auth.user?.role === 'admin'
                ? 'bg-zinc-900 text-white'
                : auth.user?.role === 'manager'
                  ? 'bg-zinc-100 text-zinc-800 ring-1 ring-zinc-300/70'
                  : 'bg-zinc-100 text-zinc-700 ring-1 ring-zinc-200'
            "
          >
            <span
              class="h-1.5 w-1.5 rounded-full"
              :class="
                auth.user?.role === 'admin'
                  ? 'bg-white'
                  : auth.user?.role === 'manager'
                    ? 'bg-zinc-700'
                    : 'bg-zinc-500'
              "
            ></span>
            {{ auth.user?.role || 'Staff' }}
          </span>
        </div>
        <p class="mt-1 text-xs sm:text-sm text-zinc-500 flex items-center gap-2">
          <span>Welcome back, {{ auth.user?.name || 'User' }}</span>
          <span class="text-zinc-300">•</span>
          <span class="text-zinc-600 font-medium">{{ currentDateFormatted }}</span>
        </p>
      </div>

      <!-- Quick Action Buttons -->
      <div class="flex flex-wrap items-center gap-2">
        <button
          type="button"
          @click="openScanModal"
          class="inline-flex items-center gap-1.5 rounded-xl border border-zinc-200 bg-zinc-50/80 px-3 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-100 transition-colors shadow-2xs cursor-pointer"
          title="Scan barcode or look up SKU (Hotkey: /)"
        >
          <svg
            class="h-3.5 w-3.5 text-zinc-500"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="1.8"
              d="M12 4v1m6 11h2m-6 0h-2v4m0-11v3m0 0h.01M12 12h4.01M16 20h4M4 12h4m12 0h.01M5 8h2a1 1 0 001-1V5a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1zm12 0h2a1 1 0 001-1V5a1 1 0 00-1-1h-2a1 1 0 00-1 1v2a1 1 0 001 1zM5 20h2a1 1 0 001-1v-2a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1z"
            />
          </svg>
          <span>Scan / Search SKU</span>
          <kbd
            class="rounded border border-zinc-200 bg-white px-1.5 py-0.2 text-[10px] font-mono text-zinc-400"
          >
            /
          </kbd>
        </button>

        <button
          type="button"
          @click="loadDashboard(true)"
          :disabled="refreshing"
          class="inline-flex items-center gap-1.5 rounded-xl border border-zinc-200 bg-white px-3 py-2 text-xs font-semibold text-zinc-700 hover:bg-zinc-50 transition-colors shadow-2xs disabled:opacity-50 cursor-pointer"
          title="Refresh statistics"
        >
          <svg
            class="h-3.5 w-3.5 text-zinc-500"
            :class="{ 'animate-spin': refreshing }"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="1.8"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"
            />
          </svg>
          <span>Refresh</span>
        </button>

        <RouterLink
          v-if="canCreateSalesOrder"
          to="/sales-orders"
          class="inline-flex items-center gap-1.5 rounded-xl bg-zinc-900 px-3.5 py-2 text-xs font-semibold text-white hover:bg-zinc-800 transition-colors shadow-2xs"
        >
          <svg
            class="h-3.5 w-3.5"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2.2"
          >
            <path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" />
          </svg>
          <span>New Sales Order</span>
        </RouterLink>

        <RouterLink
          v-if="canCreatePurchaseOrder"
          to="/purchase-orders"
          class="inline-flex items-center gap-1.5 rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs font-semibold text-zinc-800 hover:bg-zinc-50 transition-colors shadow-2xs"
        >
          <svg
            class="h-3.5 w-3.5"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2.2"
          >
            <path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" />
          </svg>
          <span>New Purchase Order</span>
        </RouterLink>
      </div>
    </div>

    <!-- Error Banner -->
    <div
      v-if="error"
      class="flex items-center justify-between rounded-2xl bg-rose-50 p-4 border border-rose-200/70 text-rose-800 text-sm"
    >
      <div class="flex items-center gap-3">
        <svg
          class="h-5 w-5 text-rose-600 shrink-0"
          fill="none"
          viewBox="0 0 24 24"
          stroke="currentColor"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
          />
        </svg>
        <span>{{ error }}</span>
      </div>
      <button
        type="button"
        @click="loadDashboard()"
        class="text-xs font-semibold underline hover:text-rose-900 cursor-pointer"
      >
        Retry
      </button>
    </div>

    <!-- Skeleton Loading State -->
    <div v-if="loading && !data" class="space-y-6">
      <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <div
          v-for="i in 4"
          :key="i"
          class="h-32 rounded-2xl bg-white p-5 shadow-2xs border border-zinc-100 animate-pulse"
        >
          <div class="h-4 w-24 bg-zinc-200 rounded"></div>
          <div class="mt-4 h-8 w-32 bg-zinc-200 rounded"></div>
          <div class="mt-2 h-3 w-40 bg-zinc-100 rounded"></div>
        </div>
      </div>
      <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
        <div
          class="h-64 rounded-2xl bg-white p-5 shadow-2xs border border-zinc-100 animate-pulse"
        ></div>
        <div
          class="h-64 rounded-2xl bg-white p-5 shadow-2xs border border-zinc-100 animate-pulse"
        ></div>
      </div>
    </div>

    <!-- Main Dashboard Content -->
    <template v-if="data">
      <!-- ============================================================ -->
      <!-- TOP EXECUTIVE KPI METRICS                                    -->
      <!-- ============================================================ -->
      <div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <!-- KPI 1: Inventory Valuation -->
        <div
          class="relative overflow-hidden rounded-2xl bg-white p-5 shadow-2xs border border-zinc-200/80 transition-all hover:border-zinc-300"
        >
          <div class="flex items-center justify-between">
            <span class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">
              Inventory Value (Cost)
            </span>
            <div
              class="flex h-9 w-9 items-center justify-center rounded-xl bg-zinc-100 text-zinc-700"
            >
              <svg
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="1.8"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M12 8c-1.657 0-3 .895-3 2s1.343 2 3 2 3 .895 3 2-1.343 2-3 2m0-8c1.11 0 2.08.402 2.599 1M12 8V7m0 1v8m0 0v1m0-1c-1.11 0-2.08-.402-2.599-1M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
                />
              </svg>
            </div>
          </div>
          <div class="mt-3">
            <p class="text-2xl sm:text-3xl font-bold tracking-tight text-zinc-900">
              {{ money(data.metrics.cost_value) }}
            </p>
            <div class="mt-2 flex items-center justify-between text-xs text-zinc-500">
              <span>Retail: {{ money(data.metrics.retail_value) }}</span>
              <span
                class="inline-flex items-center gap-1 font-semibold text-emerald-700 bg-emerald-50 px-2 py-0.5 rounded-full text-[11px] ring-1 ring-emerald-600/20"
                title="Potential profit margin across physical stock"
              >
                +{{ data.metrics.margin_percentage }}% margin
              </span>
            </div>
          </div>
        </div>

        <!-- KPI 2: Stock Units Volume -->
        <div
          class="relative overflow-hidden rounded-2xl bg-white p-5 shadow-2xs border border-zinc-200/80 transition-all hover:border-zinc-300"
        >
          <div class="flex items-center justify-between">
            <span class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">
              Physical Stock Volume
            </span>
            <div
              class="flex h-9 w-9 items-center justify-center rounded-xl bg-zinc-100 text-zinc-700"
            >
              <svg
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="1.8"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M20 7l-8-4-8 4m16 0l-8 4m8-4v10l-8 4m0-10L4 7m8 4v10M4 7v10l8 4"
                />
              </svg>
            </div>
          </div>
          <div class="mt-3">
            <p class="text-2xl sm:text-3xl font-bold tracking-tight text-zinc-900">
              {{ (data.metrics.total_units || 0).toLocaleString() }}
              <span class="text-sm font-normal text-zinc-500">units</span>
            </p>
            <div class="mt-2 flex items-center justify-between text-xs text-zinc-500">
              <span>In {{ data.metrics.warehouses_count }} Warehouses</span>
              <span class="text-zinc-600 font-medium">
                {{ data.metrics.products_count }} SKUs in {{ data.metrics.categories_count }} Cats
              </span>
            </div>
          </div>
        </div>

        <!-- KPI 3: Sales Orders & Revenue -->
        <div
          class="relative overflow-hidden rounded-2xl bg-white p-5 shadow-2xs border border-zinc-200/80 transition-all hover:border-zinc-300"
        >
          <div class="flex items-center justify-between">
            <span class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">
              Sales & Fulfillment
            </span>
            <div
              class="flex h-9 w-9 items-center justify-center rounded-xl bg-zinc-100 text-zinc-700"
            >
              <svg
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="1.8"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M16 11V7a4 4 0 00-8 0v4M5 9h14l1 12H4L5 9z"
                />
              </svg>
            </div>
          </div>
          <div class="mt-3">
            <p class="text-2xl sm:text-3xl font-bold tracking-tight text-zinc-900">
              {{ money(data.metrics.sales_orders.revenue) }}
            </p>
            <div class="mt-2 flex items-center justify-between text-xs text-zinc-500">
              <span>{{ data.metrics.sales_orders.total }} Orders total</span>
              <span
                v-if="data.metrics.sales_orders.pending > 0"
                class="font-semibold text-blue-700 bg-blue-50 px-2 py-0.5 rounded-full text-[11px] ring-1 ring-blue-600/20"
              >
                {{ data.metrics.sales_orders.pending }} pending dispatch
              </span>
              <span v-else class="text-zinc-500 font-medium">All shipped</span>
            </div>
          </div>
        </div>

        <!-- KPI 4: Stock Health & Low Stock Alerts -->
        <div
          class="relative overflow-hidden rounded-2xl bg-white p-5 shadow-2xs border border-zinc-200/80 transition-all hover:border-zinc-300"
        >
          <div class="flex items-center justify-between">
            <span class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">
              Stock Health & Alerts
            </span>
            <div
              class="flex h-9 w-9 items-center justify-center rounded-xl"
              :class="
                data.metrics.low_stock_count > 0
                  ? 'bg-rose-50 text-rose-600'
                  : 'bg-emerald-50 text-emerald-600'
              "
            >
              <svg
                v-if="data.metrics.low_stock_count > 0"
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="2"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
                />
              </svg>
              <svg
                v-else
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="2"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"
                />
              </svg>
            </div>
          </div>
          <div class="mt-3">
            <p
              class="text-2xl sm:text-3xl font-bold tracking-tight"
              :class="data.metrics.low_stock_count > 0 ? 'text-rose-600' : 'text-zinc-900'"
            >
              {{ data.metrics.low_stock_count }}
              <span class="text-sm font-normal text-zinc-500">items low</span>
            </p>
            <div class="mt-2 flex items-center justify-between text-xs text-zinc-500">
              <span v-if="data.metrics.out_of_stock_count > 0" class="text-rose-600 font-semibold">
                {{ data.metrics.out_of_stock_count }} out of stock
              </span>
              <span v-else class="text-emerald-600 font-medium">None out of stock</span>

              <span class="text-zinc-600">
                {{ data.metrics.purchase_orders.pending }} POs in transit
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- ============================================================ -->
      <!-- QUICK SHORTCUTS NAVIGATION ROW                               -->
      <!-- ============================================================ -->
      <div
        class="grid grid-cols-2 gap-2.5 sm:grid-cols-3 lg:grid-cols-6 rounded-2xl bg-white p-2.5 shadow-2xs border border-zinc-200/80 text-xs font-medium text-zinc-700"
      >
        <RouterLink
          to="/stock"
          class="flex items-center gap-2.5 p-2 rounded-xl hover:bg-zinc-100/70 hover:text-zinc-900 transition-colors group"
        >
          <div
            class="p-1.5 rounded-lg bg-zinc-100 text-zinc-600 group-hover:bg-zinc-900 group-hover:text-white transition-colors"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="1.8"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M8 7h12m0 0l-4-4m4 4l-4 4m0 6H4m0 0l4 4m-4-4l4-4"
              />
            </svg>
          </div>
          <span>Stock Adjust & Transfer</span>
        </RouterLink>

        <RouterLink
          to="/sales-orders"
          class="flex items-center gap-2.5 p-2 rounded-xl hover:bg-zinc-100/70 hover:text-zinc-900 transition-colors group"
        >
          <div
            class="p-1.5 rounded-lg bg-zinc-100 text-zinc-600 group-hover:bg-zinc-900 group-hover:text-white transition-colors"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="1.8"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2"
              />
            </svg>
          </div>
          <span>Manage Sales Orders</span>
        </RouterLink>

        <RouterLink
          to="/purchase-orders"
          class="flex items-center gap-2.5 p-2 rounded-xl hover:bg-zinc-100/70 hover:text-zinc-900 transition-colors group"
        >
          <div
            class="p-1.5 rounded-lg bg-zinc-100 text-zinc-600 group-hover:bg-zinc-900 group-hover:text-white transition-colors"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="1.8"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M3 3h2l.4 2M7 13h10l4-8H5.4M7 13L5.4 5M7 13l-2.293 2.293c-.63.63-.184 1.707.707 1.707H17m0 0a2 2 0 100 4 2 2 0 000-4zm-8 2a2 2 0 11-4 0 2 2 0 014 0z"
              />
            </svg>
          </div>
          <span>Procure & Receive POs</span>
        </RouterLink>

        <RouterLink
          to="/reports"
          class="flex items-center gap-2.5 p-2 rounded-xl hover:bg-zinc-100/70 hover:text-zinc-900 transition-colors group"
        >
          <div
            class="p-1.5 rounded-lg bg-zinc-100 text-zinc-600 group-hover:bg-zinc-900 group-hover:text-white transition-colors"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="1.8"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z"
              />
            </svg>
          </div>
          <span>Inventory Valuation</span>
        </RouterLink>

        <RouterLink
          to="/customers"
          class="flex items-center gap-2.5 p-2 rounded-xl hover:bg-zinc-100/70 hover:text-zinc-900 transition-colors group"
        >
          <div
            class="p-1.5 rounded-lg bg-zinc-100 text-zinc-600 group-hover:bg-zinc-900 group-hover:text-white transition-colors"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="1.8"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0M15 7a3 3 0 11-6 0 3 3 0 016 0zm6 3a2 2 0 11-4 0 2 2 0 014 0zM7 10a2 2 0 11-4 0 2 2 0 014 0z"
              />
            </svg>
          </div>
          <span>Customer Registry</span>
        </RouterLink>

        <button
          type="button"
          @click="openScanModal"
          class="flex items-center gap-2.5 p-2 rounded-xl hover:bg-zinc-100/70 hover:text-zinc-900 transition-colors text-left cursor-pointer group"
        >
          <div
            class="p-1.5 rounded-lg bg-zinc-100 text-zinc-600 group-hover:bg-zinc-900 group-hover:text-white transition-colors"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="1.8"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M12 4v1m6 11h2m-6 0h-2v4m0-11v3m0 0h.01M12 12h4.01M16 20h4M4 12h4m12 0h.01M5 8h2a1 1 0 001-1V5a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1zm12 0h2a1 1 0 001-1V5a1 1 0 00-1-1h-2a1 1 0 00-1 1v2a1 1 0 001 1zM5 20h2a1 1 0 001-1v-2a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1z"
              />
            </svg>
          </div>
          <span>Barcode Quick Scan</span>
        </button>
      </div>

      <!-- ============================================================ -->
      <!-- CRITICAL ATTENTION: LOW STOCK & REORDER ALERTS               -->
      <!-- ============================================================ -->
      <div
        v-if="data.low_stock_items && data.low_stock_items.length > 0"
        class="rounded-2xl bg-white shadow-2xs border border-zinc-200/80 overflow-hidden"
      >
        <div
          class="flex flex-col sm:flex-row sm:items-center sm:justify-between px-5 py-3.5 bg-rose-50/50 border-b border-rose-100/80 gap-2"
        >
          <div class="flex items-center gap-2.5">
            <div
              class="flex h-7 w-7 items-center justify-center rounded-lg bg-rose-100 text-rose-700"
            >
              <svg
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="2"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
                />
              </svg>
            </div>
            <div>
              <h2 class="text-xs sm:text-sm font-bold text-zinc-900">
                Action Required: Low Stock & Reorder Alerts
              </h2>
              <p class="text-[11px] text-rose-700">
                {{ data.low_stock_items.length }} items are at or below their safety reorder
                thresholds
              </p>
            </div>
          </div>

          <RouterLink
            to="/products"
            class="text-xs font-semibold text-rose-700 hover:text-rose-900 flex items-center gap-1 self-start sm:self-auto"
          >
            <span>View all products catalog</span>
            <svg class="h-3.5 w-3.5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M9 5l7 7-7 7"
              />
            </svg>
          </RouterLink>
        </div>

        <div class="overflow-x-auto">
          <table class="w-full text-left text-xs text-zinc-600">
            <thead
              class="bg-zinc-50/75 border-b border-zinc-100 font-semibold uppercase tracking-wider text-zinc-400 text-[11px]"
            >
              <tr>
                <th scope="col" class="py-2.5 px-5">Product & SKU</th>
                <th scope="col" class="py-2.5 px-4">Category</th>
                <th scope="col" class="py-2.5 px-4 text-right">In Stock</th>
                <th scope="col" class="py-2.5 px-4 text-right">Reorder Level</th>
                <th scope="col" class="py-2.5 px-4 text-right">Deficit</th>
                <th scope="col" class="py-2.5 px-4 text-right">Cost Value</th>
                <th scope="col" class="py-2.5 px-5 text-right">Actions</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-zinc-100">
              <tr
                v-for="item in data.low_stock_items"
                :key="item.id"
                class="hover:bg-zinc-50/60 transition-colors"
              >
                <td class="py-3 px-5 font-medium text-zinc-900">
                  <button
                    type="button"
                    @click="openProductDetail(item)"
                    class="text-left group transition-colors cursor-pointer"
                  >
                    <div class="font-semibold text-zinc-900 group-hover:text-zinc-600">
                      {{ item.name }}
                    </div>
                    <div class="text-[11px] font-mono text-zinc-400">
                      {{ item.sku }}
                      <span v-if="item.barcode" class="ml-1 text-zinc-300"
                        >| {{ item.barcode }}</span
                      >
                    </div>
                  </button>
                </td>
                <td class="py-3 px-4">
                  <span
                    class="inline-block rounded-md bg-zinc-100 px-2 py-0.5 text-[11px] font-medium text-zinc-700"
                  >
                    {{ item.category_name }}
                  </span>
                </td>
                <td class="py-3 px-4 text-right">
                  <span
                    class="inline-flex items-center gap-1 font-semibold rounded-full px-2 py-0.5 text-xs"
                    :class="
                      item.stock_total === 0
                        ? 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20'
                        : 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20'
                    "
                  >
                    {{ item.stock_total }} {{ item.unit }}
                  </span>
                </td>
                <td class="py-3 px-4 text-right font-medium text-zinc-600">
                  {{ item.reorder_level }} {{ item.unit }}
                </td>
                <td class="py-3 px-4 text-right font-semibold text-rose-600">
                  -{{ item.deficit }} {{ item.unit }}
                </td>
                <td class="py-3 px-4 text-right font-medium text-zinc-700">
                  {{ money(item.cost_price * item.stock_total) }}
                </td>
                <td class="py-3 px-5 text-right">
                  <div class="flex items-center justify-end gap-1.5">
                    <button
                      type="button"
                      @click="openProductDetail(item)"
                      class="rounded-lg border border-zinc-200 bg-white px-2.5 py-1 text-[11px] font-semibold text-zinc-700 hover:bg-zinc-50 transition-colors cursor-pointer shadow-2xs"
                    >
                      Inspect
                    </button>
                    <RouterLink
                      v-if="canCreatePurchaseOrder"
                      to="/purchase-orders"
                      class="rounded-lg bg-zinc-900 px-2.5 py-1 text-[11px] font-semibold text-white hover:bg-zinc-800 transition-colors shadow-2xs"
                    >
                      + Create PO
                    </RouterLink>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Healthy Stock State (when no low stock items) -->
      <div
        v-else
        class="flex items-center justify-between rounded-2xl bg-emerald-50/60 p-4 border border-emerald-200/70 text-emerald-900"
      >
        <div class="flex items-center gap-3">
          <div
            class="flex h-8 w-8 items-center justify-center rounded-xl bg-emerald-100 text-emerald-700"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="2.2"
            >
              <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
            </svg>
          </div>
          <div>
            <p class="text-xs sm:text-sm font-bold text-zinc-900">Inventory Levels are Optimal</p>
            <p class="text-xs text-emerald-700">
              All products have sufficient on-hand stock exceeding safety reorder points.
            </p>
          </div>
        </div>
        <RouterLink
          to="/products"
          class="text-xs font-semibold text-emerald-700 hover:text-emerald-900 underline"
        >
          View Catalog
        </RouterLink>
      </div>

      <!-- ============================================================ -->
      <!-- DISTRIBUTION ANALYTICS: WAREHOUSES & CATEGORIES               -->
      <!-- ============================================================ -->
      <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
        <!-- Warehouse Capacity & Distribution Card -->
        <div
          class="rounded-2xl bg-white p-5 sm:p-6 shadow-2xs border border-zinc-200/80 flex flex-col justify-between"
        >
          <div>
            <div class="flex items-center justify-between pb-3.5 border-b border-zinc-100">
              <div class="flex items-center gap-2">
                <svg
                  class="h-4 w-4 text-zinc-700"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke="currentColor"
                  stroke-width="1.8"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M19 21V5a2 2 0 00-2-2H7a2 2 0 00-2 2v16m14 0h2m-2 0h-5m-9 0H3m2 0h5M9 7h1m-1 4h1m4-4h1m-1 4h1m-5 10v-5a1 1 0 011-1h2a1 1 0 011 1v5m-4 0h4"
                  />
                </svg>
                <h2 class="text-sm font-bold text-zinc-900">Warehouse Storage & Valuation</h2>
              </div>
              <RouterLink
                to="/warehouses"
                class="text-xs font-semibold text-zinc-600 hover:text-zinc-900"
              >
                Manage Warehouses &rarr;
              </RouterLink>
            </div>

            <div class="mt-4 space-y-3">
              <div
                v-for="w in data.warehouses"
                :key="w.warehouse_id"
                class="rounded-xl border border-zinc-200/70 p-3 bg-zinc-50/50 hover:bg-zinc-50 transition-colors"
              >
                <div class="flex items-center justify-between text-xs">
                  <div class="font-semibold text-zinc-900 flex items-center gap-2">
                    <span>{{ w.warehouse_name }}</span>
                    <span class="text-[11px] font-normal text-zinc-400"
                      >({{ w.products_count || 0 }} SKUs)</span
                    >
                  </div>
                  <div class="text-right">
                    <span class="font-bold text-zinc-900"
                      >{{ (w.units || 0).toLocaleString() }} units</span
                    >
                    <span class="text-zinc-400 ml-1.5 font-medium"
                      >({{ w.share_percentage }}%)</span
                    >
                  </div>
                </div>

                <!-- Capacity / Share Progress Bar -->
                <div class="mt-2.5 h-1.5 w-full overflow-hidden rounded-full bg-zinc-200/70">
                  <div
                    class="h-full rounded-full bg-zinc-900 transition-all duration-500"
                    :style="{ width: `${Math.min(100, Math.max(0, w.share_percentage))}%` }"
                  ></div>
                </div>

                <div class="mt-2 flex items-center justify-between text-[11px] text-zinc-500">
                  <span>Value at cost: {{ money(w.cost_value) }}</span>
                  <span>Retail value: {{ money(w.retail_value) }}</span>
                </div>
              </div>
            </div>
          </div>

          <div
            class="mt-4 pt-3.5 border-t border-zinc-100 flex items-center justify-between text-xs text-zinc-500"
          >
            <span>{{ data.warehouses.length }} storage locations active</span>
            <RouterLink
              to="/stock"
              class="font-semibold text-zinc-700 hover:text-zinc-900 hover:underline"
            >
              Inter-warehouse transfer &rarr;
            </RouterLink>
          </div>
        </div>

        <!-- Category Mix & Product Distribution Card -->
        <div
          class="rounded-2xl bg-white p-5 sm:p-6 shadow-2xs border border-zinc-200/80 flex flex-col justify-between"
        >
          <div>
            <div class="flex items-center justify-between pb-3.5 border-b border-zinc-100">
              <div class="flex items-center gap-2">
                <svg
                  class="h-4 w-4 text-zinc-700"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke="currentColor"
                  stroke-width="1.8"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M7 7h.01M7 3h5c.512 0 1.024.195 1.414.586l7 7a2 2 0 010 2.828l-7 7a2 2 0 01-2.828 0l-7-7A1.994 1.994 0 013 12V7a4 4 0 014-4z"
                  />
                </svg>
                <h2 class="text-sm font-bold text-zinc-900">Top Categories by Valuation</h2>
              </div>
              <RouterLink
                to="/categories"
                class="text-xs font-semibold text-zinc-600 hover:text-zinc-900"
              >
                View All &rarr;
              </RouterLink>
            </div>

            <div class="mt-4 space-y-3">
              <div
                v-for="(cat, idx) in data.categories.slice(0, 5)"
                :key="cat.category_id ?? idx"
                class="rounded-xl border border-zinc-200/70 p-3 bg-zinc-50/50 hover:bg-zinc-50 transition-colors"
              >
                <div class="flex items-center justify-between text-xs">
                  <div class="font-semibold text-zinc-900 flex items-center gap-2">
                    <span>{{ cat.category_name }}</span>
                    <span class="text-[11px] font-normal text-zinc-400"
                      >({{ cat.products_count || 0 }} products)</span
                    >
                  </div>
                  <div class="text-right">
                    <span class="font-bold text-zinc-900">{{ money(cat.cost_value) }}</span>
                    <span class="text-zinc-400 ml-1.5 font-medium"
                      >({{ cat.share_percentage }}%)</span
                    >
                  </div>
                </div>

                <!-- Proportional Bar -->
                <div class="mt-2.5 h-1.5 w-full overflow-hidden rounded-full bg-zinc-200/70">
                  <div
                    class="h-full rounded-full bg-zinc-800 transition-all duration-500"
                    :style="{ width: `${Math.min(100, Math.max(0, cat.share_percentage))}%` }"
                  ></div>
                </div>

                <div class="mt-2 flex items-center justify-between text-[11px] text-zinc-500">
                  <span>{{ (cat.units || 0).toLocaleString() }} units on shelf</span>
                  <span>Retail: {{ money(cat.retail_value) }}</span>
                </div>
              </div>
            </div>
          </div>

          <div
            class="mt-4 pt-3.5 border-t border-zinc-100 flex items-center justify-between text-xs text-zinc-500"
          >
            <span
              >30-Day Velocity: +{{ data.metrics.movements_last_30_days.in_units }} / -{{
                data.metrics.movements_last_30_days.out_units
              }}
              units</span
            >
            <RouterLink
              to="/reports"
              class="font-semibold text-zinc-700 hover:text-zinc-900 hover:underline"
            >
              Full Valuation Report &rarr;
            </RouterLink>
          </div>
        </div>
      </div>

      <!-- ============================================================ -->
      <!-- OPERATIONS STREAM & AUDIT FEED                               -->
      <!-- ============================================================ -->
      <div class="grid grid-cols-1 gap-6 lg:grid-cols-3">
        <!-- Tabbed Operations Table (Sales, Purchase, Movements) (2 cols) -->
        <div
          class="lg:col-span-2 rounded-2xl bg-white shadow-2xs border border-zinc-200/80 overflow-hidden"
        >
          <div
            class="flex items-center justify-between px-5 py-3.5 border-b border-zinc-100 bg-zinc-50/60"
          >
            <!-- Tabs -->
            <div class="flex items-center gap-1.5 p-1 bg-zinc-100 rounded-xl">
              <button
                type="button"
                @click="activeTab = 'sales'"
                class="rounded-lg px-3 py-1.5 text-xs font-semibold transition-all flex items-center gap-1.5 cursor-pointer"
                :class="
                  activeTab === 'sales'
                    ? 'bg-white text-zinc-900 shadow-2xs'
                    : 'text-zinc-500 hover:text-zinc-900'
                "
              >
                <span>Recent Sales</span>
                <span class="rounded-full bg-zinc-200/70 px-1.5 py-0.2 text-[10px] text-zinc-700">
                  {{ data.recent_sales_orders?.length || 0 }}
                </span>
              </button>

              <button
                type="button"
                @click="activeTab = 'purchases'"
                class="rounded-lg px-3 py-1.5 text-xs font-semibold transition-all flex items-center gap-1.5 cursor-pointer"
                :class="
                  activeTab === 'purchases'
                    ? 'bg-white text-zinc-900 shadow-2xs'
                    : 'text-zinc-500 hover:text-zinc-900'
                "
              >
                <span>Recent Purchases</span>
                <span class="rounded-full bg-zinc-200/70 px-1.5 py-0.2 text-[10px] text-zinc-700">
                  {{ data.recent_purchase_orders?.length || 0 }}
                </span>
              </button>

              <button
                type="button"
                @click="activeTab = 'movements'"
                class="rounded-lg px-3 py-1.5 text-xs font-semibold transition-all flex items-center gap-1.5 cursor-pointer"
                :class="
                  activeTab === 'movements'
                    ? 'bg-white text-zinc-900 shadow-2xs'
                    : 'text-zinc-500 hover:text-zinc-900'
                "
              >
                <span>Stock Movements</span>
                <span class="rounded-full bg-zinc-200/70 px-1.5 py-0.2 text-[10px] text-zinc-700">
                  {{ data.recent_movements?.length || 0 }}
                </span>
              </button>
            </div>

            <!-- View All Link for Current Tab -->
            <RouterLink
              v-if="activeTab === 'sales'"
              to="/sales-orders"
              class="text-xs font-semibold text-zinc-600 hover:text-zinc-900"
            >
              All Orders &rarr;
            </RouterLink>
            <RouterLink
              v-else-if="activeTab === 'purchases'"
              to="/purchase-orders"
              class="text-xs font-semibold text-zinc-600 hover:text-zinc-900"
            >
              All POs &rarr;
            </RouterLink>
            <RouterLink
              v-else
              to="/stock"
              class="text-xs font-semibold text-zinc-600 hover:text-zinc-900"
            >
              All Movements &rarr;
            </RouterLink>
          </div>

          <!-- TAB 1: SALES ORDERS TABLE -->
          <div v-if="activeTab === 'sales'" class="overflow-x-auto">
            <table class="w-full text-left text-xs text-zinc-600">
              <thead
                class="bg-zinc-50/75 border-b border-zinc-100 font-semibold uppercase tracking-wider text-zinc-400 text-[11px]"
              >
                <tr>
                  <th scope="col" class="py-2.5 px-5">Order #</th>
                  <th scope="col" class="py-2.5 px-4">Customer</th>
                  <th scope="col" class="py-2.5 px-4">Date</th>
                  <th scope="col" class="py-2.5 px-4">Status</th>
                  <th scope="col" class="py-2.5 px-4 text-right">Total</th>
                  <th scope="col" class="py-2.5 px-5 text-right">Actions</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!data.recent_sales_orders || !data.recent_sales_orders.length">
                  <td colspan="6" class="py-8 text-center text-zinc-400">
                    No sales orders created yet.
                  </td>
                </tr>
                <tr
                  v-for="so in data.recent_sales_orders"
                  :key="so.id"
                  class="hover:bg-zinc-50/60 transition-colors"
                >
                  <td class="py-3 px-5 font-semibold text-zinc-900 font-mono">
                    {{ so.number }}
                  </td>
                  <td class="py-3 px-4 font-medium text-zinc-800">
                    {{ so.customer_name }}
                  </td>
                  <td class="py-3 px-4 text-zinc-500">
                    {{ formatDate(so.order_date) }}
                  </td>
                  <td class="py-3 px-4">
                    <span
                      class="inline-block rounded-full px-2 py-0.5 text-[10px] font-semibold capitalize"
                      :class="salesStatusStyles[so.status] || 'bg-zinc-100 text-zinc-700'"
                    >
                      {{ so.status?.replace('_', ' ') }}
                    </span>
                  </td>
                  <td class="py-3 px-4 text-right font-bold text-zinc-900">
                    {{ money(so.total) }}
                  </td>
                  <td class="py-3 px-5 text-right">
                    <RouterLink
                      to="/sales-orders"
                      class="text-zinc-600 hover:text-zinc-900 font-semibold text-[11px]"
                    >
                      Open &rarr;
                    </RouterLink>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- TAB 2: PURCHASE ORDERS TABLE -->
          <div v-if="activeTab === 'purchases'" class="overflow-x-auto">
            <table class="w-full text-left text-xs text-zinc-600">
              <thead
                class="bg-zinc-50/75 border-b border-zinc-100 font-semibold uppercase tracking-wider text-zinc-400 text-[11px]"
              >
                <tr>
                  <th scope="col" class="py-2.5 px-5">PO Number</th>
                  <th scope="col" class="py-2.5 px-4">Supplier</th>
                  <th scope="col" class="py-2.5 px-4">Expected Date</th>
                  <th scope="col" class="py-2.5 px-4">Status</th>
                  <th scope="col" class="py-2.5 px-4 text-right">Total</th>
                  <th scope="col" class="py-2.5 px-5 text-right">Actions</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!data.recent_purchase_orders || !data.recent_purchase_orders.length">
                  <td colspan="6" class="py-8 text-center text-zinc-400">
                    No purchase orders recorded yet.
                  </td>
                </tr>
                <tr
                  v-for="po in data.recent_purchase_orders"
                  :key="po.id"
                  class="hover:bg-zinc-50/60 transition-colors"
                >
                  <td class="py-3 px-5 font-semibold text-zinc-900 font-mono">
                    {{ po.number }}
                  </td>
                  <td class="py-3 px-4 font-medium text-zinc-800">
                    {{ po.supplier_name }}
                  </td>
                  <td class="py-3 px-4 text-zinc-500">
                    {{ formatDate(po.order_date) }}
                  </td>
                  <td class="py-3 px-4">
                    <span
                      class="inline-block rounded-full px-2 py-0.5 text-[10px] font-semibold capitalize"
                      :class="purchaseStatusStyles[po.status] || 'bg-zinc-100 text-zinc-700'"
                    >
                      {{ po.status?.replace('_', ' ') }}
                    </span>
                  </td>
                  <td class="py-3 px-4 text-right font-bold text-zinc-900">
                    {{ money(po.total) }}
                  </td>
                  <td class="py-3 px-5 text-right">
                    <RouterLink
                      to="/purchase-orders"
                      class="text-zinc-600 hover:text-zinc-900 font-semibold text-[11px]"
                    >
                      Open &rarr;
                    </RouterLink>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- TAB 3: STOCK MOVEMENTS TABLE -->
          <div v-if="activeTab === 'movements'" class="overflow-x-auto">
            <table class="w-full text-left text-xs text-zinc-600">
              <thead
                class="bg-zinc-50/75 border-b border-zinc-100 font-semibold uppercase tracking-wider text-zinc-400 text-[11px]"
              >
                <tr>
                  <th scope="col" class="py-2.5 px-5">Product & SKU</th>
                  <th scope="col" class="py-2.5 px-4">Warehouse</th>
                  <th scope="col" class="py-2.5 px-4">Type</th>
                  <th scope="col" class="py-2.5 px-4 text-right">Quantity</th>
                  <th scope="col" class="py-2.5 px-4 text-right">Balance</th>
                  <th scope="col" class="py-2.5 px-5 text-right">Time</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!data.recent_movements || !data.recent_movements.length">
                  <td colspan="6" class="py-8 text-center text-zinc-400">
                    No stock movements recorded yet.
                  </td>
                </tr>
                <tr
                  v-for="m in data.recent_movements"
                  :key="m.id"
                  class="hover:bg-zinc-50/60 transition-colors"
                >
                  <td class="py-3 px-5 font-medium text-zinc-900">
                    <div>{{ m.product_name }}</div>
                    <div class="text-[11px] font-mono text-zinc-400">{{ m.product_sku }}</div>
                  </td>
                  <td class="py-3 px-4 text-zinc-600">
                    {{ m.warehouse_name }}
                  </td>
                  <td class="py-3 px-4">
                    <span
                      class="inline-block rounded-full px-2 py-0.5 text-[10px] font-semibold uppercase tracking-wider"
                      :class="movementStyles[m.type] || 'bg-zinc-100 text-zinc-700'"
                    >
                      {{ m.type?.replace('_', ' ') }}
                    </span>
                  </td>
                  <td
                    class="py-3 px-4 text-right font-bold"
                    :class="m.quantity > 0 ? 'text-emerald-600' : 'text-rose-600'"
                  >
                    {{ m.quantity > 0 ? `+${m.quantity}` : m.quantity }}
                  </td>
                  <td class="py-3 px-4 text-right font-medium text-zinc-800">
                    {{ m.balance_after }}
                  </td>
                  <td class="py-3 px-5 text-right text-zinc-400">
                    {{ timeAgo(m.created_at) }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Live Activity Stream / Audit Feed (1 col) -->
        <div
          class="rounded-2xl bg-white shadow-2xs border border-zinc-200/80 p-5 sm:p-6 flex flex-col justify-between"
        >
          <div>
            <div class="flex items-center justify-between pb-3.5 border-b border-zinc-100">
              <div class="flex items-center gap-2">
                <svg
                  class="h-4 w-4 text-zinc-700"
                  fill="none"
                  viewBox="0 0 24 24"
                  stroke="currentColor"
                  stroke-width="1.8"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"
                  />
                </svg>
                <h2 class="text-sm font-bold text-zinc-900">Live Activity Feed</h2>
              </div>
              <RouterLink
                v-if="canViewActivity"
                to="/activity"
                class="text-xs font-semibold text-zinc-600 hover:text-zinc-900"
              >
                Full Log &rarr;
              </RouterLink>
            </div>

            <!-- Activity List (Admin & Manager) -->
            <div
              v-if="data.recent_activities && data.recent_activities.length"
              class="mt-4 space-y-2.5"
            >
              <div
                v-for="act in data.recent_activities"
                :key="act.id"
                class="flex items-start gap-3 rounded-xl border border-zinc-200/70 p-2.5 bg-zinc-50/50 hover:bg-zinc-50 transition-colors text-xs"
              >
                <div
                  class="flex h-7 w-7 shrink-0 items-center justify-center rounded-lg bg-zinc-900 text-white font-semibold text-[10px]"
                >
                  {{ (act.user_name || 'U').slice(0, 2).toUpperCase() }}
                </div>
                <div class="flex-1 min-w-0">
                  <div class="flex items-center justify-between gap-1">
                    <span class="font-semibold text-zinc-900 truncate">{{ act.user_name }}</span>
                    <span
                      class="rounded px-1.5 py-0.2 text-[9px] font-semibold uppercase tracking-wider shrink-0"
                      :class="activityStyles[act.action] || 'bg-zinc-100 text-zinc-700'"
                    >
                      {{ act.action?.replace('_', ' ') }}
                    </span>
                  </div>
                  <p class="mt-0.5 text-[11px] text-zinc-600 line-clamp-2">
                    {{ act.description }}
                  </p>
                  <p class="mt-1 text-[10px] text-zinc-400">
                    {{ timeAgo(act.created_at) }}
                  </p>
                </div>
              </div>
            </div>

            <!-- Empty / Staff Activity Card -->
            <div
              v-else
              class="mt-6 text-center py-6 px-4 bg-zinc-50 rounded-xl border border-zinc-100"
            >
              <div
                class="flex h-9 w-9 mx-auto items-center justify-center rounded-full bg-zinc-100 text-zinc-600"
              >
                <svg class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="1.8"
                    d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
                  />
                </svg>
              </div>
              <p class="mt-2 text-xs font-semibold text-zinc-800">Operational Log</p>
              <p class="mt-1 text-[11px] text-zinc-500">
                System events and transaction histories are continuously captured for operational
                audits.
              </p>
            </div>
          </div>

          <!-- Bottom quick hint / tip -->
          <div
            class="mt-4 pt-3 border-t border-zinc-100 text-[11px] text-zinc-500 flex items-center justify-between"
          >
            <span
              >Tip: Press
              <kbd class="px-1 border border-zinc-200 rounded font-mono text-[10px] bg-zinc-50"
                >/</kbd
              >
              anywhere to scan items</span
            >
            <span class="text-emerald-600 font-medium text-[11px]">● Connected</span>
          </div>
        </div>
      </div>
    </template>

    <!-- ============================================================ -->
    <!-- QUICK BARCODE SCANNER MODAL                                  -->
    <!-- ============================================================ -->
    <div
      v-if="showScanModal"
      class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-zinc-950/40 backdrop-blur-xs animate-in fade-in duration-200"
      @click.self="showScanModal = false"
    >
      <div
        class="w-full max-w-lg rounded-2xl bg-white shadow-xl border border-zinc-200/80 p-6 space-y-4"
        role="dialog"
        aria-modal="true"
        aria-labelledby="scan-modal-title"
      >
        <div class="flex items-center justify-between pb-3.5 border-b border-zinc-100">
          <div class="flex items-center gap-2.5">
            <div
              class="flex h-9 w-9 items-center justify-center rounded-xl bg-zinc-100 text-zinc-700"
            >
              <svg
                class="h-5 w-5"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="1.8"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M12 4v1m6 11h2m-6 0h-2v4m0-11v3m0 0h.01M12 12h4.01M16 20h4M4 12h4m12 0h.01M5 8h2a1 1 0 001-1V5a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1zm12 0h2a1 1 0 001-1V5a1 1 0 00-1-1h-2a1 1 0 00-1 1v2a1 1 0 001 1zM5 20h2a1 1 0 001-1v-2a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1z"
                />
              </svg>
            </div>
            <div>
              <h2 id="scan-modal-title" class="text-sm font-bold text-zinc-900">
                Quick Barcode / SKU Scanner
              </h2>
              <p class="text-xs text-zinc-500">
                Scan hardware barcode or type SKU / code and press Enter
              </p>
            </div>
          </div>
          <button
            type="button"
            @click="showScanModal = false"
            class="rounded-lg p-1.5 text-zinc-400 hover:text-zinc-700 hover:bg-zinc-100 cursor-pointer transition-colors"
            aria-label="Close scanner"
          >
            <svg
              class="h-4 w-4"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="2"
            >
              <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
            </svg>
          </button>
        </div>

        <!-- Scanner Input -->
        <BarcodeScanInput
          ref="scanInputRef"
          :autofocus="true"
          placeholder="Scan barcode or type SKU (e.g. PRD-00001)..."
          @scan="handleBarcodeScan"
        />

        <div
          v-if="scanLoading"
          class="py-6 text-center text-xs text-zinc-500 flex items-center justify-center gap-2"
        >
          <svg
            class="h-4 w-4 animate-spin text-zinc-900"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"
            />
          </svg>
          <span>Looking up SKU / barcode...</span>
        </div>

        <!-- Lookup Error -->
        <p
          v-if="scanError"
          class="rounded-xl bg-rose-50 p-3 text-xs text-rose-700 border border-rose-200/70"
        >
          {{ scanError }}
        </p>

        <!-- Matched Product Card -->
        <div
          v-if="scannedProduct"
          class="rounded-xl border border-zinc-200/80 bg-zinc-50/70 p-4 space-y-3"
        >
          <div class="flex items-start justify-between">
            <div>
              <span
                class="rounded-full bg-zinc-900 text-white px-2 py-0.5 text-[9px] font-semibold uppercase tracking-wider"
              >
                Match Found
              </span>
              <h3 class="mt-1.5 text-sm font-bold text-zinc-900">{{ scannedProduct.name }}</h3>
              <p class="text-xs font-mono text-zinc-500 mt-0.5">
                SKU: {{ scannedProduct.sku }}
                <span v-if="scannedProduct.barcode"> | Barcode: {{ scannedProduct.barcode }}</span>
              </p>
            </div>
            <span
              class="rounded-full px-2.5 py-0.5 text-xs font-semibold"
              :class="
                scannedProduct.stock_total > 0
                  ? 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20'
                  : 'bg-rose-50 text-rose-700 ring-1 ring-rose-600/20'
              "
            >
              {{ scannedProduct.stock_total }} {{ scannedProduct.unit }} in stock
            </span>
          </div>

          <div
            class="grid grid-cols-3 gap-2 rounded-lg bg-white p-2.5 border border-zinc-200/70 text-xs"
          >
            <div>
              <p class="text-[10px] uppercase font-semibold text-zinc-400">Category</p>
              <p class="font-medium text-zinc-800 mt-0.5">
                {{ scannedProduct.category?.name || 'Uncategorised' }}
              </p>
            </div>
            <div>
              <p class="text-[10px] uppercase font-semibold text-zinc-400">Cost Price</p>
              <p class="font-medium text-zinc-800 mt-0.5">{{ money(scannedProduct.cost_price) }}</p>
            </div>
            <div>
              <p class="text-[10px] uppercase font-semibold text-zinc-400">Sell Price</p>
              <p class="font-medium text-zinc-800 mt-0.5">{{ money(scannedProduct.sell_price) }}</p>
            </div>
          </div>

          <div class="flex items-center justify-end gap-2 pt-1">
            <button
              type="button"
              @click="inspectScannedProduct"
              class="rounded-xl bg-zinc-900 px-3.5 py-2 text-xs font-semibold text-white hover:bg-zinc-800 transition-colors shadow-2xs cursor-pointer"
            >
              Inspect Product Full Details &rarr;
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Product Inspection Modal -->
    <ProductDetailModal
      v-if="inspectionProduct"
      :product="inspectionProduct"
      @close="inspectionProduct = null"
    />
  </div>
</template>
