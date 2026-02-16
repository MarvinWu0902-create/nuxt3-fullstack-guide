# 架構與型別系統

## 目錄

1. [Nuxt Layers 模組化架構](#nuxt-layers-模組化架構)
2. [TypeScript 進階型別模式](#typescript-進階型別模式)
3. [Auto-Import 型別安全](#auto-import-型別安全)
4. [元件設計模式](#元件設計模式)
5. [依賴注入與 Provide/Inject](#依賴注入與-provideinject)
6. [大型專案分層策略](#大型專案分層策略)

---

## Nuxt Layers 模組化架構

### 何時使用 Layers

- 多個專案共享基礎設施（UI 元件庫、認證邏輯）
- Monorepo 中跨應用程式共用配置
- 需要可組合的功能模組（如 CMS layer、Analytics layer）

### 設定方式

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  extends: [
    './layers/base',        // 本地 layer
    '@company/ui-layer',    // npm 套件
    'github:org/layer#main' // Git 來源
  ]
})
```

### Layer 目錄結構

```
layers/
├── base/                    # 基礎 layer
│   ├── nuxt.config.ts       # Layer 專屬配置
│   ├── components/
│   │   └── base/            # 基礎 UI 元件
│   ├── composables/
│   │   └── useBase.ts
│   ├── assets/
│   │   └── css/base.css
│   └── utils/
└── auth/                    # 認證 layer
    ├── nuxt.config.ts
    ├── composables/
    │   └── useAuth.ts
    ├── middleware/
    │   └── auth.ts
    ├── server/
    │   └── api/auth/
    └── plugins/
        └── auth.ts
```

### Layer 配置合併規則

```typescript
// layers/base/nuxt.config.ts
export default defineNuxtConfig({
  // Layer 的配置會與主應用合併
  // 陣列型屬性：附加（如 css、plugins）
  // 物件型屬性：深度合併（如 runtimeConfig）
  css: ['~/assets/css/base.css'],
  runtimeConfig: {
    public: {
      baseUrl: 'https://api.example.com'
    }
  }
})
```

**注意**：Layer 中的 `~` 和 `@` 別名指向該 Layer 自身的根目錄，非主應用。

### Monorepo 實務設定

```
# pnpm workspace 結構
monorepo/
├── pnpm-workspace.yaml
├── package.json
├── apps/
│   ├── web/                # 主要 Nuxt 應用
│   │   ├── nuxt.config.ts
│   │   └── package.json
│   └── admin/              # 後台管理 Nuxt 應用
│       ├── nuxt.config.ts
│       └── package.json
├── packages/
│   ├── shared/             # 共用 Nuxt Layer
│   │   ├── nuxt.config.ts
│   │   ├── components/
│   │   ├── composables/
│   │   ├── utils/
│   │   └── package.json
│   ├── ui/                 # UI 元件庫（Nuxt Layer）
│   │   ├── components/
│   │   └── package.json
│   └── types/              # 共用型別定義
│       ├── index.ts
│       └── package.json
└── tsconfig.json           # 根層級 TS 配置
```

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```json
// apps/web/package.json
{
  "name": "@myapp/web",
  "dependencies": {
    "@myapp/shared": "workspace:*",
    "@myapp/ui": "workspace:*",
    "@myapp/types": "workspace:*"
  }
}
```

```typescript
// apps/web/nuxt.config.ts — 擴展共用 Layer
export default defineNuxtConfig({
  extends: [
    '../packages/shared',
    '../packages/ui'
  ]
})
```

### Monorepo 常見陷阱

```typescript
// 陷阱 1：TypeScript 路徑解析
// 問題：跨 package 的型別匯入可能失敗
// 解法：在根 tsconfig.json 設定 paths 或使用 project references
{
  "compilerOptions": {
    "paths": {
      "@myapp/types": ["./packages/types/index.ts"]
    }
  }
}

// 陷阱 2：依賴版本不一致
// 問題：不同 app 使用不同版本的 vue 或 nuxt
// 解法：在根 package.json 使用 pnpm overrides 統一版本
// package.json
{
  "pnpm": {
    "overrides": {
      "vue": "^3.5.0",
      "nuxt": "^3.14.0"
    }
  }
}

// 陷阱 3：Nuxt Layer 的 auto-import 範圍
// 問題：Layer 中的 composables 和 components 可能未被正確偵測
// 解法：確保 Layer 的 package.json 有正確的 main 或 nuxt.config.ts
```

---

## TypeScript 進階型別模式

### 定義 API 回應型別

```typescript
// types/api.ts

// 泛型 API 回應包裝
interface ApiResponse<T> {
  data: T
  meta?: {
    total: number
    page: number
    perPage: number
  }
}

// 錯誤回應
interface ApiError {
  statusCode: number
  message: string
  data?: Record<string, string[]>  // 欄位驗證錯誤
}

// 分頁參數
interface PaginationParams {
  page?: number
  perPage?: number
  sort?: string
  order?: 'asc' | 'desc'
}

// 實際使用
interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user' | 'moderator'
  createdAt: string
}

type UserListResponse = ApiResponse<User[]>
type UserResponse = ApiResponse<User>
```

### 元件 Props 型別

```vue
<script setup lang="ts">
// 推薦：使用 defineProps 的型別參數語法
interface Props {
  title: string
  count?: number
  variant?: 'primary' | 'secondary' | 'danger'
  items: Array<{ id: number; label: string }>
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  variant: 'primary'
})

// Emits 型別定義（Vue 3.3+ 推薦語法）
const emit = defineEmits<{
  update: [id: number]
  delete: [id: number]
  change: [value: string]
}>()
</script>
```

### 使用 defineModel（Nuxt 3.9+）

```vue
<script setup lang="ts">
// 取代 v-model 的 props + emit 模式
const modelValue = defineModel<string>({ required: true })
const count = defineModel<number>('count', { default: 0 })
</script>
```

### 嚴格型別的 composable

```typescript
// composables/useApi.ts
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE'

interface UseApiOptions<T> {
  method?: HttpMethod
  body?: Record<string, unknown>
  query?: Record<string, string | number | undefined>
  headers?: Record<string, string>
  transform?: (data: unknown) => T
  default?: () => T
}

export function useApi<T>(url: string, options: UseApiOptions<T> = {}) {
  // key 需包含 method 和 query，避免同 URL 不同參數時資料覆蓋
  const key = `${options.method ?? 'GET'}:${url}:${JSON.stringify(options.query ?? {})}`
  return useAsyncData<T>(
    key,
    () => $fetch<T>(url, {
      method: options.method ?? 'GET',
      body: options.body,
      query: options.query,
      headers: options.headers
    }),
    {
      transform: options.transform,
      default: options.default
    }
  )
}
```

### 型別斷言與 Type Guard

```typescript
// utils/typeGuards.ts

// Type guard 比 as 斷言更安全
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    'id' in obj &&
    'email' in obj
  )
}

