# 韌性與安全工程

## 目錄

1. [Resilience Patterns](#resilience-patterns)
   - [Failure Scenarios 分析](#failure-scenarios-分析)
   - [Retry Pattern（指數退避）](#retry-pattern指數退避)
   - [Circuit Breaker](#circuit-breaker)
   - [Graceful Degradation](#graceful-degradation)
   - [Bulkhead Pattern](#bulkhead-pattern)
2. [Security Engineering](#security-engineering)
   - [Threat Model（STRIDE）](#threat-modelstride)
   - [JWT Hardening](#jwt-hardening)
   - [Password Storage](#password-storage)
   - [CSRF Protection](#csrf-protection)
   - [CSP Nonce](#csp-nonce)
   - [Secrets Management](#secrets-management)
3. [JWT RS256/ES256 完整實作](#jwt-rs256es256-完整實作)
   - [JWT `kid` Header Matching 驗證](#jwt-kidkey-id-header-matching-驗證)
4. [OAuth 2.0 PKCE（Proof Key for Code Exchange）流程](#oauth-20-pkceproof-key-for-code-exchange流程)

---

## Resilience Patterns

### Failure Scenarios 分析

| 故障場景 | 影響範圍 | 偵測方式 | 應對策略 |
|---------|---------|---------|---------|
| API 超時 | 單一請求阻塞 | 請求計時器逾時 | Retry + Timeout 上限 |
| 資料庫連線中斷 | 所有讀寫失敗 | 連線池健康檢查 | Circuit Breaker + Cache Fallback |
| 第三方服務不可用 | 依賴功能失效 | HTTP 5xx / 連線拒絕 | Graceful Degradation + 佇列重試 |
| CDN 故障 | 靜態資源載入失敗 | 資源載入錯誤事件 | 多 CDN 備援 + 本地 fallback |

### Retry Pattern（指數退避）

```typescript
// server/utils/retry.ts
interface RetryOptions {
  maxRetries: number
  baseDelayMs: number
  maxDelayMs: number
  jitterFactor: number // 0~1，隨機抖動係數
  retryableErrors?: (error: unknown) => boolean
}

const defaultRetryOptions: RetryOptions = {
  maxRetries: 3,
  baseDelayMs: 200,
  maxDelayMs: 10_000,
  jitterFactor: 0.5,
}

export async function withRetry<T>(
  fn: () => Promise<T>,
  opts: Partial<RetryOptions> = {},
): Promise<T> {
  const options = { ...defaultRetryOptions, ...opts }
  let lastError: unknown
  for (let attempt = 0; attempt <= options.maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error
      if (options.retryableErrors && !options.retryableErrors(error)) throw error
      if (attempt === options.maxRetries) break
      // 指數退避 + jitter
      const exponentialDelay = options.baseDelayMs * Math.pow(2, attempt)
      const jitter = exponentialDelay * options.jitterFactor * Math.random()
      const delay = Math.min(exponentialDelay + jitter, options.maxDelayMs)
      await new Promise((resolve) => setTimeout(resolve, delay))
    }
  }
  throw lastError
}

// server/utils/http-client.ts — 包裝 $fetch 加上自動重試
export async function resilientFetch<T>(
  url: string,
  options: Parameters<typeof $fetch>[1] = {},
): Promise<T> {
  return withRetry<T>(
    () => $fetch<T>(url, { ...options, timeout: 5_000 }),
    {
      maxRetries: 3,
      retryableErrors: (error) => {
        if (error instanceof Error && 'statusCode' in error) {
          const code = (error as any).statusCode as number
          return code >= 500 || code === 408 || code === 429 // 僅 5xx/408/429 重試
        }
        return true // 網路錯誤一律重試
      },
    },
  )
}
```

### Circuit Breaker

三狀態模式：**CLOSED**（正常）→ **OPEN**（斷路）→ **HALF_OPEN**（探測）

```typescript
// server/utils/circuit-breaker.ts
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN'

interface CircuitBreakerOptions {
  failureThreshold: number    // 連續失敗幾次後跳閘
  resetTimeoutMs: number      // OPEN 狀態維持多久後進入 HALF_OPEN
  halfOpenMaxAttempts: number  // HALF_OPEN 允許的探測請求數
  onStateChange?: (from: CircuitState, to: CircuitState, name: string) => void
}

export class CircuitBreaker {
  private state: CircuitState = 'CLOSED'
  private failureCount = 0
  private lastFailureTime = 0
  private halfOpenAttempts = 0

  constructor(
    private readonly serviceName: string,
    private readonly options: CircuitBreakerOptions = {
      failureThreshold: 5, resetTimeoutMs: 30_000, halfOpenMaxAttempts: 3,
    },
  ) {}

  async execute<T>(fn: () => Promise<T>, fallback?: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime >= this.options.resetTimeoutMs) {
        this.transitionTo('HALF_OPEN')
      } else {
        if (fallback) return fallback()
        throw createError({ statusCode: 503, message: `${this.serviceName} unavailable (OPEN)` })
      }
    }
    if (this.state === 'HALF_OPEN' && this.halfOpenAttempts >= this.options.halfOpenMaxAttempts) {
      if (fallback) return fallback()
      throw createError({ statusCode: 503, message: `${this.serviceName} unavailable (HALF_OPEN)` })
    }
    try {
      if (this.state === 'HALF_OPEN') this.halfOpenAttempts++
      const result = await fn()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      if (fallback && this.state === 'OPEN') return fallback()
      throw error
    }
  }

  private onSuccess(): void {
    if (this.state === 'HALF_OPEN') this.transitionTo('CLOSED')
    this.failureCount = 0
  }
  private onFailure(): void {
    this.failureCount++
    this.lastFailureTime = Date.now()
    if (this.failureCount >= this.options.failureThreshold) this.transitionTo('OPEN')
  }
  private transitionTo(newState: CircuitState): void {
    const from = this.state
    this.state = newState
    if (newState === 'CLOSED' || newState === 'HALF_OPEN') this.halfOpenAttempts = 0
    if (newState === 'CLOSED') this.failureCount = 0
    this.options.onStateChange?.(from, newState, this.serviceName)
  }
  getState(): CircuitState { return this.state }
}

// server/utils/service-breakers.ts — 全域 breaker 實例管理
const breakers = new Map<string, CircuitBreaker>()
export function getCircuitBreaker(name: string): CircuitBreaker {
  if (!breakers.has(name)) {
    breakers.set(name, new CircuitBreaker(name, {
      failureThreshold: 5, resetTimeoutMs: 30_000, halfOpenMaxAttempts: 3,
      onStateChange: (_from, to, svc) => {
        if (to === 'OPEN') console.error(`[ALERT] Circuit for ${svc} tripped OPEN`)
      },
    }))
  }
  return breakers.get(name)!
}
```

```typescript
// server/api/payments/charge.post.ts — Circuit Breaker + Retry 組合用法
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const breaker = getCircuitBreaker('payment-gateway')
  return breaker.execute(
    () => withRetry(
      () => $fetch('https://payment.example.com/charge', { method: 'POST', body, timeout: 5_000 }),
      { maxRetries: 2, baseDelayMs: 500 },
    ),
    async () => {
      await storeFailedTransaction(body)
      return { status: 'queued', message: '付款服務暫時不可用，交易已排入佇列' }
    },
  )
})
```

### Graceful Degradation

降級策略分三層：Feature Flag 關閉 → Cache Fallback → Static Fallback

```typescript
// server/utils/degradation.ts
interface DegradationStrategy<T> {
  primary: () => Promise<T>
  cacheFallback?: () => Promise<T | null>
  staticFallback?: () => T
  featureFlag?: string
}

export async function withDegradation<T>(strategy: DegradationStrategy<T>): Promise<T> {
  if (strategy.featureFlag) {
    const flags = useRuntimeConfig().public.featureFlags as Record<string, boolean>
    if (flags[strategy.featureFlag] === false && strategy.staticFallback) {
      return strategy.staticFallback()
    }
  }
  try {
    return await strategy.primary()
  } catch (error) {
    console.warn('[degradation] primary failed, trying fallback', error)
    if (strategy.cacheFallback) {
      const cached = await strategy.cacheFallback()
      if (cached !== null) return cached
    }
    if (strategy.staticFallback) return strategy.staticFallback()
    throw error
  }
}
```

```typescript
// server/api/products/featured.get.ts
export default defineEventHandler(async () => {
  const storage = useStorage('cache')
  return withDegradation({
    primary: async () => {
      const products = await $fetch('https://api.example.com/featured')
      await storage.setItem('featured-products', products, { ttl: 300 })
      return products
    },
    cacheFallback: () => storage.getItem('featured-products'),
    staticFallback: () => ({ products: [], message: '推薦商品暫時無法載入', degraded: true }),
    featureFlag: 'featuredProducts',
  })
})
```

### Bulkhead Pattern

請求隔離：限制同時並發數量，防止單一服務耗盡所有資源。

```typescript
// server/utils/bulkhead.ts
export class Bulkhead {
  private activeCount = 0
  private queue: Array<{ resolve: () => void }> = []

  constructor(
    private readonly name: string,
    private readonly maxConcurrent: number = 10,
    private readonly maxQueue: number = 50,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.activeCount >= this.maxConcurrent) {
      if (this.queue.length >= this.maxQueue) {
        throw createError({ statusCode: 429, message: `Bulkhead ${this.name} queue full` })
      }
      await new Promise<void>((resolve) => { this.queue.push({ resolve }) })
    }
    this.activeCount++
    try {
      return await fn()
    } finally {
      this.activeCount--
      if (this.queue.length > 0) this.queue.shift()!.resolve()
    }
  }
}

// 全域實例管理
const bulkheads = new Map<string, Bulkhead>()
export function getBulkhead(name: string, maxConcurrent = 10, maxQueue = 50): Bulkhead {
  if (!bulkheads.has(name)) bulkheads.set(name, new Bulkhead(name, maxConcurrent, maxQueue))
  return bulkheads.get(name)!
}
// getBulkhead('database', 20, 100) — 資料庫（寬鬆）
// getBulkhead('payment', 5, 10)    — 支付閘道（嚴格）
```

```typescript
// server/api/orders/create.post.ts — Bulkhead + Circuit Breaker 組合
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const order = await getBulkhead('database', 20, 100).execute(() => createOrder(body))
  const payment = await getBulkhead('payment', 5, 10).execute(() =>
    getCircuitBreaker('payment-gateway').execute(() => chargePayment(order)),
  )
  return { order, payment }
})
```

---

## Security Engineering

### Threat Model（STRIDE）

| 威脅類別 | 攻擊面 | 防禦措施 |
|---------|--------|---------|
| **Spoofing** 身分偽造 | 偽造 JWT、Session 劫持 | JWT RS256 簽名 + HTTPS + Secure Cookie |
| **Tampering** 竄改 | API 請求竄改、XSS | HMAC 簽名 + CSP + 輸入驗證 |
| **Repudiation** 否認 | 否認操作行為 | 結構化 Audit Log + 不可變日誌 |
| **Info Disclosure** 資訊洩漏 | 錯誤訊息、原始碼暴露 | 最小權限 + 生產錯誤遮蔽 + runtimeConfig 隔離 |
| **DoS** 阻斷服務 | 暴力請求、ReDoS | Rate Limiting + Cloudflare WAF + 輸入長度限制 |
| **Elevation** 權限提升 | 越權存取、IDOR | RBAC + 資源級權限檢查 |

```typescript
// server/middleware/security-headers.ts
export default defineEventHandler((event) => {
  const res = event.node.res
  res.setHeader('X-Content-Type-Options', 'nosniff')
  res.setHeader('X-Frame-Options', 'DENY')
  res.setHeader('X-XSS-Protection', '0')
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin')
  res.setHeader('Permissions-Policy', 'camera=(), microphone=(), geolocation=()')
  res.setHeader('Strict-Transport-Security', 'max-age=63072000; includeSubDomains; preload')
})

// server/utils/audit-logger.ts
interface AuditEntry {
  timestamp: string; userId: string | null; action: string
  resource: string; ip: string; userAgent: string; result: 'success' | 'failure'
  details?: Record<string, unknown>
}
export function auditLog(event: H3Event, entry: Omit<AuditEntry, 'timestamp' | 'ip' | 'userAgent'>): void {
  const log: AuditEntry = {
    ...entry,
    timestamp: new Date().toISOString(),
    ip: getRequestIP(event, { xForwardedFor: true }) ?? 'unknown',
    userAgent: getRequestHeader(event, 'user-agent') ?? 'unknown',
  }
  console.log(JSON.stringify({ type: 'AUDIT', ...log }))
}
```

### JWT Hardening

**演算法選擇：**

| 演算法 | 類型 | 適用場景 | 金鑰要求 |
|-------|------|---------|---------|
| HS256 | 對稱 | 單一服務 | Secret >= 256 bits（32 bytes） |
| RS256 | 非對稱 | 多服務驗證、JWKS | RSA 2048+ bits |
| ES256 | 非對稱 | 高效能、行動端 | ECDSA P-256 |

> **安全警告：** JWT payload 僅是 Base64 編碼，任何人都能解碼。絕不存放密碼、信用卡等敏感資料。

```typescript
// server/utils/jwt.ts
import jwt from 'jsonwebtoken'
import crypto from 'node:crypto'

function validateHmacSecret(secret: string): void {
  if (Buffer.byteLength(secret, 'utf8') < 32)
    throw new Error('JWT secret must be at least 256 bits (32 bytes)')
}

interface TokenPayload { sub: string; role: string; permissions?: string[] }

export function signAccessToken(payload: TokenPayload): string {
  const config = useRuntimeConfig()
  validateHmacSecret(config.jwtSecret)
  return jwt.sign(payload, config.jwtSecret, {
    algorithm: 'HS256', expiresIn: '15m',
    issuer: config.appDomain, audience: config.appDomain,
    jwtid: crypto.randomUUID(),
  })
}

export function signRefreshToken(userId: string): string {
  const config = useRuntimeConfig()
  validateHmacSecret(config.jwtRefreshSecret)
  return jwt.sign(
    { sub: userId, tokenFamily: crypto.randomUUID() },
    config.jwtRefreshSecret,
    { algorithm: 'HS256', expiresIn: '7d', issuer: config.appDomain, jwtid: crypto.randomUUID() },
  )
}

export function verifyAccessToken(token: string): TokenPayload & jwt.JwtPayload {
  const config = useRuntimeConfig()
  // 明確限定 algorithms 防止 algorithm confusion 攻擊
  return jwt.verify(token, config.jwtSecret, {
    algorithms: ['HS256'], issuer: config.appDomain, audience: config.appDomain,
  }) as TokenPayload & jwt.JwtPayload
}
```

> **安全警告：** 驗證 JWT 時必須指定 `algorithms` 白名單。未指定時攻擊者可利用 `alg: none` 偽造 token。

```typescript
// server/api/auth/refresh.post.ts — Refresh Token Rotation
export default defineEventHandler(async (event) => {
  const { refreshToken } = await readBody(event)
  if (!refreshToken) throw createError({ statusCode: 400, message: 'Refresh token required' })

  const config = useRuntimeConfig()
  let decoded: jwt.JwtPayload
  try {
    decoded = jwt.verify(refreshToken, config.jwtRefreshSecret, { algorithms: ['HS256'] }) as jwt.JwtPayload
  } catch { throw createError({ statusCode: 401, message: 'Invalid refresh token' }) }

  // 偵測 token 重用攻擊
  if (await isTokenRevoked(decoded.jti!)) {
    await revokeTokenFamily(decoded.tokenFamily as string)
    auditLog(event, { userId: decoded.sub!, action: 'REFRESH_TOKEN_REUSE', resource: 'auth', result: 'failure' })
    throw createError({ statusCode: 401, message: 'Token reuse detected' })
  }
  await revokeToken(decoded.jti!) // 撤銷當前 token

  const user = await getUserById(decoded.sub!)
  setCookie(event, 'refresh_token', signRefreshToken(user.id), {
    httpOnly: true, secure: true, sameSite: 'strict',
    path: '/api/auth/refresh', maxAge: 7 * 24 * 60 * 60,
  })
  return { accessToken: signAccessToken({ sub: user.id, role: user.role }) }
})

// server/utils/token-revocation.ts — 短效期 access token + 黑名單 refresh token
export async function revokeToken(jti: string): Promise<void> {
  const storage = useStorage('redis')
  await storage.setItem(`revoked:${jti}`, true, { ttl: 7 * 24 * 60 * 60 })
}
export async function isTokenRevoked(jti: string): Promise<boolean> {
  const storage = useStorage('redis')
  return (await storage.getItem(`revoked:${jti}`)) === true
}
export async function revokeTokenFamily(family: string): Promise<void> {
  const storage = useStorage('redis')
  await storage.setItem(`revoked-family:${family}`, true, { ttl: 7 * 24 * 60 * 60 })
}
```

### Password Storage

```typescript
// server/utils/password.ts
import bcrypt from 'bcrypt'

const SALT_ROUNDS = 12 // 2^12 = 4096 次迭代

/** 密碼規則：8~72 字元、大小寫+數字+特殊字元 */
export function validatePasswordStrength(password: string): void {
  const errors: string[] = []
  if (password.length < 8) errors.push('密碼長度至少 8 字元')
  if (password.length > 72) errors.push('密碼不得超過 72 字元（bcrypt 限制）')
  if (!/[A-Z]/.test(password)) errors.push('需包含大寫字母')
  if (!/[a-z]/.test(password)) errors.push('需包含小寫字母')
  if (!/[0-9]/.test(password)) errors.push('需包含數字')
  if (!/[^A-Za-z0-9]/.test(password)) errors.push('需包含特殊字元')
  if (errors.length) throw createError({ statusCode: 400, message: errors.join('；') })
}

export async function hashPassword(plaintext: string): Promise<string> {
  validatePasswordStrength(plaintext)
  return bcrypt.hash(plaintext, SALT_ROUNDS)
}
export async function verifyPassword(plaintext: string, hash: string): Promise<boolean> {
  return bcrypt.compare(plaintext, hash)
}
```

```typescript
// server/utils/password-argon2.ts — Argon2id 替代方案（OWASP 推薦）
import argon2 from 'argon2'

export async function hashPasswordArgon2(plaintext: string): Promise<string> {
  validatePasswordStrength(plaintext)
  return argon2.hash(plaintext, {
    type: argon2.argon2id, memoryCost: 65536, timeCost: 3, parallelism: 4,
  })
}
export async function verifyPasswordArgon2(plaintext: string, hash: string): Promise<boolean> {
  return argon2.verify(hash, plaintext)
}
```

> **安全警告：** bcrypt 截斷超過 72 bytes 的密碼。若需支援更長密碼，先以 SHA-256 雜湊再餵入 bcrypt，或直接使用 Argon2id。

### CSRF Protection

**SameSite Cookie 說明：**

| 屬性值 | 跨站 POST 是否送出 | 適用場景 |
|-------|------------------|---------|
| `Strict` | 否 | 銀行、支付等高安全場景 |
| `Lax` | 否（GET 導航會送出） | 一般應用預設值 |
| `None` | 是（需搭配 `Secure`） | 跨站嵌入、第三方整合 |

```typescript
// server/utils/csrf.ts — Double Submit Cookie Pattern
import crypto from 'node:crypto'

const CSRF_COOKIE = '__csrf'
const CSRF_HEADER = 'x-csrf-token'

export function setCsrfCookie(event: H3Event): string {
  const token = crypto.randomBytes(32).toString('hex')
  setCookie(event, CSRF_COOKIE, token, {
    httpOnly: false, secure: true, sameSite: 'strict', path: '/', maxAge: 3600,
  })
  return token
}

export function validateCsrfToken(event: H3Event): void {
  if (['GET', 'HEAD', 'OPTIONS'].includes(getMethod(event))) return
  const cookieToken = getCookie(event, CSRF_COOKIE)
  const headerToken = getRequestHeader(event, CSRF_HEADER)
  if (!cookieToken || !headerToken)
    throw createError({ statusCode: 403, message: 'CSRF token missing' })
  const a = Buffer.from(cookieToken), b = Buffer.from(headerToken)
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b))
    throw createError({ statusCode: 403, message: 'CSRF token mismatch' })
}

// server/middleware/csrf.ts
export default defineEventHandler((event) => {
  const url = getRequestURL(event).pathname
  if (!url.startsWith('/api/')) return
  const exempt = ['/api/auth/login', '/api/auth/register', '/api/webhooks/']
  if (exempt.some((p) => url.startsWith(p))) return
  validateCsrfToken(event)
})
```

```typescript
// composables/useCsrfFetch.client.ts — 前端自動附加 CSRF token（.client 限定僅客戶端載入）
export function useCsrfFetch() {
  const getCsrf = () => document.cookie.match(/(?:^|;\s*)__csrf=([^;]+)/)?.[1]
  return $fetch.create({
    onRequest({ options }) {
      const token = getCsrf()
      if (token) {
        options.headers = {
          ...Object.fromEntries(new Headers(options.headers as HeadersInit).entries()),
          'x-csrf-token': token,
        }
      }
    },
  })
}
```

**nuxt-security 模組整合：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    headers: {
      crossOriginResourcePolicy: 'same-origin',
      crossOriginOpenerPolicy: 'same-origin',
      contentSecurityPolicy: {
        'base-uri': ["'none'"],
        'font-src': ["'self'", 'https:', 'data:'],
        'form-action': ["'self'"],
        'frame-ancestors': ["'none'"],
        'img-src': ["'self'", 'data:', 'https:'],
        'object-src': ["'none'"],
        'script-src-attr': ["'none'"],
        'style-src': ["'self'", "'unsafe-inline'"],
        'upgrade-insecure-requests': true,
      },
    },
    rateLimiter: { tokensPerInterval: 60, interval: 60_000 },
    requestSizeLimiter: { maxRequestSizeInBytes: 1_048_576, maxUploadFileRequestInBytes: 10_485_760 },
  },
})
```

### CSP Nonce

```typescript
// server/plugins/csp-nonce.ts — Nonce-based CSP（手動實作）
import crypto from 'node:crypto'

export default defineNitroPlugin((nitroApp) => {
  nitroApp.hooks.hook('render:html', (html, { event }) => {
    const nonce = crypto.randomBytes(16).toString('base64')
    event.context.security = { nonce }
    // 注入 nonce 到所有 <script> 標籤
    html.head = html.head.map((h) => h.replace(/<script/g, `<script nonce="${nonce}"`))
    html.bodyAppend = html.bodyAppend.map((b) => b.replace(/<script/g, `<script nonce="${nonce}"`))
  })

  nitroApp.hooks.hook('render:response', (response, { event }) => {
    const nonce = event.context.security?.nonce
    if (!nonce) return
    response.headers['content-security-policy'] = [
      `default-src 'self'`, `script-src 'self' 'nonce-${nonce}'`,
      `style-src 'self' 'unsafe-inline'`, `img-src 'self' data: https:`,
      `font-src 'self' https: data:`, `connect-src 'self' https:`,
      `frame-ancestors 'none'`, `base-uri 'none'`, `form-action 'self'`,
      `object-src 'none'`, `upgrade-insecure-requests`,
    ].join('; ')
  })
})
```

**nuxt-security nonce 配置（推薦）：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    nonce: true,
    headers: {
      contentSecurityPolicy: {
        'script-src': ["'self'", "'nonce-{{nonce}}'"],
        'style-src': ["'self'", "'unsafe-inline'"],
      },
    },
  },
})
```

```vue
<!-- 元件中使用 nonce -->
<script setup lang="ts">
const nonce = useNonce()  // nuxt-security 提供
</script>
<template>
  <component :is="'script'" :nonce="nonce" type="application/ld+json">
    {{ JSON.stringify(structuredData) }}
  </component>
</template>
```

### Secrets Management

**環境變數分層架構：**

```
Level 3: Secret Manager（Vault / AWS Secrets Manager）→ 加密儲存、存取稽核、自動輪替
Level 2: CI/CD Secrets（GitHub Actions / GitLab CI）  → 加密儲存、部署時注入
Level 1: .env 檔案（僅限本地開發）                      → 明文、不可提交至版控
```

```typescript
// nuxt.config.ts — runtimeConfig 安全模式
export default defineNuxtConfig({
  runtimeConfig: {
    // 僅伺服器端可存取
    jwtSecret: '', jwtRefreshSecret: '', databaseUrl: '',
    redisUrl: '', stripeSecretKey: '', smtpPassword: '',
    // 客戶端可存取（僅非敏感配置）
    public: { appDomain: '', apiBaseUrl: '', sentryDsn: '' },
  },
})
```

> **安全警告：** `runtimeConfig.public` 會序列化到客戶端 HTML，絕不可放入任何 secret。

```typescript
// server/utils/secrets.ts — Secret rotation 策略
interface SecretConfig { current: string; previous?: string }

export function getSecret(name: string): SecretConfig {
  const config = useRuntimeConfig()
  const current = (config as any)[name]
  if (!current) throw new Error(`Required secret '${name}' is not configured`)
  return { current, previous: (config as any)[`${name}Previous`] || undefined }
}

/** 驗證時先用 current，失敗再嘗試 previous，確保 rotation 期間不中斷 */
export function verifyWithRotation(
  token: string, secretName: string,
  verifyFn: (token: string, secret: string) => unknown,
): unknown {
  const { current, previous } = getSecret(secretName)
  try { return verifyFn(token, current) } catch (err) {
    if (previous) {
      try {
        const result = verifyFn(token, previous)
        console.warn(`[secrets] Verified with previous secret for '${secretName}'`)
        return result
      } catch { /* 兩個都失敗 */ }
    }
    throw err
  }
}
```

```bash
# .env.example（提交至版控作為範本，.env 加入 .gitignore）
NUXT_JWT_SECRET=              # openssl rand -hex 32
NUXT_JWT_SECRET_PREVIOUS=     # rotation 過渡期用
NUXT_JWT_REFRESH_SECRET=      # openssl rand -hex 32
NUXT_DATABASE_URL=            # postgresql://user:pass@host:5432/db
NUXT_REDIS_URL=               # redis://:password@host:6379
NUXT_STRIPE_SECRET_KEY=       # sk_live_...
NUXT_PUBLIC_APP_DOMAIN=       # https://example.com
NUXT_PUBLIC_API_BASE_URL=     # https://api.example.com
```

> **安全警告：** CI/CD 中不要在 build log 印出環境變數。產生密鑰務必使用密碼學安全的隨機產生器：`openssl rand -hex 32`。

---

## OWASP Top 10（2021）系統性對照表

以下對照表涵蓋 OWASP 2021 年版全部十項風險類別，並交叉引用本指南現有章節與新增防護措施。

| # | OWASP 類別 | Nuxt 3 攻擊面 | 防護策略 | 本指南參考 |
|---|-----------|--------------|---------|-----------|
| **A01** | Broken Access Control（存取控制失效） | IDOR（不安全的直接物件引用）、越權 API 呼叫、路由守衛繞過 | RBAC 中介層 + 資源級權限檢查 + Nuxt `definePageMeta({ middleware })` 路由守衛 | [Threat Model（STRIDE）](#threat-modelstride)、[CSRF Protection](#csrf-protection) |
| **A02** | Cryptographic Failures（密碼學失敗） | JWT 使用弱密鑰、明文儲存密碼、缺少 HTTPS | JWT RS256/ES256 非對稱簽名 + Argon2id/bcrypt 密碼雜湊 + HSTS 強制 HTTPS | [JWT Hardening](#jwt-hardening)、[Password Storage](#password-storage)、[JWT RS256/ES256 完整實作](#jwt-rs256es256-完整實作) |
| **A03** | Injection（注入攻擊） | SQL Injection（ORM 繞過）、NoSQL Injection、XSS（`v-html`）、OS Command Injection | 參數化查詢（Drizzle/Prisma ORM）+ Zod 輸入驗證 + DOMPurify 消毒 + CSP Nonce | [CSP Nonce](#csp-nonce)、[XSS 深度防護](#xss-深度防護) |
| **A04** | Insecure Design（不安全設計） | 缺乏威脅建模、未考慮業務邏輯攻擊、缺少安全需求 | STRIDE 威脅建模 + 完整威脅建模流程（見下方） + 安全 User Story + 濫用案例分析 | [Threat Model（STRIDE）](#threat-modelstride)、[威脅建模流程](#a04-insecure-design-威脅建模流程) |
| **A05** | Security Misconfiguration（安全設定錯誤） | 預設密鑰未更換、開發模式錯誤頁面洩漏堆疊追蹤、不必要的 HTTP 方法啟用 | `nuxt-security` 模組 + 安全 HTTP 標頭 + 生產環境錯誤遮蔽 + `.env.example` 範本 | [Security Headers](#threat-modelstride)、[Secrets Management](#secrets-management) |
| **A06** | Vulnerable & Outdated Components（易受攻擊和過時元件） | npm 供應鏈攻擊、已知漏洞的相依套件 | `npm audit` + Dependabot/Renovate 自動更新 + `package-lock.json` 鎖定版本 | [A08 供應鏈安全](#a08-software-and-data-integrity-供應鏈安全) |
| **A07** | Identification & Authentication Failures（身分驗證失敗） | 弱密碼策略、暴力破解、Session 固定攻擊、缺少 MFA | 密碼強度驗證 + Refresh Token Rotation + 登入速率限制 + 帳號鎖定機制 | [Password Storage](#password-storage)、[認證端點強化](#認證端點強化) |
| **A08** | Software and Data Integrity Failures（軟體與資料完整性失敗） | CDN 腳本竄改、CI/CD 管線入侵、惡意套件注入 | SRI（Subresource Integrity）+ npm provenance 驗證 + lockfile 完整性檢查 | [A08 供應鏈安全](#a08-software-and-data-integrity-供應鏈安全) |
| **A09** | Security Logging & Monitoring Failures（安全日誌與監控失敗） | 缺少審計日誌、未偵測暴力攻擊、入侵事件未告警 | 結構化 Audit Log + 登入失敗監控 + 即時告警（Sentry/PagerDuty） | [Threat Model（STRIDE）](#threat-modelstride) 中的 `auditLog` |
| **A10** | Server-Side Request Forgery（SSRF） | SSR 中 `$fetch` 可存取內網、DNS Rebinding、Cloud Metadata 洩漏 | URL 白名單驗證 + 私有 IP 封鎖 + DNS Rebinding 防護 | [SSRF 防護](#ssrf-防護ssr-應用關鍵) |

### A04: Insecure Design — 威脅建模流程

STRIDE 對照表（已見上方）僅為威脅建模的一環。完整的威脅建模應遵循以下結構化流程：

**步驟一：繪製系統資料流圖（DFD）**

```
使用者瀏覽器 ──HTTPS──► Nuxt SSR（Nitro）──$fetch──► 外部 API
      │                      │                         │
      │                      ├──ORM──► PostgreSQL       │
      │                      ├──Redis──► Session Store  │
      │                      └──S3──► 檔案儲存          │
      │                                                 │
      └──── CDN（靜態資源）                              │
                                                        │
                              Cloud Metadata (169.254.169.254)
```

**步驟二：辨識信任邊界**

```typescript
// server/utils/threat-model.ts — 信任邊界定義與驗證
interface TrustBoundary {
  name: string
  from: 'client' | 'server' | 'internal-service' | 'external-service' | 'database'
  to: 'client' | 'server' | 'internal-service' | 'external-service' | 'database'
  threats: string[]
  controls: string[]
}

export const trustBoundaries: TrustBoundary[] = [
  {
    name: '瀏覽器 → Nuxt SSR',
    from: 'client',
    to: 'server',
    threats: ['XSS', 'CSRF', 'Injection', '參數竄改', 'Session 劫持'],
    controls: ['CSP Nonce', 'CSRF Token', 'Zod 驗證', 'Secure Cookie', 'Rate Limiting'],
  },
  {
    name: 'Nuxt SSR → 外部 API',
    from: 'server',
    to: 'external-service',
    threats: ['SSRF', 'DNS Rebinding', '機密洩漏（API Key 外流）', '中間人攻擊'],
    controls: ['URL 白名單', '私有 IP 封鎖', 'mTLS', 'Secret 環境變數隔離'],
  },
  {
    name: 'Nuxt SSR → 資料庫',
    from: 'server',
    to: 'database',
    threats: ['SQL Injection', '連線字串洩漏', '越權查詢'],
    controls: ['ORM 參數化查詢', '最小權限資料庫帳號', '連線加密 (TLS)'],
  },
]
```

**步驟三：針對每個信任邊界進行 STRIDE 分析**

```typescript
// server/utils/threat-model.ts（續）
type StrideCategory = 'Spoofing' | 'Tampering' | 'Repudiation' | 'InfoDisclosure' | 'DoS' | 'ElevationOfPrivilege'

interface ThreatScenario {
  boundary: string
  stride: StrideCategory
  scenario: string
  likelihood: 'LOW' | 'MEDIUM' | 'HIGH'
  impact: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL'
  riskScore: number // likelihood × impact
  mitigation: string
  status: 'mitigated' | 'accepted' | 'pending'
}

/**
 * 風險評分矩陣：
 * likelihood: LOW=1, MEDIUM=2, HIGH=3
 * impact: LOW=1, MEDIUM=2, HIGH=3, CRITICAL=4
 * riskScore = likelihood × impact
 * 優先處理 riskScore >= 6 的威脅
 */
export const threatScenarios: ThreatScenario[] = [
  {
    boundary: '瀏覽器 → Nuxt SSR',
    stride: 'Spoofing',
    scenario: '攻擊者利用竊取的 JWT 偽造身分',
    likelihood: 'MEDIUM',
    impact: 'HIGH',
    riskScore: 6,
    mitigation: 'JWT RS256 非對稱簽名 + 短效期(15m) + Refresh Token Rotation',
    status: 'mitigated',
  },
  {
    boundary: 'Nuxt SSR → 外部 API',
    stride: 'Tampering',
    scenario: '攻擊者透過 SSRF 竄改內部服務請求',
    likelihood: 'HIGH',
    impact: 'CRITICAL',
    riskScore: 12,
    mitigation: 'URL 白名單 + 私有 IP 封鎖 + DNS Rebinding 防護',
    status: 'mitigated',
  },
  {
    boundary: 'Nuxt SSR → 資料庫',
    stride: 'ElevationOfPrivilege',
    scenario: '攻擊者透過 IDOR 存取其他使用者的資料',
    likelihood: 'HIGH',
    impact: 'HIGH',
    riskScore: 9,
    mitigation: '資源級權限檢查：WHERE user_id = currentUser.id',
    status: 'mitigated',
  },
]
```

**步驟四：安全 User Story 與濫用案例（Abuse Case）**

```markdown
## 安全 User Story 範本

**正常 User Story:**
身為使用者，我可以更新我的個人資料。

**安全 User Story:**
身為使用者，我只能更新「自己的」個人資料，且所有欄位皆經過驗證與消毒。

**濫用案例（Abuse Case）:**
- 攻擊者嘗試修改 URL 中的 userId 參數來存取其他使用者的資料（IDOR）
- 攻擊者在個人簡介欄位注入 `<script>` 標籤（Stored XSS）
- 攻擊者上傳偽裝為圖片的惡意檔案（檔案上傳攻擊）
```

### A08: Software and Data Integrity — 供應鏈安全

**SRI（Subresource Integrity）for 外部腳本：**

```typescript
// nuxt.config.ts — 為外部 CDN 腳本添加 SRI
export default defineNuxtConfig({
  app: {
    head: {
      script: [
        {
          src: 'https://cdn.example.com/analytics.js',
          integrity: 'sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxAhLI2MBQ2oA0VL/RPtyYPhBN+NOn',
          crossorigin: 'anonymous',
        },
      ],
    },
  },
})
```

```typescript
// server/utils/sri.ts — 動態產生 SRI 雜湊
import crypto from 'node:crypto'

/**
 * 為外部資源內容產生 SRI 雜湊值
 * @param content - 腳本或樣式表的內容
 * @param algorithm - 雜湊演算法（預設 sha384）
 */
export function generateSriHash(
  content: string | Buffer,
  algorithm: 'sha256' | 'sha384' | 'sha512' = 'sha384',
): string {
  const hash = crypto.createHash(algorithm).update(content).digest('base64')
  return `${algorithm}-${hash}`
}

/**
 * 驗證外部資源的 SRI 雜湊
 */
export async function verifySriIntegrity(
  url: string,
  expectedHash: string,
): Promise<boolean> {
  const response = await $fetch<ArrayBuffer>(url, { responseType: 'arrayBuffer' })
  const content = Buffer.from(response)
  const [algorithm] = expectedHash.split('-') as ['sha256' | 'sha384' | 'sha512']
  const actualHash = generateSriHash(content, algorithm)
  return actualHash === expectedHash
}
```

**供應鏈安全 — lockfile 完整性與 npm provenance：**

```bash
# CI/CD 管線中的供應鏈安全檢查

# 1. 確保使用 lockfile 安裝（避免意外升級）
npm ci --ignore-scripts  # 先安裝依賴但不執行 scripts

# 2. 審查套件安全性
npm audit --audit-level=high

# 3. 驗證 npm provenance（npm >= 9.5.0）
npm audit signatures

# 4. 檢查 lockfile 完整性（防止 lockfile injection）
# 安裝 lockfile-lint
npx lockfile-lint \
  --path package-lock.json \
  --type npm \
  --allowed-hosts npm \
  --validate-https \
  --validate-integrity
```

```typescript
// scripts/verify-lockfile.ts — 預提交鉤子腳本
import { readFileSync } from 'node:fs'
import crypto from 'node:crypto'

/**
 * 驗證 package-lock.json 是否被竄改
 * 在 CI/CD 中使用，或作為 husky pre-commit hook
 */
function verifyLockfileIntegrity(): void {
  const lockfile = readFileSync('package-lock.json', 'utf-8')
  const parsed = JSON.parse(lockfile)

  // 檢查所有相依是否來自可信任 registry
  const allowedRegistries = ['https://registry.npmjs.org/']

  function checkPackage(name: string, pkg: Record<string, unknown>): void {
    const resolved = pkg.resolved as string | undefined
    if (resolved && !allowedRegistries.some((r) => resolved.startsWith(r))) {
      throw new Error(
        `[供應鏈安全] 套件 ${name} 使用非可信任 registry: ${resolved}`,
      )
    }
    // 檢查 integrity 欄位存在
    if (resolved && !pkg.integrity) {
      console.warn(`[供應鏈安全] 套件 ${name} 缺少 integrity 雜湊`)
    }
  }

  const packages = parsed.packages as Record<string, Record<string, unknown>>
  for (const [name, pkg] of Object.entries(packages)) {
    if (name === '') continue // root package
    checkPackage(name, pkg)
  }

  console.log('[供應鏈安全] lockfile 驗證通過')
}

verifyLockfileIntegrity()
```

---

## SSRF 防護（SSR 應用關鍵）

### 為什麼 SSR 應用容易受到 SSRF 攻擊

Nuxt 3 的 SSR 模式在伺服器端使用 `$fetch` 或 `ofetch` 發送 HTTP 請求。這意味著：

1. **伺服器端請求可存取內部網路** — 攻擊者透過控制 URL 參數，可讓伺服器向內網服務發送請求
2. **Cloud Metadata 暴露** — AWS/GCP/Azure 的 metadata endpoint（`169.254.169.254`）可被存取，洩漏 IAM 認證資訊
3. **DNS Rebinding** — 攻擊者先讓 DNS 解析到合法 IP 通過驗證，然後切換到內網 IP
4. **協議走私** — 利用 `file://`、`gopher://` 等協議存取本地檔案系統或內部服務

**典型攻擊場景：**

```
攻擊者 → POST /api/preview?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
                                    ↓
Nuxt SSR Server → $fetch(userProvidedUrl) → 取得 AWS IAM 認證 → 回傳給攻擊者
```

### URL 白名單驗證工具

```typescript
// server/utils/ssrf-protection.ts
import { URL } from 'node:url'
import { isIP } from 'node:net'
import dns from 'node:dns/promises'

/**
 * 私有 IP 範圍（RFC 1918 / RFC 4193 / RFC 6598）
 * 包含 IPv4 與 IPv6 的保留位址
 */
const PRIVATE_IP_RANGES = [
  // IPv4 私有範圍
  { start: '10.0.0.0', end: '10.255.255.255' },         // Class A
  { start: '172.16.0.0', end: '172.31.255.255' },        // Class B
  { start: '192.168.0.0', end: '192.168.255.255' },      // Class C
  // 環回位址
  { start: '127.0.0.0', end: '127.255.255.255' },        // Loopback
  // 連結本地位址
  { start: '169.254.0.0', end: '169.254.255.255' },      // Link-local（含 Cloud Metadata）
  // 共享位址空間
  { start: '100.64.0.0', end: '100.127.255.255' },       // CGN (RFC 6598)
  // 其他保留
  { start: '0.0.0.0', end: '0.255.255.255' },            // "This" network
  { start: '192.0.0.0', end: '192.0.0.255' },            // IETF Protocol Assignments
  { start: '198.18.0.0', end: '198.19.255.255' },        // Benchmarking
  { start: '224.0.0.0', end: '239.255.255.255' },        // Multicast
  { start: '240.0.0.0', end: '255.255.255.255' },        // Reserved
]

/**
 * 將 IPv4 字串轉為數字以便範圍比對
 */
function ipToNumber(ip: string): number {
  const parts = ip.split('.').map(Number)
  return ((parts[0] << 24) | (parts[1] << 16) | (parts[2] << 8) | parts[3]) >>> 0
}

/**
 * 檢查 IP 是否為私有 / 保留位址
 */
export function isPrivateIP(ip: string): boolean {
  // IPv6 環回與連結本地
  if (ip === '::1' || ip.startsWith('fe80:') || ip.startsWith('fc00:') || ip.startsWith('fd00:')) {
    return true
  }
  // IPv4-mapped IPv6 (::ffff:x.x.x.x)
  const ipv4Match = ip.match(/^::ffff:(\d+\.\d+\.\d+\.\d+)$/)
  const ipv4 = ipv4Match ? ipv4Match[1] : ip

  if (!isIP(ipv4)) return true // 非合法 IP 一律視為私有

  const num = ipToNumber(ipv4)
  return PRIVATE_IP_RANGES.some(
    (range) => num >= ipToNumber(range.start) && num <= ipToNumber(range.end),
  )
}

/**
 * SSRF 安全配置
 */
interface SsrfValidationOptions {
  /** 允許的外部網域白名單 */
  allowedDomains?: string[]
  /** 允許的協議（預設僅 https） */
  allowedProtocols?: string[]
  /** 允許的連接埠（預設 443, 80） */
  allowedPorts?: number[]
  /** 是否執行 DNS 解析驗證（防止 DNS Rebinding） */
  dnsCheck?: boolean
  /** DNS 解析逾時（毫秒） */
  dnsTimeoutMs?: number
}

const defaultOptions: SsrfValidationOptions = {
  allowedProtocols: ['https:'],
  allowedPorts: [443, 80],
  dnsCheck: true,
  dnsTimeoutMs: 3000,
}

/**
 * 驗證外部 URL 是否安全（非 SSRF 攻擊）
 * 用於任何會根據使用者輸入發出伺服器端請求的場景
 *
 * @throws {H3Error} 當 URL 不安全時拋出 400 錯誤
 */
export async function validateExternalUrl(
  url: string,
  opts: SsrfValidationOptions = {},
): Promise<URL> {
  const options = { ...defaultOptions, ...opts }

  // 步驟 1：解析 URL
  let parsed: URL
  try {
    parsed = new URL(url)
  } catch {
    throw createError({
      statusCode: 400,
      message: 'SSRF Protection: 無效的 URL 格式',
    })
  }

  // 步驟 2：協議白名單
  if (!options.allowedProtocols!.includes(parsed.protocol)) {
    throw createError({
      statusCode: 400,
      message: `SSRF Protection: 不允許的協議 ${parsed.protocol}（僅允許 ${options.allowedProtocols!.join(', ')}）`,
    })
  }

  // 步驟 3：連接埠白名單
  const port = parsed.port ? Number(parsed.port) : (parsed.protocol === 'https:' ? 443 : 80)
  if (!options.allowedPorts!.includes(port)) {
    throw createError({
      statusCode: 400,
      message: `SSRF Protection: 不允許的連接埠 ${port}`,
    })
  }

  // 步驟 4：網域白名單（若有配置）
  if (options.allowedDomains && options.allowedDomains.length > 0) {
    const isAllowed = options.allowedDomains.some(
      (domain) => parsed.hostname === domain || parsed.hostname.endsWith(`.${domain}`),
    )
    if (!isAllowed) {
      throw createError({
        statusCode: 400,
        message: `SSRF Protection: 網域 ${parsed.hostname} 不在白名單中`,
      })
    }
  }

  // 步驟 5：直接 IP 檢查
  if (isIP(parsed.hostname)) {
    if (isPrivateIP(parsed.hostname)) {
      throw createError({
        statusCode: 400,
        message: 'SSRF Protection: 不允許存取私有 IP 位址',
      })
    }
  }

  // 步驟 6：DNS 解析驗證（防止 DNS Rebinding）
  if (options.dnsCheck && !isIP(parsed.hostname)) {
    try {
      const addresses = await Promise.race([
        dns.resolve4(parsed.hostname),
        new Promise<never>((_, reject) =>
          setTimeout(() => reject(new Error('DNS resolve timeout')), options.dnsTimeoutMs)
        ),
      ])

      for (const addr of addresses) {
        if (isPrivateIP(addr)) {
          throw createError({
            statusCode: 400,
            message: `SSRF Protection: 網域 ${parsed.hostname} 解析到私有 IP ${addr}`,
          })
        }
      }
    } catch (err) {
      // 如果是我們自己拋出的 H3Error，直接重新拋出
      if (err && typeof err === 'object' && 'statusCode' in err) throw err

      throw createError({
        statusCode: 400,
        message: `SSRF Protection: 無法解析網域 ${parsed.hostname}`,
      })
    }
  }

  return parsed
}
```

### DNS Rebinding 防護

DNS Rebinding 攻擊流程：攻擊者將惡意網域的 TTL 設為極低值，第一次 DNS 查詢回傳合法外部 IP（通過驗證），第二次查詢時改為回傳內網 IP（`127.0.0.1` 或 `169.254.169.254`）。

```typescript
// server/utils/dns-pinning.ts — DNS Pinning 防護 DNS Rebinding
import dns from 'node:dns/promises'

/**
 * DNS 解析結果快取，在驗證後鎖定（pin）解析的 IP，
 * 確保後續請求使用同一 IP，防止 DNS Rebinding
 */
const dnsCache = new Map<string, { ip: string; expiresAt: number }>()

const DNS_CACHE_TTL_MS = 5 * 60 * 1000 // 5 分鐘快取

/**
 * 解析並鎖定 DNS 結果
 * 回傳已驗證為安全的 IP 位址
 */
export async function resolvePinnedDns(hostname: string): Promise<string> {
  const cached = dnsCache.get(hostname)
  if (cached && cached.expiresAt > Date.now()) {
    return cached.ip
  }

  const addresses = await dns.resolve4(hostname)
  if (addresses.length === 0) {
    throw createError({
      statusCode: 400,
      message: `DNS 解析失敗：${hostname} 無 A 記錄`,
    })
  }

  const ip = addresses[0]

  // 驗證解析出的 IP 不是私有位址
  if (isPrivateIP(ip)) {
    throw createError({
      statusCode: 400,
      message: `DNS Rebinding 防護：${hostname} 解析到私有 IP ${ip}`,
    })
  }

  // 鎖定解析結果
  dnsCache.set(hostname, { ip, expiresAt: Date.now() + DNS_CACHE_TTL_MS })
  return ip
}

/**
 * 安全的外部 HTTP 請求 — 整合 SSRF 防護 + DNS Pinning
 */
export async function safeFetch<T>(
  url: string,
  options: Parameters<typeof $fetch>[1] = {},
  ssrfOptions?: SsrfValidationOptions,
): Promise<T> {
  // 步驟 1：驗證 URL 安全性
  const validated = await validateExternalUrl(url, ssrfOptions)

  // 步驟 2：DNS Pinning — 解析並鎖定 IP
  const pinnedIp = await resolvePinnedDns(validated.hostname)

  // 步驟 3：使用鎖定的 IP 發送請求，設定 Host header 為原始網域
  const pinnedUrl = new URL(validated.href)
  pinnedUrl.hostname = pinnedIp

  return $fetch<T>(pinnedUrl.href, {
    ...options,
    headers: {
      ...Object.fromEntries(new Headers(options.headers as HeadersInit).entries()),
      Host: validated.hostname,
    },
    timeout: 10_000,
  })
}
```

### 在 Server Route 中使用 SSRF 防護

```typescript
// server/api/preview/fetch.post.ts — 安全的 URL 預覽端點
export default defineEventHandler(async (event) => {
  const { url } = await readValidatedBody(event, (body) => {
    const schema = z.object({
      url: z.string().url('請提供合法的 URL'),
    })
    return schema.parse(body)
  })

  // 使用 safeFetch 而非直接 $fetch
  const content = await safeFetch<string>(url, {
    responseType: 'text',
    timeout: 5_000,
    headers: {
      // 不要轉發使用者的 Cookie 或 Authorization
      Accept: 'text/html',
      'User-Agent': 'NuxtPreviewBot/1.0',
    },
  }, {
    allowedProtocols: ['https:'],
    allowedDomains: ['example.com', 'trusted-cdn.com'], // 依業務需求設定
    dnsCheck: true,
  })

  // 截斷過長的回應內容
  const truncated = content.slice(0, 50_000)

  return {
    url,
    contentLength: content.length,
    preview: truncated,
  }
})
```

```typescript
// server/api/webhooks/proxy.post.ts — Webhook 代理的 SSRF 防護
export default defineEventHandler(async (event) => {
  const { callbackUrl, payload } = await readBody(event)

  // 嚴格驗證 callback URL
  await validateExternalUrl(callbackUrl, {
    allowedProtocols: ['https:'],
    allowedDomains: ['partner-a.com', 'partner-b.com'], // 僅允許已註冊的合作夥伴
    dnsCheck: true,
  })

  // 使用 safeFetch 防止 DNS Rebinding（TOCTOU）攻擊
  await safeFetch(callbackUrl, {
    method: 'POST',
    body: payload,
    timeout: 10_000,
  })

  return { status: 'delivered' }
})
```

> **安全警告：** 絕對不要在 SSR server route 中直接將使用者提供的 URL 傳給 `$fetch`。所有使用者可控制的 URL 必須經過 `validateExternalUrl()` 驗證。即使有白名單，仍需啟用 DNS Rebinding 防護。

---

## JWT RS256/ES256 完整實作

### 金鑰對產生腳本

```bash
#!/bin/bash
# scripts/generate-jwt-keys.sh — 產生 JWT 非對稱金鑰對

set -euo pipefail

KEYS_DIR="./keys"
mkdir -p "$KEYS_DIR"

echo "=== 產生 RS256 金鑰對 ==="
# RSA 2048-bit 私鑰
openssl genrsa -out "$KEYS_DIR/rs256-private.pem" 2048
# 從私鑰匯出公鑰
openssl rsa -in "$KEYS_DIR/rs256-private.pem" -pubout -out "$KEYS_DIR/rs256-public.pem"

echo "=== 產生 ES256 金鑰對 ==="
# ECDSA P-256 私鑰
openssl ecparam -genkey -name prime256v1 -noout -out "$KEYS_DIR/es256-private.pem"
# 從私鑰匯出公鑰
openssl ec -in "$KEYS_DIR/es256-private.pem" -pubout -out "$KEYS_DIR/es256-public.pem"

echo "=== 金鑰指紋（用於 JWKS kid） ==="
openssl rsa -in "$KEYS_DIR/rs256-private.pem" -pubout -outform DER 2>/dev/null | openssl dgst -sha256 | awk '{print "RS256 kid:", $2}'
openssl ec -in "$KEYS_DIR/es256-private.pem" -pubout -outform DER 2>/dev/null | openssl dgst -sha256 | awk '{print "ES256 kid:", $2}'

echo ""
echo "=== 完成！請將金鑰內容設為環境變數 ==="
echo "NUXT_JWT_RS256_PRIVATE_KEY=\$(cat $KEYS_DIR/rs256-private.pem | base64 -w 0)"
echo "NUXT_JWT_RS256_PUBLIC_KEY=\$(cat $KEYS_DIR/rs256-public.pem | base64 -w 0)"
echo ""
echo "重要：金鑰檔案不可提交至版控，請確認 .gitignore 包含 /keys/"
```

### RS256 簽名與驗證

```typescript
// server/utils/jwt-asymmetric.ts — RS256/ES256 非對稱 JWT 實作
import jwt from 'jsonwebtoken'
import crypto from 'node:crypto'

type JwtAlgorithm = 'RS256' | 'ES256'

interface AsymmetricKeyPair {
  privateKey: string
  publicKey: string
  kid: string // Key ID，用於 JWKS
  algorithm: JwtAlgorithm
}

/**
 * 從 Base64 環境變數載入金鑰
 * 金鑰以 Base64 儲存於環境變數中，避免 PEM 格式的換行問題
 */
function loadKeyFromEnv(envVar: string): string {
  const base64Key = process.env[envVar]
  if (!base64Key) {
    throw new Error(`環境變數 ${envVar} 未設定`)
  }
  return Buffer.from(base64Key, 'base64').toString('utf-8')
}

/**
 * 產生金鑰指紋作為 kid（Key ID）
 */
function generateKid(publicKey: string): string {
  return crypto
    .createHash('sha256')
    .update(publicKey)
    .digest('hex')
    .slice(0, 16)
}

/**
 * 取得 RS256 金鑰對
 */
export function getRS256Keys(): AsymmetricKeyPair {
  const privateKey = loadKeyFromEnv('NUXT_JWT_RS256_PRIVATE_KEY')
  const publicKey = loadKeyFromEnv('NUXT_JWT_RS256_PUBLIC_KEY')
  return {
    privateKey,
    publicKey,
    kid: generateKid(publicKey),
    algorithm: 'RS256',
  }
}

/**
 * 取得 ES256 金鑰對
 */
export function getES256Keys(): AsymmetricKeyPair {
  const privateKey = loadKeyFromEnv('NUXT_JWT_ES256_PRIVATE_KEY')
  const publicKey = loadKeyFromEnv('NUXT_JWT_ES256_PUBLIC_KEY')
  return {
    privateKey,
    publicKey,
    kid: generateKid(publicKey),
    algorithm: 'ES256',
  }
}

interface AsymmetricTokenPayload {
  sub: string
  role: string
  permissions?: string[]
}

/**
 * 使用 RS256 簽發 Access Token
 */
export function signAccessTokenRS256(payload: AsymmetricTokenPayload): string {
  const keys = getRS256Keys()
  return jwt.sign(payload, keys.privateKey, {
    algorithm: 'RS256',
    expiresIn: '15m',
    issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
    audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
    jwtid: crypto.randomUUID(),
    keyid: keys.kid,
  })
}

/**
 * 使用 ES256 簽發 Access Token（效能更佳，簽章更短）
 */
export function signAccessTokenES256(payload: AsymmetricTokenPayload): string {
  const keys = getES256Keys()
  return jwt.sign(payload, keys.privateKey, {
    algorithm: 'ES256',
    expiresIn: '15m',
    issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
    audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
    jwtid: crypto.randomUUID(),
    keyid: keys.kid,
  })
}

/**
 * 驗證 RS256 Token — 僅需公鑰
 * 這使得微服務可以獨立驗證 token 而不需要私鑰
 */
export function verifyAccessTokenRS256(
  token: string,
): AsymmetricTokenPayload & jwt.JwtPayload {
  const keys = getRS256Keys()
  return jwt.verify(token, keys.publicKey, {
    algorithms: ['RS256'], // 嚴格限定演算法，防止 algorithm confusion
    issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
    audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
  }) as AsymmetricTokenPayload & jwt.JwtPayload
}

/**
 * 驗證 ES256 Token — 僅需公鑰
 */
export function verifyAccessTokenES256(
  token: string,
): AsymmetricTokenPayload & jwt.JwtPayload {
  const keys = getES256Keys()
  return jwt.verify(token, keys.publicKey, {
    algorithms: ['ES256'],
    issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
    audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
  }) as AsymmetricTokenPayload & jwt.JwtPayload
}
```

### JWKS 端點實作

JWKS（JSON Web Key Set）端點讓其他服務或微服務可以自動取得公鑰來驗證 JWT，而無需預先配置。

```typescript
// server/utils/jwks.ts — JWKS 格式轉換
import crypto from 'node:crypto'

interface JwkRsa {
  kty: 'RSA'
  use: 'sig'
  alg: 'RS256'
  kid: string
  n: string  // modulus
  e: string  // exponent
}

interface JwkEc {
  kty: 'EC'
  use: 'sig'
  alg: 'ES256'
  crv: 'P-256'
  kid: string
  x: string
  y: string
}

/**
 * 將 RSA PEM 公鑰轉換為 JWK 格式
 */
export function rsaPemToJwk(pemPublicKey: string, kid: string): JwkRsa {
  const keyObject = crypto.createPublicKey(pemPublicKey)
  const exported = keyObject.export({ format: 'jwk' }) as { n: string; e: string }
  return {
    kty: 'RSA',
    use: 'sig',
    alg: 'RS256',
    kid,
    n: exported.n,
    e: exported.e,
  }
}

/**
 * 將 EC PEM 公鑰轉換為 JWK 格式
 */
export function ecPemToJwk(pemPublicKey: string, kid: string): JwkEc {
  const keyObject = crypto.createPublicKey(pemPublicKey)
  const exported = keyObject.export({ format: 'jwk' }) as { x: string; y: string }
  return {
    kty: 'EC',
    use: 'sig',
    alg: 'ES256',
    crv: 'P-256',
    kid,
    x: exported.x,
    y: exported.y,
  }
}
```

```typescript
// server/api/.well-known/jwks.json.get.ts — JWKS Discovery 端點
export default defineEventHandler((event) => {
  // 設定快取標頭（公鑰可安全快取）
  setResponseHeaders(event, {
    'Content-Type': 'application/json',
    'Cache-Control': 'public, max-age=3600', // 1 小時快取
    'Access-Control-Allow-Origin': '*',       // JWKS 通常需要跨域存取
  })

  const keys: (JwkRsa | JwkEc)[] = []

  // 載入 RS256 公鑰
  try {
    const rs256 = getRS256Keys()
    keys.push(rsaPemToJwk(rs256.publicKey, rs256.kid))
  } catch {
    // RS256 金鑰未配置，跳過
  }

  // 載入 ES256 公鑰
  try {
    const es256 = getES256Keys()
    keys.push(ecPemToJwk(es256.publicKey, es256.kid))
  } catch {
    // ES256 金鑰未配置，跳過
  }

  return { keys }
})
```

### 金鑰輪替策略

```typescript
// server/utils/jwt-key-rotation.ts — 金鑰輪替支援
import jwt from 'jsonwebtoken'

interface KeySet {
  current: AsymmetricKeyPair
  previous?: AsymmetricKeyPair
}

/**
 * 取得 RS256 金鑰集（含前一代金鑰，用於輪替過渡期）
 *
 * 環境變數命名慣例：
 * - NUXT_JWT_RS256_PRIVATE_KEY     → 當前私鑰
 * - NUXT_JWT_RS256_PUBLIC_KEY      → 當前公鑰
 * - NUXT_JWT_RS256_PRIVATE_KEY_PREV → 前一代私鑰
 * - NUXT_JWT_RS256_PUBLIC_KEY_PREV  → 前一代公鑰
 */
export function getRS256KeySet(): KeySet {
  const current = getRS256Keys()
  let previous: AsymmetricKeyPair | undefined

  try {
    const prevPrivate = Buffer.from(
      process.env.NUXT_JWT_RS256_PRIVATE_KEY_PREV || '', 'base64',
    ).toString('utf-8')
    const prevPublic = Buffer.from(
      process.env.NUXT_JWT_RS256_PUBLIC_KEY_PREV || '', 'base64',
    ).toString('utf-8')

    if (prevPrivate && prevPublic) {
      const kid = crypto.createHash('sha256').update(prevPublic).digest('hex').slice(0, 16)
      previous = { privateKey: prevPrivate, publicKey: prevPublic, kid, algorithm: 'RS256' }
    }
  } catch {
    // 前一代金鑰未配置，無需處理
  }

  return { current, previous }
}

/**
 * 驗證 token 時支援金鑰輪替
 * 先以當前金鑰驗證，失敗後嘗試前一代金鑰
 */
export function verifyTokenWithRotation(
  token: string,
): AsymmetricTokenPayload & jwt.JwtPayload {
  const keySet = getRS256KeySet()

  // 步驟 1：嘗試使用當前金鑰驗證
  try {
    return jwt.verify(token, keySet.current.publicKey, {
      algorithms: ['RS256'],
      issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
      audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
    }) as AsymmetricTokenPayload & jwt.JwtPayload
  } catch (currentKeyError) {
    // 步驟 2：若有前一代金鑰，嘗試驗證
    if (keySet.previous) {
      try {
        const result = jwt.verify(token, keySet.previous.publicKey, {
          algorithms: ['RS256'],
          issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
          audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
        }) as AsymmetricTokenPayload & jwt.JwtPayload

        console.warn('[JWT 金鑰輪替] Token 以前一代金鑰驗證成功，客戶端應盡快更新 token')
        return result
      } catch {
        // 兩代金鑰都無法驗證
      }
    }
    throw currentKeyError
  }
}

/**
 * JWKS 端點同時公開當前與前一代公鑰
 * 確保輪替期間所有已發行的 token 都能被驗證
 */
export function getAllPublicJwks(): (JwkRsa | JwkEc)[] {
  const keys: (JwkRsa | JwkEc)[] = []
  const keySet = getRS256KeySet()

  keys.push(rsaPemToJwk(keySet.current.publicKey, keySet.current.kid))
  if (keySet.previous) {
    keys.push(rsaPemToJwk(keySet.previous.publicKey, keySet.previous.kid))
  }

  return keys
}
```

```markdown
## 金鑰輪替 SOP（標準作業程序）

1. **產生新金鑰對** — 執行 `scripts/generate-jwt-keys.sh`
2. **將當前金鑰移至 PREV 位置**
   - `NUXT_JWT_RS256_PRIVATE_KEY` → `NUXT_JWT_RS256_PRIVATE_KEY_PREV`
   - `NUXT_JWT_RS256_PUBLIC_KEY` → `NUXT_JWT_RS256_PUBLIC_KEY_PREV`
3. **設定新金鑰為當前金鑰**
   - 新私鑰 → `NUXT_JWT_RS256_PRIVATE_KEY`
   - 新公鑰 → `NUXT_JWT_RS256_PUBLIC_KEY`
4. **部署更新** — JWKS 端點自動公開兩代公鑰
5. **等待過渡期** — 至少等待 Access Token 最大有效期（15 分鐘）
6. **清除前一代金鑰** — 確認所有舊 token 已過期後，移除 PREV 環境變數
7. **建議頻率** — 每 90 天輪替一次，或在金鑰洩漏時立即輪替
```

### JWT `kid`（Key ID）Header Matching 驗證

當使用 JWKS 端點進行 JWT 驗證時，必須透過 JWT header 中的 `kid` 欄位找到對應的公鑰。這在多金鑰環境和金鑰輪替期間尤為關鍵——若未正確比對 `kid`，攻擊者可能利用錯誤的金鑰繞過驗證。

#### 使用 `jose` 庫搭配遠端 JWKS 自動比對 `kid`

`jose` 的 `createRemoteJWKSet` 會自動依照 JWT header 中的 `kid` 從 JWKS 端點挑選正確的公鑰。以下展示完整的 H3 middleware 實作：

```typescript
// server/middleware/auth-jwks.ts — 使用 jose + JWKS 端點的 JWT 驗證 middleware
import {
  createRemoteJWKSet,
  jwtVerify,
  decodeProtectedHeader,
  errors as joseErrors,
} from 'jose'

// 建立遠端 JWKS 客戶端（內建快取與自動輪詢）
const JWKS = createRemoteJWKSet(
  new URL(`${process.env.NUXT_PUBLIC_APP_DOMAIN}/.well-known/jwks.json`),
  {
    cooldownDuration: 30_000,  // 抓取失敗後的冷卻時間（毫秒）
    cacheMaxAge: 600_000,      // JWKS 快取存活時間（10 分鐘）
  },
)

export default defineEventHandler(async (event) => {
  // 僅攔截需要認證的路徑
  const publicPaths = ['/api/health', '/api/.well-known']
  if (publicPaths.some((p) => event.path?.startsWith(p))) return

  const authHeader = getHeader(event, 'authorization')
  if (!authHeader?.startsWith('Bearer ')) {
    throw createError({ statusCode: 401, message: 'Missing or invalid Authorization header' })
  }

  const token = authHeader.slice(7)

  // 步驟 1：解碼 JWT header，提取 kid 和 alg
  let header: { kid?: string; alg?: string }
  try {
    header = decodeProtectedHeader(token)
  } catch {
    throw createError({ statusCode: 401, message: 'Malformed JWT header' })
  }

  // 步驟 2：驗證 kid 存在（缺少 kid 時無法從 JWKS 匹配金鑰）
  if (!header.kid) {
    throw createError({
      statusCode: 401,
      message: 'JWT header missing kid claim — cannot match signing key',
    })
  }

  // 步驟 3：驗證演算法白名單（防止 alg confusion 攻擊）
  const allowedAlgorithms = ['RS256', 'ES256'] as const
  if (!header.alg || !allowedAlgorithms.includes(header.alg as any)) {
    throw createError({
      statusCode: 401,
      message: `Unsupported JWT algorithm: ${header.alg}`,
    })
  }

  // 步驟 4：使用 jose 驗證 token（自動依照 kid 從 JWKS 選取公鑰）
  try {
    const { payload } = await jwtVerify(token, JWKS, {
      algorithms: [...allowedAlgorithms],
      issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
      audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
      clockTolerance: 30, // 允許 30 秒的時鐘偏差
    })

    // 將驗證後的 payload 附加到 event context
    event.context.auth = {
      sub: payload.sub,
      roles: payload.roles as string[] | undefined,
      kid: header.kid, // 記錄使用的 kid，便於日誌追蹤
    }
  } catch (error) {
    // 步驟 5：細緻的錯誤處理
    if (error instanceof joseErrors.JWKSNoMatchingKey) {
      // kid 不匹配 JWKS 中的任何金鑰（可能金鑰已輪替或 token 偽造）
      console.warn(`[JWT] No matching key for kid="${header.kid}" — possible key rotation or forgery`)
      throw createError({ statusCode: 401, message: 'JWT signing key not found in JWKS' })
    }
    if (error instanceof joseErrors.JWTExpired) {
      throw createError({ statusCode: 401, message: 'JWT has expired' })
    }
    if (error instanceof joseErrors.JWTClaimValidationFailed) {
      throw createError({ statusCode: 401, message: 'JWT claim validation failed' })
    }
    throw createError({ statusCode: 401, message: 'JWT verification failed' })
  }
})
```

#### 手動 `kid` 比對（本地金鑰集場景）

若不使用遠端 JWKS 而是持有本地金鑰集（例如多租戶場景），需手動依照 `kid` 查找金鑰：

```typescript
// server/utils/jwt-kid-resolver.ts — 手動 kid 比對（本地金鑰集）
import { decodeProtectedHeader, jwtVerify, importSPKI } from 'jose'

interface ManagedKey {
  kid: string
  publicKeyPem: string
  algorithm: 'RS256' | 'ES256'
  status: 'active' | 'rotated' | 'revoked'
  expiresAt?: Date // 輪替過渡期結束時間
}

/**
 * 從本地金鑰集中依照 kid 查找金鑰
 * 金鑰輪替安全：接受 active 和 rotated 狀態的金鑰
 */
function resolveKeyByKid(kid: string, keys: ManagedKey[]): ManagedKey | null {
  const key = keys.find((k) => k.kid === kid)
  if (!key) return null

  // 拒絕已撤銷的金鑰
  if (key.status === 'revoked') {
    console.warn(`[JWT] Attempt to use revoked key kid="${kid}"`)
    return null
  }

  // 檢查已輪替金鑰是否仍在過渡期內
  if (key.status === 'rotated') {
    if (key.expiresAt && new Date() > key.expiresAt) {
      console.warn(`[JWT] Rotated key kid="${kid}" has passed its grace period`)
      return null
    }
    console.info(`[JWT] Token verified with rotated key kid="${kid}" — client should refresh`)
  }

  return key
}

/**
 * 使用 kid 比對進行 JWT 驗證
 */
export async function verifyTokenWithKidMatching(
  token: string,
  managedKeys: ManagedKey[],
): Promise<{ payload: Record<string, unknown>; kid: string }> {
  // 解碼 header 提取 kid
  const header = decodeProtectedHeader(token)
  if (!header.kid) {
    throw createError({ statusCode: 401, message: 'JWT missing kid header' })
  }

  // 依照 kid 查找金鑰
  const matchedKey = resolveKeyByKid(header.kid, managedKeys)
  if (!matchedKey) {
    throw createError({
      statusCode: 401,
      message: `No valid key found for kid="${header.kid}"`,
    })
  }

  // 匯入公鑰並驗證
  const publicKey = await importSPKI(matchedKey.publicKeyPem, matchedKey.algorithm)
  const { payload } = await jwtVerify(token, publicKey, {
    algorithms: [matchedKey.algorithm],
    issuer: process.env.NUXT_PUBLIC_APP_DOMAIN,
    audience: process.env.NUXT_PUBLIC_APP_DOMAIN,
  })

  return { payload: payload as Record<string, unknown>, kid: header.kid }
}
```

#### `kid` 比對安全要點

| 要點 | 說明 |
|------|------|
| **必須驗證 `kid` 存在** | 缺少 `kid` 的 JWT 無法安全匹配金鑰，應直接拒絕 |
| **演算法白名單** | 即使 `kid` 正確，也必須檢查 `alg` 是否在白名單中，防止 alg confusion 攻擊 |
| **金鑰輪替過渡期** | 輪替期間同時接受新舊金鑰的 `kid`，過渡期結束後移除舊金鑰 |
| **`JWKSNoMatchingKey` 處理** | `jose` 在 JWKS 中找不到匹配 `kid` 的金鑰時拋出此錯誤，應記錄警告並返回 401 |
| **JWKS 快取策略** | 設定合理的 `cacheMaxAge`，避免每次驗證都遠端拉取；但也不能過長以免延遲感知金鑰輪替 |

---

## OAuth 2.0 PKCE（Proof Key for Code Exchange）流程

PKCE 是 OAuth 2.0 授權碼流程的安全擴充（[RFC 7636](https://tools.ietf.org/html/rfc7636)），用於防止授權碼攔截攻擊。對於 SPA 和行動應用等無法安全儲存 client secret 的公開客戶端（Public Client），PKCE 是**強制要求**。即使是機密客戶端（Confidential Client），OAuth 2.1 草案也建議啟用 PKCE 作為額外防護層。

### PKCE 流程概覽

```
┌──────────┐                              ┌───────────────┐                        ┌──────────────┐
│  Client   │                              │ Authorization │                        │  Token       │
│  (Nuxt)   │                              │   Server      │                        │  Endpoint    │
└─────┬─────┘                              └───────┬───────┘                        └──────┬───────┘
      │  1. 產生 code_verifier (隨機 43-128 字元)   │                                       │
      │  2. 計算 code_challenge = BASE64URL(SHA256(code_verifier))                          │
      │  3. 產生 state (CSRF 防護)                  │                                       │
      │                                             │                                       │
      │  4. GET /authorize?                         │                                       │
      │     response_type=code&                     │                                       │
      │     code_challenge=...&                     │                                       │
      │     code_challenge_method=S256&             │                                       │
      │     state=...                               │                                       │
      ├────────────────────────────────────────────>│                                       │
      │                                             │                                       │
      │  5. 使用者認證完成，重導向回 Client          │                                       │
      │     /callback?code=AUTH_CODE&state=...      │                                       │
      │<────────────────────────────────────────────┤                                       │
      │                                             │                                       │
      │  6. POST /token                             │                                       │
      │     grant_type=authorization_code&          │                                       │
      │     code=AUTH_CODE&                         │                                       │
      │     code_verifier=...  ← 原始 verifier      │                                       │
      ├─────────────────────────────────────────────┼──────────────────────────────────────>│
      │                                             │                                       │
      │  7. Server 計算 SHA256(code_verifier)        │                                       │
      │     與步驟 4 的 code_challenge 比對           │                                       │
      │                                             │                                       │
      │  8. 比對成功，回傳 access_token + refresh_token                                     │
      │<────────────────────────────────────────────┼──────────────────────────────────────<│
      │                                             │                                       │
```

### PKCE Composable 實作（客戶端）

```typescript
// composables/useOAuthPKCE.ts — OAuth 2.0 PKCE 流程 composable
export interface OAuthPKCEConfig {
  authorizationEndpoint: string
  tokenEndpoint: string
  clientId: string
  redirectUri: string
  scope: string
  /** 額外的 authorization request 參數 */
  extraParams?: Record<string, string>
}

/**
 * Base64 URL 編碼（RFC 4648 §5）
 * 與標準 Base64 的差異：+ → -，/ → _，移除尾部 =
 */
function base64UrlEncode(buffer: Uint8Array): string {
  const base64 = btoa(String.fromCharCode(...buffer))
  return base64.replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
}

export function useOAuthPKCE() {
  /**
   * 產生 code_verifier（RFC 7636 §4.1）
   * 密碼學安全隨機字串，長度 43-128 字元
   * 使用 32 bytes 隨機數 → Base64URL 編碼後約 43 字元
   */
  function generateCodeVerifier(): string {
    const array = new Uint8Array(32)
    crypto.getRandomValues(array)
    return base64UrlEncode(array)
  }

  /**
   * 計算 code_challenge（RFC 7636 §4.2）
   * S256 方法：BASE64URL(SHA-256(code_verifier))
   * 注意：禁止使用 plain 方法，僅接受 S256
   */
  async function generateCodeChallenge(verifier: string): Promise<string> {
    const encoder = new TextEncoder()
    const data = encoder.encode(verifier)
    const digest = await crypto.subtle.digest('SHA-256', data)
    return base64UrlEncode(new Uint8Array(digest))
  }

  /**
   * 產生 state 參數（CSRF 防護）
   * 16 bytes 隨機數 → Base64URL 編碼
   */
  function generateState(): string {
    const array = new Uint8Array(16)
    crypto.getRandomValues(array)
    return base64UrlEncode(array)
  }

  /**
   * 啟動 OAuth 授權流程
   * 產生 PKCE 參數並重導向至 Authorization Server
   */
  async function initiateAuth(config: OAuthPKCEConfig): Promise<void> {
    const codeVerifier = generateCodeVerifier()
    const codeChallenge = await generateCodeChallenge(codeVerifier)
    const state = generateState()

    // 將 code_verifier 和 state 儲存至 sessionStorage
    // sessionStorage 僅限同分頁存取，降低 XSS 風險
    sessionStorage.setItem('oauth_code_verifier', codeVerifier)
    sessionStorage.setItem('oauth_state', state)
    // 記錄發起時間，用於防止 replay 攻擊
    sessionStorage.setItem('oauth_initiated_at', Date.now().toString())

    const params = new URLSearchParams({
      response_type: 'code',
      client_id: config.clientId,
      redirect_uri: config.redirectUri,
      scope: config.scope,
      state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
      ...config.extraParams,
    })

    await navigateTo(`${config.authorizationEndpoint}?${params}`, { external: true })
  }

  /**
   * 處理 OAuth callback（在 redirect_uri 頁面呼叫）
   * 驗證 state、交換授權碼取得 token
   */
  async function handleCallback(config: OAuthPKCEConfig): Promise<{
    accessToken: string
    refreshToken?: string
    expiresIn: number
  }> {
    const route = useRoute()
    const code = route.query.code as string
    const returnedState = route.query.state as string
    const error = route.query.error as string

    // 檢查授權伺服器回傳的錯誤
    if (error) {
      const description = route.query.error_description as string
      throw new Error(`OAuth error: ${error} — ${description}`)
    }

    if (!code || !returnedState) {
      throw new Error('Missing code or state in callback')
    }

    // 驗證 state 參數（防 CSRF）
    const storedState = sessionStorage.getItem('oauth_state')
    if (!storedState || storedState !== returnedState) {
      // 清除所有 OAuth 暫存資料
      cleanupOAuthSession()
      throw new Error('State mismatch — possible CSRF attack')
    }

    // 檢查是否超時（防止過期的 authorization code 被重用）
    const initiatedAt = Number(sessionStorage.getItem('oauth_initiated_at') || '0')
    if (Date.now() - initiatedAt > 10 * 60 * 1000) {
      cleanupOAuthSession()
      throw new Error('OAuth flow timed out (>10 minutes)')
    }

    // 取得 code_verifier
    const codeVerifier = sessionStorage.getItem('oauth_code_verifier')
    if (!codeVerifier) {
      throw new Error('Missing code_verifier — OAuth session may have been cleared')
    }

    try {
      // 透過 server API route 交換 token（避免在客戶端暴露 client_secret）
      const tokenResponse = await $fetch('/api/auth/oauth/token', {
        method: 'POST',
        body: {
          code,
          code_verifier: codeVerifier,
          redirect_uri: config.redirectUri,
        },
      })

      return tokenResponse as {
        accessToken: string
        refreshToken?: string
        expiresIn: number
      }
    } finally {
      // 無論成功失敗都清除 OAuth 暫存資料
      cleanupOAuthSession()
    }
  }

  function cleanupOAuthSession(): void {
    sessionStorage.removeItem('oauth_code_verifier')
    sessionStorage.removeItem('oauth_state')
    sessionStorage.removeItem('oauth_initiated_at')
  }

  return {
    generateCodeVerifier,
    generateCodeChallenge,
    generateState,
    initiateAuth,
    handleCallback,
    cleanupOAuthSession,
  }
}
```

### Server-Side Token Exchange（伺服器端授權碼交換）

授權碼交換**必須**在伺服器端進行，以避免在客戶端暴露 `client_secret`：

```typescript
// server/api/auth/oauth/token.post.ts — 伺服器端授權碼交換
interface TokenExchangeBody {
  code: string
  code_verifier: string
  redirect_uri: string
}

interface TokenResponse {
  access_token: string
  token_type: string
  expires_in: number
  refresh_token?: string
  id_token?: string
}

export default defineEventHandler(async (event) => {
  const body = await readBody<TokenExchangeBody>(event)
  const config = useRuntimeConfig()

  // 驗證必要欄位
  if (!body.code || !body.code_verifier || !body.redirect_uri) {
    throw createError({
      statusCode: 400,
      message: 'Missing required fields: code, code_verifier, redirect_uri',
    })
  }

  // 驗證 code_verifier 格式（RFC 7636 §4.1：43-128 字元，[A-Z] / [a-z] / [0-9] / "-" / "." / "_" / "~"）
  if (!/^[A-Za-z0-9\-._~]{43,128}$/.test(body.code_verifier)) {
    throw createError({
      statusCode: 400,
      message: 'Invalid code_verifier format',
    })
  }

  try {
    // 向 Authorization Server 的 token endpoint 交換 token
    const tokenResponse = await $fetch<TokenResponse>(config.oauth.tokenEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: new URLSearchParams({
        grant_type: 'authorization_code',
        code: body.code,
        code_verifier: body.code_verifier,
        redirect_uri: body.redirect_uri,
        client_id: config.oauth.clientId,
        // client_secret 僅在伺服器端使用，不暴露給客戶端
        client_secret: config.oauth.clientSecret,
      }).toString(),
      timeout: 10_000,
    })

    // 驗證回傳的 token（若有 id_token，驗證其 claims）
    if (tokenResponse.id_token) {
      await validateIdToken(tokenResponse.id_token, config.oauth.clientId)
    }

    // 設定 secure httpOnly cookie 存放 refresh_token
    if (tokenResponse.refresh_token) {
      setCookie(event, 'refresh_token', tokenResponse.refresh_token, {
        httpOnly: true,
        secure: true,
        sameSite: 'lax',
        path: '/api/auth',
        maxAge: 30 * 24 * 60 * 60, // 30 天
      })
    }

    // 僅回傳 access_token 和 expires_in 給客戶端
    return {
      accessToken: tokenResponse.access_token,
      expiresIn: tokenResponse.expires_in,
      refreshToken: tokenResponse.refresh_token ? '[stored in httpOnly cookie]' : undefined,
    }
  } catch (error: any) {
    console.error('[OAuth Token Exchange] Failed:', error.data || error.message)
    throw createError({
      statusCode: 502,
      message: 'Failed to exchange authorization code for tokens',
    })
  }
})

/**
 * 驗證 ID Token（OpenID Connect）
 */
async function validateIdToken(idToken: string, expectedClientId: string): Promise<void> {
  const { createRemoteJWKSet, jwtVerify } = await import('jose')
  const config = useRuntimeConfig()

  const JWKS = createRemoteJWKSet(new URL(config.oauth.jwksUri))
  const { payload } = await jwtVerify(idToken, JWKS, {
    audience: expectedClientId,
    issuer: config.oauth.issuer,
  })

  // 驗證 nonce（若有使用）
  // 驗證 auth_time（若需要最近認證）
  if (!payload.sub) {
    throw new Error('ID Token missing sub claim')
  }
}
```

### OAuth Callback 頁面

```vue
<!-- pages/auth/callback.vue — OAuth redirect_uri 目標頁面 -->
<script setup lang="ts">
const { handleCallback } = useOAuthPKCE()

const error = ref<string | null>(null)
const loading = ref(true)

const config: OAuthPKCEConfig = {
  authorizationEndpoint: useRuntimeConfig().public.oauth.authorizationEndpoint,
  tokenEndpoint: useRuntimeConfig().public.oauth.tokenEndpoint,
  clientId: useRuntimeConfig().public.oauth.clientId,
  redirectUri: `${useRequestURL().origin}/auth/callback`,
  scope: 'openid profile email',
}

onMounted(async () => {
  try {
    const tokens = await handleCallback(config)
    // 儲存 access token（建議使用 useState 或 pinia store）
    const authStore = useAuthStore()
    authStore.setAccessToken(tokens.accessToken, tokens.expiresIn)
    // 導向原本要訪問的頁面
    const returnTo = sessionStorage.getItem('oauth_return_to') || '/'
    sessionStorage.removeItem('oauth_return_to')
    await navigateTo(returnTo)
  } catch (e: any) {
    error.value = e.message
    loading.value = false
  }
})
</script>

<template>
  <div class="flex items-center justify-center min-h-screen">
    <div v-if="loading && !error" class="text-center">
      <p>正在完成登入...</p>
    </div>
    <div v-else-if="error" class="text-center text-red-600">
      <h2>登入失敗</h2>
      <p>{{ error }}</p>
      <NuxtLink to="/login" class="underline">返回登入頁面</NuxtLink>
    </div>
  </div>
</template>
```

### Runtime Config 設定

```typescript
// nuxt.config.ts — OAuth PKCE 相關設定
export default defineNuxtConfig({
  runtimeConfig: {
    // 僅伺服器端（不暴露給客戶端）
    oauth: {
      clientSecret: process.env.NUXT_OAUTH_CLIENT_SECRET || '',
      tokenEndpoint: process.env.NUXT_OAUTH_TOKEN_ENDPOINT || '',
      jwksUri: process.env.NUXT_OAUTH_JWKS_URI || '',
      issuer: process.env.NUXT_OAUTH_ISSUER || '',
    },
    public: {
      // 客戶端可存取
      oauth: {
        clientId: process.env.NUXT_PUBLIC_OAUTH_CLIENT_ID || '',
        authorizationEndpoint: process.env.NUXT_PUBLIC_OAUTH_AUTHORIZATION_ENDPOINT || '',
        tokenEndpoint: process.env.NUXT_PUBLIC_OAUTH_TOKEN_ENDPOINT || '',
      },
    },
  },
})
```

### PKCE 安全要點

| 要點 | 說明 |
|------|------|
| **僅使用 S256** | `code_challenge_method` 必須為 `S256`，禁止 `plain` 方法（RFC 7636 §4.2） |
| **`code_verifier` 長度** | 最少 43 字元、最多 128 字元（32 bytes 隨機數 Base64URL 編碼後為 43 字元） |
| **`state` 參數** | 必須包含足夠熵值（至少 128 bits），用於防止 CSRF 攻擊 |
| **Token 交換在 Server 端** | `client_secret` 絕不可暴露至客戶端，授權碼交換必須透過 server API route |
| **`code_verifier` 生命週期** | 使用後立即清除 `sessionStorage`，避免重放攻擊 |
| **流程超時** | 從發起到 callback 應設定合理超時（建議 10 分鐘），防止過期 authorization code 被利用 |
| **Refresh Token 儲存** | 使用 `httpOnly` + `secure` + `sameSite` cookie，不可存放於 `localStorage` |

---

## XSS 深度防護

### v-html 的風險與 DOMPurify 消毒

Vue 的 `v-html` 指令會直接將 HTML 字串插入 DOM，這是 XSS 攻擊的高風險入口。

**危險範例（切勿在生產環境中使用）：**

```vue
<!-- ❌ 危險：直接渲染使用者輸入 -->
<template>
  <div v-html="userComment" />
</template>

<script setup lang="ts">
// 若 userComment 包含 <script>alert('XSS')</script>，將直接執行
const userComment = ref('')
</script>
```

**安全做法 — 使用 DOMPurify 消毒：**

```typescript
// composables/useSanitize.ts — XSS 消毒 composable
import DOMPurify from 'dompurify'

/**
 * DOMPurify 配置：僅允許安全的 HTML 標籤與屬性
 */
const PURIFY_CONFIG: DOMPurify.Config = {
  ALLOWED_TAGS: [
    'p', 'br', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li',
    'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'blockquote', 'code', 'pre',
    'table', 'thead', 'tbody', 'tr', 'th', 'td', 'img', 'span', 'div',
  ],
  ALLOWED_ATTR: [
    'href', 'target', 'rel', 'src', 'alt', 'title', 'class', 'id',
    'width', 'height',
  ],
  ALLOW_DATA_ATTR: false,
  // 強制 <a> 標籤添加 rel="noopener noreferrer"
  ADD_ATTR: ['target'],
  // 禁止 javascript: URI
  ALLOWED_URI_REGEXP: /^(?:(?:https?|mailto|tel):|[^a-z]|[a-z+.-]+(?:[^a-z+.\-:]|$))/i,
}

/**
 * 消毒 HTML 內容，移除所有可能的 XSS 載體
 */
export function useSanitize() {
  const sanitize = (dirtyHtml: string): string => {
    if (!dirtyHtml) return ''
    return DOMPurify.sanitize(dirtyHtml, PURIFY_CONFIG)
  }

  /**
   * 嚴格模式：移除所有 HTML 標籤，僅保留純文字
   */
  const sanitizeToText = (dirtyHtml: string): string => {
    if (!dirtyHtml) return ''
    return DOMPurify.sanitize(dirtyHtml, { ALLOWED_TAGS: [] })
  }

  return { sanitize, sanitizeToText }
}
```

```vue
<!-- components/SafeHtmlRenderer.vue — 安全的 HTML 渲染元件 -->
<template>
  <div v-html="cleanHtml" />
</template>

<script setup lang="ts">
const props = defineProps<{
  content: string
}>()

const { sanitize } = useSanitize()
const cleanHtml = computed(() => sanitize(props.content))
</script>
```

```typescript
// server/utils/sanitize-server.ts — 伺服器端 HTML 消毒（SSR 情境）
import { JSDOM } from 'jsdom'
import DOMPurify from 'dompurify'

// 在伺服器端，DOMPurify 需要 jsdom 提供 DOM 環境
const window = new JSDOM('').window
const purify = DOMPurify(window as unknown as Window)

/**
 * 伺服器端 HTML 消毒
 * 用於 API 端點儲存使用者輸入前的消毒處理
 */
export function sanitizeHtmlServer(dirty: string): string {
  return purify.sanitize(dirty, {
    ALLOWED_TAGS: [
      'p', 'br', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li',
      'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'blockquote', 'code', 'pre',
    ],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
    ALLOW_DATA_ATTR: false,
  })
}

// 使用範例：在 API 端點中消毒後儲存
// server/api/comments/create.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const sanitizedContent = sanitizeHtmlServer(body.content)
  // 將消毒後的內容儲存到資料庫
  return await db.insert(comments).values({
    content: sanitizedContent,
    userId: event.context.user.id,
  })
})
```

### 輸出編碼指導

```typescript
// server/utils/output-encoding.ts — 不同上下文的輸出編碼
/**
 * HTML 實體編碼 — 用於將使用者輸入安全地插入 HTML 內容
 */
export function encodeHtmlEntities(str: string): string {
  const map: Record<string, string> = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;',
    '/': '&#x2F;',
    '`': '&#96;',
  }
  return str.replace(/[&<>"'/`]/g, (char) => map[char])
}

/**
 * JavaScript 字串編碼 — 用於將使用者輸入安全地嵌入 JavaScript 上下文
 */
export function encodeForJavaScript(str: string): string {
  return str.replace(/[\\'"<>&\u2028\u2029]/g, (char) => {
    return `\\u${char.charCodeAt(0).toString(16).padStart(4, '0')}`
  })
}

/**
 * URL 參數編碼 — 用於動態產生 URL
 */
export function encodeForUrl(str: string): string {
  return encodeURIComponent(str)
}

/**
 * CSS 值編碼 — 用於動態 CSS 值（少見但必要時使用）
 */
export function encodeForCss(str: string): string {
  return str.replace(/[^a-zA-Z0-9 ]/g, (char) => {
    return `\\${char.charCodeAt(0).toString(16)} `
  })
}
```

### Trusted Types API

Trusted Types 是瀏覽器原生的 XSS 防護 API，可在 DOM 層級強制執行消毒策略。

```typescript
// plugins/trusted-types.client.ts — Trusted Types 配置（僅客戶端）
export default defineNuxtPlugin(() => {
  // 檢查瀏覽器支援
  if (typeof window === 'undefined' || !('trustedTypes' in window)) {
    return
  }

  // 建立預設 Trusted Types 策略
  const policy = (window as any).trustedTypes.createPolicy('default', {
    createHTML: (input: string): string => {
      // 使用 DOMPurify 消毒
      return DOMPurify.sanitize(input, {
        RETURN_TRUSTED_TYPE: false,
      })
    },
    createScriptURL: (input: string): string => {
      // 僅允許同源與白名單網域的腳本 URL
      const url = new URL(input, window.location.origin)
      const allowedHosts = [
        window.location.hostname,
        'cdn.example.com',
      ]
      if (!allowedHosts.includes(url.hostname)) {
        throw new TypeError(`Trusted Types: 不允許的腳本來源 ${url.hostname}`)
      }
      return input
    },
    createScript: (_input: string): string => {
      // 禁止動態腳本建立
      throw new TypeError('Trusted Types: 禁止動態建立腳本')
    },
  })

  return {
    provide: {
      trustedTypesPolicy: policy,
    },
  }
})
```

```typescript
// nuxt.config.ts — 搭配 CSP 啟用 Trusted Types
export default defineNuxtConfig({
  modules: ['nuxt-security'],
  security: {
    headers: {
      contentSecurityPolicy: {
        // 啟用 Trusted Types（漸進式導入，先用 report-only）
        'require-trusted-types-for': ["'script'"],
        'trusted-types': ['default', 'dompurify'],
      },
    },
  },
})
```

> **注意：** Trusted Types 目前（2025）主要在 Chromium 系瀏覽器中支援。建議以 CSP report-only 模式先行部署，確認不影響現有功能後再強制執行。

---

## 安全測試方法論

### SAST（靜態應用安全測試）工具建議

```bash
# 1. ESLint 安全外掛 — 開發階段即時掃描
npm install -D eslint-plugin-security eslint-plugin-no-unsanitized
```

```typescript
// eslint.config.ts — 安全相關 ESLint 配置
import security from 'eslint-plugin-security'
import noUnsanitized from 'eslint-plugin-no-unsanitized'

export default [
  {
    plugins: {
      security,
      'no-unsanitized': noUnsanitized,
    },
    rules: {
      // 偵測不安全的正規表達式（ReDoS）
      'security/detect-unsafe-regex': 'error',
      // 偵測 eval() 使用
      'security/detect-eval-with-expression': 'error',
      // 偵測非字面量的 require()
      'security/detect-non-literal-require': 'warn',
      // 偵測非字面量的 fs 操作
      'security/detect-non-literal-fs-filename': 'warn',
      // 偵測不安全的隨機數產生
      'security/detect-pseudoRandomBytes': 'error',
      // 偵測可能的 SQL injection
      'security/detect-possible-timing-attacks': 'warn',
      // 偵測 child_process 使用
      'security/detect-child-process': 'error',
      // 偵測 innerHTML 使用
      'no-unsanitized/property': 'error',
      // 偵測 document.write 使用
      'no-unsanitized/method': 'error',
    },
  },
]
```

```bash
# 2. npm audit — 相依套件漏洞掃描
npm audit --audit-level=high

# 3. Semgrep — 進階靜態分析（支援自訂規則）
npx @semgrep/semgrep --config p/javascript --config p/typescript ./server

# 4. Snyk — 商業級漏洞掃描（CI/CD 整合）
npx snyk test
npx snyk code test
```

### DAST（動態應用安全測試）工具建議

```bash
# OWASP ZAP — 開源 Web 應用安全掃描器
# 使用 Docker 執行 ZAP 基線掃描

# 基線掃描（快速，適合 CI/CD）
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t https://your-staging-app.example.com \
  -r zap-report.html

# 完整掃描（詳細，適合定期安全審計）
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py \
  -t https://your-staging-app.example.com \
  -r zap-full-report.html

# API 掃描（針對 Nuxt API routes）
docker run --rm -t ghcr.io/zaproxy/zaproxy:stable zap-api-scan.py \
  -t https://your-staging-app.example.com/api/openapi.json \
  -f openapi \
  -r zap-api-report.html
```

### 安全導向的單元測試範例

```typescript
// tests/security/ssrf-protection.test.ts
import { describe, it, expect } from 'vitest'
import { validateExternalUrl, isPrivateIP } from '~/server/utils/ssrf-protection'

describe('SSRF 防護', () => {
  describe('isPrivateIP', () => {
    it('應阻擋 IPv4 私有位址', () => {
      expect(isPrivateIP('10.0.0.1')).toBe(true)
      expect(isPrivateIP('172.16.0.1')).toBe(true)
      expect(isPrivateIP('192.168.1.1')).toBe(true)
      expect(isPrivateIP('127.0.0.1')).toBe(true)
    })

    it('應阻擋 Cloud Metadata 位址', () => {
      expect(isPrivateIP('169.254.169.254')).toBe(true)
    })

    it('應阻擋 IPv6 環回位址', () => {
      expect(isPrivateIP('::1')).toBe(true)
      expect(isPrivateIP('fe80::1')).toBe(true)
    })

    it('應允許合法的外部 IP', () => {
      expect(isPrivateIP('8.8.8.8')).toBe(false)
      expect(isPrivateIP('1.1.1.1')).toBe(false)
      expect(isPrivateIP('203.0.113.1')).toBe(false)
    })
  })

  describe('validateExternalUrl', () => {
    it('應拒絕非 HTTPS 協議', async () => {
      await expect(
        validateExternalUrl('http://example.com'),
      ).rejects.toThrow('不允許的協議')
    })

    it('應拒絕 file:// 協議', async () => {
      await expect(
        validateExternalUrl('file:///etc/passwd'),
      ).rejects.toThrow('不允許的協議')
    })

    it('應拒絕直接的私有 IP', async () => {
      await expect(
        validateExternalUrl('https://127.0.0.1/admin'),
      ).rejects.toThrow('私有 IP')
    })

    it('應拒絕 Cloud Metadata 端點', async () => {
      await expect(
        validateExternalUrl('https://169.254.169.254/latest/meta-data/'),
      ).rejects.toThrow('私有 IP')
    })

    it('應拒絕不在白名單中的網域', async () => {
      await expect(
        validateExternalUrl('https://evil.com/payload', {
          allowedDomains: ['trusted.com'],
        }),
      ).rejects.toThrow('不在白名單中')
    })

    it('應接受白名單中的網域', async () => {
      const result = await validateExternalUrl('https://api.trusted.com/data', {
        allowedDomains: ['trusted.com'],
        dnsCheck: false, // 測試中跳過 DNS 解析
      })
      expect(result.hostname).toBe('api.trusted.com')
    })
  })
})
```

```typescript
// tests/security/xss-protection.test.ts
import { describe, it, expect } from 'vitest'

// 模擬伺服器端 sanitize
import { sanitizeHtmlServer } from '~/server/utils/sanitize-server'

describe('XSS 防護', () => {
  it('應移除 <script> 標籤', () => {
    const dirty = '<p>Hello</p><script>alert("XSS")</script>'
    const clean = sanitizeHtmlServer(dirty)
    expect(clean).toBe('<p>Hello</p>')
    expect(clean).not.toContain('<script>')
  })

  it('應移除事件處理器屬性', () => {
    const dirty = '<img src="x" onerror="alert(1)">'
    const clean = sanitizeHtmlServer(dirty)
    expect(clean).not.toContain('onerror')
  })

  it('應移除 javascript: URI', () => {
    const dirty = '<a href="javascript:alert(1)">click</a>'
    const clean = sanitizeHtmlServer(dirty)
    expect(clean).not.toContain('javascript:')
  })

  it('應保留安全的 HTML 標籤', () => {
    const safe = '<p><strong>粗體</strong>和<em>斜體</em></p>'
    const clean = sanitizeHtmlServer(safe)
    expect(clean).toBe(safe)
  })

  it('應移除 data 屬性', () => {
    const dirty = '<div data-exploit="payload">text</div>'
    const clean = sanitizeHtmlServer(dirty)
    expect(clean).not.toContain('data-exploit')
  })
})
```

```typescript
// tests/security/jwt-hardening.test.ts
import { describe, it, expect } from 'vitest'
import jwt from 'jsonwebtoken'

describe('JWT 安全強化', () => {
  const secret = 'a'.repeat(32) // 測試用 256-bit 密鑰

  it('應拒絕 alg:none 攻擊', () => {
    // 模擬攻擊者篡改 token 使用 none 演算法
    const payload = { sub: 'user-1', role: 'admin' }
    const maliciousToken = jwt.sign(payload, '', { algorithm: 'none' as any })

    expect(() => {
      jwt.verify(maliciousToken, secret, { algorithms: ['HS256'] })
    }).toThrow()
  })

  it('應拒絕過期的 token', () => {
    const expiredToken = jwt.sign(
      { sub: 'user-1', role: 'user' },
      secret,
      { algorithm: 'HS256', expiresIn: '-1s' },
    )

    expect(() => {
      jwt.verify(expiredToken, secret, { algorithms: ['HS256'] })
    }).toThrow('jwt expired')
  })

  it('應拒絕錯誤 issuer 的 token', () => {
    const token = jwt.sign(
      { sub: 'user-1' },
      secret,
      { algorithm: 'HS256', issuer: 'evil.com' },
    )

    expect(() => {
      jwt.verify(token, secret, {
        algorithms: ['HS256'],
        issuer: 'myapp.com',
      })
    }).toThrow('jwt issuer invalid')
  })

  it('應拒絕 RS256 token 使用 HS256 密鑰驗證（algorithm confusion）', () => {
    // 攻擊者可能嘗試將公鑰當作 HS256 密鑰來偽造 token
    const token = jwt.sign({ sub: 'user-1', role: 'admin' }, secret, {
      algorithm: 'HS256',
    })

    // 使用 RS256 驗證時應失敗
    expect(() => {
      jwt.verify(token, secret, { algorithms: ['RS256'] })
    }).toThrow()
  })
})
```

### Nuxt SSR 應用滲透測試檢查清單

```markdown
## Nuxt SSR 應用滲透測試檢查清單

### 1. 認證與授權
- [ ] 測試所有 API 端點的未認證存取
- [ ] 測試 JWT algorithm confusion 攻擊（alg:none, HS256→RS256）
- [ ] 測試 Refresh Token 重用偵測
- [ ] 測試 IDOR（修改 URL/body 中的 ID 參數存取他人資源）
- [ ] 測試水平越權（同角色間的資料隔離）
- [ ] 測試垂直越權（低權限帳號存取管理功能）

### 2. 輸入驗證
- [ ] 在所有表單欄位測試 XSS 載體（含 stored/reflected/DOM-based）
- [ ] 在搜尋、篩選參數測試 SQL/NoSQL Injection
- [ ] 測試檔案上傳繞過（MIME type 偽裝、路徑穿越）
- [ ] 測試超長輸入與特殊字元（Unicode、Null byte）
- [ ] 測試 HTTP Parameter Pollution

### 3. SSRF（SSR 特有）
- [ ] 在所有接受 URL 輸入的端點測試 SSRF
- [ ] 測試 Cloud Metadata 存取（169.254.169.254）
- [ ] 測試 DNS Rebinding 攻擊
- [ ] 測試 IPv6 繞過（[::1]、[0:0:0:0:0:ffff:127.0.0.1]）
- [ ] 測試 URL 編碼繞過（%31%32%37.0.0.1）
- [ ] 測試 302 重定向繞過（先訪問外部 URL，重定向到內網）

### 4. Session 與 Cookie
- [ ] 確認所有敏感 Cookie 設定 HttpOnly、Secure、SameSite
- [ ] 測試 Session Fixation 攻擊
- [ ] 測試 CSRF（Double Submit Cookie 是否正確實作）
- [ ] 確認登出時 Session/Token 確實失效

### 5. HTTP 安全標頭
- [ ] 確認 CSP 正確設定且未使用 unsafe-eval
- [ ] 確認 HSTS 已啟用且含 preload
- [ ] 確認 X-Frame-Options 為 DENY
- [ ] 確認 X-Content-Type-Options 為 nosniff
- [ ] 確認沒有不必要的 Server header 洩漏版本資訊

### 6. 業務邏輯
- [ ] 測試 Race Condition（如：並發扣款、重複使用優惠碼）
- [ ] 測試 Rate Limiting 繞過（切換 IP、分散請求）
- [ ] 測試價格竄改（修改購物車金額）
- [ ] 測試流程跳過（跳過付款步驟直接完成訂單）

### 7. 資訊洩漏
- [ ] 確認生產環境不回傳堆疊追蹤
- [ ] 確認 Source Map 未在生產環境公開
- [ ] 確認 .env 檔案不可透過 HTTP 存取
- [ ] 確認 API 錯誤訊息不洩漏內部細節（資料庫錯誤、檔案路徑）
- [ ] 檢查 /_nuxt/ 下是否洩漏敏感資訊
```

---

## 認證端點強化

### 差異化速率限制

認證端點（登入、註冊、密碼重設）需要比一般 API 更嚴格的速率限制，因為這些端點是暴力破解攻擊的主要目標。

```typescript
// server/utils/rate-limiter.ts — 差異化速率限制器
interface RateLimitConfig {
  /** 時間窗口（毫秒） */
  windowMs: number
  /** 時間窗口內允許的最大請求數 */
  maxRequests: number
  /** 封鎖時間（毫秒）— 超過限制後的封鎖持續時間 */
  blockDurationMs?: number
  /** 以何種維度限制（IP / IP+endpoint / userId） */
  keyStrategy: 'ip' | 'ip-endpoint' | 'user'
}

interface RateLimitEntry {
  count: number
  resetAt: number
  blockedUntil?: number
}

/**
 * 預設速率限制設定（分層）
 */
export const RATE_LIMIT_PRESETS = {
  /** 一般 API：每分鐘 60 次 */
  standard: {
    windowMs: 60_000,
    maxRequests: 60,
    keyStrategy: 'ip' as const,
  },
  /** 認證端點：每 15 分鐘 5 次，超過後封鎖 15 分鐘 */
  auth: {
    windowMs: 15 * 60_000,
    maxRequests: 5,
    blockDurationMs: 15 * 60_000,
    keyStrategy: 'ip-endpoint' as const,
  },
  /** 密碼重設：每小時 3 次 */
  passwordReset: {
    windowMs: 60 * 60_000,
    maxRequests: 3,
    blockDurationMs: 60 * 60_000,
    keyStrategy: 'ip' as const,
  },
  /** 驗證碼驗證：每 10 分鐘 5 次 */
  otpVerify: {
    windowMs: 10 * 60_000,
    maxRequests: 5,
    blockDurationMs: 30 * 60_000,
    keyStrategy: 'ip-endpoint' as const,
  },
} satisfies Record<string, RateLimitConfig>

/**
 * 基於記憶體的速率限制器（生產環境建議使用 Redis）
 */
const store = new Map<string, RateLimitEntry>()

// 定期清理過期條目
setInterval(() => {
  const now = Date.now()
  for (const [key, entry] of store.entries()) {
    if (entry.resetAt < now && (!entry.blockedUntil || entry.blockedUntil < now)) {
      store.delete(key)
    }
  }
}, 60_000)

/**
 * 產生速率限制鍵值
 */
function generateKey(event: H3Event, config: RateLimitConfig): string {
  const ip = getRequestIP(event, { xForwardedFor: true }) ?? 'unknown'
  switch (config.keyStrategy) {
    case 'ip':
      return `rl:${ip}`
    case 'ip-endpoint':
      return `rl:${ip}:${getRequestURL(event).pathname}`
    case 'user': {
      const userId = event.context.user?.id ?? ip
      return `rl:user:${userId}`
    }
  }
}

/**
 * 執行速率限制檢查
 * @returns 剩餘請求數；若被限制則拋出 429 錯誤
 */
export function checkRateLimit(event: H3Event, config: RateLimitConfig): number {
  const key = generateKey(event, config)
  const now = Date.now()
  let entry = store.get(key)

  // 檢查是否被封鎖
  if (entry?.blockedUntil && entry.blockedUntil > now) {
    const retryAfter = Math.ceil((entry.blockedUntil - now) / 1000)
    setResponseHeader(event, 'Retry-After', String(retryAfter))
    throw createError({
      statusCode: 429,
      message: `請求過於頻繁，請 ${retryAfter} 秒後重試`,
    })
  }

  // 重置過期的計數器
  if (!entry || entry.resetAt < now) {
    entry = { count: 0, resetAt: now + config.windowMs }
    store.set(key, entry)
  }

  entry.count++

  // 設定速率限制相關 HTTP 標頭
  const remaining = Math.max(0, config.maxRequests - entry.count)
  setResponseHeaders(event, {
    'X-RateLimit-Limit': String(config.maxRequests),
    'X-RateLimit-Remaining': String(remaining),
    'X-RateLimit-Reset': String(Math.ceil(entry.resetAt / 1000)),
  })

  // 超過限制
  if (entry.count > config.maxRequests) {
    if (config.blockDurationMs) {
      entry.blockedUntil = now + config.blockDurationMs
    }
    const retryAfter = Math.ceil((entry.resetAt - now) / 1000)
    setResponseHeader(event, 'Retry-After', String(retryAfter))
    throw createError({
      statusCode: 429,
      message: `請求過於頻繁，請 ${retryAfter} 秒後重試`,
    })
  }

  return remaining
}
```

```typescript
// server/middleware/auth-rate-limit.ts — 認證端點專用速率限制中介層
export default defineEventHandler((event) => {
  const path = getRequestURL(event).pathname

  // 登入端點：最嚴格的速率限制
  if (path === '/api/auth/login' && getMethod(event) === 'POST') {
    checkRateLimit(event, RATE_LIMIT_PRESETS.auth)
    return
  }

  // 註冊端點
  if (path === '/api/auth/register' && getMethod(event) === 'POST') {
    checkRateLimit(event, RATE_LIMIT_PRESETS.auth)
    return
  }

  // 密碼重設請求
  if (path === '/api/auth/forgot-password' && getMethod(event) === 'POST') {
    checkRateLimit(event, RATE_LIMIT_PRESETS.passwordReset)
    return
  }

  // OTP / 驗證碼驗證
  if (path === '/api/auth/verify-otp' && getMethod(event) === 'POST') {
    checkRateLimit(event, RATE_LIMIT_PRESETS.otpVerify)
    return
  }

  // 一般 API 端點
  if (path.startsWith('/api/')) {
    checkRateLimit(event, RATE_LIMIT_PRESETS.standard)
  }
})
```

### 帳號鎖定 / 暴力破解防護

```typescript
// server/utils/account-lockout.ts — 帳號鎖定機制
interface LockoutConfig {
  /** 連續失敗幾次後鎖定 */
  maxFailedAttempts: number
  /** 鎖定持續時間（毫秒） */
  lockoutDurationMs: number
  /** 失敗計數重置時間（毫秒）— 超過此時間未失敗則重置計數 */
  failureWindowMs: number
}

interface AccountLockoutState {
  failedAttempts: number
  lastFailedAt: number
  lockedUntil: number | null
}

const DEFAULT_LOCKOUT_CONFIG: LockoutConfig = {
  maxFailedAttempts: 5,
  lockoutDurationMs: 15 * 60_000,  // 15 分鐘
  failureWindowMs: 30 * 60_000,    // 30 分鐘窗口
}

// 生產環境建議使用 Redis 替代 Map
const lockoutStore = new Map<string, AccountLockoutState>()

/**
 * 記錄登入失敗
 */
export function recordFailedLogin(
  identifier: string, // email 或 userId
  config: LockoutConfig = DEFAULT_LOCKOUT_CONFIG,
): { locked: boolean; remainingAttempts: number; lockoutMinutes?: number } {
  const now = Date.now()
  let state = lockoutStore.get(identifier)

  // 重置過期的失敗計數
  if (state && (now - state.lastFailedAt) > config.failureWindowMs) {
    state = undefined
  }

  if (!state) {
    state = { failedAttempts: 0, lastFailedAt: now, lockedUntil: null }
  }

  state.failedAttempts++
  state.lastFailedAt = now

  // 達到失敗上限，鎖定帳號
  if (state.failedAttempts >= config.maxFailedAttempts) {
    state.lockedUntil = now + config.lockoutDurationMs
    lockoutStore.set(identifier, state)
    return {
      locked: true,
      remainingAttempts: 0,
      lockoutMinutes: Math.ceil(config.lockoutDurationMs / 60_000),
    }
  }

  lockoutStore.set(identifier, state)
  return {
    locked: false,
    remainingAttempts: config.maxFailedAttempts - state.failedAttempts,
  }
}

/**
 * 檢查帳號是否被鎖定
 */
export function isAccountLocked(identifier: string): {
  locked: boolean
  remainingSeconds?: number
} {
  const state = lockoutStore.get(identifier)
  if (!state?.lockedUntil) return { locked: false }

  const now = Date.now()
  if (state.lockedUntil <= now) {
    // 鎖定已過期，清除狀態
    lockoutStore.delete(identifier)
    return { locked: false }
  }

  return {
    locked: true,
    remainingSeconds: Math.ceil((state.lockedUntil - now) / 1000),
  }
}

/**
 * 登入成功後重置失敗計數
 */
export function resetFailedLogins(identifier: string): void {
  lockoutStore.delete(identifier)
}
```

### 登入嘗試日誌記錄

```typescript
// server/api/auth/login.post.ts — 整合所有認證強化措施
export default defineEventHandler(async (event) => {
  const body = await readValidatedBody(event, (raw) => {
    return z.object({
      email: z.string().email(),
      password: z.string().min(1),
    }).parse(raw)
  })

  const { email, password } = body

  // 步驟 1：檢查帳號是否已被鎖定
  const lockStatus = isAccountLocked(email)
  if (lockStatus.locked) {
    auditLog(event, {
      userId: null,
      action: 'LOGIN_BLOCKED_LOCKED',
      resource: 'auth',
      result: 'failure',
      details: { email, remainingSeconds: lockStatus.remainingSeconds },
    })
    throw createError({
      statusCode: 423,
      message: `帳號已暫時鎖定，請 ${lockStatus.remainingSeconds} 秒後重試`,
    })
  }

  // 步驟 2：查詢使用者
  const user = await findUserByEmail(email)

  // 防止使用者列舉攻擊：不論帳號是否存在，都執行密碼比對
  // 這確保了回應時間一致，攻擊者無法透過時間差判斷帳號是否存在
  if (!user) {
    // 執行虛擬密碼比對以保持一致的回應時間
    await verifyPassword(password, '$2b$12$LJ3m4ys3GZxkFPMMPbqSauJOGCcjCtu0SOON4JY.2vryId0myDpCS')

    recordFailedLogin(email)
    auditLog(event, {
      userId: null,
      action: 'LOGIN_FAILED_USER_NOT_FOUND',
      resource: 'auth',
      result: 'failure',
      details: { email },
    })

    // 通用錯誤訊息，不透露帳號是否存在
    throw createError({ statusCode: 401, message: '電子郵件或密碼不正確' })
  }

  // 步驟 3：驗證密碼
  const isValid = await verifyPassword(password, user.passwordHash)
  if (!isValid) {
    const lockResult = recordFailedLogin(email)
    auditLog(event, {
      userId: user.id,
      action: 'LOGIN_FAILED_WRONG_PASSWORD',
      resource: 'auth',
      result: 'failure',
      details: {
        email,
        remainingAttempts: lockResult.remainingAttempts,
        locked: lockResult.locked,
      },
    })

    if (lockResult.locked) {
      throw createError({
        statusCode: 423,
        message: `登入失敗次數過多，帳號已鎖定 ${lockResult.lockoutMinutes} 分鐘`,
      })
    }

    throw createError({
      statusCode: 401,
      message: `電子郵件或密碼不正確（剩餘 ${lockResult.remainingAttempts} 次嘗試機會）`,
    })
  }

  // 步驟 4：登入成功
  resetFailedLogins(email)

  // 簽發 Token
  const accessToken = signAccessTokenRS256({ sub: user.id, role: user.role })
  const refreshToken = signRefreshToken(user.id)

  // 設定安全 Cookie
  setCookie(event, 'refresh_token', refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',
    path: '/api/auth/refresh',
    maxAge: 7 * 24 * 60 * 60,
  })

  // 記錄成功登入
  auditLog(event, {
    userId: user.id,
    action: 'LOGIN_SUCCESS',
    resource: 'auth',
    result: 'success',
    details: { email },
  })

  return {
    accessToken,
    user: { id: user.id, email: user.email, role: user.role },
  }
})
```

```typescript
// server/utils/login-anomaly-detection.ts — 登入異常偵測
interface LoginAttemptLog {
  timestamp: number
  ip: string
  userAgent: string
  success: boolean
  geoLocation?: string
}

const loginHistory = new Map<string, LoginAttemptLog[]>()
const MAX_HISTORY = 100

/**
 * 記錄登入嘗試（成功或失敗）
 */
export function recordLoginAttempt(
  userId: string,
  attempt: Omit<LoginAttemptLog, 'timestamp'>,
): void {
  const history = loginHistory.get(userId) ?? []
  history.push({ ...attempt, timestamp: Date.now() })
  // 僅保留最近 N 筆記錄
  if (history.length > MAX_HISTORY) history.shift()
  loginHistory.set(userId, history)
}

/**
 * 偵測可疑登入行為
 */
export function detectLoginAnomalies(
  userId: string,
  currentIp: string,
  currentUserAgent: string,
): string[] {
  const history = loginHistory.get(userId)
  if (!history || history.length < 2) return []

  const anomalies: string[] = []
  const recentSuccesses = history.filter((h) => h.success).slice(-10)

  // 偵測 1：從未見過的 IP 登入
  const knownIps = new Set(recentSuccesses.map((h) => h.ip))
  if (knownIps.size > 0 && !knownIps.has(currentIp)) {
    anomalies.push(`從新 IP 登入：${currentIp}`)
  }

  // 偵測 2：從未見過的 User-Agent 登入
  const knownAgents = new Set(recentSuccesses.map((h) => h.userAgent))
  if (knownAgents.size > 0 && !knownAgents.has(currentUserAgent)) {
    anomalies.push('從新裝置/瀏覽器登入')
  }

  // 偵測 3：短時間內從多個 IP 登入
  const recentWindow = Date.now() - 10 * 60_000 // 最近 10 分鐘
  const recentIps = new Set(
    history
      .filter((h) => h.timestamp > recentWindow && h.success)
      .map((h) => h.ip),
  )
  if (recentIps.size > 3) {
    anomalies.push(`10 分鐘內從 ${recentIps.size} 個不同 IP 登入`)
  }

  // 偵測 4：連續失敗後突然成功（可能是暴力破解成功）
  const lastFive = history.slice(-5)
  const recentFailures = lastFive.filter((h) => !h.success).length
  if (recentFailures >= 3) {
    anomalies.push(`登入前有 ${recentFailures} 次失敗嘗試`)
  }

  return anomalies
}
```

> **安全警告：** 上述帳號鎖定與速率限制的記憶體儲存方案僅適合單一伺服器部署。在多實例（Kubernetes / PM2 cluster）環境下，必須改用 Redis 等共享儲存，確保所有實例共享同一份限制狀態。可使用 Nitro 的 `useStorage('redis')` 搭配 `unstorage` 的 Redis 驅動實現。
