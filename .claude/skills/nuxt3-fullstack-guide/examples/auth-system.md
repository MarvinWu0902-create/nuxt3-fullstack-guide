# 實戰範例：完整認證系統

## 概述

JWT Token 認證流程：登入 → Cookie 儲存 Token → 自動帶 Token → 自動刷新 → 登出。

---

## 型別定義

```typescript
// types/auth.ts
export interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user'
  avatar?: string
  createdAt: string
}

export interface LoginCredentials {
  email: string
  password: string
}

export interface RegisterData extends LoginCredentials {
  name: string
  confirmPassword: string
}

export interface AuthResponse {
  user: User
  token: string
  // refreshToken 透過 HttpOnly cookie 傳遞，不在 response body 中
}
```

## 密碼雜湊與 JWT 密鑰管理

### 密碼雜湊（bcrypt）

> **安全警告**：永遠不要儲存明文密碼。使用 bcrypt 搭配適當的 salt rounds 是目前最廣泛推薦的做法。

```bash
# 安裝 bcrypt（Node.js 原生 binding，效能最佳）
npm install bcrypt
npm install -D @types/bcrypt
```

```typescript
// server/utils/password.ts
import bcrypt from 'bcrypt'

/**
 * Salt rounds 建議值：
 * - 10：一般應用（約 100ms/次，適合多數場景）
 * - 12：高安全需求（約 300ms/次，金融、醫療等）
 * - 14+：極高安全需求（約 1s+/次，注意對回應時間的影響）
 *
 * 每增加 1 輪，計算時間約翻倍。建議在目標伺服器上實測後選定。
 */
const SALT_ROUNDS = 12

/** 雜湊密碼 — 註冊時使用 */
export async function hashPassword(plainPassword: string): Promise<string> {
  return bcrypt.hash(plainPassword, SALT_ROUNDS)
}

/** 驗證密碼 — 登入時使用 */
export async function verifyPassword(
  plainPassword: string,
  hashedPassword: string
): Promise<boolean> {
  return bcrypt.compare(plainPassword, hashedPassword)
}
```

```typescript
// 在註冊 API 中使用
// server/api/auth/register.post.ts
export default defineEventHandler(async (event) => {
  const { name, email, password } = await readBody(event)

  // 檢查使用者是否已存在
  const existing = await db.user.findUnique({ where: { email } })
  if (existing) {
    throw createError({ statusCode: 409, message: '此 Email 已被註冊' })
  }

  // 使用 bcrypt 雜湊密碼後儲存
  const passwordHash = await hashPassword(password)
  const user = await db.user.create({
    data: { name, email, passwordHash }
  })

  return { user: { id: user.id, name: user.name, email: user.email } }
})
```

### JWT 密鑰管理與算法選擇

```typescript
// server/utils/jwt.ts
import jwt from 'jsonwebtoken'

/**
 * === 算法選擇指南 ===
 *
 * HS256（HMAC + SHA-256）— 對稱式
 *   - 簽發與驗證使用同一把 secret
 *   - 適合：單體應用、所有服務共享同一密鑰的架構
 *   - Secret 最小長度：32 bytes（256 bits），建議 64 bytes
 *   - 範例產生方式：openssl rand -base64 64
 *
 * RS256（RSA + SHA-256）— 非對稱式
 *   - 簽發使用 private key，驗證使用 public key
 *   - 適合：微服務架構（各服務只需 public key 即可驗證）
 *   - 最小 key 長度：2048 bits，建議 4096 bits
 *   - 範例產生方式：openssl genrsa -out private.pem 4096
 *                   openssl rsa -in private.pem -pubout -out public.pem
 */

interface JwtPayload {
  sub: number
  role: string
  type?: string
}

// 從 runtimeConfig 讀取密鑰，避免硬編碼
function getJwtConfig() {
  const config = useRuntimeConfig()

  if (!config.jwtSecret || config.jwtSecret.length < 32) {
    throw new Error(
      'JWT secret 未設定或長度不足。請設定 NUXT_JWT_SECRET 環境變數（至少 32 字元）'
    )
  }

  return {
    secret: config.jwtSecret,
    algorithm: (config.jwtAlgorithm || 'HS256') as jwt.Algorithm
  }
}

/** 簽發 JWT Token */
export function signJWT(payload: JwtPayload, expiresIn: string): string {
  const { secret, algorithm } = getJwtConfig()
  return jwt.sign(payload, secret, { algorithm, expiresIn })
}

/** 驗證並解碼 JWT Token */
export function verifyJWT(token: string): JwtPayload {
  const { secret, algorithm } = getJwtConfig()
  return jwt.verify(token, secret, { algorithms: [algorithm] }) as JwtPayload
}
```

