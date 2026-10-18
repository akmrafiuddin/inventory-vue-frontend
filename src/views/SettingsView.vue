<script setup>
import { ref, onMounted } from 'vue'
import { profileApi } from '@/api/profile'
import { useAuthStore } from '@/stores/auth'
import { useToastStore } from '@/stores/toasts'

const auth = useAuthStore()
const toast = useToastStore()

const preferences = ref({
  notify_low_stock: false,
  notify_purchase_orders: false,
})
const loading = ref(true)
const saving = ref(false)

function syncFromUser() {
  preferences.value = {
    notify_low_stock: !!auth.user?.notify_low_stock,
    notify_purchase_orders: !!auth.user?.notify_purchase_orders,
  }
}

onMounted(async () => {
  try {
    if (!auth.user) await auth.fetchUser()
    syncFromUser()
  } catch {
    toast.error('Could not load your profile.')
  } finally {
    loading.value = false
  }
})

async function save() {
  saving.value = true
  try {
    const { data } = await profileApi.updatePreferences(preferences.value)
    auth.applyUser(data)
    syncFromUser()
    toast.success('Preferences saved.')
  } catch {
    toast.error('Could not save your preferences.')
    syncFromUser()
  } finally {
    saving.value = false
  }
}
</script>

<template>
  <div class="max-w-2xl space-y-6">
    <!-- Header -->
    <div>
      <h1 class="text-xl font-bold tracking-tight text-zinc-900 sm:text-2xl">Account & Settings</h1>
      <p class="mt-0.5 text-xs text-zinc-500 sm:text-sm">
        Manage profile credentials, account authority, and automated alert delivery
      </p>
    </div>

    <!-- Account Details Card -->
    <div class="overflow-hidden rounded-2xl border border-zinc-200/80 bg-white shadow-2xs">
      <div class="border-b border-zinc-100 px-6 py-4">
        <h2 class="text-xs font-bold uppercase tracking-wider text-zinc-400">Profile Identity</h2>
      </div>
      <div class="p-6">
        <div class="flex items-center gap-4">
          <div
            class="flex h-12 w-12 items-center justify-center rounded-2xl bg-zinc-900 text-base font-bold text-white shadow-2xs"
          >
            {{ (auth.user?.name || 'U').charAt(0).toUpperCase() }}
          </div>
          <div class="flex-1">
            <h3 class="text-base font-bold text-zinc-900">{{ auth.user?.name ?? '—' }}</h3>
            <p class="text-xs text-zinc-500">{{ auth.user?.email ?? '—' }}</p>
          </div>
          <span
            class="inline-flex items-center rounded-full bg-zinc-100 px-3 py-1 text-xs font-semibold capitalize text-zinc-800 ring-1 ring-zinc-300/60"
          >
            {{ auth.user?.role ?? 'User' }}
          </span>
        </div>

        <div
          class="mt-6 grid grid-cols-1 sm:grid-cols-3 gap-4 border-t border-zinc-100 pt-5 text-xs"
        >
          <div>
            <span class="text-zinc-400">Full Name</span>
            <div class="mt-1 font-semibold text-zinc-800">{{ auth.user?.name ?? '—' }}</div>
          </div>
          <div>
            <span class="text-zinc-400">Signed-in Email</span>
            <div class="mt-1 font-semibold text-zinc-800">{{ auth.user?.email ?? '—' }}</div>
          </div>
          <div>
            <span class="text-zinc-400">System Role</span>
            <div class="mt-1 font-semibold capitalize text-zinc-800">
              {{ auth.user?.role ?? '—' }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Notification Preferences Card -->
    <div class="overflow-hidden rounded-2xl border border-zinc-200/80 bg-white shadow-2xs">
      <div class="border-b border-zinc-100 px-6 py-4">
        <h2 class="text-xs font-bold uppercase tracking-wider text-zinc-400">
          Email Notifications
        </h2>
        <p class="text-xs text-zinc-500 mt-0.5">
          Choose which operational events send an alert to your mailbox
        </p>
      </div>

      <div class="p-6 space-y-4">
        <label
          class="flex items-start gap-3 rounded-xl border border-zinc-200/80 bg-zinc-50/40 p-4 transition-colors hover:bg-zinc-50 cursor-pointer"
        >
          <input
            v-model="preferences.notify_low_stock"
            type="checkbox"
            class="mt-0.5 h-4 w-4 rounded border-zinc-300 text-zinc-900 focus:ring-zinc-900 cursor-pointer"
          />
          <div class="flex-1">
            <span class="block text-xs sm:text-sm font-semibold text-zinc-900"
              >Low stock alerts</span
            >
            <span class="mt-0.5 block text-xs text-zinc-500">
              Receive an email immediately whenever an inventory product drops below its safety
              reorder threshold.
            </span>
          </div>
        </label>

        <label
          class="flex items-start gap-3 rounded-xl border border-zinc-200/80 bg-zinc-50/40 p-4 transition-colors hover:bg-zinc-50 cursor-pointer"
        >
          <input
            v-model="preferences.notify_purchase_orders"
            type="checkbox"
            class="mt-0.5 h-4 w-4 rounded border-zinc-300 text-zinc-900 focus:ring-zinc-900 cursor-pointer"
          />
          <div class="flex-1">
            <span class="block text-xs sm:text-sm font-semibold text-zinc-900"
              >Purchase order received</span
            >
            <span class="mt-0.5 block text-xs text-zinc-500">
              Receive a confirmation email when inbound supplier shipments are fully received and
              verified.
            </span>
          </div>
        </label>

        <div class="pt-2">
          <button
            :disabled="loading || saving"
            class="inline-flex items-center gap-2 rounded-xl bg-zinc-900 px-4 py-2 text-xs font-semibold text-white shadow-2xs hover:bg-zinc-800 disabled:opacity-50 transition-colors cursor-pointer"
            @click="save"
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
            {{ saving ? 'Saving...' : 'Save preferences' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
