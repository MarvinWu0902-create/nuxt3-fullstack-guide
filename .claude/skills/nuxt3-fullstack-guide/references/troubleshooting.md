# 常見問題與疑難排解

## 目錄

1. [Hydration Mismatch（水合不匹配）](#hydration-mismatch)
2. [記憶體洩漏](#記憶體洩漏)
3. [循環依賴](#循環依賴)
4. [SSR 常見陷阱](#ssr-常見陷阱)
5. [TypeScript 型別錯誤](#typescript-型別錯誤)
6. [Composable 常見錯誤](#composable-常見錯誤)
7. [Tailwind CSS 問題](#tailwind-css-問題)
8. [Vitest 測試問題](#vitest-測試問題)
9. [建置與部署問題](#建置與部署問題)
10. [效能問題診斷](#效能問題診斷)

---

## Hydration Mismatch

### 症狀

控制台出現警告：
```
[Vue warn]: Hydration node mismatch
[Vue warn]: Hydration text content mismatch
```

### 原因 1：瀏覽器 API 在 SSR 中不存在

```typescript
// 問題
const width = ref(window.innerWidth)  // SSR 時 window 不存在

// 解法 1：onMounted
const width = ref(0)
onMounted(() => {
  width.value = window.innerWidth
})

// 解法 2：使用 <ClientOnly> 包裝依賴瀏覽器 API 的元件
// 注意：import.meta.client 條件式初始值仍可能造成 hydration mismatch
// 因為 SSR 渲染的 0 與 CSR 的實際值不同

// 解法 3：ClientOnly 元件
```

```vue
<template>
  <ClientOnly>
    <BrowserOnlyComponent />
    <template #fallback>
      <div class="skeleton h-32" />
    </template>
  </ClientOnly>
</template>
```

### 原因 2：時間/隨機值不一致

```typescript
// 問題：SSR 和 CSR 產生不同的值
const id = ref(Math.random().toString(36))  // 每次渲染不同
const now = ref(new Date().toLocaleString()) // 伺服器和客戶端時間不同

// 解法：使用 useState 確保水合一致
const id = useState('unique-id', () => Math.random().toString(36))
const now = useState('current-time', () => new Date().toISOString())
```

### 原因 3：v-if 條件在 SSR 和 CSR 不同

```vue
<!-- 問題 -->
<div v-if="isMobile">手機版</div>
<div v-else>桌面版</div>

<!-- 解法：使用 ClientOnly 或延遲判斷 -->
<ClientOnly>
  <MobileView v-if="isMobile" />
  <DesktopView v-else />
  <template #fallback>
    <DesktopView /> <!-- SSR 預設渲染桌面版 -->
  </template>
</ClientOnly>
```

### 原因 4：第三方套件操作 DOM

```typescript
// 問題：第三方套件在 SSR 時嘗試存取 DOM
import SomeLibrary from 'some-dom-library'

// 解法 1：動態 import
let SomeLibrary: any
if (import.meta.client) {
  SomeLibrary = (await import('some-dom-library')).default
}

// 解法 2：在 plugin 中使用 .client 後綴
// plugins/some-library.client.ts
export default defineNuxtPlugin(() => {
  // 只在客戶端執行
})
```

### 原因 5：HTML 結構無效

```vue
<!-- 問題：<p> 裡面不能有 <div> -->
<p><div>invalid</div></p>

<!-- 解法：使用正確的 HTML 結構 -->
<div><div>valid</div></div>
```

---

## 記憶體洩漏

### 原因 1：未清理的事件監聽器

```typescript
// 問題
onMounted(() => {
  window.addEventListener('resize', handleResize)
  // 忘記清理！
})

// 解法
onMounted(() => {
  window.addEventListener('resize', handleResize)
})
onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})

// 更好的解法：使用 VueUse
import { useEventListener } from '@vueuse/core'
useEventListener('resize', handleResize) // 自動清理
```

### 原因 2：未清理的計時器

```typescript
// 問題
onMounted(() => {
  setInterval(() => pollData(), 5000) // 永遠不會清理
})

// 解法
let intervalId: ReturnType<typeof setInterval> | undefined
onMounted(() => {
  intervalId = setInterval(() => pollData(), 5000)
})
onUnmounted(() => {
  if (intervalId) clearInterval(intervalId)
})
```

### 原因 3：SSR 中的全域狀態污染

```typescript
// 問題：在模組頂層定義響應式狀態（所有請求共用同一個引用）
const sharedState = ref([]) // 全域共用！請求之間會互相影響

// 解法：使用 useState 或 Pinia（每個請求獨立狀態）
export function useItems() {
  const items = useState<Item[]>('items', () => [])
  return { items }
}
```

### 原因 4：過大的快取

```typescript
// 問題：不斷增長的 Map 快取
const cache = new Map()
function getData(key: string) {
  if (!cache.has(key)) {
    cache.set(key, fetchData(key)) // 永遠不清理
  }
  return cache.get(key)
}

// 解法：使用 LRU 快取限制大小
import { LRUCache } from 'lru-cache'
const cache = new LRUCache({ max: 500, ttl: 1000 * 60 * 5 })
```

---

## 循環依賴

### 症狀

```
Warning: Circular dependency detected
```

### Store 之間的循環引用

```typescript
// 問題：A 引用 B，B 引用 A
// stores/auth.ts
import { useCartStore } from './cart'  // 頂層 import
// stores/cart.ts
import { useAuthStore } from './auth'  // 循環！

// 解法：在函式內部使用
// stores/cart.ts
export const useCartStore = defineStore('cart', () => {
  async function checkout() {
    const authStore = useAuthStore()  // 延遲引用
    if (!authStore.isLoggedIn) return
  }
  return { checkout }
})
```

### Composable 循環依賴

```typescript
// 問題：composable 相互引用
// composables/useA.ts
import { useB } from './useB'  // useB 也引用 useA

// 解法 1：合併成單一 composable
// 解法 2：提取共用邏輯到第三個 composable
// 解法 3：使用 provide/inject 解耦
```

---

## SSR 常見陷阱

### 陷阱 1：在 setup 外使用 composable

```typescript
// 問題：在非同步回呼中呼叫 composable
setTimeout(() => {
  const route = useRoute()  // 錯誤！已脫離 setup context
}, 1000)

// 解法：在 setup 中先取得引用
const route = useRoute()
setTimeout(() => {
  console.log(route.path)  // 使用已存在的引用
}, 1000)
```

### 陷阱 2：Pinia 在 SSR 中的序列化

```typescript
// 問題：store 中包含不可序列化的值
const store = defineStore('app', () => {
  const socket = ref(new WebSocket('ws://...'))  // 不可序列化！
  return { socket }
})

// 解法：不可序列化的值放在 store 外部，或用 skipHydrate
import { skipHydrate } from 'pinia'

const store = defineStore('app', () => {
  const socket = ref<WebSocket | null>(null)

  // 使用 skipHydrate 排除水合
  return { socket: skipHydrate(socket) }
})
```

### 陷阱 3：Cookie 在 API route 中的存取

```typescript
// 問題：在 server/api 中使用 useCookie（它是 Vue composable）
// server/api/me.get.ts
const token = useCookie('token')  // 錯誤！

// 解法：使用 H3 的 getCookie
export default defineEventHandler((event) => {
  const token = getCookie(event, 'auth-token')
  // ...
})
```

### 陷阱 4：useAsyncData 的 key 衝突

```typescript
// 問題：同一頁面中兩個 useAsyncData 使用相同 key
const { data: a } = await useAsyncData('data', () => $fetch('/api/a'))
const { data: b } = await useAsyncData('data', () => $fetch('/api/b'))
// b 會覆蓋 a！

// 解法：每個 useAsyncData 使用唯一 key
const { data: a } = await useAsyncData('users', () => $fetch('/api/a'))
const { data: b } = await useAsyncData('posts', () => $fetch('/api/b'))
```

---

## TypeScript 型別錯誤

### 錯誤 1：Auto-import 型別遺失

```
Cannot find name 'ref'. Did you mean 'Ref'?
Cannot find name 'useFetch'.
```

```bash
# 解法：重新產生型別
npx nuxi prepare
# 並確保 tsconfig.json 繼承 .nuxt/tsconfig.json
```

### 錯誤 2：definePageMeta 自訂屬性型別

```typescript
// 問題：definePageMeta 不認識自訂屬性
definePageMeta({
  requiresAuth: true  // 型別錯誤
})

// 解法：擴展 PageMeta 型別
// types/index.d.ts
declare module '#app' {
  interface PageMeta {
    requiresAuth?: boolean
    requiredRole?: string
    guestOnly?: boolean
  }
}
export {}
```

### 錯誤 3：元件 ref 型別

```vue
<script setup lang="ts">
// 問題：ref 取到的元件缺少型別
const formRef = ref()  // 型別為 any

// 解法：使用元件實例型別
import type BaseForm from '~/components/base/BaseForm.vue'
const formRef = ref<InstanceType<typeof BaseForm>>()

// 呼叫元件暴露的方法
formRef.value?.validate()
</script>
```

### 錯誤 4：$fetch 回應型別

```typescript
// 問題：$fetch 推斷不出型別
const data = await $fetch('/api/users')  // unknown

// 解法 1：泛型參數
const data = await $fetch<User[]>('/api/users')

// 解法 2：使用 Nitro 的路由型別推斷（確保 server/api 正確 export）
// 當 API route 有正確的 return type 時，useFetch 會自動推斷
const { data } = await useFetch('/api/users')
// data 會自動推斷為 API 回傳的型別
```

### 錯誤 5：runtimeConfig 型別

```typescript
// types/index.d.ts
declare module 'nuxt/schema' {
  interface RuntimeConfig {
    jwtSecret: string
    dbUrl: string
  }
  interface PublicRuntimeConfig {
    apiBase: string
    appName: string
  }
}
export {}
```

---

## Composable 常見錯誤

### 錯誤 1：在非同步函式中使用 composable

```typescript
// 問題
export async function useUserData(id: string) {
  const response = await fetch(`/api/users/${id}`)
  const route = useRoute()  // 可能在 await 之後失去 context
  return response
}

// 解法：在 await 之前呼叫所有 composable
export async function useUserData(id: string) {
  const route = useRoute()  // 先呼叫
  const response = await fetch(`/api/users/${id}`)
  return response
}
```

### 錯誤 2：Composable 返回未包裝的值

```typescript
// 問題：返回 .value 後失去響應性
export function useCounter() {
  const count = ref(0)
  return {
    count: count.value,  // 只是數字，非響應式！
    increment: () => count.value++
  }
}

// 解法：返回 ref 本身
export function useCounter() {
  const count = ref(0)
  return {
    count,  // 保持響應性
    increment: () => count.value++
  }
}
```

### 錯誤 3：Composable 中的命名空間衝突

```typescript
// 問題：多個 composable export 同名函式
// composables/useA.ts
export function useData() { ... }
// composables/useB.ts
export function useData() { ... }  // 衝突！

// 解法：使用更具體的名稱
// composables/useUserData.ts
export function useUserData() { ... }
// composables/useProductData.ts
export function useProductData() { ... }
```

---

## Tailwind CSS 問題

### 問題 1：動態 class 被 purge 掉

```vue
<!-- 問題 -->
<div :class="`bg-${color}-500`" />  <!-- 不會生效 -->

<!-- 解法：見 references/performance-and-seo.md 的 Tailwind 效能章節 -->
```

### 問題 2：Tailwind 與 Vue Transition 衝突

```vue
<!-- 問題：transition class 被 purge -->
<Transition
  enter-active-class="transition duration-300"
  enter-from-class="opacity-0 translate-y-4"
  enter-to-class="opacity-100 translate-y-0"
  leave-active-class="transition duration-200"
  leave-from-class="opacity-100"
  leave-to-class="opacity-0"
>
  <div v-if="show">內容</div>
</Transition>
<!-- 解法：這些 class 因為直接出現在模板中，通常不會被 purge。
     若仍被 purge，加入 safelist -->
```

### 問題 3：@apply 在 Scoped Style 中不生效

```vue
<!-- 問題 -->
<style scoped>
.btn {
  @apply bg-blue-500 text-white;  /* 可能不生效 */
}
</style>

<!-- 最佳解法：在 template 中直接使用 Tailwind class，避免 @apply -->
<!-- 若必須用 @apply，可以在非 scoped 的 style 區塊中使用 -->
<style>
/* 非 scoped 區塊中 @apply 正常運作 */
/* 但注意要用更具體的選擇器避免全域污染 */
.component-name .btn {
  @apply bg-blue-500 text-white;
}
</style>
```

> **建議**：優先在 template 中直接寫 Tailwind class。`@apply` 在 scoped style 中的行為依 PostCSS 處理順序而異，容易出問題。

### 問題 4：深色模式整合

```typescript
// tailwind.config.ts
export default {
  darkMode: 'class',  // 使用 class 策略
  // ...
}
```

```typescript
// composables/useDarkMode.ts
// 注意：建議改用 @nuxtjs/color-mode 模組，功能更完整
// 以下為手動實作範例
export function useDarkMode() {
  // 使用 undefined 預設值，區分「從未設定」和「明確選擇 light」
  const colorCookie = useCookie<string | undefined>('color-mode', {
    default: () => undefined
  })
  const isDark = useState('dark-mode', () => colorCookie.value === 'dark')

  function applyTheme(dark: boolean) {
    if (!import.meta.client) return
    document.documentElement.classList.toggle('dark', dark)
    colorCookie.value = dark ? 'dark' : 'light'
  }

  function toggle() {
    isDark.value = !isDark.value
    applyTheme(isDark.value)
  }

  // 客戶端初始化與系統偏好監聽（onMounted 僅在客戶端執行，無需 import.meta.client 包裝）
  onMounted(() => {
    // 從未設定 cookie → 使用系統偏好
    if (colorCookie.value === undefined) {
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
      isDark.value = prefersDark
      colorCookie.value = prefersDark ? 'dark' : 'light'
    }
    applyTheme(isDark.value)

    // 監聽系統偏好變化
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
    const handler = (e: MediaQueryListEvent) => {
      isDark.value = e.matches
      applyTheme(e.matches)
    }
    mediaQuery.addEventListener('change', handler)
    onUnmounted(() => mediaQuery.removeEventListener('change', handler))
  })

  return { isDark: readonly(isDark), toggle }
}
```

> **FOUC 防止**：在 `app.vue` 的 `<head>` 中加入內聯腳本讀取 cookie，在 HTML 渲染前套用 `dark` class。或直接使用 `@nuxtjs/color-mode` 模組。

---

## Vitest 測試問題

### 問題 1：Cannot find module '#app'

```typescript
// 確保 vitest.config.ts 使用 @nuxt/test-utils
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    environment: 'nuxt'  // 這行是關鍵
  }
})
```

### 問題 2：mountSuspended 超時

```typescript
// 增加超時時間
import { mountSuspended } from '@nuxt/test-utils/runtime'

it('渲染含非同步資料的元件', async () => {
  const wrapper = await mountSuspended(AsyncComponent, {
    // 先 mock API 端點
  })
}, 10000)  // 增加超時
```

### 問題 3：Auto-import 在測試中不生效

```typescript
// vitest.config.ts
export default defineVitestConfig({
  test: {
    environment: 'nuxt',  // 啟用 Nuxt 環境自動處理 auto-imports
  }
})

// 若仍不生效，手動 import
import { ref, computed } from 'vue'
import { useFetch, useAsyncData } from '#app'
```

---

## 建置與部署問題

### 問題 1：建置時間過長

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 減少不必要的頁面預渲染
  nitro: {
    prerender: {
      routes: ['/'],     // 只預渲染首頁
      crawlLinks: false  // 不自動爬取連結
    }
  },
  // 停用型別檢查（CI 中另外做）
  typescript: {
    typeCheck: false
  }
})
```

### 問題 2：部署後 500 錯誤

```bash
# 檢查清單：
# 1. 環境變數是否正確設定
# 2. Node.js 版本是否 >= 18
# 3. 是否安裝所有依賴
# 4. 建置輸出目錄是否正確

# 本地驗證
npm run build && node .output/server/index.mjs
```

### 問題 3：API route 404

```
# 確認檔案命名正確
server/api/users.get.ts  → GET /api/users    ✓
server/api/users.ts      → 所有方法 /api/users ✓
server/api/users/index.get.ts → GET /api/users ✓

# 注意：HTTP 方法後綴是可選的
# 無後綴 → 匹配所有 HTTP 方法
# .get.ts → 僅匹配 GET
# .post.ts → 僅匹配 POST
```

### Chunk 載入失敗（ChunkLoadError）

```typescript
// plugins/chunk-error.client.ts
// 處理動態匯入的 chunk 載入失敗（常見於部署後使用者仍使用舊版頁面）
export default defineNuxtPlugin((nuxtApp) => {
  // 監聽 Vite 預載錯誤
  nuxtApp.hook('app:chunkError', ({ error }) => {
    console.warn('Chunk 載入失敗，準備重新載入:', error)
    // 避免無限重載
    const reloadKey = 'chunk-reload-count'
    const count = Number(sessionStorage.getItem(reloadKey) || '0')
    if (count < 2) {
      sessionStorage.setItem(reloadKey, String(count + 1))
      window.location.reload()
    }
  })
})
```

```typescript
// nuxt.config.ts — 使用內建的實驗性功能（Nuxt 3.14+）
export default defineNuxtConfig({
  experimental: {
    // 自動處理 chunk 載入錯誤（自動重載頁面）
    emitRouteChunkError: 'automatic'
  }
})
```

常見原因：部署新版本後，使用者瀏覽器快取了舊的路由 manifest，
嘗試載入已不存在的 chunk 檔案。解決方式：
1. 使用 `emitRouteChunkError: 'automatic'` 自動處理（推薦）
2. 或自訂 plugin 控制重載邏輯（如上）
3. 部署策略：保留舊版 chunk 檔案一段時間（CDN 設定）

---

## 錯誤追蹤整合（Sentry）

```typescript
// 安裝
// npm install @sentry/nuxt

// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@sentry/nuxt/module'],
  sentry: {
    sourceMapsUploadOptions: {
      org: 'your-org',
      project: 'your-project'
    }
  },
  sourcemap: {
    client: 'hidden'  // 產生 source map 但不公開
  }
})
```

```typescript
// sentry.client.config.ts（專案根目錄）
import * as Sentry from '@sentry/nuxt'

Sentry.init({
  // sentry.client.config.ts 由 @sentry/nuxt 模組作為 plugin 載入
  // 但為安全起見，使用 import.meta.env 取代 useRuntimeConfig()
  dsn: import.meta.env.NUXT_PUBLIC_SENTRY_DSN ?? '',
  // 正式環境取樣 20% 的交易
  tracesSampleRate: import.meta.dev ? 1.0 : 0.2,
  // Session Replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.replayIntegration()
  ]
})
```

```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/nuxt'

Sentry.init({
  // sentry.server.config.ts 透過 Node --import 載入，Nuxt context 尚未建立
  // 必須使用 process.env 而非 useRuntimeConfig()
  dsn: process.env.NUXT_PUBLIC_SENTRY_DSN ?? '',
  tracesSampleRate: 0.2
})
```

```typescript
// 手動捕獲錯誤與添加上下文
import * as Sentry from '@sentry/nuxt'

// 在 composable 或元件中
function handleCriticalAction() {
  try {
    // ...
  } catch (error) {
    Sentry.captureException(error, {
      tags: { module: 'checkout' },
      extra: { orderId: '12345' }
    })
  }
}

// 設定使用者上下文（登入後）
Sentry.setUser({ id: user.id, email: user.email })
// 登出時清除
Sentry.setUser(null)
```

### 進階 error.vue（含重試機制）

```vue
<!-- error.vue — 增強版（含重試與錯誤類型判斷） -->
<script setup lang="ts">
const props = defineProps<{
  error: NuxtError
}>()

const retryCount = ref(0)
const maxRetries = 3
const isRetrying = ref(false)

const errorConfig = computed(() => {
  const code = props.error.statusCode ?? 500
  const configs: Record<number, { title: string; message: string; canRetry: boolean }> = {
    400: { title: '請求錯誤', message: '請檢查輸入資料後重試', canRetry: false },
    401: { title: '請先登入', message: '您的登入狀態已過期', canRetry: false },
    403: { title: '權限不足', message: '您沒有存取此頁面的權限', canRetry: false },
    404: { title: '找不到頁面', message: '您要找的頁面不存在或已被移除', canRetry: false },
    500: { title: '伺服器錯誤', message: '請稍後再試，我們正在處理中', canRetry: true },
    503: { title: '服務暫時不可用', message: '伺服器正在維護中', canRetry: true }
  }
  return configs[code] ?? { title: '發生錯誤', message: '發生未預期的錯誤', canRetry: true }
})

async function retry() {
  if (retryCount.value >= maxRetries || isRetrying.value) return
  isRetrying.value = true
  retryCount.value++

  // 指數退避：1s, 2s, 4s
  await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, retryCount.value - 1)))
  clearError({ redirect: useRequestURL().pathname })
}

function goHome() {
  clearError({ redirect: '/' })
}
</script>

<template>
  <div class="min-h-screen flex items-center justify-center bg-gray-50">
    <div class="text-center max-w-md px-6">
      <h1 class="text-7xl font-bold text-gray-200">{{ error.statusCode }}</h1>
      <h2 class="mt-4 text-xl font-semibold text-gray-800">{{ errorConfig.title }}</h2>
      <p class="mt-2 text-gray-500">{{ errorConfig.message }}</p>

      <div class="mt-8 flex gap-4 justify-center">
        <button
          v-if="errorConfig.canRetry && retryCount < maxRetries"
          @click="retry"
          :disabled="isRetrying"
          class="px-6 py-3 bg-blue-600 text-white rounded-lg disabled:opacity-50"
        >
          {{ isRetrying ? '重試中...' : `重試 (${maxRetries - retryCount} 次)` }}
        </button>
        <button
          @click="goHome"
          class="px-6 py-3 border border-gray-300 text-gray-700 rounded-lg"
        >
          回首頁
        </button>
      </div>
    </div>
  </div>
</template>
```

---

## 效能問題診斷

### 診斷清單

```
首次載入慢？
├── 檢查 bundle 大小 → npx nuxi analyze
├── 檢查是否有未必要的 SSR → routeRules 設定 ssr: false
├── 檢查字體載入策略 → font-display: swap
└── 檢查第三方腳本 → 延遲載入或使用 useScript

頁面互動卡頓？
├── 檢查是否有 CPU 密集計算 → Web Worker
├── 檢查是否有過多 re-render → Vue DevTools
├── 檢查是否有記憶體洩漏 → Chrome DevTools Memory
└── 檢查 Hydration 成本 → 使用 Component Islands

API 回應慢？
├── 檢查資料庫查詢 → N+1 問題、缺少 index
├── 加入快取 → routeRules cache 或 Redis
├── 檢查是否可以並行查詢 → Promise.all
└── 考慮 Edge Functions → 離用戶更近
```

### Vue DevTools 效能分析

```typescript
// nuxt.config.ts — 開發時啟用 DevTools
export default defineNuxtConfig({
  devtools: { enabled: true }
})
```

> **提示**：時間線分析功能可在 DevTools UI 介面中開啟，無需額外配置。

---

## 路由參數驗證

### 使用 definePageMeta validate

```typescript
// pages/users/[id].vue
definePageMeta({
  validate: async (route) => {
    // id 必須是純數字
    return /^\d+$/.test(route.params.id as string)
  }
})
// 驗證失敗會自動顯示 404 錯誤頁面
```

```typescript
// pages/posts/[slug].vue — 進階驗證（檢查資源是否存在）
definePageMeta({
  validate: async (route) => {
    const slug = route.params.slug as string
    // 正則驗證格式
    if (!/^[a-z0-9-]+$/.test(slug)) return false
    // 可選：檢查資源是否存在（會在 SSR 時執行）
    // const exists = await $fetch(`/api/posts/${slug}/exists`)
    // return exists
    return true
  }
})
```

**注意事項**：
- `validate` 在每次路由變更時執行（SSR 和 CSR 都會）
- 回傳 `false` 或拋出錯誤會觸發 404 頁面
- 可回傳 `createError({ statusCode: 403 })` 來自訂錯誤碼
- 避免在 validate 中進行昂貴的資料庫查詢，簡單格式驗證即可
