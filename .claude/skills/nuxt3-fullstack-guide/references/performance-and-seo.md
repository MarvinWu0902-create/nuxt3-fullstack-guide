# 效能最佳化與 SEO

## 目錄

1. [Lazy Loading 策略](#lazy-loading-策略)
2. [Code Splitting](#code-splitting)
3. [圖片最佳化](#圖片最佳化)
4. [Bundle 分析與瘦身](#bundle-分析與瘦身)
5. [Core Web Vitals 最佳化](#core-web-vitals-最佳化)
6. [SEO Meta 設定](#seo-meta-設定)
7. [結構化資料](#結構化資料)
8. [Tailwind CSS 效能](#tailwind-css-效能)
9. [Lighthouse CI 自動化](#lighthouse-ci-自動化)
10. [效能監控（Real User Monitoring）](#效能監控real-user-monitoring)
11. [Bundle 預算管理](#bundle-預算管理)

---

## Lazy Loading 策略

### 元件 Lazy Loading

```vue
<template>
  <!-- Nuxt 自動支援：加上 Lazy 前綴即延遲載入 -->
  <LazyTheFooter />

  <!-- 條件式載入重量級元件 -->
  <LazyDataChart v-if="showChart" :data="chartData" />

  <!-- 搭配 ClientOnly 避免 SSR -->
  <ClientOnly>
    <LazyRichTextEditor v-model="content" />
    <template #fallback>
      <div class="h-64 bg-gray-100 animate-pulse rounded-lg" />
    </template>
  </ClientOnly>
</template>
```

### 路由 Lazy Loading

```typescript
// nuxt.config.ts — 頁面預設就是 lazy loaded
// 但可以針對特定路由設定預載策略
export default defineNuxtConfig({
  experimental: {
    // 啟用 Speculation Rules API，允許預取跨域連結資源
    crossOriginPrefetch: true
  },
  router: {
    options: {
      scrollBehaviorType: 'smooth'
    }
  }
})
```

### 非同步元件（手動控制）

```typescript
// 手動定義非同步元件
const HeavyEditor = defineAsyncComponent({
  loader: () => import('~/components/HeavyEditor.vue'),
  loadingComponent: () => h('div', { class: 'skeleton' }),
  delay: 200,      // 200ms 後才顯示 loading
  timeout: 10000   // 10 秒超時
})
```

---

## Code Splitting

### 路由層級自動分割

Nuxt 3 預設按頁面分割。每個 `pages/` 下的 `.vue` 檔案會產生獨立 chunk。

### 共用模組抽取

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  vite: {
    build: {
      rollupOptions: {
        output: {
          manualChunks(id) {
            // 第三方套件分組
            if (id.includes('node_modules')) {
              if (id.includes('chart.js') || id.includes('d3')) {
                return 'vendor-charts'
              }
              if (id.includes('@headlessui') || id.includes('@heroicons')) {
                return 'vendor-ui'
              }
              return 'vendor'
            }
          }
        }
      }
    }
  }
})
```

### 動態 Import

```typescript
// 僅在需要時載入重量級函式庫
async function generatePDF() {
  const { jsPDF } = await import('jspdf')
  const doc = new jsPDF()
  doc.text('Hello', 10, 10)
  doc.save('output.pdf')
}

// 條件式載入
if (import.meta.client) {
  const module = await import('browser-only-lib')
}
```

---

## 圖片最佳化

### Nuxt Image 模組

```bash
npm install @nuxt/image
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/image'],
  image: {
    quality: 80,
    format: ['webp', 'avif'],
    screens: {
      xs: 320,
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280,
      '2xl': 1536
    },
    // 外部圖片來源
    domains: ['images.unsplash.com', 'cdn.example.com'],
    providers: {
      cloudinary: {
        baseURL: 'https://res.cloudinary.com/your-cloud/image/upload/'
      }
    }
  }
})
```

### 使用 NuxtImg 和 NuxtPicture

```vue
<template>
  <!-- 自動最佳化：格式轉換 + 尺寸調整 + lazy loading -->
  <NuxtImg
    src="/images/hero.jpg"
    alt="主視覺圖"
    width="1200"
    height="600"
    sizes="100vw sm:50vw md:33vw"
    loading="lazy"
    placeholder
    format="webp"
  />

  <!-- 自動產生 <picture> 元素搭配多格式 -->
  <NuxtPicture
    src="/images/product.jpg"
    alt="產品照片"
    sizes="(max-width: 768px) 100vw, 50vw"
    :modifiers="{ roundCorner: '16' }"
  />

  <!-- 背景圖最佳化 -->
  <div
    :style="{
      backgroundImage: `url(${$img('/images/bg.jpg', { width: 1920, quality: 75 })})`
    }"
    class="bg-cover bg-center min-h-screen"
  />
</template>
```

---

## Bundle 分析與瘦身

### 分析工具

```bash
# 產生 bundle 分析報告
npx nuxi analyze

# 或使用 vite-bundle-visualizer
npx vite-bundle-visualizer
```

### 常見瘦身策略

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 1. Tree-shaking 未使用的元件
  components: {
    dirs: [
      {
        path: '~/components',
        pathPrefix: false,
        // 不全域註冊所有元件，按需匯入
        global: false
      }
    ]
  },

  // 2. treeshakeClientOnly 在 Nuxt 3.0+ 已預設啟用，3.12+ 已移除此選項
  // 無需手動設定

  // 3. 壓縮設定
  nitro: {
    compressPublicAssets: true,
    minify: true
  },

  // 4. 外部化大型依賴（SSR 時不打包）
  vite: {
    optimizeDeps: {
      include: ['vue', 'vue-router', 'pinia']
    }
  }
})
```

### 字體最佳化

```typescript
// nuxt.config.ts — 使用 @nuxt/fonts 自動最佳化字體載入
// 注意：@nuxtjs/fontaine 已被棄用，請改用 @nuxt/fonts
// npm install @nuxt/fonts
export default defineNuxtConfig({
  modules: ['@nuxt/fonts'],
  fonts: {
    // 自動偵測 CSS 和 Vue 檔案中使用的字體，從 Google Fonts、Bunny Fonts 等來源下載
    // 預設會自動處理 font-display: swap 和字體 fallback 以減少 CLS
    families: [
      { name: 'Noto Sans TC', provider: 'google' }
    ],
    defaults: {
      weights: [400, 500, 700],
      styles: ['normal']
    }
  }
})
```

```css
/* assets/css/main.css — 字體載入最佳化 */
@font-face {
  font-family: 'CustomFont';
  src: url('/fonts/custom.woff2') format('woff2');
  font-display: swap; /* 避免 FOIT */
  unicode-range: U+0000-00FF; /* 僅載入需要的字元範圍 */
}
```

---

## Core Web Vitals 最佳化

### LCP（Largest Contentful Paint）

```vue
<!-- 最大內容繪製 — 確保主要元素快速載入 -->
<template>
  <!-- 1. 預載主要圖片 -->
  <NuxtImg
    src="/images/hero.jpg"
    alt="主視覺"
    loading="eager"
    fetchpriority="high"
    preload
  />
</template>

<script setup lang="ts">
// 2. 避免在首屏使用 lazy 元件
// 3. 減少 render-blocking CSS/JS

// 4. 預連結重要的外部資源
useHead({
  link: [
    { rel: 'preconnect', href: 'https://fonts.googleapis.com' },
    { rel: 'dns-prefetch', href: 'https://api.example.com' }
  ]
})
</script>
```

### CLS（Cumulative Layout Shift）

```vue
<!-- 避免版面位移 -->
<template>
  <!-- 1. 圖片永遠指定寬高 -->
  <NuxtImg src="/photo.jpg" width="400" height="300" />

  <!-- 2. 骨架畫面佔位 -->
  <div v-if="status === 'pending'" class="h-48 w-full bg-gray-200 animate-pulse rounded" />
  <div v-else>{{ data }}</div>

  <!-- 3. 廣告/動態內容預留空間 -->
  <div class="min-h-[250px]">
    <LazyAdBanner />
  </div>
</template>
```

### INP（Interaction to Next Paint）

```typescript
// 避免長任務阻塞主執行緒
// 使用 requestIdleCallback 延遲非關鍵工作
function heavyComputation(items: unknown[]) {
  if (typeof requestIdleCallback !== 'undefined') {
    requestIdleCallback(() => {
      // 非關鍵計算
      processItems(items)
    })
  } else {
    setTimeout(() => processItems(items), 0)
  }
}

// 使用 Web Worker 處理 CPU 密集任務
// nuxt.config.ts
export default defineNuxtConfig({
  vite: {
    worker: {
      format: 'es'
    }
  }
})
```

---

## SEO Meta 設定

### useHead vs useSeoMeta 選擇

| 用途 | 推薦 | 說明 |
|------|------|------|
| SEO meta 標籤（title、description、OG） | `useSeoMeta()` | 型別安全、扁平結構、避免重複 |
| 非 SEO 用途（script、link、style、noscript） | `useHead()` | 完整的 head 管理 |
| 兩者混用 | ✅ 可以 | 各自管理不同職責 |

```typescript
// ✅ SEO 標籤用 useSeoMeta（推薦）
useSeoMeta({
  title: '頁面標題',
  description: '頁面描述',
  ogTitle: '頁面標題',
  ogImage: '/images/og.jpg'
})

// ✅ 非 SEO 用途用 useHead
useHead({
  script: [{ src: 'https://analytics.example.com/script.js', defer: true }],
  link: [{ rel: 'canonical', href: 'https://example.com/page' }]
})
```

### 全站 SEO 預設值

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      htmlAttrs: { lang: 'zh-TW' },
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
      titleTemplate: '%s | 網站名稱'
    }
  }
})
```

### 頁面級 SEO

```vue
<script setup lang="ts">
// 使用 useSeoMeta — 型別安全且簡潔
useSeoMeta({
  title: '產品列表',
  description: '瀏覽我們的優質產品系列，提供最佳價格與品質保證。',
  ogTitle: '產品列表 | 品牌名稱',
  ogDescription: '瀏覽我們的優質產品系列',
  ogImage: 'https://example.com/og-products.jpg',
  ogType: 'website',
  ogLocale: 'zh_TW',
  twitterCard: 'summary_large_image',
  twitterTitle: '產品列表',
  twitterDescription: '瀏覽我們的優質產品系列',
  twitterImage: 'https://example.com/og-products.jpg',
  robots: 'index, follow'
})

// 動態 SEO（依資料變化）
const { data: product } = await useFetch<Product>(`/api/products/${route.params.id}`)

useHead({
  title: () => product.value?.name ?? '載入中...',
  meta: [
    { name: 'description', content: () => product.value?.description ?? '' }
  ],
  link: [
    { rel: 'canonical', href: `https://example.com/products/${route.params.id}` }
  ]
})
</script>
```

### Sitemap

```bash
npm install @nuxtjs/sitemap
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/sitemap'],
  site: {
    url: 'https://example.com'
  },
  sitemap: {
    // 動態路由來源
    sources: ['/api/__sitemap__/urls']
  }
})
```

```typescript
// server/api/__sitemap__/urls.ts
import { defineSitemapEventHandler } from '#imports'

