<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const auth = useAuthStore()
const router = useRouter()

const email = ref('')
const password = ref('')
const message = ref('')
const errors = ref({})
const loading = ref(false)

async function submit() {
  loading.value = true
  message.value = ''
  errors.value = {}

  try {
    await auth.login(email.value, password.value)
    router.push({ name: 'dashboard' })
  } catch (e) {
    const status = e.response?.status
    if (status === 422) {
      message.value = e.response.data.message
      errors.value = e.response.data.errors ?? {}
    } else if (status === 429) {
      message.value = 'Too many attempts. Please wait a minute.'
    } else {
      message.value = 'Something went wrong. Is the server running?'
    }
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="flex min-h-screen items-center justify-center bg-zinc-50 px-4 py-12">
    <div class="w-full max-w-sm">
      <div class="text-center mb-8">
        <div
          class="mx-auto mb-3 flex h-11 w-11 items-center justify-center rounded-xl bg-zinc-900 text-white shadow-2xs"
        >
          <svg
            class="h-6 w-6"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="1.8"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path
              d="M21 8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16Z"
            />
            <path d="m3.3 7 8.7 5 8.7-5" />
            <path d="M12 22V12" />
          </svg>
        </div>
        <h1 class="text-xl font-bold tracking-tight text-zinc-900">Sign in to Inventory</h1>
        <p class="mt-1 text-xs text-zinc-500">Enter your credentials to access the workspace</p>
      </div>

      <form
        @submit.prevent="submit"
        class="space-y-4 rounded-2xl border border-zinc-200/80 bg-white p-6 sm:p-7 shadow-xs"
      >
        <div
          v-if="message"
          role="alert"
          class="rounded-xl border border-rose-200/70 bg-rose-50 px-3.5 py-2.5 text-xs text-rose-700"
        >
          {{ message }}
        </div>

        <div>
          <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Email address</label>
          <input
            v-model="email"
            type="email"
            placeholder="you@company.com"
            required
            class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs transition-colors focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          />
          <p v-if="errors.email" class="mt-1 text-xs text-rose-600">{{ errors.email[0] }}</p>
        </div>

        <div>
          <label class="mb-1.5 block text-xs font-semibold text-zinc-700">Password</label>
          <input
            v-model="password"
            type="password"
            placeholder="••••••••"
            required
            class="w-full rounded-xl border border-zinc-200 bg-white px-3.5 py-2 text-sm text-zinc-900 placeholder:text-zinc-400 shadow-2xs transition-colors focus:border-zinc-900 focus:outline-none focus:ring-1 focus:ring-zinc-900"
          />
          <p v-if="errors.password" class="mt-1 text-xs text-rose-600">{{ errors.password[0] }}</p>
        </div>

        <button
          :disabled="loading"
          class="w-full rounded-xl bg-zinc-900 py-2.5 text-sm font-medium text-white shadow-2xs hover:bg-zinc-800 disabled:opacity-50 transition-all active:scale-[0.99] cursor-pointer"
        >
          {{ loading ? 'Signing in...' : 'Sign in' }}
        </button>
      </form>
    </div>
  </div>
</template>