// 用於 API 回應驗證
function assertApiResponse<T>(
  response: unknown,
  guard: (v: unknown) => v is T
): T {
  if (!guard(response)) {
    throw createError({
      statusCode: 500,
      message: '非預期的 API 回應格式'
    })
  }
  return response
}
```

---

## Auto-Import 型別安全

### 自動匯入的運作範圍

Nuxt 3 自動匯入以下來源：
- `components/` — Vue 元件
- `composables/` — 組合式函式（僅頂層檔案或含 index.ts 的資料夾）
- `utils/` — 工具函式
- Vue API（ref, computed, watch 等）
- Nuxt API（useAsyncData, useFetch, navigateTo 等）

### 確保型別正確

```typescript
// 產生型別宣告
// 執行 nuxi prepare 或 nuxi dev 後會生成 .nuxt/types/
// 確保 tsconfig.json 繼承 .nuxt/tsconfig.json

// tsconfig.json
{
  "extends": "./.nuxt/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true  // 推薦啟用
  }
}
```

### 巢狀目錄的自動匯入

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 預設只匯入 composables/ 頂層
  // 若要匯入子目錄：
  imports: {
    dirs: [
      'composables',
      'composables/*/index.{ts,js}',  // 具名資料夾匯出
      'utils'
    ]
  },
  // 元件也支援路徑前綴
  components: [
    {
      path: '~/components',
      pathPrefix: false  // 停用路徑前綴（預設用資料夾名作前綴）
    }
  ]
})
```

**陷阱**：若 `pathPrefix: true`（預設），`components/base/Button.vue` 會註冊為 `<BaseButton>`，非 `<Button>`。

---

## 元件設計模式

### 基礎元件模式（Base Components）

```vue
<!-- components/base/BaseButton.vue -->
<script setup lang="ts">
interface Props {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  loading?: boolean
  disabled?: boolean
  as?: string | Component
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  size: 'md',
  as: 'button'
})

const variantClasses: Record<NonNullable<Props['variant']>, string> = {
  primary: 'bg-blue-600 text-white hover:bg-blue-700',
  secondary: 'bg-gray-200 text-gray-900 hover:bg-gray-300',
  danger: 'bg-red-600 text-white hover:bg-red-700',
  ghost: 'bg-transparent hover:bg-gray-100'
}

const sizeClasses: Record<NonNullable<Props['size']>, string> = {
  sm: 'px-3 py-1.5 text-sm',
  md: 'px-4 py-2 text-base',
  lg: 'px-6 py-3 text-lg'
}
</script>

<template>
  <component
    :is="as"
    :disabled="disabled || loading"
    :class="[
      'inline-flex items-center justify-center rounded-lg font-medium',
      'transition-colors duration-150 focus:outline-none focus:ring-2 focus:ring-offset-2',
      variantClasses[variant],
      sizeClasses[size],
      { 'opacity-50 cursor-not-allowed': disabled || loading }
    ]"
  >
    <svg v-if="loading" class="animate-spin -ml-1 mr-2 h-4 w-4" viewBox="0 0 24 24">
      <circle cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none" class="opacity-25" />
      <path fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" class="opacity-75" />
    </svg>
    <slot />
  </component>
</template>
```

### Slots 與 Expose 模式

```vue
<!-- components/feature/DataTable.vue -->
<script setup lang="ts" generic="T extends Record<string, unknown>">
interface Column<T> {
  key: keyof T
  label: string
  sortable?: boolean
  render?: (value: T[keyof T], row: T) => string
}

interface Props {
  data: T[]
  columns: Column<T>[]
  loading?: boolean
}

defineProps<Props>()

defineSlots<{
  header(props: { columns: Column<T>[] }): void
  cell(props: { column: Column<T>; row: T; value: T[keyof T] }): void
  empty(): void
  loading(): void
}>()

// 暴露方法給父元件
const selectedRows = ref<T[]>([])
defineExpose({
  selectedRows,
  clearSelection: () => { selectedRows.value = [] }
})
</script>
```

---

## 依賴注入與 Provide/Inject

### 型別安全的 Provide/Inject

```typescript
// composables/useTheme.ts
import type { InjectionKey } from 'vue'

interface ThemeContext {
  isDark: Ref<boolean>
  toggle: () => void
  colors: ComputedRef<Record<string, string>>
}

// 使用 Symbol 作為 injection key 確保型別安全
export const ThemeKey: InjectionKey<ThemeContext> = Symbol('theme')

export function provideTheme() {
  const isDark = useState('theme-dark', () => false)

  const context: ThemeContext = {
    isDark,
    toggle: () => { isDark.value = !isDark.value },
    colors: computed(() =>
      isDark.value
        ? { bg: '#1a1a2e', text: '#e0e0e0' }
        : { bg: '#ffffff', text: '#1a1a2e' }
    )
  }

  provide(ThemeKey, context)
  return context
}

export function useTheme(): ThemeContext {
  const context = inject(ThemeKey)
  if (!context) {
    throw new Error('useTheme() 必須在 provideTheme() 之後使用')
  }
  return context
}
```

---

## 大型專案分層策略

### Domain-Driven 目錄結構（適用大型專案）

```
project-root/
├── domains/                   # 依領域分組
│   ├── auth/
│   │   ├── components/
│   │   ├── composables/
│   │   ├── stores/
│   │   ├── types/
│   │   └── pages/
│   ├── billing/
│   │   ├── components/
│   │   ├── composables/
│   │   ├── stores/
│   │   └── types/
│   └── settings/
├── shared/                    # 跨領域共用
│   ├── components/base/
│   ├── composables/
│   ├── utils/
│   └── types/
├── server/                    # 伺服器端（依功能分組）
│   └── api/
│       ├── auth/
│       ├── billing/
│       └── settings/
├── layers/                    # 可抽取的功能 layers
│   └── analytics/
└── nuxt.config.ts
```

配合 `nuxt.config.ts` 掃描 domain 目錄：

```typescript
export default defineNuxtConfig({
  components: [
    { path: '~/shared/components', pathPrefix: false },
    { path: '~/domains/auth/components', prefix: 'Auth' },
    { path: '~/domains/billing/components', prefix: 'Billing' }
  ],
  imports: {
    dirs: [
      'shared/composables',
      'shared/utils',
      'domains/*/composables',
      'domains/*/utils'
    ]
  }
})
```