export default defineSitemapEventHandler(async () => {
  const products = await db.product.findMany({ select: { slug: true, updatedAt: true } })
  return products.map((p) => ({
    loc: `/products/${p.slug}`,
    lastmod: p.updatedAt
  }))
})
```

---

## 結構化資料

### JSON-LD

```vue
<script setup lang="ts">
// 產品頁結構化資料
const { data: product } = await useFetch<Product>(`/api/products/${route.params.id}`)

// 使用函式形式讓 JSON-LD 在資料變化時自動更新
useHead(() => ({
  script: product.value ? [
    {
      type: 'application/ld+json',
      innerHTML: JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'Product',
        name: product.value.name,
        description: product.value.description,
        image: product.value.imageUrl,
        offers: {
          '@type': 'Offer',
          price: product.value.price,
          priceCurrency: 'TWD',
          availability: 'https://schema.org/InStock'
        }
      })
    }
  ] : []
}))

// 麵包屑結構化資料（使用函式形式確保 product.value 變化時自動更新）
useHead(() => ({
  script: [
    {
      type: 'application/ld+json',
      innerHTML: JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'BreadcrumbList',
        itemListElement: [
          { '@type': 'ListItem', position: 1, name: '首頁', item: 'https://example.com' },
          { '@type': 'ListItem', position: 2, name: '產品', item: 'https://example.com/products' },
          { '@type': 'ListItem', position: 3, name: product.value?.name }
        ]
      })
    }
  ]
}))
</script>
```

---

## Tailwind CSS 效能

### PurgeCSS（內建）

Tailwind CSS 自動 tree-shake 未使用的 class。

- **使用 `@nuxtjs/tailwindcss` 模組時**：content 路徑已自動配置（掃描 `components/`、`layouts/`、`pages/` 等），無需手動設定。
- **Tailwind v3 不搭配模組時**：需手動配置 content 路徑。
- **Tailwind v4**：改為自動偵測，不需要 content 配置。

```typescript
// tailwind.config.ts（僅在不使用 @nuxtjs/tailwindcss 模組時才需要手動配置 content）
import type { Config } from 'tailwindcss'

