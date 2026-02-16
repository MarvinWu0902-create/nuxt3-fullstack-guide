# 伺服器端 API 設計

## 目錄

1. [H3/Nitro API 設計](#h3nitro-api-設計)
2. [伺服器中介層](#伺服器中介層)
3. [驗證與安全性](#驗證與安全性)
4. [檔案上傳](#檔案上傳)
5. [資料庫整合](#資料庫整合)
6. [Prisma 最佳化](#prisma-最佳化)
7. [WebSocket 與 Server-Sent Events](#websocket-與-server-sent-events)
8. [背景任務與排程](#背景任務與排程)

---

## H3/Nitro API 設計

### API Route 結構

```
server/
├── api/
│   ├── auth/
│   │   ├── login.post.ts      # POST /api/auth/login
│   │   ├── register.post.ts   # POST /api/auth/register
│   │   ├── me.get.ts          # GET /api/auth/me
│   │   └── logout.post.ts     # POST /api/auth/logout
│   ├── users/
│   │   ├── index.get.ts       # GET /api/users
│   │   ├── index.post.ts      # POST /api/users
│   │   ├── [id].get.ts        # GET /api/users/:id
│   │   ├── [id].put.ts        # PUT /api/users/:id
│   │   └── [id].delete.ts     # DELETE /api/users/:id
│   └── health.get.ts          # GET /api/health
├── middleware/
│   └── auth.ts                # 全域或路由級伺服器中介層
├── utils/
│   ├── db.ts                  # 資料庫連線
│   └── validators.ts          # 輸入驗證
└── plugins/
    └── db.ts                  # 資料庫初始化
```

### RESTful API Handler 範例

```typescript
// server/api/users/index.get.ts
export default defineEventHandler(async (event) => {
  // 取得查詢參數
  const query = getQuery(event)
  const page = Number(query.page) || 1
  const perPage = Math.min(Number(query.perPage) || 20, 100)
  // 白名單驗證 sort 欄位，防止注入
  const allowedSortFields = ['createdAt', 'name', 'email', 'updatedAt']
  const rawSort = Array.isArray(query.sort) ? query.sort[0] : query.sort
  const sort = (typeof rawSort === 'string' && allowedSortFields.includes(rawSort)) ? rawSort : 'createdAt'
  const order = query.order === 'asc' ? 'asc' : 'desc'

  // 從資料庫查詢（示範）
  const offset = (page - 1) * perPage
  const [users, total] = await Promise.all([
    db.user.findMany({
      skip: offset,
      take: perPage,
      orderBy: { [sort]: order }
    }),
    db.user.count()
  ])

  return {
    data: users,
    meta: {
      total,
      page,
      perPage,
      totalPages: Math.ceil(total / perPage)
    }
  }
})
```

```typescript
// server/api/users/index.post.ts
export default defineEventHandler(async (event) => {
  // 讀取並驗證 body
  const body = await readBody(event)
  const validated = validateUserInput(body)

  const user = await db.user.create({
    data: validated
  })

  setResponseStatus(event, 201)
  return { data: user }
})
```

```typescript
// server/api/users/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = Number(getRouterParam(event, 'id'))
  if (Number.isNaN(id)) {
    throw createError({ statusCode: 400, message: '無效的用戶 ID' })
  }

  const user = await db.user.findUnique({
    where: { id }
  })

  if (!user) {
    throw createError({ statusCode: 404, message: '找不到此用戶' })
  }

  return { data: user }
})
```

### 錯誤處理模式

```typescript
// server/utils/errors.ts
export function notFound(resource: string) {
  throw createError({
    statusCode: 404,
    message: `找不到${resource}`
  })
}

export function unauthorized(message = '未經授權') {
  throw createError({
    statusCode: 401,
    message
  })
}

export function forbidden(message = '權限不足') {
  throw createError({
    statusCode: 403,
    message
  })
}

export function badRequest(message: string, data?: Record<string, string[]>) {
  throw createError({
    statusCode: 400,
    message,
    data
  })
}
```

---

## 伺服器中介層

### 認證中介層

```typescript
// server/middleware/auth.ts
export default defineEventHandler(async (event) => {
  // 僅對 /api/ 路由生效（排除公開端點）
  const url = getRequestURL(event)
  const publicPaths = ['/api/auth/login', '/api/auth/register', '/api/auth/refresh', '/api/health']

  if (!url.pathname.startsWith('/api/') || publicPaths.includes(url.pathname)) {
    return
  }

  const authorization = getHeader(event, 'authorization')
  if (!authorization?.startsWith('Bearer ')) {
    throw createError({ statusCode: 401, message: '缺少認證 Token' })
  }

  const token = authorization.slice(7)
  try {
    const payload = await verifyJWT(token)
    // 將用戶資訊存到 event context
    event.context.auth = {
      userId: payload.sub,
      role: payload.role
    }
  } catch {
    throw createError({ statusCode: 401, message: 'Token 已過期或無效' })
  }
})
```

### 擴展 H3EventContext 型別

```typescript
// server/types/auth.d.ts — 為 event.context.auth 提供型別安全
declare module 'h3' {
  interface H3EventContext {
    auth?: {
      userId: number
      role: 'admin' | 'user' | 'moderator'
    }
  }
}
export {}
```

### 使用 event context

```typescript
// server/api/users/me.get.ts
export default defineEventHandler(async (event) => {
  // 從中介層設定的 context 取得認證資訊（型別由上方 H3EventContext 擴展提供）
  const auth = event.context.auth
  if (!auth) throw createError({ statusCode: 401, message: '未認證' })
  const { userId } = auth

  const user = await db.user.findUnique({
    where: { id: userId }
  })

  return { data: user }
})
```

### 速率限制

```typescript
// server/middleware/rate-limit.ts
const rateLimitMap = new Map<string, { count: number; resetAt: number }>()

// 定期清理過期記錄，防止記憶體洩漏
const CLEANUP_INTERVAL = 5 * 60 * 1000  // 每 5 分鐘清理
setInterval(() => {
  const now = Date.now()
  for (const [ip, record] of rateLimitMap) {
    if (now > record.resetAt) rateLimitMap.delete(ip)
  }
}, CLEANUP_INTERVAL)

export default defineEventHandler((event) => {
  if (!getRequestURL(event).pathname.startsWith('/api/')) return

  // 注意：xForwardedFor: true 僅在受信任的反向代理後方才安全
  // 若應用直接暴露在網際網路上，請設為 false
  const ip = getRequestIP(event, { xForwardedFor: true }) ?? 'unknown'
  const now = Date.now()
  const windowMs = 60 * 1000  // 1 分鐘
  const maxRequests = 100     // 每分鐘最多 100 次

  const record = rateLimitMap.get(ip)

  if (!record || now > record.resetAt) {
    rateLimitMap.set(ip, { count: 1, resetAt: now + windowMs })
    return
  }

  if (record.count >= maxRequests) {
    setResponseHeader(event, 'Retry-After', String(Math.ceil((record.resetAt - now) / 1000)))
    throw createError({ statusCode: 429, message: '請求過於頻繁，請稍後再試' })
  }

  record.count++
})
```

> **注意**：此為單進程記憶體方案。多進程/多實例部署、以及 Serverless 環境（Vercel、AWS Lambda 等）皆不適用，因為函式實例隨時可能被回收重建。這些環境請使用 Redis/Upstash 等外部存儲做速率限制。

---

## 驗證與安全性

### 輸入驗證（使用 zod）

```typescript
// server/utils/validators.ts
import { z } from 'zod'

export const loginSchema = z.object({
  email: z.string().email('請輸入有效的電子郵件'),
  password: z.string().min(8, '密碼至少 8 個字元')
})

export const createUserSchema = z.object({
  name: z.string().min(2, '名稱至少 2 個字元').max(50),
  email: z.string().email('請輸入有效的電子郵件'),
  password: z.string()
    .min(8, '密碼至少 8 個字元')
    .regex(/[A-Z]/, '密碼需含至少一個大寫字母')
    .regex(/[0-9]/, '密碼需含至少一個數字'),
  role: z.enum(['user', 'admin']).default('user')
})

export const paginationSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  perPage: z.coerce.number().int().min(1).max(100).default(20),
  sort: z.string().optional(),
  order: z.enum(['asc', 'desc']).default('desc')
})

// 通用驗證 helper
export async function validateBody<T extends z.ZodSchema>(
  event: H3Event,
  schema: T
): Promise<z.infer<T>> {
  const body = await readBody(event)
  const result = schema.safeParse(body)

  if (!result.success) {
    const fieldErrors: Record<string, string[]> = {}
    for (const issue of result.error.issues) {
      const path = issue.path.join('.')
      if (!fieldErrors[path]) fieldErrors[path] = []
      fieldErrors[path].push(issue.message)
    }
    throw createError({
      statusCode: 400,
      message: '輸入驗證失敗',
      data: fieldErrors
    })
  }

  return result.data
}
```

```typescript
// 使用方式
// server/api/auth/login.post.ts
export default defineEventHandler(async (event) => {
  const { email, password } = await validateBody(event, loginSchema)
  // 繼續處理...
})
```

### CORS 設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/api/**': {
      cors: true,
      headers: {
        'Access-Control-Allow-Methods': 'GET,HEAD,PUT,PATCH,POST,DELETE',
        // 重要：此值在 BUILD TIME 決定，部署後無法更改
        // 需指定明確的 origin；若需 runtime 動態 CORS，請改用 server middleware + useRuntimeConfig(event)
        'Access-Control-Allow-Origin': process.env.CORS_ORIGIN ?? 'http://localhost:3000',
        'Access-Control-Allow-Credentials': 'true'
      }
    }
  }
})
```

> **安全警告**：`Access-Control-Allow-Origin: *` 搭配 `Access-Control-Allow-Credentials: true` 違反 CORS 規範，瀏覽器會直接拒絕。務必在生產環境設定明確的 `CORS_ORIGIN`。

---

## 檔案上傳

### 使用 readMultipartFormData 處理上傳

```typescript
// server/api/upload.post.ts
export default defineEventHandler(async (event) => {
  const files = await readMultipartFormData(event)
  if (!files || files.length === 0) {
    throw createError({ statusCode: 400, statusMessage: '未提供檔案' })
  }

  const file = files[0]
  const maxSize = 5 * 1024 * 1024 // 5MB
  if (file.data.length > maxSize) {
    throw createError({ statusCode: 413, statusMessage: '檔案大小超過限制' })
  }

  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp']
  if (!allowedTypes.includes(file.type || '')) {
    throw createError({ statusCode: 415, statusMessage: '不支援的檔案格式' })
  }

  // 安全的檔名清理
  const safeName = `${Date.now()}-${file.filename?.replace(/[^a-zA-Z0-9.-]/g, '_')}`

  // 儲存檔案（使用 Nitro 的 useStorage）
  const storage = useStorage('assets:uploads')
  await storage.setItemRaw(safeName, file.data)

  return { url: `/uploads/${safeName}` }
})
```

### 客戶端上傳實作

```vue
<template>
  <div>
    <input type="file" @change="handleFileChange" accept="image/*" />
    <button @click="upload" :disabled="!file || uploading">
      {{ uploading ? '上傳中...' : '上傳' }}
    </button>
    <div v-if="uploadedUrl">
      <p>上傳成功！</p>
      <img :src="uploadedUrl" alt="已上傳圖片" style="max-width: 300px" />
    </div>
  </div>
</template>

<script setup lang="ts">
const file = ref<File | null>(null)
const uploading = ref(false)
const uploadedUrl = ref<string>('')

function handleFileChange(event: Event) {
  const target = event.target as HTMLInputElement
  if (target.files && target.files.length > 0) {
    file.value = target.files[0]
  }
}

async function upload() {
  if (!file.value) return

  uploading.value = true
  try {
    const formData = new FormData()
    formData.append('file', file.value)

    const result = await $fetch<{ url: string }>('/api/upload', {
      method: 'POST',
      body: formData
    })

    uploadedUrl.value = result.url
  } catch (error) {
    console.error('上傳失敗:', error)
  } finally {
    uploading.value = false
  }
}
</script>
```

### 多檔案上傳與驗證

```typescript
// server/api/upload/multiple.post.ts
export default defineEventHandler(async (event) => {
  const files = await readMultipartFormData(event)
  if (!files || files.length === 0) {
    throw createError({ statusCode: 400, statusMessage: '未提供檔案' })
  }

  const maxFiles = 10
  if (files.length > maxFiles) {
    throw createError({ statusCode: 400, statusMessage: `最多只能上傳 ${maxFiles} 個檔案` })
  }

  const maxSize = 5 * 1024 * 1024 // 5MB
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf']
  const storage = useStorage('assets:uploads')
  const uploadedFiles: Array<{ name: string; url: string; size: number }> = []

  for (const file of files) {
    // 檔案大小驗證
    if (file.data.length > maxSize) {
      throw createError({
        statusCode: 413,
        statusMessage: `檔案 ${file.filename} 超過大小限制`
      })
    }

    // MIME 類型驗證
    if (!allowedTypes.includes(file.type || '')) {
      throw createError({
        statusCode: 415,
        statusMessage: `檔案 ${file.filename} 格式不支援`
      })
    }

    // 清理檔名
    const safeName = `${Date.now()}-${Math.random().toString(36).substring(7)}-${file.filename?.replace(/[^a-zA-Z0-9.-]/g, '_')}`

    await storage.setItemRaw(safeName, file.data)

    uploadedFiles.push({
      name: file.filename || safeName,
      url: `/uploads/${safeName}`,
      size: file.data.length
    })
  }

  return { files: uploadedFiles }
})
```

### 設定靜態資源服務

```typescript
// nuxt.config.ts — 設定 uploads 目錄為靜態資源
export default defineNuxtConfig({
  nitro: {
    storage: {
      'assets:uploads': {
        driver: 'fs',
        base: './.data/uploads'
      }
    },
    publicAssets: [
      {
        dir: './.data/uploads',
        maxAge: 60 * 60 * 24 * 365, // 1 年
        baseURL: '/uploads'
      }
    ]
  }
})
```

> **安全提醒**：
> - 永遠驗證檔案大小與 MIME 類型
> - 使用安全的檔名清理策略，防止路徑穿越攻擊
> - 考慮使用雲端儲存（S3、Cloudflare R2）處理大量檔案
> - 對圖片進行額外驗證（使用 sharp 等函式庫檢查實際內容）
> - 在生產環境使用病毒掃描服務

---

## 資料庫整合

### Prisma 整合

```typescript
// server/utils/db.ts
import { PrismaClient } from '@prisma/client'

// 防止 HMR 時建立多個連線
declare global {
  var __prisma: PrismaClient | undefined
}

function createPrismaClient() {
  const client = new PrismaClient({
    log: process.env.NODE_ENV === 'development'
      ? ['query', 'warn', 'error']
      : ['error']
  })
  return client
}

export const db = globalThis.__prisma ?? createPrismaClient()

if (process.env.NODE_ENV !== 'production') {
  globalThis.__prisma = db
}
```

### Drizzle ORM 整合

```typescript
// server/utils/db.ts
import { drizzle } from 'drizzle-orm/better-sqlite3'
import Database from 'better-sqlite3'
import * as schema from '~/server/database/schema'

const sqlite = new Database('./data/db.sqlite')
export const db = drizzle(sqlite, { schema })
```

---

## Prisma 最佳化

### 連線池與單例模式

```typescript
// server/utils/db.ts
import { PrismaClient } from '@prisma/client'

// Nitro 環境下的單例模式（含 HMR 保護）
const globalForPrisma = globalThis as unknown as { __prisma?: PrismaClient }

// 開發時從 globalThis 復用既有連線，避免 HMR 造成連線洩漏
let prisma: PrismaClient = globalForPrisma.__prisma!

export function usePrisma() {
  if (!prisma) {
    prisma = new PrismaClient({
      log: import.meta.dev ? ['query', 'warn', 'error'] : ['error'],
      datasources: {
        db: {
          url: useRuntimeConfig().databaseUrl
        }
      }
    })
    // 開發時存入 globalThis，HMR 重載後可復用
    if (import.meta.dev) {
      globalForPrisma.__prisma = prisma
    }
  }
  return prisma
}
```

### 使用 Prisma Extensions 擴展功能

```typescript
// server/utils/db.ts — 使用 $extends 添加功能
import { PrismaClient } from '@prisma/client'

const basePrisma = new PrismaClient()

export const prisma = basePrisma.$extends({
  query: {
    $allOperations({ operation, args, query }) {
      const start = performance.now()
      return query(args).finally(() => {
        const duration = performance.now() - start
        if (duration > 100) {
          console.warn(`慢查詢 [${operation}]: ${duration.toFixed(1)}ms`)
        }
      })
    }
  }
})

export function usePrisma() {
  return prisma
}
```

### 自動軟刪除擴展

```typescript
// server/utils/db.ts — 軟刪除功能
import { PrismaClient, Prisma } from '@prisma/client'

const basePrisma = new PrismaClient()

/** 透過 DMMF 檢查模型是否有指定欄位 */
function modelHasField(modelName: string, fieldName: string): boolean {
  const model = Prisma.dmmf.datamodel.models.find(m => m.name === modelName)
  return model?.fields.some(f => f.name === fieldName) ?? false
}

export const prisma = basePrisma.$extends({
  query: {
    // 為所有有 deletedAt 欄位的模型啟用軟刪除
    $allModels: {
      async delete({ model, args, query }) {
        if (modelHasField(model, 'deletedAt')) {
          const delegate = basePrisma[model.charAt(0).toLowerCase() + model.slice(1) as keyof typeof basePrisma] as any
          return delegate.update({
            ...args,
            data: { deletedAt: new Date() }
          })
        }
        return query(args)
      },
      async findMany({ model, args, query }) {
        if (modelHasField(model, 'deletedAt')) {
          args.where = { ...args.where, deletedAt: null }
        }
        return query(args)
      }
    }
  }
})
```

### 連線池設定（生產環境）

```env
# .env.production
# PostgreSQL 連線池設定
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public&connection_limit=5&pool_timeout=20"

# Serverless 環境建議使用連線池服務
# DATABASE_URL="prisma://accelerate.prisma-data.net/?api_key=xxx"
```

### 查詢效能最佳化

```typescript
// server/api/users/[id]/posts.get.ts
export default defineEventHandler(async (event) => {
  const userId = Number(getRouterParam(event, 'id'))
  const db = usePrisma()

  // ✅ 好：使用 select 只取需要的欄位
  const user = await db.user.findUnique({
    where: { id: userId },
    select: {
      id: true,
      name: true,
      posts: {
        select: {
          id: true,
          title: true,
          createdAt: true
        },
        orderBy: { createdAt: 'desc' },
        take: 10
      }
    }
  })

  // ❌ 壞：取得所有欄位（包含不需要的資料）
  // const user = await db.user.findUnique({
  //   where: { id: userId },
  //   include: { posts: true }
  // })

  return user
})
```

### 批次操作與交易

```typescript
// server/api/batch/users.post.ts
export default defineEventHandler(async (event) => {
  const db = usePrisma()
  const users = await readBody<Array<{ name: string; email: string }>>(event)

  // 使用交易確保原子性
  const result = await db.$transaction(async (tx) => {
    const created = []

    for (const userData of users) {
      const user = await tx.user.create({
        data: userData
      })
      created.push(user)
    }

    // 若其中一個失敗，全部回滾
    return created
  })

  return { count: result.length, users: result }
})
```

```typescript
// server/api/batch/users-bulk.post.ts — 批次建立（更快）
export default defineEventHandler(async (event) => {
  const db = usePrisma()
  const users = await readBody<Array<{ name: string; email: string }>>(event)

  // createMany 比迴圈 create 快得多
  const result = await db.user.createMany({
    data: users,
    skipDuplicates: true // 跳過重複的 email
  })

  return { count: result.count }
})
```

### CI/CD 中的資料庫遷移

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Generate Prisma Client
        run: npx prisma generate

      - name: Run database migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: Build application
        run: pnpm build
```

### Prisma Accelerate（連線池服務）

```typescript
// nuxt.config.ts — 使用 Prisma Accelerate
export default defineNuxtConfig({
  runtimeConfig: {
    // 使用 Prisma Accelerate 的連線字串
    databaseUrl: process.env.DATABASE_URL // prisma://accelerate...
  }
})
```

```typescript
// server/utils/db.ts
import { PrismaClient } from '@prisma/client'
import { withAccelerate } from '@prisma/extension-accelerate'

const basePrisma = new PrismaClient()

export const prisma = basePrisma.$extends(withAccelerate())

// 使用快取
export default defineEventHandler(async () => {
  const users = await prisma.user.findMany({
    cacheStrategy: {
      ttl: 60, // 快取 60 秒
      swr: 30  // Stale-While-Revalidate
    }
  })
  return users
})
```

> **生產環境建議**：
> - 使用 Prisma Accelerate 處理 Serverless 環境的連線池問題
> - 啟用查詢日誌監控慢查詢（duration > 100ms）
> - 定期執行 `prisma db pull` 確保 schema 與資料庫同步
> - 使用 `prisma migrate deploy` 而非 `migrate dev` 在生產環境部署

---

## WebSocket 與 Server-Sent Events

### WebSocket 設定

```typescript
// nuxt.config.ts — 啟用 WebSocket 實驗性功能
export default defineNuxtConfig({
  nitro: {
    experimental: {
      websocket: true
    }
  }
})
```

### WebSocket Handler 實作

```typescript
// server/routes/_ws.ts
export default defineWebSocketHandler({
  open(peer) {
    console.log('[ws] 連線開啟:', peer.id)
    peer.subscribe('chat')
    // 發送歡迎訊息
    peer.send(JSON.stringify({ type: 'welcome', message: '歡迎加入聊天室' }))
  },

  message(peer, message) {
    const text = message.text()
    console.log('[ws] 收到訊息:', text)

    try {
      const data = JSON.parse(text)

      // 廣播給同頻道的所有連線
      peer.publish('chat', JSON.stringify({
        type: 'message',
        from: peer.id,
        content: data.content,
        timestamp: Date.now()
      }))

      // 回傳確認給發送者
      peer.send(JSON.stringify({ type: 'ack', messageId: data.id }))
    } catch (error) {
      peer.send(JSON.stringify({ type: 'error', message: '訊息格式錯誤' }))
    }
  },

  close(peer, event) {
    console.log('[ws] 連線關閉:', peer.id, event.code, event.reason)
    // 通知其他用戶
    peer.publish('chat', JSON.stringify({
      type: 'user-left',
      userId: peer.id,
      timestamp: Date.now()
    }))
  },

  error(peer, error) {
    console.error('[ws] 錯誤:', peer.id, error)
  }
})
```

### 客戶端 WebSocket Composable

```typescript
// composables/useWebSocket.ts
export function useWebSocket(url: string) {
  const isConnected = ref(false)
  const messages = ref<Array<{ type: string; content: string; timestamp: number }>>([])
  let ws: WebSocket | null = null

  function connect() {
    if (!import.meta.client) return

    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:'
    const wsUrl = `${protocol}//${window.location.host}${url}`

    ws = new WebSocket(wsUrl)

    ws.onopen = () => {
      isConnected.value = true
      console.log('[ws] 已連線')
    }

    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data)
        if (data.type === 'message') {
          messages.value.push(data)
        }
      } catch (error) {
        console.error('[ws] 訊息解析失敗:', error)
      }
    }

    ws.onerror = (error) => {
      console.error('[ws] WebSocket 錯誤:', error)
    }

    ws.onclose = () => {
      isConnected.value = false
      console.log('[ws] 連線已關閉')
      // 僅在非主動斷線時自動重連
      if (!intentionalClose) {
        reconnectTimer = window.setTimeout(connect, 3000)
      }
    }
  }

  function send(content: string) {
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({
        id: Date.now(),
        content
      }))
    }
  }

  let intentionalClose = false
  let reconnectTimer: ReturnType<typeof setTimeout> | null = null

  function disconnect() {
    intentionalClose = true
    if (reconnectTimer) clearTimeout(reconnectTimer)
    if (ws) {
      ws.close()
      ws = null
    }
  }

  onMounted(connect)
  onUnmounted(disconnect)

  return {
    isConnected: readonly(isConnected),
    messages: readonly(messages),
    send,
    disconnect
  }
}
```

### Server-Sent Events (SSE) 實作

```typescript
// server/api/events.get.ts
export default defineEventHandler(async (event) => {
  const stream = createEventStream(event)

  // 定期推送事件
  const interval = setInterval(async () => {
    await stream.push({
      event: 'ping',
      data: JSON.stringify({ time: Date.now() })
    })
  }, 1000)

  // 監聽自訂事件（例如資料庫更新）
  const updateListener = async (data: any) => {
    await stream.push({
      event: 'update',
      data: JSON.stringify(data)
    })
  }

  // 假設有一個事件系統
  // eventBus.on('data-updated', updateListener)

  stream.onClosed(() => {
    clearInterval(interval)
    // eventBus.off('data-updated', updateListener)
    console.log('[sse] 客戶端斷線')
  })

  return stream.send()
})
```

### 即時通知系統範例

```typescript
// server/api/notifications/stream.get.ts
interface Notification {
  id: string
  userId: number
  type: 'info' | 'warning' | 'error'
  message: string
  timestamp: number
}