### 模組化的配置管理

```typescript
// nuxt.config.ts — 使用函式組合大型配置
// defineNuxtConfig 為 Nuxt 自動匯入，無需 import

const authConfig = {
  runtimeConfig: {
    jwtSecret: '',  // 由 NUXT_JWT_SECRET 環境變數自動覆蓋
    public: {
      authProvider: 'local'  // 由 NUXT_PUBLIC_AUTH_PROVIDER 環境變數自動覆蓋
    }
  }
}

const performanceConfig = {
  experimental: {
    payloadExtraction: true,
    renderJsonPayloads: true,
    componentIslands: true
  },
  routeRules: {
    '/api/**': { cors: true },
    '/dashboard/**': { ssr: false },    // SPA 模式
    '/blog/**': { isr: 3600 },          // ISR：每小時重新生成
    '/': { prerender: true }            // 靜態預渲染
  }
}

export default defineNuxtConfig({
  // 注意：直接展開物件會造成淺層合併，nested 屬性會被覆蓋
  // 推薦使用 defu 或直接寫在同一個 config 中
  runtimeConfig: authConfig.runtimeConfig,
  experimental: performanceConfig.experimental,
  routeRules: performanceConfig.routeRules,
  devtools: { enabled: true },
  typescript: { strict: true, typeCheck: true }
})
```

---

## app.config.ts vs runtimeConfig

這是 Nuxt 3 中最常見的混淆點之一。了解何時使用 `app.config.ts` 或 `runtimeConfig` 對於建立可維護的應用至關重要。

### 核心差異對比

| 特性 | app.config.ts | runtimeConfig |
|------|--------------|---------------|
| **用途** | 應用層配置（UI 主題、功能開關等） | 環境變數（API keys、URLs 等） |
| **時機** | 建置時確定，可在執行時透過 hook 更新 | 執行時從環境變數讀取 |
| **響應式** | ✅ 可透過 useAppConfig() 響應式存取 | ✅ 透過 useRuntimeConfig() |
| **序列化** | 完整支援（嵌套物件、陣列） | 僅支援 JSON 可序列化值 |
| **環境變數覆蓋** | ❌ 不支援 | ✅ NUXT_ / NUXT_PUBLIC_ 前綴 |
| **SSR 安全** | ✅ 客戶端/伺服器都可用 | ⚠️ 非 public 的僅伺服器端可用 |
| **HMR** | ✅ 開發時修改立即生效 | ❌ 需重啟開發伺服器 |

### app.config.ts 範例

`app.config.ts` 適用於應用本身的配置，例如 UI 主題、功能開關、預設設定等：

```typescript
// app.config.ts
export default defineAppConfig({
  ui: {
    primaryColor: 'blue',
    rounded: 'lg',
    notifications: {
      position: 'top-right',
      duration: 5000
    }
  },
  features: {
    darkMode: true,
    analytics: true,
    maintenance: false
  },
  branding: {
    logo: '/logo.svg',
    tagline: '打造現代化的 Web 應用'
  }
})
```

### 型別安全

為 `app.config.ts` 添加型別宣告，確保使用時的型別安全：

```typescript
// app.config.ts
export default defineAppConfig({
  ui: {
    primaryColor: 'blue',
    rounded: 'lg',
    notifications: { position: 'top-right' }
  },
  features: {
    darkMode: true,
    analytics: true,
    maintenance: false
  }
})

// 型別宣告
declare module 'nuxt/schema' {
  interface AppConfigInput {
    ui?: {
      primaryColor?: string
      rounded?: string
      notifications?: {
        position?: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'
        duration?: number
      }
    }
    features?: {
      darkMode?: boolean
      analytics?: boolean
      maintenance?: boolean
    }
    branding?: {
      logo?: string
      tagline?: string
    }
  }
}

// 為了確保型別被匯出，需要此行
export {}
```

### 使用方式

在元件或 composable 中使用 `app.config.ts`：

```typescript
// 在元件中使用
const appConfig = useAppConfig()
console.log(appConfig.ui.primaryColor) // 'blue'
console.log(appConfig.features.darkMode) // true

// 執行時更新（僅影響當前 session）
const appConfig = useAppConfig()
appConfig.ui.primaryColor = 'green' // 響應式更新，UI 會即時反應

// 在 computed 中使用
const buttonColor = computed(() => appConfig.ui.primaryColor)
```

### runtimeConfig 範例（對比）

相對地，`runtimeConfig` 適用於需要環境變數覆蓋的配置：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 僅伺服器端可用（API keys、secrets）
    apiSecret: '',
    databaseUrl: '',

    // public 區塊在客戶端和伺服器端都可用
    public: {
      apiBase: 'https://api.example.com',
      appVersion: '1.0.0'
    }
  }
})

// 在伺服器端使用
// server/api/example.ts
export default defineEventHandler((event) => {
  const config = useRuntimeConfig(event)
  console.log(config.apiSecret) // ✅ 可存取
  console.log(config.public.apiBase) // ✅ 可存取
})

// 在客戶端使用
// pages/index.vue
const config = useRuntimeConfig()
// console.log(config.apiSecret) // ❌ 錯誤！客戶端無法存取
console.log(config.public.apiBase) // ✅ 可存取
```

環境變數覆蓋：

```bash
# .env
NUXT_API_SECRET=super-secret-key
NUXT_DATABASE_URL=postgresql://localhost:5432/mydb
NUXT_PUBLIC_API_BASE=https://api-staging.example.com
```

### 決策指引

使用此決策樹來判斷應該使用哪種配置方式：

```
需要在不同環境有不同值？（dev/staging/prod）
├── 是 → runtimeConfig（搭配 NUXT_ 環境變數）
│   └── 包含敏感資訊？（API keys、secrets）
│       ├── 是 → runtimeConfig（非 public 區塊）
│       └── 否 → runtimeConfig.public
│
└── 否 → 是應用本身的配置？（主題、功能開關、UI 設定）
    ├── 是 → app.config.ts
    │   └── 需要執行時動態更新？
    │       ├── 是 → app.config.ts（透過 updateAppConfig()）
    │       └── 否 → app.config.ts
    │
    └── 否 → 是建置階段的配置？（plugins、modules、vite 設定）
        └── 是 → nuxt.config.ts
```

### 實務範例：功能開關系統

結合兩者的優勢，建立功能開關系統：

```typescript
// app.config.ts — 定義預設功能開關
export default defineAppConfig({
  features: {
    newDashboard: false,
    betaEditor: false,
    advancedAnalytics: true
  }
})