export default {
  content: [
    './components/**/*.{vue,ts}',
    './layouts/**/*.vue',
    './pages/**/*.vue',
    './composables/**/*.ts',
    './plugins/**/*.ts',
    './utils/**/*.ts',
    './app.vue',
    './error.vue'
  ]
} satisfies Config
```

### 避免動態 class 被 purge

```vue
<!-- 錯誤：動態拼接的 class 會被 purge 掉 -->
<div :class="`text-${color}-500`" />

<!-- 正確：使用完整的 class 名稱 -->
<div :class="{
  'text-red-500': color === 'red',
  'text-blue-500': color === 'blue',
  'text-green-500': color === 'green'
}" />

<!-- 正確：使用 safelist -->
<!-- tailwind.config.ts 中加入 safelist -->
```

```typescript
// tailwind.config.ts
export default {
  safelist: [
    // 確保動態使用的 class 不被移除
    { pattern: /^bg-(red|blue|green)-(100|500|900)$/ },
    { pattern: /^text-(red|blue|green)-(100|500|900)$/ }
  ]
}
```

---

## Lighthouse CI 自動化

### 配置檔案

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/products'],
      startServerCommand: 'node .output/server/index.mjs',
      startServerReadyPattern: 'Listening on',
      numberOfRuns: 3
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'categories:best-practices': ['error', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],
        // 個別指標
        'first-contentful-paint': ['warn', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }]
      }
    },
    upload: {
      target: 'temporary-public-storage'
    }
  }
}
```

