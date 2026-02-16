# 企業級架構與可擴展性

## 目錄

1. [分層架構 (Layered Architecture)](#分層架構-layered-architecture)
2. [邊界定義 (Bounded Context)](#邊界定義-bounded-context)
3. [ADR (Architecture Decision Records)](#adr-architecture-decision-records)
4. [Scalability 策略](#scalability-策略)
5. [Engineering Thinking](#engineering-thinking)
6. [CQRS 模式（Command Query Responsibility Segregation）](#cqrs-模式command-query-responsibility-segregation)
7. [Use Case / Application Service 層](#use-case--application-service-層)
8. [API 版本管理策略](#api-版本管理策略)
9. [服務端依賴注入容器](#服務端依賴注入容器)
10. [Event Sourcing 模式](#event-sourcing-模式)
11. [Saga / Process Manager 模式](#saga--process-manager-模式)
12. [Value Objects（值物件）](#value-objects值物件)
13. [Command Bus / Mediator 模式](#command-bus--mediator-模式)
14. [架構模式關係圖](#架構模式關係圖)

---

## 分層架構 (Layered Architecture)

### 四層架構與 Nuxt 3 對應

```
┌─────────────────────────────────────────────────┐
│  Presentation Layer（展示層）                      │
│  pages/ ・ components/ ・ layouts/ ・ app.vue      │
├─────────────────────────────────────────────────┤
│  Application Layer（應用層）                       │
│  composables/ ・ stores/ ・ middleware/            │
├─────────────────────────────────────────────────┤
│  Domain Layer（領域層）                            │
│  types/ ・ domain/ ・ server/domain/               │
├─────────────────────────────────────────────────┤
│  Infrastructure Layer（基礎設施層）                 │
│  server/utils/ ・ server/plugins/ ・ prisma/       │
└─────────────────────────────────────────────────┘
```

| 層級 | Nuxt 3 對應 | 職責 |
|------|------------|------|
| **Presentation** | `pages/`、`components/`、`layouts/` | UI 渲染、使用者互動、路由呈現 |
| **Application** | `composables/`、`stores/`、`middleware/` | 業務流程編排、狀態管理、導航守衛 |
| **Domain** | `types/`、`domain/`、`server/domain/` | 核心業務規則、實體、值物件 |
| **Infrastructure** | `server/utils/`、`prisma/`、外部 SDK | 資料庫存取、第三方 API、快取 |

> **依賴規則（Dependency Rule）**：內層絕不可依賴外層。Domain Layer 不應 import 任何 composable 或 component；依賴方向永遠由外向內。

### Clean Architecture 實作

```typescript
// domain/entities/Order.ts — 純領域邏輯，零框架依賴
interface OrderItem { productId: string; quantity: number; unitPrice: number }

export class Order {
  constructor(
    public readonly id: string,
    public readonly customerId: string,
    private items: OrderItem[],
    public status: 'pending' | 'confirmed' | 'shipped'
  ) {}

  get totalAmount(): number {
    return this.items.reduce((sum, i) => sum + i.quantity * i.unitPrice, 0)
  }

  confirm(): void {
    if (this.status !== 'pending') throw new Error(`Cannot confirm: ${this.status}`)
    if (this.items.length === 0) throw new Error('Cannot confirm empty order')
    this.status = 'confirmed'
  }
}

// domain/repositories/OrderRepository.ts — Port（介面）
export interface OrderRepository {
  findById(id: string): Promise<Order | null>
  save(order: Order): Promise<void>
}
```

```typescript
// server/infrastructure/PrismaOrderRepository.ts — Adapter（實作）
import type { OrderRepository } from '~/domain/repositories/OrderRepository'
import { Order } from '~/domain/entities/Order'

export class PrismaOrderRepository implements OrderRepository {
  async findById(id: string): Promise<Order | null> {
    const db = usePrisma()
    const record = await db.order.findUnique({ where: { id }, include: { items: true } })
    return record ? new Order(record.id, record.customerId, record.items, record.status) : null
  }

  async save(order: Order): Promise<void> {
    const db = usePrisma()
    await db.order.upsert({
      where: { id: order.id },
      update: { status: order.status },
      create: { id: order.id, customerId: order.customerId, status: order.status }
    })
  }
}
```

```typescript
// server/api/orders/[id]/confirm.post.ts — 組裝依賴、呼叫領域邏輯
import { PrismaOrderRepository } from '~/server/infrastructure/PrismaOrderRepository'

export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  if (!id) throw createError({ statusCode: 400, message: 'Missing order ID' })

  const repo = new PrismaOrderRepository()
  const order = await repo.findById(id)
  if (!order) throw createError({ statusCode: 404, message: 'Order not found' })

  order.confirm()   // 領域邏輯在 Domain Layer
  await repo.save(order)
  return { success: true, status: order.status }
})
```

---

## 邊界定義 (Bounded Context)

### Feature Module Boundaries

每個 Feature Module 是自治的垂直切片，擁有自身的 components、composables、types 和 API routes：

```
domains/
├── catalog/                    # 商品目錄 Context
│   ├── components/
│   ├── composables/
│   ├── types/index.ts
│   └── stores/catalog.ts
├── checkout/                   # 結帳 Context
│   ├── components/
│   ├── composables/
│   ├── types/
│   └── stores/
└── account/                    # 帳戶 Context
```

### API Contract 定義

```typescript
// shared/types/api-contracts.ts — 跨 Context 共用契約
export interface PaginatedResponse<T> {
  data: T[]
  meta: { total: number; page: number; perPage: number }
}

// domains/catalog/types/index.ts — Context 內部型別
export interface Product { id: string; name: string; price: number }

// domains/checkout/types/index.ts — 不直接依賴 catalog 的 Product
export interface CheckoutItem {
  productId: string    // 僅引用 ID
  productName: string  // 複製所需欄位，避免跨 Context 依賴
  quantity: number
  unitPrice: number
}
```

> **原則**：Bounded Context 之間不應直接 import 對方的型別。透過 shared contract 或只引用 ID，在自身 Context 內重新定義所需欄位。

### Nuxt Layers 作為 Bounded Context

```typescript
// nuxt.config.ts — 每個 Layer 即一個 Bounded Context
export default defineNuxtConfig({
  extends: ['./layers/shared', './layers/catalog', './layers/checkout']
})

// layers/catalog/ 擁有完整的 pages/、components/、composables/、server/api/
// layers/shared/ 提供 base components、共用 composables 與 utils
```

### 跨模組通訊模式

```typescript
// 方式 1：Event Bus（鬆耦合一次性事件）
type EventMap = {
  'cart:item-added': { productId: string; quantity: number }
  'order:confirmed': { orderId: string }
}
const listeners = new Map<string, Set<Function>>()

export function useEventBus() {
  function emit<K extends keyof EventMap>(event: K, payload: EventMap[K]) {
    listeners.get(event)?.forEach(fn => fn(payload))
  }
  function on<K extends keyof EventMap>(event: K, handler: (payload: EventMap[K]) => void) {
    if (!listeners.has(event)) listeners.set(event, new Set())
    listeners.get(event)!.add(handler)
    return () => listeners.get(event)?.delete(handler)
  }
  return { emit, on }
}

// 方式 2：Pinia Store（持久化共享狀態）
export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const totalPrice = computed(() =>
    items.value.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0)
  )
  return { items, totalPrice }
})
```

| 模式 | 適用場景 | 耦合程度 |
|------|---------|---------|
| **Event Bus** | 跨 Context 通知、一次性事件 | 低 |
| **Pinia Store** | 多處讀寫的共享狀態 | 中 |
| **Composable Injection** | 父子元件樹、同一 Context 內部 | 低～中 |

---

## ADR (Architecture Decision Records)

### ADR 模板

```markdown
# ADR-{NNN}: {決策標題}

## 狀態
{Proposed | Accepted | Deprecated | Superseded by ADR-XXX}

## 背景
描述促成此決策的技術或業務背景。

## 決策
我們決定 {具體做法}。

## 考量的替代方案
1. **方案 A**：{描述} — {優點} / {缺點}

## 後果
- **正面**：{預期正面影響}
- **負面**：{取捨}
- **風險**：{潛在風險與緩解措施}
```

### ADR-001: 選擇 Pinia over Vuex

**狀態**：Accepted

**背景**：專案需要全域狀態管理。Nuxt 3 官方推薦 Pinia，Vuex 已進入維護模式。

**決策**：使用 Pinia 作為狀態管理方案。

**考量的替代方案**：
1. **Vuex 4** — 生態系成熟 / mutations 冗餘、TypeScript 支援弱、已停止新功能開發
2. **純 Composable（useState + ref）** — 零依賴 / 缺乏 DevTools、大型專案難追蹤
3. **Pinia** — 原生 TS 支援、無 mutations、~1.5KB、Nuxt 3 官方整合

**後果**：
- **正面**：TypeScript 零配置推導、Setup Store 語法與 Composition API 一致
- **負面**：團隊需適應無 mutations 的直接修改模式
- **風險**：需建立 store 命名規範，避免 store 膨脹

### ADR-002: 選擇 SSR + ISR 混合渲染策略

**狀態**：Accepted

**背景**：電商平台含首頁（SEO 必要）、商品頁（頻繁更新）、Dashboard（純互動）、Blog（低頻變更），各有不同效能與 SEO 需求。

**決策**：採用混合渲染，透過 `routeRules` 逐路由配置。

```typescript
export default defineNuxtConfig({
  routeRules: {
    '/':              { prerender: true },              // SSG
    '/blog/**':       { isr: 3600 },                    // ISR：每小時
    '/products/**':   { swr: 600, cache: { maxAge: 600 } },  // SWR：10 分鐘
    '/dashboard/**':  { ssr: false },                   // SPA
    '/api/**':        { cors: true, headers: { 'cache-control': 'no-store' } }
  }
})
```

**後果**：
- **正面**：各路由獲得最適渲染策略，ISR 大幅降低伺服器負載
- **負面**：除錯複雜度增加，需理解各模式行為差異
- **風險**：ISR 快取失效可能短暫提供過期內容

### ADR-003: 選擇 Prisma over Drizzle

**狀態**：Accepted

**背景**：需要 TypeScript ORM 整合 PostgreSQL。

**決策**：使用 Prisma，搭配 Prisma Accelerate 處理 Edge Runtime 連線。

**考量的替代方案**：
1. **Drizzle** — 極輕量、SQL-like、Edge 原生 / 生態系年輕、migration 較弱
2. **Prisma** — Schema-first、強大 migration、Studio GUI / bundle 較大、Edge 需 Accelerate
3. **Knex.js** — 極度靈活 / 需手動型別維護

**後果**：
- **正面**：`schema.prisma` 為 DB single source of truth、migration 成熟
- **負面**：增加 bundle size、Edge 需付費 Accelerate
- **風險**：複雜查詢可能需 `$queryRaw`

---

## Scalability 策略

### Horizontal Scaling：無狀態設計

```typescript
// ❌ 伺服器記憶體 session（無法水平擴展）
const sessions = new Map<string, UserSession>()

// ✅ Session 外部化到 Redis
import { Redis } from '@upstash/redis'
const redis = new Redis({ url: process.env.UPSTASH_REDIS_URL!, token: process.env.UPSTASH_REDIS_TOKEN! })

export async function getSession(sessionId: string) {
  return await redis.get<UserSession>(`session:${sessionId}`)
}
export async function setSession(sessionId: string, data: UserSession, ttl = 86400) {
  await redis.set(`session:${sessionId}`, data, { ex: ttl })
}
```

> **原則**：SSR 伺服器任何 instance 被終止時，不應遺失使用者狀態。所有狀態存放外部儲存（Redis / Database / Object Storage）。

### Caching Strategy：多層快取

```
Browser Cache → CDN (Cloudflare) → Application (Redis/KV) → Database
   <60s            <1hr                 <10min
```

```typescript
// nuxt.config.ts — 路由級快取
export default defineNuxtConfig({
  routeRules: {
    '/_nuxt/**': { headers: { 'cache-control': 'public, max-age=31536000, immutable' } },
    '/products/**': { isr: 600, cache: { maxAge: 600, staleMaxAge: 86400 } },
    '/api/products': { cache: { maxAge: 60 } },
    '/api/user/**': { cache: false, headers: { 'cache-control': 'private, no-store' } }
  }
})
```

```typescript
// composables/useProductList.ts — getCachedData 客戶端快取
export function useProductList() {
  return useAsyncData('products', () => $fetch('/api/products'), {
    getCachedData(key, nuxtApp) {
      const cached = nuxtApp.payload.data[key] || nuxtApp.static.data[key]
      if (!cached) return undefined
      const cachedAt = nuxtApp.payload._fetchTimestamps?.[key]
      if (cachedAt && Date.now() - cachedAt < 30_000) return cached
      return undefined
    }
  })
}

// server/utils/cache.ts — Redis 應用層快取
export async function cachedQuery<T>(key: string, fetcher: () => Promise<T>, ttl = 300): Promise<T> {
  const cached = await redis.get<T>(key)
  if (cached) return cached
  const result = await fetcher()
  await redis.set(key, result, { ex: ttl })
  return result
}
```

### Database Scaling

```typescript
// server/utils/prisma.ts — 讀寫分離（Read Replica）
import { PrismaClient } from '@prisma/client'
import { readReplicas } from '@prisma/extension-read-replicas'

const prisma = new PrismaClient().$extends(
  readReplicas({ url: process.env.DATABASE_REPLICA_URL! })
)

export function usePrisma() { return prisma }

// 使用：findMany 自動路由 Replica，create/update 自動路由 Primary
await db.product.findMany()             // → Read Replica
await db.order.create({ data: {} })     // → Primary
await db.$primary().product.findMany()  // 強制 Primary
```

**Connection Pooling**：使用 Prisma Accelerate，在 `schema.prisma` 中將 `url` 指向 `prisma://accelerate.prisma-data.net/?api_key=xxx`，`directUrl` 指向實際資料庫（供 migration 使用）。

### Edge Computing

```typescript
// nuxt.config.ts — Cloudflare Workers + KV
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
    storage: { cache: { driver: 'cloudflare-kv-binding', binding: 'CACHE_KV' } }
  }
})

// server/api/products/[id].get.ts — Edge-side KV 快取
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  const storage = useStorage('cache')
  const cacheKey = `product:${id}`

  const cached = await storage.getItem(cacheKey)
  if (cached) return cached

  const product = await $fetch(`https://api.internal/products/${id}`)
  await storage.setItem(cacheKey, product, { ttl: 600 })
  return product
})
```

---

## Engineering Thinking

### Risk Analysis 矩陣

| 風險 | 機率 | 影響 | 級別 | 緩解措施 |
|------|------|------|------|---------|
| SSR 記憶體洩漏 | 高 | 高 | 立即處理 | 避免全域可變狀態；用 `useState` 取代全域 `ref` |
| Hydration Mismatch | 高 | 中 | 優先處理 | `<ClientOnly>` 包裹瀏覽器 API；嚴格 SSR 相容 |
| Bundle Size 爆增 | 中 | 中 | 計畫處理 | Bundle Budget；`npx nuxi analyze`；動態 import |
| API 回應劣化 | 中 | 高 | 優先處理 | 多層快取；DB index 審查；N+1 偵測 |
| 第三方 API 中斷 | 中 | 高 | 優先處理 | Circuit Breaker；Fallback 資料；快取上次回應 |
| 部署後回退 | 低 | 高 | 計畫處理 | Blue-Green Deploy；可逆 DB migration |

### 技術決策 Tradeoffs 框架（簡化 ATAM）

```
1. 識別品質屬性：Performance / Scalability / Maintainability / Security / DX

2. 方案評估矩陣（每項 1-5 分）
   ┌──────────┬──────┬──────┬──────┬──────┬────┐
   │ 方案      │ 效能 │ 擴展 │ 維護 │ 安全 │ DX │
   ├──────────┼──────┼──────┼──────┼──────┼────┤
   │ 方案 A    │  4   │  3   │  5   │  4   │  5 │
   │ 方案 B    │  5   │  5   │  3   │  4   │  3 │
   └──────────┴──────┴──────┴──────┴──────┴────┘

3. 權重（依專案優先級）：效能 ×1.5 / 擴展 ×1.2 / 維護 ×1.0 / 安全 ×1.3 / DX ×0.8

4. 加權分數 → 決策 → 記錄為 ADR
```

### Cost Analysis：渲染模式成本比較

| 面向 | SSR | SSG | ISR / SWR |
|------|-----|-----|-----------|
| **伺服器成本** | 高（每請求運算） | 極低（僅 CDN） | 中（偶爾重生成） |
| **CDN 成本** | 中（動態難快取） | 低（全靜態） | 低～中（命中率高） |
| **建置時間** | 短 | 長（頁數正比） | 短（按需） |
| **TTFB** | 中～高 | 極低 | 低（快取命中） |
| **適用** | 即時個人化 | 文件/Blog | 電商/新聞 |
| **月費估算（萬次瀏覽）** | $20-50 | $0-5 | $5-15 |

> **建議**：以 ISR/SWR 為基礎策略，僅「使用者個人化內容」用 SSR，純靜態頁面用 SSG，兼顧效能與成本。

### Build vs Buy 決策框架

```
是核心業務差異化功能？
├── 是 → 自建（Build）
└── 否 → 市場上有成熟方案？
    ├── 是 → 採購/整合（Buy）
    └── 否 → 自建（Build）
```

| 功能 | Build | Buy | 建議 |
|------|-------|-----|------|
| **認證** | `server/api/auth/*` + JWT | Clerk / Auth0 / Lucia | Buy（除非嚴格合規） |
| **CMS** | 自建 admin + API | Nuxt Content / Strapi | Buy |
| **支付** | 直接串接銀行 | Stripe / TapPay | Buy（PCI-DSS 成本高） |
| **即時通訊** | WebSocket 自建 | Ably / Pusher | 視規模決定 |
| **搜尋** | PostgreSQL full-text | Algolia / Meilisearch | 大資料量 Buy |
| **Email** | SMTP 直送 | Resend / SendGrid | Buy（確保送達率） |

> **決策原則**：非核心競爭力的功能，且市場有成熟方案，優先 Buy。將工程資源集中在創造業務價值的地方。

---

## CQRS 模式（Command Query Responsibility Segregation）

### 核心概念

CQRS 將「讀取（Query）」與「寫入（Command）」職責拆分為獨立的模型與路徑。在 Nuxt 3 中，這意味著 `server/api/` 下的 API routes 明確區分為 Command（寫入端）與 Query（讀取端），各自擁有不同的資料流、驗證邏輯與最佳化策略。

```
┌─────────────────────────────────────────────────────────┐
│                      Client (Nuxt App)                  │
│     $fetch('/api/posts', { method: 'POST' })            │
│     $fetch('/api/posts')                                │
└────────────┬──────────────────────┬─────────────────────┘
             │ Command (寫入)        │ Query (讀取)
             ▼                      ▼
┌────────────────────┐   ┌────────────────────────┐
│  Command Handler   │   │    Query Handler       │
│  - 驗證輸入         │   │  - 讀取最佳化模型       │
│  - 執行業務邏輯     │   │  - 可用快取 / 讀取副本  │
│  - 寫入主資料庫     │   │  - 回傳 DTO            │
│  - 發出領域事件     │   │                        │
└────────┬───────────┘   └───────────┬────────────┘
         │                           │
         ▼                           ▼
   ┌───────────┐              ┌────────────┐
   │  Primary   │  ──同步──▶  │  Read Model │
   │  Database  │   或事件     │  (Replica / │
   └───────────┘              │   KV Cache) │
                              └────────────┘
```

### Command Handler 模式（寫入端）

Command 代表「改變系統狀態的意圖」。每個 Command 物件攜帶執行所需的全部資料，由對應的 Handler 負責驗證與執行。

```typescript
// server/cqrs/commands/types.ts — Command 基礎型別
export interface Command {
  readonly type: string
}

export interface CommandResult<T = void> {
  success: boolean
  data?: T
  error?: string
}

export interface CommandHandler<TCommand extends Command, TResult = void> {
  execute(command: TCommand): Promise<CommandResult<TResult>>
}
```

```typescript
// server/cqrs/commands/CreatePostCommand.ts
import type { Command, CommandHandler, CommandResult } from './types'

// Command：純資料物件，描述意圖
export interface CreatePostCommand extends Command {
  type: 'CreatePost'
  title: string
  content: string
  authorId: string
  tags: string[]
}

// Command Handler：執行業務邏輯
export class CreatePostHandler
  implements CommandHandler<CreatePostCommand, { id: string }>
{
  constructor(
    private readonly postRepo: PostRepository,
    private readonly eventBus: EventBus
  ) {}

  async execute(
    command: CreatePostCommand
  ): Promise<CommandResult<{ id: string }>> {
    // 1. 業務驗證
    if (!command.title.trim()) {
      return { success: false, error: '標題不可為空' }
    }
    if (command.content.length < 10) {
      return { success: false, error: '內容至少 10 個字元' }
    }

    // 2. 建立領域實體
    const post = Post.create({
      title: command.title,
      content: command.content,
      authorId: command.authorId,
      tags: command.tags,
    })

    // 3. 持久化（寫入主資料庫）
    await this.postRepo.save(post)

    // 4. 發出領域事件，通知讀取端更新
    await this.eventBus.publish({
      type: 'PostCreated',
      payload: { id: post.id, title: post.title, authorId: post.authorId },
      occurredAt: new Date(),
    })

    return { success: true, data: { id: post.id } }
  }
}
```

### Query Handler 模式（讀取端）

Query 代表「不改變狀態的資料請求」。讀取端可針對查詢場景做最佳化（扁平化 DTO、快取、讀取副本）。

```typescript
// server/cqrs/queries/types.ts — Query 基礎型別
export interface Query {
  readonly type: string
}

export interface QueryHandler<TQuery extends Query, TResult> {
  execute(query: TQuery): Promise<TResult>
}
```

```typescript
// server/cqrs/queries/GetPostsQuery.ts
import type { Query, QueryHandler } from './types'

export interface GetPostsQuery extends Query {
  type: 'GetPosts'
  page: number
  perPage: number
  tag?: string
  authorId?: string
}

// 讀取端專用 DTO（非完整 Domain Entity）
export interface PostListItem {
  id: string
  title: string
  excerpt: string
  authorName: string
  tags: string[]
  createdAt: string
}

export interface GetPostsResult {
  data: PostListItem[]
  meta: { total: number; page: number; perPage: number }
}

export class GetPostsHandler
  implements QueryHandler<GetPostsQuery, GetPostsResult>
{
  constructor(private readonly readDb: ReadDatabase) {}

  async execute(query: GetPostsQuery): Promise<GetPostsResult> {
    const { page, perPage, tag, authorId } = query
    const offset = (page - 1) * perPage

    // 讀取端可使用最佳化的檢視或讀取副本
    const where: Record<string, unknown> = {}
    if (tag) where.tags = { has: tag }
    if (authorId) where.authorId = authorId

    const [posts, total] = await Promise.all([
      this.readDb.post.findMany({
        where,
        skip: offset,
        take: perPage,
        orderBy: { createdAt: 'desc' },
        select: {
          id: true,
          title: true,
          content: true,
          author: { select: { name: true } },
          tags: true,
          createdAt: true,
        },
      }),
      this.readDb.post.count({ where }),
    ])

    return {
      data: posts.map((p) => ({
        id: p.id,
        title: p.title,
        excerpt: p.content.slice(0, 120) + '...',
        authorName: p.author.name,
        tags: p.tags,
        createdAt: p.createdAt.toISOString(),
      })),
      meta: { total, page, perPage },
    }
  }
}
```

### 事件通知：Command 端與 Query 端的橋樑

```typescript
// server/cqrs/events/EventBus.ts
export interface DomainEvent {
  type: string
  payload: Record<string, unknown>
  occurredAt: Date
}

export interface EventHandler {
  handle(event: DomainEvent): Promise<void>
}

export interface EventBus {
  subscribe(eventType: string, handler: EventHandler): void
  publish(event: DomainEvent): Promise<void>
}

export class InMemoryEventBus implements EventBus {
  private handlers = new Map<string, EventHandler[]>()

  subscribe(eventType: string, handler: EventHandler): void {
    const existing = this.handlers.get(eventType) ?? []
    existing.push(handler)
    this.handlers.set(eventType, existing)
  }

  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.type) ?? []
    await Promise.allSettled(handlers.map((h) => h.handle(event)))
  }
}

// server/cqrs/events/handlers/InvalidateCacheOnPostCreated.ts
export class InvalidateCacheOnPostCreated implements EventHandler {
  constructor(private readonly cache: CacheStore) {}

  async handle(event: DomainEvent): Promise<void> {
    if (event.type !== 'PostCreated') return
    // 當文章建立時，清除列表快取讓 Query 端取得最新資料
    await this.cache.invalidate('posts:list:*')
  }
}
```

### 完整整合：Nuxt 3 API Routes 中的 CQRS

```typescript
// server/api/posts/index.post.ts — Command 端（寫入）
import { CreatePostHandler } from '~/server/cqrs/commands/CreatePostCommand'
import { PrismaPostRepository } from '~/server/infrastructure/PrismaPostRepository'
import { InMemoryEventBus } from '~/server/cqrs/events/EventBus'
import { InvalidateCacheOnPostCreated } from '~/server/cqrs/events/handlers/InvalidateCacheOnPostCreated'

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const session = await requireUserSession(event)

  // 組裝依賴
  const eventBus = new InMemoryEventBus()
  eventBus.subscribe(
    'PostCreated',
    new InvalidateCacheOnPostCreated(useStorage('cache'))
  )
  const handler = new CreatePostHandler(new PrismaPostRepository(), eventBus)

  // 執行 Command
  const result = await handler.execute({
    type: 'CreatePost',
    title: body.title,
    content: body.content,
    authorId: session.user.id,
    tags: body.tags ?? [],
  })

  if (!result.success) {
    throw createError({ statusCode: 400, message: result.error })
  }

  setResponseStatus(event, 201)
  return result.data
})
```

```typescript
// server/api/posts/index.get.ts — Query 端（讀取）
import { GetPostsHandler } from '~/server/cqrs/queries/GetPostsQuery'

export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const page = Number(query.page) || 1
  const perPage = Math.min(Number(query.perPage) || 20, 100)

  const handler = new GetPostsHandler(usePrismaReadReplica())

  return await handler.execute({
    type: 'GetPosts',
    page,
    perPage,
    tag: query.tag as string | undefined,
    authorId: query.authorId as string | undefined,
  })
})
```

> **何時採用 CQRS**：當讀取與寫入的模型差異大（例如列表頁需要扁平化 DTO，但寫入需嚴格驗證）、或讀取量遠大於寫入量需獨立擴展時，CQRS 能帶來顯著效益。小型 CRUD 應用不需要此模式。

---

## Use Case / Application Service 層

### 為何需要 Use Case 層

在 Nuxt 3 後端開發中，常見的反模式是將所有業務邏輯直接塞入 `defineEventHandler`。這導致：

1. **無法測試**：業務邏輯與 H3 的 `event` 物件緊密耦合，單元測試需模擬 HTTP 請求
2. **難以重用**：相同邏輯在 REST API、WebSocket、排程任務中需重複撰寫
3. **職責模糊**：HTTP 處理（解析 body、設 status code）與業務規則混在一起

Use Case 類別是「應用層」的核心元件，負責編排一個完整的業務流程：

```
API Handler (HTTP 入口)
   │
   │  1. 解析 / 驗證 HTTP 輸入
   │  2. 建構 Use Case Input DTO
   │
   ▼
Use Case (業務流程編排)
   │
   │  1. 呼叫 Domain Entity 執行業務規則
   │  2. 協調 Repository / 外部服務
   │  3. 回傳 Output DTO
   │
   ▼
API Handler (HTTP 出口)
   │
   │  設定 status code、格式化回應
```

### Use Case 基礎結構

```typescript
// server/application/types.ts
export interface UseCase<TInput, TOutput> {
  execute(input: TInput): Promise<TOutput>
}

// Use Case 結果：明確區分成功與失敗
export type UseCaseResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: string; code: 'VALIDATION' | 'NOT_FOUND' | 'CONFLICT' | 'FORBIDDEN' }
```

### 範例：CreatePostUseCase

```typescript
// server/application/use-cases/CreatePostUseCase.ts
import type { UseCase, UseCaseResult } from '../types'
import type { PostRepository } from '~/server/domain/repositories/PostRepository'
import type { SlugGenerator } from '~/server/domain/services/SlugGenerator'
import { Post } from '~/server/domain/entities/Post'

// Input DTO — 純資料，不含任何 HTTP 概念
export interface CreatePostInput {
  title: string
  content: string
  authorId: string
  tags: string[]
}

// Output DTO — 前端所需的資料形狀
export interface CreatePostOutput {
  id: string
  slug: string
  createdAt: string
}

export class CreatePostUseCase
  implements UseCase<CreatePostInput, UseCaseResult<CreatePostOutput>>
{
  constructor(
    private readonly postRepo: PostRepository,
    private readonly slugGenerator: SlugGenerator
  ) {}

  async execute(
    input: CreatePostInput
  ): Promise<UseCaseResult<CreatePostOutput>> {
    // 1. 業務驗證
    if (!input.title.trim()) {
      return { ok: false, error: '標題不可為空', code: 'VALIDATION' }
    }
    if (input.content.length < 10) {
      return { ok: false, error: '內容至少 10 個字元', code: 'VALIDATION' }
    }
    if (input.tags.length > 5) {
      return { ok: false, error: '標籤最多 5 個', code: 'VALIDATION' }
    }

    // 2. 產生唯一 slug
    const slug = await this.slugGenerator.generate(input.title)
    const existing = await this.postRepo.findBySlug(slug)
    if (existing) {
      return { ok: false, error: '標題產生的 slug 已存在', code: 'CONFLICT' }
    }

    // 3. 建立 Domain Entity
    const post = Post.create({
      title: input.title,
      content: input.content,
      slug,
      authorId: input.authorId,
      tags: input.tags,
    })

    // 4. 持久化
    await this.postRepo.save(post)

    // 5. 回傳 Output DTO
    return {
      ok: true,
      value: {
        id: post.id,
        slug: post.slug,
        createdAt: post.createdAt.toISOString(),
      },
    }
  }
}
```

### 範例：GetPostsQuery（查詢用 Use Case）

```typescript
// server/application/use-cases/GetPostsQuery.ts
import type { UseCase } from '../types'
import type { PostReadRepository } from '~/server/domain/repositories/PostReadRepository'

export interface GetPostsInput {
  page: number
  perPage: number
  tag?: string
}

export interface PostSummary {
  id: string
  title: string
  slug: string
  excerpt: string
  authorName: string
  tags: string[]
  publishedAt: string
}

export interface GetPostsOutput {
  data: PostSummary[]
  meta: { total: number; page: number; perPage: number; totalPages: number }
}

export class GetPostsQuery
  implements UseCase<GetPostsInput, GetPostsOutput>
{
  constructor(private readonly postReadRepo: PostReadRepository) {}

  async execute(input: GetPostsInput): Promise<GetPostsOutput> {
    const { page, perPage, tag } = input
    const offset = (page - 1) * perPage

    const [posts, total] = await Promise.all([
      this.postReadRepo.findPublished({ offset, limit: perPage, tag }),
      this.postReadRepo.countPublished({ tag }),
    ])

    return {
      data: posts.map((p) => ({
        id: p.id,
        title: p.title,
        slug: p.slug,
        excerpt: p.content.slice(0, 150) + '...',
        authorName: p.author.name,
        tags: p.tags,
        publishedAt: p.publishedAt.toISOString(),
      })),
      meta: {
        total,
        page,
        perPage,
        totalPages: Math.ceil(total / perPage),
      },
    }
  }
}
```

### API Handler 中使用 Use Case（職責分離）

```typescript
// server/api/posts/index.post.ts — HTTP 層僅處理 HTTP 關注點
import { CreatePostUseCase } from '~/server/application/use-cases/CreatePostUseCase'
import { PrismaPostRepository } from '~/server/infrastructure/PrismaPostRepository'
import { DefaultSlugGenerator } from '~/server/infrastructure/DefaultSlugGenerator'

export default defineEventHandler(async (event) => {
  // 1. HTTP 關注點：認證、解析 body
  const session = await requireUserSession(event)
  const body = await readBody(event)

  // 2. 注入依賴、執行 Use Case
  const useCase = new CreatePostUseCase(
    new PrismaPostRepository(),
    new DefaultSlugGenerator()
  )
  const result = await useCase.execute({
    title: body.title,
    content: body.content,
    authorId: session.user.id,
    tags: body.tags ?? [],
  })

  // 3. HTTP 關注點：將 Use Case 結果映射為 HTTP 回應
  if (!result.ok) {
    const statusMap = {
      VALIDATION: 400,
      NOT_FOUND: 404,
      CONFLICT: 409,
      FORBIDDEN: 403,
    } as const
    throw createError({
      statusCode: statusMap[result.code],
      message: result.error,
    })
  }

  setResponseStatus(event, 201)
  return result.value
})
```

```typescript
// server/api/posts/index.get.ts
import { GetPostsQuery } from '~/server/application/use-cases/GetPostsQuery'
import { PrismaPostReadRepository } from '~/server/infrastructure/PrismaPostReadRepository'

export default defineEventHandler(async (event) => {
  const query = getQuery(event)

  const useCase = new GetPostsQuery(new PrismaPostReadRepository())
  return await useCase.execute({
    page: Number(query.page) || 1,
    perPage: Math.min(Number(query.perPage) || 20, 100),
    tag: query.tag as string | undefined,
  })
})
```

### Use Case 的依賴注入

Use Case 透過建構函式接收所有外部依賴（Repository、Service），使其易於測試：

```typescript
// 單元測試：完全不需要 HTTP、資料庫
import { describe, it, expect } from 'vitest'
import { CreatePostUseCase } from '~/server/application/use-cases/CreatePostUseCase'

describe('CreatePostUseCase', () => {
  it('should reject empty title', async () => {
    const mockRepo = { save: vi.fn(), findBySlug: vi.fn() }
    const mockSlug = { generate: vi.fn().mockResolvedValue('test-slug') }

    const useCase = new CreatePostUseCase(mockRepo, mockSlug)
    const result = await useCase.execute({
      title: '',
      content: 'Some valid content here',
      authorId: 'user-1',
      tags: [],
    })

    expect(result.ok).toBe(false)
    if (!result.ok) {
      expect(result.code).toBe('VALIDATION')
      expect(result.error).toContain('標題')
    }
    expect(mockRepo.save).not.toHaveBeenCalled()
  })

  it('should create post successfully', async () => {
    const mockRepo = {
      save: vi.fn().mockResolvedValue(undefined),
      findBySlug: vi.fn().mockResolvedValue(null),
    }
    const mockSlug = {
      generate: vi.fn().mockResolvedValue('my-post-title'),
    }

    const useCase = new CreatePostUseCase(mockRepo, mockSlug)
    const result = await useCase.execute({
      title: 'My Post Title',
      content: 'This is valid content for the post',
      authorId: 'user-1',
      tags: ['nuxt', 'vue'],
    })

    expect(result.ok).toBe(true)
    expect(mockRepo.save).toHaveBeenCalledOnce()
  })
})
```

> **原則**：API Handler 應是「薄的黏合層」——解析 HTTP 輸入、呼叫 Use Case、映射 HTTP 輸出。所有業務邏輯封裝於 Use Case 與 Domain Entity 中。

---

## API 版本管理策略

### 為何需要 API 版本管理

當 API 有外部消費者（行動 App、第三方整合）時，breaking changes 必須透過版本控制來管理，確保舊客戶端不會因後端更新而中斷。

### 策略一：URL-based 版本（推薦用於 Nuxt 3）

最直觀、對開發者最友善的方式，直接在 URL 路徑中嵌入版本號。

```
server/
├── api/
│   ├── v1/
│   │   ├── posts/
│   │   │   ├── index.get.ts      # GET /api/v1/posts
│   │   │   ├── index.post.ts     # POST /api/v1/posts
│   │   │   └── [id].get.ts       # GET /api/v1/posts/:id
│   │   └── users/
│   │       └── index.get.ts      # GET /api/v1/users
│   ├── v2/
│   │   ├── posts/
│   │   │   ├── index.get.ts      # GET /api/v2/posts（新回應格式）
│   │   │   └── [slug].get.ts     # GET /api/v2/posts/:slug（改用 slug）
│   │   └── users/
│   │       └── index.get.ts
│   └── health.get.ts             # 不版本化的端點
```

```typescript
// server/api/v1/posts/index.get.ts — V1 回傳扁平結構
export default defineEventHandler(async (event) => {
  const posts = await usePrisma().post.findMany({
    include: { author: true },
  })

  // V1 回應格式：陣列外包一個 data 欄位
  return {
    data: posts.map((p) => ({
      id: p.id,
      title: p.title,
      body: p.content, // V1 用 "body" 欄位名
      author: p.author.name,
      created: p.createdAt.toISOString(),
    })),
  }
})
```

```typescript
// server/api/v2/posts/index.get.ts — V2 改進回應格式
export default defineEventHandler(async (event) => {
  const query = getQuery(event)
  const page = Number(query.page) || 1
  const perPage = Math.min(Number(query.perPage) || 20, 100)

  const [posts, total] = await Promise.all([
    usePrisma().post.findMany({
      skip: (page - 1) * perPage,
      take: perPage,
      include: { author: true, tags: true },
      orderBy: { createdAt: 'desc' },
    }),
    usePrisma().post.count(),
  ])

  // V2 回應格式：增加 meta 分頁、改用 "content" 與嵌套 author
  return {
    data: posts.map((p) => ({
      id: p.id,
      title: p.title,
      slug: p.slug,
      content: p.content, // V2 改為 "content"
      author: {            // V2 改為嵌套物件
        id: p.author.id,
        name: p.author.name,
      },
      tags: p.tags.map((t) => t.name),
      createdAt: p.createdAt.toISOString(),
      updatedAt: p.updatedAt.toISOString(),
    })),
    meta: {
      total,
      page,
      perPage,
      totalPages: Math.ceil(total / perPage),
    },
  }
})
```

### 策略二：Header-based 版本

透過自訂 HTTP Header 指定版本，URL 保持乾淨。適合內部 API 或有嚴格 REST 要求的場景。

```typescript
// server/middleware/api-version.ts — 全域中介層解析版本
export default defineEventHandler((event) => {
  const path = getRequestURL(event).pathname
  if (!path.startsWith('/api/')) return

  // 優先使用 Accept-Version header，預設 v1
  const version =
    getHeader(event, 'Accept-Version') ??
    getHeader(event, 'X-API-Version') ??
    'v1'

  // 將版本資訊存入 event context 供後續 handler 使用
  event.context.apiVersion = version
})
```

```typescript
// server/api/posts/index.get.ts — 單一路由、依版本切換行為
import { formatPostV1, formatPostV2 } from '~/server/utils/post-formatters'

export default defineEventHandler(async (event) => {
  const version = event.context.apiVersion ?? 'v1'
  const posts = await usePrisma().post.findMany({ include: { author: true } })

  switch (version) {
    case 'v2':
      return { data: posts.map(formatPostV2), apiVersion: 'v2' }
    case 'v1':
    default:
      return { data: posts.map(formatPostV1), apiVersion: 'v1' }
  }
})
```

```typescript
// 客戶端呼叫
const posts = await $fetch('/api/posts', {
  headers: { 'Accept-Version': 'v2' },
})
```

### 策略三：Content Negotiation（媒體型別版本）

遵循 REST 最佳實踐，透過 `Accept` header 中的自訂媒體型別指定版本。

```typescript
// server/middleware/content-negotiation.ts
export default defineEventHandler((event) => {
  const path = getRequestURL(event).pathname
  if (!path.startsWith('/api/')) return

  const accept = getHeader(event, 'Accept') ?? ''

  // 解析自訂媒體型別：application/vnd.myapp.v2+json
  const versionMatch = accept.match(/application\/vnd\.myapp\.v(\d+)\+json/)
  event.context.apiVersion = versionMatch ? `v${versionMatch[1]}` : 'v1'

  // 設定回應 Content-Type
  setHeader(
    event,
    'Content-Type',
    `application/vnd.myapp.${event.context.apiVersion}+json`
  )
})
```

### 版本棄用策略

```typescript
// server/middleware/version-deprecation.ts
interface VersionPolicy {
  status: 'active' | 'deprecated' | 'sunset'
  sunsetDate?: string
  successor?: string
}

const VERSION_POLICY: Record<string, VersionPolicy> = {
  v1: {
    status: 'deprecated',
    sunsetDate: '2026-06-01',
    successor: 'v2',
  },
  v2: { status: 'active' },
}

export default defineEventHandler((event) => {
  const path = getRequestURL(event).pathname
  if (!path.startsWith('/api/')) return

  // 從 URL 路徑提取版本號
  const versionMatch = path.match(/\/api\/(v\d+)\//)
  if (!versionMatch) return

  const version = versionMatch[1]
  const policy = VERSION_POLICY[version]

  if (!policy) {
    throw createError({ statusCode: 400, message: `不支援的 API 版本：${version}` })
  }

  if (policy.status === 'sunset') {
    throw createError({
      statusCode: 410, // Gone
      message: `API ${version} 已停止服務，請升級至 ${policy.successor}`,
    })
  }

  if (policy.status === 'deprecated') {
    // 設定棄用相關 Header（RFC 8594）
    setHeader(event, 'Deprecation', 'true')
    setHeader(event, 'Sunset', policy.sunsetDate!)
    setHeader(
      event,
      'Link',
      `</api/${policy.successor}/>; rel="successor-version"`
    )
    // 加入警告 Header
    setHeader(
      event,
      'Warning',
      `299 - "API ${version} 已棄用，將於 ${policy.sunsetDate} 停止服務，請遷移至 ${policy.successor}"`
    )
  }
})
```

### 版本策略比較

| 面向 | URL-based (`/api/v1/`) | Header-based | Content Negotiation |
|------|----------------------|--------------|---------------------|
| **可見性** | 高（URL 自帶版本） | 低（需檢查 Header） | 低 |
| **快取友善** | 高（不同 URL 獨立快取） | 需 `Vary` Header | 需 `Vary` Header |
| **Nuxt 3 實作** | 最簡單（資料夾即版本） | 需全域中介層 | 需全域中介層 |
| **REST 純度** | 低（URI 含版本資訊） | 中 | 高 |
| **適用場景** | 公開 API、多數專案 | 內部微服務 | 嚴格 REST API |
| **推薦** | 大多數 Nuxt 3 專案首選 | 進階場景 | 特殊需求 |

> **建議**：Nuxt 3 專案優先使用 URL-based 版本管理。資料夾結構天然對應版本，開發者體驗最佳，且 CDN 快取無需額外配置。

---

## 服務端依賴注入容器

### 為何在 Nitro 中需要 DI

隨著後端規模增長，手動在每個 API handler 中 `new` 出依賴會導致：

- 依賴建立邏輯分散在多處
- 無法統一管理生命週期（singleton vs per-request）
- 測試時替換依賴困難

一個輕量的 DI 容器能解決這些問題，無需引入 `tsyringe` 或 `inversify` 等重量級框架。

### 型別安全的 DI 容器

```typescript
// server/infrastructure/container/Container.ts

type Factory<T> = () => T
type Lifetime = 'singleton' | 'transient'

interface Registration<T> {
  factory: Factory<T>
  lifetime: Lifetime
  instance?: T
}

// 服務註冊表介面：用 interface 定義所有可注入的服務
export interface ServiceRegistry {
  // Repositories
  postRepository: PostRepository
  postReadRepository: PostReadRepository
  userRepository: UserRepository
  commentRepository: CommentRepository

  // Services
  slugGenerator: SlugGenerator
  emailService: EmailService
  eventBus: EventBus
  commandBus: CommandBus

  // Use Cases
  createPostUseCase: CreatePostUseCase
  getPostsQuery: GetPostsQuery

  // Infrastructure
  prisma: PrismaClient
  redis: RedisClient
  cache: CacheStore
}

export class DIContainer {
  private registrations = new Map<string, Registration<unknown>>()

  /**
   * 註冊服務工廠
   */
  register<K extends keyof ServiceRegistry>(
    name: K,
    factory: Factory<ServiceRegistry[K]>,
    lifetime: Lifetime = 'transient'
  ): void {
    this.registrations.set(name as string, { factory, lifetime })
  }

  /**
   * 解析服務實例
   */
  resolve<K extends keyof ServiceRegistry>(name: K): ServiceRegistry[K] {
    const registration = this.registrations.get(name as string)
    if (!registration) {
      throw new Error(`服務未註冊：${String(name)}`)
    }

    // Singleton：只建立一次，後續回傳同一實例
    if (registration.lifetime === 'singleton') {
      if (!registration.instance) {
        registration.instance = registration.factory()
      }
      return registration.instance as ServiceRegistry[K]
    }

    // Transient：每次解析建立新實例
    return registration.factory() as ServiceRegistry[K]
  }

  /**
   * 檢查服務是否已註冊
   */
  has<K extends keyof ServiceRegistry>(name: K): boolean {
    return this.registrations.has(name as string)
  }

  /**
   * 清除所有 singleton 實例（用於測試）
   */
  reset(): void {
    for (const reg of this.registrations.values()) {
      reg.instance = undefined
    }
  }
}
```

### 容器初始化與註冊

```typescript
// server/infrastructure/container/setup.ts
import { DIContainer } from './Container'
import { PrismaClient } from '@prisma/client'
import { PrismaPostRepository } from '~/server/infrastructure/PrismaPostRepository'
import { PrismaUserRepository } from '~/server/infrastructure/PrismaUserRepository'
import { DefaultSlugGenerator } from '~/server/infrastructure/DefaultSlugGenerator'
import { ResendEmailService } from '~/server/infrastructure/ResendEmailService'
import { InMemoryEventBus } from '~/server/cqrs/events/EventBus'
import { CreatePostUseCase } from '~/server/application/use-cases/CreatePostUseCase'
import { GetPostsQuery } from '~/server/application/use-cases/GetPostsQuery'
import { PrismaPostReadRepository } from '~/server/infrastructure/PrismaPostReadRepository'

let container: DIContainer | null = null

export function createContainer(): DIContainer {
  const c = new DIContainer()

  // --- Infrastructure（Singleton）---
  c.register('prisma', () => new PrismaClient(), 'singleton')
  c.register('eventBus', () => new InMemoryEventBus(), 'singleton')

  // --- Repositories（Transient — 每次解析取得新實例）---
  c.register('postRepository', () => new PrismaPostRepository(c.resolve('prisma')))
  c.register('postReadRepository', () => new PrismaPostReadRepository(c.resolve('prisma')))
  c.register('userRepository', () => new PrismaUserRepository(c.resolve('prisma')))

  // --- Services ---
  c.register('slugGenerator', () => new DefaultSlugGenerator(), 'singleton')
  c.register(
    'emailService',
    () => new ResendEmailService(process.env.RESEND_API_KEY!),
    'singleton'
  )

  // --- Use Cases（Transient — 每次請求建立新的）---
  c.register(
    'createPostUseCase',
    () => new CreatePostUseCase(c.resolve('postRepository'), c.resolve('slugGenerator'))
  )
  c.register(
    'getPostsQuery',
    () => new GetPostsQuery(c.resolve('postReadRepository'))
  )

  return c
}

/**
 * 取得全域容器實例（lazy 初始化）
 */
export function useContainer(): DIContainer {
  if (!container) {
    container = createContainer()
  }
  return container
}
```

### 透過 Nitro Plugin 初始化容器

```typescript
// server/plugins/container.ts — Nitro 啟動時初始化
import { createContainer } from '~/server/infrastructure/container/setup'

export default defineNitroPlugin((nitroApp) => {
  const container = createContainer()

  // 將容器掛載到 Nitro context，所有 handler 均可存取
  nitroApp.hooks.hook('request', (event) => {
    event.context.container = container
  })

  // 伺服器關閉時清理資源
  nitroApp.hooks.hook('close', async () => {
    const prisma = container.resolve('prisma')
    await prisma.$disconnect()
  })
})
```

### 在 API Handler 中使用容器

```typescript
// server/utils/container.ts — 便利函式
import type { DIContainer } from '~/server/infrastructure/container/Container'
import type { H3Event } from 'h3'

/**
 * 從 event context 取得 DI 容器
 */
export function useContainerFromEvent(event: H3Event): DIContainer {
  const container = event.context.container as DIContainer | undefined
  if (!container) {
    throw createError({
      statusCode: 500,
      message: 'DI container not initialized',
    })
  }
  return container
}
```

```typescript
// server/api/posts/index.post.ts — 使用 DI 容器
export default defineEventHandler(async (event) => {
  const session = await requireUserSession(event)
  const body = await readBody(event)

  // 透過容器解析 Use Case（所有依賴自動注入）
  const container = useContainerFromEvent(event)
  const useCase = container.resolve('createPostUseCase')

  const result = await useCase.execute({
    title: body.title,
    content: body.content,
    authorId: session.user.id,
    tags: body.tags ?? [],
  })

  if (!result.ok) {
    throw createError({ statusCode: 400, message: result.error })
  }

  setResponseStatus(event, 201)
  return result.value
})
```

### 測試時替換依賴

```typescript
// tests/unit/setup.ts — 測試用容器
import { DIContainer } from '~/server/infrastructure/container/Container'

export function createTestContainer(): DIContainer {
  const c = new DIContainer()

  // 註冊 mock 實作
  c.register('postRepository', () => ({
    save: vi.fn().mockResolvedValue(undefined),
    findById: vi.fn().mockResolvedValue(null),
    findBySlug: vi.fn().mockResolvedValue(null),
  }))
  c.register('slugGenerator', () => ({
    generate: vi.fn().mockResolvedValue('mock-slug'),
  }))
  c.register(
    'createPostUseCase',
    () =>
      new CreatePostUseCase(
        c.resolve('postRepository'),
        c.resolve('slugGenerator')
      )
  )

  return c
}

// tests/unit/CreatePostUseCase.test.ts
import { createTestContainer } from './setup'

describe('CreatePostUseCase', () => {
  it('should create post via container', async () => {
    const container = createTestContainer()
    const useCase = container.resolve('createPostUseCase')

    const result = await useCase.execute({
      title: 'Test Post',
      content: 'Valid content for testing',
      authorId: 'user-1',
      tags: [],
    })

    expect(result.ok).toBe(true)
  })
})
```

### Scoped vs Singleton 生命週期指南

| 生命週期 | 何時使用 | 範例 |
|---------|---------|------|
| **Singleton** | 無狀態、昂貴初始化、全域共用 | `PrismaClient`、`EventBus`、`Redis` 連線 |
| **Transient** | 含請求級狀態、輕量建立 | `Repository`（因可能含請求 context）、`UseCase` |

> **注意**：Nitro 執行在單一 Node.js 程序中，Singleton 服務在所有請求間共用。確保 Singleton 是執行緒安全（無可變共享狀態）的。Transient 服務每次解析產生新實例，適合需要隔離狀態的場景。

---

## Event Sourcing 模式

### 核心概念

Event Sourcing 不儲存實體的「當前狀態」，而是儲存導致狀態變更的「所有事件」。透過重播事件序列即可還原任何時間點的狀態。此模式與 CQRS 天然互補——Command 端產生事件，Query 端由事件投影（Projection）生成讀取模型。

```
┌──────────────┐     append      ┌──────────────────┐
│   Command    │ ──────────────▶ │   Event Store    │
│   Handler    │                 │  (Append-Only)   │
└──────────────┘                 └───────┬──────────┘
                                         │ publish
                                         ▼
                                 ┌──────────────────┐
                                 │  Event Handlers   │
                                 │  (Projections)    │
                                 └───────┬──────────┘
                                         │ update
                                         ▼
                                 ┌──────────────────┐
                                 │   Read Model     │
                                 │  (Query 端)      │
                                 └──────────────────┘
```

### 領域事件定義

```typescript
// server/domain/events/types.ts — 領域事件基礎型別
export interface DomainEvent {
  readonly eventId: string
  readonly aggregateId: string
  readonly aggregateType: string
  readonly eventType: string
  readonly version: number
  readonly timestamp: Date
  readonly payload: Record<string, unknown>
  readonly metadata: EventMetadata
}

export interface EventMetadata {
  /** 觸發此事件的 Command ID，用於追蹤因果鏈 */
  readonly causationId: string
  /** 同一業務流程的關聯 ID，串聯所有相關事件 */
  readonly correlationId: string
  /** 執行操作的使用者 ID */
  readonly userId?: string
}

export interface AggregateSnapshot {
  readonly aggregateId: string
  readonly aggregateType: string
  readonly version: number
  readonly state: Record<string, unknown>
  readonly createdAt: Date
}
```

### Event Store 介面與實作

```typescript
// server/domain/events/event-store.ts — Event Store Port（介面）
export interface EventStore {
  /** 附加事件到聚合的事件流，expectedVersion 用於樂觀並發控制 */
  append(aggregateId: string, events: DomainEvent[], expectedVersion: number): Promise<void>
  /** 取得聚合的事件流，可從指定版本開始 */
  getEvents(aggregateId: string, fromVersion?: number): Promise<DomainEvent[]>
  /** 取得聚合快照（用於最佳化大量事件的重播） */
  getSnapshot(aggregateId: string): Promise<AggregateSnapshot | null>
  /** 儲存聚合快照 */
  saveSnapshot(snapshot: AggregateSnapshot): Promise<void>
}
```

```typescript
// server/infrastructure/PrismaEventStore.ts — Event Store Adapter（實作）
import type { DomainEvent, AggregateSnapshot } from '~/server/domain/events/types'
import type { EventStore } from '~/server/domain/events/event-store'

export class PrismaEventStore implements EventStore {
  constructor(private readonly prisma: PrismaClient) {}

  async append(
    aggregateId: string,
    events: DomainEvent[],
    expectedVersion: number
  ): Promise<void> {
    // 樂觀並發控制：使用交易確保版本檢查與寫入的原子性
    await this.prisma.$transaction(async (tx) => {
      const currentVersion = await tx.event.count({
        where: { aggregateId },
      })

      if (currentVersion !== expectedVersion) {
        throw new ConcurrencyError(
          `Expected version ${expectedVersion}, but current is ${currentVersion}`
        )
      }

      // 批次寫入事件（交易內原子操作）
      await tx.event.createMany({
        data: events.map((e) => ({
          eventId: e.eventId,
          aggregateId: e.aggregateId,
          aggregateType: e.aggregateType,
          eventType: e.eventType,
          version: e.version,
          timestamp: e.timestamp,
          payload: e.payload as any,
          causationId: e.metadata.causationId,
          correlationId: e.metadata.correlationId,
          userId: e.metadata.userId,
        })),
      })
    })
  }

  async getEvents(aggregateId: string, fromVersion = 0): Promise<DomainEvent[]> {
    const records = await this.prisma.event.findMany({
      where: { aggregateId, version: { gte: fromVersion } },
      orderBy: { version: 'asc' },
    })

    return records.map((r) => ({
      eventId: r.eventId,
      aggregateId: r.aggregateId,
      aggregateType: r.aggregateType,
      eventType: r.eventType,
      version: r.version,
      timestamp: r.timestamp,
      payload: r.payload as Record<string, unknown>,
      metadata: {
        causationId: r.causationId,
        correlationId: r.correlationId,
        userId: r.userId ?? undefined,
      },
    }))
  }

  async getSnapshot(aggregateId: string): Promise<AggregateSnapshot | null> {
    const snapshot = await this.prisma.aggregateSnapshot.findFirst({
      where: { aggregateId },
      orderBy: { version: 'desc' },
    })
    return snapshot
      ? {
          aggregateId: snapshot.aggregateId,
          aggregateType: snapshot.aggregateType,
          version: snapshot.version,
          state: snapshot.state as Record<string, unknown>,
          createdAt: snapshot.createdAt,
        }
      : null
  }

  async saveSnapshot(snapshot: AggregateSnapshot): Promise<void> {
    await this.prisma.aggregateSnapshot.upsert({
      where: { aggregateId: snapshot.aggregateId },
      update: {
        version: snapshot.version,
        state: snapshot.state as any,
        createdAt: snapshot.createdAt,
      },
      create: {
        aggregateId: snapshot.aggregateId,
        aggregateType: snapshot.aggregateType,
        version: snapshot.version,
        state: snapshot.state as any,
        createdAt: snapshot.createdAt,
      },
    })
  }
}
```

### Event-Sourced Aggregate 基礎類別

```typescript
// server/domain/events/event-sourced-aggregate.ts
import type { DomainEvent } from './types'
import { randomUUID } from 'uncrypto'

export abstract class EventSourcedAggregate {
  private _version = 0
  private _uncommittedEvents: DomainEvent[] = []

  constructor(public readonly id: string) {}

  get version(): number {
    return this._version
  }

  /** 取得尚未持久化的新事件 */
  get uncommittedEvents(): ReadonlyArray<DomainEvent> {
    return this._uncommittedEvents
  }

  /** 持久化後清除未提交事件 */
  clearUncommittedEvents(): void {
    this._uncommittedEvents = []
  }

  /** 從歷史事件重建聚合狀態 */
  loadFromHistory(events: DomainEvent[]): void {
    for (const event of events) {
      this.applyEvent(event, false)
      this._version = event.version
    }
  }

  /** 套用新事件（業務操作時呼叫） */
  protected raise(
    eventType: string,
    payload: Record<string, unknown>,
    metadata: { causationId: string; correlationId: string; userId?: string }
  ): void {
    const event: DomainEvent = {
      eventId: randomUUID(),
      aggregateId: this.id,
      aggregateType: this.constructor.name,
      eventType,
      version: this._version + 1,
      timestamp: new Date(),
      payload,
      metadata,
    }

    this.applyEvent(event, true)
  }

  private applyEvent(event: DomainEvent, isNew: boolean): void {
    this.apply(event)
    this._version = event.version
    if (isNew) {
      this._uncommittedEvents.push(event)
    }
  }

  /** 子類別實作：根據事件類型更新內部狀態 */
  protected abstract apply(event: DomainEvent): void
}
```

### 具體聚合範例：Order

```typescript
// server/domain/entities/OrderES.ts — Event-Sourced Order Aggregate
import { EventSourcedAggregate } from '../events/event-sourced-aggregate'
import type { DomainEvent } from '../events/types'

interface OrderItem {
  productId: string
  quantity: number
  unitPrice: number
}

export class OrderAggregate extends EventSourcedAggregate {
  private _customerId = ''
  private _items: OrderItem[] = []
  private _status: 'pending' | 'confirmed' | 'shipped' | 'cancelled' = 'pending'
  private _totalAmount = 0

  get customerId(): string { return this._customerId }
  get items(): ReadonlyArray<OrderItem> { return this._items }
  get status(): string { return this._status }
  get totalAmount(): number { return this._totalAmount }

  /** 工廠方法：建立新訂單 */
  static create(
    id: string,
    customerId: string,
    items: OrderItem[],
    metadata: { causationId: string; correlationId: string; userId?: string }
  ): OrderAggregate {
    const order = new OrderAggregate(id)
    order.raise('OrderCreated', { customerId, items }, metadata)
    return order
  }

  /** 確認訂單 */
  confirm(metadata: { causationId: string; correlationId: string; userId?: string }): void {
    if (this._status !== 'pending') {
      throw new Error(`Cannot confirm order in status: ${this._status}`)
    }
    this.raise('OrderConfirmed', { confirmedAt: new Date().toISOString() }, metadata)
  }

  /** 取消訂單 */
  cancel(
    reason: string,
    metadata: { causationId: string; correlationId: string; userId?: string }
  ): void {
    if (this._status === 'shipped') {
      throw new Error('Cannot cancel shipped order')
    }
    this.raise('OrderCancelled', { reason, cancelledAt: new Date().toISOString() }, metadata)
  }

  /** 根據事件類型更新狀態（純函式，無副作用） */
  protected apply(event: DomainEvent): void {
    switch (event.eventType) {
      case 'OrderCreated': {
        this._customerId = event.payload.customerId as string
        this._items = event.payload.items as OrderItem[]
        this._status = 'pending'
        this._totalAmount = this._items.reduce(
          (sum, i) => sum + i.quantity * i.unitPrice, 0
        )
        break
      }
      case 'OrderConfirmed':
        this._status = 'confirmed'
        break
      case 'OrderCancelled':
        this._status = 'cancelled'
        break
    }
  }
}
```

### 快照最佳化

當事件流過長（例如上千筆事件），每次重播的成本很高。快照記錄某個版本的完整狀態，後續只需從快照版本開始重播：

```typescript
// server/domain/repositories/EventSourcedRepository.ts
import type { AggregateSnapshot } from '~/server/domain/events/types'
import type { EventStore } from '~/server/domain/events/event-store'
import { EventSourcedAggregate } from '~/server/domain/events/event-sourced-aggregate'

const SNAPSHOT_INTERVAL = 50 // 每 50 個事件建立一次快照

export class EventSourcedRepository<T extends EventSourcedAggregate> {
  constructor(
    private readonly eventStore: EventStore,
    private readonly factory: (id: string) => T
  ) {}

  async load(aggregateId: string): Promise<T | null> {
    // 1. 嘗試載入快照
    const snapshot = await this.eventStore.getSnapshot(aggregateId)
    const aggregate = this.factory(aggregateId)

    if (snapshot) {
      // 從快照還原基礎狀態
      Object.assign(aggregate, snapshot.state)
    }

    // 2. 從快照版本之後開始重播事件
    const fromVersion = snapshot ? snapshot.version + 1 : 0
    const events = await this.eventStore.getEvents(aggregateId, fromVersion)

    if (!snapshot && events.length === 0) return null

    aggregate.loadFromHistory(events)
    return aggregate
  }

  async save(aggregate: T): Promise<void> {
    const uncommitted = aggregate.uncommittedEvents
    if (uncommitted.length === 0) return

    const expectedVersion = aggregate.version - uncommitted.length

    // 1. 附加新事件
    await this.eventStore.append(aggregate.id, [...uncommitted], expectedVersion)

    // 2. 定期建立快照
    if (aggregate.version % SNAPSHOT_INTERVAL === 0) {
      await this.eventStore.saveSnapshot({
        aggregateId: aggregate.id,
        aggregateType: aggregate.constructor.name,
        version: aggregate.version,
        state: this.serializeState(aggregate),
        createdAt: new Date(),
      })
    }

    aggregate.clearUncommittedEvents()
  }

  private serializeState(aggregate: T): Record<string, unknown> {
    // 序列化聚合的可還原狀態（排除 id 與未提交事件清單）
    const { id, _uncommittedEvents, ...state } = JSON.parse(JSON.stringify(aggregate))
    return state
  }
}
```

### 與 CQRS 整合：事件投影

```typescript
// server/infrastructure/projections/OrderProjection.ts
import type { DomainEvent } from '~/server/domain/events/types'
import type { EventHandler } from '~/server/cqrs/events/EventBus'

/**
 * 事件投影：監聽事件並更新讀取模型（Read Model）
 * 讀取模型是為查詢最佳化的扁平化資料結構
 */
export class OrderProjection implements EventHandler {
  constructor(private readonly prisma: PrismaClient) {}

  async handle(event: DomainEvent): Promise<void> {
    switch (event.eventType) {
      case 'OrderCreated':
        await this.prisma.orderReadModel.create({
          data: {
            id: event.aggregateId,
            customerId: event.payload.customerId as string,
            status: 'pending',
            totalAmount: (event.payload.items as any[]).reduce(
              (sum: number, i: any) => sum + i.quantity * i.unitPrice, 0
            ),
            itemCount: (event.payload.items as any[]).length,
            createdAt: event.timestamp,
            updatedAt: event.timestamp,
          },
        })
        break

      case 'OrderConfirmed':
        await this.prisma.orderReadModel.update({
          where: { id: event.aggregateId },
          data: { status: 'confirmed', updatedAt: event.timestamp },
        })
        break

      case 'OrderCancelled':
        await this.prisma.orderReadModel.update({
          where: { id: event.aggregateId },
          data: {
            status: 'cancelled',
            cancelReason: event.payload.reason as string,
            updatedAt: event.timestamp,
          },
        })
        break
    }
  }
}
```

> **何時採用 Event Sourcing**：當需要完整審計軌跡（金融、醫療）、需要時間旅行除錯、或業務邏輯天然以事件描述（訂單生命週期、工作流程）時，Event Sourcing 帶來顯著價值。但它增加了儲存與查詢複雜度，小型 CRUD 應用不建議使用。

---

## Saga / Process Manager 模式

### 核心概念

Saga 模式用於管理跨多個服務或聚合的分散式交易。與傳統的兩階段提交（2PC）不同，Saga 透過一系列本地交易加上補償動作（Compensating Action）來維持最終一致性。當任一步驟失敗時，已完成的步驟會以相反順序執行補償，回滾至一致狀態。

```
正常流程：
  Step 1 ──▶ Step 2 ──▶ Step 3 ──▶ 完成 ✓

失敗補償（Step 3 失敗）：
  Step 1 ──▶ Step 2 ──▶ Step 3 ✗
                 │         │
                 ▼         ▼
          Compensate 2  Compensate 1  ──▶ 回滾完成
```

### Saga 步驟與協調器

```typescript
// server/domain/sagas/types.ts — Saga 基礎型別

export interface SagaStep<TContext> {
  /** 步驟名稱（用於日誌與錯誤追蹤） */
  name: string
  /** 正向執行：執行業務邏輯並回傳更新後的 context */
  execute(context: TContext): Promise<TContext>
  /** 補償動作：回滾此步驟的變更 */
  compensate(context: TContext): Promise<void>
}

export class SagaFailedError extends Error {
  constructor(
    public readonly failedStep: string,
    public readonly originalError: unknown,
    public readonly compensationErrors: Array<{ step: string; error: unknown }> = []
  ) {
    super(`Saga failed at step "${failedStep}": ${String(originalError)}`)
    this.name = 'SagaFailedError'
  }
}
```

```typescript
// server/domain/sagas/saga-orchestrator.ts — Saga 協調器
import type { SagaStep } from './types'
import { SagaFailedError } from './types'

export class SagaOrchestrator<TContext> {
  private steps: SagaStep<TContext>[] = []
  private completedSteps: SagaStep<TContext>[] = []

  addStep(step: SagaStep<TContext>): this {
    this.steps.push(step)
    return this
  }

  async execute(initialContext: TContext): Promise<TContext> {
    let context = initialContext
    this.completedSteps = []

    for (const step of this.steps) {
      try {
        context = await step.execute(context)
        this.completedSteps.push(step)
      } catch (error) {
        // 步驟失敗，啟動補償流程
        const compensationErrors = await this.compensate(context)
        throw new SagaFailedError(step.name, error, compensationErrors)
      }
    }

    return context
  }

  private async compensate(
    context: TContext
  ): Promise<Array<{ step: string; error: unknown }>> {
    const errors: Array<{ step: string; error: unknown }> = []

    // 以相反順序執行補償動作
    for (const step of [...this.completedSteps].reverse()) {
      try {
        await step.compensate(context)
      } catch (compensateError) {
        // 記錄補償錯誤但繼續補償其他步驟
        errors.push({ step: step.name, error: compensateError })
        console.error(
          `Compensation failed for step "${step.name}":`,
          compensateError
        )
      }
    }

    return errors
  }
}
```

### 具體範例：訂單處理 Saga

```typescript
// server/domain/sagas/order-saga.ts — 訂單建立 Saga
import type { SagaStep } from './types'
import { SagaOrchestrator } from './saga-orchestrator'

// Saga Context：在步驟間傳遞的共享狀態
interface OrderSagaContext {
  orderId: string
  customerId: string
  items: Array<{ productId: string; quantity: number; unitPrice: number }>
  totalAmount: number
  // 各步驟產生的結果
  paymentId?: string
  inventoryReservationId?: string
  shippingId?: string
}

// Step 1：扣款
const processPaymentStep: SagaStep<OrderSagaContext> = {
  name: 'ProcessPayment',
  async execute(context) {
    const paymentResult = await paymentService.charge({
      customerId: context.customerId,
      amount: context.totalAmount,
      orderId: context.orderId,
    })
    return { ...context, paymentId: paymentResult.paymentId }
  },
  async compensate(context) {
    if (context.paymentId) {
      await paymentService.refund(context.paymentId)
    }
  },
}

// Step 2：預留庫存
const reserveInventoryStep: SagaStep<OrderSagaContext> = {
  name: 'ReserveInventory',
  async execute(context) {
    const reservation = await inventoryService.reserve({
      items: context.items.map((i) => ({
        productId: i.productId,
        quantity: i.quantity,
      })),
      orderId: context.orderId,
    })
    return { ...context, inventoryReservationId: reservation.reservationId }
  },
  async compensate(context) {
    if (context.inventoryReservationId) {
      await inventoryService.releaseReservation(context.inventoryReservationId)
    }
  },
}

// Step 3：建立出貨單
const createShipmentStep: SagaStep<OrderSagaContext> = {
  name: 'CreateShipment',
  async execute(context) {
    const shipment = await shippingService.createShipment({
      orderId: context.orderId,
      customerId: context.customerId,
      items: context.items,
    })
    return { ...context, shippingId: shipment.shippingId }
  },
  async compensate(context) {
    if (context.shippingId) {
      await shippingService.cancelShipment(context.shippingId)
    }
  },
}

// 組裝 Saga
export function createOrderSaga(): SagaOrchestrator<OrderSagaContext> {
  return new SagaOrchestrator<OrderSagaContext>()
    .addStep(processPaymentStep)
    .addStep(reserveInventoryStep)
    .addStep(createShipmentStep)
}
```

### 在 API Handler 中使用 Saga

```typescript
// server/api/orders/index.post.ts — 使用 Order Saga
import { createOrderSaga, type OrderSagaContext } from '~/server/domain/sagas/order-saga'
import { SagaFailedError } from '~/server/domain/sagas/types'

export default defineEventHandler(async (event) => {
  const session = await requireUserSession(event)
  const body = await readBody(event)

  const sagaContext: OrderSagaContext = {
    orderId: randomUUID(),
    customerId: session.user.id,
    items: body.items,
    totalAmount: body.items.reduce(
      (sum: number, i: any) => sum + i.quantity * i.unitPrice, 0
    ),
  }

  try {
    const result = await createOrderSaga().execute(sagaContext)

    setResponseStatus(event, 201)
    return {
      orderId: result.orderId,
      paymentId: result.paymentId,
      shippingId: result.shippingId,
    }
  } catch (error) {
    if (error instanceof SagaFailedError) {
      console.error(`Order saga failed at "${error.failedStep}":`, error.originalError)

      if (error.compensationErrors.length > 0) {
        // 補償也失敗，需人工介入
        console.error('Compensation errors:', error.compensationErrors)
        throw createError({
          statusCode: 500,
          message: '訂單處理失敗，部分補償未完成，請聯繫客服',
        })
      }

      throw createError({
        statusCode: 422,
        message: `訂單處理失敗（步驟：${error.failedStep}），所有變更已回滾`,
      })
    }
    throw error
  }
})
```

### Saga 狀態持久化（進階）

對於長時間運行的 Saga（跨越多個 HTTP 請求），需要將 Saga 狀態持久化：

```typescript
// server/domain/sagas/persistent-saga.ts
interface SagaState<TContext> {
  sagaId: string
  sagaType: string
  context: TContext
  currentStep: number
  status: 'running' | 'completed' | 'compensating' | 'failed'
  completedSteps: string[]
  createdAt: Date
  updatedAt: Date
}

export class PersistentSagaOrchestrator<TContext> {
  constructor(
    private readonly steps: SagaStep<TContext>[],
    private readonly sagaStore: SagaStore
  ) {}

  async execute(sagaId: string, initialContext: TContext): Promise<TContext> {
    // 載入或建立 Saga 狀態
    let state = await this.sagaStore.load<TContext>(sagaId)
    if (!state) {
      state = {
        sagaId,
        sagaType: this.constructor.name,
        context: initialContext,
        currentStep: 0,
        status: 'running',
        completedSteps: [],
        createdAt: new Date(),
        updatedAt: new Date(),
      }
      await this.sagaStore.save(state)
    }

    // 從上次中斷處繼續執行
    for (let i = state.currentStep; i < this.steps.length; i++) {
      const step = this.steps[i]
      try {
        state.context = await step.execute(state.context)
        state.completedSteps.push(step.name)
        state.currentStep = i + 1
        state.updatedAt = new Date()
        await this.sagaStore.save(state)
      } catch (error) {
        state.status = 'compensating'
        await this.sagaStore.save(state)
        await this.compensateFromState(state)
        state.status = 'failed'
        await this.sagaStore.save(state)
        throw new SagaFailedError(step.name, error)
      }
    }

    state.status = 'completed'
    await this.sagaStore.save(state)
    return state.context
  }

  private async compensateFromState(state: SagaState<TContext>): Promise<void> {
    for (const stepName of [...state.completedSteps].reverse()) {
      const step = this.steps.find((s) => s.name === stepName)
      if (step) {
        await step.compensate(state.context)
      }
    }
  }
}
```

> **何時採用 Saga**：當業務流程跨越多個服務或聚合（例如「扣款 → 預留庫存 → 建立出貨單」），且每個步驟都需要可回滾時，Saga 是管理分散式交易一致性的標準模式。對於單一資料庫內的操作，直接使用資料庫交易即可，不需要 Saga。

---

## Value Objects（值物件）

### 核心概念

Value Object 是 DDD 中以「值」定義身份的物件——兩個 Value Object 只要屬性相同就視為相等，不需要唯一 ID。它們是不可變的（Immutable），任何修改都會回傳新實例。常見範例：Email、Money、Address、DateRange。

```
Entity（實體）                 Value Object（值物件）
┌─────────────────────┐     ┌─────────────────────┐
│ id: "user-123"      │     │ value: "a@b.com"    │
│ name: "Alice"       │     │                     │
│ email: Email("...") │     │ 無 id               │
│                     │     │ 以值判斷相等         │
│ 以 id 判斷相等       │     │ 不可變              │
│ 可變（有生命週期）    │     │ 可自由替換           │
└─────────────────────┘     └─────────────────────┘
```

### 基礎 ValueObject 抽象類別

```typescript
// server/domain/value-objects/base.ts — Value Object 基礎類別
export abstract class ValueObject<T extends Record<string, unknown>> {
  protected readonly props: Readonly<T>

  protected constructor(props: T) {
    this.validate(props)
    this.props = Object.freeze(props)
  }

  /** 子類別實作：驗證值的合法性，不合法則拋出錯誤 */
  protected abstract validate(props: T): void

  /** 以值判斷相等（非參照比較） */
  equals(other: ValueObject<T>): boolean {
    if (!(other instanceof this.constructor)) return false
    return JSON.stringify(this.props) === JSON.stringify(other.props)
  }

  /** 回傳原始值的不可變副本 */
  toJSON(): Readonly<T> {
    return this.props
  }
}
```

### 具體範例：Email

```typescript
// server/domain/value-objects/email.ts
import { ValueObject } from './base'

export class InvalidEmailError extends Error {
  constructor(value: string) {
    super(`Invalid email address: "${value}"`)
    this.name = 'InvalidEmailError'
  }
}

export class Email extends ValueObject<{ value: string }> {
  /** 工廠方法：統一建立流程（正規化 + 驗證） */
  static create(value: string): Email {
    return new Email({ value: value.toLowerCase().trim() })
  }

  protected validate({ value }: { value: string }): void {
    if (!value || value.length === 0) {
      throw new InvalidEmailError(value)
    }
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      throw new InvalidEmailError(value)
    }
    if (value.length > 254) {
      throw new InvalidEmailError(value)
    }
  }

  get value(): string {
    return this.props.value
  }

  get domain(): string {
    return this.props.value.split('@')[1]
  }
}
```

### 具體範例：Money

```typescript
// server/domain/value-objects/money.ts
import { ValueObject } from './base'

export class CurrencyMismatchError extends Error {
  constructor(a: string, b: string) {
    super(`Cannot operate on different currencies: ${a} and ${b}`)
    this.name = 'CurrencyMismatchError'
  }
}

export class InvalidMoneyError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'InvalidMoneyError'
  }
}

export class Money extends ValueObject<{ amount: number; currency: string }> {
  static create(amount: number, currency: string): Money {
    return new Money({ amount, currency: currency.toUpperCase() })
  }

  static zero(currency: string): Money {
    return Money.create(0, currency)
  }

  protected validate({ amount, currency }: { amount: number; currency: string }): void {
    if (!Number.isFinite(amount)) {
      throw new InvalidMoneyError(`Amount must be finite: ${amount}`)
    }
    if (!currency || currency.length !== 3) {
      throw new InvalidMoneyError(`Invalid currency code: ${currency}`)
    }
  }

  get amount(): number { return this.props.amount }
  get currency(): string { return this.props.currency }

  private assertSameCurrency(other: Money): void {
    if (this.props.currency !== other.props.currency) {
      throw new CurrencyMismatchError(this.props.currency, other.props.currency)
    }
  }

  /** 加法：回傳新的 Money 實例（不可變） */
  add(other: Money): Money {
    this.assertSameCurrency(other)
    return Money.create(this.props.amount + other.props.amount, this.props.currency)
  }

  /** 減法 */
  subtract(other: Money): Money {
    this.assertSameCurrency(other)
    return Money.create(this.props.amount - other.props.amount, this.props.currency)
  }

  /** 乘以倍數 */
  multiply(factor: number): Money {
    return Money.create(
      Math.round(this.props.amount * factor * 100) / 100,
      this.props.currency
    )
  }

  /** 是否為正數 */
  isPositive(): boolean {
    return this.props.amount > 0
  }

  /** 是否為零 */
  isZero(): boolean {
    return this.props.amount === 0
  }

  /** 比較大小 */
  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other)
    return this.props.amount > other.props.amount
  }

  /** 格式化顯示 */
  format(locale = 'zh-TW'): string {
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency: this.props.currency,
    }).format(this.props.amount)
  }
}
```

### 具體範例：DateRange

```typescript
// server/domain/value-objects/date-range.ts
import { ValueObject } from './base'

export class InvalidDateRangeError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'InvalidDateRangeError'
  }
}

export class DateRange extends ValueObject<{ start: string; end: string }> {
  static create(start: Date, end: Date): DateRange {
    return new DateRange({
      start: start.toISOString(),
      end: end.toISOString(),
    })
  }

  protected validate({ start, end }: { start: string; end: string }): void {
    const startDate = new Date(start)
    const endDate = new Date(end)

    if (isNaN(startDate.getTime())) {
      throw new InvalidDateRangeError(`Invalid start date: ${start}`)
    }
    if (isNaN(endDate.getTime())) {
      throw new InvalidDateRangeError(`Invalid end date: ${end}`)
    }
    if (startDate >= endDate) {
      throw new InvalidDateRangeError('Start date must be before end date')
    }
  }

  get start(): Date { return new Date(this.props.start) }
  get end(): Date { return new Date(this.props.end) }

  /** 計算持續天數 */
  get durationInDays(): number {
    const diff = this.end.getTime() - this.start.getTime()
    return Math.ceil(diff / (1000 * 60 * 60 * 24))
  }

  /** 檢查日期是否在範圍內 */
  contains(date: Date): boolean {
    return date >= this.start && date <= this.end
  }

  /** 檢查兩個範圍是否重疊 */
  overlaps(other: DateRange): boolean {
    return this.start < other.end && this.end > other.start
  }
}
```

### 在 Domain Entity 中使用 Value Objects

```typescript
// server/domain/entities/User.ts — Entity 組合 Value Objects
import { Email } from '../value-objects/email'
import { Money } from '../value-objects/money'

export class User {
  constructor(
    public readonly id: string,
    private _email: Email,
    private _name: string,
    private _balance: Money
  ) {}

  get email(): Email { return this._email }
  get name(): string { return this._name }
  get balance(): Money { return this._balance }

  changeEmail(newEmail: Email): void {
    // Email Value Object 已在建立時驗證合法性
    this._email = newEmail
  }

  deposit(amount: Money): void {
    if (!amount.isPositive()) {
      throw new Error('Deposit amount must be positive')
    }
    this._balance = this._balance.add(amount)
  }

  withdraw(amount: Money): void {
    if (!amount.isPositive()) {
      throw new Error('Withdrawal amount must be positive')
    }
    if (amount.isGreaterThan(this._balance)) {
      throw new Error('Insufficient balance')
    }
    this._balance = this._balance.subtract(amount)
  }
}
```

> **Value Object 的價值**：將驗證邏輯封裝在 Value Object 中，避免「原始型別偏執（Primitive Obsession）」。Email 不再是一個可能非法的 `string`，而是保證合法的 `Email` 物件。Money 不再是容易出錯的 `number`，而是具備幣別檢查與安全運算的 `Money` 物件。

---

## Command Bus / Mediator 模式

### 核心概念

Command Bus（又稱 Mediator）將「命令的發送」與「命令的處理」解耦。發送端只需知道 Command 的形狀，不需要知道由哪個 Handler 處理。這提供了集中化的關注點（日誌、驗證、權限、交易管理）注入點。

```
┌──────────┐    dispatch     ┌─────────────┐    route     ┌──────────────┐
│  API     │ ──────────────▶ │  Command    │ ──────────▶  │  Command     │
│  Handler │    Command      │  Bus        │              │  Handler     │
└──────────┘                 │             │              └──────────────┘
                             │ Middleware: │
                             │ - Logging   │
                             │ - Validation│
                             │ - Auth      │
                             └─────────────┘
```

### Command 與 Handler 介面

```typescript
// server/infrastructure/command-bus/types.ts — 基礎型別

/** Command：描述一個改變系統狀態的意圖 */
export interface Command {
  readonly type: string
}

/** CommandHandler：處理特定類型的 Command */
export interface CommandHandler<TCommand extends Command, TResult = void> {
  handle(command: TCommand): Promise<TResult>
}

/** Middleware：Command Bus 管線中的攔截器 */
export interface CommandMiddleware {
  execute<TResult>(
    command: Command,
    next: (command: Command) => Promise<TResult>
  ): Promise<TResult>
}
```

### Command Bus 實作

```typescript
// server/infrastructure/command-bus/command-bus.ts
import type { Command, CommandHandler, CommandMiddleware } from './types'

export class UnhandledCommandError extends Error {
  constructor(commandType: string) {
    super(`No handler registered for command: "${commandType}"`)
    this.name = 'UnhandledCommandError'
  }
}

export class CommandBus {
  private handlers = new Map<string, CommandHandler<any, any>>()
  private middlewares: CommandMiddleware[] = []

  /** 註冊 Command Handler */
  register<TCommand extends Command, TResult>(
    commandType: string,
    handler: CommandHandler<TCommand, TResult>
  ): void {
    if (this.handlers.has(commandType)) {
      throw new Error(`Handler already registered for command: "${commandType}"`)
    }
    this.handlers.set(commandType, handler)
  }

  /** 加入 Middleware（按加入順序執行） */
  use(middleware: CommandMiddleware): void {
    this.middlewares.push(middleware)
  }

  /** 發送 Command，透過 Middleware 管線路由至對應 Handler */
  async dispatch<TResult>(command: Command): Promise<TResult> {
    const handler = this.handlers.get(command.type)
    if (!handler) {
      throw new UnhandledCommandError(command.type)
    }

    // 建構 Middleware 管線（洋蔥模型）
    const execute = this.middlewares.reduceRight<
      (cmd: Command) => Promise<TResult>
    >(
      (next, middleware) => {
        return (cmd) => middleware.execute(cmd, next)
      },
      (cmd) => handler.handle(cmd)
    )

    return execute(command)
  }
}
```

### 常用 Middleware

```typescript
// server/infrastructure/command-bus/middlewares/logging.ts — 日誌 Middleware
import type { Command, CommandMiddleware } from '../types'

export class LoggingMiddleware implements CommandMiddleware {
  async execute<TResult>(
    command: Command,
    next: (command: Command) => Promise<TResult>
  ): Promise<TResult> {
    const start = performance.now()
    console.log(`[CommandBus] Dispatching: ${command.type}`, {
      payload: command,
    })

    try {
      const result = await next(command)
      const duration = Math.round(performance.now() - start)
      console.log(`[CommandBus] Completed: ${command.type} (${duration}ms)`)
      return result
    } catch (error) {
      const duration = Math.round(performance.now() - start)
      console.error(
        `[CommandBus] Failed: ${command.type} (${duration}ms)`,
        error
      )
      throw error
    }
  }
}
```

```typescript
// server/infrastructure/command-bus/middlewares/validation.ts — 驗證 Middleware
import type { Command, CommandMiddleware } from '../types'
import type { ZodSchema } from 'zod'

export class ValidationMiddleware implements CommandMiddleware {
  private schemas = new Map<string, ZodSchema>()

  registerSchema(commandType: string, schema: ZodSchema): void {
    this.schemas.set(commandType, schema)
  }

  async execute<TResult>(
    command: Command,
    next: (command: Command) => Promise<TResult>
  ): Promise<TResult> {
    const schema = this.schemas.get(command.type)
    if (schema) {
      const result = schema.safeParse(command)
      if (!result.success) {
        throw new CommandValidationError(command.type, result.error.issues)
      }
    }
    return next(command)
  }
}

export class CommandValidationError extends Error {
  constructor(
    public readonly commandType: string,
    public readonly issues: Array<{ message: string; path: (string | number)[] }>
  ) {
    super(`Validation failed for ${commandType}: ${issues.map((i) => i.message).join(', ')}`)
    this.name = 'CommandValidationError'
  }
}
```

### 定義具體 Command 與 Handler

```typescript
// server/application/commands/create-post.command.ts
import type { Command, CommandHandler } from '~/server/infrastructure/command-bus/types'
import type { PostRepository } from '~/server/domain/repositories/PostRepository'
import { Post } from '~/server/domain/entities/Post'

export interface CreatePostCommand extends Command {
  type: 'CreatePost'
  title: string
  content: string
  authorId: string
  tags: string[]
}

export class CreatePostCommandHandler
  implements CommandHandler<CreatePostCommand, { id: string; slug: string }>
{
  constructor(
    private readonly postRepo: PostRepository,
    private readonly slugGenerator: SlugGenerator
  ) {}

  async handle(command: CreatePostCommand): Promise<{ id: string; slug: string }> {
    const slug = await this.slugGenerator.generate(command.title)
    const post = Post.create({
      title: command.title,
      content: command.content,
      slug,
      authorId: command.authorId,
      tags: command.tags,
    })

    await this.postRepo.save(post)
    return { id: post.id, slug: post.slug }
  }
}
```

### Command Bus 初始化與容器整合

```typescript
// server/infrastructure/command-bus/setup.ts — 初始化 Command Bus
import { CommandBus } from './command-bus'
import { LoggingMiddleware } from './middlewares/logging'
import { ValidationMiddleware } from './middlewares/validation'
import { CreatePostCommandHandler } from '~/server/application/commands/create-post.command'
import { z } from 'zod'

export function createCommandBus(container: DIContainer): CommandBus {
  const bus = new CommandBus()

  // 1. 註冊 Middleware（依序執行）
  bus.use(new LoggingMiddleware())

  const validation = new ValidationMiddleware()
  validation.registerSchema(
    'CreatePost',
    z.object({
      type: z.literal('CreatePost'),
      title: z.string().min(1).max(200),
      content: z.string().min(10),
      authorId: z.string().uuid(),
      tags: z.array(z.string()).max(5),
    })
  )
  bus.use(validation)

  // 2. 註冊 Command Handlers
  bus.register(
    'CreatePost',
    new CreatePostCommandHandler(
      container.resolve('postRepository'),
      container.resolve('slugGenerator')
    )
  )

  return bus
}
```

### 在 API Handler 中使用 Command Bus

```typescript
// server/api/posts/index.post.ts — 透過 Command Bus 發送命令
export default defineEventHandler(async (event) => {
  const session = await requireUserSession(event)
  const body = await readBody(event)

  const commandBus = useContainerFromEvent(event).resolve('commandBus')

  try {
    const result = await commandBus.dispatch<{ id: string; slug: string }>({
      type: 'CreatePost',
      title: body.title,
      content: body.content,
      authorId: session.user.id,
      tags: body.tags ?? [],
    })

    setResponseStatus(event, 201)
    return result
  } catch (error) {
    if (error instanceof CommandValidationError) {
      throw createError({ statusCode: 400, message: error.message })
    }
    throw error
  }
})
```

### Command Bus 的測試

```typescript
// tests/unit/command-bus.test.ts
import { describe, it, expect, vi } from 'vitest'
import { CommandBus } from '~/server/infrastructure/command-bus/command-bus'
import { LoggingMiddleware } from '~/server/infrastructure/command-bus/middlewares/logging'

describe('CommandBus', () => {
  it('should dispatch command to registered handler', async () => {
    const bus = new CommandBus()
    const mockHandler = { handle: vi.fn().mockResolvedValue({ id: '123' }) }

    bus.register('TestCommand', mockHandler)

    const result = await bus.dispatch({ type: 'TestCommand', data: 'test' })

    expect(result).toEqual({ id: '123' })
    expect(mockHandler.handle).toHaveBeenCalledWith({
      type: 'TestCommand',
      data: 'test',
    })
  })

  it('should throw UnhandledCommandError for unregistered command', async () => {
    const bus = new CommandBus()

    await expect(
      bus.dispatch({ type: 'UnknownCommand' })
    ).rejects.toThrow('No handler registered for command: "UnknownCommand"')
  })

  it('should execute middlewares in order', async () => {
    const bus = new CommandBus()
    const order: string[] = []

    bus.use({
      async execute(cmd, next) {
        order.push('middleware-1-before')
        const result = await next(cmd)
        order.push('middleware-1-after')
        return result
      },
    })
    bus.use({
      async execute(cmd, next) {
        order.push('middleware-2-before')
        const result = await next(cmd)
        order.push('middleware-2-after')
        return result
      },
    })

    bus.register('Test', {
      async handle() {
        order.push('handler')
      },
    })

    await bus.dispatch({ type: 'Test' })

    expect(order).toEqual([
      'middleware-1-before',
      'middleware-2-before',
      'handler',
      'middleware-2-after',
      'middleware-1-after',
    ])
  })
})
```

> **何時採用 Command Bus**：當專案需要跨多個 Command 統一處理關注點（日誌、驗證、權限、交易）時，Command Bus 提供了乾淨的攔截機制。它與 Use Case 層互補——Use Case 封裝業務流程，Command Bus 負責調度與橫切關注點。對於少量 API 的小型專案，直接呼叫 Use Case 更簡潔。

---

## 架構模式關係圖

### Clean Architecture、Hexagonal Architecture 與 Onion Architecture

這三種架構模式的核心理念相同——**依賴反轉（Dependency Inversion）**，但各自的描述角度與強調重點不同。

```
┌─────────────────────────────────────────────────────────────────┐
│                     三種架構模式的共同核心                         │
│                                                                 │
│   「業務邏輯（Domain）不依賴任何外部技術細節，                      │
│    所有依賴方向由外向內指向核心。」                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Clean Architecture（Robert C. Martin, 2012）

以「同心圓」描述層級，強調**依賴規則（Dependency Rule）**：外層可依賴內層，內層絕不依賴外層。

```
┌─────────────────────────────────────────┐
│  Frameworks & Drivers（最外層）           │
│  ┌───────────────────────────────────┐  │
│  │  Interface Adapters（轉接層）       │  │
│  │  ┌───────────────────────────┐    │  │
│  │  │  Application Business     │    │  │
│  │  │  Rules（Use Cases）        │    │  │
│  │  │  ┌───────────────────┐   │    │  │
│  │  │  │  Enterprise       │   │    │  │
│  │  │  │  Business Rules   │   │    │  │
│  │  │  │  (Entities)       │   │    │  │
│  │  │  └───────────────────┘   │    │  │
│  │  └───────────────────────────┘    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**Nuxt 3 對應**：
- **Entities**：`server/domain/entities/` — 純 TypeScript class，無框架依賴
- **Use Cases**：`server/application/use-cases/` — 業務流程編排
- **Interface Adapters**：`server/api/`（Controller）、`server/infrastructure/`（Repository 實作）
- **Frameworks & Drivers**：Nuxt、Prisma、Redis SDK、H3

### Hexagonal Architecture（Alistair Cockburn, 2005）

又稱 **Ports & Adapters**。核心概念是 Application 透過「Port（介面）」與外界溝通，具體實作由「Adapter」負責。不區分層級數量，只區分「內部」與「外部」。

```
                    ┌──────────────┐
      Driving       │              │       Driven
      Adapters      │  Application │       Adapters
                    │    Core      │
  ┌──────────┐     │              │     ┌──────────────┐
  │ REST API │◄───►│  ┌────────┐  │◄───►│ PostgreSQL   │
  │ (H3)     │ Port│  │ Domain │  │Port │ (Prisma)     │
  └──────────┘     │  │ Logic  │  │     └──────────────┘
                    │  └────────┘  │
  ┌──────────┐     │              │     ┌──────────────┐
  │ CLI /    │◄───►│  ┌────────┐  │◄───►│ Redis Cache  │
  │ 排程任務  │ Port│  │  Use   │  │Port │              │
  └──────────┘     │  │ Cases  │  │     └──────────────┘
                    │  └────────┘  │
  ┌──────────┐     │              │     ┌──────────────┐
  │ GraphQL  │◄───►│              │◄───►│ Email (SMTP) │
  └──────────┘     │              │     └──────────────┘
                    └──────────────┘

  左側：Driving（主動觸發）        右側：Driven（被動呼叫）
  Port = interface                Adapter = 實作
```

**Nuxt 3 對應**：
- **Driving Adapters**（輸入端）：`server/api/` handlers、`server/cron/` 排程
- **Driving Ports**：Use Case 的 public interface
- **Application Core**：`server/domain/` + `server/application/`
- **Driven Ports**（輸出端介面）：`server/domain/repositories/` 中的 interface
- **Driven Adapters**（輸出端實作）：`server/infrastructure/` 中的 class

### Onion Architecture（Jeffrey Palermo, 2008）

與 Clean Architecture 非常相似，以「洋蔥」層層包裹的方式呈現。特別強調 **Domain Model 在最核心**，所有基礎設施在最外層。

```
┌──────────────────────────────────────────┐
│  Infrastructure（基礎設施）                │
│  ┌────────────────────────────────────┐  │
│  │  Application Services（應用服務）    │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │  Domain Services（領域服務）   │  │  │
│  │  │  ┌────────────────────────┐  │  │  │
│  │  │  │  Domain Model          │  │  │  │
│  │  │  │  (Entities + VOs)      │  │  │  │
│  │  │  └────────────────────────┘  │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

**Nuxt 3 對應**：
- **Domain Model**：`server/domain/entities/`、`server/domain/value-objects/`
- **Domain Services**：`server/domain/services/`（跨 Entity 的業務邏輯）
- **Application Services**：`server/application/use-cases/`
- **Infrastructure**：`server/infrastructure/`、`prisma/`、外部 SDK

### 三者比較

| 面向 | Clean Architecture | Hexagonal Architecture | Onion Architecture |
|------|-------------------|----------------------|-------------------|
| **提出者** | Robert C. Martin (2012) | Alistair Cockburn (2005) | Jeffrey Palermo (2008) |
| **核心隱喻** | 同心圓（4 層） | 六角形 + Ports/Adapters | 洋蔥（多層包裹） |
| **核心原則** | 依賴規則（外→內） | Port/Adapter 隔離 | 依賴方向外→內 |
| **層級定義** | 明確 4 層 | 僅內/外，不強制層數 | 明確多層 |
| **強調重點** | Use Case 驅動 | 可替換性（Adapter 隨插即用） | Domain Model 為核心 |
| **輸入/輸出** | 未特別區分 | 明確區分 Driving / Driven | 未特別區分 |
| **測試策略** | Mock 外層介面 | 替換 Adapter | Mock 外層 |
| **複雜度** | 中 | 中 | 中 |

### 在 Nuxt 3 中如何選擇

```
你的 Nuxt 3 專案需要什麼？
│
├── 清晰的業務用例驅動開發？
│   └── ✅ Clean Architecture
│       重點放在 Use Case 層，
│       適合大部分企業應用
│
├── 高度可替換的外部依賴？
│   └── ✅ Hexagonal Architecture
│       多種輸入管道（REST + GraphQL + CLI）
│       或頻繁更換基礎設施（換 DB、換快取）
│
├── 極致的 Domain Model 保護？
│   └── ✅ Onion Architecture
│       複雜業務規則、DDD 導向開發
│
└── 小型 CRUD 專案？
    └── ⚠️ 以上皆不需要
        直接在 server/api/ + composables/ 開發即可
        過度架構反而增加不必要的複雜度
```

### 實務建議

| 專案規模 | 推薦方式 | Nuxt 3 實作 |
|---------|---------|------------|
| **小型**（< 10 API routes） | 無額外架構 | `server/api/` 直接撰寫邏輯 |
| **中型**（10-50 API routes） | 簡化 Clean Architecture | 加入 `server/application/` Use Case 層 |
| **大型**（> 50 API routes） | 完整 Clean / Hexagonal | 完整分層 + DI 容器 + Port/Adapter |
| **DDD 專案** | Onion + Hexagonal 混合 | 完整 Domain Model + Bounded Context |

> **核心共識**：三種模式的目標一致——將業務邏輯與技術細節解耦。選擇哪一種取決於團隊熟悉度與專案需求。在 Nuxt 3 中，最常見的是 **Clean Architecture 的簡化版**：Domain Entity + Use Case + Repository Interface，搭配 Prisma 作為 Infrastructure Adapter。不要為了架構而架構——先用最簡單的方式解決問題，當複雜度增長時再逐步引入分層。