// nuxt.config.ts — 允許環境變數覆蓋
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      // 生產環境可透過 NUXT_PUBLIC_FEATURE_FLAGS 覆蓋
      featureFlags: {
        newDashboard: false,
        betaEditor: false
      }
    }
  }
})

// composables/useFeatureFlag.ts — 統一的功能開關 API
type AppConfig = ReturnType<typeof useAppConfig>

export function useFeatureFlag(flagName: keyof AppConfig['features']) {
  const appConfig = useAppConfig()
  const runtimeConfig = useRuntimeConfig()

  // 優先使用 runtime config（環境變數），否則使用 app config
  return computed(() => {
    return runtimeConfig.public.featureFlags?.[flagName]
      ?? appConfig.features[flagName]
      ?? false
  })
}

// 在元件中使用
const showNewDashboard = useFeatureFlag('newDashboard')
```

### 常見陷阱

**陷阱 1：在 app.config.ts 中儲存敏感資料**

```typescript
// ❌ 錯誤：app.config.ts 會被打包到客戶端
export default defineAppConfig({
  apiKey: 'sk_live_1234567890' // 會暴露在客戶端！
})

// ✅ 正確：使用 runtimeConfig
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiKey: '' // 僅伺服器端可用，由 NUXT_API_KEY 環境變數設定
  }
})
```

**陷阱 2：誤以為 runtimeConfig 支援 HMR**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      apiBase: 'https://api.example.com'
    }
  }
})

// 修改 .env 檔案後需要重啟開發伺服器
// 若需要頻繁修改，應考慮使用 app.config.ts
```

**陷阱 3：在客戶端存取非 public 的 runtimeConfig**

```typescript
// pages/index.vue
const config = useRuntimeConfig()
// ❌ 錯誤：客戶端無法存取，會是 undefined
console.log(config.apiSecret)

// ✅ 正確：僅在伺服器端存取
// server/api/data.ts
export default defineEventHandler((event) => {
  const config = useRuntimeConfig(event)
  console.log(config.apiSecret) // ✅ 可以存取
})
```

### 總結

- **app.config.ts**：用於應用本身的配置，支援 HMR，可在執行時更新，適合 UI 主題、功能開關
- **runtimeConfig**：用於環境變數，建置後可透過環境變數覆蓋，適合 API endpoints、feature flags
- **nuxt.config.ts**：用於建置階段配置，如 plugins、modules、Vite 設定

選擇正確的配置方式能讓你的應用更易維護，也能避免敏感資料洩漏的風險。

---

## 進階元件模式（Advanced Component Patterns）

本章節涵蓋企業級 Vue 3 / Nuxt 3 應用中常見的進階元件設計模式，協助建構可重用、可測試、高度彈性的元件 API。所有範例使用 Vue 3.5+ 的 `<script setup>` 語法與 TypeScript。

### Compound Component 模式（複合元件）

複合元件讓一組相關元件透過隱式狀態共享協同運作，提供直覺的 API 且保持內部邏輯封裝。經典範例：`<Tabs>` + `<TabList>` + `<Tab>` + `<TabPanels>` + `<TabPanel>`。

```typescript
// composables/useTabs.ts
import type { InjectionKey, Ref } from 'vue'

interface TabsContext {
  /** 當前選中的 tab id */
  activeId: Ref<string>
  /** 已註冊的 tab id 清單 */
  tabIds: Ref<string[]>
  /** 已註冊的 panel id 清單 */
  panelIds: Ref<string[]>
  /** 註冊一個 tab */
  registerTab: (id: string) => void
  /** 註銷一個 tab */
  unregisterTab: (id: string) => void
  /** 註冊一個 panel */
  registerPanel: (id: string) => void
  /** 註銷一個 panel */
  unregisterPanel: (id: string) => void
  /** 切換到指定 tab */
  setActive: (id: string) => void
}

export const TabsKey: InjectionKey<TabsContext> = Symbol('tabs')

export function useTabsProvider(defaultId?: string) {
  const activeId = ref(defaultId ?? '')
  const tabIds = ref<string[]>([])
  const panelIds = ref<string[]>([])

  const context: TabsContext = {
    activeId,
    tabIds,
    panelIds,
    registerTab: (id) => {
      if (!tabIds.value.includes(id)) {
        tabIds.value.push(id)
        // 若尚未設定預設值，使用第一個註冊的 tab
        if (!activeId.value) activeId.value = id
      }
    },
    unregisterTab: (id) => {
      tabIds.value = tabIds.value.filter((t) => t !== id)
    },
    registerPanel: (id) => {
      if (!panelIds.value.includes(id)) {
        panelIds.value.push(id)
      }
    },
    unregisterPanel: (id) => {
      panelIds.value = panelIds.value.filter((p) => p !== id)
    },
    setActive: (id) => {
      activeId.value = id
    },
  }

  provide(TabsKey, context)
  return context
}

export function useTabsContext(): TabsContext {
  const context = inject(TabsKey)
  if (!context) {
    throw new Error(
      '[Tabs] <Tab> 與 <TabPanel> 必須放在 <Tabs> 內部使用。'
    )
  }
  return context
}
```

```vue
<!-- components/compound/Tabs.vue — 容器元件 -->
<script setup lang="ts">
interface Props {
  /** 預設選中的 tab id */
  defaultValue?: string
  /** 外部控制的 v-model */
  modelValue?: string
}

interface Emits {
  'update:modelValue': [value: string]
  change: [value: string]
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

const { activeId } = useTabsProvider(props.defaultValue)

// 支援 v-model 雙向綁定
if (props.modelValue !== undefined) {
  activeId.value = props.modelValue
}

watch(
  () => props.modelValue,
  (val) => {
    if (val !== undefined) activeId.value = val
  }
)

watch(activeId, (val) => {
  emit('update:modelValue', val)
  emit('change', val)
})
</script>

<template>
  <div class="w-full">
    <slot />
  </div>
</template>
```

```vue
<!-- components/compound/TabList.vue -->
<script setup lang="ts">
defineProps<{
  ariaLabel?: string
}>()
</script>

<template>
  <div
    role="tablist"
    :aria-label="ariaLabel"
    class="flex border-b border-[var(--color-border)]"
  >
    <slot />
  </div>
</template>
```

