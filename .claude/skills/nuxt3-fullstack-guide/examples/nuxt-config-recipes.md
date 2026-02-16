# 實戰範例：nuxt.config.ts 常用配方

## 概述

常見場景的 `nuxt.config.ts` 配置範例，可直接複製使用。

---

## 完整生產環境配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // === 基礎 ===
  devtools: { enabled: true },
  typescript: {
    strict: true,
    typeCheck: true
  },

  // === 模組 ===
  modules: [
    '@nuxtjs/tailwindcss',
    '@pinia/nuxt',
    '@nuxt/image',
    '@vueuse/nuxt',
    '@nuxtjs/color-mode',
    '@nuxt/eslint',
    '@nuxt/test-utils/module'
  ],

  // === App 設定 ===
  app: {
    head: {
      htmlAttrs: { lang: 'zh-TW' },
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
      titleTemplate: '%s | 品牌名稱',
      meta: [
        { name: 'theme-color', content: '#0c8ce9' },
        { name: 'format-detection', content: 'telephone=no' }
      ],
      link: [
        { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' },
        { rel: 'preconnect', href: 'https://fonts.googleapis.com' }
      ]
    },
    pageTransition: { name: 'page', mode: 'out-in' },
    layoutTransition: { name: 'layout', mode: 'out-in' }
  },

  // === CSS ===
  css: ['~/assets/css/main.css'],

  // === Runtime Config ===
  runtimeConfig: {
    jwtSecret: '',  // 由 NUXT_JWT_SECRET 環境變數覆蓋
    dbUrl: '',      // 由 NUXT_DB_URL 環境變數覆蓋
    // NUXT_PUBLIC_ 前綴環境變數會自動覆蓋 public 中的對應欄位
    // 此處只需設定預設值，不需要手動使用 process.env
    public: {
      apiBase: 'http://localhost:3000',   // 由 NUXT_PUBLIC_API_BASE 覆蓋
      appName: '我的應用',                // 由 NUXT_PUBLIC_APP_NAME 覆蓋
      siteUrl: 'http://localhost:3000'    // 由 NUXT_PUBLIC_SITE_URL 覆蓋
    }
  },

  // === 路由規則（混合渲染）===
  routeRules: {
    '/': { prerender: true },
    '/blog/**': { isr: 3600 },
    '/dashboard/**': { ssr: false },
    // 注意：cors: true 預設為 Allow-Origin: *，生產環境應改用 server middleware 設定明確的 origin
    '/api/**': { cors: true }
  },

  // === Nitro ===
  nitro: {
    compressPublicAssets: true,
    minify: true,
    // 安全標頭（也可合併到頂層 routeRules 中）
    routeRules: {
      '/**': {
        headers: {
          'X-Content-Type-Options': 'nosniff',
          'X-Frame-Options': 'SAMEORIGIN',
          'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
          'Referrer-Policy': 'strict-origin-when-cross-origin',
          'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
          // ⚠ 不建議使用 'unsafe-inline'，請改用下方 nonce-based CSP 配置
          // 此處僅作為不支援 nuxt-security 模組之環境的最低限度 fallback
          'Content-Security-Policy': "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data: https:; font-src 'self' https://fonts.gstatic.com; connect-src 'self'"
        }
      }
    }
  },

  // === 圖片 ===
  image: {
    quality: 80,
    format: ['webp', 'avif']
  },

  // === 色彩模式 ===
  colorMode: {
    classSuffix: '',
    preference: 'system',
    fallback: 'light'
  },

  // === 實驗性功能（以下在 Nuxt 3.12+ 已為預設值）===
  // experimental: {
  //   payloadExtraction: true,
  //   renderJsonPayloads: true
  // },

  // === Vite ===
  vite: {
    css: {
      preprocessorOptions: {
        scss: {
          additionalData: '@use "~/assets/css/variables" as *;'
        }
      }
    }
  }
})
```

## Nonce-Based CSP 配置（推薦）

> **為什麼不用 `'unsafe-inline'`？** `'unsafe-inline'` 允許任何行內腳本執行，等於完全關閉 CSP 對 XSS 的防護。使用 nonce-based CSP 時，只有攜帶正確隨機 nonce 的 `<script>` 標籤才會被執行。

### 使用 `nuxt-security` 模組（推薦方式）

```bash
npx nuxi module add nuxt-security
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-security'],

  security: {
    // nonce 會自動注入到所有 Nuxt 渲染的 <script> 與 <link> 標籤
    nonce: true,

    headers: {
      // nuxt-security 會自動將 nonce 值替換到 CSP 中
      contentSecurityPolicy: {
        'default-src': ["'self'"],
        'script-src': [
          "'self'",
          "'nonce-{{nonce}}'",  // nuxt-security 自動替換為每次請求的隨機 nonce
          "'strict-dynamic'"   // 允許被信任腳本動態載入的子腳本
        ],
        'style-src': [
          "'self'",
          "'nonce-{{nonce}}'"
        ],
        'img-src': ["'self'", 'data:', 'https:'],
        'font-src': ["'self'", 'https://fonts.gstatic.com'],
        'connect-src': ["'self'"],
        'object-src': ["'none'"],
        'base-uri': ["'self'"],
        'form-action': ["'self'"]
      },
      // 其他安全標頭
      crossOriginEmbedderPolicy: 'require-corp',
      crossOriginOpenerPolicy: 'same-origin',
      xContentTypeOptions: 'nosniff',
      xFrameOptions: 'SAMEORIGIN',
      referrerPolicy: 'strict-origin-when-cross-origin'
    }
  }
})
```

### 手動實作 Nonce-Based CSP（不使用模組）

若無法使用 `nuxt-security` 模組（例如 Edge Runtime 或特殊部署環境），可手動實作：

```typescript
// server/middleware/csp-nonce.ts
import { randomBytes } from 'node:crypto'

