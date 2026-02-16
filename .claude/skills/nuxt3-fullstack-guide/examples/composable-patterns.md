# 實戰範例：Composable 設計模式

## 概述

常用 composable 模式集合，涵蓋分頁、防抖搜尋、無限滾動、表單狀態、WebSocket、本地儲存等。

---

## 分頁 Composable

```typescript
// composables/usePagination.ts
interface PaginationOptions {
  initialPage?: number
  initialPerPage?: number
  maxPerPage?: number
}

export function usePagination(options: PaginationOptions = {}) {
  const page = ref(options.initialPage ?? 1)
  const perPage = ref(options.initialPerPage ?? 20)
  const total = ref(0)

  const totalPages = computed(() =>
    Math.max(1, Math.ceil(total.value / perPage.value))
  )

  const offset = computed(() => (page.value - 1) * perPage.value)

  const hasPrev = computed(() => page.value > 1)
  const hasNext = computed(() => page.value < totalPages.value)

  const displayRange = computed(() => {
    const start = offset.value + 1
    const end = Math.min(offset.value + perPage.value, total.value)
    return { start, end }
  })

  function goToPage(p: number) {
    page.value = Math.max(1, Math.min(p, totalPages.value))
  }

  function nextPage() {
    if (hasNext.value) page.value++
  }

  function prevPage() {
    if (hasPrev.value) page.value--
  }

  function setTotal(t: number) {
    total.value = t
    // 若當前頁超出範圍，回到最後一頁
    if (page.value > totalPages.value) {
      page.value = totalPages.value
    }
  }

  function reset() {
    page.value = 1
  }

  return {
    page: readonly(page),
    perPage: readonly(perPage),
    total: readonly(total),
    totalPages,
    offset,
    hasPrev,
    hasNext,
    displayRange,
    goToPage,
    nextPage,
    prevPage,
    setTotal,
    reset
  }
}
```

## 防抖搜尋

```typescript
// composables/useDebouncedSearch.ts
interface UseDebouncedSearchOptions {
  delay?: number
  minLength?: number
}

export function useDebouncedSearch(options: UseDebouncedSearchOptions = {}) {
  const { delay = 300, minLength = 0 } = options

  const searchTerm = ref('')
  const debouncedTerm = ref('')
  const isDebouncing = ref(false)

  let timeoutId: ReturnType<typeof setTimeout>

  watch(searchTerm, (newValue) => {
    isDebouncing.value = true
    clearTimeout(timeoutId)

    if (newValue.length < minLength) {
      debouncedTerm.value = ''
      isDebouncing.value = false
      return
    }

    timeoutId = setTimeout(() => {
      debouncedTerm.value = newValue
      isDebouncing.value = false
    }, delay)
  })

  onUnmounted(() => {
    clearTimeout(timeoutId)
  })

  function clear() {
    searchTerm.value = ''
    debouncedTerm.value = ''
    clearTimeout(timeoutId)
    isDebouncing.value = false
  }

  return {
    searchTerm,
    debouncedTerm: readonly(debouncedTerm),
    isDebouncing: readonly(isDebouncing),
    clear
  }
}

// 使用
// const { searchTerm, debouncedTerm } = useDebouncedSearch({ delay: 500 })
// watch(debouncedTerm, (term) => fetchResults(term))
```

## 無限滾動

```typescript
// composables/useInfiniteScroll.ts
interface UseInfiniteScrollOptions<T> {
  url: string
  perPage?: number
  query?: Record<string, string | number | boolean | undefined>
}

// 注意：此 composable 僅在客戶端抓取資料（onMounted 內），SSR 階段不會預載資料
// 適用於 SPA 模式（routeRules: { ssr: false }）或不需要 SEO 的頁面
export function useInfiniteScroll<T>(options: UseInfiniteScrollOptions<T>) {
  const items = shallowRef<T[]>([])
  const page = ref(1)
  const isLoading = ref(false)
  const hasMore = ref(true)
  const error = ref<Error | null>(null)

  async function loadMore() {
    if (isLoading.value || !hasMore.value) return

    isLoading.value = true
    error.value = null

    try {
      const response = await $fetch<{ data: T[]; meta: { totalPages: number } }>(options.url, {
        query: {
          page: page.value,
          perPage: options.perPage ?? 20,
          ...options.query
        }
      })

      items.value = [...items.value, ...response.data]
      hasMore.value = page.value < response.meta.totalPages
      page.value++
    } catch (e: unknown) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      isLoading.value = false
    }
  }

  function reset() {
    items.value = []
    page.value = 1
    hasMore.value = true
    error.value = null
    loadMore()
  }

  // 觀察元素進入視窗
  const sentinel = ref<HTMLElement>()
  let observer: IntersectionObserver | undefined

  onMounted(() => {
    // 初始載入第一頁
    loadMore()

    // 使用 watch 監聽 sentinel ref，處理條件渲染（v-if）的情況
    watch(sentinel, (el) => {
      observer?.disconnect()
      if (!el) return

      observer = new IntersectionObserver(
        (entries) => {
          if (entries[0]?.isIntersecting) {
            loadMore()
          }
        },
        { rootMargin: '200px' }
      )

      observer.observe(el)
    }, { immediate: true })
  })

  onUnmounted(() => {
    observer?.disconnect()
  })

  return {
    items: readonly(items),
    isLoading: readonly(isLoading),
    hasMore: readonly(hasMore),
    error: readonly(error),
    sentinel,
    loadMore,
    reset
  }
}
```