### GitHub Actions 整合

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI
on: [pull_request]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v12
        with:
          configPath: './lighthouserc.js'
          uploadArtifacts: true
          temporaryPublicStorage: true
```

建議在 PR 中設定 Lighthouse CI 作為必要檢查，確保每次合併前效能分數達標。

---

## 效能監控（Real User Monitoring）

### Web Vitals 收集

```typescript
// plugins/web-vitals.client.ts
// 僅在客戶端執行的效能監控外掛
export default defineNuxtPlugin(() => {
  // 使用 PerformanceObserver 收集 Core Web Vitals
  if (typeof PerformanceObserver === 'undefined') return

  function sendMetric(name: string, value: number) {
    // 發送到分析服務（可替換為 Google Analytics、自建 API 等）
    if (navigator.sendBeacon) {
      navigator.sendBeacon('/api/metrics', JSON.stringify({
        name,
        value: Math.round(value),
        path: window.location.pathname,
        timestamp: Date.now()
      }))
    }
  }

  // LCP
  const lcpObserver = new PerformanceObserver((list) => {
    const entries = list.getEntries()
    const lastEntry = entries[entries.length - 1] as PerformancePaintTiming
    sendMetric('LCP', lastEntry.startTime)
  })
  lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true })

  // FID / INP
  const fidObserver = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      const eventEntry = entry as PerformanceEventTiming
      sendMetric('INP', eventEntry.processingStart - eventEntry.startTime)
    }
  })
  fidObserver.observe({ type: 'event', buffered: true })

  // CLS
  let clsValue = 0
  const clsObserver = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      const layoutShift = entry as PerformanceEntry & { hadRecentInput: boolean; value: number }
      if (!layoutShift.hadRecentInput) {
        clsValue += layoutShift.value
      }
    }
  })
  clsObserver.observe({ type: 'layout-shift', buffered: true })

  // 頁面離開時發送 CLS
  document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'hidden') {
      sendMetric('CLS', clsValue)
    }
  })
})
```

### 伺服器端指標收集

```typescript
// server/api/metrics.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody<{
    name: string
    value: number
    path: string
    timestamp: number
  }>(event)

  // 記錄到日誌或資料庫
  console.log(`[RUM] ${body.name}: ${body.value}ms @ ${body.path}`)

  // 實際專案中，寫入時序資料庫（如 InfluxDB）或分析服務
  return { ok: true }
})
```

### 搭配 web-vitals 套件（替代方案）

```typescript
// plugins/web-vitals.client.ts
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals'