```
# .env — JWT 密鑰配置
# HS256 secret：至少 32 字元（建議 64 字元），使用 openssl rand -base64 64 產生
NUXT_JWT_SECRET=此處填入至少-32-字元的隨機密鑰-請用-openssl-rand-base64-64-產生
# 可選：切換算法（預設 HS256）
# NUXT_JWT_ALGORITHM=HS256
```

```typescript
// nuxt.config.ts — 對應的 runtimeConfig 設定
export default defineNuxtConfig({
  runtimeConfig: {
    jwtSecret: '',       // 由 NUXT_JWT_SECRET 覆蓋
    jwtAlgorithm: 'HS256' // 由 NUXT_JWT_ALGORITHM 覆蓋
  }
})
```

> **安全警告**：JWT secret 絕不可提交至版本控制。請透過環境變數或 Secret Manager（如 AWS Secrets Manager、HashiCorp Vault）注入。生產環境必須使用高強度隨機密鑰（`openssl rand -base64 64`），切勿使用可預測的字串。

---

## Server API

```typescript
// server/api/auth/login.post.ts
import { z } from 'zod'

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8)
})

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const result = loginSchema.safeParse(body)
  if (!result.success) {
    throw createError({ statusCode: 400, message: '輸入格式錯誤' })
  }
  const { email, password } = result.data

  const user = await db.user.findUnique({ where: { email } })
  if (!user || !await verifyPassword(password, user.passwordHash)) {
    throw createError({ statusCode: 401, message: '帳號或密碼錯誤' })
  }

  const token = signJWT({ sub: user.id, role: user.role }, '15m')
  const refreshToken = signJWT({ sub: user.id, role: user.role, type: 'refresh' }, '7d')

  // 設定 HttpOnly Cookie（更安全）
  setCookie(event, 'refresh-token', refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    maxAge: 60 * 60 * 24 * 7
  })

  return {
    user: { id: user.id, name: user.name, email: user.email, role: user.role },
    token
  }
})
```

```typescript
// server/api/auth/me.get.ts
export default defineEventHandler(async (event) => {
  const auth = event.context.auth
  if (!auth) throw createError({ statusCode: 401, message: '未認證' })

  const user = await db.user.findUnique({
    where: { id: auth.userId },
    select: { id: true, name: true, email: true, role: true, avatar: true, createdAt: true }
  })
  if (!user) throw createError({ statusCode: 404, message: '用戶不存在' })
  return user
})
```

```typescript
// server/api/auth/refresh.post.ts
export default defineEventHandler(async (event) => {
  const refreshToken = getCookie(event, 'refresh-token')
  if (!refreshToken) {
    throw createError({ statusCode: 401, message: '缺少 Refresh Token' })
  }

  let payload: ReturnType<typeof verifyJWT>
  try {
    payload = verifyJWT(refreshToken)
    if (payload.type !== 'refresh') throw new Error('非 Refresh Token')
  } catch {
    // JWT 驗證失敗：清除無效的 refresh token
    deleteCookie(event, 'refresh-token', { httpOnly: true, secure: true, sameSite: 'strict' })
    throw createError({ statusCode: 401, message: 'Refresh Token 已過期' })
  }

  // 從資料庫重新查詢使用者，取得最新 role（避免使用 token 中過時的 role）
  const user = await db.user.findUnique({ where: { id: payload.sub } })
  if (!user) {
    deleteCookie(event, 'refresh-token', { httpOnly: true, secure: true, sameSite: 'strict' })
    throw createError({ statusCode: 401, message: '使用者不存在' })
  }

  const newToken = signJWT({ sub: user.id, role: user.role }, '15m')
  return { token: newToken }
})
```