export default defineEventHandler(async (event) => {
  // 驗證使用者
  const auth = event.context.auth
  if (!auth) {
    throw createError({ statusCode: 401, message: '未認證' })
  }

  const stream = createEventStream(event)

  // 發送初始連線確認
  await stream.push({
    event: 'connected',
    data: JSON.stringify({ userId: auth.userId })
  })

  // 模擬即時通知（實際應用中應從 Redis/消息隊列訂閱）
  const interval = setInterval(async () => {
    const notification: Notification = {
      id: Math.random().toString(36).substring(7),
      userId: auth.userId,
      type: 'info',
      message: `通知時間：${new Date().toLocaleTimeString('zh-TW')}`,
      timestamp: Date.now()
    }

    await stream.push({
      event: 'notification',
      data: JSON.stringify(notification)
    })
  }, 10000) // 每 10 秒

  stream.onClosed(() => {
    clearInterval(interval)
  })

  return stream.send()
})
```

### 客戶端 SSE Composable

```typescript
// composables/useSSE.ts
export function useSSE(url: string) {
  const data = ref<string>('')
  const status = ref<'connecting' | 'open' | 'closed'>('connecting')
  const notifications = ref<Array<any>>([])
  let eventSource: EventSource | null = null

  function connect() {
    if (!import.meta.client) return

    eventSource = new EventSource(url)

    eventSource.onopen = () => {
      status.value = 'open'
      console.log('[sse] 已連線')
    }

    eventSource.onmessage = (e) => {
      data.value = e.data
    }

    // 監聽自訂事件
    eventSource.addEventListener('notification', (e) => {
      try {
        const notification = JSON.parse(e.data)
        notifications.value.push(notification)
      } catch (error) {
        console.error('[sse] 解析通知失敗:', error)
      }
    })

    eventSource.addEventListener('ping', (e) => {
      const data = JSON.parse(e.data)
      console.log('[sse] ping:', data.time)
    })

    eventSource.onerror = () => {
      status.value = 'closed'
      eventSource?.close()
      console.error('[sse] 連線錯誤')
    }
  }

  function close() {
    eventSource?.close()
    status.value = 'closed'
  }

  onMounted(connect)
  onUnmounted(close)

  return {
    data: readonly(data),
    status: readonly(status),
    notifications: readonly(notifications),
    close
  }
}
```

### 在頁面中使用 SSE

```vue
<template>
  <div>
    <div class="status">
      狀態: <span :class="status">{{ status }}</span>
    </div>
    <div class="notifications">
      <h3>即時通知</h3>
      <ul>
        <li v-for="notif in notifications" :key="notif.id">
          <span class="badge" :class="notif.type">{{ notif.type }}</span>
          {{ notif.message }}
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup lang="ts">
const { status, notifications } = useSSE('/api/notifications/stream')
</script>
```

> **部署注意事項**：
> - **WebSocket** 需要持久連線的伺服器環境（Node.js、Docker、VM），不支援純 Serverless 環境（Vercel、AWS Lambda 等）
> - **Cloudflare Workers** 支援 WebSocket，但需使用 Durable Objects
> - **SSE** 在大部分環境皆可運作，包含 Serverless（但連線會受函式執行時間限制）
> - 生產環境建議使用 Redis Pub/Sub 或專門的即時服務（Pusher、Ably、Socket.io）

---

## 背景任務與排程

### 啟用 Nitro Tasks

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    experimental: {
      tasks: true
    },
    scheduledTasks: {
      // Cron 語法：分 時 日 月 週
      // 每天凌晨 3 點清理過期 session
      '0 3 * * *': ['session:cleanup'],
      // 每 5 分鐘同步外部資料
      '*/5 * * * *': ['data:sync'],
      // 每小時執行報表生成
      '0 * * * *': ['reports:generate'],
      // 每週一早上 9 點發送週報
      '0 9 * * 1': ['reports:weekly']
    }
  }
})
```

