---
name: nuxt3-fullstack-guide
description: "Nuxt 3 Enterprise-Grade 全端開發工程指南（Staff/Principal Engineer 等級）。適用情境：(1) 建立或架構 Nuxt 3 專案、(2) 撰寫 TypeScript 元件、composables、API routes、(3) 使用 Vitest 撰寫測試、(4) Tailwind CSS 整合與設計實作、(5) 解決 SSR/CSR hydration 問題、(6) 效能最佳化、SEO 設定、(7) 部署策略選擇（SSR/SSG/ISR）、(8) 狀態管理（Pinia/useState）、(9) 資料抓取與快取策略、(10) 中介層與外掛設計、(11) 企業級架構（Clean Architecture、ADR、Scalability）、(12) 韌性與安全工程（STRIDE、JWT Hardening、CSRF、CSP nonce）、(13) DevOps 與可觀測性（CI/CD、IaC、OpenTelemetry、SLO/SLA）、(14) 生產就緒檢查清單。當使用者提及 Nuxt、Nuxt 3、Vue 3 全端開發、或上述任何技術組合時觸發此技能。所有指引皆為繁體中文。"
---

# Nuxt 3 Enterprise-Grade 全端開發工程指南

Nuxt 3 + TypeScript + Vitest + Tailwind CSS **企業級**最佳實踐與架構指南。涵蓋專案架構、元件設計、狀態管理、資料抓取、測試策略、效能最佳化、SEO、部署、**韌性工程、安全工程、DevOps、可觀測性**等全方位開發實務。適用 Staff / Principal Engineer 等級的技術決策與實作指引。

### 技術棧版本

| 技術 | 版本 |
|------|------|
| Nuxt | 3.14+（建議啟用 `compatibilityVersion: 4` 為 Nuxt 4 做準備） |
| Vue | 3.5+（含 `useTemplateRef` 等新特性） |
| TypeScript | 5.x（strict mode） |
| Tailwind CSS | **v4**（主要推薦）/ v3.x（既有專案相容） |
| @nuxtjs/tailwindcss | **v7+**（支援 Tailwind v4）/ v6.x（Tailwind v3） |
| @nuxt/fonts | 取代已棄用的 `@nuxtjs/fontaine` |
| Vitest | 2.x |
| ESLint | v9（Flat Config） |

> **Tailwind CSS 版本說明**：本指南以 **Tailwind CSS v4** 為主要推薦版本。v4 採用 CSS-first 配置（`@import "tailwindcss"`、`@theme` 指令），不再需要 `tailwind.config.ts`。既有 v3 專案可繼續使用，但新專案建議直接採用 v4。相關範例請參閱 [Tailwind 設計系統](examples/tailwind-design-system.md)。

## 目錄