export default defineEventHandler((event) => {
  // 為每次請求產生唯一的 nonce（128-bit 隨機值）
  const nonce = randomBytes(16).toString('base64')
  // 存入 event context，供模板渲染使用
  event.context.cspNonce = nonce

  // 僅對 HTML 回應設定 CSP 標頭
  if (getRequestURL(event).pathname.startsWith('/api/')) return

  setHeader(event, 'Content-Security-Policy', [
    "default-src 'self'",
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`,
    `style-src 'self' 'nonce-${nonce}'`,
    "img-src 'self' data: https:",
    "font-src 'self' https://fonts.gstatic.com",
    "connect-src 'self'",
    "object-src 'none'",
    "base-uri 'self'"
  ].join('; '))
})
```

```typescript
// plugins/csp-nonce.server.ts
// 將 nonce 注入到 Nuxt 渲染的 HTML 中
export default defineNuxtPlugin((nuxtApp) => {
  const event = useRequestEvent()
  if (!event?.context.cspNonce) return

  const nonce = event.context.cspNonce as string

  useHead({
    script: [],
    // 為所有由 Nuxt 注入的 script/style 標籤設定 nonce 屬性
    noscript: []
  })

  // 透過 Nitro 的 render:html hook 為所有 <script> 和 <style> 標籤加上 nonce
  nuxtApp.hook('app:rendered', () => {
    event.context._nitro = event.context._nitro || {}
    event.context._nitro.cspNonce = nonce
  })
})
```

> **注意**：`'strict-dynamic'` 讓被信任的腳本可以動態載入子腳本（如 Nuxt 的 chunk），不需逐一列出所有 CDN 網址。此指令在 CSP Level 3 中支援，覆蓋主流瀏覽器。若需支援 IE11，請改用 hash-based 方式。

## 國際化（i18n）配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/i18n'],
  i18n: {
    locales: [
      { code: 'zh-TW', name: '繁體中文', file: 'zh-TW.json' },
      { code: 'en', name: 'English', file: 'en.json' }
    ],
    defaultLocale: 'zh-TW',
    lazy: true,
    langDir: 'locales/',
    strategy: 'prefix_except_default',
    detectBrowserLanguage: {
      useCookie: true,
      cookieKey: 'i18n_locale',
      redirectOn: 'root'
    }
  }
})
```

### i18n 實際使用模式

```vue
<!-- 在元件中使用翻譯 -->
<script setup lang="ts">
const { t, locale, locales, setLocale } = useI18n()

// 動態切換語系
async function switchLang(code: string) {
  await setLocale(code)
}
</script>

<template>
  <div>
    <h1>{{ t('welcome') }}</h1>
    <p>{{ t('greeting', { name: '使用者' }) }}</p>

    <!-- 語系切換器 -->
    <select @change="switchLang(($event.target as HTMLSelectElement).value)">
      <option
        v-for="loc in locales"
        :key="loc.code"
        :value="loc.code"
        :selected="loc.code === locale"
      >
        {{ loc.name }}
      </option>
    </select>
  </div>
</template>
```

```typescript
// i18n/locales/zh-TW.json
{
  "welcome": "歡迎來到我們的網站",
  "greeting": "你好，{name}！",
  "nav": {
    "home": "首頁",
    "products": "產品",
    "about": "關於我們"
  },
  "product": {
    "price": "價格：{price} 元",
    "count": "沒有商品 | {n} 件商品 | {n} 件商品"
  }
}
```

### i18n SEO 整合

```vue
<script setup lang="ts">
// 自動設定 lang 屬性與 hreflang 標籤
const head = useLocaleHead({
  addDirAttribute: true,
  addSeoAttributes: true
})

useHead({
  htmlAttrs: { lang: head.value.htmlAttrs!.lang },
  link: head.value.link,
  meta: head.value.meta
})
</script>
```

```typescript
// nuxt.config.ts — i18n SEO 策略
export default defineNuxtConfig({
  i18n: {
    strategy: 'prefix_except_default',  // /en/about, /about（預設中文）
    defaultLocale: 'zh-TW',
    detectBrowserLanguage: {
      useCookie: true,
      cookieKey: 'i18n_lang',
      redirectOn: 'root'  // 僅在根路徑偵測
    },
    // Lazy-loaded 翻譯檔（減少初始 bundle 大小）
    lazy: true,
    langDir: 'i18n/locales',
    locales: [
      { code: 'zh-TW', name: '繁體中文', file: 'zh-TW.json' },
      { code: 'en', name: 'English', file: 'en.json' },
      { code: 'ja', name: '日本語', file: 'ja.json' }
    ]
  }
})
```

## Monorepo 配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  extends: [
    './layers/base',        // 共用 UI 元件與樣式
    './layers/auth'         // 認證功能
  ],
  // Monorepo 中的套件解析
  alias: {
    '@shared': '../packages/shared/src',
    '@ui': '../packages/ui/src'
  },
  // 確保 monorepo 中的套件被正確轉譯
  build: {
    transpile: ['@company/shared', '@company/ui']
  }
})
```

## Nuxt Content 整合

```bash
npx nuxi module add @nuxt/content
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/content'],
  content: {
    highlight: {
      // 程式碼高亮（使用 Shiki）
      theme: 'github-dark',
      langs: ['typescript', 'vue', 'bash', 'json']
    },
    markdown: {
      toc: { depth: 3 }
    }
  }
})
```

```
// content/ 目錄結構
content/
├── blog/
│   ├── first-post.md
│   └── second-post.md
├── docs/
│   ├── 1.getting-started/
│   │   ├── 1.introduction.md
│   │   └── 2.installation.md
│   └── 2.guide/
│       └── 1.configuration.md
└── index.md
```

```markdown
<!-- content/blog/first-post.md -->
---
title: '第一篇文章'
description: '這是部落格的第一篇文章'
date: 2024-01-15
tags: ['nuxt', 'vue']
image: '/images/blog/first.jpg'
---

# {{ $doc.title }}

這是 **Markdown** 內容，支援 Vue 元件：

::alert{type="info"}
這是一個自訂元件（MDC 語法）
::

```ts
const hello = 'world'
```
```

```vue
<!-- pages/blog/[...slug].vue — 單篇文章頁面 -->
<script setup lang="ts">
const route = useRoute()

// 使用 useSeoMeta 設定 SEO
const { data: article } = await useAsyncData(
  `blog-${route.path}`,
  () => queryContent(route.path).findOne()
)

if (!article.value) {
  throw createError({ statusCode: 404, statusMessage: '文章不存在' })
}

useSeoMeta({
  title: article.value.title,
  description: article.value.description,
  ogImage: article.value.image
})
</script>

<template>
  <article class="prose prose-lg mx-auto">
    <ContentRenderer :value="article!" />
  </article>
</template>
```

```vue
<!-- pages/blog/index.vue — 文章列表 -->
<script setup lang="ts">
const { data: articles } = await useAsyncData('blog-list',
  () => queryContent('blog')
    .sort({ date: -1 })
    .only(['_path', 'title', 'description', 'date', 'tags'])
    .find()
)
</script>

<template>
  <div>
    <h1>部落格</h1>
    <ul>
      <li v-for="article in articles" :key="article._path">
        <NuxtLink :to="article._path">
          <h2>{{ article.title }}</h2>
          <p>{{ article.description }}</p>
          <time>{{ article.date }}</time>
        </NuxtLink>
      </li>
    </ul>
  </div>
</template>
```

## PWA 配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@vite-pwa/nuxt'],
  pwa: {
    registerType: 'autoUpdate',
    manifest: {
      name: '我的應用',
      short_name: 'MyApp',
      theme_color: '#0c8ce9',
      background_color: '#ffffff',
      display: 'standalone',
      lang: 'zh-TW',
      icons: [
        { src: '/icons/icon-192.png', sizes: '192x192', type: 'image/png' },
        { src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png' }
      ]
    },
    workbox: {
      navigateFallback: '/',
      globPatterns: ['**/*.{js,css,html,png,svg,ico}']
    }
  }
})
```

## Vitest 整合配置

```typescript
// vitest.config.ts — 獨立設定檔
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    environment: 'nuxt',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
    include: ['tests/**/*.{test,spec}.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json-summary', 'html'],
      include: ['components/**', 'composables/**', 'stores/**', 'utils/**', 'server/**'],
      exclude: ['**/*.d.ts', 'tests/**', '.nuxt/**'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80
      }
    },
    // 測試超時
    testTimeout: 10000,
    hookTimeout: 10000
  }
})
```

## 頁面過渡動畫 CSS

```css
/* assets/css/main.css 中加入 */