### 定義背景任務

```typescript
// server/tasks/session/cleanup.ts
export default defineTask({
  meta: {
    name: 'session:cleanup',
    description: '清理過期的使用者 session'
  },
  async run() {
    console.log('[task] 開始清理過期 session...')
    const db = usePrisma()

    const deleted = await db.session.deleteMany({
      where: {
        expiresAt: { lt: new Date() }
      }
    })

    console.log(`[task] 已清理 ${deleted.count} 個過期 session`)
    return { result: `已清理 ${deleted.count} 個過期 session` }
  }
})
```

```typescript
// server/tasks/data/sync.ts
export default defineTask({
  meta: {
    name: 'data:sync',
    description: '從外部 API 同步資料'
  },
  async run({ payload, context }) {
    console.log('[task] 開始同步資料...', payload)
    const db = usePrisma()

    try {
      // 從外部 API 抓取資料
      const response = await $fetch<Array<{ id: string; name: string }>>('https://api.example.com/data')

      // 批次更新資料庫
      const operations = response.map(item =>
        db.externalData.upsert({
          where: { externalId: item.id },
          update: { name: item.name, lastSyncedAt: new Date() },
          create: { externalId: item.id, name: item.name, lastSyncedAt: new Date() }
        })
      )

      await db.$transaction(operations)

      return { result: `已同步 ${response.length} 筆資料`, count: response.length }
    } catch (error) {
      console.error('[task] 同步失敗:', error)
      throw error
    }
  }
})
```

