# 維護與版本管理

## 目錄

1. [Dependency Lifecycle Management](#dependency-lifecycle-management)
2. [Nuxt 4 Migration Strategy](#nuxt-4-migration-strategy)
3. [Tailwind CSS v4 主線化](#tailwind-css-v4-主線化)
4. [Maintenance Plan](#maintenance-plan)

---

## Dependency Lifecycle Management

### 依賴更新策略 — Renovate Bot（推薦）

在專案根目錄建立 `renovate.json`，自動建立依賴更新 PR：

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":separateMajorReleases", "group:allNonMajor"],
  "labels": ["dependencies"],
  "rangeStrategy": "pin",
  "packageRules": [
    {
      "description": "Nuxt 生態系 — 獨立追蹤",
      "matchPackagePatterns": ["^nuxt", "^@nuxt/", "^nitro", "^h3"],
      "groupName": "nuxt ecosystem",
      "automerge": false
    },
    {
      "description": "devDependencies — 自動合併 minor/patch",
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true
    },
    {
      "description": "Tailwind CSS 生態系",
      "matchPackagePatterns": ["^tailwindcss", "^@tailwindcss/", "^@nuxtjs/tailwindcss"],
      "groupName": "tailwind ecosystem"
    }
  ],
  "schedule": ["before 9am on monday"],
  "timezone": "Asia/Taipei",
  "vulnerabilityAlerts": { "enabled": true, "labels": ["security"] }
}
```

> **Dependabot 替代方案**：使用 `.github/dependabot.yml` 配置，原生整合 GitHub 但功能較少。Renovate 支援群組合併、正則匹配、自動合併策略，為首選。

### Semver 版本鎖定策略

| 策略 | 範例 | 適用場景 | 風險 |
|------|------|----------|------|
| Exact (pin) | `"nuxt": "3.15.4"` | 生產依賴、CI 穩定性 | 需手動更新 |
| Tilde | `"lodash": "~4.17.0"` | 信任的 patch 更新 | 極低 |
| Caret | `"vue": "^3.5.0"` | 活躍維護套件 | Minor 可能含行為變更 |

```jsonc
// package.json — 建議策略
{
  "dependencies": {
    "nuxt": "3.15.4",          // exact — 框架核心鎖死
    "vue": "^3.5.0",           // caret — 跟隨 Nuxt 相容範圍
    "pinia": "^3.0.0"          // caret — 狀態管理
  },
  "devDependencies": {
    "typescript": "~5.7.0",    // tilde — 型別穩定性
    "vitest": "^3.0.0"         // caret — 測試工具
  }
}
```

### 安全性漏洞自動掃描

```yaml
# .github/workflows/security.yml
name: Security Audit
on:
  schedule:
    - cron: '0 2 * * 1'  # 每週一凌晨 2 點
  push:
    paths: ['package-lock.json']
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npm audit --omit=dev --audit-level=moderate
      - name: Snyk 掃描（選用）
        uses: snyk/actions/node@master
        env: { SNYK_TOKEN: "${{ secrets.SNYK_TOKEN }}" }
        with: { args: "--severity-threshold=high" }
```

### 依賴更新的 CI 驗證流程

每個依賴更新 PR 須通過：型別檢查 → Lint → 單元測試 → 建置驗證 → Bundle Size → 安全掃描。

```yaml
# .github/workflows/dependency-validation.yml
name: Dependency Validation
on:
  pull_request:
    paths: ['package.json', 'package-lock.json']
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'npm' }
      - run: npm ci
      - run: npx nuxi typecheck
      - run: npx eslint .
      - run: npx vitest run --coverage
      - run: npx nuxi build
      - run: npm audit --omit=dev --audit-level=moderate
```

> **檔案參考**：CI 配置詳見 [`server-and-deployment.md`](./server-and-deployment.md)

---

## Nuxt 4 Migration Strategy

### `compatibilityVersion: 4` 設定與影響

Nuxt 3.x 提供 `future` 旗標，在正式升級前逐步啟用 Nuxt 4 行為：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  future: {
    compatibilityVersion: 4  // 一次啟用所有 Nuxt 4 行為
  }
})
```

| 行為變更 | Nuxt 3 預設 | Nuxt 4 預設 | 影響範圍 |
|----------|------------|------------|---------|
| 目錄結構 | `app/` 非必要 | `app/` 為主目錄 | 檔案路徑 |
| `useAsyncData` 重設 | 保留舊值 | 重設為 `undefined` | 資料流 |
| 元件名稱正規化 | 保留路徑前綴 | 簡化名稱 | 模板引用 |
| 共用預渲染資料 | 停用 | 啟用 | SSG 效能 |

### 已知的 Breaking Changes 清單

1. **目錄結構重組** — `components/`、`composables/`、`pages/`、`layouts/`、`middleware/`、`plugins/` 移至 `app/` 下；`server/` 維持根目錄
2. **資料取得行為** — `useAsyncData` 導航時預設重設為 `undefined`；`sharedPrerenderData` 預設啟用
3. **元件解析** — 巢狀元件自動命名邏輯調整；頁面 meta 掃描更積極
4. **TypeScript 與建置** — 預設啟用更嚴格的型別檢查；模板編譯策略調整

### 漸進式遷移路線圖

#### Phase 1：評估與準備（1-2 週）

```bash
git checkout -b feat/nuxt4-migration
npx nuxi upgrade                    # 更新至最新 Nuxt 3.x
npx nuxi@latest compatibility       # 執行相容性檢查
```

- [ ] 更新所有 Nuxt 生態系依賴至最新版本
- [ ] 執行完整測試套件建立 baseline
- [ ] 審查第三方模組的 Nuxt 4 相容性

#### Phase 2：啟用相容模式（2-3 週）

- [ ] 啟用 `compatibilityVersion: 4`
- [ ] 逐一修復型別錯誤
- [ ] 調整 `useAsyncData` 使用處（處理 `undefined` 重設）
- [ ] 驗證元件名稱解析
- [ ] 執行完整 E2E 測試

#### Phase 3：正式升級（1-2 週）

- [ ] 升級至 Nuxt 4 正式版本
- [ ] 移除 `future.compatibilityVersion` 配置
- [ ] 執行 Codemod 遷移工具
- [ ] 完整回歸測試與 Staging 驗證

### Codemods 工具使用

```bash
npx @nuxt/codemod@latest <codemod-name> <path>   # 執行特定 codemod
npx @nuxt/codemod@latest --list                   # 列出所有可用 codemods
```

> **檔案參考**：架構調整詳見 [`architecture.md`](./architecture.md)

---

## Tailwind CSS v4 主線化

### v4 作為新專案預設

Tailwind CSS v4 採用 **CSS-first 配置**，不再需要 `tailwind.config.js`：

```css
/* assets/css/main.css — v4 CSS-first 配置 */
@import "tailwindcss";

@theme {
  --color-primary: oklch(0.6 0.22 275);
  --color-secondary: oklch(0.7 0.15 200);
  --color-accent: oklch(0.75 0.18 150);
  --font-display: "Inter Variable", sans-serif;
  --font-body: "Noto Sans TC", sans-serif;
  --breakpoint-3xl: 120rem;
  --radius-pill: 9999px;
  --animate-slide-in: slide-in 0.3s ease-out;
}

@keyframes slide-in {
  from { transform: translateX(-100%); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}

@utility content-grid {
  display: grid;
  grid-template-columns:
    [full-start] minmax(1rem, 1fr)
    [content-start] minmax(0, 80rem)
    [content-end] minmax(1rem, 1fr)
    [full-end];
}
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/tailwindcss'],
  css: ['~/assets/css/main.css'],
  tailwindcss: { exposeConfig: false, viewer: false }
})
```

#### v4 核心概念對照

| 概念 | v3 (tailwind.config.js) | v4 (CSS-first) |
|------|------------------------|-----------------|
| 配置入口 | `tailwind.config.js` | CSS `@theme` |
| 主題擴展 | `theme.extend.colors` | `--color-*` CSS 變數 |
| 插件 | `plugins: [...]` | `@plugin` 指令 |
| 自定義 Utility | `addUtilities()` | `@utility` 指令 |
| 內容掃描 | `content: [...]` | 自動偵測 |
| 前綴 | `prefix: 'tw-'` | `@import "tailwindcss" prefix(tw)` |

### v3 到 v4 遷移策略

```bash
# 使用官方升級工具（自動處理 config 轉換、class 更名、PostCSS 更新）
npx @tailwindcss/upgrade
```

#### 遷移 Checklist

- [ ] 執行 `npx @tailwindcss/upgrade` 自動遷移
- [ ] 移除 `tailwind.config.js`（轉為 CSS `@theme`）
- [ ] 更新 `postcss.config.js`（v4 有內建 PostCSS 支援）
- [ ] 檢查自定義插件是否已轉為 `@plugin` / `@utility`
- [ ] 驗證 Dark Mode（v4 預設 `@media (prefers-color-scheme: dark)`）
- [ ] 檢查 `@apply` 使用處是否正常
- [ ] 全面視覺回歸測試

### `@nuxtjs/tailwindcss` v7+ 配置

```typescript
// nuxt.config.ts — v7+ 搭配 Tailwind v4
export default defineNuxtConfig({
  modules: ['@nuxtjs/tailwindcss'],
  tailwindcss: {
    cssPath: '~/assets/css/main.css',
    configPath: undefined,  // v4 不需要 JS 配置檔
    viewer: process.env.NODE_ENV === 'development',
    exposeConfig: false,
    injectPosition: 'first'
  }
})
```

> **檔案參考**：效能最佳化詳見 [`performance-and-seo.md`](./performance-and-seo.md)

---

## Maintenance Plan

### 季度維護排程模板

| 週次 | 焦點 | 項目 |
|------|------|------|
| 第 1 週 | 依賴更新 | 審查合併 Renovate PR、`npm audit` 修復、Node.js LTS 更新 |
| 第 2 週 | 效能審查 | Lighthouse CI 報告、Bundle Size 趨勢、Core Web Vitals |
| 第 3 週 | 程式碼品質 | Tech Debt 處理、測試覆蓋率、TypeScript strict 檢查 |
| 第 4 週 | 文件與知識 | ADR 更新、Onboarding 文件、季度事件回顧 |

### 技術債管理（Tech Debt Register）

使用結構化方式追蹤，存放於 `docs/tech-debt-register.md`：

| ID | 描述 | 影響 | 優先級 | 預估工時 | 狀態 |
|----|------|------|--------|---------|------|
| TD-001 | API 缺少統一錯誤處理 | 維護性 | P1 | 3d | 進行中 |
| TD-002 | 元件缺少 unit test | 品質 | P2 | 5d | 待處理 |

**優先級定義**：
- **P0（緊急）**：影響生產環境穩定性，立即處理
- **P1（高）**：影響開發效率或 UX，當季處理
- **P2（中）**：改善程式碼品質，下一季處理
- **P3（低）**：Nice-to-have，有空處理

> 每個 Sprint 分配 20% 時間消化技術債，P0/P1 優先。

### 升級風險評估 Checklist

執行重大升級前逐項確認：

- [ ] 已閱讀完整 Changelog / Migration Guide
- [ ] 已確認所有 Breaking Changes
- [ ] 已確認第三方依賴相容性
- [ ] 已確認 Node.js / TypeScript 版本需求
- [ ] 單元 / 整合 / E2E / 視覺回歸測試全數通過
- [ ] 效能基準無顯著退化
- [ ] 已準備回滾步驟文件
- [ ] 已確認 `package-lock.json` 可回復
- [ ] Tech Lead 審核通過、已排入部署視窗

### EOL (End of Life) 追蹤

| 技術 | 目前版本 | EOL 日期 | 追蹤連結 |
|------|---------|---------|---------|
| Node.js | 22.x LTS | 2027-04 | [nodejs.org/releases](https://nodejs.org/en/about/releases/) |
| Nuxt | 3.x | TBA | [nuxt.com](https://nuxt.com) |
| Vue | 3.x | 持續維護 | [vuejs.org](https://vuejs.org) |
| TypeScript | 5.x | 持續維護 | [typescriptlang.org](https://www.typescriptlang.org/) |
| Tailwind CSS | v4 | 持續維護 | [tailwindcss.com](https://tailwindcss.com) |

```typescript
// scripts/check-eol.ts — EOL 檢查腳本
import { execSync } from 'node:child_process'

function checkOutdated() {
  let output: string
  try {
    output = execSync('npm outdated --json', { encoding: 'utf-8' })
  } catch (e: any) {
    // npm outdated exits with code 1 when outdated packages exist
    output = e.stdout ?? '{}'
  }
  const outdated = JSON.parse(output || '{}')
  return Object.entries(outdated).map(([name, info]: [string, any]) => ({
    name,
    current: info.current,
    latest: info.latest,
    isOutdated: info.current !== info.latest
  }))
}

const critical = checkOutdated().filter(dep =>
  ['nuxt', 'vue', 'typescript'].includes(dep.name) && dep.isOutdated
)
if (critical.length > 0) {
  critical.forEach(d => console.warn(`  ${d.name}: ${d.current} → ${d.latest}`))
  process.exit(1)
}
```

> **檔案參考**：部署與 CI/CD 詳見 [`server-and-deployment.md`](./server-and-deployment.md)；測試策略詳見 [`testing.md`](./testing.md)