```vue
<!-- components/compound/Tab.vue -->
<script setup lang="ts">
interface Props {
  /** 此 tab 的唯一識別值 */
  value: string
  /** 是否禁用 */
  disabled?: boolean
}

const props = defineProps<Props>()
const { activeId, registerTab, unregisterTab, setActive } = useTabsContext()

const isActive = computed(() => activeId.value === props.value)

onMounted(() => registerTab(props.value))
onUnmounted(() => unregisterTab(props.value))
</script>

<template>
  <button
    role="tab"
    :id="`tab-${value}`"
    :aria-selected="isActive"
    :aria-controls="`panel-${value}`"
    :tabindex="isActive ? 0 : -1"
    :disabled="disabled"
    :class="[
      'px-4 py-2.5 text-sm font-medium transition-colors whitespace-nowrap',
      'focus-visible:outline-2 focus-visible:outline-brand-500 focus-visible:outline-offset-[-2px]',
      isActive
        ? 'text-brand-600 border-b-2 border-brand-600 -mb-px'
        : 'text-[var(--color-text-secondary)] hover:text-[var(--color-text-primary)]',
      disabled && 'opacity-40 cursor-not-allowed',
    ]"
    @click="!disabled && setActive(value)"
  >
    <slot />
  </button>
</template>
```

```vue
<!-- components/compound/TabPanel.vue -->
<script setup lang="ts">
interface Props {
  /** 對應的 tab value */
  value: string
  /** 是否在非活躍時保持 DOM（僅用 CSS 隱藏） */
  keepAlive?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  keepAlive: false,
})

const { activeId, registerPanel, unregisterPanel } = useTabsContext()
const isActive = computed(() => activeId.value === props.value)

onMounted(() => registerPanel(props.value))
onUnmounted(() => unregisterPanel(props.value))
</script>

<template>
  <div
    v-if="keepAlive || isActive"
    v-show="!keepAlive || isActive"
    role="tabpanel"
    :id="`panel-${value}`"
    :aria-labelledby="`tab-${value}`"
    :tabindex="0"
    class="py-4 focus-visible:outline-none"
  >
    <slot />
  </div>
</template>
```

```vue
<!-- 使用範例：直覺的 Compound Component API -->
<script setup lang="ts">
const activeTab = ref('overview')
</script>

<template>
  <Tabs v-model="activeTab">
    <TabList aria-label="商品資訊">
      <Tab value="overview">總覽</Tab>
      <Tab value="specs">規格</Tab>
      <Tab value="reviews">評價</Tab>
      <Tab value="faq" disabled>常見問題</Tab>
    </TabList>

    <TabPanel value="overview">
      <p>商品總覽內容...</p>
    </TabPanel>
    <TabPanel value="specs">
      <p>技術規格內容...</p>
    </TabPanel>
    <TabPanel value="reviews">
      <p>使用者評價內容...</p>
    </TabPanel>
    <TabPanel value="faq">
      <p>常見問題內容...</p>
    </TabPanel>
  </Tabs>
</template>
```

### Headless / Renderless Component 模式

Headless 元件只封裝邏輯與狀態管理，不渲染任何 UI，讓使用者完全掌控視覺呈現。這種模式透過 scoped slot 或 composable 暴露內部狀態與方法。

```vue
<!-- components/headless/HeadlessToggle.vue — Renderless 元件 -->
<script setup lang="ts">
interface Props {
  /** 初始狀態 */
  defaultValue?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  defaultValue: false,
})

const isOn = defineModel<boolean>({ default: undefined })

// 若未提供 v-model，以 defaultValue 初始化內部狀態
if (isOn.value === undefined) {
  isOn.value = props.defaultValue
}

function toggle() {
  isOn.value = !isOn.value
}

function setOn() {
  isOn.value = true
}

function setOff() {
  isOn.value = false
}

// 透過 slot props 暴露所有狀態與方法
defineSlots<{
  default(props: {
    isOn: boolean
    toggle: () => void
    setOn: () => void
    setOff: () => void
    /** 用於綁定在觸發元素上的 attrs */
    triggerProps: {
      role: 'switch'
      'aria-checked': boolean
      onClick: () => void
      onKeydown: (e: KeyboardEvent) => void
      tabindex: 0
    }
  }): void
}>()

function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault()
    toggle()
  }
}
</script>

<template>
  <slot
    :is-on="isOn"
    :toggle="toggle"
    :set-on="setOn"
    :set-off="setOff"
    :trigger-props="{
      role: 'switch' as const,
      'aria-checked': isOn,
      onClick: toggle,
      onKeydown: handleKeydown,
      tabindex: 0 as const,
    }"
  />
</template>
```

```vue
<!-- 使用範例：同一個 Headless 元件，不同的視覺呈現 -->
<template>
  <!-- 風格 A：經典 Toggle Switch -->
  <HeadlessToggle v-model="darkMode">
    <template #default="{ isOn, triggerProps }">
      <button
        v-bind="triggerProps"
        :class="[
          'relative inline-flex h-6 w-11 items-center rounded-full transition-colors',
          isOn ? 'bg-brand-600' : 'bg-gray-300',
        ]"
      >
        <span
          :class="[
            'inline-block h-4 w-4 transform rounded-full bg-white transition-transform',
            isOn ? 'translate-x-6' : 'translate-x-1',
          ]"
        />
      </button>
    </template>
  </HeadlessToggle>

  <!-- 風格 B：Checkbox 風格 -->
  <HeadlessToggle v-model="notifications">
    <template #default="{ isOn, triggerProps }">
      <div
        v-bind="triggerProps"
        class="flex items-center gap-2 cursor-pointer select-none"
      >
        <div :class="[
          'w-5 h-5 rounded border-2 flex items-center justify-center transition-colors',
          isOn ? 'bg-brand-600 border-brand-600' : 'border-gray-300',
        ]">
          <svg v-if="isOn" class="w-3 h-3 text-white" fill="currentColor" viewBox="0 0 20 20">
            <path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
          </svg>
        </div>
        <span class="text-sm">啟用通知</span>
      </div>
    </template>
  </HeadlessToggle>
</template>
```