### 報表生成任務

```typescript
// server/tasks/reports/generate.ts
export default defineTask({
  meta: {
    name: 'reports:generate',
    description: '生成每小時營運報表'
  },
  async run() {
    const db = usePrisma()
    const now = new Date()
    const oneHourAgo = new Date(now.getTime() - 60 * 60 * 1000)

    // 統計過去一小時的資料
    const [newUsers, newOrders, revenue] = await Promise.all([
      db.user.count({
        where: { createdAt: { gte: oneHourAgo } }
      }),
      db.order.count({
        where: { createdAt: { gte: oneHourAgo } }
      }),
      db.order.aggregate({
        where: {
          createdAt: { gte: oneHourAgo },
          status: 'completed'
        },
        _sum: { total: true }
      })
    ])

    // 儲存報表
    const report = await db.hourlyReport.create({
      data: {
        periodStart: oneHourAgo,
        periodEnd: now,
        newUsers,
        newOrders,
        revenue: revenue._sum.total || 0
      }
    })

    console.log('[task] 報表已生成:', report.id)
    return { result: '報表生成成功', reportId: report.id }
  }
})
```

### 手動觸發任務

```typescript
// server/api/admin/tasks/run.post.ts
export default defineEventHandler(async (event) => {
  // 驗證管理員權限
  const auth = event.context.auth
  if (!auth || auth.role !== 'admin') {
    throw createError({ statusCode: 403, message: '權限不足' })
  }

  const { taskName, payload } = await readBody<{ taskName: string; payload?: any }>(event)

  try {
    const result = await runTask(taskName, { payload })
    return {
      success: true,
      taskName,
      result
    }
  } catch (error) {
    throw createError({
      statusCode: 500,
      message: `任務執行失敗: ${error}`
    })
  }
})
```

