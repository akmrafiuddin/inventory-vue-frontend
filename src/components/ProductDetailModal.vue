<script setup>
import { ref, watch } from 'vue'
import { stocksApi } from '@/api/stocks'
import { stockMovementsApi } from '@/api/stockMovements'
import { productSuppliersApi } from '@/api/productSuppliers'

const props = defineProps({
  product: { type: Object, default: null },
})
const emit = defineEmits(['close'])

// image URLs are served by the backend origin, not the frontend one
const API_ORIGIN = (import.meta.env.VITE_API_URL || '').replace(/\/api\/?$/, '')

const loading = ref(false)
const error = ref('')
const detail = ref(null)
const movements = ref([])
const suppliers = ref([])

const movementStyles = {
  in: 'text-green-600',
  out: 'text-red-600',
  adjustment: 'text-amber-600',
  transfer_in: 'text-blue-600',
  transfer_out: 'text-blue-600',
}

function imageUrl(path) {
  return path ? `${API_ORIGIN}/storage/${path}` : null
}

function money(value) {
  return `$${Number(value ?? 0).toFixed(2)}`
}

function stripSku(sku) {
  return sku ? sku.replace(/::deleted::\d+$/, '') : sku
}

watch(
  () => props.product,
  async (product) => {
    if (!product) return

    loading.value = true
    error.value = ''
    detail.value = null
    movements.value = []
    suppliers.value = []

    try {
      const [stock, movementList, supplierList] = await Promise.all([
        stocksApi.forProduct(product.id),
        stockMovementsApi.list({ product_id: product.id, per_page: 10 }),
        productSuppliersApi.list(product.id),
      ])
      detail.value = stock.data
      movements.value = movementList.data.data
      suppliers.value = supplierList.data.data
    } catch {
      error.value = 'Could not load product details.'
    } finally {
      loading.value = false
    }
  },
  { immediate: true },
)
</script>

