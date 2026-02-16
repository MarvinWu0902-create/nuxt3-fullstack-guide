# 資料抓取與狀態管理

## 目錄

1. [useAsyncData vs useFetch](#useasyncdata-vs-usefetch)
2. [資料抓取最佳實踐](#資料抓取最佳實踐)
3. [快取策略](#快取策略)
4. [錯誤處理](#錯誤處理)
5. [Pinia 狀態管理](#pinia-狀態管理)
6. [useState vs Pinia](#usestate-vs-pinia)
7. [SSR 狀態水合](#ssr-狀態水合)

---

## useAsyncData vs useFetch

### 選擇指引

| 特性 | `useFetch` | `useAsyncData` |
|------|-----------|----------------|
| 適用 | 直接 API 呼叫 | 自訂抓取邏輯、多 API 組合 |
| 語法 | `useFetch(url, opts)` | `useAsyncData(key, () => fetch())` |
| Key | 自動產生 | 手動指定唯一 key（3.14+ 可自動推斷）|
| 快取 | 依 URL 自動快取 | 依 key 快取 |
| 適合 | 簡單 CRUD | 複雜資料轉換、多來源組合 |

### useFetch — 適合直接 API 呼叫

```vue
<script setup lang="ts">
// 基礎用法 — 自動處理 SSR 資料序列化
// API 回傳 { data: User[], meta: ... }，用 transform 只取 data 部分
const { data: users, status, error, refresh } = await useFetch<{ data: User[] }>('/api/users', {
  query: { page: 1, perPage: 20 },
  // 轉換回應資料（response 型別為泛型參數 { data: User[] }）
  transform: (response) => response.data,
  // 設定預設值避免 null 檢查
  default: () => []
})

// 攔截器
const token = useCookie('auth-token')
const { data } = await useFetch('/api/protected', {
  onRequest({ options }) {
    if (token.value) {
      options.headers = new Headers(options.headers)
      options.headers.set('Authorization', `Bearer ${token.value}`)
    }
  },
  onResponseError({ response }) {
    if (response.status === 401) {
      navigateTo('/login')
    }
  }
})
</script>
```

### useAsyncData — 適合複雜場景

```vue
<script setup lang="ts">
// 多 API 組合
const { data: dashboard } = await useAsyncData('dashboard', async () => {
  const [users, stats, notifications] = await Promise.all([
    $fetch<User[]>('/api/users'),
    $fetch<Stats>('/api/stats'),
    $fetch<Notification[]>('/api/notifications')
  ])
  return { users, stats, notifications }
})

// 依賴其他響應式資料（watch 模式）
const route = useRoute()
const { data: user } = await useAsyncData(
  `user-${route.params.id}`,
  () => $fetch<User>(`/api/users/${route.params.id}`),
  {
    // 當 route.params.id 變化時重新抓取
    watch: [() => route.params.id]
  }
)
```

### 關鍵規則

**絕對不要在 composable 外部使用 `useFetch`/`useAsyncData`**
— 它們必須在 setup 上下文或 Nuxt 生命週期中呼叫。

**每個 `useAsyncData` 都必須有唯一的 key**
— key 重複會導致資料覆蓋。使用動態 key（如 `` `user-${id}` ``）。

**避免在 `useFetch`/`useAsyncData` 中巢狀使用 `await`**

```typescript
// 錯誤：造成瀑布式請求
const { data: user } = await useFetch('/api/user')
const { data: posts } = await useFetch(`/api/posts?userId=${user.value?.id}`)

// 正確：使用 useAsyncData 組合
const { data } = await useAsyncData('user-posts', async () => {
  const user = await $fetch<User>('/api/user')
  const posts = await $fetch<Post[]>(`/api/posts?userId=${user.id}`)
  return { user, posts }
})
```

---

## 資料抓取最佳實踐

### 封裝 API 層

```typescript
// composables/useApi.ts
export function useApi() {
  const config = useRuntimeConfig()
  const token = useCookie('auth-token')

  const apiFetch = $fetch.create({
    baseURL: config.public.apiBase,
    onRequest({ options }) {
      if (token.value) {
        options.headers = new Headers(options.headers)
        options.headers.set('Authorization', `Bearer ${token.value}`)
      }
    },
    onResponseError({ response }) {
      if (response.status === 401) {
        token.value = null
        navigateTo('/login')
      }
    }
  })

  return { apiFetch }
}

// 使用
const { apiFetch } = useApi()
const { data } = await useAsyncData('users',
  () => apiFetch<User[]>('/users')
)
```

### Plugin 方式提供 API 單例

```typescript
// plugins/api.ts
// 透過 plugin provide 確保整個應用共用同一個 $fetch 實例
export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()
  const token = useCookie('auth-token')

  const api = $fetch.create({
    baseURL: config.public.apiBase,
    onRequest({ options }) {
      if (token.value) {
        options.headers = new Headers(options.headers)
        options.headers.set('Authorization', `Bearer ${token.value}`)
      }
    },
    onResponseError({ response }) {
      if (response.status === 401) {
        token.value = null
        navigateTo('/login')
      }
    }
  })

  return {
    provide: { api }
  }
})

// 使用方式 — 在任何元件或 composable 中
const { $api } = useNuxtApp()
const users = await $api<User[]>('/users')

// 搭配 useAsyncData
const { data } = await useAsyncData('users',
  () => useNuxtApp().$api<User[]>('/users')
)
```

比較說明：

```typescript
// composable 方式 vs plugin 方式：
// - useApi() composable：每次呼叫建立新的 $fetch.create 實例（但因 composable 快取，實際影響小）
// - plugin provide：應用啟動時建立一次，所有地方共用同一實例（推薦用於大型專案）
```

### 樂觀更新（Optimistic Updates）

```typescript
// composables/useOptimisticUpdate.ts
export function useOptimisticUpdate<T>(key: string) {
  async function optimisticUpdate(
    updateFn: () => Promise<T>,
    optimisticData: T
  ) {
    // 透過 useNuxtData 取得響應式資料引用
    const { data } = useNuxtData<T>(key)
    const previousData = data.value

    // 樂觀更新 UI（直接修改 data.value 會觸發 UI 更新）
    data.value = optimisticData

    try {
      const result = await updateFn()
      // 用實際結果更新
      data.value = result
      return result
    } catch (error) {
      // 失敗時回滾
      data.value = previousData
      throw error
    }
  }

  return { optimisticUpdate }
}
```

### Lazy 載入（延遲抓取）

```vue
<script setup lang="ts">
// 不阻塞導航，頁面先渲染再抓取
const { data, status } = useLazyFetch<User[]>('/api/users')

// 或使用 useAsyncData 的 lazy 選項
const { data: stats } = useAsyncData('stats',
  () => $fetch('/api/stats'),
  { lazy: true }
)
</script>

<template>
  <div>
    <SkeletonLoader v-if="status === 'pending'" />
    <UserList v-else-if="data" :users="data" />
  </div>
</template>
```

---

## 快取策略

### 使用 getCachedData 控制快取

```typescript
const { data } = await useAsyncData('config',
  () => $fetch('/api/config'),
  {
    // 透過 transform 記錄抓取時間，供 getCachedData 判斷過期
    transform: (input) => ({ ...input, fetchedAt: new Date() }),
    // 自訂快取策略
    getCachedData(key, nuxtApp) {
      const cached = nuxtApp.payload.data[key] || nuxtApp.static.data[key]
      if (!cached) return undefined

      // 檢查快取是否過期（例如 5 分鐘）
      const expirationDate = new Date(cached.fetchedAt)
      expirationDate.setTime(expirationDate.getTime() + 5 * 60 * 1000)
      if (expirationDate.getTime() < Date.now()) return undefined

      return cached
    }
  }
)
```

### Route Rules 快取（伺服器端）

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // API 回應快取
    '/api/products/**': {
      cache: { maxAge: 300 }          // 快取 5 分鐘
    },
    '/api/config': {
      cache: { maxAge: 3600 }         // 快取 1 小時
    },
    // SWR（Stale-While-Revalidate）
    '/api/feed': {
      cache: { swr: true, maxAge: 60 }
    }
  }
})
```

### 手動清除快取

```typescript
const nuxtApp = useNuxtApp()

// 清除特定 key 的快取
function clearCache(key: string) {
  delete nuxtApp.payload.data[key]
  // 然後重新抓取
  refreshNuxtData(key)
}

// 清除所有快取
function clearAllCache() {
  clearNuxtData()
}
```

---

## 錯誤處理

### 頁面級錯誤

```vue
<script setup lang="ts">
const route = useRoute()
const { data, error } = await useFetch<User>(`/api/users/${route.params.id}`)

// 拋出致命錯誤（顯示 error.vue）
if (error.value) {
  throw createError({
    statusCode: error.value.statusCode ?? 500,
    statusMessage: '找不到使用者',
    fatal: true
  })
}
</script>
```

### 元件級錯誤邊界

```vue
<!-- 使用 NuxtErrorBoundary 隔離錯誤 -->
<template>
  <NuxtErrorBoundary @error="handleError">
    <SomeRiskyComponent />
    <template #error="{ error, clearError }">
      <div class="p-4 bg-red-50 rounded-lg">
        <p class="text-red-600">{{ error.message }}</p>
        <button @click="clearError" class="mt-2 text-sm underline">
          重試
        </button>
      </div>
    </template>
  </NuxtErrorBoundary>
</template>

<script setup lang="ts">
function handleError(error: Error) {
  console.error('元件錯誤:', error)
  // 發送到錯誤追蹤服務
}
</script>
```

### 全域錯誤頁面

```vue
<!-- error.vue（專案根目錄）-->
<script setup lang="ts">
// NuxtError 由 Nuxt 自動匯入，無需手動 import
const props = defineProps<{
  error: NuxtError
}>()

const errorMessages: Record<number, string> = {
  400: '請求格式錯誤',
  401: '請先登入',
  403: '無存取權限',
  404: '頁面不存在',
  500: '伺服器錯誤，請稍後再試'
}

const message = computed(() =>
  errorMessages[props.error.statusCode ?? 500] ?? '發生未知錯誤'
)

function handleError() {
  clearError({ redirect: '/' })
}
</script>

<template>
  <div class="min-h-screen flex items-center justify-center">
    <div class="text-center">
      <h1 class="text-6xl font-bold text-gray-300">
        {{ error.statusCode }}
      </h1>
      <p class="mt-4 text-xl text-gray-600">{{ message }}</p>
      <button
        @click="handleError"
        class="mt-8 px-6 py-3 bg-blue-600 text-white rounded-lg"
      >
        回首頁
      </button>
    </div>
  </div>
</template>
```

---

## Pinia 狀態管理

### Store 設計模式

```typescript
// stores/auth.ts
export const useAuthStore = defineStore('auth', () => {
  // === State ===
  const user = ref<User | null>(null)
  const token = useCookie('auth-token', {
    maxAge: 60 * 15,  // 15 分鐘（與 JWT 到期時間一致）
    secure: true,
    sameSite: 'strict'
  })

  // === Getters ===
  const isLoggedIn = computed(() => !!token.value && !!user.value)
  const isAdmin = computed(() => user.value?.role === 'admin')
  const displayName = computed(() =>
    user.value?.name ?? '訪客'
  )

  // === Actions ===
  async function login(credentials: { email: string; password: string }) {
    const response = await $fetch<{ user: User; token: string }>('/api/auth/login', {
      method: 'POST',
      body: credentials
    })
    user.value = response.user
    token.value = response.token
  }

  async function fetchUser() {
    if (!token.value) return
    try {
      user.value = await $fetch<User>('/api/auth/me', {
        headers: { Authorization: `Bearer ${token.value}` }
      })
    } catch {
      await logout()
    }
  }

  async function logout() {
    user.value = null
    token.value = null
    await navigateTo('/login')
  }

  return {
    user: readonly(user),
    isLoggedIn,
    isAdmin,
    displayName,
    login,
    fetchUser,
    logout
  }
})
```

### Store 之間的互相引用

```typescript
// stores/cart.ts
export const useCartStore = defineStore('cart', () => {
  const authStore = useAuthStore()
  const items = ref<CartItem[]>([])

  const totalPrice = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  async function checkout() {
    if (!authStore.isLoggedIn) {
      navigateTo('/login')
      return
    }
    await $fetch('/api/orders', {
      method: 'POST',
      body: { items: items.value }
    })
    items.value = []
  }

  return { items, totalPrice, checkout }
})
```

---

## useState vs Pinia

### 選擇指引

| 場景 | 使用 | 理由 |
|------|------|------|
| 簡單布林值（如 sidebar 開關）| `useState` | 不需要 actions/getters |
| 主題色/語言偏好 | `useState` | 搭配 `useCookie` 簡潔 |
| 使用者認證狀態 | Pinia | 需要多個 actions、getters |
| 購物車 | Pinia | 複雜操作邏輯 |
| 表單暫存 | `ref` (元件內) | 不需要跨元件共享 |
| 從 API 抓取的列表 | `useAsyncData` | 資料抓取自帶快取機制 |

### useState 範例

```typescript
// composables/useUI.ts

// 全域 UI 狀態 — 跨頁面持久化（SSR 安全）
export function useUI() {
  const isSidebarOpen = useState('sidebar', () => false)
  const activeModal = useState<string | null>('modal', () => null)

  function toggleSidebar() {
    isSidebarOpen.value = !isSidebarOpen.value
  }

  function openModal(name: string) {
    activeModal.value = name
  }

  function closeModal() {
    activeModal.value = null
  }

  return {
    isSidebarOpen: readonly(isSidebarOpen),
    activeModal: readonly(activeModal),
    toggleSidebar,
    openModal,
    closeModal
  }
}
```

---

## SSR 狀態水合

### 水合不匹配的常見原因與解法

**問題 1：直接存取瀏覽器 API**

```typescript
// 錯誤 — SSR 時 window 不存在
const width = ref(window.innerWidth)

// 正確 — 使用 onMounted 或 import.meta.client
const width = ref(0)
onMounted(() => {
  width.value = window.innerWidth
})

// 或使用 Nuxt 提供的 <ClientOnly>
```

**問題 2：時間/隨機值在 SSR 與 CSR 不一致**

```typescript
// 錯誤 — SSR 和 CSR 產生不同值
const timestamp = ref(Date.now())

// 正確 — 使用 useState 確保水合一致
const timestamp = useState('timestamp', () => Date.now())
```

**問題 3：Pinia Store 的 SSR 水合**

```typescript
// plugins/01.auth.ts
export default defineNuxtPlugin(async () => {
  const authStore = useAuthStore()

  // 僅在伺服器端初始化（資料會透過 payload 傳遞到客戶端）
  if (import.meta.server) {
    await authStore.fetchUser()
  }
})
```