### 開發環境測試任務

```typescript
// server/api/_dev/tasks/test.get.ts (僅開發環境)
export default defineEventHandler(async (event) => {
  if (process.env.NODE_ENV === 'production') {
    throw createError({ statusCode: 404 })
  }

  const query = getQuery(event)
  const taskName = query.task as string

  if (!taskName) {
    return {
      availableTasks: [
        'session:cleanup',
        'data:sync',
        'reports:generate',
        'reports:weekly'
      ]
    }
  }

  const result = await runTask(taskName)
  return { task: taskName, result }
})
```

### 帶參數的任務

```typescript
// server/tasks/email/send-batch.ts
interface EmailPayload {
  recipients: string[]
  subject: string
  template: string
  data: Record<string, any>
}

export default defineTask({
  meta: {
    name: 'email:send-batch',
    description: '批次發送電子郵件'
  },
  async run({ payload }: { payload: EmailPayload }) {
    const { recipients, subject, template, data } = payload

    console.log(`[task] 準備發送 ${recipients.length} 封郵件...`)

    const results = await Promise.allSettled(
      recipients.map(email =>
        sendEmail({
          to: email,
          subject,
          template,
          data
        })
      )
    )

    const successful = results.filter(r => r.status === 'fulfilled').length
    const failed = results.filter(r => r.status === 'rejected').length

    return {
      result: `成功: ${successful}, 失敗: ${failed}`,
      successful,
      failed
    }
  }
})

// 呼叫方式
await runTask('email:send-batch', {
  payload: {
    recipients: ['user1@example.com', 'user2@example.com'],
    subject: '系統通知',
    template: 'notification',
    data: { message: '您有新訊息' }
  }
})
```

### Cron 語法參考

```typescript
// Cron 表達式格式：分 時 日 月 週
// * * * * * 代表每分鐘執行

const cronExamples = {
  '0 0 * * *': '每天午夜 00:00',
  '0 */6 * * *': '每 6 小時',
  '*/15 * * * *': '每 15 分鐘',
  '0 9 * * 1-5': '平日每天早上 9:00',
  '0 0 1 * *': '每月 1 號午夜',
  '0 0 * * 0': '每週日午夜',
  '30 2 * * *': '每天凌晨 2:30',
  '0 9,17 * * *': '每天早上 9:00 和下午 5:00'
}
```

> **重要警告**：
> - 排程任務僅在持久化伺服器環境中運作（Node.js、Docker、長駐進程）
> - **Serverless 環境（Vercel、AWS Lambda、Netlify Functions）不支援**，因為函式實例會隨時被回收
> - Serverless 環境請改用平台提供的 Cron 服務：
>   - **Cloudflare**: Cron Triggers
>   - **AWS**: EventBridge Scheduler
>   - **Vercel**: Vercel Cron Jobs
>   - **GitHub Actions**: Scheduled workflows
> - 開發環境可透過 `/api/_dev/tasks/test?task=task:name` 手動測試任務