export default defineNuxtPlugin(() => {
  function sendMetric(metric: { name: string; value: number }) {
    navigator.sendBeacon?.('/api/metrics', JSON.stringify({
      name: metric.name,
      value: Math.round(metric.value),
      path: window.location.pathname,
      timestamp: Date.now()
    }))
  }

  onCLS(sendMetric)
  onINP(sendMetric)
  onLCP(sendMetric)
  onFCP(sendMetric)
  onTTFB(sendMetric)
})
```

Note: `web-vitals` 套件提供更準確的指標計算，建議在正式環境中使用。

---

## Bundle 預算管理

### Vite 建置預算

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  vite: {
    build: {
      // chunk 大小警告閾值（KB）
      chunkSizeWarningLimit: 500,
      rollupOptions: {
        output: {
          // 控制 chunk 分割
          experimentalMinChunkSize: 10_000
        }
      }
    }
  }
})
```

### 解讀 nuxi analyze 報告

```bash
# 執行分析
npx nuxi analyze

# 報告會自動在瀏覽器中開啟，重點關注：
# 1. 總 bundle 大小（Initial JS 建議 < 200KB gzip）
# 2. 最大 chunk 大小（單一 chunk 建議 < 100KB gzip）
# 3. 重複依賴（同一套件出現在多個 chunk）
# 4. 未使用的依賴（tree-shaking 未生效）
```

### CI 中的 Bundle 大小檢查

```yaml
# .github/workflows/bundle-check.yml
name: Bundle Size Check
on: [pull_request]

jobs:
  check-size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - name: Check bundle size
        run: |
          # 檢查 .output/public 目錄的 JS 總大小
          TOTAL_JS=$(find .output/public/_nuxt -name '*.js' -exec wc -c {} + | tail -1 | awk '{print $1}')
          MAX_SIZE=1048576  # 1MB (未壓縮)
          echo "Total JS size: $TOTAL_JS bytes"
          if [ "$TOTAL_JS" -gt "$MAX_SIZE" ]; then
            echo "::error::Bundle size ($TOTAL_JS bytes) exceeds budget ($MAX_SIZE bytes)"
            exit 1
          fi
```

### 常見瘦身技巧

```typescript
// 1. 檢查哪些套件佔用最多空間
// 執行 npx nuxi analyze 後，在 treemap 中找出最大的模組

// 2. 動態匯入大型套件
// 避免：import dayjs from 'dayjs'（全量匯入）
// 改用：const dayjs = await import('dayjs')（按需載入）

// 3. 替換肥大套件
// moment.js (300KB) → dayjs (2KB)
// lodash (70KB) → lodash-es（支援 tree-shaking）或原生方法

// 4. 分析 node_modules 中的隱藏依賴
// npx depcheck（找出未使用的依賴）
```
