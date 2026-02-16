# Production Readiness Checklist

## 目錄

1. [安全性 (Security)](#安全性-security)
2. [效能 (Performance)](#效能-performance)
3. [可靠性 (Reliability)](#可靠性-reliability)
4. [可觀測性 (Observability)](#可觀測性-observability)
5. [部署 (Deployment)](#部署-deployment)
6. [程式碼品質 (Code Quality)](#程式碼品質-code-quality)
7. [文件 (Documentation)](#文件-documentation)

---

> **使用方式**：複製此 Checklist 至 Issue 或 PR 描述中，逐項確認後勾選。P0 項目必須在上線前完成；P1 項目建議於上線後 30 天內補齊。

---

## 安全性 (Security)

### 傳輸與通訊

- [ ] **HTTPS 強制啟用** — 所有環境皆透過 TLS 傳輸，HTTP 自動重導 HTTPS。
  > 配置：反向代理（Nginx/Cloudflare）或 `nuxt.config.ts` 的 `routeRules`

- [ ] **CORS 正確配置** — 僅允許已知 Origin，禁止 wildcard `*` 用於需認證的 API。
  > 參考：[`server-api.md`](./server-api.md)

### 安全標頭

- [ ] **CSP nonce-based 配置** — 使用 nonce 而非 `unsafe-inline`，防止 XSS。
  ```typescript
  // nuxt.config.ts — 使用 nuxt-security 模組
  security: {
    headers: {
      contentSecurityPolicy: {
        'script-src': ["'nonce-{{nonce}}'", "'strict-dynamic'"]
      }
    },
    nonce: true
  }
  ```

- [ ] **安全標頭完整** — 確認以下標頭皆已設定：
  - `Strict-Transport-Security`：`max-age=63072000; includeSubDomains; preload`
  - `X-Frame-Options`：`DENY` 或 `SAMEORIGIN`
  - `X-Content-Type-Options`：`nosniff`
  - `Referrer-Policy`：`strict-origin-when-cross-origin`
  - `Permissions-Policy`：停用非必要瀏覽器功能

### 認證與授權

- [ ] **CSRF 防護啟用** — 所有狀態變更請求（POST/PUT/DELETE）皆驗證 CSRF Token。
  > 參考：[`server-api.md`](./server-api.md) 認證章節

- [ ] **JWT secret 長度 ≥ 256 bits** — HS256 需 secret ≥ 32 bytes；建議改用 RS256/EdDSA。
  ```bash
  openssl rand -base64 64  # 產生安全 secret
  ```

- [ ] **密碼使用 bcrypt/argon2 雜湊** — 禁止明文或弱雜湊（MD5、SHA-1）。

### 防護機制

- [ ] **Rate Limiting 啟用** — API 端點設有頻率限制，防止暴力破解與 DDoS。

- [ ] **敏感資料不外洩** — 生產環境的錯誤回應不包含 stack trace、內部路徑或 SQL 查詢。

- [ ] **依賴安全掃描通過** — `npm audit` 無 high/critical 等級漏洞。
  > 參考：[`maintenance-and-versioning.md`](./maintenance-and-versioning.md) 安全掃描章節

---

## 效能 (Performance)

### Core Web Vitals

- [ ] **Lighthouse Score ≥ 90** — Performance、Accessibility、Best Practices、SEO 四項皆 ≥ 90。
  ```bash
  npx lhci autorun --config=lighthouserc.json
  ```

- [ ] **Core Web Vitals 通過** — 符合 Google 良好體驗門檻：
  - LCP (Largest Contentful Paint) < 2.5 秒
  - CLS (Cumulative Layout Shift) < 0.1
  - INP (Interaction to Next Paint) < 200ms
  > 使用 `web-vitals` 套件收集 RUM 指標

### 資源最佳化

- [ ] **Bundle Size 預算內** — 首次載入 JS < 200KB gzip。
  > 參考：[`performance-and-seo.md`](./performance-and-seo.md)

- [ ] **圖片最佳化** — 使用次世代格式（WebP/AVIF），實作 lazy loading 與 responsive sizes。
  ```vue
  <NuxtImg src="/hero.jpg" format="webp" loading="lazy" sizes="sm:100vw md:50vw lg:800px" />
  ```

### 基礎設施

- [ ] **CDN 配置完成** — 靜態資源透過 CDN 分發，設有適當 Cache-Control。

- [ ] **快取策略定義** — 各類資源快取策略：
  - 靜態資源：`public, max-age=31536000, immutable`
  - HTML 頁面：`no-cache` 或 `s-maxage=60, stale-while-revalidate=600`
  - API 回應：依業務邏輯設定 `swr`
  > 參考：[`performance-and-seo.md`](./performance-and-seo.md) 快取章節

---

## 可靠性 (Reliability)

- [ ] **健康檢查端點** — `/api/health` 回傳應用與依賴服務健康狀態。
  ```typescript
  // server/api/health.get.ts
  export default defineEventHandler(async () => ({
    status: (await checkDB()) ? 'healthy' : 'degraded',
    timestamp: new Date().toISOString()
  }))
  ```

- [ ] **優雅關機處理** — 收到 SIGTERM 時完成進行中請求後再關閉。

- [ ] **錯誤追蹤整合（Sentry）** — 前後端整合錯誤追蹤，含 source map 上傳。
  ```typescript
  // nuxt.config.ts
  modules: ['@sentry/nuxt/module'],
  sentry: { sourceMapsUploadOptions: { org: 'your-org', project: 'your-project' } }
  ```
  > 參考：[`troubleshooting.md`](./troubleshooting.md)

- [ ] **日誌結構化** — JSON 格式輸出，包含 request ID、timestamp、level。

- [ ] **SLO 定義** — 已定義並監控關鍵指標：
  - 可用性 ≥ 99.9%（月停機 < 43 分鐘）
  - 回應時間 P95 < 500ms（API）、P95 < 2s（頁面）
  - 錯誤率 < 0.1%（5xx）

---

## 可觀測性 (Observability)

- [ ] **Metrics 收集** — 收集請求量、錯誤率、回應時間、CPU/Memory 使用率。
  > 工具：Prometheus + Grafana 或 Datadog

- [ ] **Distributed Tracing** — 跨服務追蹤，Trace ID 串連前後端日誌。
  > 工具：OpenTelemetry + Jaeger/Zipkin 或 Sentry Performance

- [ ] **Alerting 配置** — 指標超出 SLO 時自動通知：
  - 錯誤率 > 1% 持續 5 分鐘 → P1 告警
  - P95 延遲 > 2s 持續 10 分鐘 → P2 告警
  - 健康檢查失敗 → P0 告警

- [ ] **Dashboard 建立** — 涵蓋即時流量、錯誤率、Core Web Vitals、基礎設施使用率、業務 KPI。
  > 工具：Grafana、Datadog 或 Vercel Analytics

---

## 部署 (Deployment)

- [ ] **CI/CD Pipeline 完整** — Lint → Type Check → Test → Build → Deploy。
  ```yaml
  # .github/workflows/deploy.yml
  jobs:
    ci:
      steps: [checkout, setup-node, install, lint, typecheck, test, build]
    deploy-staging:
      needs: ci
      if: github.ref == 'refs/heads/develop'
    deploy-production:
      needs: ci
      if: github.ref == 'refs/heads/main'
  ```
  > 參考：[`server-and-deployment.md`](./server-and-deployment.md)

- [ ] **環境分離** — dev / staging / prod 各自獨立，配置透過環境變數區分。

- [ ] **回滾策略測試** — 已驗證可在 5 分鐘內回滾至前一穩定版本。

- [ ] **Database Migration 策略** — 向前相容，支援 zero-downtime 部署。
  - 新增欄位設有預設值；刪除欄位分兩階段（先停用再移除）

- [ ] **環境變數管理** — 所有變數有文件記錄，敏感變數透過 Secrets Manager 管理。
  ```typescript
  // server/utils/env.ts
  import { z } from 'zod'
  const envSchema = z.object({
    DATABASE_URL: z.string().url(),
    JWT_SECRET: z.string().min(32),
    SENTRY_DSN: z.string().url().optional()
  })
  export const env = envSchema.parse(process.env)
  ```

---

## 程式碼品質 (Code Quality)

- [ ] **TypeScript Strict Mode** — `tsconfig.json` 啟用 `strict: true`，無 `any` escape。

- [ ] **ESLint 零警告** — CI 以 `npx eslint . --max-warnings 0` 執行。

- [ ] **測試覆蓋率 ≥ 80%** — Lines、Functions、Branches 皆達標。
  > 參考：[`testing.md`](./testing.md)

- [ ] **無已知安全漏洞** — `npm audit` 無 high/critical 漏洞。
  > 參考：[`maintenance-and-versioning.md`](./maintenance-and-versioning.md)

- [ ] **ADR 文件更新** — 所有架構決策皆有對應的 Architecture Decision Record。
  > 參考：[`architecture.md`](./architecture.md)

---

## 文件 (Documentation)

- [ ] **API 文件（OpenAPI/Swagger）** — 所有公開 API 有 OpenAPI 3.0 規格，含請求/回應範例。
  ```typescript
  // nuxt.config.ts — 自動產生 OpenAPI
  nitro: { experimental: { openAPI: true } }
  // 存取：/_nitro/openapi.json、/_nitro/swagger
  ```

- [ ] **部署文件** — 完整部署流程、環境需求、設定步驟與常見問題。

- [ ] **故障排除指南 (Runbook)** — 常見故障情境的診斷步驟與修復方法。
  > 參考：[`troubleshooting.md`](./troubleshooting.md)

- [ ] **Onboarding 文件** — 新成員可在 1 天內完成環境設定並提交首個 PR。

---

## 簽核紀錄

| 角色 | 姓名 | 日期 | 狀態 |
|------|------|------|------|
| Tech Lead | | | 待審核 |
| QA Lead | | | 待審核 |
| Security | | | 待審核 |
| SRE / DevOps | | | 待審核 |
| Product Owner | | | 待審核 |

> **注意**：所有 P0 項目（安全性、健康檢查、CI/CD）必須上線前 100% 完成。P1 項目（進階可觀測性、完整文件）建議上線後 30 天內補齊。