/* 頁面過渡 */
.page-enter-active,
.page-leave-active {
  transition: all 0.3s ease;
}
.page-enter-from {
  opacity: 0;
  transform: translateY(8px);
}
.page-leave-to {
  opacity: 0;
  transform: translateY(-8px);
}

/* 佈局過渡 */
.layout-enter-active,
.layout-leave-active {
  transition: all 0.4s ease;
}
.layout-enter-from,
.layout-leave-to {
  opacity: 0;
  filter: blur(4px);
}
```

## ESLint 整合

### 安裝

```bash
# @nuxt/eslint 模組（推薦，整合 Flat Config）
npx nuxi module add @nuxt/eslint
```

> `@nuxt/eslint` 使用 ESLint v9 Flat Config 格式，取代了舊版 `@nuxtjs/eslint-module` + `.eslintrc`。

### nuxt.config.ts 設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/eslint'],
  eslint: {
    config: {
      stylistic: true  // 啟用 ESLint Stylistic 格式化規則（取代 Prettier）
      // stylistic: {
      //   indent: 2,
      //   quotes: 'single',
      //   semi: false,
      //   commaDangle: 'never',
      //   braceStyle: '1tbs'
      // }
    }
  }
})
```

### eslint.config.mjs — 完整生產配置

```javascript
// eslint.config.mjs
import withNuxt from './.nuxt/eslint.config.mjs'

export default withNuxt(
  // 全域規則
  {
    rules: {
      // --- Vue ---
      'vue/multi-word-component-names': 'off',     // Nuxt pages 通常是單字（index.vue）
      'vue/no-multiple-template-root': 'off',      // Vue 3 支援多根節點
      'vue/block-order': ['error', {
        order: ['script', 'template', 'style']      // <script setup> 放最上面
      }],
      'vue/define-macros-order': ['error', {
        order: ['defineProps', 'defineEmits', 'defineModel', 'defineSlots']
      }],
      'vue/no-unused-refs': 'error',
      'vue/no-useless-v-bind': 'error',
      'vue/prefer-true-attribute-shorthand': 'error',

      // --- TypeScript ---
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',
        fixStyle: 'inline-type-imports'
      }],
      '@typescript-eslint/no-import-type-side-effects': 'error',

      // --- 一般 ---
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'prefer-const': 'error'
    }
  },

  // server/ 目錄專用規則
  {
    files: ['server/**/*.ts'],
    rules: {
      'no-console': 'off'  // 伺服器端允許 console
    }
  },

  // 測試檔案專用規則
  {
    files: ['tests/**/*.{test,spec}.ts'],
    rules: {
      '@typescript-eslint/no-explicit-any': 'off',
      'no-console': 'off'
    }
  }
)
```