```vue
<!-- components/headless/HeadlessDropdown.vue — 進階 Headless 範例 -->
<script setup lang="ts">
interface Props {
  /** 是否在點擊外部時關閉 */
  closeOnClickOutside?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  closeOnClickOutside: true,
})

const isOpen = ref(false)
const triggerRef = ref<HTMLElement | null>(null)
const contentRef = ref<HTMLElement | null>(null)

function toggle() {
  isOpen.value = !isOpen.value
}

function open() {
  isOpen.value = true
}

function close() {
  isOpen.value = false
  // 關閉時將焦點還原至觸發元素
  triggerRef.value?.focus()
}

// 點擊外部關閉（依據 prop 決定是否啟用）
function handleClickOutside(event: MouseEvent) {
  if (!props.closeOnClickOutside) return
  const target = event.target as Node
  if (
    isOpen.value &&
    !triggerRef.value?.contains(target) &&
    !contentRef.value?.contains(target)
  ) {
    close()
  }
}

onMounted(() => {
  document.addEventListener('mousedown', handleClickOutside)
})

onUnmounted(() => {
  document.removeEventListener('mousedown', handleClickOutside)
})

// Escape 關閉
function handleKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape' && isOpen.value) {
    event.preventDefault()
    close()
  }
}

defineSlots<{
  trigger(props: {
    isOpen: boolean
    toggle: () => void
    triggerRef: (el: HTMLElement | null) => void
    triggerProps: {
      'aria-expanded': boolean
      'aria-haspopup': 'true'
      onClick: () => void
      onKeydown: (e: KeyboardEvent) => void
    }
  }): void
  content(props: {
    isOpen: boolean
    close: () => void
    contentRef: (el: HTMLElement | null) => void
  }): void
}>()
</script>

<template>
  <div class="relative inline-block">
    <slot
      name="trigger"
      :is-open="isOpen"
      :toggle="toggle"
      :trigger-ref="(el: HTMLElement | null) => { triggerRef = el }"
      :trigger-props="{
        'aria-expanded': isOpen,
        'aria-haspopup': 'true' as const,
        onClick: toggle,
        onKeydown: handleKeydown,
      }"
    />
    <slot
      v-if="isOpen"
      name="content"
      :is-open="isOpen"
      :close="close"
      :content-ref="(el: HTMLElement | null) => { contentRef = el }"
    />
  </div>
</template>
```

### 元件組合與 `$attrs` 轉發（useAttrs）

在包裝元件時，正確轉發 HTML attributes 和事件監聽器至內部元素，避免屬性遺失或重複。

```vue
<!-- components/base/BaseInput.vue — $attrs 轉發範例 -->
<script setup lang="ts">
/**
 * 此元件包裝原生 <input>，加上 label、錯誤訊息等外層結構。
 * 透過 useAttrs() 將未在 props 中宣告的屬性轉發至內部 <input>。
 */

interface Props {
  label: string
  error?: string
  hint?: string
}

defineProps<Props>()

const modelValue = defineModel<string>({ required: true })

// 使用 useAttrs() 取得所有未被 props 消費的屬性
const attrs = useAttrs()

// 產生唯一 id 供 label-input 關聯
const inputId = useId()
</script>

<!-- 設定 inheritAttrs: false，防止 attrs 自動綁定到根元素 -->
<script lang="ts">
export default { inheritAttrs: false }
</script>

<template>
  <div class="space-y-1.5">
    <!-- Label -->
    <label
      :for="inputId"
      class="block text-sm font-medium text-[var(--color-text-primary)]"
    >
      {{ label }}
    </label>

    <!-- Input：透過 v-bind="attrs" 轉發所有額外屬性 -->
    <input
      :id="inputId"
      v-model="modelValue"
      v-bind="attrs"
      :aria-describedby="error ? `${inputId}-error` : hint ? `${inputId}-hint` : undefined"
      :aria-invalid="!!error"
      :class="[
        'block w-full rounded-lg border px-3 py-2 text-sm transition-colors',
        'focus:outline-none focus:ring-2 focus:ring-offset-0',
        error
          ? 'border-red-500 focus:ring-red-500/30'
          : 'border-[var(--color-border)] focus:ring-brand-500/30 focus:border-brand-500',
      ]"
    />

    <!-- 錯誤訊息 -->
    <p
      v-if="error"
      :id="`${inputId}-error`"
      class="text-sm text-red-600"
      role="alert"
    >
      {{ error }}
    </p>

    <!-- 提示文字 -->
    <p
      v-else-if="hint"
      :id="`${inputId}-hint`"
      class="text-sm text-[var(--color-text-muted)]"
    >
      {{ hint }}
    </p>
  </div>
</template>
```

```vue
<!-- 使用時，所有原生 input 屬性自動轉發 -->
<template>
  <BaseInput
    v-model="email"
    label="電子郵件"
    hint="我們不會分享你的電子郵件"
    :error="emailError"
    type="email"
    placeholder="user@example.com"
    required
    autocomplete="email"
    maxlength="255"
    @focus="handleFocus"
    @blur="validate"
  />
  <!--
    type, placeholder, required, autocomplete, maxlength
    以及 @focus, @blur 都會透過 v-bind="attrs" 轉發至內部 <input>
  -->
</template>
```

```vue
<!-- components/base/BaseFormField.vue — 更通用的 $attrs 轉發模式 -->
<script setup lang="ts">
/**
 * 泛用表單欄位包裝器：提供 label / error / hint 外殼，
 * 內部透過 slot 渲染任何表單控件，並將 attrs 轉發至 slot。
 */

interface Props {
  label: string
  error?: string
  hint?: string
  required?: boolean
}

defineProps<Props>()

const attrs = useAttrs()
const fieldId = useId()

defineSlots<{
  default(props: {
    id: string
    attrs: Record<string, unknown>
    ariaDescribedby: string | undefined
    ariaInvalid: boolean
  }): void
}>()
</script>

<script lang="ts">
export default { inheritAttrs: false }
</script>

<template>
  <div class="space-y-1.5">
    <label
      :for="fieldId"
      class="block text-sm font-medium text-[var(--color-text-primary)]"
    >
      {{ label }}
      <span v-if="required" class="text-red-500 ml-0.5" aria-hidden="true">*</span>
    </label>

    <!-- 將 attrs、id、aria 屬性一起透過 slot props 傳遞 -->
    <slot
      :id="fieldId"
      :attrs="attrs"
      :aria-describedby="error ? `${fieldId}-error` : hint ? `${fieldId}-hint` : undefined"
      :aria-invalid="!!error"
    />

    <p v-if="error" :id="`${fieldId}-error`" class="text-sm text-red-600" role="alert">
      {{ error }}
    </p>
    <p v-else-if="hint" :id="`${fieldId}-hint`" class="text-sm text-[var(--color-text-muted)]">
      {{ hint }}
    </p>
  </div>
</template>
```