## Pinia Store

```typescript
// stores/auth.ts
import type { User, LoginCredentials } from '~/types/auth'

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const token = useCookie('auth-token', {
    maxAge: 60 * 15,
    secure: true,
    sameSite: 'strict'
    // 注意：此 cookie 未設 httpOnly，因為客戶端需讀取 token 來設定 Authorization header
    // 確保搭配嚴格的 CSP 防護 XSS
  })
  const refreshTimer = ref<ReturnType<typeof setTimeout>>()

  const isLoggedIn = computed(() => !!token.value && !!user.value)
  const isAdmin = computed(() => user.value?.role === 'admin')

  async function login(credentials: LoginCredentials) {
    const response = await $fetch<{ user: User; token: string }>('/api/auth/login', {
      method: 'POST',
      body: credentials
    })
    user.value = response.user
    token.value = response.token
    scheduleRefresh()
  }

  async function fetchUser() {
    if (!token.value) return
    try {
      user.value = await $fetch<User>('/api/auth/me', {
        headers: { Authorization: `Bearer ${token.value}` }
      })
      scheduleRefresh()
    } catch {
      await logout()
    }
  }

  // === Token Refresh 併發控制說明 ===
  // 問題：當 access token 過期時，多個並行的 API 請求會同時觸發 refresh
  // 例如頁面載入時同時發出 3 個 API 請求 → 3 個都收到 401 → 3 個都嘗試 refresh token
  //
  // isRefreshing 機制的解法：
  // 1. 第一個 401 請求設定 isRefreshing = true 並發起 refresh
  // 2. 後續的 401 請求看到 isRefreshing = true，不再發起新的 refresh
  //    而是等待（透過 Promise/queue）第一個 refresh 完成
  // 3. refresh 完成後，所有等待中的請求使用新的 token 重試
  //
  // 這避免了：
  // - 多次無效的 refresh 請求（refresh token 可能是一次性的）
  // - Race condition 導致 refresh token 被消耗但部分請求未更新
  let isRefreshing = false
  async function refreshAccessToken() {
    // 若使用者已登出或正在刷新中，不重複呼叫
    if (!token.value || isRefreshing) return
    isRefreshing = true

    try {
      const { token: newToken } = await $fetch<{ token: string }>('/api/auth/refresh', {
        method: 'POST'
      })
      // 再次檢查：fetch 期間使用者可能已登出
      if (!token.value) return

      token.value = newToken
      scheduleRefresh()
    } catch {
      await logout()
    } finally {
      isRefreshing = false
    }
  }

  function scheduleRefresh() {
    // 僅在客戶端排程（SSR 中 setTimeout 會造成記憶體洩漏）
    if (!import.meta.client) return
    if (refreshTimer.value) clearTimeout(refreshTimer.value)
    // Token 到期前 1 分鐘刷新
    refreshTimer.value = setTimeout(refreshAccessToken, 14 * 60 * 1000)
  }

  async function logout() {
    if (refreshTimer.value) clearTimeout(refreshTimer.value)
    user.value = null
    token.value = null
    await navigateTo('/login')
  }

  return {
    user: readonly(user),
    isLoggedIn,
    isAdmin,
    login,
    fetchUser,
    logout
  }
})
```

## 中介層

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to) => {
  const authStore = useAuthStore()

  if (to.meta.requiresAuth && !authStore.isLoggedIn) {
    return navigateTo({
      path: '/login',
      query: { redirect: to.fullPath }
    })
  }

  if (to.meta.guestOnly && authStore.isLoggedIn) {
    return navigateTo('/dashboard')
  }

  if (to.meta.requiredRole && typeof to.meta.requiredRole === 'string' && authStore.user?.role !== to.meta.requiredRole) {
    return abortNavigation(createError({ statusCode: 403, message: '權限不足' }))
  }
})
```

## 外掛

```typescript
// plugins/01.auth.ts
export default defineNuxtPlugin(async () => {
  const authStore = useAuthStore()

  // 僅在伺服器端初始化（資料會透過 Pinia payload 傳遞到客戶端）
  if (import.meta.server) {
    const token = useCookie('auth-token')
    if (token.value && !authStore.user) {
      await authStore.fetchUser()
    }
  }
})
```

## Type Guard 工具函式

```typescript
// utils/typeGuards.ts

