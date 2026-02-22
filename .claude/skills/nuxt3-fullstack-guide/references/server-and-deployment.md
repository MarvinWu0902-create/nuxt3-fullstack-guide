# 伺服器端與部署

## 目錄

1. [H3/Nitro API 設計](#h3nitro-api-設計)
2. [伺服器中介層](#伺服器中介層)
3. [驗證與安全性](#驗證與安全性)
4. [檔案上傳](#檔案上傳)
5. [資料庫整合](#資料庫整合)
6. [Prisma 最佳化](#prisma-最佳化)
7. [WebSocket 與 Server-Sent Events](#websocket-與-server-sent-events)
8. [背景任務與排程](#背景任務與排程)
9. [路由中介層](#路由中介層)
10. [外掛設計](#外掛設計)
11. [健康檢查與優雅關機](#健康檢查與優雅關機)
12. [部署策略](#部署策略)
13. [Cloudflare 部署](#cloudflare-部署)
14. [NuxtHub 部署](#nuxthub-部署)
15. [AWS Lambda 部署](#aws-lambda-部署)
16. [PM2 程序管理](#pm2-程序管理)
17. [CI/CD Pipeline](#cicd-pipeline)
18. [環境設定管理](#環境設定管理)

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

### CSRF 防護

跨站請求偽造（CSRF）攻擊會利用使用者已驗證的 cookie 發送惡意請求。所有變更狀態的端點（POST/PUT/DELETE）皆須防護。

#### 方案一：使用 `nuxt-security` 模組（推薦）

```bash
npx nuxi module add nuxt-security
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    // 啟用 CSRF 防護（基於 Double Submit Cookie 模式）
    csrf: true,
    // 自動在所有 HTML response 中注入 CSRF meta tag
    // 並驗證 POST/PUT/DELETE 請求中的 csrf-token header
  }
})
```

```typescript
// composables/useSafeFetch.ts
// 客戶端發送請求時自動附帶 CSRF Token
export function useSafeFetch<T>(url: string, opts?: Parameters<typeof $fetch>[1]) {
  const csrfToken = useCookie('csrf-token')

  return $fetch<T>(url, {
    ...opts,
    headers: {
      ...opts?.headers,
      'csrf-token': csrfToken.value || ''
    }
  })
}
```

#### 方案二：手動實作 Double Submit Cookie 模式

```typescript
// server/utils/csrf.ts
import { randomBytes, timingSafeEqual } from 'node:crypto'
import type { H3Event } from 'h3'

const CSRF_COOKIE = 'csrf-token'
const CSRF_HEADER = 'x-csrf-token'

/** 產生 CSRF Token 並寫入 cookie */
export function setCsrfToken(event: H3Event): string {
  const token = randomBytes(32).toString('hex')
  setCookie(event, CSRF_COOKIE, token, {
    httpOnly: false,   // 客戶端需讀取此 cookie 以附帶到 header
    secure: true,
    sameSite: 'strict',
    path: '/'
  })
  return token
}

/** 驗證 CSRF Token — 比對 cookie 與 header 是否一致 */
export function verifyCsrfToken(event: H3Event): void {
  const method = getMethod(event)
  // 僅對變更狀態的方法驗證
  if (['GET', 'HEAD', 'OPTIONS'].includes(method)) return

  const cookieToken = getCookie(event, CSRF_COOKIE)
  const headerToken = getHeader(event, CSRF_HEADER)

  // 使用 timing-safe 比較防止計時攻擊
  if (!cookieToken || !headerToken
    || Buffer.byteLength(cookieToken) !== Buffer.byteLength(headerToken)
    || !timingSafeEqual(Buffer.from(cookieToken), Buffer.from(headerToken))) {
    throw createError({
      statusCode: 403,
      message: 'CSRF 驗證失敗：請重新載入頁面後再試'
    })
  }
}
```

```typescript
// server/middleware/csrf.ts
// 伺服器中介層：自動為所有請求驗證 CSRF Token
export default defineEventHandler((event) => {
  // 排除不需要 CSRF 防護的路徑（如 webhook）
  const path = getRequestURL(event).pathname
  if (path.startsWith('/api/webhooks/')) return

  verifyCsrfToken(event)
})
```

```typescript
// server/api/auth/csrf.get.ts
// 提供端點讓客戶端取得 CSRF Token（首次載入或 token 過期時呼叫）
export default defineEventHandler((event) => {
  const token = setCsrfToken(event)
  return { csrfToken: token }
})
```

```typescript
// plugins/csrf.client.ts
// 客戶端外掛：自動在 $fetch 請求中附帶 CSRF Token
export default defineNuxtPlugin(() => {
  const csrfToken = useCookie('csrf-token')

  // 攔截所有 $fetch 請求，自動帶上 CSRF header
  globalThis.$fetch = $fetch.create({
    onRequest({ options }) {
      const method = (options.method || 'GET').toUpperCase()
      if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(method)) {
        options.headers = {
          ...options.headers,
          'x-csrf-token': csrfToken.value || ''
        }
      }
    }
  })
})
```

> **注意**：CSRF 防護依賴 `SameSite=Strict` cookie。若需支援跨站 OAuth 回呼，請對回呼路徑單獨排除 CSRF 檢查。此外，搭配 CORS 設定明確的 `Access-Control-Allow-Origin` 可形成多層防禦。

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

// 利用 Prisma DMMF 檢查模型是否有特定欄位
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

// 批次建立（更快）
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

---

## 路由中介層

### 客戶端路由中介層

```typescript
// middleware/auth.ts（全域或具名）
export default defineNuxtRouteMiddleware((to, from) => {
  const authStore = useAuthStore()

  // 需要認證的路由
  if (to.meta.requiresAuth && !authStore.isLoggedIn) {
    return navigateTo({
      path: '/login',
      query: { redirect: to.fullPath }
    })
  }

  // 已登入則跳過登入頁
  if (to.path === '/login' && authStore.isLoggedIn) {
    return navigateTo('/dashboard')
  }

  // 角色檢查
  if (to.meta.requiredRole && typeof to.meta.requiredRole === 'string') {
    if (authStore.user?.role !== to.meta.requiredRole) {
      return abortNavigation(
        createError({ statusCode: 403, message: '權限不足' })
      )
    }
  }
})
```

### 在頁面中使用

```vue
<!-- pages/dashboard.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: 'auth',
  requiresAuth: true
})
</script>

<!-- pages/admin/index.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: 'auth',
  requiresAuth: true,
  requiredRole: 'admin',
  layout: 'dashboard'
})
</script>
```

### 全域中介層

```typescript
// middleware/analytics.global.ts（.global 後綴 = 自動套用所有路由）
export default defineNuxtRouteMiddleware((to) => {
  if (import.meta.client) {
    trackPageView(to.fullPath)
  }
})
```

---

## 外掛設計

### 外掛執行順序

```
1. server/plugins/ → Nitro 伺服器啟動時
2. plugins/01.xxx.ts → 數字前綴控制順序
3. plugins/02.xxx.ts
4. plugins/xxx.client.ts → 僅客戶端
5. plugins/xxx.server.ts → 僅伺服器端
```

### 典型外掛範例

```typescript
// plugins/01.auth.ts — 認證初始化
export default defineNuxtPlugin(async () => {
  const authStore = useAuthStore()

  // SSR 時取得使用者資料
  if (import.meta.server) {
    await authStore.fetchUser()
  }
})

// plugins/02.api.ts — API 客戶端設定
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
    }
  })

  return {
    provide: {
      api  // 使用 useNuxtApp().$api 存取
    }
  }
})

// plugins/toast.client.ts — 僅客戶端的 toast 通知
export default defineNuxtPlugin(() => {
  // 僅在客戶端初始化 toast 系統
  const toastContainer = document.createElement('div')
  toastContainer.id = 'toast-container'
  document.body.appendChild(toastContainer)

  return {
    provide: {
      toast: {
        success: (msg: string) => showToast(msg, 'success'),
        error: (msg: string) => showToast(msg, 'error')
      }
    }
  }
})
```

---

## 健康檢查與優雅關機

### 健康檢查端點

```typescript
// server/api/_health.get.ts
export default defineEventHandler(async (event) => {
  const checks: Record<string, 'ok' | 'error'> = {}

  // 資料庫連線檢查
  try {
    const db = usePrisma()
    await db.$queryRaw`SELECT 1`
    checks.database = 'ok'
  } catch (error) {
    console.error('[health] 資料庫檢查失敗:', error)
    checks.database = 'error'
  }

  // Redis 連線檢查（若有使用）
  try {
    const redis = useRedis()
    await redis.ping()
    checks.redis = 'ok'
  } catch (error) {
    console.error('[health] Redis 檢查失敗:', error)
    checks.redis = 'error'
  }

  // 外部服務檢查
  try {
    const response = await $fetch('https://api.external-service.com/health', {
      timeout: 3000
    })
    checks.externalApi = 'ok'
  } catch (error) {
    checks.externalApi = 'error'
  }

  const allHealthy = Object.values(checks).every(v => v === 'ok')

  // 若任一檢查失敗，回傳 503 狀態碼
  if (!allHealthy) {
    setResponseStatus(event, 503)
  }

  return {
    status: allHealthy ? 'healthy' : 'degraded',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: {
      used: Math.round(process.memoryUsage().heapUsed / 1024 / 1024), // MB
      total: Math.round(process.memoryUsage().heapTotal / 1024 / 1024)
    },
    checks
  }
})
```

### 詳細系統資訊端點（僅內部使用）

```typescript
// server/api/_health/detailed.get.ts
export default defineEventHandler(async (event) => {
  // 僅允許內部網路或管理員存取
  const ip = getRequestIP(event)
  const allowedIPs = ['127.0.0.1', '::1', '10.0.0.0/8'] // 內部網路

  // 或檢查管理員權限
  const auth = event.context.auth
  if (auth?.role !== 'admin') {
    throw createError({ statusCode: 403, message: '權限不足' })
  }

  const db = usePrisma()

  return {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    server: {
      nodeVersion: process.version,
      platform: process.platform,
      uptime: process.uptime(),
      env: process.env.NODE_ENV
    },
    memory: {
      rss: Math.round(process.memoryUsage().rss / 1024 / 1024),
      heapUsed: Math.round(process.memoryUsage().heapUsed / 1024 / 1024),
      heapTotal: Math.round(process.memoryUsage().heapTotal / 1024 / 1024),
      external: Math.round(process.memoryUsage().external / 1024 / 1024)
    },
    database: {
      activeConnections: await db.$queryRaw`SELECT count(*) FROM pg_stat_activity WHERE datname = current_database()`
    }
  }
})
```

### Kubernetes 健康探測配置

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuxt-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nuxt-app
  template:
    metadata:
      labels:
        app: nuxt-app
    spec:
      containers:
      - name: nuxt-app
        image: your-registry/nuxt-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: NUXT_DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url

        # 存活探測：失敗時重啟容器
        livenessProbe:
          httpGet:
            path: /api/_health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3

        # 就緒探測：決定是否接收流量
        readinessProbe:
          httpGet:
            path: /api/_health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 2

        # 啟動探測：給應用更長的啟動時間
        startupProbe:
          httpGet:
            path: /api/_health
            port: 3000
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 12  # 最多等待 60 秒

        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### 優雅關機處理

```typescript
// server/plugins/graceful-shutdown.ts
export default defineNitroPlugin((nitro) => {
  nitro.hooks.hook('close', async () => {
    console.log('[shutdown] 開始優雅關機程序...')

    // 1. 關閉資料庫連線
    try {
      const db = usePrisma()
      await db.$disconnect()
      console.log('[shutdown] 資料庫連線已關閉')
    } catch (error) {
      console.error('[shutdown] 關閉資料庫連線失敗:', error)
    }

    // 2. 關閉 Redis 連線
    try {
      const redis = useRedis()
      await redis.quit()
      console.log('[shutdown] Redis 連線已關閉')
    } catch (error) {
      console.error('[shutdown] 關閉 Redis 連線失敗:', error)
    }

    // 3. 完成所有進行中的請求（Nitro 自動處理）
    console.log('[shutdown] 等待進行中的請求完成...')

    // 4. 清理其他資源
    // 例如：關閉 WebSocket 連線、停止背景任務等

    console.log('[shutdown] 優雅關機完成')
  })

  // 處理未捕獲的錯誤
  nitro.hooks.hook('error', (error) => {
    console.error('[error] 未捕獲的錯誤:', error)
  })

  // 請求開始時的日誌
  nitro.hooks.hook('request', (event) => {
    console.log('[request]', event.method, event.path)
  })

  // 請求完成時的日誌
  nitro.hooks.hook('afterResponse', (event) => {
    const duration = Date.now() - event.context.__startTime
    console.log(
      '[response]',
      event.method,
      event.path,
      event.node.res.statusCode,
      `${duration}ms`
    )
  })
})

// 記錄請求開始時間
// server/middleware/timing.ts
export default defineEventHandler((event) => {
  event.context.__startTime = Date.now()
})
```

### Docker 優雅關機配置

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app

# 安裝 dumb-init 處理訊號
RUN apk add --no-cache dumb-init

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nuxtuser

COPY --from=builder --chown=nuxtuser:nodejs /app/.output ./.output

USER nuxtuser

ENV NODE_ENV=production
ENV HOST=0.0.0.0
ENV PORT=3000

EXPOSE 3000

# 使用 dumb-init 確保訊號正確傳遞
ENTRYPOINT ["dumb-init", "--"]

# 優雅關機：收到 SIGTERM 後等待最多 30 秒
STOPSIGNAL SIGTERM
CMD ["node", ".output/server/index.mjs"]
```

### PM2 優雅關機配置

```javascript
// ecosystem.config.cjs
module.exports = {
  apps: [{
    name: 'nuxt-app',
    script: '.output/server/index.mjs',
    instances: 'max',
    exec_mode: 'cluster',

    // 優雅關機設定
    kill_timeout: 5000,        // 給進程 5 秒時間完成關機
    wait_ready: true,          // 等待應用發送 ready 訊號
    listen_timeout: 10000,     // 啟動超時時間

    env_production: {
      NODE_ENV: 'production',
      NITRO_PORT: 3000
    }
  }]
}
```

### 監控告警整合

```typescript
// server/api/_health/metrics.get.ts — Prometheus 格式指標
export default defineEventHandler(async (event) => {
  const db = usePrisma()

  const [userCount, orderCount] = await Promise.all([
    db.user.count(),
    db.order.count()
  ])

  const metrics = `
# HELP app_users_total Total number of users
# TYPE app_users_total gauge
app_users_total ${userCount}

# HELP app_orders_total Total number of orders
# TYPE app_orders_total gauge
app_orders_total ${orderCount}

# HELP app_uptime_seconds Application uptime in seconds
# TYPE app_uptime_seconds gauge
app_uptime_seconds ${process.uptime()}

# HELP app_memory_used_bytes Memory used in bytes
# TYPE app_memory_used_bytes gauge
app_memory_used_bytes ${process.memoryUsage().heapUsed}
  `.trim()

  setResponseHeader(event, 'Content-Type', 'text/plain; version=0.0.4')
  return metrics
})
```

> **最佳實踐**：
> - 健康檢查端點應輕量且快速（< 100ms）
> - Liveness 探測失敗會重啟容器，設定較寬鬆
> - Readiness 探測決定流量分發，可較嚴格
> - 優雅關機應在 30 秒內完成
> - 生產環境建議整合 Prometheus + Grafana 監控

---

## 部署策略

### SSR vs SSG vs ISR 選擇

| 策略 | 適用場景 | 優點 | 缺點 |
|------|---------|------|------|
| **SSR** | 動態內容、個人化頁面 | 即時資料、SEO 友善 | 需要伺服器、TTFB 較慢 |
| **SSG** | 部落格、文件、行銷頁面 | 最快載入、可 CDN 部署 | 建置時間長、資料不即時 |
| **ISR** | 電商產品頁、新聞 | 兼顧效能與即時性 | 配置較複雜 |
| **SPA** | 後台管理、儀表板 | 不需 SSR 伺服器 | 不利 SEO |

### 混合渲染設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 首頁：預渲染（建置時產生靜態 HTML）
    '/': { prerender: true },

    // 部落格：ISR，每小時重新生成
    '/blog/**': { isr: 3600 },

    // 產品頁：SWR，60 秒後在背景更新
    '/products/**': { swr: 60 },

    // 後台：SPA 模式（不做 SSR）
    '/dashboard/**': { ssr: false },

    // API：設定 CORS 和快取
    '/api/**': { cors: true },
    '/api/config': { cache: { maxAge: 3600 } }
  }
})
```

### Vercel 部署

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'vercel'
  }
})
```

### Docker 部署

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nuxtuser
COPY --from=builder --chown=nuxtuser:nodejs /app/.output ./.output
USER nuxtuser
ENV NODE_ENV=production
ENV HOST=0.0.0.0
ENV PORT=3000
EXPOSE 3000
CMD ["node", ".output/server/index.mjs"]
```

---

## Cloudflare 部署

### Cloudflare Pages 設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages'
  }
})
```

### Wrangler 配置

```toml
# wrangler.toml
name = "my-nuxt-app"
compatibility_date = "2024-09-19"
pages_build_output_dir = ".output/public"

# KV 命名空間綁定
[[kv_namespaces]]
binding = "KV"
id = "your-kv-namespace-id"
preview_id = "your-preview-kv-namespace-id"

# D1 資料庫綁定
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "your-database-id"

# R2 儲存桶綁定
[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-uploads"
preview_bucket_name = "my-uploads-preview"

# Durable Objects（WebSocket 支援）
[[durable_objects.bindings]]
name = "CHAT"
class_name = "ChatRoom"
script_name = "my-nuxt-app"

# 環境變數
[vars]
ENVIRONMENT = "production"

# 秘密環境變數（使用 wrangler secret put）
# wrangler secret put JWT_SECRET
# wrangler secret put DATABASE_URL
```

### 存取 Cloudflare Bindings

```typescript
// server/api/data.get.ts
export default defineEventHandler(async (event) => {
  // 取得 Cloudflare 綁定
  const { KV, DB, BUCKET } = event.context.cloudflare.env

  // KV 操作
  await KV.put('key', 'value', {
    expirationTtl: 3600 // 1 小時後過期
  })
  const value = await KV.get('key')

  // D1 資料庫查詢
  const { results } = await DB.prepare('SELECT * FROM users WHERE id = ?')
    .bind(1)
    .all()

  // R2 檔案上傳
  await BUCKET.put('file.txt', 'Hello World', {
    httpMetadata: {
      contentType: 'text/plain'
    }
  })

  return { value, users: results }
})
```

### D1 資料庫操作

```typescript
// server/utils/d1.ts
import type { D1Database } from '@cloudflare/workers-types'

export function useD1(event: H3Event): D1Database {
  return event.context.cloudflare.env.DB
}

// server/api/users/index.get.ts
export default defineEventHandler(async (event) => {
  const db = useD1(event)

  const { results } = await db.prepare(`
    SELECT id, name, email, created_at
    FROM users
    ORDER BY created_at DESC
    LIMIT 20
  `).all<{ id: number; name: string; email: string; created_at: string }>()

  return { users: results }
})

// server/api/users/index.post.ts
export default defineEventHandler(async (event) => {
  const db = useD1(event)
  const { name, email } = await readBody(event)

  const result = await db.prepare(`
    INSERT INTO users (name, email, created_at)
    VALUES (?, ?, datetime('now'))
    RETURNING id, name, email
  `).bind(name, email).first()

  return { user: result }
})
```

### KV 快取策略

```typescript
// server/utils/cache.ts
import type { KVNamespace } from '@cloudflare/workers-types'

export function useKV(event: H3Event): KVNamespace {
  return event.context.cloudflare.env.KV
}

export async function getCached<T>(
  kv: KVNamespace,
  key: string,
  fetcher: () => Promise<T>,
  ttl: number = 3600
): Promise<T> {
  // 嘗試從快取取得
  const cached = await kv.get(key, 'json')
  if (cached) return cached as T

  // 快取未命中，執行 fetcher
  const data = await fetcher()

  // 儲存到快取
  await kv.put(key, JSON.stringify(data), { expirationTtl: ttl })

  return data
}

// 使用範例
// server/api/products/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  const kv = useKV(event)

  const product = await getCached(
    kv,
    `product:${id}`,
    async () => {
      // 從資料庫或外部 API 取得資料
      const db = useD1(event)
      return await db.prepare('SELECT * FROM products WHERE id = ?').bind(id).first()
    },
    1800 // 快取 30 分鐘
  )

  return { product }
})
```

### R2 檔案儲存

```typescript
// server/api/upload.post.ts
export default defineEventHandler(async (event) => {
  const { BUCKET } = event.context.cloudflare.env
  const files = await readMultipartFormData(event)

  if (!files || files.length === 0) {
    throw createError({ statusCode: 400, message: '未提供檔案' })
  }

  const file = files[0]
  const key = `uploads/${Date.now()}-${file.filename}`

  // 上傳到 R2
  await BUCKET.put(key, file.data, {
    httpMetadata: {
      contentType: file.type || 'application/octet-stream'
    },
    customMetadata: {
      originalName: file.filename || 'unknown',
      uploadedAt: new Date().toISOString()
    }
  })

  // 產生公開 URL（需設定 R2 bucket 為公開或使用 signed URL）
  const url = `https://your-bucket.r2.dev/${key}`

  return { url, key }
})

// server/api/files/[key].delete.ts
export default defineEventHandler(async (event) => {
  const { BUCKET } = event.context.cloudflare.env
  const key = getRouterParam(event, 'key')

  await BUCKET.delete(`uploads/${key}`)

  return { success: true }
})

// server/api/files/[key].get.ts — 取得檔案資訊
export default defineEventHandler(async (event) => {
  const { BUCKET } = event.context.cloudflare.env
  const key = getRouterParam(event, 'key')

  const object = await BUCKET.head(`uploads/${key}`)

  if (!object) {
    throw createError({ statusCode: 404, message: '檔案不存在' })
  }

  return {
    key,
    size: object.size,
    uploaded: object.uploaded,
    contentType: object.httpMetadata?.contentType,
    metadata: object.customMetadata
  }
})
```

### D1 資料庫遷移

```bash
# 建立遷移檔案
wrangler d1 migrations create my-database create_users_table

# 執行遷移（本地）
wrangler d1 migrations apply my-database --local

# 執行遷移（生產環境）
wrangler d1 migrations apply my-database --remote
```

```sql
-- migrations/0001_create_users_table.sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

### 部署指令

```bash
# 建置專案
npx nuxi build --preset cloudflare-pages

# 本地預覽（使用 Wrangler）
npx wrangler pages dev .output/public

# 部署到 Cloudflare Pages
npx wrangler pages deploy .output/public

# 設定環境變數（秘密）
npx wrangler secret put JWT_SECRET
npx wrangler secret put DATABASE_URL

# 查看部署狀態
npx wrangler pages deployment list
```

### 環境變數設定

```typescript
// server/utils/env.ts — 存取環境變數
export function useEnv(event: H3Event) {
  return event.context.cloudflare.env
}

// 使用方式
export default defineEventHandler(async (event) => {
  const env = useEnv(event)
  const jwtSecret = env.JWT_SECRET
  const apiKey = env.EXTERNAL_API_KEY

  // ... 使用環境變數
})
```

### Cloudflare Workers AI 整合

```typescript
// server/api/ai/generate.post.ts
export default defineEventHandler(async (event) => {
  const { AI } = event.context.cloudflare.env
  const { prompt } = await readBody(event)

  const response = await AI.run('@cf/meta/llama-2-7b-chat-int8', {
    messages: [
      { role: 'user', content: prompt }
    ]
  })

  return response
})
```

> **Cloudflare 部署注意事項**：
> - Cloudflare Workers 有 CPU 時間限制（免費版 10ms，付費版 50ms）
> - 不支援 Node.js 特定 API（如 fs、child_process）
> - WebSocket 需使用 Durable Objects
> - D1 已正式 GA（General Availability），適用於生產環境
> - R2 無出站流量費用，非常適合儲存靜態資源

---

## NuxtHub 部署

### 安裝 NuxtHub

```bash
npx nuxi module add @nuxthub/core
```

### NuxtHub 配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxthub/core'],

  hub: {
    database: true,  // 啟用 D1 資料庫
    kv: true,        // 啟用 KV 儲存
    blob: true,      // 啟用 R2 Blob 儲存
    cache: true      // 啟用快取層
  },

  // 遠端開發模式（連接 nuxthub.com 的遠端資源）
  // hub: {
  //   remote: true
  // }
})
```

### 使用 NuxtHub Database (D1)

```typescript
// server/api/users/index.get.ts
export default defineEventHandler(async () => {
  const db = hubDatabase()

  const users = await db.prepare(`
    SELECT id, name, email, created_at
    FROM users
    ORDER BY created_at DESC
    LIMIT 20
  `).all()

  return { users: users.results }
})

// server/api/users/index.post.ts
export default defineEventHandler(async (event) => {
  const db = hubDatabase()
  const { name, email } = await readBody(event)

  const result = await db.prepare(`
    INSERT INTO users (name, email, created_at)
    VALUES (?, ?, datetime('now'))
    RETURNING id, name, email, created_at
  `).bind(name, email).first()

  return { user: result }
})

// server/api/users/[id].delete.ts
export default defineEventHandler(async (event) => {
  const db = hubDatabase()
  const id = getRouterParam(event, 'id')

  await db.prepare('DELETE FROM users WHERE id = ?').bind(id).run()

  return { success: true }
})
```

### 使用 NuxtHub KV

```typescript
// server/api/cache/set.post.ts
export default defineEventHandler(async (event) => {
  const kv = hubKV()
  const { key, value, ttl } = await readBody<{ key: string; value: any; ttl?: number }>(event)

  await kv.set(key, value, { ttl: ttl || 3600 })

  return { success: true }
})

// server/api/cache/get.get.ts
export default defineEventHandler(async (event) => {
  const kv = hubKV()
  const { key } = getQuery(event)

  const value = await kv.get(key as string)

  return { value }
})

// 實際應用：快取 API 回應
// server/api/posts/index.get.ts
export default defineEventHandler(async (event) => {
  const kv = hubKV()
  const cacheKey = 'posts:all'

  // 嘗試從快取取得
  const cached = await kv.get(cacheKey)
  if (cached) {
    return { posts: cached, fromCache: true }
  }

  // 從資料庫查詢
  const db = hubDatabase()
  const { results } = await db.prepare('SELECT * FROM posts ORDER BY created_at DESC').all()

  // 儲存到快取（5 分鐘）
  await kv.set(cacheKey, results, { ttl: 300 })

  return { posts: results, fromCache: false }
})
```

### 使用 NuxtHub Blob (R2)

```typescript
// server/api/files/upload.post.ts
export default defineEventHandler(async (event) => {
  const form = await readMultipartFormData(event)
  const file = form?.[0]

  if (!file) {
    throw createError({ statusCode: 400, message: '未提供檔案' })
  }

  // 使用 NuxtHub Blob 儲存
  const blob = hubBlob()
  const result = await blob.put(file.filename!, file.data, {
    contentType: file.type,
    addRandomSuffix: true  // 自動添加隨機後綴避免衝突
  })

  return {
    pathname: result.pathname,
    url: result.url,
    size: result.size,
    uploadedAt: result.uploadedAt
  }
})

// server/api/files/[...pathname].get.ts — 取得檔案
export default defineEventHandler(async (event) => {
  const pathname = getRouterParam(event, 'pathname')
  const blob = hubBlob()

  const file = await blob.head(pathname!)

  if (!file) {
    throw createError({ statusCode: 404, message: '檔案不存在' })
  }

  return file
})

// server/api/files/[...pathname].delete.ts — 刪除檔案
export default defineEventHandler(async (event) => {
  const pathname = getRouterParam(event, 'pathname')
  const blob = hubBlob()

  await blob.del(pathname!)

  return { success: true }
})

// server/api/files/list.get.ts — 列出所有檔案
export default defineEventHandler(async () => {
  const blob = hubBlob()

  const { blobs } = await blob.list({
    limit: 100,
    prefix: 'uploads/' // 可選的前綴過濾
  })

  return { files: blobs }
})
```

### 圖片上傳與處理

```typescript
// server/api/images/upload.post.ts
export default defineEventHandler(async (event) => {
  const form = await readMultipartFormData(event)
  const file = form?.[0]

  if (!file) {
    throw createError({ statusCode: 400, message: '未提供圖片' })
  }

  // 驗證圖片類型
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif']
  if (!allowedTypes.includes(file.type || '')) {
    throw createError({ statusCode: 415, message: '不支援的圖片格式' })
  }

  // 檔案大小限制 5MB
  const maxSize = 5 * 1024 * 1024
  if (file.data.length > maxSize) {
    throw createError({ statusCode: 413, message: '圖片大小超過限制（5MB）' })
  }

  const blob = hubBlob()

  // 儲存原始圖片
  const result = await blob.put(`images/${file.filename}`, file.data, {
    contentType: file.type,
    addRandomSuffix: true
  })

  // 儲存圖片元資訊到資料庫
  const db = hubDatabase()
  await db.prepare(`
    INSERT INTO images (pathname, url, size, content_type, uploaded_at)
    VALUES (?, ?, ?, ?, datetime('now'))
  `).bind(result.pathname, result.url, result.size, file.type).run()

  return result
})
```

### 資料庫遷移管理

```typescript
// server/database/migrations/0001_create_tables.sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  published BOOLEAN DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE IF NOT EXISTS images (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  pathname TEXT NOT NULL,
  url TEXT NOT NULL,
  size INTEGER NOT NULL,
  content_type TEXT NOT NULL,
  uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at);
CREATE INDEX idx_images_uploaded_at ON images(uploaded_at);
```

### 本地開發與遠端資源

```typescript
// nuxt.config.ts — 開發時連接遠端資源
export default defineNuxtConfig({
  modules: ['@nuxthub/core'],

  hub: {
    // 使用遠端資源（需先在 nuxthub.com 建立專案）
    remote: true,

    // 或僅在生產環境使用遠端
    // remote: process.env.NODE_ENV === 'production'
  }
})
```

### Admin Dashboard

NuxtHub 提供內建的管理介面。

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxthub/core'],

  hub: {
    database: true,
    kv: true,
    blob: true
  },

  // 啟用開發工具
  devtools: { enabled: true }
})
```

在開發模式下訪問 `http://localhost:3000/_hub` 查看：
- 資料庫查詢工具
- KV 儲存瀏覽器
- Blob 檔案管理器

### 部署到 NuxtHub

```bash
# 登入 NuxtHub
npx nuxthub login

# 部署（首次會建立專案）
npx nuxthub deploy

# 部署到特定專案
npx nuxthub deploy my-project

# 查看部署狀態
npx nuxthub status

# 查看日誌
npx nuxthub logs
```

### GitHub 自動部署

連接 GitHub repository 到 NuxtHub，每次 push 到 main 分支時自動部署。

1. 前往 [nuxthub.com](https://nuxthub.com)
2. 建立新專案
3. 連接 GitHub repository
4. 設定環境變數
5. 自動部署完成

### 環境變數管理

```bash
# 透過 CLI 設定環境變數
npx nuxthub env set JWT_SECRET your-secret-key
npx nuxthub env set DATABASE_URL your-database-url

# 列出所有環境變數
npx nuxthub env list

# 刪除環境變數
npx nuxthub env unset JWT_SECRET
```

### NuxtHub Cache

```typescript
// server/api/data.get.ts
// 使用 Nitro cachedEventHandler 搭配 NuxtHub 快取層
export default cachedEventHandler(async (event) => {
  const db = hubDatabase()
  const { results } = await db.prepare('SELECT * FROM data').all()
  return results
}, {
  maxAge: 60 * 5, // 快取 5 分鐘
  swr: true       // Stale-While-Revalidate
})
```

### 完整範例：部落格系統

```typescript
// server/api/blog/posts.get.ts
export default defineEventHandler(async () => {
  const db = hubDatabase()

  const { results } = await db.prepare(`
    SELECT p.id, p.title, p.slug, p.excerpt, p.published_at,
           u.name as author_name
    FROM posts p
    JOIN users u ON p.author_id = u.id
    WHERE p.published = 1
    ORDER BY p.published_at DESC
    LIMIT 20
  `).all()

  return { posts: results }
})

// server/api/blog/posts/[slug].get.ts
export default defineEventHandler(async (event) => {
  const slug = getRouterParam(event, 'slug')
  const db = hubDatabase()
  const kv = hubKV()

  // 嘗試從快取取得
  const cacheKey = `post:${slug}`
  const cached = await kv.get(cacheKey)
  if (cached) return { post: cached, fromCache: true }

  // 從資料庫查詢
  const post = await db.prepare(`
    SELECT p.*, u.name as author_name, u.email as author_email
    FROM posts p
    JOIN users u ON p.author_id = u.id
    WHERE p.slug = ? AND p.published = 1
  `).bind(slug).first()

  if (!post) {
    throw createError({ statusCode: 404, message: '文章不存在' })
  }

  // 快取 1 小時
  await kv.set(cacheKey, post, { ttl: 3600 })

  return { post, fromCache: false }
})
```

> **NuxtHub 優勢**：
> - 簡化的 Cloudflare 資源管理
> - 內建管理介面
> - 無縫本地開發與遠端資源整合
> - 自動化部署流程
> - 零配置的 D1、KV、R2 整合
> - 適合快速建立全端應用

---

## AWS Lambda 部署

### Lambda Preset 設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'aws-lambda'
  }
})
```

### SST (Serverless Stack) 整合

> **⚠️ 版本注意**：以下範例基於 **SST v2**（使用 AWS CDK constructs）。SST v3 (Ion) 已於 2024 年發布，改用 Pulumi 作為基礎設施引擎，API 完全不同。若您要開始新專案，建議直接使用 SST v3。SST v2 仍持續維護但不再添加新功能。
>
> SST v3 的 Nuxt 部署範例：
> ```typescript
> // sst.config.ts (SST v3 / Ion)
> export default $config({
>   app(input) {
>     return { name: 'my-nuxt-app', home: 'aws' }
>   },
>   async run() {
>     new sst.aws.Nuxt('Site', {
>       domain: 'app.example.com',
>       environment: {
>         DATABASE_URL: process.env.DATABASE_URL!
>       }
>     })
>   }
> })
> ```

```bash
# 安裝 SST
npm install --save-dev sst aws-cdk-lib
```

```typescript
// sst.config.ts
import type { SSTConfig } from 'sst'
import { NuxtSite } from 'sst/constructs'

export default {
  config() {
    return {
      name: 'my-nuxt-app',
      region: 'ap-northeast-1' // 東京
    }
  },

  stacks(app) {
    app.stack(function Site({ stack }) {
      const site = new NuxtSite(stack, 'Site', {
        // 環境變數
        environment: {
          DATABASE_URL: process.env.DATABASE_URL!,
          JWT_SECRET: process.env.JWT_SECRET!
        },

        // 自訂網域（可選）
        customDomain: {
          domainName: 'app.example.com',
          hostedZone: 'example.com'
        }
      })

      // 輸出網站 URL
      stack.addOutputs({
        SiteUrl: site.url
      })
    })
  }
} satisfies SSTConfig
```

### 部署指令

```bash
# 開發模式（本地測試）
npx sst dev

# 部署到 AWS
npx sst deploy --stage production

# 移除資源
npx sst remove --stage production
```

### CloudFormation 手動部署

```yaml
# cloudformation-template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  NuxtFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: nuxt-app
      Runtime: nodejs20.x
      Handler: index.handler
      CodeUri: .output/server/
      MemorySize: 1024
      Timeout: 30
      Environment:
        Variables:
          NODE_ENV: production
          DATABASE_URL: !Ref DatabaseUrl
          JWT_SECRET: !Ref JwtSecret
      Events:
        ApiEvent:
          Type: HttpApi
          Properties:
            Path: /{proxy+}
            Method: ANY

  CloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Enabled: true
        Origins:
          - Id: LambdaOrigin
            DomainName: !GetAtt NuxtFunction.FunctionUrl
            CustomOriginConfig:
              OriginProtocolPolicy: https-only
        DefaultCacheBehavior:
          TargetOriginId: LambdaOrigin
          ViewerProtocolPolicy: redirect-to-https
          AllowedMethods: [GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE]
          CachedMethods: [GET, HEAD, OPTIONS]
          ForwardedValues:
            QueryString: true
            Cookies:
              Forward: all

Outputs:
  ApiUrl:
    Value: !GetAtt NuxtFunction.FunctionUrl
  CloudFrontUrl:
    Value: !GetAtt CloudFrontDistribution.DomainName
```

### 冷啟動最佳化

```typescript
// nuxt.config.ts — 減少 bundle 大小
export default defineNuxtConfig({
  nitro: {
    preset: 'aws-lambda',

    // 外部化大型依賴（減少部署包大小）
    externals: {
      inline: ['nitro']
    },

    // 壓縮輸出
    minify: true,

    // Rollup 最佳化
    rollupConfig: {
      output: {
        manualChunks: undefined
      }
    }
  },

  // 啟用 tree-shaking
  experimental: {
    payloadExtraction: false
  }
})
```

### Lambda Layer 使用（共享依賴）

```bash
# 建立 Lambda Layer
mkdir -p layers/nodejs/node_modules
cd layers/nodejs
npm install prisma @prisma/client
cd ../..
zip -r prisma-layer.zip layers/
```

```typescript
// sst.config.ts — 使用 Layer
import { LayerVersion } from 'aws-cdk-lib/aws-lambda'

export default {
  stacks(app) {
    app.stack(function Site({ stack }) {
      // 建立 Lambda Layer
      const prismaLayer = new LayerVersion(stack, 'PrismaLayer', {
        code: Code.fromAsset('./prisma-layer.zip'),
        compatibleRuntimes: [Runtime.NODEJS_20_X]
      })

      const site = new NuxtSite(stack, 'Site', {
        // 附加 Layer
        cdk: {
          server: {
            layers: [prismaLayer]
          }
        }
      })
    })
  }
} satisfies SSTConfig
```

### 環境變數管理（使用 SSM Parameter Store）

```typescript
// server/utils/config.ts
import { SSMClient, GetParameterCommand } from '@aws-sdk/client-ssm'

const ssm = new SSMClient({ region: 'ap-northeast-1' })

export async function getSecretParameter(name: string): Promise<string> {
  const command = new GetParameterCommand({
    Name: name,
    WithDecryption: true
  })

  const response = await ssm.send(command)
  return response.Parameter?.Value || ''
}

// 使用方式
const dbPassword = await getSecretParameter('/myapp/database/password')
```

### Provisioned Concurrency（避免冷啟動）

```typescript
// sst.config.ts
export default {
  stacks(app) {
    app.stack(function Site({ stack }) {
      const site = new NuxtSite(stack, 'Site', {
        cdk: {
          server: {
            // 設定預留並發（避免冷啟動，但會產生費用）
            reservedConcurrentExecutions: 5,

            // 或使用 Provisioned Concurrency
            currentVersionOptions: {
              provisionedConcurrentExecutions: 3
            }
          }
        }
      })
    })
  }
} satisfies SSTConfig
```

### RDS Proxy 整合（資料庫連線池）

```typescript
// sst.config.ts
import { RdsProxy } from 'aws-cdk-lib/aws-rds'

export default {
  stacks(app) {
    app.stack(function Database({ stack }) {
      // 建立 RDS Proxy
      const proxy = new RdsProxy(stack, 'Proxy', {
        proxyTarget: /* your RDS instance */,
        secrets: [/* database credentials */],
        vpc: /* your VPC */
      })

      const site = new NuxtSite(stack, 'Site', {
        environment: {
          DATABASE_URL: proxy.endpoint
        }
      })
    })
  }
} satisfies SSTConfig
```

### S3 + CloudFront 靜態資源

```typescript
// sst.config.ts
import { Bucket } from 'sst/constructs'

export default {
  stacks(app) {
    app.stack(function Storage({ stack }) {
      // 靜態資源 Bucket
      const bucket = new Bucket(stack, 'Assets', {
        cdk: {
          bucket: {
            publicReadAccess: true
          }
        }
      })

      const site = new NuxtSite(stack, 'Site', {
        environment: {
          ASSETS_BUCKET: bucket.bucketName
        }
      })

      stack.addOutputs({
        AssetsBucket: bucket.bucketName
      })
    })
  }
} satisfies SSTConfig
```

### Lambda + CloudFront（SSR at Edge）

透過 AWS Lambda 搭配 CloudFront 實現邊緣快取的 SSR。Nitro 目前不支援 `aws-lambda-edge` preset，但可以用 `aws-lambda` 搭配 CloudFront 分發達到類似效果：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'aws-lambda'
  }
})
```

### 監控與日誌

```typescript
// server/plugins/cloudwatch.ts
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch'

const cloudwatch = new CloudWatchClient({ region: 'ap-northeast-1' })

export default defineNitroPlugin((nitro) => {
  nitro.hooks.hook('afterResponse', async (event) => {
    const duration = Date.now() - event.context.__startTime

    // 發送自訂指標到 CloudWatch
    await cloudwatch.send(new PutMetricDataCommand({
      Namespace: 'NuxtApp',
      MetricData: [{
        MetricName: 'ResponseTime',
        Value: duration,
        Unit: 'Milliseconds',
        Timestamp: new Date()
      }]
    }))
  })
})
```

### Lambda 函式 URL（簡化部署）

```typescript
// sst.config.ts
import { Function, FunctionUrlAuthType } from 'aws-cdk-lib/aws-lambda'

export default {
  stacks(app) {
    app.stack(function Site({ stack }) {
      const fn = new Function(stack, 'NuxtFunction', {
        runtime: Runtime.NODEJS_20_X,
        handler: 'index.handler',
        code: Code.fromAsset('.output/server/')
      })

      // 啟用 Function URL（無需 API Gateway）
      const fnUrl = fn.addFunctionUrl({
        authType: FunctionUrlAuthType.NONE
      })

      stack.addOutputs({
        FunctionUrl: fnUrl.url
      })
    })
  }
} satisfies SSTConfig
```

### 成本最佳化建議

```typescript
// nuxt.config.ts — 減少 Lambda 執行時間
export default defineNuxtConfig({
  nitro: {
    preset: 'aws-lambda',

    // 啟用輸出壓縮
    compressPublicAssets: true,

    // 預渲染靜態頁面（減少 Lambda 調用）
    prerender: {
      routes: ['/', '/about', '/contact']
    }
  },

  routeRules: {
    // 靜態頁面不呼叫 Lambda
    '/blog/**': { prerender: true },

    // API 路由使用較短快取
    '/api/**': {
      cache: {
        maxAge: 60 // 1 分鐘
      }
    }
  }
})
```

> **AWS Lambda 最佳實踐**：
> - 使用 Provisioned Concurrency 避免重要端點的冷啟動
> - 外部化大型依賴到 Lambda Layers
> - 使用 RDS Proxy 管理資料庫連線池
> - 靜態資源託管在 S3 + CloudFront
> - 監控 CloudWatch 指標（執行時間、錯誤率、記憶體使用）
> - 設定合理的記憶體大小（記憶體越大，CPU 越快，但成本越高）

> **⚠️ SST 版本提醒**：上述關於 Lambda Layer、RDS Proxy、S3+CloudFront、Provisioned Concurrency 的 SST 範例均基於 **SST v2** 語法。若您使用 SST v3 (Ion)，請參考 [SST v3 官方文件](https://sst.dev/docs/)，因為 API 已完全改變（從 CDK constructs 遷移至 Pulumi）。

---

## PM2 程序管理

### 安裝 PM2

```bash
# 全域安裝 PM2
npm install -g pm2

# 或在專案中安裝
npm install --save-dev pm2
```

### PM2 配置檔案

```javascript
// ecosystem.config.cjs
module.exports = {
  apps: [{
    name: 'nuxt-app',
    script: '.output/server/index.mjs',
    instances: 'max',       // 使用所有 CPU 核心
    exec_mode: 'cluster',   // 叢集模式（負載平衡）

    env_production: {
      NODE_ENV: 'production',
      NITRO_PORT: 3000,
      NITRO_HOST: '0.0.0.0'
    },

    env_development: {
      NODE_ENV: 'development',
      NITRO_PORT: 3001
    },

    // 日誌管理
    error_file: './logs/error.log',
    out_file: './logs/output.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,

    // 記憶體管理
    max_memory_restart: '512M',  // 超過 512MB 自動重啟

    // 自動重啟設定
    watch: false,                // 生產環境不監聽檔案變化
    ignore_watch: ['node_modules', 'logs', '.nuxt'],

    // 重啟策略
    min_uptime: '10s',          // 最小運行時間（避免頻繁重啟）
    max_restarts: 10,           // 最大重啟次數
    restart_delay: 4000,        // 重啟延遲（毫秒）

    // Cron 重啟（可選）
    cron_restart: '0 3 * * *',  // 每天凌晨 3 點重啟

    // 環境特定設定
    instance_var: 'INSTANCE_ID'
  }],

  // 部署設定（選用）
  deploy: {
    production: {
      user: 'deploy',
      host: ['server1.example.com', 'server2.example.com'],
      ref: 'origin/main',
      repo: 'git@github.com:username/nuxt-app.git',
      path: '/var/www/nuxt-app',
      'post-deploy': 'npm install && npm run build && pm2 reload ecosystem.config.cjs --env production'
    }
  }
}
```

### 基本指令

```bash
# 建置應用
npm run build

# 啟動應用
pm2 start ecosystem.config.cjs --env production

# 查看運行狀態
pm2 status
pm2 list

# 查看詳細資訊
pm2 show nuxt-app

# 即時監控
pm2 monit

# 查看日誌
pm2 logs nuxt-app
pm2 logs nuxt-app --lines 100
pm2 logs nuxt-app --err  # 僅錯誤日誌

# 清空日誌
pm2 flush

# 停止應用
pm2 stop nuxt-app

# 重啟應用
pm2 restart nuxt-app

# 零停機重載（叢集模式）
pm2 reload nuxt-app

# 刪除應用
pm2 delete nuxt-app

# 停止所有應用
pm2 stop all

# 重啟所有應用
pm2 restart all
```

### 零停機部署

```bash
#!/bin/bash
# deploy.sh — 零停機部署腳本

set -e  # 遇到錯誤立即停止

echo "🚀 開始部署..."

# 拉取最新程式碼
git pull origin main

# 安裝依賴
echo "📦 安裝依賴..."
npm ci

# 建置應用
echo "🔨 建置應用..."
npm run build

# 零停機重載
echo "🔄 重載應用..."
pm2 reload ecosystem.config.cjs --env production

# 顯示狀態
pm2 status

echo "✅ 部署完成！"
```

```bash
# 賦予執行權限
chmod +x deploy.sh

# 執行部署
./deploy.sh
```

### 開機自啟動

```bash
# 生成啟動腳本
pm2 startup

# 執行輸出的命令（範例）
# sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u deploy --hp /home/deploy

# 儲存當前 PM2 進程列表
pm2 save

# 測試重啟
sudo reboot

# 重啟後檢查
pm2 status
```

### 進階配置：多應用管理

```javascript
// ecosystem.config.cjs — 管理多個應用
module.exports = {
  apps: [
    // 主應用
    {
      name: 'nuxt-app',
      script: '.output/server/index.mjs',
      instances: 4,
      exec_mode: 'cluster',
      env_production: {
        NODE_ENV: 'production',
        NITRO_PORT: 3000
      }
    },

    // 背景任務處理器
    {
      name: 'worker',
      script: './workers/queue-processor.js',
      instances: 2,
      exec_mode: 'cluster',
      env_production: {
        NODE_ENV: 'production'
      }
    },

    // 排程任務
    {
      name: 'cron-jobs',
      script: './workers/cron.js',
      instances: 1,  // 排程任務通常只需一個實例
      exec_mode: 'fork',
      cron_restart: '0 * * * *'  // 每小時重啟
    }
  ]
}
```

### 監控與告警

```bash
# 安裝 PM2 監控模組
pm2 install pm2-logrotate

# 配置日誌輪替
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
pm2 set pm2-logrotate:dateFormat YYYY-MM-DD_HH-mm-ss
```

### PM2 Plus（進階監控）

```bash
# 連接 PM2 Plus（需註冊）
pm2 link <secret_key> <public_key>

# 查看儀表板
# https://app.pm2.io
```

### Nginx 反向代理整合

```nginx
# /etc/nginx/sites-available/nuxt-app
upstream nuxt_backend {
    least_conn;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    server_name example.com;

    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # 靜態資源（若使用 CDN 可省略）
    location /_nuxt/ {
        alias /var/www/nuxt-app/.output/public/_nuxt/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 代理到 PM2 叢集
    location / {
        proxy_pass http://nuxt_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # 超時設定
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### 多端口叢集配置

```javascript
// ecosystem.config.cjs — 使用多端口
module.exports = {
  apps: [{
    name: 'nuxt-app',
    script: '.output/server/index.mjs',
    instances: 4,
    exec_mode: 'cluster',

    // 動態分配端口
    env_production: {
      NODE_ENV: 'production',
      NITRO_PORT: 3000  // 第一個實例用 3000，之後遞增
    },

    // 使用環境變數模板
    instance_var: 'INSTANCE_ID',

    // 每個實例使用不同端口
    increment_var: 'NITRO_PORT'
  }]
}
```

### 健康檢查腳本

```javascript
// healthcheck.js
const http = require('http')

const options = {
  hostname: 'localhost',
  port: 3000,
  path: '/api/_health',
  method: 'GET',
  timeout: 5000
}

const req = http.request(options, (res) => {
  if (res.statusCode === 200) {
    console.log('✅ 健康檢查通過')
    process.exit(0)
  } else {
    console.error(`❌ 健康檢查失敗: ${res.statusCode}`)
    process.exit(1)
  }
})

req.on('error', (error) => {
  console.error('❌ 健康檢查失敗:', error.message)
  process.exit(1)
})

req.on('timeout', () => {
  console.error('❌ 健康檢查逾時')
  req.destroy()
  process.exit(1)
})

req.end()
```

```bash
# 定期執行健康檢查（crontab）
*/5 * * * * /usr/bin/node /var/www/nuxt-app/healthcheck.js || pm2 restart nuxt-app
```

### 效能調校

```javascript
// ecosystem.config.cjs — 效能最佳化
module.exports = {
  apps: [{
    name: 'nuxt-app',
    script: '.output/server/index.mjs',
    instances: 'max',
    exec_mode: 'cluster',

    // Node.js 記憶體選項
    node_args: '--max-old-space-size=2048 --max-http-header-size=16384',

    // 環境變數
    env_production: {
      NODE_ENV: 'production',
      NITRO_PORT: 3000,

      // 啟用 Node.js 效能選項
      NODE_OPTIONS: '--max-old-space-size=2048'
    },

    // 自動重啟條件
    max_memory_restart: '1G',
    min_uptime: '30s',
    max_restarts: 5,

    // 啟用叢集模式的優雅重載
    kill_timeout: 5000,
    listen_timeout: 10000,
    shutdown_with_message: true
  }]
}
```

> **PM2 最佳實踐**：
> - 生產環境使用叢集模式提高吞吐量
> - 設定合理的記憶體限制避免 OOM
> - 使用日誌輪替避免磁碟空間耗盡
> - 配置開機自啟動確保服務可用性
> - 搭配 Nginx 做負載平衡與 SSL 終止
> - 定期執行健康檢查並自動重啟失敗實例

---

## CI/CD Pipeline

### GitHub Actions 完整工作流程

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

# 環境變數
env:
  NODE_VERSION: 20
  PNPM_VERSION: 8

jobs:
  # 程式碼檢查與型別檢查
  lint-and-typecheck:
    name: 程式碼檢查與型別檢查
    runs-on: ubuntu-latest

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 安裝 pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: 設定 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'

      - name: 安裝依賴
        run: pnpm install --frozen-lockfile

      - name: 執行 Lint
        run: pnpm lint

      - name: 執行型別檢查
        run: pnpm typecheck

  # 單元測試與整合測試
  test:
    name: 執行測試
    runs-on: ubuntu-latest
    needs: lint-and-typecheck

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 安裝 pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: 設定 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'

      - name: 安裝依賴
        run: pnpm install --frozen-lockfile

      - name: 執行資料庫遷移
        run: pnpm prisma migrate deploy
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb

      - name: 執行測試
        run: pnpm test:ci
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb

      - name: 上傳測試覆蓋率
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info

  # 建置應用
  build:
    name: 建置應用
    runs-on: ubuntu-latest
    needs: test

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 安裝 pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: 設定 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'

      - name: 安裝依賴
        run: pnpm install --frozen-lockfile

      - name: 生成 Prisma Client
        run: pnpm prisma generate

      - name: 建置應用
        run: pnpm build
        env:
          NUXT_PUBLIC_API_BASE: ${{ secrets.API_BASE_URL }}

      - name: 上傳建置產物
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: .output/
          retention-days: 7

  # Preview 環境部署（PR 時）
  deploy-preview:
    name: 部署預覽環境
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: preview
      url: ${{ steps.deploy.outputs.url }}

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 下載建置產物
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: .output/

      - name: 部署到 Vercel Preview
        id: deploy
        run: |
          npm i -g vercel
          vercel deploy --prebuilt --token=${{ secrets.VERCEL_TOKEN }} > deployment-url.txt
          echo "url=$(cat deployment-url.txt)" >> $GITHUB_OUTPUT

      - name: 評論 PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🚀 預覽環境已部署: ${{ steps.deploy.outputs.url }}`
            })

  # 生產環境部署（main 分支）
  deploy-production:
    name: 部署生產環境
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: production
      url: https://example.com

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 安裝 pnpm
        uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - name: 設定 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'

      - name: 下載建置產物
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: .output/

      - name: 執行資料庫遷移
        run: |
          pnpm install --frozen-lockfile
          pnpm prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: 部署到生產環境
        run: |
          # 依據您的部署平台調整
          # Vercel
          npm i -g vercel
          vercel deploy --prod --prebuilt --token=${{ secrets.VERCEL_TOKEN }}

          # 或 SSH 部署到自有伺服器
          # - name: 部署到伺服器
          #   uses: appleboy/ssh-action@master
          #   with:
          #     host: ${{ secrets.HOST }}
          #     username: ${{ secrets.USERNAME }}
          #     key: ${{ secrets.SSH_PRIVATE_KEY }}
          #     script: |
          #       cd /var/www/nuxt-app
          #       git pull origin main
          #       pnpm install --frozen-lockfile
          #       pnpm build
          #       pm2 reload ecosystem.config.cjs

      - name: 通知部署成功
        uses: 8398a7/action-slack@v3
        if: success()
        with:
          status: custom
          custom_payload: |
            {
              text: '✅ 生產環境部署成功',
              attachments: [{
                color: 'good',
                text: `Commit: ${{ github.sha }}\nAuthor: ${{ github.actor }}`
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: 通知部署失敗
        uses: 8398a7/action-slack@v3
        if: failure()
        with:
          status: custom
          custom_payload: |
            {
              text: '❌ 生產環境部署失敗',
              attachments: [{
                color: 'danger',
                text: `Commit: ${{ github.sha }}\nAuthor: ${{ github.actor }}`
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # E2E 測試（選用）
  e2e-test:
    name: E2E 測試
    runs-on: ubuntu-latest
    needs: deploy-preview
    if: github.event_name == 'pull_request'

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 安裝 pnpm
        uses: pnpm/action-setup@v4

      - name: 設定 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'pnpm'

      - name: 安裝依賴
        run: pnpm install --frozen-lockfile

      - name: 安裝 Playwright
        run: pnpm exec playwright install --with-deps

      - name: 執行 E2E 測試
        run: pnpm test:e2e
        env:
          BASE_URL: ${{ needs.deploy-preview.outputs.url }}

      - name: 上傳測試報告
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

### 依賴更新自動化

```yaml
# .github/workflows/dependency-update.yml
name: 依賴更新檢查

on:
  schedule:
    - cron: '0 0 * * 1'  # 每週一午夜執行
  workflow_dispatch:      # 手動觸發

jobs:
  update-dependencies:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 安裝 pnpm
        uses: pnpm/action-setup@v4

      - name: 設定 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - name: 更新依賴
        run: |
          pnpm update --latest
          pnpm audit fix

      - name: 執行測試
        run: pnpm test

      - name: 建立 Pull Request
        uses: peter-evans/create-pull-request@v6
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: 'chore: 更新依賴套件'
          title: '🔄 自動更新依賴套件'
          body: |
            自動化依賴更新 Bot 已更新以下套件：

            請檢查測試是否通過後合併。
          branch: deps/auto-update
          delete-branch: true
```

### 快取最佳化

```yaml
# .github/workflows/ci.yml — 快取最佳化
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      # 快取 Nuxt 建置產物
      - name: 快取 Nuxt 建置
        uses: actions/cache@v4
        with:
          path: |
            .nuxt
            .output
          key: ${{ runner.os }}-nuxt-${{ hashFiles('**/pnpm-lock.yaml') }}-${{ hashFiles('**/*.vue', '**/*.ts') }}
          restore-keys: |
            ${{ runner.os }}-nuxt-${{ hashFiles('**/pnpm-lock.yaml') }}-
            ${{ runner.os }}-nuxt-

      # 快取 Playwright
      - name: 快取 Playwright
        uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: ${{ runner.os }}-playwright-${{ hashFiles('**/pnpm-lock.yaml') }}

      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

### Docker 建置與推送

```yaml
# .github/workflows/docker.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
    tags:
      - 'v*'

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout 程式碼
        uses: actions/checkout@v4

      - name: 設定 Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: 登入 Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: 提取 metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: username/nuxt-app
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: 建置並推送
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=username/nuxt-app:buildcache
          cache-to: type=registry,ref=username/nuxt-app:buildcache,mode=max
```

### 分支保護規則建議

在 GitHub Repository Settings > Branches 中設定：

- **Require pull request reviews before merging**: 至少 1 位審查者
- **Require status checks to pass before merging**: 勾選所有 CI jobs
- **Require branches to be up to date before merging**: 確保分支最新
- **Include administrators**: 管理員也遵守規則
- **Require linear history**: 保持提交歷史線性

### 環境變數管理

```yaml
# 在 GitHub Repository Settings > Secrets and variables > Actions 中設定：

# Secrets（敏感資訊）:
DATABASE_URL
JWT_SECRET
VERCEL_TOKEN
DOCKERHUB_TOKEN
SLACK_WEBHOOK
SSH_PRIVATE_KEY

# Variables（非敏感配置）:
API_BASE_URL
NODE_VERSION
PNPM_VERSION
```

### GitLab CI/CD 範例

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "20"
  PNPM_VERSION: "8"

# 快取設定
cache:
  key:
    files:
      - pnpm-lock.yaml
  paths:
    - .pnpm-store
    - node_modules

before_script:
  - corepack enable
  - corepack prepare pnpm@${PNPM_VERSION} --activate
  - pnpm config set store-dir .pnpm-store
  - pnpm install --frozen-lockfile

# Lint
lint:
  stage: lint
  image: node:${NODE_VERSION}
  script:
    - pnpm lint
    - pnpm typecheck

# Test
test:
  stage: test
  image: node:${NODE_VERSION}
  services:
    - postgres:16
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://test:test@postgres:5432/testdb
  script:
    - pnpm prisma migrate deploy
    - pnpm test:ci
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# Build
build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - pnpm build
  artifacts:
    paths:
      - .output/
    expire_in: 1 week

# Deploy to production
deploy:production:
  stage: deploy
  image: alpine:latest
  only:
    - main
  before_script:
    - apk add --no-cache openssh-client
  script:
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan $DEPLOY_HOST >> ~/.ssh/known_hosts
    - ssh $DEPLOY_USER@$DEPLOY_HOST "cd /var/www/nuxt-app && ./deploy.sh"
  environment:
    name: production
    url: https://example.com
```

> **CI/CD 最佳實踐**：
> - 分離 lint、test、build 階段以快速失敗
> - 使用快取加速建置（pnpm store、Nuxt cache）
> - PR 時部署預覽環境供審查
> - 合併到 main 後自動部署生產環境
> - 設定分支保護規則確保程式碼品質
> - 整合 Slack/Discord 通知部署狀態
> - 使用 Secrets 管理敏感環境變數

---

## 環境設定管理

### Runtime Config

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 僅伺服器端可存取
    // 注意：NUXT_ 前綴環境變數會自動覆蓋對應欄位
    // NUXT_JWT_SECRET → runtimeConfig.jwtSecret
    jwtSecret: '',  // 由 NUXT_JWT_SECRET 環境變數覆蓋
    dbUrl: '',      // 由 NUXT_DB_URL 環境變數覆蓋
    // public 中的值可在客戶端存取
    // NUXT_PUBLIC_ 前綴環境變數同樣會自動覆蓋，此處只需設定預設值
    public: {
      apiBase: 'http://localhost:3000',   // 由 NUXT_PUBLIC_API_BASE 覆蓋
      appName: '我的應用'                 // 由 NUXT_PUBLIC_APP_NAME 覆蓋
    }
  }
})
```

### 使用 Runtime Config

```typescript
// 伺服器端 — 可存取所有設定
// server/api/auth/login.post.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig(event)  // 伺服器端必須傳入 event 以支援 runtime 環境變數覆蓋
  const token = signJWT(user, config.jwtSecret)
  return { token }
})

// 客戶端 — 僅可存取 public 設定
// composables/useApi.ts
export function useApi() {
  const config = useRuntimeConfig()
  return $fetch.create({
    baseURL: config.public.apiBase
  })
}
```

### .env 檔案

```env
# .env — 開發環境
NUXT_JWT_SECRET=dev-secret-key
NUXT_DB_URL=postgresql://localhost:5432/mydb
NUXT_PUBLIC_API_BASE=http://localhost:3000
NUXT_PUBLIC_APP_NAME=MyApp Dev

# 注意：NUXT_ 前綴的環境變數會自動映射到 runtimeConfig
# NUXT_JWT_SECRET → runtimeConfig.jwtSecret
# NUXT_PUBLIC_API_BASE → runtimeConfig.public.apiBase
```
