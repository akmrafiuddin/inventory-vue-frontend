<script setup>
import { ref, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { purchaseOrdersApi } from '@/api/purchaseOrders'

const route = useRoute()
const router = useRouter()

const order = ref(null)
const error = ref('')

function money(value) {
  return `$${Number(value ?? 0).toFixed(2)}`
}

function printPage() {
  window.print()
}

onMounted(async () => {
  try {
    const { data } = await purchaseOrdersApi.show(route.params.id)
    order.value = data.data
    document.title = `${order.value.number} — Purchase order`
  } catch {
    error.value = 'Could not load the purchase order.'
  }
})
</script>

<template>
  <div class="min-h-screen bg-zinc-100/70 p-4 sm:p-8 print:bg-white print:p-0">
    <div class="mx-auto max-w-3xl">
      <!-- Toolbar (hidden in print) -->
      <div class="mb-6 flex items-center justify-between print:hidden">
        <button
          class="inline-flex items-center gap-1.5 rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-xs font-semibold text-zinc-700 shadow-2xs hover:bg-zinc-50 transition-colors cursor-pointer"
          @click="router.back()"
        >
          <svg class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M10 19l-7-7m0 0l7-7m-7 7h18"
            />
          </svg>
          Back
        </button>
        <button
          v-if="order"
          class="inline-flex items-center gap-2 rounded-xl bg-zinc-900 px-4 py-2 text-xs font-semibold text-white shadow-2xs hover:bg-zinc-800 transition-colors cursor-pointer"
          @click="printPage"
        >
          <svg class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M17 17h2a2 2 0 002-2v-4a2 2 0 00-2-2H5a2 2 0 00-2 2v4a2 2 0 002 2h2m2 4h6a2 2 0 002-2v-4a2 2 0 00-2-2H9a2 2 0 00-2 2v4a2 2 0 002 2zm8-12V5a2 2 0 00-2-2H9a2 2 0 00-2 2v4h10z"
            />
          </svg>
          Print / Save PDF
        </button>
      </div>

      <div
        v-if="error"
        class="rounded-xl border border-rose-200 bg-rose-50/80 p-4 text-xs sm:text-sm text-rose-800"
      >
        {{ error }}
      </div>

      <!-- Printable Document Sheet -->
      <div
        v-else-if="order"
        class="rounded-2xl border border-zinc-200/80 bg-white p-8 sm:p-10 shadow-sm print:rounded-none print:border-none print:p-0 print:shadow-none"
      >
        <!-- Document Header -->
        <div class="mb-8 flex items-start justify-between border-b border-zinc-100 pb-6">
          <div class="flex items-start gap-3">
            <div
              class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-zinc-900 text-white font-black text-sm"
            >
              PO
            </div>
            <div>
              <h1 class="text-xl font-bold tracking-tight text-zinc-900">Purchase Order</h1>
              <p class="font-mono text-xs font-semibold text-zinc-500 mt-0.5">{{ order.number }}</p>
            </div>
          </div>
          <div class="text-right text-xs space-y-1">
            <div>
              <span
                class="inline-flex items-center rounded-full bg-zinc-100 px-2.5 py-0.5 text-[11px] font-semibold uppercase tracking-wider text-zinc-800 ring-1 ring-zinc-300/60"
              >
                {{ order.status.replace('_', ' ') }}
              </span>
            </div>
            <p class="text-zinc-500">
              Issued:
              <span class="font-medium text-zinc-800">{{
                order.created_at ? order.created_at.substring(0, 10) : '—'
              }}</span>
            </p>
            <p v-if="order.expected_at" class="text-zinc-500">
              Expected:
              <span class="font-medium text-zinc-800">{{
                order.expected_at.substring(0, 10)
              }}</span>
            </p>
            <p v-if="order.received_at" class="text-zinc-500">
              Received:
              <span class="font-medium text-zinc-800">{{
                order.received_at.substring(0, 10)
              }}</span>
            </p>
          </div>
        </div>

        <!-- Supplier Details -->
        <div class="mb-8 text-xs space-y-1">
          <span class="font-bold uppercase tracking-wider text-zinc-400">Vendor / Supplier</span>
          <p class="font-bold text-sm text-zinc-900">{{ order.supplier?.name ?? '—' }}</p>
          <p v-if="order.supplier?.email" class="text-zinc-500">
            {{ order.supplier.email }}
          </p>
          <p v-if="order.supplier?.phone" class="text-zinc-500 font-mono">
            {{ order.supplier.phone }}
          </p>
        </div>

        <!-- Line Items Table -->
        <div class="overflow-x-auto rounded-xl border border-zinc-200/80 mb-6">
          <table class="w-full text-left text-xs">
            <thead
              class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
            >
              <tr>
                <th scope="col" class="px-4 py-3">Product Description</th>
                <th scope="col" class="px-4 py-3">SKU</th>
                <th scope="col" class="px-4 py-3 text-right">Ordered</th>
                <th scope="col" class="px-4 py-3 text-right">Received</th>
                <th scope="col" class="px-4 py-3 text-right">Unit Cost</th>
                <th scope="col" class="px-4 py-3 text-right">Line Total</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-zinc-100">
              <tr v-for="item in order.items" :key="item.id">
                <td class="px-4 py-3 font-medium text-zinc-900">{{ item.product_name }}</td>
                <td class="px-4 py-3 font-mono text-[11px] text-zinc-500">
                  {{ item.product_sku }}
                </td>
                <td class="px-4 py-3 text-right font-medium text-zinc-700">{{ item.quantity }}</td>
                <td class="px-4 py-3 text-right text-zinc-600">{{ item.quantity_received }}</td>
                <td class="px-4 py-3 text-right text-zinc-600">{{ money(item.unit_cost) }}</td>
                <td class="px-4 py-3 text-right font-bold text-zinc-900">
                  {{ money(item.line_total) }}
                </td>
              </tr>
            </tbody>
            <tfoot class="border-t border-zinc-200/80 bg-zinc-50/40 text-xs">
              <tr>
                <td colspan="5" class="px-4 py-3 text-right text-sm font-bold text-zinc-900">
                  Total Procurement Cost
                </td>
                <td class="px-4 py-3 text-right text-base font-extrabold text-zinc-900">
                  {{ money(order.total) }}
                </td>
              </tr>
            </tfoot>
          </table>
        </div>

        <!-- Notes -->
        <div
          v-if="order.notes"
          class="rounded-xl border border-zinc-200/80 bg-zinc-50/50 p-4 text-xs"
        >
          <p class="font-bold uppercase tracking-wider text-zinc-400 mb-1">
            Notes & Procurement Terms
          </p>
          <p class="whitespace-pre-line text-zinc-700">{{ order.notes }}</p>
        </div>
      </div>
    </div>
  </div>
</template>