/** $fetch 拋出的錯誤結構（FetchError from ofetch） */
interface NuxtFetchError {
  statusCode?: number
  statusMessage?: string
  data?: unknown
  message: string
}

/** 判斷 unknown error 是否為 $fetch / useFetch 拋出的錯誤 */
export function isNuxtFetchError(error: unknown): error is NuxtFetchError {
  return (
    error instanceof Error &&
    'statusCode' in error
  )
}
```

## 登入頁面

```vue
<!-- pages/login.vue -->
<script setup lang="ts">
definePageMeta({
  middleware: 'auth',
  guestOnly: true,
  layout: 'auth'
})

const authStore = useAuthStore()
const route = useRoute()

const form = ref({
  email: '',
  password: ''
})
const errors = ref<Record<string, string>>({})
const isSubmitting = ref(false)

async function handleLogin() {
  errors.value = {}
  isSubmitting.value = true

  try {
    await authStore.login(form.value)
    // 防止 Open Redirect：只允許相對路徑，禁止絕對 URL 和 protocol-relative URL
    const rawRedirect = route.query.redirect
    const redirectParam = Array.isArray(rawRedirect) ? rawRedirect[0] : rawRedirect
    const redirect = redirectParam && redirectParam.startsWith('/') && !redirectParam.startsWith('//')
      ? redirectParam
      : '/dashboard'
    await navigateTo(redirect)
  } catch (error: unknown) {
    if (isNuxtFetchError(error)) {
      if (error.statusCode === 401) {
        errors.value.general = '帳號或密碼錯誤'
      } else if (error.data && typeof error.data === 'object') {
        errors.value = error.data as Record<string, string>
      } else {
        errors.value.general = '發生未知錯誤，請稍後再試'
      }
    } else {
      errors.value.general = '發生未知錯誤，請稍後再試'
    }
  } finally {
    isSubmitting.value = false
  }
}
</script>

<template>
  <div class="min-h-screen flex items-center justify-center bg-gray-50">
    <div class="w-full max-w-md p-8 bg-white rounded-2xl shadow-lg">
      <h1 class="text-2xl font-bold text-center mb-8">登入</h1>

      <div v-if="errors.general" class="mb-4 p-3 bg-red-50 text-red-600 rounded-lg text-sm">
        {{ errors.general }}
      </div>

      <form @submit.prevent="handleLogin" class="space-y-6">
        <div>
          <label for="email" class="block text-sm font-medium text-gray-700 mb-1">
            電子郵件
          </label>
          <input
            id="email"
            v-model="form.email"
            type="email"
            required
            autocomplete="email"
            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            placeholder="you@example.com"
          />
        </div>

        <div>
          <label for="password" class="block text-sm font-medium text-gray-700 mb-1">
            密碼
          </label>
          <input
            id="password"
            v-model="form.password"
            type="password"
            required
            autocomplete="current-password"
            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            placeholder="至少 8 個字元"
          />
        </div>

        <BaseButton
          type="submit"
          variant="primary"
          :loading="isSubmitting"
          class="w-full"
        >
          登入
        </BaseButton>
      </form>
    </div>
  </div>
</template>
```

## 測試

```typescript
// tests/unit/stores/auth.test.ts
import { setActivePinia, createPinia } from 'pinia'
import { useAuthStore } from '~/stores/auth'
import { registerEndpoint, mockNuxtImport } from '@nuxt/test-utils/runtime'

// navigateTo 在 logout() 中被呼叫，必須 mock
mockNuxtImport('navigateTo', () => vi.fn())