<template>
  <div
    role="dialog"
    aria-modal="true"
    aria-label="Product details"
    class="fixed inset-0 z-50 flex items-start justify-center overflow-y-auto bg-zinc-950/40 backdrop-blur-xs p-4"
    @click.self="emit('close')"
  >
    <div
      class="my-8 w-full max-w-3xl space-y-6 rounded-2xl bg-white p-6 sm:p-7 shadow-xl border border-zinc-200/80"
    >
      <div class="flex items-start justify-between border-b border-zinc-100 pb-5">
        <div class="flex items-center gap-4">
          <img
            v-if="imageUrl(product?.image)"
            :src="imageUrl(product.image)"
            class="h-14 w-14 rounded-xl object-cover border border-zinc-200/80"
            alt=""
          />
          <div
            v-else
            class="flex h-14 w-14 items-center justify-center rounded-xl bg-zinc-100 border border-zinc-200/80 text-zinc-400"
          >
            <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="1.5"
                d="M20 7l-8-4-8 4m16 0l-8 4m8-4v10l-8 4m0-10L4 7m8 4v10M4 7v10l8 4"
              />
            </svg>
          </div>
          <div>
            <h3 class="text-lg font-semibold text-zinc-900 tracking-tight">{{ product?.name }}</h3>
            <p class="font-mono text-xs text-zinc-500 mt-0.5">{{ stripSku(product?.sku) }}</p>
          </div>
        </div>
        <button
          type="button"
          class="rounded-lg p-1.5 text-zinc-400 hover:bg-zinc-100 hover:text-zinc-700 transition-colors"
          @click="emit('close')"
        >
          <svg
            class="h-4 w-4"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
          >
            <line x1="18" y1="6" x2="6" y2="18"></line>
            <line x1="6" y1="6" x2="18" y2="18"></line>
          </svg>
        </button>
      </div>

      <p
        v-if="error"
        class="rounded-xl bg-rose-50 border border-rose-200/60 px-4 py-3 text-sm text-rose-700"
      >
        {{ error }}
      </p>
      <div v-else-if="loading" class="py-8 text-center text-sm text-zinc-500">
        Loading details...
      </div>

      <template v-else-if="detail">
        <div class="grid grid-cols-2 gap-3 sm:grid-cols-4">
          <div class="rounded-xl border border-zinc-200/70 bg-zinc-50/60 p-3.5">
            <p class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">On hand</p>
            <p class="text-xl font-bold tracking-tight text-zinc-900 mt-1">{{ detail.total }}</p>
          </div>
          <div class="rounded-xl border border-zinc-200/70 bg-zinc-50/60 p-3.5">
            <p class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">
              Reorder level
            </p>
            <p class="text-xl font-bold tracking-tight text-zinc-900 mt-1">
              {{ detail.reorder_level }}
            </p>
          </div>
          <div class="rounded-xl border border-zinc-200/70 bg-zinc-50/60 p-3.5">
            <p class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">
              Cost price
            </p>
            <p class="text-xl font-bold tracking-tight text-zinc-900 mt-1">
              {{ money(detail.product?.cost_price) }}
            </p>
          </div>
          <div class="rounded-xl border border-zinc-200/70 bg-zinc-50/60 p-3.5">
            <p class="text-[11px] font-semibold uppercase tracking-wider text-zinc-500">
              Sell price
            </p>
            <p class="text-xl font-bold tracking-tight text-zinc-900 mt-1">
              {{ money(detail.product?.sell_price) }}
            </p>
          </div>
        </div>

        <div class="space-y-2">
          <h4 class="text-xs font-semibold uppercase tracking-wider text-zinc-500">
            Stock by warehouse
          </h4>
          <div class="overflow-hidden rounded-xl border border-zinc-200/80">
            <table class="w-full text-left text-sm">
              <thead
                class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-500"
              >
                <tr>
                  <th scope="col" class="px-4 py-2.5">Warehouse</th>
                  <th scope="col" class="px-4 py-2.5 text-right">Quantity</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!detail.stocks?.length">
                  <td colspan="2" class="px-4 py-6 text-center text-sm text-zinc-500">
                    No stock recorded.
                  </td>
                </tr>
                <tr
                  v-for="s in detail.stocks"
                  :key="s.id"
                  class="hover:bg-zinc-50/50 transition-colors"
                >
                  <td class="px-4 py-2.5 font-medium text-zinc-800">
                    {{ s.warehouse?.name ?? '-' }}
                  </td>
                  <td class="px-4 py-2.5 text-right font-semibold text-zinc-900">
                    {{ s.quantity }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <div class="space-y-2">
          <h4 class="text-xs font-semibold uppercase tracking-wider text-zinc-500">Suppliers</h4>
          <div class="overflow-hidden rounded-xl border border-zinc-200/80">
            <table class="w-full text-left text-sm">
              <thead
                class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-500"
              >
                <tr>
                  <th scope="col" class="px-4 py-2.5">Supplier</th>
                  <th scope="col" class="px-4 py-2.5">Supplier SKU</th>
                  <th scope="col" class="px-4 py-2.5 text-right">Unit cost</th>
                  <th scope="col" class="px-4 py-2.5 text-right">Lead time</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!suppliers.length">
                  <td colspan="4" class="px-4 py-6 text-center text-sm text-zinc-500">
                    Not linked to any supplier.
                  </td>
                </tr>
                <tr
                  v-for="s in suppliers"
                  :key="s.id"
                  class="hover:bg-zinc-50/50 transition-colors"
                >
                  <td class="px-4 py-2.5 font-medium text-zinc-800">
                    {{ s.name }}
                    <span
                      v-if="s.is_preferred"
                      class="ml-1.5 inline-flex items-center rounded-full bg-emerald-50 px-2 py-0.5 text-[10px] font-semibold text-emerald-700 ring-1 ring-emerald-600/20"
                      >Preferred</span
                    >
                  </td>
                  <td class="px-4 py-2.5 font-mono text-xs text-zinc-500">
                    {{ s.supplier_sku ?? '-' }}
                  </td>
                  <td class="px-4 py-2.5 text-right font-medium text-zinc-700">
                    {{ s.unit_cost == null ? '-' : money(s.unit_cost) }}
                  </td>
                  <td class="px-4 py-2.5 text-right text-zinc-600">
                    {{ s.lead_time_days == null ? '-' : `${s.lead_time_days} days` }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <div class="space-y-2">
          <h4 class="text-xs font-semibold uppercase tracking-wider text-zinc-500">
            Recent movements
          </h4>
          <div class="overflow-hidden rounded-xl border border-zinc-200/80">
            <table class="w-full text-left text-sm">
              <thead
                class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-500"
              >
                <tr>
                  <th scope="col" class="px-4 py-2.5">Date</th>
                  <th scope="col" class="px-4 py-2.5">Type</th>
                  <th scope="col" class="px-4 py-2.5">Warehouse</th>
                  <th scope="col" class="px-4 py-2.5 text-right">Change</th>
                  <th scope="col" class="px-4 py-2.5 text-right">Balance</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!movements.length">
                  <td colspan="5" class="px-4 py-6 text-center text-sm text-zinc-500">
                    No movements yet.
                  </td>
                </tr>
                <tr
                  v-for="m in movements"
                  :key="m.id"
                  class="hover:bg-zinc-50/50 transition-colors"
                >
                  <td class="px-4 py-2.5 text-xs text-zinc-500 font-mono">
                    {{ m.created_at ? m.created_at.substring(0, 10) : '-' }}
                  </td>
                  <td
                    class="px-4 py-2.5 capitalize font-medium text-xs"
                    :class="movementStyles[m.type]"
                  >
                    {{ m.type.replace('_', ' ') }}
                  </td>
                  <td class="px-4 py-2.5 text-zinc-600">{{ m.warehouse_name ?? '-' }}</td>
                  <td
                    class="px-4 py-2.5 text-right font-semibold text-xs"
                    :class="movementStyles[m.type]"
                  >
                    {{ m.quantity > 0 ? '+' : '' }}{{ m.quantity }}
                  </td>
                  <td class="px-4 py-2.5 text-right font-mono text-xs font-medium text-zinc-700">
                    {{ m.balance_after }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </template>
    </div>
  </div>
</template>