```vue
<!-- 搭配使用：可包裝任何表單控件 -->
<template>
  <!-- 包裝 input -->
  <BaseFormField label="姓名" :error="nameError" required>
    <template #default="{ id, attrs, ariaDescribedby, ariaInvalid }">
      <input
        :id="id"
        v-model="name"
        v-bind="attrs"
        :aria-describedby="ariaDescribedby"
        :aria-invalid="ariaInvalid"
        type="text"
        class="block w-full rounded-lg border border-[var(--color-border)] px-3 py-2 text-sm"
      />
    </template>
  </BaseFormField>

  <!-- 包裝 select -->
  <BaseFormField label="國家" hint="選擇您的所在國家">
    <template #default="{ id, attrs, ariaDescribedby }">
      <select
        :id="id"
        v-model="country"
        v-bind="attrs"
        :aria-describedby="ariaDescribedby"
        class="block w-full rounded-lg border border-[var(--color-border)] px-3 py-2 text-sm"
      >
        <option value="">請選擇</option>
        <option value="TW">台灣</option>
        <option value="JP">日本</option>
      </select>
    </template>
  </BaseFormField>
</template>
```

### Class Variance Authority（cva）建立 Variant API

[CVA](https://cva.style/) 提供型別安全的 class variant 管理，取代手動維護的 `Record<string, string>` 映射表。特別適合設計系統中的基礎元件。

```bash
# 安裝
npm install class-variance-authority
# 選用：搭配 tailwind-merge 處理 class 衝突
npm install tailwind-merge
```

```typescript
// utils/cn.ts — 工具函式：合併 class 並解決 Tailwind 衝突
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs))
}
```

```typescript
// components/base/button.variants.ts — 獨立的 variant 定義檔
import { cva, type VariantProps } from 'class-variance-authority'

export const buttonVariants = cva(
  // 基礎樣式（所有 variant 共享）
  [
    'inline-flex items-center justify-center gap-2',
    'rounded-lg font-medium transition-colors',
    'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-brand-500',
    'disabled:opacity-50 disabled:pointer-events-none',
  ],
  {
    variants: {
      variant: {
        primary: 'bg-brand-600 text-white hover:bg-brand-700 active:bg-brand-800',
        secondary: 'bg-[var(--color-surface-secondary)] text-[var(--color-text-primary)] hover:bg-[var(--color-surface-tertiary)] border border-[var(--color-border)]',
        danger: 'bg-red-600 text-white hover:bg-red-700 active:bg-red-800',
        ghost: 'text-[var(--color-text-secondary)] hover:bg-[var(--color-surface-secondary)] hover:text-[var(--color-text-primary)]',
        link: 'text-brand-600 hover:text-brand-700 underline-offset-4 hover:underline p-0 h-auto',
      },
      size: {
        xs: 'h-7 px-2.5 text-xs',
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4 text-sm',
        lg: 'h-11 px-6 text-base',
        xl: 'h-12 px-8 text-lg',
        icon: 'h-10 w-10',            // 正方形，用於圖示按鈕
        'icon-sm': 'h-8 w-8',
        'icon-lg': 'h-12 w-12',
      },
      fullWidth: {
        true: 'w-full',
      },
    },
    // 預設值
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
    // 複合 variant：特定組合的額外樣式
    compoundVariants: [
      {
        variant: 'link',
        size: ['xs', 'sm', 'md', 'lg', 'xl'],
        class: 'px-0',
      },
    ],
  }
)

// 匯出 variant 的型別（用於元件 Props）
export type ButtonVariants = VariantProps<typeof buttonVariants>
```

```vue
<!-- components/base/BaseButton.vue — 搭配 CVA 的按鈕元件 -->
<script setup lang="ts">
import { buttonVariants, type ButtonVariants } from './button.variants'
import { cn } from '~/utils/cn'

interface Props {
  variant?: NonNullable<ButtonVariants['variant']>
  size?: NonNullable<ButtonVariants['size']>
  fullWidth?: boolean
  loading?: boolean
  disabled?: boolean
  as?: string | Component
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  size: 'md',
  as: 'button',
})

const attrs = useAttrs()

const computedClass = computed(() =>
  cn(
    buttonVariants({
      variant: props.variant,
      size: props.size,
      fullWidth: props.fullWidth,
    }),
    // 允許使用者透過 class 覆蓋（twMerge 會正確處理衝突）
    attrs.class as string
  )
)
</script>

<script lang="ts">
export default { inheritAttrs: false }
</script>

<template>
  <component
    :is="as"
    v-bind="{ ...attrs, class: undefined }"
    :disabled="disabled || loading"
    :class="computedClass"
  >
    <!-- 載入動畫 -->
    <svg
      v-if="loading"
      class="animate-spin h-4 w-4"
      xmlns="http://www.w3.org/2000/svg"
      fill="none"
      viewBox="0 0 24 24"
      aria-hidden="true"
    >
      <circle cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" class="opacity-25" />
      <path fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" class="opacity-75" />
    </svg>

    <slot />
  </component>
</template>
```

```vue
<!-- 使用範例 -->
<template>
  <div class="flex flex-wrap gap-3">
    <!-- 基本使用 -->
    <BaseButton>主要按鈕</BaseButton>
    <BaseButton variant="secondary">次要按鈕</BaseButton>
    <BaseButton variant="danger" size="sm">刪除</BaseButton>
    <BaseButton variant="ghost" size="lg">更多</BaseButton>
    <BaseButton variant="link">瞭解更多</BaseButton>

    <!-- 圖示按鈕 -->
    <BaseButton variant="ghost" size="icon" aria-label="設定">
      <svg class="w-5 h-5"><!-- icon --></svg>
    </BaseButton>

    <!-- 載入狀態 -->
    <BaseButton :loading="isSubmitting" :disabled="!isValid">
      {{ isSubmitting ? '送出中...' : '送出' }}
    </BaseButton>

    <!-- 全寬按鈕 -->
    <BaseButton full-width size="lg">
      完成結帳
    </BaseButton>

    <!-- 作為連結渲染 -->
    <BaseButton as="NuxtLink" to="/about" variant="secondary">
      關於我們
    </BaseButton>

    <!-- 覆蓋 class（twMerge 會正確處理衝突） -->
    <BaseButton class="rounded-full shadow-lg">
      自訂樣式
    </BaseButton>
  </div>
</template>
```

### defineModel 進階用法

Vue 3.5+ 的 `defineModel` 大幅簡化了雙向綁定，以下展示進階使用場景。

```vue
<!-- components/form/RatingInput.vue — 多重 v-model -->
<script setup lang="ts">
/**
 * 評分元件：展示多重 defineModel 與 get/set 轉換。
 */

// 主要的 v-model（評分值）
const rating = defineModel<number>('rating', {
  required: true,
  // Vue 3.5+：支援 get/set 轉換器
  get(value) {
    // 確保值在有效範圍內
    return Math.max(0, Math.min(5, value))
  },
  set(value) {
    return Math.round(value) // 只接受整數
  },
})

// 第二個 v-model：評論文字
const comment = defineModel<string>('comment', { default: '' })

// 第三個 v-model：是否匿名
const anonymous = defineModel<boolean>('anonymous', { default: false })

const hoverRating = ref(0)
const isHovering = ref(false)

const displayRating = computed(() =>
  isHovering.value ? hoverRating.value : rating.value
)
</script>

<template>
  <div class="space-y-4">
    <!-- 星星評分 -->
    <div class="flex items-center gap-1" role="radiogroup" aria-label="評分">
      <button
        v-for="star in 5"
        :key="star"
        type="button"
        role="radio"
        :aria-checked="rating === star"
        :aria-label="`${star} 顆星`"
        class="p-1 transition-transform hover:scale-110 focus-visible:outline-2 focus-visible:outline-brand-500 rounded"
        @click="rating = star"
        @mouseenter="hoverRating = star; isHovering = true"
        @mouseleave="isHovering = false"
      >
        <svg
          class="w-6 h-6 transition-colors"
          :class="star <= displayRating ? 'text-yellow-400' : 'text-gray-300'"
          fill="currentColor"
          viewBox="0 0 20 20"
        >
          <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z" />
        </svg>
      </button>
      <span class="ml-2 text-sm text-[var(--color-text-secondary)]">
        {{ rating }} / 5
      </span>
    </div>

    <!-- 評論 -->
    <textarea
      v-model="comment"
      placeholder="分享你的想法（選填）"
      rows="3"
      class="block w-full rounded-lg border border-[var(--color-border)] px-3 py-2 text-sm
             focus:outline-none focus:ring-2 focus:ring-brand-500/30 focus:border-brand-500"
    />

    <!-- 匿名選項 -->
    <label class="flex items-center gap-2 cursor-pointer select-none">
      <input
        v-model="anonymous"
        type="checkbox"
        class="rounded border-[var(--color-border)] text-brand-600 focus:ring-brand-500"
      />
      <span class="text-sm text-[var(--color-text-secondary)]">匿名提交</span>
    </label>
  </div>
</template>
```

```vue
<!-- 使用方式：三個獨立的 v-model -->
<script setup lang="ts">
const userRating = ref(0)
const userComment = ref('')
const isAnonymous = ref(false)

async function submitReview() {
  await $fetch('/api/reviews', {
    method: 'POST',
    body: {
      rating: userRating.value,
      comment: userComment.value,
      anonymous: isAnonymous.value,
    },
  })
}
</script>

<template>
  <RatingInput
    v-model:rating="userRating"
    v-model:comment="userComment"
    v-model:anonymous="isAnonymous"
  />
  <BaseButton
    :disabled="userRating === 0"
    class="mt-4"
    @click="submitReview"
  >
    提交評價
  </BaseButton>
</template>
```

```vue
<!-- components/form/CurrencyInput.vue — defineModel 搭配 get/set 轉換 -->
<script setup lang="ts">
/**
 * 幣值輸入元件：內部顯示格式化數字，外部 v-model 為原始數值。
 * 展示 defineModel 的 get/set 轉換器用法。
 */

interface Props {
  currency?: string
  locale?: string
  min?: number
  max?: number
}

const props = withDefaults(defineProps<Props>(), {
  currency: 'TWD',
  locale: 'zh-TW',
  min: 0,
  max: Number.MAX_SAFE_INTEGER,
})

// v-model 綁定的是數值，但透過 get/set 轉換顯示格式
const amount = defineModel<number>({ default: 0 })

const formatter = computed(() =>
  new Intl.NumberFormat(props.locale, {
    style: 'currency',
    currency: props.currency,
    minimumFractionDigits: 0,
    maximumFractionDigits: 0,
  })
)

// 內部使用的顯示文字
const displayValue = ref(amount.value > 0 ? amount.value.toString() : '')

if (import.meta.client) {
  watch(amount, (val) => {
    if (document.activeElement !== inputRef.value) {
      displayValue.value = val > 0 ? val.toString() : ''
    }
  })
}

const inputRef = ref<HTMLInputElement | null>(null)

function handleInput(event: Event) {
  const raw = (event.target as HTMLInputElement).value.replace(/[^\d]/g, '')
  displayValue.value = raw
  const num = parseInt(raw, 10)
  if (!isNaN(num)) {
    amount.value = Math.max(props.min, Math.min(props.max, num))
  } else {
    amount.value = 0
  }
}

function handleBlur() {
  if (amount.value > 0) {
    displayValue.value = amount.value.toString()
  } else {
    displayValue.value = ''
  }
}

const formattedDisplay = computed(() =>
  amount.value > 0 ? formatter.value.format(amount.value) : ''
)
</script>

<template>
  <div class="relative">
    <input
      ref="inputRef"
      :value="displayValue"
      type="text"
      inputmode="numeric"
      class="block w-full rounded-lg border border-[var(--color-border)] pl-3 pr-20 py-2 text-sm
             focus:outline-none focus:ring-2 focus:ring-brand-500/30 focus:border-brand-500"
      placeholder="輸入金額"
      @input="handleInput"
      @blur="handleBlur"
    />
    <span
      v-if="formattedDisplay"
      class="absolute right-3 top-1/2 -translate-y-1/2 text-sm text-[var(--color-text-muted)] pointer-events-none"
    >
      {{ formattedDisplay }}
    </span>
  </div>
</template>
```

### 模式選擇決策指引

```
選擇元件設計模式：

需要一組元件協同運作？（如 Tabs、Accordion、Menu）
├── 是 → Compound Component
│   └── 使用 provide/inject 共享狀態
│   └── 每個子元件有獨立的 role 與 ARIA
│
└── 否 → 元件需要完全自訂 UI？
    ├── 是 → Headless / Renderless
    │   └── 透過 scoped slot 暴露狀態與方法
    │   └── 使用者掌控所有 DOM 與樣式
    │
    └── 否 → 元件有多種視覺變體？（size / variant / color）
        ├── 是 → CVA（Class Variance Authority）
        │   └── 型別安全的 variant 定義
        │   └── 搭配 tailwind-merge 解決 class 衝突
        │
        └── 否 → 是包裝原生元素的基礎元件？
            ├── 是 → $attrs 轉發 + inheritAttrs: false
            │   └── 使用 useAttrs() 控制屬性目標
            │
            └── 否 → 需要雙向綁定？
                ├── 是 → defineModel（支援多重 + get/set 轉換）
                └── 否 → 標準 defineProps + defineEmits
```