- [專案架構](#專案架構)
- [開發工作流程](#開發工作流程)
- [核心決策指引](#核心決策指引)
- [參考文件索引](#參考文件索引)

## 專案架構

### 推薦目錄結構

```
project-root/
├── .nuxt/                    # 自動產生（勿提交）
├── .output/                  # 建置輸出（勿提交）
├── assets/                   # 需經建置處理的靜態資源
│   ├── css/
│   │   └── main.css          # Tailwind 入口 + 全域樣式
│   ├── images/
│   └── fonts/
├── components/               # 自動匯入的 Vue 元件
│   ├── base/                 # 基礎 UI 元件（BaseButton, BaseInput...）
│   ├── layout/               # 佈局元件（TheHeader, TheSidebar...）
│   ├── feature/              # 功能模組元件
│   │   ├── auth/
│   │   ├── dashboard/
│   │   └── settings/
│   └── shared/               # 跨功能共用元件
├── composables/              # 自動匯入的組合式函式
│   ├── useAuth.ts
│   ├── useApi.ts
│   └── useFormValidation.ts
├── layouts/                  # 佈局模板
│   ├── default.vue
│   ├── auth.vue
│   └── dashboard.vue
├── middleware/               # 路由中介層
│   ├── auth.ts
│   └── guest.ts
├── pages/                    # 檔案式路由
│   ├── index.vue
│   ├── login.vue
│   └── dashboard/
│       ├── index.vue
│       └── [id].vue
├── plugins/                  # Nuxt 外掛
│   ├── 01.auth.ts            # 數字前綴控制載入順序
│   └── 02.api.ts
├── public/                   # 不經處理的靜態資源
│   ├── favicon.ico
│   └── robots.txt
├── server/                   # 伺服器端（Nitro/H3）
│   ├── api/                  # API 路由
│   │   ├── auth/
│   │   │   ├── login.post.ts
│   │   │   └── me.get.ts
│   │   └── users/
│   │       ├── index.get.ts
│   │       └── [id].get.ts
│   ├── middleware/            # 伺服器中介層
│   │   └── auth.ts
│   ├── utils/                # 伺服器工具函式
│   │   ├── db.ts
│   │   └── validators.ts
│   └── plugins/              # Nitro 外掛
├── stores/                   # Pinia 狀態管理
│   ├── auth.ts
│   └── ui.ts
├── tests/                    # 測試檔案
│   ├── unit/
│   │   ├── components/
│   │   └── composables/
│   ├── integration/
│   └── setup.ts
├── types/                    # TypeScript 型別定義
│   ├── index.d.ts
│   ├── api.ts
│   └── models.ts
├── utils/                    # 自動匯入的工具函式
│   ├── format.ts
│   └── validation.ts
├── nuxt.config.ts
├── eslint.config.mjs            # ESLint Flat Config（由 @nuxt/eslint 驅動）
├── tailwind.config.ts        # 僅 Tailwind v3 需要（v4 改用 CSS-first 配置，不需此檔案）
├── vitest.config.ts
├── tsconfig.json
├── app.vue
└── error.vue
```

### 命名規範

| 類型 | 規範 | 範例 |
|------|------|------|
| 元件 | PascalCase，基礎元件用 `Base` 前綴 | `BaseButton.vue`, `UserCard.vue` |
| 單例元件 | `The` 前綴 | `TheHeader.vue`, `TheSidebar.vue` |
| Composables | `use` 前綴，camelCase | `useAuth.ts`, `useFormValidation.ts` |
| API 路由 | kebab-case + HTTP 方法後綴 | `login.post.ts`, `users.get.ts` |
| Store | camelCase，與 domain 對應 | `auth.ts`, `userProfile.ts` |
| 型別 | PascalCase，用 interface 優先 | `User`, `ApiResponse<T>` |
| 工具函式 | camelCase | `formatDate.ts`, `slugify.ts` |

## 開發工作流程

### 新功能開發流程

1. **定義型別** → `types/` 中建立介面與型別
2. **建立 API** → `server/api/` 中實作端點，含驗證
3. **建立 Store** → `stores/` 中管理狀態（若需要）
4. **建立 Composable** → `composables/` 中封裝業務邏輯
5. **建立元件** → `components/` 中實作 UI
6. **建立頁面** → `pages/` 中組裝元件
7. **撰寫測試** → `tests/` 中涵蓋關鍵路徑
8. **設計審查** → 參照 frontend-design 技能確保 UI 品質

### 決定何時使用什麼

```
需要跨頁面共享狀態？
├── 是 → 用戶端狀態？
│   ├── 簡單狀態 → useState()
│   └── 複雜狀態（多 action、getter）→ Pinia Store
└── 否 → 僅在元件內部用 ref

需要從 API 抓取資料？
├── 簡單直接的 API 呼叫 → useFetch()
├── 複雜邏輯（多 API 組合、資料轉換）→ useAsyncData() + $fetch
└── 使用者互動觸發（如按鈕點擊）→ $fetch（不用 composable 包裝）

需要可重用邏輯？
├── 包含響應式狀態 → composable（use 前綴）
├── 純函式 → utils/
└── 伺服器端 → server/utils/
```

## 核心決策指引

### TypeScript 嚴格模式

`nuxt.config.ts` 中啟用嚴格模式：

```typescript
export default defineNuxtConfig({
  typescript: {
    strict: true,
    typeCheck: true
  }
})
```

### Tailwind CSS 設定

#### Tailwind v4（推薦新專案使用）

```css
/* assets/css/main.css — Tailwind v4 CSS-first 配置 */
@import "tailwindcss";

/* 設計 Token（取代 tailwind.config.ts 的 theme.extend） */
@theme {
  --color-brand-500: #0c8ce9;
  --color-brand-600: #006fc7;
  --font-sans: 'Noto Sans TC', system-ui, sans-serif;
  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;
}
```

```typescript
// nuxt.config.ts — Tailwind v4 搭配方式

// 方式 1：使用 @nuxtjs/tailwindcss v7+（推薦）
export default defineNuxtConfig({
  modules: ['@nuxtjs/tailwindcss'],
  // v4 不需要 tailwind.config.ts，模組會自動偵測
})

// 方式 2：使用 @tailwindcss/vite 外掛（更輕量）
// npm install tailwindcss @tailwindcss/vite
import tailwindcss from '@tailwindcss/vite'

export default defineNuxtConfig({
  vite: { plugins: [tailwindcss()] },
  css: ['~/assets/css/main.css']
})
```

#### Tailwind v3（既有專案相容）

```typescript
// tailwind.config.ts — 僅 Tailwind v3 需要此檔案
import type { Config } from 'tailwindcss'

export default {
  // 使用 @nuxtjs/tailwindcss 模組時 content 會自動偵測，無需手動配置
  // 僅在需要額外掃描路徑時才加入 content
  darkMode: 'class',
  theme: {
    extend: {
      // 在此擴展設計系統
    }
  }
} satisfies Config
```

### Vitest 基礎設定

```typescript
// vitest.config.ts
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    environment: 'nuxt',
    globals: true,
    setupFiles: ['./tests/setup.ts']
  }
})
```

## Nuxt 4 遷移準備

Nuxt 4 預計為下一個主要版本，目前可透過 `compatibilityVersion: 4` 在 Nuxt 3 中提前啟用即將到來的 breaking changes，逐步遷移。

### 啟用相容模式

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  future: {
    compatibilityVersion: 4
  }
})
```

### 已知的 Breaking Changes

啟用 `compatibilityVersion: 4` 後會套用以下變更：

| 變更項目 | Nuxt 3 行為 | Nuxt 4 行為 |
|----------|------------|------------|
| **預設目錄結構** | 專案根目錄即為源碼目錄 | 源碼移至 `app/` 目錄（`pages/`、`components/` 等移入 `app/`） |
| **`useAsyncData` / `useFetch` 的 `data` 預設值** | `data` 初始值為 `null` | 未設定 `default` 時，`data` 初始值為 `undefined` |
| **`sharedPrerenderData`** | 預設關閉 | 預設開啟，預渲染時共享跨頁面資料 |
| **`compilerOptions.propsDestructure`** | 預設關閉 | 預設開啟，支援 Vue 3.5 的解構 props 響應式 |
| **ID 與 chunk 命名** | 舊式命名 | 更一致的命名規則 |

### 新的目錄結構

Nuxt 4 預設將應用程式碼移至 `app/` 目錄，與 `server/` 平行：

```
project-root/
├── app/                      # 應用程式碼（Nuxt 4 預設）
│   ├── components/
│   ├── composables/
│   ├── layouts/
│   ├── middleware/
│   ├── pages/
│   ├── plugins/
│   └── app.vue
├── server/                   # 伺服器端不變
│   ├── api/
│   └── utils/
├── public/
├── shared/                   # 客戶端與伺服器端共用的型別與工具
│   ├── types/
│   └── utils/
└── nuxt.config.ts
```

> **提示**：若不想立即搬遷目錄，可設定 `dir.app` 指向專案根目錄：
> ```typescript
> export default defineNuxtConfig({
>   future: { compatibilityVersion: 4 },
>   dir: { app: '' }  // 維持舊版目錄結構
> })
> ```

### 遷移建議

1. **在既有專案中啟用** `compatibilityVersion: 4`，逐步修正相容性問題
2. **測試 `data` 初始值變更** — 檢查是否有 `data.value === null` 的判斷需改為 `data.value == null`
3. **新專案直接採用** `app/` 目錄結構，提前適應 Nuxt 4 慣例

## 參考文件索引

依需要查閱以下參考文件，每個文件聚焦特定主題：

### 核心開發指引

- **架構與型別系統** → [references/architecture.md](references/architecture.md)
  涵蓋：Nuxt Layers、模組化架構、Monorepo、TypeScript 進階型別、auto-import 型別安全、app.config.ts vs runtimeConfig

- **資料抓取與狀態管理** → [references/data-and-state.md](references/data-and-state.md)
  涵蓋：useAsyncData/useFetch 完整指南、Pinia SSR、快取策略、錯誤處理

- **測試策略** → [references/testing.md](references/testing.md)
  涵蓋：Vitest 設定、元件測試、composable 測試、API route 測試、mock 技巧

- **效能與 SEO** → [references/performance-and-seo.md](references/performance-and-seo.md)
  涵蓋：Lazy loading、Code splitting、圖片最佳化、Core Web Vitals、SEO meta

- **伺服器端 API** → [references/server-api.md](references/server-api.md)
  涵蓋：H3/Nitro API 設計、伺服器中介層、輸入驗證、檔案上傳、Prisma 整合、WebSocket/SSE、背景任務

- **伺服器端與部署** → [references/server-and-deployment.md](references/server-and-deployment.md)
  涵蓋：路由中介層、Plugin、健康檢查、部署策略（Cloudflare/NuxtHub/AWS Lambda/PM2）、CI/CD、環境管理

- **常見問題與解法** → [references/troubleshooting.md](references/troubleshooting.md)
  涵蓋：Hydration mismatch、記憶體洩漏、循環依賴、SSR 陷阱、Sentry 整合、Chunk 載入錯誤、error.vue 重試

### 企業級工程指引 (Enterprise-Grade)

- **企業架構與可擴展性** → [references/enterprise-architecture.md](references/enterprise-architecture.md)
  涵蓋：Clean Architecture 分層設計、Bounded Context 邊界定義、ADR 決策記錄、Horizontal Scaling、多層快取架構、Database Scaling、Edge Computing、Risk Analysis、Cost Analysis

- **韌性與安全工程** → [references/resilience-and-security.md](references/resilience-and-security.md)
  涵蓋：Retry Pattern（指數退避）、Circuit Breaker、Graceful Degradation、STRIDE 威脅模型、JWT Hardening（RS256/ES256）、Password Storage（bcrypt/argon2）、CSRF Protection、CSP Nonce、Secrets Management

- **DevOps 與可觀測性** → [references/devops-and-observability.md](references/devops-and-observability.md)
  涵蓋：Blue-Green/Canary 部署、Rollback 策略、CI/CD 企業級 Pipeline、IaC（SST/Terraform）、Prometheus Metrics、OpenTelemetry Tracing、Structured Logging、SLO/SLA、Error Budget、Alert 分級

- **企業級測試與型別架構** → [references/enterprise-testing-and-types.md](references/enterprise-testing-and-types.md)
  涵蓋：Schema-Driven Types、Runtime Validation、Branded Types、Type-Safe API、Testing Pyramid、Contract Testing、Performance Testing、Chaos Testing

- **維護與版本管理** → [references/maintenance-and-versioning.md](references/maintenance-and-versioning.md)
  涵蓋：依賴生命週期管理（Renovate/Dependabot）、Nuxt 4 遷移策略、Tailwind v4 主線化、技術債管理、維護計畫

- **生產就緒檢查清單** → [references/production-checklist.md](references/production-checklist.md)
  涵蓋：安全性、效能、可靠性、可觀測性、部署、程式碼品質、文件等 7 大類 40+ 項檢查

### 實戰範例集

- **實戰範例** → 瀏覽 [examples/](examples/) 目錄
  包含：認證系統（JWT + OAuth）、CRUD 模式、Composable 設計模式、表單驗證、Tailwind 設計系統、nuxt.config.ts 配方

## 設計整合

建構 UI 時，遵循 frontend-design 技能中的設計思維：
- 選擇大膽且有辨識度的美學方向，避免 AI 生成的泛用風格
- 使用 Tailwind CSS 建立一致的設計系統（CSS 變數 + 設計 Token）
- 字體選擇要有特色，避免預設的 Inter、Roboto、Arial
- 動效使用 CSS transitions 搭配 Vue Transition 元件
- 佈局善用 Tailwind 的 Grid/Flexbox，打造有層次的空間組合