### ESLint Stylistic vs Prettier

```
選擇格式化工具？
├── 全用 ESLint → 啟用 stylistic: true（推薦，少一個工具）
├── 全用 Prettier → stylistic: false + 安裝 eslint-config-prettier
└── 混用（不推薦）→ 規則衝突風險高
```

若選擇 Prettier 而非 Stylistic：

```javascript
// eslint.config.mjs
import withNuxt from './.nuxt/eslint.config.mjs'
import prettierConfig from 'eslint-config-prettier'

export default withNuxt(
  prettierConfig,  // 關閉與 Prettier 衝突的規則
  {
    rules: {
      // 僅保留邏輯規則，格式由 Prettier 處理
    }
  }
)
```

### VS Code 整合

```jsonc
// .vscode/settings.json
{
  // ESLint Flat Config 支援
  "eslint.useFlatConfig": true,

  // 儲存時自動修正
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },

  // 若使用 Stylistic，停用其他格式化器避免衝突
  "editor.formatOnSave": false,

  // ESLint 驗證的檔案類型
  "eslint.validate": [
    "javascript",
    "typescript",
    "vue"
  ]
}
```

### package.json scripts

```jsonc
// package.json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  }
}
```

### 常見自訂規則場景

```javascript
// eslint.config.mjs — 依專案需求追加

export default withNuxt(
  // 強制 composables 命名慣例
  {
    files: ['composables/**/*.ts'],
    rules: {
      // 確保 composable 檔案匯出 use 前綴函式
    }
  },

  // pages 目錄：允許預設匯出
  {
    files: ['pages/**/*.vue', 'layouts/**/*.vue'],
    rules: {
      'vue/multi-word-component-names': 'off'
    }
  },

  // 忽略自動產生的檔案
  {
    ignores: [
      '.nuxt/**',
      '.output/**',
      'node_modules/**',
      'dist/**',
      '*.d.ts'
    ]
  }
)
```

### CI/CD 整合

```yaml
# .github/workflows/lint.yml
name: Lint
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx nuxi prepare  # 產生 .nuxt/eslint.config.mjs
      - run: npm run lint
```

> **重要**：`@nuxt/eslint` 的配置檔由 `.nuxt/eslint.config.mjs` 自動產生，CI 中必須先執行 `nuxi prepare`。