使用：

```vue
<template>
  <div>
    <div v-for="item in items" :key="item.id">{{ item.name }}</div>
    <div ref="sentinel" v-if="hasMore" class="h-10 flex items-center justify-center">
      <span v-if="isLoading" class="text-gray-500">載入中...</span>
    </div>
  </div>
</template>

<script setup lang="ts">
const { items, isLoading, hasMore, sentinel } = useInfiniteScroll<Product>({
  url: '/api/products',
  perPage: 20
})
</script>
```

## 本地儲存

```typescript
// composables/useLocalStorage.ts
export function useLocalStorage<T>(key: string, defaultValue: T) {
  const data = ref<T>(defaultValue) as Ref<T>

  // 從 localStorage 讀取（在 onMounted 中避免 hydration mismatch）
  if (import.meta.client) {
    onMounted(() => {
      const stored = localStorage.getItem(key)
      if (stored) {
        try {
          data.value = JSON.parse(stored)
        } catch {
          data.value = defaultValue
        }
      }

      // 監聽其他分頁的變化
      const handleStorage = (event: StorageEvent) => {
        if (event.key === key && event.newValue) {
          try {
            data.value = JSON.parse(event.newValue)
          } catch {}
        }
      }
      window.addEventListener('storage', handleStorage)
      onUnmounted(() => window.removeEventListener('storage', handleStorage))

      // 在讀取 localStorage 之後才註冊 watch，避免 watch 在 onMounted 之前觸發而覆蓋已存的值
      watch(data, (newValue) => {
        localStorage.setItem(key, JSON.stringify(newValue))
      }, { deep: true })
    })
  }

  function remove() {
    data.value = defaultValue
    if (import.meta.client) {
      localStorage.removeItem(key)
    }
  }

  return { data, remove }
}
```

## 確認對話框

```typescript
// composables/useConfirmDialog.ts
interface ConfirmOptions {
  title?: string
  message: string
  confirmText?: string
  cancelText?: string
  variant?: 'danger' | 'warning' | 'info'
}

// 使用 useState 避免 SSR 全域狀態污染（模組頂層的 ref 在 SSR 中會跨請求共享）
// 使用模組層級 Map 存放 resolver，確保跨元件共享（客戶端安全）
const resolvers = new Map<string, (value: boolean) => void>()

export function useConfirmDialog() {
  const dialogId = 'confirm-dialog'
  const isOpen = useState(`${dialogId}-open`, () => false)
  const options = useState<ConfirmOptions | null>(`${dialogId}-options`, () => null)

  function confirm(opts: ConfirmOptions): Promise<boolean> {
    // 對話框僅在客戶端顯示，SSR 中直接回傳 false
    if (!import.meta.client) return Promise.resolve(false)

    // 若已有未完成的對話框，先以 false 結束前一個 Promise
    resolvers.get(dialogId)?.(false)

    options.value = opts
    isOpen.value = true

    return new Promise((resolve) => {
      resolvers.set(dialogId, resolve)
    })
  }

  function handleConfirm() {
    isOpen.value = false
    resolvers.get(dialogId)?.(true)
    resolvers.delete(dialogId)
  }

  function handleCancel() {
    isOpen.value = false
    resolvers.get(dialogId)?.(false)
    resolvers.delete(dialogId)
  }

  return {
    isOpen: readonly(isOpen),
    options: readonly(options),
    confirm,
    handleConfirm,
    handleCancel
  }
}

// 使用：
// const { confirm } = useConfirmDialog()
// const ok = await confirm({ message: '確定要刪除？', variant: 'danger' })
// if (ok) { ... }
```

