<script setup>
import { ref, onMounted } from 'vue'
import { stocksApi } from '@/api/stocks'

const report = ref(null)
const error = ref('')

function money(value) {
  return `$${Number(value ?? 0).toFixed(2)}`
}

onMounted(async () => {
  try {
    const { data } = await stocksApi.valuation()
    report.value = data
  } catch {
    error.value = 'Could not load the valuation report.'
  }
})
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="flex flex-col gap-1 sm:flex-row sm:items-center sm:justify-between">
      <div>
        <h1 class="text-xl font-bold tracking-tight text-zinc-900 sm:text-2xl">
          Inventory Valuation
        </h1>
        <p class="mt-0.5 text-xs text-zinc-500 sm:text-sm">
          Real-time asset value, retail potential, and margin breakdowns
        </p>
      </div>
    </div>

    <!-- Error Alert -->
    <div
      v-if="error"
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
        <span>{{ error }}</span>
      </div>
    </div>

    <template v-if="report">
      <!-- Valuation KPI Cards -->
      <div class="grid grid-cols-2 gap-4 lg:grid-cols-4">
        <div class="rounded-2xl border border-zinc-200/80 bg-white p-5 shadow-2xs">
          <span class="text-xs font-semibold uppercase tracking-wider text-zinc-400"
            >Units in Stock</span
          >
          <div class="mt-2 text-2xl sm:text-3xl font-extrabold tracking-tight text-zinc-900">
            {{ Number(report.total_units).toLocaleString() }}
          </div>
          <span class="mt-1 block text-xs text-zinc-500">Across all active facilities</span>
        </div>

        <div class="rounded-2xl border border-zinc-200/80 bg-white p-5 shadow-2xs">
          <span class="text-xs font-semibold uppercase tracking-wider text-zinc-400"
            >Value at Cost</span
          >
          <div class="mt-2 text-2xl sm:text-3xl font-extrabold tracking-tight text-zinc-900">
            {{ money(report.cost_value) }}
          </div>
          <span class="mt-1 block text-xs text-zinc-500">Current procurement baseline</span>
        </div>

        <div class="rounded-2xl border border-zinc-200/80 bg-white p-5 shadow-2xs">
          <span class="text-xs font-semibold uppercase tracking-wider text-zinc-400"
            >Value at Retail</span
          >
          <div class="mt-2 text-2xl sm:text-3xl font-extrabold tracking-tight text-zinc-900">
            {{ money(report.retail_value) }}
          </div>
          <span class="mt-1 block text-xs text-zinc-500">Gross market listing value</span>
        </div>

        <div class="rounded-2xl border border-zinc-200/80 bg-white p-5 shadow-2xs">
          <span class="text-xs font-semibold uppercase tracking-wider text-zinc-400"
            >Potential Margin</span
          >
          <div class="mt-2 text-2xl sm:text-3xl font-extrabold tracking-tight text-emerald-600">
            {{ money(report.potential_margin) }}
          </div>
          <span class="mt-1 block text-xs text-zinc-500">Expected gross profit margin</span>
        </div>
      </div>

      <!-- Breakdown Tables -->
      <div class="grid gap-6 lg:grid-cols-2">
        <!-- By Warehouse -->
        <div class="overflow-hidden rounded-2xl border border-zinc-200/80 bg-white shadow-2xs">
          <div class="border-b border-zinc-100 px-5 py-3.5">
            <h3 class="text-sm font-bold text-zinc-900">Stock Valuation by Warehouse</h3>
            <p class="text-xs text-zinc-500">Physical inventory allocation by storage hub</p>
          </div>
          <div class="overflow-x-auto">
            <table class="w-full text-left text-xs sm:text-sm">
              <thead
                class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
              >
                <tr>
                  <th scope="col" class="px-5 py-3">Warehouse</th>
                  <th scope="col" class="px-5 py-3 text-right">Units</th>
                  <th scope="col" class="px-5 py-3 text-right">Cost</th>
                  <th scope="col" class="px-5 py-3 text-right">Retail</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!report.by_warehouse.length">
                  <td colspan="4" class="px-5 py-8 text-center text-xs text-zinc-400">
                    No stock records found.
                  </td>
                </tr>
                <tr
                  v-for="row in report.by_warehouse"
                  :key="row.warehouse_id"
                  class="hover:bg-zinc-50/60 transition-colors"
                >
                  <td class="px-5 py-3 font-semibold text-zinc-900">
                    {{ row.warehouse_name ?? '—' }}
                  </td>
                  <td class="px-5 py-3 text-right font-medium text-zinc-700">
                    {{ Number(row.units).toLocaleString() }}
                  </td>
                  <td class="px-5 py-3 text-right text-zinc-600">{{ money(row.cost_value) }}</td>
                  <td class="px-5 py-3 text-right font-semibold text-zinc-900">
                    {{ money(row.retail_value) }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- By Category -->
        <div class="overflow-hidden rounded-2xl border border-zinc-200/80 bg-white shadow-2xs">
          <div class="border-b border-zinc-100 px-5 py-3.5">
            <h3 class="text-sm font-bold text-zinc-900">Stock Valuation by Category</h3>
            <p class="text-xs text-zinc-500">Valuation distribution across product catalogs</p>
          </div>
          <div class="overflow-x-auto">
            <table class="w-full text-left text-xs sm:text-sm">
              <thead
                class="border-b border-zinc-200/60 bg-zinc-50/75 text-[11px] font-semibold uppercase tracking-wider text-zinc-400"
              >
                <tr>
                  <th scope="col" class="px-5 py-3">Category</th>
                  <th scope="col" class="px-5 py-3 text-right">Units</th>
                  <th scope="col" class="px-5 py-3 text-right">Cost</th>
                  <th scope="col" class="px-5 py-3 text-right">Retail</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-zinc-100">
                <tr v-if="!report.by_category.length">
                  <td colspan="4" class="px-5 py-8 text-center text-xs text-zinc-400">
                    No stock records found.
                  </td>
                </tr>
                <tr
                  v-for="row in report.by_category"
                  :key="row.category_id ?? 'none'"
                  class="hover:bg-zinc-50/60 transition-colors"
                >
                  <td class="px-5 py-3 font-semibold text-zinc-900">{{ row.category_name }}</td>
                  <td class="px-5 py-3 text-right font-medium text-zinc-700">
                    {{ Number(row.units).toLocaleString() }}
                  </td>
                  <td class="px-5 py-3 text-right text-zinc-600">{{ money(row.cost_value) }}</td>
                  <td class="px-5 py-3 text-right font-semibold text-zinc-900">
                    {{ money(row.retail_value) }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </template>

    <div v-else-if="!error" class="py-16 text-center text-xs text-zinc-400">
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
        Generating valuation report...
      </div>
    </div>
  </div>
</template>
