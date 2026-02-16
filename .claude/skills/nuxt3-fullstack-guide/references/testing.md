# 測試策略

## 目錄

1. [Vitest 環境設定](#vitest-環境設定)
2. [元件測試](#元件測試)
3. [Composable 測試](#composable-測試)
4. [API Route 測試](#api-route-測試)
5. [Mock 技巧](#mock-技巧)
6. [測試組織與命名](#測試組織與命名)
7. [E2E 測試整合](#e2e-測試整合)

---

## Vitest 環境設定

### 完整設定

```typescript
// vitest.config.ts
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    environment: 'nuxt',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
    // 測試檔案匹配模式
    include: ['tests/**/*.{test,spec}.{ts,js}'],
    // 覆蓋率設定
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      include: [
        'components/**/*.vue',
        'composables/**/*.ts',
        'utils/**/*.ts',
        'server/**/*.ts',
        'stores/**/*.ts'
      ],
      exclude: [
        'node_modules',
        'tests',
        '.nuxt',
        '**/*.d.ts'
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80
      }
    }
  }
})
```

### 測試 Setup 檔案

```typescript
// tests/setup.ts
import { vi } from 'vitest'

// 全域 mock — 避免 SSR 環境問題
vi.stubGlobal('IntersectionObserver', class {
  observe() {}
  unobserve() {}
  disconnect() {}
})

vi.stubGlobal('ResizeObserver', class {
  observe() {}
  unobserve() {}
  disconnect() {}
})

// 清理
afterEach(() => {
  vi.restoreAllMocks()
})
```

### 套件安裝

```bash
npm install -D @nuxt/test-utils vitest @vue/test-utils @testing-library/vue happy-dom
```

---

## 元件測試

### 基礎元件測試

```typescript
// tests/unit/components/BaseButton.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import BaseButton from '~/components/base/BaseButton.vue'

describe('BaseButton', () => {
  it('渲染預設按鈕', async () => {
    const wrapper = await mountSuspended(BaseButton, {
      slots: { default: '點擊我' }
    })
    expect(wrapper.text()).toBe('點擊我')
    expect(wrapper.classes()).toContain('bg-blue-600')
  })

  it('套用 variant 樣式', async () => {
    const wrapper = await mountSuspended(BaseButton, {
      props: { variant: 'danger' },
      slots: { default: '刪除' }
    })
    expect(wrapper.classes()).toContain('bg-red-600')
  })

  it('disabled 時不可點擊', async () => {
    const wrapper = await mountSuspended(BaseButton, {
      props: { disabled: true },
      slots: { default: '送出' }
    })
    expect(wrapper.attributes('disabled')).toBeDefined()
    expect(wrapper.classes()).toContain('opacity-50')
  })

  it('loading 時顯示載入動畫', async () => {
    const wrapper = await mountSuspended(BaseButton, {
      props: { loading: true },
      slots: { default: '送出中' }
    })
    expect(wrapper.find('svg.animate-spin').exists()).toBe(true)
    expect(wrapper.attributes('disabled')).toBeDefined()
  })

  it('觸發 click 事件', async () => {
    const wrapper = await mountSuspended(BaseButton, {
      slots: { default: '點擊' }
    })
    await wrapper.trigger('click')
    expect(wrapper.emitted('click')).toHaveLength(1)
  })
})
```

### 含非同步資料的元件測試

```typescript
// tests/unit/components/UserProfile.test.ts
import { mountSuspended, registerEndpoint } from '@nuxt/test-utils/runtime'
import UserProfile from '~/components/feature/UserProfile.vue'

describe('UserProfile', () => {
  // 註冊模擬 API 端點
  registerEndpoint('/api/users/1', () => ({
    id: 1,
    name: '測試用戶',
    email: 'test@example.com',
    role: 'admin'
  }))

  it('顯示使用者資訊', async () => {
    const wrapper = await mountSuspended(UserProfile, {
      props: { userId: 1 }
    })
    expect(wrapper.text()).toContain('測試用戶')
    expect(wrapper.text()).toContain('test@example.com')
  })
})
```

### 使用 renderSuspended（Testing Library 風格）

```typescript
// tests/unit/components/LoginForm.test.ts
import { renderSuspended } from '@nuxt/test-utils/runtime'
import { screen, fireEvent } from '@testing-library/vue'
import LoginForm from '~/components/feature/auth/LoginForm.vue'

describe('LoginForm', () => {
  it('驗證必填欄位', async () => {
    await renderSuspended(LoginForm)

    const submitButton = screen.getByRole('button', { name: '登入' })
    await fireEvent.click(submitButton)

    expect(screen.getByText('請輸入電子郵件')).toBeTruthy()
    expect(screen.getByText('請輸入密碼')).toBeTruthy()
  })

  it('送出表單時呼叫 login', async () => {
    await renderSuspended(LoginForm)

    await fireEvent.update(
      screen.getByLabelText('電子郵件'),
      'user@test.com'
    )
    await fireEvent.update(
      screen.getByLabelText('密碼'),
      'password123'
    )
    await fireEvent.click(screen.getByRole('button', { name: '登入' }))

    // 驗證導航或狀態變更
  })
})
```

---

## Composable 測試

### 基礎 Composable 測試

```typescript
// tests/unit/composables/useCounter.test.ts
import { useCounter } from '~/composables/useCounter'

describe('useCounter', () => {
  it('初始值為 0', () => {
    const { count } = useCounter()
    expect(count.value).toBe(0)
  })

  it('可設定初始值', () => {
    const { count } = useCounter(10)
    expect(count.value).toBe(10)
  })

  it('increment 增加計數', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
    increment()
    expect(count.value).toBe(2)
  })

  it('decrement 不低於 0', () => {
    const { count, decrement } = useCounter(0)
    decrement()
    expect(count.value).toBe(0)
  })
})
```

### 含 API 呼叫的 Composable 測試

```typescript
// tests/unit/composables/useUsers.test.ts
import { registerEndpoint } from '@nuxt/test-utils/runtime'

const mockUsers = [
  { id: 1, name: '用戶A', email: 'a@test.com' },
  { id: 2, name: '用戶B', email: 'b@test.com' }
]

describe('useUsers', () => {
  registerEndpoint('/api/users', () => mockUsers)

  it('抓取使用者列表', async () => {
    const { data, status } = await useAsyncData('test-users',
      () => $fetch('/api/users')
    )
    expect(data.value).toEqual(mockUsers)
    expect(status.value).toBe('success')
  })
})
```

### 測試含有 Pinia 的 Composable

```typescript
// tests/unit/stores/auth.test.ts
import { setActivePinia, createPinia } from 'pinia'
import { useAuthStore } from '~/stores/auth'
import { registerEndpoint } from '@nuxt/test-utils/runtime'

describe('useAuthStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  registerEndpoint('/api/auth/login', () => ({
    user: { id: 1, name: '測試', email: 'test@test.com', role: 'user' },
    token: 'mock-jwt-token'
  }))

  it('初始狀態為未登入', () => {
    const store = useAuthStore()
    expect(store.isLoggedIn).toBe(false)
    expect(store.user).toBeNull()
  })

  it('login 後更新狀態', async () => {
    const store = useAuthStore()
    await store.login({ email: 'test@test.com', password: '123456' })
    expect(store.isLoggedIn).toBe(true)
    expect(store.user?.name).toBe('測試')
  })

  it('logout 清除狀態', async () => {
    const store = useAuthStore()
    await store.login({ email: 'test@test.com', password: '123456' })
    await store.logout()
    expect(store.isLoggedIn).toBe(false)
    expect(store.user).toBeNull()
  })
})
```

---

## API Route 測試

### 使用 $fetch 測試 API

```typescript
// tests/integration/api/users.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('GET /api/users', async () => {
  await setup({
    server: true  // 啟動完整 Nuxt 伺服器
  })

  it('回傳使用者列表', async () => {
    const users = await $fetch('/api/users')
    expect(Array.isArray(users)).toBe(true)
  })

  it('支援分頁查詢', async () => {
    const result = await $fetch('/api/users?page=1&perPage=10')
    expect(result.meta.page).toBe(1)
    expect(result.data.length).toBeLessThanOrEqual(10)
  })
})
```

### 單元測試 API Handler

```typescript
// tests/unit/server/api/users.test.ts
import { describe, it, expect, vi } from 'vitest'

// 直接匯入 handler 做單元測試
import handler from '~/server/api/users/index.get'

describe('GET /api/users handler', () => {
  it('回傳正確格式', async () => {
    // 模擬 H3 event
    const event = {
      node: { req: {}, res: {} },
      context: {},
      method: 'GET',
    } as unknown as Parameters<typeof handler>[0]

    const result = await handler(event)
    expect(result).toHaveProperty('data')
  })
})
```

---

## Mock 技巧

### Mock Nuxt 自動匯入

#### 方式一（推薦）：mockNuxtImport

```typescript
// tests/unit/components/ProtectedPage.test.ts
import { vi } from 'vitest'
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

mockNuxtImport('navigateTo', () => vi.fn())
mockNuxtImport('useFetch', () => {
  return () => ({
    data: ref({ name: '測試用戶' }),
    error: ref(null),
    status: ref('success'),
    pending: ref(false),
    refresh: vi.fn(),
    execute: vi.fn()
  })
})
```

#### 方式二（不推薦，不可與方式一混用）：vi.mock

```typescript
// tests/unit/components/ProtectedPage.test.ts — 獨立檔案，勿與 mockNuxtImport 混用
import { vi } from 'vitest'

// 所有 mock 必須在同一個 vi.mock 呼叫中（多次 vi.mock 同一模組會互相覆蓋）
vi.mock('#app', async (importOriginal) => {
  const original = await importOriginal<typeof import('#app')>()
  return {
    ...original,
    navigateTo: vi.fn(),
    useFetch: vi.fn().mockReturnValue({
      data: ref({ name: '測試用戶' }),
      error: ref(null),
      status: ref('success'),
      pending: ref(false),
      refresh: vi.fn(),
      execute: vi.fn()
    })
  }
})
```

### Mock 外部模組

```typescript
// Mock 第三方套件
vi.mock('some-analytics', () => ({
  track: vi.fn(),
  identify: vi.fn()
}))

// Mock useCookie — 注意：這是另一個獨立的測試檔案（例如 CookieDemo.test.ts）
// 不可與上方的 vi.mock('#app') 放在同一檔案，否則會互相覆蓋
// 若需同時 mock useCookie 和 useFetch，必須合併到同一個 vi.mock('#app') 呼叫
const mockCookie = ref<string | null>(null)
vi.mock('#app', async (importOriginal) => {
  const original = await importOriginal<typeof import('#app')>()
  return {
    ...original,
    useCookie: vi.fn(() => mockCookie)
  }
})
```

### 使用 registerEndpoint 模擬 API

```typescript
import { registerEndpoint } from '@nuxt/test-utils/runtime'

// 靜態回應
registerEndpoint('/api/health', () => ({ status: 'ok' }))

// 動態回應
registerEndpoint('/api/users/:id', (event) => {
  const id = event.context.params?.id
  if (id === '999') {
    throw createError({ statusCode: 404, message: '找不到用戶' })
  }
  return { id: Number(id), name: `用戶${id}` }
})

// 帶 HTTP 方法的 handler（在 handler 內檢查 method）
registerEndpoint('/api/users', async (event) => {
  if (event.method === 'POST') {
    const body = await readBody(event)
    return { id: Date.now(), ...body }
  }
  return []
})
```

---

## 測試組織與命名

### 目錄結構

```
tests/
├── unit/                      # 單元測試（不啟動伺服器）
│   ├── components/
│   │   ├── base/
│   │   │   └── BaseButton.test.ts
│   │   └── feature/
│   │       └── UserCard.test.ts
│   ├── composables/
│   │   ├── useAuth.test.ts
│   │   └── useFormValidation.test.ts
│   ├── stores/
│   │   └── auth.test.ts
│   ├── utils/
│   │   └── format.test.ts
│   └── server/
│       └── api/
│           └── users.test.ts
├── integration/               # 整合測試（啟動伺服器）
│   ├── api/
│   │   └── auth-flow.test.ts
│   └── pages/
│       └── login.test.ts
└── setup.ts
```

### 命名規範

```typescript
// describe 區塊：元件/函式名稱
describe('BaseButton', () => {
  // it 區塊：描述行為，使用中文
  it('渲染預設樣式', async () => { })
  it('disabled 狀態下不觸發點擊', async () => { })
  it('loading 時顯示轉圈動畫', async () => { })
})

describe('useAuth', () => {
  describe('login', () => {
    it('成功時設定 token 和使用者', async () => { })
    it('失敗時拋出錯誤', async () => { })
  })
  describe('logout', () => {
    it('清除所有認證狀態', () => { })
    it('導向登入頁', () => { })
  })
})
```

---

## E2E 測試整合

### Playwright 設定

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test'
import type { ConfigOptions } from '@nuxt/test-utils/playwright'

export default defineConfig<ConfigOptions>({
  use: {
    nuxt: {
      rootDir: '.'
    }
  },
  testDir: './tests/e2e',
  timeout: 30000
})
```

### E2E 測試範例

```typescript
// tests/e2e/auth.test.ts
import { expect, test } from '@nuxt/test-utils/playwright'

test('完整登入流程', async ({ page, goto }) => {
  await goto('/login', { waitUntil: 'hydration' })

  await page.fill('[data-testid="email"]', 'user@test.com')
  await page.fill('[data-testid="password"]', 'password123')
  await page.click('[data-testid="submit"]')

  // 驗證導航到 dashboard
  await expect(page).toHaveURL('/dashboard')
  await expect(page.locator('[data-testid="user-name"]')).toContainText('測試用戶')
})
```
