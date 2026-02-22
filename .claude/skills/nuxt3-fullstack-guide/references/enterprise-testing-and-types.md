# 企業級 TypeScript 架構與進階測試策略

## 目錄

### Part 1: TypeScript Architecture
1. [Schema-Driven Types](#schema-driven-types)
2. [Runtime Validation Architecture](#runtime-validation-architecture)
3. [Advanced Generics](#advanced-generics)
4. [Type-Safe API Layer](#type-safe-api-layer)

### Part 2: Enterprise Testing Strategy
5. [Testing Pyramid](#testing-pyramid)
6. [Contract Testing](#contract-testing)
7. [Performance Testing](#performance-testing)
8. [Chaos Testing](#chaos-testing)
9. [Testing in CI](#testing-in-ci)

### Part 3: Advanced Testing Techniques
10. [Visual Regression Testing](#visual-regression-testing)
11. [Accessibility Testing（無障礙測試）](#accessibility-testing無障礙測試)
12. [Mutation Testing](#mutation-testing)
13. [Test Data Factories](#test-data-factories)
14. [Network-Level Mocking (MSW)](#network-level-mocking-msw)
15. [Property-Based Testing](#property-based-testing)

### Part 4: Security, Snapshots & Resilience
16. [Security Testing（安全性測試）](#security-testing安全性測試)
17. [Snapshot Testing（快照測試）](#snapshot-testing快照測試)
18. [Infrastructure-Level Chaos Testing（基礎設施層級混沌測試）](#infrastructure-level-chaos-testing基礎設施層級混沌測試)

---

## Schema-Driven Types

### Zod Schema 作為 Single Source of Truth

單一 Zod schema 同時產出前後端共用的所有型別，消除手動同步的風險。

```typescript
// shared/schemas/user.ts
import { z } from 'zod'

// --- Single Source of Truth ---
export const UserSchema = z.object({
  id: z.number().int().positive(),
  name: z.string().min(2).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'moderator']),
  avatar: z.string().url().nullable(),
  createdAt: z.coerce.date(),
  updatedAt: z.coerce.date(),
})
export type User = z.infer<typeof UserSchema>

// API Create Request（省略自動欄位）
export const CreateUserSchema = UserSchema.omit({
  id: true, createdAt: true, updatedAt: true,
}).extend({
  password: z.string().min(8).regex(/[A-Z]/).regex(/[0-9]/),
})
export type CreateUserInput = z.infer<typeof CreateUserSchema>

// API Update Request（全部可選）
export const UpdateUserSchema = CreateUserSchema.partial().omit({ password: true })
export type UpdateUserInput = z.infer<typeof UpdateUserSchema>

// API List Query
export const UserQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  perPage: z.coerce.number().int().min(1).max(100).default(20),
  role: UserSchema.shape.role.optional(),
  search: z.string().optional(),
  sort: z.enum(['name', 'createdAt', 'email']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
})
export type UserQuery = z.infer<typeof UserQuerySchema>

// Form Type（前端表單用，含密碼確認）
export const UserFormSchema = CreateUserSchema.extend({
  passwordConfirm: z.string(),
}).refine((d) => d.password === d.passwordConfirm, {
  message: '密碼不一致', path: ['passwordConfirm'],
})
export type UserFormData = z.infer<typeof UserFormSchema>
```

### 前後端共用

```typescript
// server/api/users/index.post.ts — 後端
import { CreateUserSchema } from '~/shared/schemas/user'
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const validated = CreateUserSchema.parse(body) // 自動推導為 CreateUserInput
  const user = await db.user.create({ data: validated })
  setResponseStatus(event, 201)
  return { data: user }
})
```

```vue
<!-- components/feature/UserForm.vue — 前端 -->
<script setup lang="ts">
import { UserFormSchema, type UserFormData } from '~/shared/schemas/user'
const form = reactive<UserFormData>({
  name: '', email: '', role: 'user', avatar: null, password: '', passwordConfirm: '',
})
async function onSubmit() {
  const result = UserFormSchema.safeParse(form)
  if (!result.success) return // result.error.issues 有完整欄位錯誤
  await $fetch('/api/users', { method: 'POST', body: result.data })
}
</script>
```

---

## Runtime Validation Architecture

### 三層驗證設計

```
前端表單驗證 → API Gateway 驗證 → Domain 驗證
(即時回饋)     (輸入格式檢查)      (業務規則檢查)
```

### 統一的 validateRequest Middleware

```typescript
// server/utils/validate.ts
import type { ZodSchema, ZodError } from 'zod'
import type { H3Event } from 'h3'

export async function validateRequest<B = unknown, Q = unknown, P = unknown>(
  event: H3Event,
  schemas: { body?: ZodSchema<B>; query?: ZodSchema<Q>; params?: ZodSchema<P> },
): Promise<{ body: B; query: Q; params: P }> {
  const errors: Record<string, string[]> = {}
  let body = undefined as B, query = undefined as Q, params = undefined as P

  if (schemas.body) {
    const r = schemas.body.safeParse(await readBody(event))
    if (!r.success) Object.assign(errors, fmtErr(r.error, 'body')); else body = r.data
  }
  if (schemas.query) {
    const r = schemas.query.safeParse(getQuery(event))
    if (!r.success) Object.assign(errors, fmtErr(r.error, 'query')); else query = r.data
  }
  if (schemas.params) {
    const r = schemas.params.safeParse(event.context.params ?? {})
    if (!r.success) Object.assign(errors, fmtErr(r.error, 'params')); else params = r.data
  }
  if (Object.keys(errors).length > 0) throw new ValidationError('輸入驗證失敗', errors)
  return { body, query, params }
}

function fmtErr(error: ZodError, prefix: string): Record<string, string[]> {
  const out: Record<string, string[]> = {}
  for (const i of error.issues) {
    const k = `${prefix}.${i.path.join('.')}`
    ;(out[k] ??= []).push(i.message)
  }
  return out
}
```

### 自訂錯誤型別階層

```typescript
// server/utils/errors.ts
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code: string,
    public readonly data?: Record<string, unknown>,
  ) {
    super(message)
    this.name = this.constructor.name
  }
}
export class ValidationError extends AppError {
  constructor(msg: string, fieldErrors: Record<string, string[]>) {
    super(msg, 400, 'VALIDATION_ERROR', { fieldErrors })
  }
}
export class AuthError extends AppError {
  constructor(msg = '未經授權') { super(msg, 401, 'AUTH_ERROR') }
}
export class ForbiddenError extends AppError {
  constructor(msg = '權限不足') { super(msg, 403, 'FORBIDDEN') }
}
export class NotFoundError extends AppError {
  constructor(resource: string) { super(`找不到${resource}`, 404, 'NOT_FOUND') }
}

// server/plugins/error-handler.ts — 統一處理 AppError
export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('error', (error, { event }) => {
    if (error instanceof AppError && event) {
      setResponseStatus(event, error.statusCode)
      return send(event, JSON.stringify({
        statusCode: error.statusCode, code: error.code,
        message: error.message, data: error.data,
      }), 'application/json')
    }
  })
})
```

---

## Advanced Generics

### Conditional Types — API Response

```typescript
// 根據是否分頁，回傳不同結構
type ApiResponse<T, Paginated extends boolean = false> =
  Paginated extends true
    ? { data: T[]; meta: { total: number; page: number; perPage: number; totalPages: number } }
    : { data: T }

type UserListResponse = ApiResponse<User, true>   // { data: User[]; meta: {...} }
type UserDetailResponse = ApiResponse<User, false> // { data: User }
```

### Template Literal Types — Route 型別安全

```typescript
// 從路徑字串提取動態參數
type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}` ? Param | ExtractParams<Rest>
  : T extends `${string}:${infer Param}` ? Param
  : never

type RouteParams<T extends string> = { [K in ExtractParams<T>]: string }

// 使用：少傳或多傳參數皆型別錯誤
function buildUrl<T extends string>(
  path: T,
  params: ExtractParams<T> extends never ? never : RouteParams<T>,
): string {
  return Object.entries(params as Record<string, string>).reduce(
    (url, [key, val]) => url.replace(`:${key}`, val), path as string,
  )
}
const url = buildUrl('/api/users/:id/posts/:postId', { id: '1', postId: '42' })
```

### `infer` 關鍵字 — Composable Return Type

```typescript
// 從 composable 推導回傳型別
type ComposableReturn<T> = T extends (...args: any[]) => infer R ? R : never

// 從 useAsyncData 推導 data 型別
type AsyncDataReturn<T> =
  T extends (...args: any[]) => Promise<{ data: Ref<infer D> }> ? D : never

// 自動推導 store actions
type StoreActions<T> = {
  [K in keyof T as T[K] extends (...args: any[]) => any ? K : never]:
    T[K] extends (...args: infer A) => infer R ? (...args: A) => R : never
}
```

### Branded Types（防止 ID 混淆）

```typescript
declare const __brand: unique symbol
type Brand<T, B extends string> = T & { readonly [__brand]: B }

export type UserId = Brand<number, 'UserId'>
export type PostId = Brand<number, 'PostId'>

export const UserId = (id: number) => id as UserId
export const PostId = (id: number) => id as PostId

function getUser(id: UserId) { return $fetch(`/api/users/${id}`) }
function getPost(id: PostId) { return $fetch(`/api/posts/${id}`) }

const userId = UserId(1), postId = PostId(42)
getUser(userId)   // OK
// getUser(postId) // 編譯錯誤！PostId 不可賦值給 UserId
```

---

## Type-Safe API Layer

### End-to-End Type Safety

```typescript
// composables/useTypedFetch.ts — 型別安全的 API Client
import type { CreateUserInput, UserQuery } from '~/shared/schemas/user'

export function useUserApi() {
  return {
    list: (query?: UserQuery) => $fetch('/api/users', { query }),
    getById: (id: number) => $fetch(`/api/users/${id}`),
    create: (data: CreateUserInput) => $fetch('/api/users', { method: 'POST', body: data }),
    update: (id: number, data: Partial<CreateUserInput>) =>
      $fetch(`/api/users/${id}`, { method: 'PUT', body: data }),
    remove: (id: number) => $fetch(`/api/users/${id}`, { method: 'DELETE' }),
  }
}
```

```vue
<!-- pages/users/[id].vue — $fetch 型別由 Nitro route types 自動推導 -->
<script setup lang="ts">
const route = useRoute()
const api = useUserApi()
const { data: user, status } = await useAsyncData(
  `user-${route.params.id}`,
  () => api.getById(Number(route.params.id)),
)
</script>
```

### API Contract Testing 搭配 TypeScript

```typescript
// tests/contracts/user-api.contract.test.ts
import { UserSchema, CreateUserSchema } from '~/shared/schemas/user'

describe('User API Contract', () => {
  it('GET /api/users/:id 回應符合 UserSchema', async () => {
    const response = await $fetch('/api/users/1')
    expect(UserSchema.safeParse(response.data).success).toBe(true)
  })
  it('POST /api/users 拒絕不合規的 body', () => {
    const result = CreateUserSchema.safeParse({ name: 'A' })
    expect(result.success).toBe(false)
    if (!result.success) {
      expect(result.error.issues.map((i) => i.path[0])).toContain('email')
    }
  })
})
```

---

## Testing Pyramid

### 比例指引與成本效益

```
                  /   E2E   \         10% — 高成本、覆蓋關鍵使用者流程
                 / Integration \      20% — 中等成本、驗證模組間互動
                /  Unit  Tests  \     70% — 低成本、快速、覆蓋邏輯細節
```

| 層級 | 佔比 | 速度 | Nuxt 3 測試目標 |
|------|------|------|----------------|
| Unit | 70% | < 50ms | composables、utils、stores、server handlers |
| Integration | 20% | 100-500ms | API routes + middleware、元件 + store |
| E2E | 10% | 2-10s | 登入/結帳流程、SSR hydration |

### 各層具體範例

```typescript
// --- Unit: 純邏輯 ---
// tests/unit/utils/price.test.ts
describe('calculateDiscount', () => {
  it('百分比折扣', () => {
    expect(calculateDiscount(1000, { type: 'percent', value: 20 })).toBe(800)
  })
  it('折扣不超過原價', () => {
    expect(calculateDiscount(100, { type: 'fixed', value: 200 })).toBe(0)
  })
})

// --- Integration: API + middleware ---
// tests/integration/api/auth.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'
describe('Auth API', async () => {
  await setup({ server: true })
  it('登入成功回傳 token', async () => {
    const res = await $fetch('/api/auth/login', {
      method: 'POST', body: { email: 'admin@test.com', password: 'Password1' },
    })
    expect(res.token).toBeDefined()
  })
  it('無 token 回傳 401', async () => {
    const err = await $fetch('/api/auth/me').catch((e) => e)
    expect(err.statusCode).toBe(401)
  })
})

// --- E2E: 完整使用者流程 ---
// tests/e2e/checkout.test.ts
import { expect, test } from '@nuxt/test-utils/playwright'
test('完整結帳流程', async ({ page, goto }) => {
  await goto('/products', { waitUntil: 'hydration' })
  await page.click('[data-testid="add-to-cart"]')
  await page.click('[data-testid="checkout-btn"]')
  await page.fill('[data-testid="card-number"]', '4242424242424242')
  await page.click('[data-testid="pay-btn"]')
  await expect(page).toHaveURL(/\/orders\/\d+/)
})
```

---

## Contract Testing

### Provider-Driven（Server schema 為契約基準）

```typescript
// tests/contracts/provider-driven.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'
import { UserSchema } from '~/shared/schemas/user'
import { z } from 'zod'

const UserListContract = z.object({
  data: z.array(UserSchema),
  meta: z.object({ total: z.number(), page: z.number(), perPage: z.number(), totalPages: z.number() }),
})

describe('Provider-Driven Contract', async () => {
  await setup({ server: true })
  it('GET /api/users 符合契約', async () => {
    const res = await $fetch('/api/users?page=1&perPage=5')
    expect(UserListContract.safeParse(res).success).toBe(true)
  })
})
```

### Consumer-Driven（前端定義最小契約，後端驗證滿足）

```typescript
// tests/contracts/consumer-contracts.ts — 前端團隊定義
import { z } from 'zod'
export const UserCardContract = z.object({
  data: z.object({ id: z.number(), name: z.string(), avatar: z.string().nullable() }),
})

// tests/contracts/consumer-driven.test.ts — 後端驗證
import { UserCardContract } from './consumer-contracts'
describe('Consumer-Driven Contract', async () => {
  await setup({ server: true })
  it('GET /api/users/:id 滿足 UserCard 需求', async () => {
    expect(UserCardContract.safeParse(await $fetch('/api/users/1')).success).toBe(true)
  })
})
```

### Schema 穩定性測試

```typescript
// tests/contracts/schema-stability.test.ts
describe('Schema Contract Stability', () => {
  it('CreateUserSchema 接受已知合法輸入', () => {
    const valid = [
      { name: '小明', email: 'ming@test.com', role: 'user', avatar: null, password: 'Pass1234' },
      { name: '管理員', email: 'admin@co.tw', role: 'admin', avatar: 'https://img.co/1.jpg', password: 'Admin123' },
    ]
    valid.forEach((v) => expect(CreateUserSchema.safeParse(v).success).toBe(true))
  })
  it('拒絕已知非法輸入', () => {
    const invalid = [{ name: '', email: 'bad', password: '123' }, { name: '小明', email: 'a@b.c' }]
    invalid.forEach((v) => expect(CreateUserSchema.safeParse(v).success).toBe(false))
  })
})
```

---

## Performance Testing

### Lighthouse CI 配置

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/products'],
      numberOfRuns: 3,
      startServerCommand: 'node .output/server/index.mjs',
      startServerReadyPattern: 'Listening on',
      settings: { preset: 'desktop', chromeFlags: '--no-sandbox --headless' },
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'first-contentful-paint': ['error', { maxNumericValue: 1500 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
      },
    },
    upload: { target: 'temporary-public-storage' },
  },
}
```

### k6 負載測試

```javascript
// tests/load/api-load.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate, Trend } from 'k6/metrics'
const errorRate = new Rate('errors')
const apiDuration = new Trend('api_duration', true)

export const options = {
  stages: [
    { duration: '30s', target: 20 },  { duration: '1m', target: 50 },
    { duration: '30s', target: 100 }, { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    errors: ['rate<0.05'],
  },
}

const BASE = __ENV.BASE_URL || 'http://localhost:3000'
export default function () {
  const page = http.get(`${BASE}/`)
  check(page, {
    'SSR 200': (r) => r.status === 200,
    'SSR < 500ms': (r) => r.timings.duration < 500,
  })
  const api = http.get(`${BASE}/api/users?page=1&perPage=10`)
  apiDuration.add(api.timings.duration)
  check(api, { 'API 200': (r) => r.status === 200 })
  errorRate.add(page.status !== 200 || api.status !== 200)
  sleep(1)
}
```

### SSR Baseline 監控

```typescript
// tests/performance/ssr-baseline.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'
describe('SSR Performance Baseline', async () => {
  await setup({ server: true })
  const ROUTES = [{ path: '/', maxMs: 200 }, { path: '/products', maxMs: 300 }]
  for (const route of ROUTES) {
    it(`${route.path} < ${route.maxMs}ms`, async () => {
      const start = performance.now()
      await $fetch(route.path)
      expect(performance.now() - start).toBeLessThan(route.maxMs)
    })
  }
})
```

---

## Chaos Testing

### 故障注入（Vitest 模擬）

```typescript
// tests/chaos/network-failures.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UserList from '~/components/feature/UserList.vue'

describe('Chaos: 網路故障', () => {
  beforeEach(() => vi.restoreAllMocks())

  it('API 超時 → 顯示重試按鈕', async () => {
    vi.stubGlobal('$fetch', vi.fn().mockRejectedValue(new Error('timeout')))
    const w = await mountSuspended(UserList)
    expect(w.find('[data-testid="retry-btn"]').exists()).toBe(true)
  })

  it('500 錯誤 → 顯示錯誤訊息', async () => {
    vi.stubGlobal('$fetch', vi.fn().mockRejectedValue(
      Object.assign(new Error('ISE'), { statusCode: 500 }),
    ))
    const w = await mountSuspended(UserList)
    expect(w.text()).toContain('伺服器錯誤')
  })

  it('畸形回應 → 不崩潰', async () => {
    vi.stubGlobal('$fetch', vi.fn().mockResolvedValue({ data: null }))
    const w = await mountSuspended(UserList)
    expect(w.text()).not.toContain('TypeError')
  })
})
```

### 資料庫超時

```typescript
// tests/chaos/database-timeout.test.ts
describe('Chaos: DB 超時', () => {
  it('查詢超時拋出錯誤', async () => {
    vi.mock('~/server/utils/db', () => ({
      db: { user: { findMany: vi.fn().mockRejectedValue(new Error('Connection timed out')) } },
    }))
    const handler = (await import('~/server/api/users/index.get')).default
    await expect(handler({ node: { req: {}, res: {} }, context: {} } as any)).rejects.toThrow()
  })
})
```

### Game Day 原則

```typescript
// tests/chaos/game-day.test.ts — 定期排程執行，非每次 CI
describe('Game Day: 服務降級', async () => {
  await setup({ server: true })
  it('健康檢查始終可用', async () => {
    expect((await $fetch('/api/health')).status).toBe('ok')
  })
  it('靜態頁面在 API 故障時仍可渲染', async () => {
    expect(await $fetch('/about')).toContain('</html>')
  })
  it('404 回傳正確狀態碼', async () => {
    expect((await $fetch('/api/nonexistent').catch((e) => e)).statusCode).toBe(404)
  })
})
```

---

## Testing in CI

### 分層並行 GitHub Actions

```yaml
# .github/workflows/test.yml — unit 與 integration 並行，e2e 等兩者通過
name: Test Pipeline
on: [push, pull_request]
jobs:
  unit:                        # 第一層：快速回饋
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm vitest run --project unit
  integration:                 # 第二層：與 unit 並行
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: test, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm vitest run --project integration
        env: { DATABASE_URL: 'postgresql://test:test@localhost:5432/test' }
  e2e:                         # 第三層：需 unit + integration 通過
    needs: [unit, integration]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile && npx playwright install --with-deps chromium
      - run: pnpm build && pnpm playwright test
```

### Vitest Workspace 分層

```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config'
export default defineWorkspace([
  {
    extends: './vitest.config.ts',
    test: { name: 'unit', include: ['tests/unit/**/*.test.ts'], environment: 'nuxt' },
  },
  {
    extends: './vitest.config.ts',
    test: { name: 'integration', include: ['tests/integration/**/*.test.ts'], environment: 'nuxt', testTimeout: 15000 },
  },
  {
    extends: './vitest.config.ts',
    test: { name: 'contracts', include: ['tests/contracts/**/*.test.ts'], environment: 'nuxt' },
  },
])
```

### Flakiness 處理

```typescript
// vitest.config.ts
import { defineVitestConfig } from '@nuxt/test-utils/config'
export default defineVitestConfig({
  test: {
    retry: process.env.CI ? 2 : 0,          // CI 自動重試
    exclude: ['tests/quarantine/**', 'node_modules/**'], // 隔離不穩定測試
    testTimeout: 10000,
    hookTimeout: 15000,
    pool: 'forks',                           // 每個檔案獨立 process
    poolOptions: { forks: { singleFork: false } },
  },
})
```

```typescript
// tests/helpers/flaky.ts — Flaky test 標記
import { describe } from 'vitest'
/** CI 跳過，本地執行 */
export const flakyDescribe = process.env.CI ? describe.skip : describe
```

### Coverage 門檻與趨勢

```typescript
// vitest.config.ts — coverage
export default defineVitestConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json-summary', 'html', 'lcov'],
      include: ['components/**/*.vue', 'composables/**/*.ts', 'utils/**/*.ts', 'server/**/*.ts'],
      exclude: ['**/*.d.ts', '**/*.test.ts', 'server/plugins/**'],
      thresholds: { lines: 80, functions: 80, branches: 75, statements: 80 },
    },
  },
})
```

CI 趨勢監控：搭配 `davelosert/vitest-coverage-report-action@v2` 在 PR 留下 coverage diff 報告，追蹤 `coverage/coverage-summary.json` 的變化趨勢。

---

## Visual Regression Testing

視覺回歸測試（Visual Regression Testing）能在每次程式碼變更後自動比對頁面截圖，捕捉 CSS 破壞、版面位移、元件樣式退化等肉眼才能發現的問題。

### Percy / Chromatic 整合 Nuxt

Percy 與 Chromatic 是兩大主流雲端視覺回歸平台，皆支援與 Nuxt 的 CI 流程整合。

```typescript
// Percy 整合：在 Playwright E2E 測試中擷取快照
// tests/e2e/visual/homepage.test.ts
import { test } from '@nuxt/test-utils/playwright'
import percySnapshot from '@percy/playwright'

test.describe('首頁視覺回歸', () => {
  test('桌面版首頁快照', async ({ page, goto }) => {
    await goto('/', { waitUntil: 'hydration' })
    // Percy 會自動上傳快照至雲端，與 baseline 進行像素比對
    await percySnapshot(page, '首頁 - 桌面版', {
      widths: [1280, 1920],
      minHeight: 1024,
    })
  })

  test('手機版首頁快照', async ({ page, goto }) => {
    await page.setViewportSize({ width: 375, height: 812 })
    await goto('/', { waitUntil: 'hydration' })
    await percySnapshot(page, '首頁 - 手機版')
  })
})
```

```yaml
# .github/workflows/visual.yml — Percy CI 整合
name: Visual Regression
on: [pull_request]
jobs:
  percy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile && npx playwright install --with-deps chromium
      - run: pnpm build
      - run: npx percy exec -- pnpm playwright test tests/e2e/visual/
        env:
          PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
```

### Playwright 內建視覺比對設定

Playwright 本身提供 `toHaveScreenshot()` 斷言，無需外部服務即可進行視覺回歸測試。

```typescript
// playwright.config.ts — 視覺比對全域設定
import { defineConfig } from '@playwright/test'

export default defineConfig({
  expect: {
    toHaveScreenshot: {
      // 允許的像素差異比例（0.1% 以內視為相同）
      maxDiffPixelRatio: 0.001,
      // 動畫完成後才截圖
      animations: 'disabled',
      // 截圖尺寸一致性
      scale: 'device',
    },
  },
  // 快照儲存路徑
  snapshotPathTemplate: '{testDir}/__screenshots__/{testFilePath}/{arg}{ext}',
})
```

### 截圖式元件測試

針對單一元件進行截圖回歸，粒度比全頁截圖更精細，定位問題更快。

```typescript
// tests/e2e/visual/components.test.ts
import { test, expect } from '@nuxt/test-utils/playwright'

test.describe('元件視覺回歸', () => {
  test('ProductCard 元件快照', async ({ page, goto }) => {
    await goto('/products', { waitUntil: 'hydration' })
    const card = page.locator('[data-testid="product-card"]').first()
    // 等待圖片載入完成
    await card.locator('img').waitFor({ state: 'visible' })
    // 僅截取該元件區域
    await expect(card).toHaveScreenshot('product-card.png', {
      maxDiffPixelRatio: 0.001,
    })
  })

  test('NavigationBar 在不同狀態下的快照', async ({ page, goto }) => {
    await goto('/', { waitUntil: 'hydration' })
    const nav = page.locator('[data-testid="navbar"]')

    // 未登入狀態
    await expect(nav).toHaveScreenshot('navbar-logged-out.png')

    // 模擬登入後
    await page.evaluate(() => {
      localStorage.setItem('auth_token', 'mock-token')
    })
    await page.reload({ waitUntil: 'networkidle' })
    await expect(nav).toHaveScreenshot('navbar-logged-in.png')
  })

  test('深色模式切換', async ({ page, goto }) => {
    await goto('/', { waitUntil: 'hydration' })
    // 切換至深色模式
    await page.click('[data-testid="theme-toggle"]')
    await expect(page).toHaveScreenshot('homepage-dark-mode.png', {
      fullPage: true,
    })
  })
})
```

### CI Pipeline 整合視覺差異

```yaml
# .github/workflows/visual-playwright.yml — 使用 Playwright 內建截圖比對
name: Visual Diff (Playwright)
on: [pull_request]
jobs:
  visual:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile && npx playwright install --with-deps chromium
      - run: pnpm build
      - run: pnpm playwright test tests/e2e/visual/
      # 截圖差異作為 artifact 上傳，供 PR Review 查看
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: visual-diff-report
          path: test-results/
          retention-days: 14
```

---

## Accessibility Testing（無障礙測試）

無障礙測試確保應用程式符合 WCAG 2.1 AA 標準，讓所有使用者（包括使用螢幕閱讀器、鍵盤操作的使用者）都能順利使用。

### axe-core 整合 Vitest

axe-core 是業界標準的無障礙檢測引擎，可在單元測試與整合測試中自動掃描 DOM 違規。

```typescript
// tests/setup/a11y.ts — 全域 a11y 測試工具
import { configureAxe, getViolations } from 'vitest-axe'
import type { Result } from 'axe-core'

// 設定預設規則（WCAG 2.1 AA）
configureAxe({
  rules: {
    // 根據專案需求自訂規則
    'color-contrast': { enabled: true },
    'link-name': { enabled: true },
    'image-alt': { enabled: true },
    region: { enabled: true },
  },
})

/**
 * 對 DOM 容器執行 axe-core 掃描，回傳可讀的違規摘要
 */
export async function checkA11y(container: HTMLElement): Promise<Result[]> {
  const violations = await getViolations(container)
  return violations
}
```

```typescript
// tests/unit/components/a11y/UserForm.a11y.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { axe, toHaveNoViolations } from 'vitest-axe'
import UserForm from '~/components/feature/UserForm.vue'

// 擴充 Vitest 的 expect matcher
expect.extend(toHaveNoViolations)

describe('UserForm 無障礙測試', () => {
  it('表單無 axe-core 違規', async () => {
    const wrapper = await mountSuspended(UserForm)
    const results = await axe(wrapper.element as HTMLElement)
    expect(results).toHaveNoViolations()
  })

  it('錯誤狀態下仍無 a11y 違規', async () => {
    const wrapper = await mountSuspended(UserForm, {
      props: {
        errors: {
          name: '名稱為必填',
          email: '電子郵件格式不正確',
        },
      },
    })
    const results = await axe(wrapper.element as HTMLElement)
    expect(results).toHaveNoViolations()
  })
})
```

### @testing-library 無障礙查詢

使用 @testing-library 的無障礙查詢方法，確保元件可被輔助技術正確識別。

```typescript
// tests/unit/components/a11y/ProductCard.a11y.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { within } from '@testing-library/vue'
import ProductCard from '~/components/feature/ProductCard.vue'

describe('ProductCard 無障礙查詢', () => {
  const mockProduct = {
    id: 1,
    name: '無線藍牙耳機',
    price: 2990,
    image: '/images/headphones.jpg',
    description: '高音質藍牙 5.3 耳機',
  }

  it('圖片具有替代文字', async () => {
    const wrapper = await mountSuspended(ProductCard, {
      props: { product: mockProduct },
    })
    const screen = within(wrapper.element as HTMLElement)
    // getByRole 會在找不到元素時自動報錯
    const img = screen.getByRole('img', { name: /無線藍牙耳機/i })
    expect(img).toBeDefined()
    expect(img.getAttribute('alt')).toBeTruthy()
  })

  it('加入購物車按鈕有明確標籤', async () => {
    const wrapper = await mountSuspended(ProductCard, {
      props: { product: mockProduct },
    })
    const screen = within(wrapper.element as HTMLElement)
    const button = screen.getByRole('button', { name: /加入購物車/i })
    expect(button).toBeDefined()
  })

  it('價格資訊使用語意化標記', async () => {
    const wrapper = await mountSuspended(ProductCard, {
      props: { product: mockProduct },
    })
    const screen = within(wrapper.element as HTMLElement)
    // 確認價格文字可被螢幕閱讀器識別
    expect(screen.getByText(/2,990/)).toBeDefined()
  })

  it('互動元素可透過鍵盤聚焦', async () => {
    const wrapper = await mountSuspended(ProductCard, {
      props: { product: mockProduct },
    })
    const screen = within(wrapper.element as HTMLElement)
    const button = screen.getByRole('button', { name: /加入購物車/i })
    // 確認 tabIndex 未被設為負值
    expect(button.tabIndex).toBeGreaterThanOrEqual(0)
  })
})
```

### WCAG 2.1 AA 合規測試模式

```typescript
// tests/unit/components/a11y/wcag-patterns.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { axe, toHaveNoViolations } from 'vitest-axe'
import AppModal from '~/components/ui/AppModal.vue'
import AppAlert from '~/components/ui/AppAlert.vue'

expect.extend(toHaveNoViolations)

describe('WCAG 2.1 AA 合規模式', () => {
  describe('模態對話框（Modal）— WCAG 2.4.3 焦點順序', () => {
    it('開啟 Modal 時焦點移至 Modal 內部', async () => {
      const wrapper = await mountSuspended(AppModal, {
        props: { modelValue: true, title: '確認刪除' },
        slots: { default: '<button>確定</button><button>取消</button>' },
      })
      // Modal 容器應有 role="dialog"
      const dialog = wrapper.find('[role="dialog"]')
      expect(dialog.exists()).toBe(true)
      // 應有 aria-labelledby 或 aria-label
      expect(
        dialog.attributes('aria-labelledby') || dialog.attributes('aria-label'),
      ).toBeTruthy()
      // 應有 aria-modal="true"
      expect(dialog.attributes('aria-modal')).toBe('true')
    })
  })

  describe('警示訊息（Alert）— WCAG 4.1.3 狀態訊息', () => {
    it('成功提示具有正確的 ARIA 角色', async () => {
      const wrapper = await mountSuspended(AppAlert, {
        props: { type: 'success', message: '儲存成功' },
      })
      const alert = wrapper.find('[role="status"]')
      expect(alert.exists()).toBe(true)
      expect(alert.text()).toContain('儲存成功')
    })

    it('錯誤提示使用 role="alert"', async () => {
      const wrapper = await mountSuspended(AppAlert, {
        props: { type: 'error', message: '操作失敗' },
      })
      const alert = wrapper.find('[role="alert"]')
      expect(alert.exists()).toBe(true)
      expect(alert.attributes('aria-live')).toBe('assertive')
    })
  })

  describe('表單 — WCAG 1.3.1 資訊與關聯', () => {
    it('所有表單欄位皆有關聯的 label', async () => {
      const FormComponent = defineComponent({
        template: `
          <form>
            <label for="email">電子郵件</label>
            <input id="email" type="email" required aria-describedby="email-hint" />
            <span id="email-hint">請輸入有效的電子郵件地址</span>
          </form>
        `,
      })
      const wrapper = await mountSuspended(FormComponent)
      const results = await axe(wrapper.element as HTMLElement)
      expect(results).toHaveNoViolations()
    })
  })
})
```

### Lighthouse 無障礙分數 CI 門檻

```yaml
# lighthouserc.js 中已包含 a11y 設定（見 Performance Testing 章節）
# 以下為獨立的 a11y 專用 CI 步驟
# .github/workflows/a11y.yml
name: Accessibility Audit
on: [pull_request]
jobs:
  a11y:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - name: Lighthouse A11y Audit
        uses: treosh/lighthouse-ci-action@v12
        with:
          configPath: ./lighthouserc-a11y.js
          uploadArtifacts: true
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

```javascript
// lighthouserc-a11y.js — 專注無障礙的 Lighthouse 設定
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/products',
        'http://localhost:3000/login',
        'http://localhost:3000/contact',
      ],
      numberOfRuns: 1,
      startServerCommand: 'node .output/server/index.mjs',
      startServerReadyPattern: 'Listening on',
      settings: {
        onlyCategories: ['accessibility'],
        chromeFlags: '--no-sandbox --headless',
      },
    },
    assert: {
      assertions: {
        'categories:accessibility': ['error', { minScore: 0.95 }],
        // 個別 a11y 規則門檻
        'aria-allowed-attr': 'error',
        'aria-required-attr': 'error',
        'button-name': 'error',
        'color-contrast': 'error',
        'image-alt': 'error',
        'label': 'error',
        'link-name': 'error',
      },
    },
    upload: { target: 'temporary-public-storage' },
  },
}
```

---

## Mutation Testing

變異測試（Mutation Testing）透過對原始碼注入微小變異（如將 `===` 改為 `!==`、刪除條件分支等），驗證現有測試套件是否能偵測到這些變異。若測試套件無法「殺死」某個變異體，代表該區域的測試覆蓋不足。

### Stryker 整合 Vitest（Nuxt 專案）

```typescript
// stryker.config.mts
import { defineConfig } from '@stryker-mutator/core'

export default defineConfig({
  // 使用 Vitest runner
  testRunner: 'vitest',
  vitest: {
    configFile: 'vitest.config.ts',
    // 僅跑 unit 測試（速度考量）
    dir: 'tests/unit',
  },
  // 限定變異範圍：僅對核心業務邏輯進行變異
  mutate: [
    'composables/**/*.ts',
    'utils/**/*.ts',
    'server/utils/**/*.ts',
    'shared/schemas/**/*.ts',
    // 排除測試檔與型別宣告
    '!**/*.test.ts',
    '!**/*.spec.ts',
    '!**/*.d.ts',
  ],
  // 變異運算子設定
  mutator: {
    excludedMutations: [
      // 排除不太有意義的變異
      'StringLiteral',  // 字串常量替換（如 i18n key）
    ],
  },
  // 報告格式
  reporters: ['html', 'clear-text', 'progress', 'json'],
  htmlReporter: {
    fileName: 'reports/mutation/index.html',
  },
  jsonReporter: {
    fileName: 'reports/mutation/mutation-report.json',
  },
  // 效能最佳化
  concurrency: 4,
  timeoutMS: 30000,
  // 增量模式：僅對有變更的檔案進行變異
  incremental: true,
  incrementalFile: 'reports/mutation/.stryker-incremental.json',
})
```

### 設定範例

```json
// package.json — 新增 mutation test 指令
{
  "scripts": {
    "test:unit": "vitest run --project unit",
    "test:mutation": "stryker run",
    "test:mutation:incremental": "stryker run --incremental",
    "test:mutation:report": "stryker run && open reports/mutation/index.html"
  },
  "devDependencies": {
    "@stryker-mutator/core": "^8.6.0",
    "@stryker-mutator/vitest-runner": "^8.6.0"
  }
}
```

### 解讀變異分數（Mutation Score）

```typescript
// 範例：被變異測試揭露的覆蓋不足
// utils/pricing.ts
export function calculateFinalPrice(
  basePrice: number,
  discount: number,
  taxRate: number,
): number {
  // Stryker 會產生以下變異：
  // 1. basePrice * (1 - discount) → basePrice * (1 + discount)  ← 你的測試抓得到嗎？
  // 2. price * (1 + taxRate) → price * (1 - taxRate)            ← 你的測試抓得到嗎？
  // 3. price < 0 → price <= 0                                   ← 邊界條件有測到嗎？
  const discounted = basePrice * (1 - discount)
  const price = discounted * (1 + taxRate)
  return price < 0 ? 0 : Math.round(price)
}

// tests/unit/utils/pricing.test.ts — 補強測試以殺死所有變異
describe('calculateFinalPrice', () => {
  it('正確計算折扣與稅', () => {
    // 殺死變異 1 和 2：驗證具體數值
    expect(calculateFinalPrice(1000, 0.2, 0.05)).toBe(840) // 1000 * 0.8 * 1.05
  })

  it('零折扣', () => {
    expect(calculateFinalPrice(500, 0, 0.1)).toBe(550) // 500 * 1.0 * 1.1
  })

  it('折扣超過 100% 時回傳 0', () => {
    // 殺死變異 3：測試邊界條件 price < 0
    expect(calculateFinalPrice(100, 1.5, 0.1)).toBe(0)
  })

  it('極小正數不觸發下限保護', () => {
    // 殺死 < 被改成 <= 的變異：price=0.5 時原始 0.5<0 為 false 回傳 1，變異 0.5<=0 為 false 也回傳 1 — 等價變異
    // 改用 discount=0.995 使 price=0.5，Math.round(0.5)=1 vs 下限 0 來驗證正數不被截斷
    expect(calculateFinalPrice(100, 0.995, 0)).toBe(1) // Math.round(0.5) = 1
  })
})
```

變異分數指標解讀：

| 分數範圍 | 評等 | 意義 |
|----------|------|------|
| 90-100%  | 優秀 | 測試套件對程式碼變更高度敏感 |
| 75-89%   | 良好 | 大部分邏輯有被覆蓋，部分邊界可加強 |
| 60-74%   | 待改善 | 有明顯的測試盲區，需補強 |
| < 60%    | 不合格 | 測試套件品質不足，存在大量未覆蓋邏輯 |

### CI 變異覆蓋率門檻

```yaml
# .github/workflows/mutation.yml
name: Mutation Testing
on:
  pull_request:
    paths:
      - 'composables/**'
      - 'utils/**'
      - 'server/utils/**'
      - 'shared/**'
jobs:
  mutation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - name: 執行增量變異測試
        run: pnpm test:mutation:incremental
      - name: 檢查變異分數門檻
        run: |
          SCORE=$(node -e "
            const r = require('./reports/mutation/mutation-report.json');
            const s = (r.files ? Object.values(r.files) : [])
              .reduce((a, f) => ({
                killed: a.killed + (f.mutants?.filter(m => m.status === 'Killed').length || 0),
                total: a.total + (f.mutants?.filter(m => m.status !== 'Ignored').length || 0),
              }), { killed: 0, total: 0 });
            console.log(s.total ? Math.round(s.killed / s.total * 100) : 100);
          ")
          echo "變異分數: ${SCORE}%"
          if [ "$SCORE" -lt 80 ]; then
            echo "::error::變異分數 ${SCORE}% 低於門檻 80%"
            exit 1
          fi
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: mutation-report
          path: reports/mutation/
          retention-days: 7
```

---

## Test Data Factories

測試資料工廠（Test Data Factory）提供一致、可重複、型別安全的測試資料產生機制，取代散落在各測試檔中的硬編碼資料。

### fishery 工廠模式實作

```typescript
// tests/factories/index.ts
import { Factory } from 'fishery'
import { faker } from '@faker-js/faker/locale/zh_TW'
import type { User, Post, Comment } from '~/shared/schemas'

// --- User Factory ---
export const userFactory = Factory.define<User>(({ sequence }) => ({
  id: sequence,
  name: faker.person.fullName(),
  email: faker.internet.email(),
  role: faker.helpers.arrayElement(['admin', 'user', 'moderator'] as const),
  avatar: faker.helpers.maybe(() => faker.image.avatar(), { probability: 0.7 }) ?? null,
  createdAt: faker.date.past({ years: 2 }),
  updatedAt: faker.date.recent({ days: 30 }),
}))

// --- Post Factory ---
export const postFactory = Factory.define<Post>(({ sequence, associations }) => ({
  id: sequence,
  title: faker.lorem.sentence({ min: 3, max: 10 }),
  content: faker.lorem.paragraphs({ min: 2, max: 5 }),
  slug: faker.helpers.slugify(faker.lorem.words(3)).toLowerCase(),
  status: faker.helpers.arrayElement(['draft', 'published', 'archived'] as const),
  authorId: associations.authorId ?? userFactory.build().id,
  tags: faker.helpers.arrayElements(
    ['vue', 'nuxt', 'typescript', 'testing', 'devops'],
    { min: 1, max: 3 },
  ),
  publishedAt: faker.helpers.maybe(() => faker.date.past(), { probability: 0.6 }) ?? null,
  createdAt: faker.date.past({ years: 1 }),
  updatedAt: faker.date.recent({ days: 14 }),
}))

// --- Comment Factory ---
export const commentFactory = Factory.define<Comment>(({ sequence, associations }) => ({
  id: sequence,
  body: faker.lorem.paragraph(),
  postId: associations.postId ?? postFactory.build().id,
  authorId: associations.authorId ?? userFactory.build().id,
  parentId: null,
  createdAt: faker.date.recent({ days: 7 }),
}))
```

### faker.js 產生逼真測試資料

```typescript
// tests/factories/helpers.ts
import { faker } from '@faker-js/faker/locale/zh_TW'
import type { CreateUserInput, UserQuery } from '~/shared/schemas/user'

/**
 * 產生合法的使用者建立請求
 */
export function buildCreateUserInput(
  overrides?: Partial<CreateUserInput>,
): CreateUserInput {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: 'user',
    avatar: null,
    // 確保密碼符合 schema 要求（含大寫字母與數字）
    password: `${faker.internet.password({ length: 8 })}A1`,
    ...overrides,
  }
}

/**
 * 產生 API 查詢參數
 */
export function buildUserQuery(overrides?: Partial<UserQuery>): UserQuery {
  return {
    page: faker.number.int({ min: 1, max: 10 }),
    perPage: faker.helpers.arrayElement([10, 20, 50]),
    sort: faker.helpers.arrayElement(['name', 'createdAt', 'email'] as const),
    order: faker.helpers.arrayElement(['asc', 'desc'] as const),
    ...overrides,
  }
}

/**
 * 批次產生測試資料
 */
export function buildUserList(count: number = 10) {
  return userFactory.buildList(count)
}
```

### Factory 使用範例：User、Post、Comment 模型

```typescript
// tests/unit/composables/useUsers.test.ts
import { userFactory, postFactory, commentFactory } from '~/tests/factories'
import { buildCreateUserInput } from '~/tests/factories/helpers'

describe('useUsers composable', () => {
  it('根據角色篩選使用者', () => {
    const users = [
      ...userFactory.buildList(3, { role: 'admin' }),
      ...userFactory.buildList(5, { role: 'user' }),
    ]
    const admins = users.filter((u) => u.role === 'admin')
    expect(admins).toHaveLength(3)
  })

  it('使用 transient params 建立關聯資料', () => {
    const author = userFactory.build({ name: '測試作者' })
    const posts = postFactory.buildList(3, { authorId: author.id })
    const comments = posts.flatMap((post) =>
      commentFactory.buildList(2, { postId: post.id, authorId: author.id }),
    )
    expect(posts.every((p) => p.authorId === author.id)).toBe(true)
    expect(comments).toHaveLength(6)
  })

  it('覆寫特定欄位建立邊界案例', () => {
    // 名稱含特殊字元
    const specialUser = userFactory.build({ name: "O'Brien <script>alert(1)</script>" })
    expect(specialUser.name).toContain("O'Brien")

    // 建立 input 並驗證 schema
    const input = buildCreateUserInput({ email: 'edge-case@test.com' })
    expect(input.email).toBe('edge-case@test.com')
  })
})
```

### 整合測試的資料庫填充策略

```typescript
// tests/helpers/seed.ts — 測試資料庫填充
import { faker } from '@faker-js/faker/locale/zh_TW'
import { userFactory, postFactory, commentFactory } from '~/tests/factories'
import type { PrismaClient } from '@prisma/client'

/**
 * 填充測試資料庫：建立完整的關聯資料圖
 */
export async function seedTestDatabase(prisma: PrismaClient) {
  // 清除既有資料（順序很重要：先刪子表）
  await prisma.comment.deleteMany()
  await prisma.post.deleteMany()
  await prisma.user.deleteMany()

  // 建立使用者
  const adminData = userFactory.build({ role: 'admin', email: 'admin@test.com' })
  const usersData = userFactory.buildList(5, { role: 'user' })

  const admin = await prisma.user.create({ data: adminData })
  const users = await Promise.all(
    usersData.map((u) => prisma.user.create({ data: u })),
  )

  // 建立文章（每位使用者 2 篇）
  const allUsers = [admin, ...users]
  const posts = await Promise.all(
    allUsers.flatMap((user) =>
      postFactory.buildList(2, { authorId: user.id }).map((p) =>
        prisma.post.create({ data: p }),
      ),
    ),
  )

  // 建立留言（每篇文章 3 則）
  await Promise.all(
    posts.flatMap((post) =>
      commentFactory.buildList(3, {
        postId: post.id,
        authorId: faker.helpers.arrayElement(allUsers).id,
      }).map((c) => prisma.comment.create({ data: c })),
    ),
  )

  return { admin, users, posts }
}

// tests/integration/api/users.test.ts — 使用 seed
import { setup, $fetch } from '@nuxt/test-utils/e2e'
import { seedTestDatabase } from '~/tests/helpers/seed'

describe('Users API Integration', async () => {
  await setup({ server: true })

  let seeded: Awaited<ReturnType<typeof seedTestDatabase>>

  beforeAll(async () => {
    const { PrismaClient } = await import('@prisma/client')
    const prisma = new PrismaClient()
    seeded = await seedTestDatabase(prisma)
    await prisma.$disconnect()
  })

  it('GET /api/users 回傳已填充的使用者', async () => {
    const res = await $fetch('/api/users')
    // admin + 5 位一般使用者 = 6
    expect(res.data.length).toBeGreaterThanOrEqual(6)
  })

  it('GET /api/users?role=admin 只回傳管理員', async () => {
    const res = await $fetch('/api/users?role=admin')
    expect(res.data.every((u: any) => u.role === 'admin')).toBe(true)
  })
})
```

---

## Network-Level Mocking (MSW)

MSW（Mock Service Worker）在網路層攔截 HTTP 請求，提供比 `vi.mock('$fetch')` 更真實的 API 模擬。它不修改應用程式碼，而是在 request/response 層面運作，使測試更貼近生產環境。

### MSW 安裝與 Nuxt 設定

```typescript
// tests/mocks/server.ts — MSW 伺服器設定
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

// 建立 MSW server（Node.js 環境，用於 Vitest）
export const mswServer = setupServer(...handlers)

// tests/setup/msw.ts — 全域 setup
import { mswServer } from '~/tests/mocks/server'
import { afterAll, afterEach, beforeAll } from 'vitest'

beforeAll(() => {
  mswServer.listen({
    // 遇到未處理的請求時警告（幫助找出遺漏的 mock）
    onUnhandledRequest: 'warn',
  })
})

afterEach(() => {
  // 每個測試後重置 handler（移除 test-specific overrides）
  mswServer.resetHandlers()
})

afterAll(() => {
  mswServer.close()
})
```

```typescript
// vitest.config.ts — 註冊 MSW setup
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    setupFiles: ['./tests/setup/msw.ts'],
    // 確保 MSW 在 Node 環境正常運作
    environment: 'nuxt',
  },
})
```

### MSW Handlers 對應 API 路由

```typescript
// tests/mocks/handlers.ts
import { http, HttpResponse, delay } from 'msw'
import { userFactory, postFactory } from '~/tests/factories'

const API_BASE = 'http://localhost:3000'

export const handlers = [
  // GET /api/users — 列表
  http.get(`${API_BASE}/api/users`, async ({ request }) => {
    const url = new URL(request.url)
    const page = Number(url.searchParams.get('page') || 1)
    const perPage = Number(url.searchParams.get('perPage') || 20)
    const role = url.searchParams.get('role')

    let users = userFactory.buildList(50)
    if (role) {
      users = users.filter((u) => u.role === role)
    }
    const start = (page - 1) * perPage
    const paginated = users.slice(start, start + perPage)

    return HttpResponse.json({
      data: paginated,
      meta: {
        total: users.length,
        page,
        perPage,
        totalPages: Math.ceil(users.length / perPage),
      },
    })
  }),

  // GET /api/users/:id — 單筆
  http.get(`${API_BASE}/api/users/:id`, async ({ params }) => {
    const id = Number(params.id)
    if (id === 999) {
      return HttpResponse.json(
        { statusCode: 404, code: 'NOT_FOUND', message: '找不到使用者' },
        { status: 404 },
      )
    }
    return HttpResponse.json({
      data: userFactory.build({ id }),
    })
  }),

  // POST /api/users — 建立
  http.post(`${API_BASE}/api/users`, async ({ request }) => {
    const body = (await request.json()) as Record<string, unknown>
    const newUser = userFactory.build({
      name: body.name as string,
      email: body.email as string,
      role: (body.role as 'admin' | 'user' | 'moderator') ?? 'user',
    })
    return HttpResponse.json({ data: newUser }, { status: 201 })
  }),

  // DELETE /api/users/:id — 刪除
  http.delete(`${API_BASE}/api/users/:id`, async () => {
    return HttpResponse.json({ success: true })
  }),

  // 模擬慢回應
  http.get(`${API_BASE}/api/slow-endpoint`, async () => {
    await delay(3000)
    return HttpResponse.json({ data: 'slow response' })
  }),
]
```

### 整合測試範例：使用 MSW

```typescript
// tests/integration/components/UserList.msw.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { mswServer } from '~/tests/mocks/server'
import { http, HttpResponse, delay } from 'msw'
import UserList from '~/components/feature/UserList.vue'
import { userFactory } from '~/tests/factories'

describe('UserList（MSW 整合測試）', () => {
  it('成功載入並渲染使用者列表', async () => {
    const wrapper = await mountSuspended(UserList)
    // MSW 會自動攔截 $fetch('/api/users') 並回傳 factory 資料
    await vi.waitFor(() => {
      expect(wrapper.findAll('[data-testid="user-item"]').length).toBeGreaterThan(0)
    })
  })

  it('處理伺服器錯誤', async () => {
    // 為這個測試覆寫 handler
    mswServer.use(
      http.get('http://localhost:3000/api/users', () => {
        return HttpResponse.json(
          { statusCode: 500, message: '內部伺服器錯誤' },
          { status: 500 },
        )
      }),
    )
    const wrapper = await mountSuspended(UserList)
    await vi.waitFor(() => {
      expect(wrapper.text()).toContain('伺服器錯誤')
    })
  })

  it('處理網路超時', async () => {
    mswServer.use(
      http.get('http://localhost:3000/api/users', async () => {
        // 模擬超時：延遲超過應用設定的 timeout
        await delay('infinite')
        return HttpResponse.json({})
      }),
    )
    const wrapper = await mountSuspended(UserList)
    await vi.waitFor(() => {
      expect(wrapper.find('[data-testid="loading-spinner"]').exists()).toBe(true)
    })
  })

  it('空列表顯示提示', async () => {
    mswServer.use(
      http.get('http://localhost:3000/api/users', () => {
        return HttpResponse.json({
          data: [],
          meta: { total: 0, page: 1, perPage: 20, totalPages: 0 },
        })
      }),
    )
    const wrapper = await mountSuspended(UserList)
    await vi.waitFor(() => {
      expect(wrapper.text()).toContain('沒有找到使用者')
    })
  })
})
```

### MSW 與 registerEndpoint 比較

| 特性 | MSW | `registerEndpoint` (@nuxt/test-utils) |
|------|-----|---------------------------------------|
| 攔截層級 | 網路層（Service Worker / Node interceptor） | Nuxt server handler 層 |
| 適用場景 | 元件整合測試、外部 API 模擬 | Nuxt API route 單元測試 |
| 真實度 | 高：完整 HTTP request/response 週期 | 中：繞過實際 HTTP |
| 設定複雜度 | 需額外安裝與設定 | 內建，零設定 |
| 可模擬項目 | headers、status、delay、network error | response body、status |
| 外部服務模擬 | 適合（如第三方 API） | 不適合（僅限 Nuxt 內部路由） |
| 瀏覽器測試 | 支援（Service Worker 模式） | 不支援 |

```typescript
// registerEndpoint 範例（適合簡單的 API route 測試）
import { registerEndpoint } from '@nuxt/test-utils/runtime'

registerEndpoint('/api/users', {
  method: 'GET',
  handler: () => ({
    data: userFactory.buildList(5),
    meta: { total: 5, page: 1, perPage: 20, totalPages: 1 },
  }),
})

// MSW 範例（適合需要完整 HTTP 行為的整合測試）
// 見上方 handlers.ts — 可模擬延遲、錯誤碼、headers 等
```

---

## Property-Based Testing

屬性基測試（Property-Based Testing）不同於傳統的範例基測試（Example-Based Testing）：不寫死輸入/輸出，而是定義「對所有合法輸入都必須成立的性質」，由框架自動產生大量隨機輸入來驗證。

### fast-check 整合

```typescript
// tests/setup/property.ts — fast-check 全域設定
import fc from 'fast-check'

// 設定預設參數
fc.configureGlobal({
  // 每次測試的隨機輸入數量
  numRuns: 100,
  // CI 中增加執行次數以提高信心
  ...(process.env.CI ? { numRuns: 500 } : {}),
  // 固定 seed 以利重現（可選）
  // seed: 42,
})
```

```typescript
// vitest.config.ts — 註冊 property test setup
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    setupFiles: ['./tests/setup/property.ts'],
  },
})
```

### 範例：測試 API 驗證器的任意輸入

```typescript
// tests/property/schemas/user-schema.property.test.ts
import fc from 'fast-check'
import { CreateUserSchema, UserQuerySchema } from '~/shared/schemas/user'

describe('CreateUserSchema 屬性基測試', () => {
  it('任何合法輸入皆能通過驗證', () => {
    // 定義符合 schema 的 Arbitrary
    const validUserArb = fc.record({
      name: fc.string({ minLength: 2, maxLength: 100 }).filter((s) => s.trim().length >= 2),
      email: fc.emailAddress(),
      role: fc.constantFrom('admin', 'user', 'moderator'),
      avatar: fc.option(fc.webUrl(), { nil: null }),
      // 密碼：至少 8 字元，含大寫與數字
      password: fc
        .tuple(
          fc.string({ minLength: 6, maxLength: 50 }),
          fc.constantFrom('A', 'B', 'C', 'X', 'Y', 'Z'),
          fc.constantFrom('0', '1', '2', '3', '7', '8', '9'),
        )
        .map(([base, upper, digit]) => `${base}${upper}${digit}`),
    })

    fc.assert(
      fc.property(validUserArb, (input) => {
        const result = CreateUserSchema.safeParse(input)
        // 性質：所有合法輸入都必須通過
        return result.success === true
      }),
    )
  })

  it('名稱過短必定被拒絕', () => {
    fc.assert(
      fc.property(
        fc.record({
          name: fc.string({ maxLength: 1 }), // 長度 0 或 1
          email: fc.emailAddress(),
          role: fc.constantFrom('admin', 'user', 'moderator'),
          avatar: fc.constant(null),
          password: fc.constant('ValidPass1'),
        }),
        (input) => {
          const result = CreateUserSchema.safeParse(input)
          // 性質：名稱少於 2 字元必定失敗
          return result.success === false
        },
      ),
    )
  })

  it('email 格式錯誤必定被拒絕', () => {
    // 產生「看起來像 email 但不合規」的字串
    const invalidEmailArb = fc.oneof(
      fc.string().filter((s) => !s.includes('@')),    // 無 @
      fc.string().map((s) => `${s}@`),                // @ 後無內容
      fc.string().map((s) => `@${s}`),                // @ 前無內容
    )

    fc.assert(
      fc.property(invalidEmailArb, (email) => {
        const result = CreateUserSchema.safeParse({
          name: '測試使用者',
          email,
          role: 'user',
          avatar: null,
          password: 'ValidPass1',
        })
        return result.success === false
      }),
    )
  })
})

describe('UserQuerySchema 屬性基測試', () => {
  it('page 與 perPage 經驗證後必為正整數', () => {
    const queryArb = fc.record({
      page: fc.oneof(fc.integer({ min: 1, max: 1000 }), fc.constant(undefined)),
      perPage: fc.oneof(fc.integer({ min: 1, max: 100 }), fc.constant(undefined)),
      sort: fc.oneof(
        fc.constantFrom('name', 'createdAt', 'email'),
        fc.constant(undefined),
      ),
      order: fc.oneof(fc.constantFrom('asc', 'desc'), fc.constant(undefined)),
    })

    fc.assert(
      fc.property(queryArb, (query) => {
        const result = UserQuerySchema.safeParse(query)
        if (!result.success) return true // 驗證失敗也是合法行為
        // 性質：通過驗證後 page >= 1 且 perPage 在 1~100
        return result.data.page >= 1 && result.data.perPage >= 1 && result.data.perPage <= 100
      }),
    )
  })
})
```

### 收縮與邊界案例發現（Shrinking）

fast-check 在發現失敗案例後會自動「收縮」（shrink）輸入，找到導致失敗的最小反例。

```typescript
// tests/property/utils/string-utils.property.test.ts
import fc from 'fast-check'
import { slugify, truncate, sanitizeHtml } from '~/utils/string'

describe('字串工具函式 — 屬性基測試', () => {
  describe('slugify', () => {
    it('輸出只包含合法 slug 字元', () => {
      fc.assert(
        fc.property(fc.string(), (input) => {
          const slug = slugify(input)
          // 性質：slug 只能包含小寫字母、數字、連字號
          return /^[a-z0-9-]*$/.test(slug)
        }),
      )
    })

    it('不產生連續連字號', () => {
      fc.assert(
        fc.property(fc.string(), (input) => {
          const slug = slugify(input)
          // 性質：不會出現 "--"
          return !slug.includes('--')
        }),
      )
    })

    it('冪等性：slugify 兩次結果相同', () => {
      fc.assert(
        fc.property(fc.string(), (input) => {
          // 性質：f(f(x)) === f(x)
          return slugify(slugify(input)) === slugify(input)
        }),
      )
    })
  })

  describe('truncate', () => {
    it('結果長度永遠不超過限制', () => {
      fc.assert(
        fc.property(
          fc.string(),
          fc.integer({ min: 1, max: 1000 }),
          (input, maxLength) => {
            const result = truncate(input, maxLength)
            // 性質：輸出長度 <= maxLength
            return result.length <= maxLength
          },
        ),
      )
    })

    it('短於限制的字串不被截斷', () => {
      fc.assert(
        fc.property(
          fc.string({ maxLength: 50 }),
          fc.integer({ min: 51, max: 200 }),
          (input, maxLength) => {
            // 性質：若 input.length < maxLength，則輸出 === 輸入
            return truncate(input, maxLength) === input
          },
        ),
      )
    })
  })

  describe('sanitizeHtml', () => {
    it('輸出絕不包含 script 標籤', () => {
      // 刻意混入惡意字串
      const maliciousArb = fc.oneof(
        fc.string(),
        fc.constant('<script>alert(1)</script>'),
        fc.string().map((s) => `<script>${s}</script>`),
        fc.string().map((s) => `<img onerror="${s}" src=x>`),
        fc.string().map((s) => `<div onmouseover="${s}">${s}</div>`),
      )

      fc.assert(
        fc.property(maliciousArb, (input) => {
          const sanitized = sanitizeHtml(input)
          // 性質：不含 script 標籤與事件處理屬性
          return (
            !sanitized.toLowerCase().includes('<script') &&
            !sanitized.toLowerCase().includes('onerror') &&
            !sanitized.toLowerCase().includes('onmouseover')
          )
        }),
      )
    })
  })
})
```

```typescript
// tests/property/composables/usePagination.property.test.ts
import fc from 'fast-check'
import { usePagination } from '~/composables/usePagination'

describe('usePagination 屬性基測試', () => {
  it('totalPages 計算永遠正確', () => {
    fc.assert(
      fc.property(
        fc.integer({ min: 0, max: 10000 }),   // total
        fc.integer({ min: 1, max: 100 }),      // perPage
        (total, perPage) => {
          const { totalPages } = usePagination({ total, perPage, currentPage: 1 })
          // 性質：totalPages === Math.ceil(total / perPage)
          return totalPages.value === Math.ceil(total / perPage)
        },
      ),
    )
  })

  it('currentPage 永遠在有效範圍內', () => {
    fc.assert(
      fc.property(
        fc.integer({ min: 0, max: 500 }),
        fc.integer({ min: 1, max: 50 }),
        fc.integer({ min: -10, max: 100 }),
        (total, perPage, requestedPage) => {
          const { currentPage, totalPages } = usePagination({
            total,
            perPage,
            currentPage: requestedPage,
          })
          // 性質：currentPage 在 1 到 totalPages 之間（或 total=0 時為 1）
          const maxPage = Math.max(totalPages.value, 1)
          return currentPage.value >= 1 && currentPage.value <= maxPage
        },
      ),
    )
  })
})

---

## Security Testing（安全性測試）

安全性測試驗證應用程式能抵禦常見攻擊向量（OWASP Top 10），確保認證、授權、輸入驗證與 HTTP 安全標頭皆正確實作。此節涵蓋 CSRF、XSS、認證/授權邊界、速率限制、SQL 注入防護與安全標頭測試。

### CSRF Token 驗證測試

```typescript
// tests/security/csrf.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('CSRF Protection', async () => {
  await setup({ server: true })

  it('rejects requests without CSRF token', async () => {
    const res = await $fetch.raw('/api/orders', {
      method: 'POST',
      body: { item: 'test' },
    }).catch(e => e.response)
    expect(res.status).toBe(403)
  })

  it('rejects requests with mismatched CSRF token', async () => {
    const res = await $fetch.raw('/api/orders', {
      method: 'POST',
      headers: { 'x-csrf-token': 'invalid-token' },
      body: { item: 'test' },
    }).catch(e => e.response)
    expect(res.status).toBe(403)
  })

  it('accepts requests with valid CSRF token', async () => {
    // Get valid token first
    const tokenRes = await $fetch('/api/auth/csrf')
    const res = await $fetch('/api/orders', {
      method: 'POST',
      headers: { 'x-csrf-token': tokenRes.token },
      body: { item: 'test' },
    })
    expect(res).toBeDefined()
  })

  it('rejects CSRF token reuse after consumption (one-time tokens)', async () => {
    const tokenRes = await $fetch('/api/auth/csrf')
    // First request succeeds
    await $fetch('/api/orders', {
      method: 'POST',
      headers: { 'x-csrf-token': tokenRes.token },
      body: { item: 'first' },
    })
    // Second request with same token should fail
    const res = await $fetch.raw('/api/orders', {
      method: 'POST',
      headers: { 'x-csrf-token': tokenRes.token },
      body: { item: 'replay' },
    }).catch(e => e.response)
    expect(res.status).toBe(403)
  })
})
```

### XSS 防護測試

```typescript
// tests/security/xss.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UserComment from '~/components/feature/UserComment.vue'

describe('XSS Prevention', async () => {
  await setup({ server: true })

  describe('API Input Sanitization', () => {
    it('sanitizes script tags in user input', async () => {
      const maliciousInput = '<script>alert("xss")</script>'
      const res = await $fetch('/api/comments', {
        method: 'POST',
        body: { content: maliciousInput },
      })
      expect(res.content).not.toContain('<script>')
    })

    it('sanitizes event handler attributes', async () => {
      const maliciousInput = '<img onerror=alert(1) src=x>'
      const res = await $fetch('/api/comments', {
        method: 'POST',
        body: { content: maliciousInput },
      })
      expect(res.content).not.toContain('onerror')
    })

    it('sanitizes javascript: protocol URIs', async () => {
      const maliciousInput = '<a href="javascript:alert(1)">click</a>'
      const res = await $fetch('/api/comments', {
        method: 'POST',
        body: { content: maliciousInput },
      })
      expect(res.content).not.toContain('javascript:')
    })

    it('preserves safe HTML while stripping dangerous content', async () => {
      const mixedInput = '<p>Hello</p><script>alert(1)</script><b>World</b>'
      const res = await $fetch('/api/comments', {
        method: 'POST',
        body: { content: mixedInput },
      })
      expect(res.content).toContain('<p>Hello</p>')
      expect(res.content).toContain('<b>World</b>')
      expect(res.content).not.toContain('<script>')
    })
  })

  describe('Component Output Encoding', () => {
    it('escapes HTML entities in rendered output', async () => {
      const wrapper = await mountSuspended(UserComment, {
        props: { content: '<img onerror=alert(1) src=x>' },
      })
      expect(wrapper.html()).not.toContain('onerror')
      // Vue 3 auto-escapes by default in template interpolation
      expect(wrapper.html()).toContain('&lt;img')
    })

    it('does not render v-html with unsanitized user content', async () => {
      const wrapper = await mountSuspended(UserComment, {
        props: { content: '<b onmouseover="alert(1)">hover me</b>' },
      })
      const html = wrapper.html()
      expect(html).not.toContain('onmouseover')
    })
  })
})
```

### 認證與授權邊界測試

```typescript
// tests/security/auth.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'
import jwt from 'jsonwebtoken'

const JWT_SECRET = process.env.JWT_SECRET || 'test-secret'

function createTestJWT(payload: Record<string, unknown>): string {
  return jwt.sign(payload, JWT_SECRET, { algorithm: 'HS256' })
}

describe('Authentication Security', async () => {
  await setup({ server: true })

  describe('Token Validation', () => {
    it('rejects expired JWT tokens', async () => {
      const expiredToken = createTestJWT({
        sub: 'user-1',
        role: 'user',
        exp: Math.floor(Date.now() / 1000) - 3600, // 1 hour ago
      })
      const res = await $fetch.raw('/api/protected', {
        headers: { authorization: `Bearer ${expiredToken}` },
      }).catch(e => e.response)
      expect(res.status).toBe(401)
    })

    it('rejects tokens with invalid signature', async () => {
      const forgedToken = jwt.sign(
        { sub: 'user-1', role: 'admin' },
        'wrong-secret',
        { algorithm: 'HS256' },
      )
      const res = await $fetch.raw('/api/protected', {
        headers: { authorization: `Bearer ${forgedToken}` },
      }).catch(e => e.response)
      expect(res.status).toBe(401)
    })

    it('rejects tokens with algorithm confusion (none attack)', async () => {
      // Manually craft a token with "alg": "none"
      const header = Buffer.from(JSON.stringify({ alg: 'none', typ: 'JWT' })).toString('base64url')
      const payload = Buffer.from(JSON.stringify({ sub: 'user-1', role: 'admin' })).toString('base64url')
      const noneToken = `${header}.${payload}.`

      const res = await $fetch.raw('/api/protected', {
        headers: { authorization: `Bearer ${noneToken}` },
      }).catch(e => e.response)
      expect(res.status).toBe(401)
    })

    it('rejects malformed authorization headers', async () => {
      const testCases = [
        'InvalidPrefix token123',
        'Bearer',          // missing token
        'Bearer  ',        // whitespace only
        'bearer valid',    // wrong case (if strict)
      ]
      for (const header of testCases) {
        const res = await $fetch.raw('/api/protected', {
          headers: { authorization: header },
        }).catch(e => e.response)
        expect(res.status).toBe(401)
      }
    })
  })

  describe('Authorization & Privilege Escalation', () => {
    it('prevents privilege escalation — user cannot access admin routes', async () => {
      const userToken = createTestJWT({
        sub: 'user-1',
        role: 'user',
        exp: Math.floor(Date.now() / 1000) + 3600,
      })
      const res = await $fetch.raw('/api/admin/users', {
        headers: { authorization: `Bearer ${userToken}` },
      }).catch(e => e.response)
      expect(res.status).toBe(403)
    })

    it('prevents horizontal privilege escalation — user cannot access other user data', async () => {
      const userToken = createTestJWT({
        sub: 'user-1',
        role: 'user',
        exp: Math.floor(Date.now() / 1000) + 3600,
      })
      const res = await $fetch.raw('/api/users/user-2/settings', {
        headers: { authorization: `Bearer ${userToken}` },
      }).catch(e => e.response)
      expect(res.status).toBe(403)
    })

    it('prevents role tampering via request body', async () => {
      const userToken = createTestJWT({
        sub: 'user-1',
        role: 'user',
        exp: Math.floor(Date.now() / 1000) + 3600,
      })
      const res = await $fetch.raw('/api/users/user-1', {
        method: 'PUT',
        headers: { authorization: `Bearer ${userToken}` },
        body: { role: 'admin' }, // attempt to self-promote
      }).catch(e => e.response)
      // Should either ignore the role field or return 403
      expect([200, 403]).toContain(res.status)
      if (res.status === 200) {
        const updated = await $fetch('/api/users/user-1', {
          headers: { authorization: `Bearer ${userToken}` },
        })
        expect(updated.data.role).toBe('user') // role unchanged
      }
    })
  })
})
```

### 速率限制測試

```typescript
// tests/security/rate-limiting.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('Rate Limiting', async () => {
  await setup({ server: true })

  it('allows requests within rate limit', async () => {
    const res = await $fetch.raw('/api/auth/login', {
      method: 'POST',
      body: { email: 'user@test.com', password: 'WrongPass1' },
    }).catch(e => e.response)
    // Should get normal response (401 for wrong password, not 429)
    expect(res.status).not.toBe(429)
  })

  it('returns 429 after exceeding rate limit', async () => {
    const maxAttempts = 10 // match your rate limit config
    const results: number[] = []

    for (let i = 0; i < maxAttempts + 5; i++) {
      const res = await $fetch.raw('/api/auth/login', {
        method: 'POST',
        body: { email: 'brute@test.com', password: 'WrongPass1' },
      }).catch(e => e.response)
      results.push(res.status)
    }

    // At some point, we should start seeing 429
    expect(results).toContain(429)
  })

  it('includes Retry-After header when rate limited', async () => {
    // Exhaust rate limit first
    for (let i = 0; i < 15; i++) {
      await $fetch.raw('/api/auth/login', {
        method: 'POST',
        body: { email: 'retry@test.com', password: 'WrongPass1' },
      }).catch(() => {})
    }

    const res = await $fetch.raw('/api/auth/login', {
      method: 'POST',
      body: { email: 'retry@test.com', password: 'WrongPass1' },
    }).catch(e => e.response)

    if (res.status === 429) {
      expect(res.headers.get('retry-after')).toBeDefined()
    }
  })
})
```

### SQL 注入防護測試

```typescript
// tests/security/sql-injection.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('SQL Injection Prevention', async () => {
  await setup({ server: true })

  const sqlPayloads = [
    "'; DROP TABLE users; --",
    "1' OR '1'='1",
    "1; SELECT * FROM users --",
    "' UNION SELECT password FROM users --",
    "1' AND SLEEP(5) --",
  ]

  for (const payload of sqlPayloads) {
    it(`rejects SQL injection payload: ${payload.slice(0, 30)}...`, async () => {
      // Test via query parameter
      const queryRes = await $fetch.raw(`/api/users?search=${encodeURIComponent(payload)}`)
        .catch(e => e.response)
      // Should either return empty results or 400, never leak data
      expect([200, 400]).toContain(queryRes.status)
      if (queryRes.status === 200) {
        // Ensure the payload didn't cause data leakage
        const body = queryRes._data
        expect(JSON.stringify(body)).not.toContain('password')
      }

      // Test via path parameter
      const pathRes = await $fetch.raw(`/api/users/${encodeURIComponent(payload)}`)
        .catch(e => e.response)
      expect([400, 404]).toContain(pathRes.status)
    })
  }

  it('parameterized queries prevent injection in POST body', async () => {
    const res = await $fetch('/api/users', {
      method: 'POST',
      body: {
        name: "Robert'); DROP TABLE users;--",
        email: 'bobby@tables.com',
        role: 'user',
        avatar: null,
        password: 'SafePass123',
      },
    }).catch(e => e)
    // Should succeed (the name is just a string) or fail validation — not crash
    if (!('statusCode' in res)) {
      expect(res.data.name).toBe("Robert'); DROP TABLE users;--")
    }
  })
})
```

### HTTP 安全標頭測試

```typescript
// tests/security/headers.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('Security Headers', async () => {
  await setup({ server: true })

  let headers: Headers

  beforeAll(async () => {
    const res = await $fetch.raw('/')
    headers = res.headers
  })

  it('sets Content-Security-Policy header', () => {
    const csp = headers.get('content-security-policy')
    expect(csp).toBeDefined()
    // Verify essential directives
    expect(csp).toContain("default-src")
    expect(csp).toContain("script-src")
    // Ensure no unsafe-inline for scripts (unless nonce-based)
    if (!csp!.includes('nonce-')) {
      expect(csp).not.toContain("'unsafe-inline'")
    }
  })

  it('sets Strict-Transport-Security header', () => {
    const hsts = headers.get('strict-transport-security')
    expect(hsts).toBeDefined()
    expect(hsts).toContain('max-age=')
    // Recommended: at least 1 year (31536000 seconds)
    const maxAge = parseInt(hsts!.match(/max-age=(\d+)/)?.[1] || '0')
    expect(maxAge).toBeGreaterThanOrEqual(31536000)
  })

  it('sets X-Frame-Options header to prevent clickjacking', () => {
    const xfo = headers.get('x-frame-options')
    expect(xfo).toBeDefined()
    expect(['DENY', 'SAMEORIGIN']).toContain(xfo!.toUpperCase())
  })

  it('sets X-Content-Type-Options to prevent MIME sniffing', () => {
    expect(headers.get('x-content-type-options')).toBe('nosniff')
  })

  it('sets Referrer-Policy header', () => {
    const rp = headers.get('referrer-policy')
    expect(rp).toBeDefined()
    const safeValues = [
      'no-referrer',
      'same-origin',
      'strict-origin',
      'strict-origin-when-cross-origin',
    ]
    expect(safeValues).toContain(rp)
  })

  it('sets X-XSS-Protection header (legacy browser support)', () => {
    const xxp = headers.get('x-xss-protection')
    // Either "0" (let CSP handle it) or "1; mode=block"
    if (xxp) {
      expect(['0', '1; mode=block']).toContain(xxp)
    }
  })

  it('sets Permissions-Policy to restrict browser features', () => {
    const pp = headers.get('permissions-policy')
    if (pp) {
      // Ensure camera and microphone are restricted
      expect(pp).toMatch(/camera=\(\)/)
      expect(pp).toMatch(/microphone=\(\)/)
    }
  })

  it('API responses include proper CORS headers', async () => {
    const res = await $fetch.raw('/api/users', {
      method: 'OPTIONS',
      headers: { origin: 'https://evil.com' },
    }).catch(e => e.response)
    // Should not reflect arbitrary origins
    const allowOrigin = res.headers.get('access-control-allow-origin')
    if (allowOrigin) {
      expect(allowOrigin).not.toBe('https://evil.com')
    }
  })
})
```

### Nuxt Security Module 整合

搭配 [`nuxt-security`](https://nuxt-security.vercel.app/) 模組可自動套用大部分安全標頭。以上測試可驗證模組設定是否正確生效。

```typescript
// nuxt.config.ts — nuxt-security 設定範例
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    headers: {
      contentSecurityPolicy: {
        'default-src': ["'self'"],
        'script-src': ["'self'", "'nonce-{{nonce}}'"],
        'style-src': ["'self'", "'unsafe-inline'"],
        'img-src': ["'self'", 'data:', 'https:'],
      },
      strictTransportSecurity: {
        maxAge: 31536000,
        includeSubdomains: true,
        preload: true,
      },
      xFrameOptions: 'SAMEORIGIN',
      xContentTypeOptions: 'nosniff',
      referrerPolicy: 'strict-origin-when-cross-origin',
      permissionsPolicy: {
        camera: [],
        microphone: [],
        geolocation: [],
      },
    },
    rateLimiter: {
      tokensPerInterval: 60,
      interval: 60000, // 1 minute
    },
    csrf: true,
  },
})
```

---

## Snapshot Testing（快照測試）

快照測試（Snapshot Testing）將元件輸出或 API 回應結構序列化儲存，在後續測試中自動比對，捕捉非預期的結構變更。正確使用快照測試能高效防止回歸；但濫用會導致脆弱、難以維護的測試。

### 何時使用快照 vs 明確斷言

| 場景 | 推薦方式 | 原因 |
|------|---------|------|
| 元件渲染結構穩定、少變動 | 快照測試 | 快速捕捉非預期的結構變更 |
| 驗證特定值或業務邏輯 | 明確斷言 | 意圖清晰、失敗訊息有意義 |
| API 回應結構契約 | 快照 + 動態欄位匹配器 | 確保結構穩定，忽略動態值 |
| CSS class 或 style 頻繁變動 | 明確斷言 | 避免因樣式調整頻繁更新快照 |
| 大型元件樹（含子元件） | 避免快照 | 快照過大、頻繁破壞、難以 review |
| 純函式輸出 | 明確斷言 | 直接驗證回傳值更清晰 |

### Inline Snapshots vs File Snapshots

```typescript
// Inline snapshot — 適合小型、穩定的輸出
// 優點：不需切換檔案即可看到預期結果
// 缺點：大型輸出會讓測試檔可讀性下降
it('renders badge with correct structure', () => {
  const wrapper = mount(RoleBadge, {
    props: { role: 'admin' },
  })
  expect(wrapper.html()).toMatchInlineSnapshot(`
    "<span class="badge badge-admin">admin</span>"
  `)
})

// File snapshot — 適合中型輸出（由 Vitest 自動管理 __snapshots__ 目錄）
// 優點：測試檔保持簡潔
// 缺點：需打開快照檔才能看到預期結果
it('renders full user profile', () => {
  const wrapper = mount(UserProfile, {
    props: { user: createTestUser({ name: 'Alice', role: 'admin' }) },
  })
  expect(wrapper.html()).toMatchSnapshot()
})
```

### 快照最佳實踐

1. **保持快照小而聚焦** — 快照單一元件或區塊，而非整頁
2. **避免包含動態資料** — 使用 `expect.any()` 或 `toMatchSnapshot()` 的屬性匹配器過濾時間戳、ID 等
3. **快照必須可 review** — 如果 reviewer 無法在 PR 中有意義地審查快照 diff，快照就太大了
4. **命名快照** — 使用描述性測試名稱，讓快照檔案中的條目可被理解
5. **定期清理** — 執行 `vitest --update` 後檢視 diff，確認每個變更都是預期的

### 元件快照測試

```typescript
// tests/unit/components/snapshots/UserProfile.snapshot.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UserProfile from '~/components/feature/UserProfile.vue'
import { userFactory } from '~/tests/factories'

describe('UserProfile Snapshots', () => {
  it('renders user profile correctly (inline)', async () => {
    const wrapper = await mountSuspended(UserProfile, {
      props: {
        user: userFactory.build({
          id: 1,
          name: 'Alice',
          role: 'admin',
          avatar: 'https://example.com/alice.jpg',
          email: 'alice@example.com',
        }),
      },
    })
    expect(wrapper.html()).toMatchInlineSnapshot(`
      "<div class="user-profile">
        <img src="https://example.com/alice.jpg" alt="Alice" class="avatar" />
        <h2>Alice</h2>
        <span class="badge badge-admin">admin</span>
        <p>alice@example.com</p>
      </div>"
    `)
  })

  it('renders empty state when no avatar', async () => {
    const wrapper = await mountSuspended(UserProfile, {
      props: {
        user: userFactory.build({ name: 'Bob', avatar: null }),
      },
    })
    // Focus on the avatar area only — keep snapshot small
    const avatarArea = wrapper.find('.avatar-container')
    expect(avatarArea.html()).toMatchInlineSnapshot(`
      "<div class="avatar-container">
        <div class="avatar-placeholder">B</div>
      </div>"
    `)
  })

  it('matches snapshot for each role variant', async () => {
    const roles = ['admin', 'user', 'moderator'] as const
    for (const role of roles) {
      const wrapper = await mountSuspended(UserProfile, {
        props: { user: userFactory.build({ name: 'Test', role }) },
      })
      // File snapshot with role in name for clarity
      expect(wrapper.find('.badge').html()).toMatchSnapshot(`role-badge-${role}`)
    }
  })
})
```

### API 回應結構快照測試

```typescript
// tests/integration/api/snapshots/orders.snapshot.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('Order API Response Snapshots', async () => {
  await setup({ server: true })

  it('returns expected order structure', async () => {
    const order = await $fetch('/api/orders/123')
    // Use property matchers to ignore dynamic fields
    expect(order).toMatchSnapshot({
      data: {
        id: expect.any(String),
        createdAt: expect.any(String),
        updatedAt: expect.any(String),
        orderNumber: expect.stringMatching(/^ORD-\d+$/),
        items: expect.arrayContaining([
          expect.objectContaining({
            productId: expect.any(String),
            quantity: expect.any(Number),
            unitPrice: expect.any(Number),
          }),
        ]),
        total: expect.any(Number),
        status: expect.stringMatching(/^(pending|confirmed|shipped|delivered)$/),
      },
    })
  })

  it('list endpoint returns stable pagination structure', async () => {
    const res = await $fetch('/api/orders?page=1&perPage=5')
    // Only snapshot the meta structure, not the data
    expect(res.meta).toMatchObject({
      page: 1,
      perPage: 5,
      total: expect.any(Number),
      totalPages: expect.any(Number),
    })
  })

  it('error response has consistent shape', async () => {
    const err = await $fetch('/api/orders/nonexistent').catch(e => e.data)
    expect(err).toMatchSnapshot({
      statusCode: 404,
      code: 'NOT_FOUND',
      message: expect.any(String),
    })
  })
})
```

### 快照更新工作流

```bash
# 查看哪些快照過期（不更新）
npx vitest run --reporter=verbose 2>&1 | grep "Snapshot"

# 更新所有快照（務必在 PR 中仔細 review diff）
npx vitest run --update

# 僅更新特定測試檔的快照
npx vitest run tests/unit/components/snapshots/ --update

# CI 中禁止自動更新 — 過期快照應導致失敗
# vitest.config.ts
# test: { snapshotSerializers: [], ... }
```

```yaml
# .github/workflows/test.yml — 快照 diff 檢查
- name: Check for uncommitted snapshot changes
  run: |
    pnpm vitest run
    if ! git diff --exit-code -- '**/__snapshots__/**'; then
      echo "::error::Snapshot files are out of date. Run 'vitest --update' and commit the changes."
      exit 1
    fi
```

---

## Infrastructure-Level Chaos Testing（基礎設施層級混沌測試）

基礎設施層級混沌測試超越單純的 mock，模擬真實的基礎設施故障場景（網路中斷、資料庫不可用、上游服務故障），驗證應用程式的韌性（resilience）與優雅降級（graceful degradation）能力。

### 網路故障模擬

```typescript
// tests/chaos/network-resilience.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'
import { http, HttpResponse, delay } from 'msw'
import { mswServer } from '~/tests/mocks/server'

describe('Network Failure Resilience', async () => {
  await setup({ server: true })

  it('handles connection timeout with user-friendly error', async () => {
    mswServer.use(
      http.get('http://upstream-api.example.com/data', async () => {
        await delay(30000) // exceed timeout
        return HttpResponse.json({})
      }),
    )

    const res = await $fetch.raw('/api/aggregated-data').catch(e => e.response)
    expect(res.status).toBe(504)
    expect(res._data).toMatchObject({
      message: expect.stringContaining('timeout'),
    })
  })

  it('retries transient network failures with exponential backoff', async () => {
    let callCount = 0
    mswServer.use(
      http.get('http://upstream-api.example.com/data', () => {
        callCount++
        if (callCount <= 2) {
          return HttpResponse.error() // network error
        }
        return HttpResponse.json({ data: 'success' })
      }),
    )

    const res = await $fetch('/api/aggregated-data')
    expect(res.data).toBe('success')
    expect(callCount).toBe(3) // 2 failures + 1 success
  })

  it('handles DNS resolution failure gracefully', async () => {
    mswServer.use(
      http.get('http://upstream-api.example.com/data', () => {
        return HttpResponse.error() // simulate DNS failure
      }),
    )

    const res = await $fetch.raw('/api/aggregated-data').catch(e => e.response)
    expect([502, 503]).toContain(res.status)
  })
})
```

### 服務依賴故障測試

```typescript
// tests/chaos/service-dependency.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('Service Dependency Failures', async () => {
  await setup({ server: true })

  it('handles database connection failure gracefully', async () => {
    // Simulate DB connection failure via environment or test double
    // In integration tests, you can stop the DB container or use a mock
    const mockDb = vi.spyOn(await import('~/server/utils/db'), 'db', 'get')
    mockDb.mockImplementation(() => {
      throw Object.assign(new Error('Connection refused'), { code: 'ECONNREFUSED' })
    })

    const res = await $fetch.raw('/api/products').catch(e => e.response)
    expect(res.status).toBe(503)
    expect(res._data).toMatchObject({
      message: expect.stringContaining('temporarily unavailable'),
      retryAfter: expect.any(Number),
    })

    mockDb.mockRestore()
  })

  it('handles Redis/cache unavailability without crashing', async () => {
    const mockCache = vi.spyOn(await import('~/server/utils/cache'), 'cache', 'get')
    mockCache.mockImplementation(() => {
      throw new Error('ECONNREFUSED: Redis unavailable')
    })

    // App should still function, just without caching
    const res = await $fetch('/api/products')
    expect(res.data).toBeDefined()
    expect(res.data.length).toBeGreaterThan(0)

    mockCache.mockRestore()
  })

  it('returns 503 with structured error when all retries exhausted', async () => {
    const mockFetch = vi.spyOn(globalThis, 'fetch')
    mockFetch.mockRejectedValue(new Error('ECONNRESET'))

    const res = await $fetch.raw('/api/external-service').catch(e => e.response)
    expect(res.status).toBe(503)
    expect(res._data).toMatchObject({
      statusCode: 503,
      code: 'SERVICE_UNAVAILABLE',
      message: expect.any(String),
    })

    mockFetch.mockRestore()
  })
})
```

### Circuit Breaker 測試

```typescript
// tests/chaos/circuit-breaker.test.ts
import { CircuitBreaker } from '~/server/utils/circuit-breaker'

describe('Circuit Breaker', () => {
  it('starts in closed state', () => {
    const cb = new CircuitBreaker({
      failureThreshold: 3,
      resetTimeout: 5000,
    })
    expect(cb.state).toBe('closed')
  })

  it('opens after reaching failure threshold', async () => {
    const cb = new CircuitBreaker({
      failureThreshold: 3,
      resetTimeout: 5000,
    })

    // Trigger failures up to threshold
    for (let i = 0; i < 3; i++) {
      await cb.execute(() => Promise.reject(new Error('service down'))).catch(() => {})
    }

    expect(cb.state).toBe('open')
  })

  it('fails fast when circuit is open (does not call service)', async () => {
    const cb = new CircuitBreaker({
      failureThreshold: 3,
      resetTimeout: 5000,
    })

    // Open the circuit
    for (let i = 0; i < 3; i++) {
      await cb.execute(() => Promise.reject(new Error('fail'))).catch(() => {})
    }

    const serviceFn = vi.fn().mockResolvedValue('ok')
    await expect(cb.execute(serviceFn)).rejects.toThrow('Circuit breaker is open')
    // Service function should NOT have been called
    expect(serviceFn).not.toHaveBeenCalled()
  })

  it('transitions to half-open after reset timeout', async () => {
    vi.useFakeTimers()

    const cb = new CircuitBreaker({
      failureThreshold: 3,
      resetTimeout: 5000,
    })

    // Open the circuit
    for (let i = 0; i < 3; i++) {
      await cb.execute(() => Promise.reject(new Error('fail'))).catch(() => {})
    }
    expect(cb.state).toBe('open')

    // Advance past reset timeout
    vi.advanceTimersByTime(5001)

    expect(cb.state).toBe('half-open')

    vi.useRealTimers()
  })

  it('closes again after successful call in half-open state', async () => {
    vi.useFakeTimers()

    const cb = new CircuitBreaker({
      failureThreshold: 3,
      resetTimeout: 5000,
    })

    // Open the circuit
    for (let i = 0; i < 3; i++) {
      await cb.execute(() => Promise.reject(new Error('fail'))).catch(() => {})
    }

    // Move to half-open
    vi.advanceTimersByTime(5001)

    // Successful call should close the circuit
    await cb.execute(() => Promise.resolve('recovered'))
    expect(cb.state).toBe('closed')

    vi.useRealTimers()
  })
})
```

### 優雅降級驗證

```typescript
// tests/chaos/graceful-degradation.test.ts
import { setup, $fetch } from '@nuxt/test-utils/e2e'
import { mswServer } from '~/tests/mocks/server'
import { http, HttpResponse } from 'msw'

describe('Graceful Degradation', async () => {
  await setup({ server: true })

  it('returns cached data when upstream is unavailable', async () => {
    // Step 1: Warm the cache with a successful request
    const freshRes = await $fetch('/api/products')
    expect(freshRes.data.length).toBeGreaterThan(0)

    // Step 2: Kill upstream service
    mswServer.use(
      http.get('http://upstream-api.example.com/products', () => {
        return HttpResponse.error()
      }),
    )

    // Step 3: Should return stale cached data
    const staleRes = await $fetch('/api/products')
    expect(staleRes.data).toBeDefined()
    expect(staleRes.data.length).toBeGreaterThan(0)
    // Optionally: response may include a staleness indicator
    if (staleRes.meta?.stale !== undefined) {
      expect(staleRes.meta.stale).toBe(true)
    }
  })

  it('health check endpoint remains available during partial outage', async () => {
    // Simulate database failure
    mswServer.use(
      http.get('http://localhost:5432/*', () => HttpResponse.error()),
    )

    const res = await $fetch('/api/health')
    expect(res.status).toBe('degraded')
    expect(res.checks).toMatchObject({
      server: 'ok',
      database: 'error',
    })
  })

  it('static pages render even when API layer is down', async () => {
    // Kill all API endpoints
    mswServer.use(
      http.all('http://localhost:3000/api/*', () => HttpResponse.error()),
    )

    const html = await $fetch('/about')
    expect(html).toContain('</html>')
    expect(html).toContain('<main') // page shell still renders
  })

  it('feature flags disable broken features instead of crashing', async () => {
    // Simulate recommendations service failure
    mswServer.use(
      http.get('http://upstream-api.example.com/recommendations', () => {
        return HttpResponse.error()
      }),
    )

    const html = await $fetch('/products/1')
    // Page should render without recommendations section
    expect(html).toContain('</html>')
    expect(html).not.toContain('data-testid="recommendations-error"')
    // The recommendations section should be hidden, not errored
    expect(html).not.toContain('TypeError')
    expect(html).not.toContain('Cannot read properties')
  })
})
```

### k6 負載與壓力測試

```javascript
// tests/load/api-load.js
import http from 'k6/http'
import { check, sleep } from 'k6'

export const options = {
  stages: [
    { duration: '30s', target: 50 },   // ramp up to 50 VUs
    { duration: '1m', target: 50 },    // steady state
    { duration: '10s', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95th percentile under 500ms
    http_req_failed: ['rate<0.01'],     // less than 1% error rate
  },
}

export default function () {
  const res = http.get('http://localhost:3000/api/products')
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'response has data': (r) => {
      const body = JSON.parse(r.body)
      return body.data && body.data.length > 0
    },
  })
  sleep(1)
}
```

```javascript
// tests/load/stress-test.js — 壓力測試：找出系統極限
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate, Trend } from 'k6/metrics'

const errorRate = new Rate('error_rate')
const responseTime = new Trend('response_time', true)

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp to 100 VUs
    { duration: '5m', target: 100 },   // hold at 100
    { duration: '2m', target: 200 },   // push to 200
    { duration: '5m', target: 200 },   // hold at 200
    { duration: '2m', target: 300 },   // push to 300 (stress)
    { duration: '5m', target: 300 },   // hold at 300
    { duration: '5m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<2000'],  // 99th percentile under 2s
    error_rate: ['rate<0.05'],          // less than 5% errors under stress
  },
}

const BASE = __ENV.BASE_URL || 'http://localhost:3000'

export default function () {
  // Simulate mixed traffic patterns
  const endpoints = [
    { url: `${BASE}/`, method: 'GET', weight: 40 },
    { url: `${BASE}/api/products`, method: 'GET', weight: 30 },
    { url: `${BASE}/api/products/1`, method: 'GET', weight: 20 },
    { url: `${BASE}/api/health`, method: 'GET', weight: 10 },
  ]

  // Weighted random selection
  const rand = Math.random() * 100
  let cumulative = 0
  let selected = endpoints[0]
  for (const ep of endpoints) {
    cumulative += ep.weight
    if (rand <= cumulative) {
      selected = ep
      break
    }
  }

  const res = http.request(selected.method, selected.url)
  responseTime.add(res.timings.duration)
  errorRate.add(res.status >= 400)

  check(res, {
    'not server error': (r) => r.status < 500,
    'response time < 2s': (r) => r.timings.duration < 2000,
  })

  sleep(Math.random() * 2 + 0.5) // random 0.5-2.5s think time
}
```

### CI 整合：韌性與負載測試

```yaml
# .github/workflows/chaos.yml — 定期執行（非每次 CI push）
name: Resilience & Load Testing
on:
  schedule:
    - cron: '0 3 * * 1' # 每週一凌晨 3 點
  workflow_dispatch:       # 手動觸發

jobs:
  resilience:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: test, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm vitest run tests/chaos/
        env: { DATABASE_URL: 'postgresql://test:test@localhost:5432/test' }

  load-test:
    runs-on: ubuntu-latest
    needs: [resilience]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile && pnpm build
      - name: Start server
        run: node .output/server/index.mjs &
        env: { PORT: 3000, NODE_ENV: production }
      - name: Wait for server
        run: npx wait-on http://localhost:3000/api/health --timeout 30000
      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D68
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update && sudo apt-get install -y k6
      - name: Run load test
        run: k6 run tests/load/api-load.js --out json=k6-results.json
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: k6-results
          path: k6-results.json
          retention-days: 30
```