## 倒數計時器

```typescript
// composables/useCountdown.ts
export function useCountdown(seconds: number) {
  const remaining = ref(seconds)
  const isRunning = ref(false)
  const isFinished = computed(() => remaining.value <= 0)

  let intervalId: ReturnType<typeof setInterval>

  function start() {
    if (isRunning.value) return
    isRunning.value = true

    intervalId = setInterval(() => {
      remaining.value = Math.max(0, remaining.value - 1)
      if (remaining.value <= 0) {
        stop()
      }
    }, 1000)
  }

  function stop() {
    isRunning.value = false
    clearInterval(intervalId)
  }

  function reset(newSeconds?: number) {
    stop()
    remaining.value = newSeconds ?? seconds
  }

  const formatted = computed(() => {
    const mins = Math.floor(remaining.value / 60)
    const secs = remaining.value % 60
    return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
  })

  onUnmounted(stop)

  return {
    remaining: readonly(remaining),
    isRunning: readonly(isRunning),
    isFinished,
    formatted,
    start,
    stop,
    reset
  }
}
```

## 複製到剪貼簿

```typescript
// composables/useClipboard.ts
export function useClipboard() {
  const copied = ref(false)
  let timeoutId: ReturnType<typeof setTimeout>

  async function copy(text: string, resetDelay = 2000) {
    if (!import.meta.client) return false

    try {
      await navigator.clipboard.writeText(text)
      copied.value = true

      clearTimeout(timeoutId)
      timeoutId = setTimeout(() => {
        copied.value = false
      }, resetDelay)

      return true
    } catch {
      // fallback（deprecated execCommand，用 try-catch 防止二次失敗）
      try {
        const textarea = document.createElement('textarea')
        textarea.value = text
        textarea.style.position = 'fixed'
        textarea.style.opacity = '0'
        document.body.appendChild(textarea)
        textarea.select()
        const success = document.execCommand('copy')
        document.body.removeChild(textarea)

        if (!success) return false

        copied.value = true
        clearTimeout(timeoutId)
        timeoutId = setTimeout(() => { copied.value = false }, resetDelay)
        return true
      } catch {
        return false
      }
    }
  }

  onUnmounted(() => clearTimeout(timeoutId))

  return { copy, copied: readonly(copied) }
}
```

## useWebSocket — WebSocket 連線管理

搭配伺服器端 WebSocket 處理器使用（見 server-and-deployment.md）

```typescript
// composables/useWebSocket.ts
interface UseWebSocketOptions {
  /** 自動重連（預設 true） */
  autoReconnect?: boolean
  /** 最大重連次數（預設 5） */
  maxRetries?: number
  /** 重連基礎延遲 ms（預設 1000，使用指數退避） */
  retryDelay?: number
}

export function useWebSocket(url: string | Ref<string>, options: UseWebSocketOptions = {}) {
  const {
    autoReconnect = true,
    maxRetries = 5,
    retryDelay = 1000
  } = options

  const data = ref<string>('')
  const status = ref<'connecting' | 'open' | 'closed'>('closed')
  let ws: WebSocket | null = null
  let retryCount = 0

  function connect() {
    if (!import.meta.client) return

    const wsUrl = unref(url)
    ws = new WebSocket(wsUrl)
    status.value = 'connecting'

    ws.onopen = () => {
      status.value = 'open'
      retryCount = 0
    }

    ws.onmessage = (event) => {
      data.value = event.data
    }

    ws.onclose = () => {
      status.value = 'closed'
      if (autoReconnect && retryCount < maxRetries) {
        const delay = retryDelay * Math.pow(2, retryCount)
        retryCount++
        setTimeout(connect, delay)
      }
    }

    ws.onerror = () => {
      ws?.close()
    }
  }

  function send(payload: string | object) {
    if (ws?.readyState === WebSocket.OPEN) {
      ws.send(typeof payload === 'object' ? JSON.stringify(payload) : payload)
    }
  }

  function close() {
    retryCount = maxRetries // 阻止自動重連
    ws?.close()
  }

  onMounted(connect)
  onUnmounted(close)

  return {
    data: readonly(data),
    status: readonly(status),
    send,
    close
  }
}
```

使用範例：

```vue
<script setup lang="ts">
const { data, status, send } = useWebSocket('ws://localhost:3000/_ws')

function sendMessage(text: string) {
  send({ type: 'chat', text })
}
</script>

<template>
  <div>
    <span>狀態: {{ status }}</span>
    <p>最新訊息: {{ data }}</p>
  </div>
</template>
```