describe('useAuthStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  // registerEndpoint 可在 describe 層級呼叫，作用於整個測試套件
  registerEndpoint('/api/auth/login', () => ({
    user: { id: 1, name: '測試用戶', email: 'test@test.com', role: 'user' },
    token: 'mock-jwt'
  }))

  registerEndpoint('/api/auth/me', () => ({
    id: 1, name: '測試用戶', email: 'test@test.com', role: 'user'
  }))

  it('登入後設定用戶和 token', async () => {
    const store = useAuthStore()
    await store.login({ email: 'test@test.com', password: '12345678' })
    expect(store.isLoggedIn).toBe(true)
    expect(store.user?.name).toBe('測試用戶')
  })

  it('登出後清除狀態', async () => {
    const store = useAuthStore()
    await store.login({ email: 'test@test.com', password: '12345678' })
    await store.logout()
    expect(store.isLoggedIn).toBe(false)
    expect(store.user).toBeNull()
  })
})
```

## OAuth 社交登入

```bash
# 使用 nuxt-auth-utils 套件（由 Nuxt 團隊維護）
npx nuxi module add nuxt-auth-utils
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-auth-utils']
  // OAuth 密鑰透過環境變數設定，不需要在 config 中配置
})
```

```
# .env
NUXT_OAUTH_GOOGLE_CLIENT_ID=your-client-id
NUXT_OAUTH_GOOGLE_CLIENT_SECRET=your-client-secret
NUXT_OAUTH_GITHUB_CLIENT_ID=your-client-id
NUXT_OAUTH_GITHUB_CLIENT_SECRET=your-client-secret
NUXT_SESSION_PASSWORD=至少-32-字元的隨機密碼
```

```typescript
// server/routes/auth/google.get.ts
export default defineOAuthGoogleEventHandler({
  config: {
    scope: ['openid', 'email', 'profile']
  },
  async onSuccess(event, { user: oauthUser }) {
    // 查找或建立使用者
    const db = usePrisma()
    let user = await db.user.findUnique({
      where: { email: oauthUser.email }
    })

    if (!user) {
      user = await db.user.create({
        data: {
          email: oauthUser.email,
          name: oauthUser.name,
          avatarUrl: oauthUser.picture,
          provider: 'google'
        }
      })
    }

    // 設定 session
    await setUserSession(event, {
      user: { id: user.id, name: user.name, email: user.email }
    })

    return sendRedirect(event, '/dashboard')
  },
  onError(event, error) {
    console.error('Google OAuth 錯誤:', error)
    return sendRedirect(event, '/login?error=oauth')
  }
})
```

```typescript
// server/routes/auth/github.get.ts
export default defineOAuthGitHubEventHandler({
  config: {
    scope: ['user:email']
  },
  async onSuccess(event, { user: oauthUser }) {
    const db = usePrisma()
    let user = await db.user.findUnique({
      where: { email: oauthUser.email }
    })

    if (!user) {
      user = await db.user.create({
        data: {
          email: oauthUser.email,
          name: oauthUser.login,
          avatarUrl: oauthUser.avatar_url,
          provider: 'github'
        }
      })
    }

    await setUserSession(event, {
      user: { id: user.id, name: user.name, email: user.email }
    })

    return sendRedirect(event, '/dashboard')
  }
})
```

```vue
<!-- components/OAuthButtons.vue -->
<template>
  <div class="space-y-3">
    <a href="/auth/google" class="flex items-center justify-center gap-2 w-full px-4 py-3 border rounded-lg hover:bg-gray-50">
      <svg class="w-5 h-5" viewBox="0 0 24 24"><!-- Google icon --></svg>
      使用 Google 登入
    </a>
    <a href="/auth/github" class="flex items-center justify-center gap-2 w-full px-4 py-3 bg-gray-900 text-white rounded-lg hover:bg-gray-800">
      <svg class="w-5 h-5" viewBox="0 0 24 24"><!-- GitHub icon --></svg>
      使用 GitHub 登入
    </a>
  </div>
</template>
```

```vue
<!-- 在頁面或元件中使用 session -->
<script setup lang="ts">
// useUserSession 由 nuxt-auth-utils 提供
const { loggedIn, user, session, clear: logout } = useUserSession()
</script>

<template>
  <div v-if="loggedIn">
    <p>歡迎，{{ user.name }}</p>
    <button @click="logout()">登出</button>
  </div>
</template>
```
