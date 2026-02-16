# 實戰範例：Tailwind 設計系統

## 目錄

1. [設計 Token 配置（Tailwind v4）](#設計-token-配置tailwind-v4)
2. [CSS 變數整合（Tailwind v4）](#css-變數整合tailwind-v4)
3. [設計 Token 配置（Tailwind v3 相容）](#設計-token-配置tailwind-v3-相容)
4. [CSS 變數整合（Tailwind v3 相容）](#css-變數整合tailwind-v3-相容)
5. [Vue Transition 搭配 Tailwind](#vue-transition-搭配-tailwind)
6. [響應式佈局元件](#響應式佈局元件)
7. [深色模式切換](#深色模式切換)
8. [從 Tailwind v3 遷移至 v4](#從-tailwind-v3-遷移至-v4)
9. [動畫減少偏好防護（Motion-Reduce Guards）](#動畫減少偏好防護motion-reduce-guards)
10. [圖片格式協商與 `<picture>` 元素](#圖片格式協商與-picture-元素picture-element-with-format-negotiation)
11. [View Transitions API（頁面視圖轉場）](#view-transitions-api頁面視圖轉場)
12. [Tailwind v4 `@source` 指令](#tailwind-v4-source-指令content-source-detection)

## 概述

使用 Tailwind CSS 建立一致且可擴展的設計系統，包含設計 Token、基礎元件、佈局系統、深色模式、動效系統。

> **版本說明**：本文件以 **Tailwind CSS v4** 為主要推薦版本。v4 採用 CSS-first 配置，使用 `@import "tailwindcss"` 和 `@theme` 指令取代 JavaScript 設定檔。既有 v3 專案的配置方式保留在「v3 相容」章節中供參考。

---

## 設計 Token 配置（Tailwind v4）

Tailwind v4 採用 CSS-first 配置，所有設計 Token 都在 CSS 檔案中使用 `@theme` 指令定義，不再需要 `tailwind.config.ts`。

```css
/* assets/css/main.css — Tailwind CSS v4 */
@import "tailwindcss";

/* ===== 設計 Token（取代 tailwind.config.ts 的 theme.extend）===== */
@theme {
  /* 品牌色彩系統 */
  --color-brand-50:  #f0f7ff;
  --color-brand-100: #e0effe;
  --color-brand-200: #b9dffd;
  --color-brand-300: #7cc4fc;
  --color-brand-400: #36a6f8;
  --color-brand-500: #0c8ce9;  /* 主色 */
  --color-brand-600: #006fc7;
  --color-brand-700: #0058a1;
  --color-brand-800: #054b85;
  --color-brand-900: #0a3f6e;
  --color-brand-950: #072849;

  --color-surface: #ffffff;
  --color-surface-secondary: #f8fafc;
  --color-surface-tertiary: #f1f5f9;
  --color-surface-dark: #0f172a;
  --color-surface-dark-secondary: #1e293b;
  --color-surface-dark-tertiary: #334155;

  /* 字體 */
  --font-sans: 'Noto Sans TC', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', ui-monospace, monospace;
  --font-display: 'Noto Serif TC', serif;

  /* 流體排版 */
  --text-fluid-xs: clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem);
  --text-fluid-sm: clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  --text-fluid-base: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  --text-fluid-lg: clamp(1.125rem, 1rem + 0.625vw, 1.25rem);
  --text-fluid-xl: clamp(1.25rem, 1rem + 1.25vw, 1.5rem);
  --text-fluid-2xl: clamp(1.5rem, 1rem + 2.5vw, 2rem);
  --text-fluid-3xl: clamp(1.875rem, 1rem + 4.375vw, 3rem);
  --text-fluid-4xl: clamp(2.25rem, 1rem + 6.25vw, 3.75rem);

  /* 間距 */
  --spacing-4\.5: 1.125rem;
  --spacing-18: 4.5rem;
  --spacing-88: 22rem;
  --spacing-128: 32rem;

  /* 圓角 */
  --radius-4xl: 2rem;
  --radius-5xl: 2.5rem;

  /* 陰影 */
  --shadow-soft: 0 2px 15px -3px rgba(0, 0, 0, 0.07), 0 10px 20px -2px rgba(0, 0, 0, 0.04);
  --shadow-soft-lg: 0 10px 40px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
  --shadow-inner-soft: inset 0 2px 4px 0 rgba(0, 0, 0, 0.04);
  --shadow-glow: 0 0 15px rgba(12, 140, 233, 0.3);
  --shadow-glow-lg: 0 0 30px rgba(12, 140, 233, 0.4);

  /* 動畫 */
  --animate-fade-in: fade-in 0.5s ease-out;
  --animate-fade-up: fade-up 0.5s ease-out;
  --animate-slide-in-right: slide-in-right 0.3s ease-out;
  --animate-scale-in: scale-in 0.2s ease-out;
  --animate-shimmer: shimmer 2s linear infinite;

  /* 過渡曲線 */
  --ease-bounce-in: cubic-bezier(0.68, -0.55, 0.265, 1.55);
  --ease-smooth: cubic-bezier(0.4, 0, 0.2, 1);
}

/* Keyframes */
@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
@keyframes fade-up {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}
@keyframes slide-in-right {
  from { opacity: 0; transform: translateX(20px); }
  to { opacity: 1; transform: translateX(0); }
}
@keyframes scale-in {
  from { opacity: 0; transform: scale(0.95); }
  to { opacity: 1; transform: scale(1); }
}
@keyframes shimmer {
  from { transform: translateX(-100%); }
  to { transform: translateX(100%); }
}
```

### Nuxt 整合方式（v4）

```typescript
// nuxt.config.ts — 方式 1：使用 @nuxtjs/tailwindcss v7+（推薦）
export default defineNuxtConfig({
  modules: ['@nuxtjs/tailwindcss'],
  // v4 不需要 tailwind.config.ts，模組會自動偵測 CSS 中的 @theme
})

// nuxt.config.ts — 方式 2：使用 @tailwindcss/vite 外掛
// npm install tailwindcss @tailwindcss/vite
import tailwindcss from '@tailwindcss/vite'

export default defineNuxtConfig({
  vite: { plugins: [tailwindcss()] },
  css: ['~/assets/css/main.css']
})
```

### 官方外掛（v4 語法）

```css
/* v4 的外掛引入方式改為 CSS @plugin 指令 */
@import "tailwindcss";
@plugin "@tailwindcss/typography";
@plugin "@tailwindcss/forms";
@plugin "@tailwindcss/container-queries";
```

## CSS 變數整合（Tailwind v4）

```css
/* assets/css/main.css — 接續 @theme 區塊之後 */

/* 語義化 Token（v4 直接使用 CSS 變數，無需 theme() 函式） */
:root {
  --color-text-primary: var(--color-gray-900);
  --color-text-secondary: var(--color-gray-600);
  --color-text-muted: var(--color-gray-400);
  --color-bg-primary: white;
  --color-bg-secondary: var(--color-gray-50);
  --color-border: var(--color-gray-200);
  --color-focus-ring: var(--color-brand-500);

  --spacing-page: var(--spacing-6);
  --spacing-section: var(--spacing-16);
  --spacing-card: var(--spacing-6);
}

.dark {
  --color-text-primary: var(--color-gray-100);
  --color-text-secondary: var(--color-gray-400);
  --color-text-muted: var(--color-gray-500);
  --color-bg-primary: var(--color-gray-900);
  --color-bg-secondary: var(--color-gray-800);
  --color-border: var(--color-gray-700);
  --color-focus-ring: var(--color-brand-400);
}

/* 全域基底樣式 */
body {
  background-color: var(--color-bg-primary);
  color: var(--color-text-primary);
  -webkit-font-smoothing: antialiased;
  font-feature-settings: 'kern' 1;
}

:focus-visible {
  outline: 2px solid var(--color-focus-ring);
  outline-offset: 2px;
}

/* 自訂 utility（v4 使用 @utility 取代 @layer utilities） */
@utility container-narrow {
  max-width: 48rem;
  margin-inline: auto;
  padding-inline: var(--spacing-page);
}

@utility container-wide {
  max-width: 80rem;
  margin-inline: auto;
  padding-inline: var(--spacing-page);
}

/* 自訂 variant（v4 使用 @variant） */
@variant hocus (&:hover, &:focus-visible);
```

---

## 設計 Token 配置（Tailwind v3 相容）

> 以下為 Tailwind CSS v3 的配置方式，供既有 v3 專案參考。新專案建議直接使用上方的 v4 配置。

```typescript
// tailwind.config.ts — 僅 Tailwind v3 需要此檔案
import type { Config } from 'tailwindcss'
import defaultTheme from 'tailwindcss/defaultTheme'
import forms from '@tailwindcss/forms'
import typography from '@tailwindcss/typography'
import containerQueries from '@tailwindcss/container-queries'

export default {
  darkMode: 'class',
  // 使用 @nuxtjs/tailwindcss 模組時，content 路徑已自動配置，以下僅供未使用模組時參考
  content: [
    './components/**/*.{vue,ts}',
    './layouts/**/*.vue',
    './pages/**/*.vue',
    './composables/**/*.ts',
    './plugins/**/*.ts',
    './utils/**/*.ts',
    './app.vue',
    './error.vue'
  ],
  theme: {
    extend: {
      colors: {
        brand: {
          50:  '#f0f7ff',
          100: '#e0effe',
          200: '#b9dffd',
          300: '#7cc4fc',
          400: '#36a6f8',
          500: '#0c8ce9',
          600: '#006fc7',
          700: '#0058a1',
          800: '#054b85',
          900: '#0a3f6e',
          950: '#072849'
        },
        surface: {
          DEFAULT: '#ffffff',
          secondary: '#f8fafc',
          tertiary: '#f1f5f9',
          dark: '#0f172a',
          'dark-secondary': '#1e293b',
          'dark-tertiary': '#334155'
        }
      },
      fontFamily: {
        sans: ['Noto Sans TC', ...defaultTheme.fontFamily.sans],
        mono: ['JetBrains Mono', ...defaultTheme.fontFamily.mono],
        display: ['Noto Serif TC', 'serif']
      },
      fontSize: {
        'fluid-xs': ['clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem)', { lineHeight: '1.5' }],
        'fluid-sm': ['clamp(0.875rem, 0.8rem + 0.375vw, 1rem)', { lineHeight: '1.5' }],
        'fluid-base': ['clamp(1rem, 0.9rem + 0.5vw, 1.125rem)', { lineHeight: '1.6' }],
        'fluid-lg': ['clamp(1.125rem, 1rem + 0.625vw, 1.25rem)', { lineHeight: '1.5' }],
        'fluid-xl': ['clamp(1.25rem, 1rem + 1.25vw, 1.5rem)', { lineHeight: '1.4' }],
        'fluid-2xl': ['clamp(1.5rem, 1rem + 2.5vw, 2rem)', { lineHeight: '1.3' }],
        'fluid-3xl': ['clamp(1.875rem, 1rem + 4.375vw, 3rem)', { lineHeight: '1.2' }],
        'fluid-4xl': ['clamp(2.25rem, 1rem + 6.25vw, 3.75rem)', { lineHeight: '1.1' }],
      },
      spacing: {
        '4.5': '1.125rem',
        '18': '4.5rem',
        '88': '22rem',
        '128': '32rem'
      },
      borderRadius: {
        '4xl': '2rem',
        '5xl': '2.5rem'
      },
      boxShadow: {
        'soft': '0 2px 15px -3px rgba(0, 0, 0, 0.07), 0 10px 20px -2px rgba(0, 0, 0, 0.04)',
        'soft-lg': '0 10px 40px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05)',
        'inner-soft': 'inset 0 2px 4px 0 rgba(0, 0, 0, 0.04)',
        'glow': '0 0 15px rgba(12, 140, 233, 0.3)',
        'glow-lg': '0 0 30px rgba(12, 140, 233, 0.4)'
      },
      animation: {
        'fade-in': 'fadeIn 0.5s ease-out',
        'fade-up': 'fadeUp 0.5s ease-out',
        'slide-in-right': 'slideInRight 0.3s ease-out',
        'scale-in': 'scaleIn 0.2s ease-out',
        'shimmer': 'shimmer 2s linear infinite'
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' }
        },
        fadeUp: {
          '0%': { opacity: '0', transform: 'translateY(10px)' },
          '100%': { opacity: '1', transform: 'translateY(0)' }
        },
        slideInRight: {
          '0%': { opacity: '0', transform: 'translateX(20px)' },
          '100%': { opacity: '1', transform: 'translateX(0)' }
        },
        scaleIn: {
          '0%': { opacity: '0', transform: 'scale(0.95)' },
          '100%': { opacity: '1', transform: 'scale(1)' }
        },
        shimmer: {
          '0%': { transform: 'translateX(-100%)' },
          '100%': { transform: 'translateX(100%)' }
        }
      },
      transitionTimingFunction: {
        'bounce-in': 'cubic-bezier(0.68, -0.55, 0.265, 1.55)',
        'smooth': 'cubic-bezier(0.4, 0, 0.2, 1)'
      }
    }
  },
  plugins: [forms, typography, containerQueries]
} satisfies Config
```

## CSS 變數整合（Tailwind v3 相容）

```css
/* assets/css/main.css — Tailwind CSS v3 語法 */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    /* 語義化顏色 Token */
    --color-text-primary: theme('colors.gray.900');
    --color-text-secondary: theme('colors.gray.600');
    --color-text-muted: theme('colors.gray.400');
    --color-bg-primary: theme('colors.white');
    --color-bg-secondary: theme('colors.gray.50');
    --color-border: theme('colors.gray.200');
    --color-focus-ring: theme('colors.brand.500');

    /* 間距 Token */
    --spacing-page: theme('spacing.6');
    --spacing-section: theme('spacing.16');
    --spacing-card: theme('spacing.6');

    /* 圓角 Token */
    --radius-sm: theme('borderRadius.lg');
    --radius-md: theme('borderRadius.xl');
    --radius-lg: theme('borderRadius.2xl');
  }

  .dark {
    --color-text-primary: theme('colors.gray.100');
    --color-text-secondary: theme('colors.gray.400');
    --color-text-muted: theme('colors.gray.500');
    --color-bg-primary: theme('colors.gray.900');
    --color-bg-secondary: theme('colors.gray.800');
    --color-border: theme('colors.gray.700');
    --color-focus-ring: theme('colors.brand.400');
  }

  /* 全域基底樣式 */
  body {
    @apply bg-[var(--color-bg-primary)] text-[var(--color-text-primary)] antialiased;
    font-feature-settings: 'kern' 1;
  }

  /* 滾動條美化 */
  ::-webkit-scrollbar {
    @apply w-2;
  }
  ::-webkit-scrollbar-track {
    @apply bg-transparent;
  }
  ::-webkit-scrollbar-thumb {
    @apply bg-gray-300 dark:bg-gray-600 rounded-full;
  }

  /* Focus 可見性 */
  :focus-visible {
    @apply outline-2 outline outline-offset-2 outline-[var(--color-focus-ring)];
  }
}

@layer components {
  /* 骨架載入動畫 */
  .skeleton {
    @apply relative overflow-hidden bg-gray-200 dark:bg-gray-700 rounded-lg;
  }
  .skeleton::after {
    @apply absolute inset-0 animate-shimmer;
    content: '';
    background: linear-gradient(
      90deg,
      transparent,
      rgba(255, 255, 255, 0.15),
      transparent
    );
  }

  /* 玻璃態效果 */
  .glass {
    @apply bg-white/70 dark:bg-gray-900/70 backdrop-blur-xl
           border border-white/20 dark:border-gray-700/30;
  }

  /* 文字漸層 */
  .text-gradient {
    @apply bg-clip-text text-transparent bg-gradient-to-r
           from-brand-500 to-brand-700;
  }
}

@layer utilities {
  /* 容器查詢 */
  .container-narrow {
    @apply max-w-3xl mx-auto px-[var(--spacing-page)];
  }
  .container-wide {
    @apply max-w-7xl mx-auto px-[var(--spacing-page)];
  }
}
```

## Vue Transition 搭配 Tailwind

```vue
<!-- components/shared/FadeTransition.vue -->
<template>
  <Transition
    enter-active-class="transition duration-300 ease-out"
    enter-from-class="opacity-0 translate-y-2"
    enter-to-class="opacity-100 translate-y-0"
    leave-active-class="transition duration-200 ease-in"
    leave-from-class="opacity-100 translate-y-0"
    leave-to-class="opacity-0 translate-y-2"
  >
    <slot />
  </Transition>
</template>
```

```vue
<!-- components/shared/SlideTransition.vue -->
<template>
  <Transition
    enter-active-class="transition-all duration-300 ease-out"
    enter-from-class="opacity-0 -translate-x-4"
    enter-to-class="opacity-100 translate-x-0"
    leave-active-class="transition-all duration-200 ease-in"
    leave-from-class="opacity-100 translate-x-0"
    leave-to-class="opacity-0 translate-x-4"
  >
    <slot />
  </Transition>
</template>
```

```vue
<!-- 列表過渡 -->
<TransitionGroup
  tag="div"
  class="relative"
  enter-active-class="transition-all duration-300 ease-out"
  enter-from-class="opacity-0 translate-y-4"
  enter-to-class="opacity-100 translate-y-0"
  leave-active-class="transition-all duration-200 ease-in absolute"
  leave-from-class="opacity-100"
  leave-to-class="opacity-0"
  move-class="transition-transform duration-300"
>
  <div v-for="item in items" :key="item.id">
    {{ item.name }}
  </div>
</TransitionGroup>
```

## 響應式佈局元件

```vue
<!-- components/layout/ResponsiveGrid.vue -->
<script setup lang="ts">
interface Props {
  cols?: 1 | 2 | 3 | 4
  gap?: 'sm' | 'md' | 'lg'
}

const props = withDefaults(defineProps<Props>(), {
  cols: 3,
  gap: 'md'
})

const colClasses: Record<NonNullable<Props['cols']>, string> = {
  1: 'grid-cols-1',
  2: 'grid-cols-1 sm:grid-cols-2',
  3: 'grid-cols-1 sm:grid-cols-2 lg:grid-cols-3',
  4: 'grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4'
}

const gapClasses: Record<NonNullable<Props['gap']>, string> = {
  sm: 'gap-3',
  md: 'gap-6',
  lg: 'gap-8'
}
</script>

<template>
  <div :class="['grid', colClasses[cols], gapClasses[gap]]">
    <slot />
  </div>
</template>
```

## 深色模式切換

需搭配 `@nuxtjs/color-mode` 模組，並設定 `classSuffix: ''` 使其產生 `class="dark"` 以配合 Tailwind：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/color-mode'],
  colorMode: {
    classSuffix: '',     // 產生 class="dark" 而非 class="dark-mode"
    preference: 'system',
    fallback: 'light'
  }
})
```

```vue
<!-- components/base/BaseThemeToggle.vue -->
<script setup lang="ts">
const colorMode = useColorMode()

function toggle() {
  colorMode.preference = colorMode.value === 'dark' ? 'light' : 'dark'
}
</script>

<template>
  <button
    @click="toggle"
    class="relative p-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors"
    :aria-label="colorMode.value === 'dark' ? '切換為亮色模式' : '切換為深色模式'"
  >
    <!-- 太陽圖示 -->
    <svg
      v-if="colorMode.value === 'dark'"
      class="w-5 h-5 text-yellow-400"
      fill="none" viewBox="0 0 24 24" stroke="currentColor"
    >
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
        d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z" />
    </svg>
    <!-- 月亮圖示 -->
    <svg
      v-else
      class="w-5 h-5 text-gray-600"
      fill="none" viewBox="0 0 24 24" stroke="currentColor"
    >
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
        d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z" />
    </svg>
  </button>
</template>
```

---

## Tailwind CSS v4 遷移策略

> 本指南其餘內容基於 Tailwind CSS v3.x。以下說明 v4 的關鍵差異與遷移策略。

### v3 vs v4 核心差異

| 特性 | v3 | v4 |
|------|-----|-----|
| 配置方式 | `tailwind.config.ts`（JS） | CSS-first（`@theme` 指令） |
| 入口指令 | `@tailwind base/components/utilities` | `@import "tailwindcss"` |
| 設計 Token | `theme.extend` in config | `@theme { }` in CSS |
| Content 掃描 | 手動配置 `content` 陣列 | 自動偵測（不需配置） |
| 套件安裝 | `tailwindcss` + `postcss` + `autoprefixer` | 僅 `tailwindcss` |
| 瀏覽器支援 | 包含舊版瀏覽器 fallback | 現代瀏覽器優先 |

### v4 CSS 配置範例

```css
/* assets/css/main.css — Tailwind v4 */
@import "tailwindcss";

/* 設計 Token（取代 tailwind.config.ts 的 theme.extend） */
@theme {
  --color-primary: #3b82f6;
  --color-primary-dark: #2563eb;
  --color-secondary: #64748b;

  --font-sans: 'Noto Sans TC', system-ui, sans-serif;

  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;

  --spacing-18: 4.5rem;

  --animate-fade-in: fade-in 0.3s ease-out;
}

@keyframes fade-in {
  from { opacity: 0; transform: translateY(-4px); }
  to { opacity: 1; transform: translateY(0); }
}

/* 自訂 utilities（取代 @layer utilities） */
@utility container-narrow {
  max-width: 64rem;
  margin-inline: auto;
  padding-inline: 1rem;
}
```

### 遷移步驟

```bash
# 1. 使用官方遷移工具（自動轉換大部分配置）
npx @tailwindcss/upgrade

# 2. 手動處理遷移工具無法處理的項目：
#    - 自訂 plugin → 改用 @utility / @variant
#    - JavaScript-based 動態配置 → 改用 CSS 變數
#    - 第三方 Tailwind 外掛 → 確認是否有 v4 版本
```

### 與 @nuxtjs/tailwindcss 模組的相容性

```typescript
// Tailwind v4 搭配 Nuxt 的方式：

// 方式 1：使用 @nuxtjs/tailwindcss v7+（支援 v4）
export default defineNuxtConfig({
  modules: ['@nuxtjs/tailwindcss'],
  // v4 不需要 tailwind.config.ts，模組會自動偵測
})

// 方式 2：不使用模組，直接安裝（更輕量）
// npm install tailwindcss @tailwindcss/vite
import tailwindcss from '@tailwindcss/vite'

export default defineNuxtConfig({
  vite: {
    plugins: [
      tailwindcss()
    ]
  },
  css: ['~/assets/css/main.css']
})
```

### 何時遷移

- **新專案**：直接使用 v4（2025 年後的新專案建議預設使用）
- **現有專案**：當所有使用的 Tailwind 外掛都支援 v4 後再遷移
- **暫不遷移**：若專案依賴大量 v3 專屬外掛或自訂 plugin，可繼續使用 v3（LTS 會持續維護）

---

## 無障礙設計模式（Accessibility Patterns）

無障礙（a11y）不是可選功能，而是專業前端工程的基本要求。以下模式符合 WCAG 2.1 AA 標準，適用於 Nuxt 3 + Tailwind v4 的 SPA/SSR 混合架構。

### Skip Navigation Link（跳過導覽連結）

讓鍵盤使用者可直接跳至主要內容區域，避免每次頁面載入都要 Tab 過整個導覽列。

```vue
<!-- components/a11y/SkipNavLink.vue -->
<script setup lang="ts">
interface Props {
  /** 目標錨點 id，預設為 main-content */
  to?: string
  /** 顯示文字 */
  label?: string
}

withDefaults(defineProps<Props>(), {
  to: 'main-content',
  label: '跳至主要內容'
})
</script>

<template>
  <a
    :href="`#${to}`"
    class="skip-nav-link"
  >
    {{ label }}
  </a>
</template>

<style scoped>
/*
  Tailwind v4：使用原生 CSS 搭配 @theme token。
  此元素預設隱藏於畫面外，focus 時滑入可見區域。
*/
.skip-nav-link {
  position: fixed;
  top: -100%;
  left: 1rem;
  z-index: 9999;
  padding: 0.75rem 1.5rem;
  background-color: var(--color-brand-600);
  color: white;
  font-weight: 600;
  border-radius: var(--radius-lg, 0.5rem);
  text-decoration: none;
  transition: top 0.2s ease;
}

.skip-nav-link:focus {
  top: 1rem;
  outline: 2px solid var(--color-brand-400);
  outline-offset: 2px;
}
</style>
```

```vue
<!-- app.vue — 使用方式 -->
<template>
  <div>
    <SkipNavLink />
    <AppHeader />
    <main id="main-content" tabindex="-1">
      <NuxtPage />
    </main>
    <AppFooter />
  </div>
</template>
```

### Focus Trap Composable（焦點陷阱）

Modal、Dialog、Drawer 等浮層元件必須將焦點鎖定在其內部，防止使用者 Tab 到背景元素。

```typescript
// composables/useFocusTrap.ts
import { ref, watch, onUnmounted, type Ref } from 'vue'

interface UseFocusTrapOptions {
  /** 是否在啟用時自動聚焦第一個可聚焦元素 */
  autoFocus?: boolean
  /** 關閉時還原焦點至先前元素 */
  restoreFocus?: boolean
  /** 按下 Escape 時的回呼 */
  onEscape?: () => void
}

const FOCUSABLE_SELECTORS = [
  'a[href]',
  'button:not([disabled])',
  'input:not([disabled])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
  '[contenteditable]',
].join(', ')

export function useFocusTrap(
  containerRef: Ref<HTMLElement | null>,
  isActive: Ref<boolean>,
  options: UseFocusTrapOptions = {}
) {
  const {
    autoFocus = true,
    restoreFocus = true,
    onEscape,
  } = options

  const previouslyFocused = ref<HTMLElement | null>(null)

  function getFocusableElements(): HTMLElement[] {
    if (!containerRef.value) return []
    return Array.from(
      containerRef.value.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTORS)
    ).filter((el) => el.offsetParent !== null) // 排除隱藏元素
  }

  function handleKeyDown(event: KeyboardEvent) {
    if (event.key === 'Escape' && onEscape) {
      event.preventDefault()
      onEscape()
      return
    }

    if (event.key !== 'Tab') return

    const focusable = getFocusableElements()
    if (focusable.length === 0) return

    const first = focusable[0]!
    const last = focusable[focusable.length - 1]!

    if (event.shiftKey) {
      // Shift+Tab：若在第一個元素，跳到最後一個
      if (document.activeElement === first) {
        event.preventDefault()
        last.focus()
      }
    } else {
      // Tab：若在最後一個元素，跳到第一個
      if (document.activeElement === last) {
        event.preventDefault()
        first.focus()
      }
    }
  }

  function activate() {
    previouslyFocused.value = document.activeElement as HTMLElement
    document.addEventListener('keydown', handleKeyDown)

    if (autoFocus) {
      nextTick(() => {
        const focusable = getFocusableElements()
        // 優先聚焦帶有 autofocus 屬性的元素
        const autoFocusEl = containerRef.value?.querySelector<HTMLElement>('[autofocus]')
        ;(autoFocusEl ?? focusable[0])?.focus()
      })
    }
  }

  function deactivate() {
    document.removeEventListener('keydown', handleKeyDown)

    if (restoreFocus && previouslyFocused.value) {
      previouslyFocused.value.focus()
      previouslyFocused.value = null
    }
  }

  watch(isActive, (active) => {
    if (active) activate()
    else deactivate()
  })

  onUnmounted(() => {
    deactivate()
  })

  return { activate, deactivate }
}
```

```vue
<!-- components/base/BaseDialog.vue — 搭配 Focus Trap 使用 -->
<script setup lang="ts">
const props = defineProps<{
  title: string
  description?: string
}>()

const isOpen = defineModel<boolean>({ required: true })
const dialogRef = ref<HTMLElement | null>(null)

useFocusTrap(dialogRef, isOpen, {
  onEscape: () => { isOpen.value = false }
})
</script>

<template>
  <Teleport to="body">
    <Transition
      enter-active-class="transition duration-200 ease-out"
      enter-from-class="opacity-0"
      enter-to-class="opacity-100"
      leave-active-class="transition duration-150 ease-in"
      leave-from-class="opacity-100"
      leave-to-class="opacity-0"
    >
      <div
        v-if="isOpen"
        class="fixed inset-0 z-50 flex items-center justify-center"
      >
        <!-- 背景遮罩 -->
        <div
          class="absolute inset-0 bg-black/50"
          aria-hidden="true"
          @click="isOpen = false"
        />

        <!-- Dialog 本體 -->
        <div
          ref="dialogRef"
          role="dialog"
          :aria-label="title"
          :aria-describedby="description ? 'dialog-desc' : undefined"
          aria-modal="true"
          class="relative z-10 w-full max-w-lg rounded-xl bg-surface p-6 shadow-soft-lg"
        >
          <h2 class="text-lg font-semibold text-[var(--color-text-primary)]">
            {{ title }}
          </h2>
          <p
            v-if="description"
            id="dialog-desc"
            class="mt-1 text-sm text-[var(--color-text-secondary)]"
          >
            {{ description }}
          </p>

          <div class="mt-4">
            <slot />
          </div>

          <div class="mt-6 flex justify-end gap-3">
            <slot name="actions">
              <button
                class="rounded-lg px-4 py-2 text-sm font-medium
                       text-[var(--color-text-secondary)]
                       hover:bg-[var(--color-surface-tertiary)]
                       transition-colors"
                @click="isOpen = false"
              >
                取消
              </button>
            </slot>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>
```

### 鍵盤導覽模式（Keyboard Navigation Patterns）

常見互動元件的鍵盤操作規範：

```typescript
// composables/useKeyboardNavigation.ts
import { ref, type Ref } from 'vue'

type Direction = 'horizontal' | 'vertical' | 'both'

interface UseKeyboardNavigationOptions {
  /** 導覽方向，決定使用哪些方向鍵 */
  direction?: Direction
  /** 是否循環（到最後一個再按下箭頭會回到第一個） */
  loop?: boolean
  /** 聚焦變更時的回呼 */
  onFocusChange?: (index: number) => void
}

export function useKeyboardNavigation(
  items: Ref<HTMLElement[]>,
  options: UseKeyboardNavigationOptions = {}
) {
  const {
    direction = 'vertical',
    loop = true,
    onFocusChange,
  } = options

  const currentIndex = ref(0)

  const prevKeys = new Set<string>()
  const nextKeys = new Set<string>()

  if (direction === 'vertical' || direction === 'both') {
    prevKeys.add('ArrowUp')
    nextKeys.add('ArrowDown')
  }
  if (direction === 'horizontal' || direction === 'both') {
    prevKeys.add('ArrowLeft')
    nextKeys.add('ArrowRight')
  }

  function focusItem(index: number) {
    const item = items.value[index]
    if (item) {
      item.focus()
      currentIndex.value = index
      onFocusChange?.(index)
    }
  }

  function handleKeyDown(event: KeyboardEvent) {
    const total = items.value.length
    if (total === 0) return

    if (prevKeys.has(event.key)) {
      event.preventDefault()
      const next = currentIndex.value - 1
      if (next >= 0) focusItem(next)
      else if (loop) focusItem(total - 1)
    }

    if (nextKeys.has(event.key)) {
      event.preventDefault()
      const next = currentIndex.value + 1
      if (next < total) focusItem(next)
      else if (loop) focusItem(0)
    }

    if (event.key === 'Home') {
      event.preventDefault()
      focusItem(0)
    }

    if (event.key === 'End') {
      event.preventDefault()
      focusItem(total - 1)
    }
  }

  return {
    currentIndex,
    focusItem,
    handleKeyDown,
  }
}
```

```vue
<!-- components/base/BaseTabList.vue — 鍵盤導覽範例：Tabs -->
<script setup lang="ts">
interface Tab {
  id: string
  label: string
  disabled?: boolean
}

const props = defineProps<{
  tabs: Tab[]
}>()

const activeTab = defineModel<string>({ required: true })

const tabRefs = ref<HTMLElement[]>([])
const { currentIndex, handleKeyDown } = useKeyboardNavigation(tabRefs, {
  direction: 'horizontal',
  loop: true,
  onFocusChange: (index) => {
    const tab = props.tabs[index]
    if (tab && !tab.disabled) {
      activeTab.value = tab.id
    }
  },
})
</script>

<template>
  <div
    role="tablist"
    aria-orientation="horizontal"
    @keydown="handleKeyDown"
  >
    <button
      v-for="(tab, index) in tabs"
      :key="tab.id"
      :ref="(el) => { if (el) tabRefs[index] = el as HTMLElement }"
      role="tab"
      :id="`tab-${tab.id}`"
      :aria-selected="activeTab === tab.id"
      :aria-controls="`tabpanel-${tab.id}`"
      :aria-disabled="tab.disabled"
      :tabindex="activeTab === tab.id ? 0 : -1"
      :class="[
        'px-4 py-2 text-sm font-medium transition-colors rounded-t-lg',
        'focus-visible:outline-2 focus-visible:outline-brand-500 focus-visible:outline-offset-[-2px]',
        activeTab === tab.id
          ? 'bg-surface text-brand-600 border-b-2 border-brand-600'
          : 'text-[var(--color-text-secondary)] hover:text-[var(--color-text-primary)] hover:bg-[var(--color-surface-secondary)]',
        tab.disabled && 'opacity-40 cursor-not-allowed',
      ]"
      @click="!tab.disabled && (activeTab = tab.id)"
    >
      {{ tab.label }}
    </button>
  </div>
</template>
```

### ARIA Live Regions（動態內容通知）

SPA 中的動態內容更新（如表單驗證訊息、通知、載入狀態）需透過 ARIA Live Regions 告知螢幕閱讀器。

```vue
<!-- components/a11y/LiveAnnouncer.vue -->
<script setup lang="ts">
/**
 * 全域 ARIA Live Region，透過 provide/inject 讓任何元件都能發出通知。
 * 在 app.vue 或 layout 中放置一次即可。
 */
const announcement = ref('')
const politeness = ref<'polite' | 'assertive'>('polite')

function announce(message: string, level: 'polite' | 'assertive' = 'polite') {
  // 先清空再設定，確保相同訊息也能被重新朗讀
  announcement.value = ''
  politeness.value = level
  nextTick(() => {
    announcement.value = message
  })
}

provide('announce', announce)

// 同時匯出為 composable 供外部使用
defineExpose({ announce })
</script>

<template>
  <div
    :aria-live="politeness"
    aria-atomic="true"
    class="sr-only"
  >
    {{ announcement }}
  </div>
  <slot />
</template>
```

```typescript
// composables/useAnnounce.ts
/**
 * 取得 LiveAnnouncer 提供的 announce 函式。
 * 用於在非元件上下文中發出螢幕閱讀器通知。
 */
type AnnounceFn = (message: string, level?: 'polite' | 'assertive') => void

export function useAnnounce(): AnnounceFn {
  const announce = inject<AnnounceFn>('announce')
  if (!announce) {
    console.warn('[useAnnounce] 未找到 LiveAnnouncer，請確認已在 layout 中放置 <LiveAnnouncer>。')
    return () => {} // fallback：無操作
  }
  return announce
}

// 使用範例
// const announce = useAnnounce()
// announce('商品已加入購物車', 'polite')
// announce('表單驗證失敗：電子郵件格式不正確', 'assertive')
```

### SPA 導覽的螢幕閱讀器考量

在 Nuxt SPA 模式下，路由切換不會觸發傳統的頁面載入事件，螢幕閱讀器可能不知道頁面已變更。

```typescript
// plugins/a11y-route-announce.client.ts
/**
 * Nuxt plugin：在路由切換後自動通知螢幕閱讀器。
 * 僅在客戶端執行。
 */
export default defineNuxtPlugin(() => {
  const router = useRouter()

  router.afterEach((to) => {
    nextTick(() => {
      // 取得頁面標題作為通知內容
      const pageTitle =
        document.title ||
        (to.meta.title as string | undefined) ||
        '頁面已載入'

      // 使用 aria-live region 通知
      let announcer = document.getElementById('route-announcer')
      if (!announcer) {
        announcer = document.createElement('div')
        announcer.id = 'route-announcer'
        announcer.setAttribute('aria-live', 'assertive')
        announcer.setAttribute('aria-atomic', 'true')
        announcer.setAttribute('role', 'status')
        announcer.className = 'sr-only'
        document.body.appendChild(announcer)
      }

      // 清空後重新設定，確保螢幕閱讀器朗讀
      announcer.textContent = ''
      setTimeout(() => {
        announcer!.textContent = `已導覽至 ${pageTitle}`
      }, 100)

      // 將焦點移至主內容區域
      const mainContent = document.getElementById('main-content')
      if (mainContent) {
        mainContent.focus({ preventScroll: false })
      }
    })
  })
})
```

### 色彩對比要求（WCAG AA）

WCAG AA 要求一般文字的色彩對比度至少為 **4.5:1**，大號文字（18px 以上粗體或 24px 以上一般）為 **3:1**。

```css
/* assets/css/main.css — 確保對比度的語義色彩 Token */
@import "tailwindcss";

@theme {
  /*
    以下色彩組合皆已通過 WCAG AA 4.5:1 對比度檢查：
    - brand-700 on white → 8.2:1 ✓
    - brand-600 on white → 5.4:1 ✓
    - brand-500 on white → 4.0:1 ✗（僅限大號文字）
    - gray-700 on white  → 7.2:1 ✓（適合正文）
    - gray-600 on white  → 5.7:1 ✓（適合次要文字）
    - gray-500 on white  → 4.6:1 ✓（臨界值，僅限 16px+ 文字）
    - white on brand-600 → 5.4:1 ✓
    - white on brand-700 → 8.2:1 ✓
  */

  /* 確保文字對比度的安全色彩 */
  --color-text-safe: #374151;       /* gray-700，對白色背景 7.2:1 */
  --color-text-safe-muted: #4b5563; /* gray-600，對白色背景 5.7:1 */
  --color-link-safe: #0058a1;       /* brand-700，對白色背景 8.2:1 */
}
```

```typescript
// utils/contrastChecker.ts
/**
 * 計算兩個顏色之間的相對亮度對比度（WCAG 2.1 演算法）。
 * 用於開發階段驗證色彩搭配是否符合標準。
 */
function hexToRgb(hex: string): [number, number, number] {
  const cleaned = hex.replace('#', '')
  return [
    parseInt(cleaned.slice(0, 2), 16),
    parseInt(cleaned.slice(2, 4), 16),
    parseInt(cleaned.slice(4, 6), 16),
  ]
}

function relativeLuminance(r: number, g: number, b: number): number {
  const [rs, gs, bs] = [r, g, b].map((c) => {
    const s = c / 255
    return s <= 0.03928 ? s / 12.92 : Math.pow((s + 0.055) / 1.055, 2.4)
  })
  return 0.2126 * rs! + 0.7152 * gs! + 0.0722 * bs!
}

export function getContrastRatio(color1: string, color2: string): number {
  const [r1, g1, b1] = hexToRgb(color1)
  const [r2, g2, b2] = hexToRgb(color2)

  const l1 = relativeLuminance(r1, g1, b1)
  const l2 = relativeLuminance(r2, g2, b2)

  const lighter = Math.max(l1, l2)
  const darker = Math.min(l1, l2)

  return (lighter + 0.05) / (darker + 0.05)
}

export function meetsWcagAA(
  foreground: string,
  background: string,
  isLargeText = false
): boolean {
  const ratio = getContrastRatio(foreground, background)
  return isLargeText ? ratio >= 3 : ratio >= 4.5
}

// 使用範例（開發階段驗證）：
// console.log(getContrastRatio('#0058a1', '#ffffff')) // 8.2
// console.log(meetsWcagAA('#0058a1', '#ffffff'))      // true
```

### 無障礙設計檢查清單

| 項目 | 要求 | Tailwind / Nuxt 實作方式 |
|------|------|--------------------------|
| Skip Navigation | 鍵盤使用者可跳過導覽列 | `<SkipNavLink>` 元件 + `sr-only` + `focus:not-sr-only` |
| 焦點管理 | Modal/Dialog 鎖定焦點 | `useFocusTrap` composable |
| 焦點可見性 | 所有互動元素有清晰焦點樣式 | `focus-visible:outline-2 focus-visible:outline-brand-500` |
| 鍵盤操作 | Tab/Escape/Arrow 遵循 WAI-ARIA 規範 | `useKeyboardNavigation` + `role` + `aria-*` 屬性 |
| 動態通知 | 內容變更通知螢幕閱讀器 | `aria-live="polite"` / `aria-live="assertive"` |
| 路由切換 | SPA 導覽通知螢幕閱讀器 | `a11y-route-announce` 插件 |
| 色彩對比 | 文字 4.5:1 / 大字 3:1 | 使用安全色彩 Token + `getContrastRatio()` 驗證 |
| 替代文字 | 圖片有適當 alt 描述 | `<img :alt="描述">` / 裝飾圖用 `alt=""` + `aria-hidden="true"` |
| 表單標籤 | 每個表單控件有關聯的 label | `<label :for="id">` / `aria-label` / `aria-labelledby` |
| 動畫偏好 | 尊重減少動畫偏好 | `motion-safe:` / `motion-reduce:` variant |

---

## 響應式設計進階（Advanced Responsive Design）

### Mobile-First 設計哲學與 Tailwind v4

Tailwind 預設採用 mobile-first 方法：無前綴的 class 作用於所有螢幕尺寸，斷點前綴（`sm:`、`md:` 等）向上覆蓋。在 Tailwind v4 中，斷點可於 `@theme` 中自訂。

```css
/* assets/css/main.css — Tailwind v4 自訂斷點 */
@import "tailwindcss";

@theme {
  /* 覆蓋預設斷點（若需自訂） */
  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;
  --breakpoint-2xl: 1536px;

  /* 新增自訂斷點 */
  --breakpoint-3xl: 1920px;

  /* 最大寬度容器 */
  --container-max-sm: 640px;
  --container-max-md: 768px;
  --container-max-lg: 1024px;
  --container-max-xl: 1280px;
}
```

```vue
<!-- Mobile-First 的 class 撰寫順序範例 -->
<template>
  <!-- ✅ 正確：先定義行動版，再逐步覆蓋 -->
  <div class="
    px-4 py-6                         /* 行動版：小間距 */
    sm:px-6 sm:py-8                   /* ≥640px：中間距 */
    lg:px-8 lg:py-12                  /* ≥1024px：大間距 */
    xl:max-w-7xl xl:mx-auto           /* ≥1280px：限制最大寬度 */
  ">
    <h1 class="
      text-2xl font-bold              /* 行動版：基礎字級 */
      md:text-3xl                     /* ≥768px：放大 */
      lg:text-4xl                     /* ≥1024px：再放大 */
    ">
      響應式標題
    </h1>
  </div>
</template>
```

### Container Queries（容器查詢）

Container Queries 讓元件根據**父容器寬度**而非視窗寬度來調整佈局，真正實現元件的獨立響應式設計。

```css
/* assets/css/main.css — Tailwind v4 搭配 container queries */
@import "tailwindcss";
@plugin "@tailwindcss/container-queries";
```

```vue
<!-- components/feature/ProductCard.vue -->
<script setup lang="ts">
interface Props {
  title: string
  description: string
  price: number
  imageUrl: string
}

defineProps<Props>()
</script>

<template>
  <!-- @container 標記此元素為容器查詢的參考容器 -->
  <div class="@container">
    <article class="
      flex flex-col gap-3 rounded-xl bg-surface p-4 shadow-soft
      @sm:flex-row @sm:items-center @sm:gap-4
      @md:p-6
      @lg:gap-6
    ">
      <!-- 圖片：根據容器寬度調整尺寸 -->
      <img
        :src="imageUrl"
        :alt="title"
        class="
          w-full rounded-lg object-cover aspect-video
          @sm:w-32 @sm:h-32 @sm:aspect-square
          @md:w-40 @md:h-40
          @lg:w-48 @lg:h-48
        "
      />

      <!-- 文字內容 -->
      <div class="flex-1 min-w-0">
        <h3 class="
          text-base font-semibold text-[var(--color-text-primary)] truncate
          @md:text-lg
          @lg:text-xl
        ">
          {{ title }}
        </h3>
        <p class="
          mt-1 text-sm text-[var(--color-text-secondary)] line-clamp-2
          @md:text-base @md:line-clamp-3
        ">
          {{ description }}
        </p>
        <p class="
          mt-2 text-lg font-bold text-brand-600
          @lg:text-xl
        ">
          NT$ {{ price.toLocaleString() }}
        </p>
      </div>
    </article>
  </div>
</template>
```

```vue
<!-- 容器查詢的強大之處：同一個元件在不同容器寬度中自動調整 -->
<template>
  <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
    <!-- 側邊欄：容器較窄 → ProductCard 呈現垂直排列 -->
    <aside class="lg:col-span-1">
      <ProductCard v-bind="product" />
    </aside>

    <!-- 主內容區：容器較寬 → 同一個 ProductCard 呈現水平排列 -->
    <main class="lg:col-span-2">
      <ProductCard v-bind="product" />
    </main>
  </div>
</template>
```

### 響應式導覽（Hamburger Menu + Drawer）

```vue
<!-- components/layout/AppNavbar.vue -->
<script setup lang="ts">
interface NavItem {
  label: string
  to: string
  icon?: string
}

const props = defineProps<{
  items: NavItem[]
}>()

const isMobileMenuOpen = ref(false)
const drawerRef = ref<HTMLElement | null>(null)

// 關閉 menu 時還原焦點
useFocusTrap(drawerRef, isMobileMenuOpen, {
  onEscape: () => { isMobileMenuOpen.value = false },
})

// 路由切換時自動關閉
const route = useRoute()
watch(() => route.path, () => {
  isMobileMenuOpen.value = false
})

// 防止背景滾動（僅客戶端）
if (import.meta.client) {
  watch(isMobileMenuOpen, (open) => {
    document.body.style.overflow = open ? 'hidden' : ''
  })
}
</script>

<template>
  <nav class="sticky top-0 z-40 bg-surface/80 backdrop-blur-lg border-b border-[var(--color-border)]">
    <div class="container-wide flex items-center justify-between h-16">
      <!-- Logo -->
      <NuxtLink to="/" class="text-xl font-bold text-brand-600">
        MyApp
      </NuxtLink>

      <!-- 桌面版導覽 -->
      <ul class="hidden md:flex items-center gap-1" role="menubar">
        <li v-for="item in items" :key="item.to" role="none">
          <NuxtLink
            :to="item.to"
            role="menuitem"
            class="px-4 py-2 rounded-lg text-sm font-medium
                   text-[var(--color-text-secondary)]
                   hover:text-[var(--color-text-primary)]
                   hover:bg-[var(--color-surface-secondary)]
                   transition-colors"
            active-class="text-brand-600 bg-brand-50"
          >
            {{ item.label }}
          </NuxtLink>
        </li>
      </ul>

      <!-- 行動版漢堡按鈕 -->
      <button
        class="md:hidden p-2 rounded-lg hover:bg-[var(--color-surface-secondary)] transition-colors"
        :aria-expanded="isMobileMenuOpen"
        aria-controls="mobile-menu"
        aria-label="開啟導覽選單"
        @click="isMobileMenuOpen = true"
      >
        <svg class="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
        </svg>
      </button>
    </div>

    <!-- 行動版側邊抽屜 -->
    <Teleport to="body">
      <Transition
        enter-active-class="transition duration-300 ease-out"
        enter-from-class="opacity-0"
        enter-to-class="opacity-100"
        leave-active-class="transition duration-200 ease-in"
        leave-from-class="opacity-100"
        leave-to-class="opacity-0"
      >
        <div
          v-if="isMobileMenuOpen"
          class="fixed inset-0 z-50 md:hidden"
        >
          <!-- 背景遮罩 -->
          <div
            class="absolute inset-0 bg-black/50"
            aria-hidden="true"
            @click="isMobileMenuOpen = false"
          />

          <!-- 抽屜面板 -->
          <Transition
            appear
            enter-active-class="transition-transform duration-300 ease-out"
            enter-from-class="-translate-x-full"
            enter-to-class="translate-x-0"
            leave-active-class="transition-transform duration-200 ease-in"
            leave-from-class="translate-x-0"
            leave-to-class="-translate-x-full"
          >
            <div
              v-if="isMobileMenuOpen"
              ref="drawerRef"
              id="mobile-menu"
              role="dialog"
              aria-label="導覽選單"
              aria-modal="true"
              class="relative w-72 max-w-[80vw] h-full bg-surface shadow-soft-lg p-6"
            >
              <!-- 關閉按鈕 -->
              <button
                class="absolute top-4 right-4 p-2 rounded-lg
                       hover:bg-[var(--color-surface-secondary)] transition-colors"
                aria-label="關閉導覽選單"
                @click="isMobileMenuOpen = false"
              >
                <svg class="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                </svg>
              </button>

              <!-- 導覽項目 -->
              <ul class="mt-8 space-y-1" role="menu">
                <li v-for="item in items" :key="item.to" role="none">
                  <NuxtLink
                    :to="item.to"
                    role="menuitem"
                    class="block px-4 py-3 rounded-lg text-base font-medium
                           text-[var(--color-text-secondary)]
                           hover:text-[var(--color-text-primary)]
                           hover:bg-[var(--color-surface-secondary)]
                           transition-colors
                           min-h-[44px] flex items-center"
                    active-class="text-brand-600 bg-brand-50"
                    @click="isMobileMenuOpen = false"
                  >
                    {{ item.label }}
                  </NuxtLink>
                </li>
              </ul>
            </div>
          </Transition>
        </div>
      </Transition>
    </Teleport>
  </nav>
</template>
```

### 觸控目標尺寸（Touch Target Sizing）

WCAG 2.5.5 要求互動目標的最小尺寸為 **44x44 CSS 像素**。Tailwind v4 可透過自訂 utility 強制執行。

```css
/* assets/css/main.css — 觸控目標 utility */
@import "tailwindcss";

/* 確保互動元素的最小觸控尺寸 */
@utility touch-target {
  min-width: 44px;
  min-height: 44px;
}

@utility touch-target-sm {
  min-width: 36px;
  min-height: 36px;
}

@utility touch-target-lg {
  min-width: 48px;
  min-height: 48px;
}
```

```vue
<!-- 觸控目標的實務應用 -->
<template>
  <!-- ✅ 正確：按鈕有足夠的觸控區域 -->
  <button class="touch-target flex items-center justify-center p-2 rounded-lg">
    <svg class="w-5 h-5" aria-hidden="true"><!-- icon --></svg>
    <span class="sr-only">關閉</span>
  </button>

  <!-- ✅ 正確：連結有足夠的 padding 來達到 44px -->
  <NuxtLink
    to="/settings"
    class="inline-flex items-center px-4 py-3 min-h-[44px] text-sm"
  >
    設定
  </NuxtLink>

  <!-- ✅ 正確：清單項目有足夠的高度 -->
  <ul class="divide-y divide-[var(--color-border)]">
    <li v-for="item in items" :key="item.id">
      <button class="w-full text-left px-4 py-3 min-h-[44px] hover:bg-[var(--color-surface-secondary)]">
        {{ item.label }}
      </button>
    </li>
  </ul>

  <!-- ❌ 錯誤：觸控目標太小 -->
  <!-- <button class="p-1 text-xs">X</button> -->
</template>
```

### 響應式圖片（srcset 與 sizes）

```vue
<!-- components/base/BaseResponsiveImage.vue -->
<script setup lang="ts">
interface ImageSource {
  src: string
  width: number
}

interface Props {
  /** 各尺寸的圖片來源 */
  sources: ImageSource[]
  /** 圖片替代文字 */
  alt: string
  /** sizes 屬性，描述圖片在各斷點的顯示寬度 */
  sizes?: string
  /** 圖片長寬比，用於 CLS 預防 */
  aspectRatio?: string
  /** 是否使用 lazy loading */
  lazy?: boolean
  /** 是否為裝飾性圖片 */
  decorative?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  sizes: '(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw',
  aspectRatio: '16/9',
  lazy: true,
  decorative: false,
})

const srcset = computed(() =>
  props.sources
    .map((s) => `${s.src} ${s.width}w`)
    .join(', ')
)

// 預設使用最大的圖片作為 fallback src
const fallbackSrc = computed(() => {
  const largest = [...props.sources].sort((a, b) => b.width - a.width)[0]
  return largest?.src ?? ''
})
</script>

<template>
  <img
    :src="fallbackSrc"
    :srcset="srcset"
    :sizes="sizes"
    :alt="decorative ? '' : alt"
    :aria-hidden="decorative ? 'true' : undefined"
    :loading="lazy ? 'lazy' : 'eager'"
    :decoding="lazy ? 'async' : 'auto'"
    :style="{ aspectRatio }"
    class="w-full h-auto object-cover"
  />
</template>
```

```vue
<!-- 使用範例 -->
<template>
  <BaseResponsiveImage
    :sources="[
      { src: '/images/hero-400.webp', width: 400 },
      { src: '/images/hero-800.webp', width: 800 },
      { src: '/images/hero-1200.webp', width: 1200 },
      { src: '/images/hero-1600.webp', width: 1600 },
    ]"
    alt="首頁橫幅圖片"
    sizes="(max-width: 640px) 100vw, (max-width: 1024px) 80vw, 1200px"
    aspect-ratio="21/9"
    :lazy="false"
  />
</template>
```

### 斷點策略決策指南

選擇正確的響應式策略取決於專案性質與目標使用者。

```
選擇斷點策略：

1. 專案是否以行動端為主要平台？
   ├── 是 → Mobile-First（Tailwind 預設）
   │   └── 以無前綴 class 定義行動版，sm:/md:/lg: 向上覆蓋
   │
   └── 否 → 是否為管理後台 / 桌面工具？
       ├── 是 → 考慮 Desktop-First
       │   └── Tailwind v4 可使用 max-* variant 反向定義
       │       例：max-md:hidden 在 <768px 時隱藏
       │
       └── 否 → 採用 Mobile-First + Container Queries 混合策略
           └── 頁面佈局用斷點，元件用 @container
```

| 策略 | 適用場景 | Tailwind 寫法 |
|------|---------|---------------|
| **Mobile-First** | 消費端產品、電商、部落格 | `text-sm md:text-base lg:text-lg` |
| **Desktop-First** | 企業後台、資料密集型介面 | `text-lg max-md:text-base max-sm:text-sm` |
| **Container Queries** | 可重用元件庫、Widget | `@container` + `@sm:` `@md:` |
| **流體排版** | 內容導向網站 | `text-fluid-base`（使用 `clamp()`） |

```vue
<!-- 混合策略範例：頁面佈局用斷點，卡片用容器查詢 -->
<template>
  <div class="container-wide">
    <!-- 頁面層級：傳統斷點控制欄數 -->
    <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6">
      <!-- 元件層級：容器查詢控制內部排版 -->
      <div v-for="item in items" :key="item.id" class="@container">
        <article class="
          flex flex-col gap-3 p-4
          @sm:flex-row @sm:items-start
          @md:p-6
        ">
          <img
            :src="item.image"
            :alt="item.title"
            class="
              w-full rounded-lg aspect-video object-cover
              @sm:w-24 @sm:h-24 @sm:aspect-square
            "
          />
          <div class="flex-1 min-w-0">
            <h3 class="font-semibold truncate @md:text-lg">{{ item.title }}</h3>
            <p class="text-sm text-[var(--color-text-secondary)] line-clamp-2 mt-1">
              {{ item.description }}
            </p>
          </div>
        </article>
      </div>
    </div>
  </div>
</template>
```

---

## 動畫減少偏好防護（Motion-Reduce Guards）

尊重使用者的 `prefers-reduced-motion` 設定是無障礙設計的基本要求。部分使用者（如前庭障礙患者）會在作業系統中啟用「減少動畫」，前端應用必須據此抑制或移除非必要的動畫效果。

### Tailwind `motion-reduce:` Variant

Tailwind 內建 `motion-reduce:` 和 `motion-safe:` variant，可直接在 class 中根據使用者偏好切換樣式。

```vue
<!-- components/AnimatedCard.vue -->
<template>
  <div
    class="transform transition-all duration-300
           motion-reduce:transform-none motion-reduce:transition-none"
    :class="{ 'translate-y-0 opacity-100': isVisible, '-translate-y-4 opacity-0': !isVisible }"
  >
    <slot />
  </div>
</template>

<script setup lang="ts">
const props = defineProps<{
  isVisible: boolean
}>()
</script>
```

```vue
<!-- 反向策略：僅在允許動畫時才啟用（motion-safe:） -->
<template>
  <button
    class="rounded-lg bg-brand-600 px-4 py-2 text-white
           motion-safe:transition-all motion-safe:duration-200
           motion-safe:hover:scale-105 motion-safe:hover:shadow-glow
           motion-safe:active:scale-95"
  >
    <slot />
  </button>
</template>
```

### CSS 層級 `@media` Fallback

對於無法使用 Tailwind variant 的情境（如 keyframe 動畫、第三方元件），使用 CSS media query 作為防護。

```css
/* assets/css/main.css — 全域動畫減少防護 */
@import "tailwindcss";

/* 在 @theme 定義的動畫基礎上，加入全域防護 */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

### Vue Composable：`useReducedMotion`

在 Vue 元件邏輯中偵測使用者偏好，用於動態控制 JavaScript 驅動的動畫行為。

```typescript
// composables/useReducedMotion.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useReducedMotion() {
  const prefersReducedMotion = ref(false)

  onMounted(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)')
    prefersReducedMotion.value = mediaQuery.matches

    const handler = (e: MediaQueryListEvent) => {
      prefersReducedMotion.value = e.matches
    }
    mediaQuery.addEventListener('change', handler)
    onUnmounted(() => mediaQuery.removeEventListener('change', handler))
  })

  return { prefersReducedMotion: readonly(prefersReducedMotion) }
}
```

### 搭配 Vue Transition 使用

當 `prefers-reduced-motion` 啟用時，動態禁用 `<Transition>` 的動畫名稱，使內容立即切換而不播放過渡動畫。

```vue
<!-- components/shared/SafeTransition.vue -->
<script setup lang="ts">
/**
 * 尊重 prefers-reduced-motion 的 Transition 包裝元件。
 * 當使用者偏好減少動畫時，自動停用過渡效果。
 */
interface Props {
  name?: string
  mode?: 'in-out' | 'out-in' | 'default'
}

const props = withDefaults(defineProps<Props>(), {
  name: 'fade',
  mode: 'out-in',
})

const { prefersReducedMotion } = useReducedMotion()
</script>

<template>
  <Transition
    :name="prefersReducedMotion ? '' : name"
    :mode="mode"
  >
    <slot />
  </Transition>
</template>
```

```vue
<!-- 使用範例：在頁面或元件中使用 SafeTransition -->
<script setup lang="ts">
const currentView = ref('dashboard')
</script>

<template>
  <SafeTransition name="slide-fade">
    <component :is="currentView" />
  </SafeTransition>
</template>
```

```vue
<!-- 直接在頁面中使用 composable 控制動畫行為 -->
<script setup lang="ts">
const { prefersReducedMotion } = useReducedMotion()
const currentView = shallowRef(HomeView)
const items = ref([
  { id: 1, name: 'Item A' },
  { id: 2, name: 'Item B' },
  { id: 3, name: 'Item C' },
])
</script>

<template>
  <Transition :name="prefersReducedMotion ? '' : 'slide-fade'" mode="out-in">
    <component :is="currentView" />
  </Transition>

  <!-- TransitionGroup 同理 -->
  <TransitionGroup
    :name="prefersReducedMotion ? '' : 'list'"
    tag="ul"
    :move-class="prefersReducedMotion ? '' : 'transition-transform duration-300'"
  >
    <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  </TransitionGroup>
</template>
```

---

## 圖片格式協商與 `<picture>` 元素（Picture Element with Format Negotiation）

現代瀏覽器支援 AVIF 和 WebP 等高效圖片格式，透過 `<picture>` 元素可實現自動格式協商（format negotiation），讓瀏覽器選用最佳格式，同時為不支援的瀏覽器提供 fallback。

### 手動 `<picture>` 格式協商

```vue
<!-- components/base/BasePicture.vue -->
<script setup lang="ts">
interface Props {
  /** 圖片路徑（不含副檔名） */
  src: string
  alt: string
  width: number
  height: number
  /** 是否支援 2x 高解析度 */
  retina?: boolean
  /** 是否使用 lazy loading（預設 true） */
  lazy?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  retina: true,
  lazy: true,
})
</script>

<template>
  <picture>
    <!-- AVIF：最小檔案、最佳壓縮 -->
    <source
      type="image/avif"
      :srcset="retina
        ? `${src}.avif 1x, ${src}@2x.avif 2x`
        : `${src}.avif`"
    />
    <!-- WebP：廣泛支援的現代格式 -->
    <source
      type="image/webp"
      :srcset="retina
        ? `${src}.webp 1x, ${src}@2x.webp 2x`
        : `${src}.webp`"
    />
    <!-- JPEG fallback：所有瀏覽器支援 -->
    <img
      :src="`${src}.jpg`"
      :alt="alt"
      :width="width"
      :height="height"
      :loading="lazy ? 'lazy' : 'eager'"
      :decoding="lazy ? 'async' : 'auto'"
      class="object-cover"
    />
  </picture>
</template>
```

### Art Direction：不同視窗使用不同裁切

Art direction 讓你在不同螢幕尺寸提供不同構圖的圖片（而非僅縮放同一張圖），適用於 Hero Banner 等場景。

```vue
<!-- components/feature/HeroBanner.vue -->
<template>
  <picture>
    <!-- 桌面版：寬版橫幅 -->
    <source media="(min-width: 1024px)" srcset="/hero-wide.avif" type="image/avif" />
    <source media="(min-width: 1024px)" srcset="/hero-wide.webp" type="image/webp" />
    <!-- 平板版：中等裁切 -->
    <source media="(min-width: 640px)" srcset="/hero-medium.avif" type="image/avif" />
    <source media="(min-width: 640px)" srcset="/hero-medium.webp" type="image/webp" />
    <!-- 手機版：直式裁切（預設） -->
    <source srcset="/hero-small.avif" type="image/avif" />
    <source srcset="/hero-small.webp" type="image/webp" />
    <img
      src="/hero-small.jpg"
      alt="Hero Banner"
      width="800"
      height="400"
      loading="eager"
      decoding="async"
      fetchpriority="high"
      class="w-full h-auto object-cover rounded-xl"
    />
  </picture>
</template>
```

### 整合 Nuxt Image 模組

`@nuxt/image` 提供 `<NuxtPicture>` 元件，可自動產生多格式 `<source>` 標籤並整合 CDN 圖片最佳化。

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/image'],
  image: {
    // 可選：配置圖片 provider（如 Cloudinary、imgix）
    quality: 80,
    format: ['avif', 'webp'],
    screens: {
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280,
    },
  },
})
```

```vue
<!-- 使用 <NuxtPicture> 自動格式協商 -->
<template>
  <NuxtPicture
    src="/images/hero.jpg"
    format="avif,webp"
    sizes="sm:100vw md:50vw lg:800px"
    :img-attrs="{
      class: 'rounded-lg object-cover w-full',
      fetchpriority: 'high',
    }"
    loading="lazy"
    alt="產品展示圖"
    width="800"
    height="450"
  />

  <!-- 搭配 placeholder 實現漸進式載入 -->
  <NuxtPicture
    src="/images/product.jpg"
    format="avif,webp"
    sizes="sm:100vw md:50vw lg:400px"
    placeholder
    :img-attrs="{ class: 'rounded-xl object-cover aspect-square' }"
    loading="lazy"
    alt="商品圖片"
  />
</template>
```

### 圖片最佳實踐檢查清單

| 項目 | 說明 |
|------|------|
| **格式優先序** | AVIF > WebP > JPEG（`<source>` 由上到下） |
| **明確尺寸** | 始終設定 `width` 和 `height`，防止 CLS（Cumulative Layout Shift） |
| **Lazy Loading** | 首屏圖片用 `loading="eager"` + `fetchpriority="high"`，其餘用 `loading="lazy"` |
| **Decoding** | 非首屏圖片加上 `decoding="async"` 避免阻塞主執行緒 |
| **Alt 文字** | 內容圖片提供描述性 `alt`，裝飾圖片用 `alt=""` + `aria-hidden="true"` |
| **Art Direction** | Hero / Banner 等跨裝置構圖差異大的場景使用 `<source media="...">` |

---

## View Transitions API（頁面視圖轉場）

View Transitions API 為頁面間導覽提供原生的動畫轉場能力，可實現如 shared element transition（共享元素轉場）等效果。Nuxt 3.4+ 內建實驗性支援。

### 啟用 View Transitions

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  experimental: {
    viewTransition: true,
  },
})
```

### 共享元素轉場（Shared Element Transitions）

透過 `view-transition-name` 將兩個頁面中對應的元素關聯起來，瀏覽器會自動計算並動畫化它們之間的位置與尺寸變化。

```vue
<!-- pages/products/index.vue — 商品列表頁 -->
<script setup lang="ts">
const { data: products } = await useFetch('/api/products')
</script>

<template>
  <div class="container-wide py-8">
    <h1 class="text-fluid-2xl font-bold mb-8">商品列表</h1>
    <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
      <NuxtLink
        v-for="product in products"
        :key="product.id"
        :to="`/products/${product.id}`"
        class="group block rounded-xl bg-surface shadow-soft
               motion-safe:hover:shadow-soft-lg motion-safe:transition-shadow"
      >
        <img
          :src="product.image"
          :alt="product.title"
          :style="{ viewTransitionName: `product-image-${product.id}` }"
          class="w-full aspect-square object-cover rounded-t-xl"
        />
        <div class="p-4">
          <h2
            :style="{ viewTransitionName: `product-title-${product.id}` }"
            class="font-semibold text-lg"
          >
            {{ product.title }}
          </h2>
          <p class="text-brand-600 font-bold mt-1">
            NT$ {{ product.price.toLocaleString() }}
          </p>
        </div>
      </NuxtLink>
    </div>
  </div>
</template>
```

```vue
<!-- pages/products/[id].vue — 商品詳情頁 -->
<script setup lang="ts">
const route = useRoute()
const { data: product } = await useFetch(`/api/products/${route.params.id}`)
</script>

<template>
  <div v-if="product" class="container-wide py-8">
    <img
      :src="product.image"
      :alt="product.title"
      :style="{ viewTransitionName: `product-image-${product.id}` }"
      class="w-full max-w-2xl rounded-xl object-cover"
    />
    <h1
      :style="{ viewTransitionName: `product-title-${product.id}` }"
      class="text-fluid-3xl font-bold mt-6"
    >
      {{ product.title }}
    </h1>
    <p class="text-fluid-base text-[var(--color-text-secondary)] mt-4">
      {{ product.description }}
    </p>
  </div>
</template>
```

### 自訂轉場動畫 CSS

```css
/* assets/css/transitions.css */

/* 預設根元素的進出動畫 */
::view-transition-old(root) {
  animation: fade-and-scale-out 0.2s ease-in forwards;
}

::view-transition-new(root) {
  animation: fade-and-scale-in 0.3s ease-out forwards;
}

@keyframes fade-and-scale-out {
  to {
    opacity: 0;
    transform: scale(0.98);
  }
}

@keyframes fade-and-scale-in {
  from {
    opacity: 0;
    transform: scale(1.02);
  }
}

/* 共享元素的轉場設定 */
::view-transition-group(product-image-*) {
  animation-duration: 0.4s;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}

::view-transition-group(product-title-*) {
  animation-duration: 0.35s;
  animation-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}

/* Motion-Reduce Guard：尊重使用者的減少動畫偏好 */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(root),
  ::view-transition-new(root),
  ::view-transition-group(*),
  ::view-transition-image-pair(*) {
    animation: none !important;
  }
}
```

```typescript
// nuxt.config.ts — 引入轉場樣式
export default defineNuxtConfig({
  experimental: {
    viewTransition: true,
  },
  css: [
    '~/assets/css/main.css',
    '~/assets/css/transitions.css',
  ],
})
```

### 程式化控制與 Fallback

```typescript
// composables/useViewTransition.ts
/**
 * 封裝 View Transitions API，提供 fallback 支援。
 * 當瀏覽器不支援時，直接執行回呼而不播放動畫。
 */
export function useViewTransition() {
  const isSupported = ref(false)

  onMounted(() => {
    isSupported.value = 'startViewTransition' in document
  })

  function startTransition(callback: () => void | Promise<void>) {
    if (!isSupported.value) {
      callback()
      return
    }

    // @ts-expect-error -- View Transitions API 尚未在所有 TypeScript 版本中定義
    document.startViewTransition(callback)
  }

  return { isSupported: readonly(isSupported), startTransition }
}
```

```vue
<!-- 程式化觸發視圖轉場 -->
<script setup lang="ts">
const { startTransition } = useViewTransition()
const layout = ref<'grid' | 'list'>('grid')

function toggleLayout() {
  startTransition(() => {
    layout.value = layout.value === 'grid' ? 'list' : 'grid'
  })
}
</script>

<template>
  <button @click="toggleLayout" class="rounded-lg bg-brand-600 px-4 py-2 text-white">
    切換為{{ layout === 'grid' ? '列表' : '網格' }}檢視
  </button>

  <div :class="layout === 'grid' ? 'grid grid-cols-3 gap-4' : 'flex flex-col gap-2'">
    <div v-for="item in items" :key="item.id" :style="{ viewTransitionName: `item-${item.id}` }">
      {{ item.name }}
    </div>
  </div>
</template>
```

---

## Tailwind v4 `@source` 指令（Content Source Detection）

Tailwind v4 預設會自動偵測專案中的 class 使用情況，但在某些場景（第三方套件的 class、動態產生的 class 字串）需要透過 `@source` 指令明確指定掃描範圍。

### 基本用法

```css
/* app.css — Tailwind v4 CSS-first config */
@import "tailwindcss";

/* 明確指定 content 掃描來源（自動偵測不足時使用） */
@source "../components/**/*.vue";
@source "../pages/**/*.vue";
@source "../layouts/**/*.vue";
@source "../composables/**/*.ts";
```

### 掃描第三方套件

當專案使用的第三方 UI 套件中包含 Tailwind class（如公司內部元件庫），自動偵測可能無法涵蓋 `node_modules` 中的檔案，需手動加入。

```css
@import "tailwindcss";

/* 掃描公司內部 UI 套件的原始碼 */
@source "../node_modules/@company/ui-lib/src/**/*.vue";
@source "../node_modules/@company/ui-lib/src/**/*.ts";

/* 掃描特定第三方元件庫 */
@source "../node_modules/@headlessui/vue/dist/**/*.js";
```

### Inline Source：動態 Class 字串

當 class 名稱是在 JavaScript 中動態組合的，Tailwind 的靜態掃描無法辨識，可使用 `@source inline()` 明確列出需要保留的 class。

```css
@import "tailwindcss";

/* 動態產生的狀態色彩 class — 確保不被 tree-shake */
@source inline("
  text-red-500 bg-red-50 border-red-200
  text-yellow-500 bg-yellow-50 border-yellow-200
  text-green-500 bg-green-50 border-green-200
  text-blue-500 bg-blue-50 border-blue-200
");
```

```typescript
// utils/statusColors.ts
// 這些 class 是動態組合的，需搭配 @source inline() 確保產出
type Status = 'error' | 'warning' | 'success' | 'info'

const statusMap: Record<Status, { text: string; bg: string; border: string }> = {
  error:   { text: 'text-red-500',    bg: 'bg-red-50',    border: 'border-red-200' },
  warning: { text: 'text-yellow-500', bg: 'bg-yellow-50', border: 'border-yellow-200' },
  success: { text: 'text-green-500',  bg: 'bg-green-50',  border: 'border-green-200' },
  info:    { text: 'text-blue-500',   bg: 'bg-blue-50',   border: 'border-blue-200' },
}

export function getStatusClasses(status: Status) {
  return statusMap[status]
}
```

### 完整配置範例

```css
/* assets/css/main.css — Tailwind v4 完整 @source 配置 */
@import "tailwindcss";

/* ===== Content Source 設定 ===== */

/* 應用程式原始碼（通常自動偵測已涵蓋，此處為明確宣告） */
@source "../components/**/*.vue";
@source "../pages/**/*.vue";
@source "../layouts/**/*.vue";
@source "../composables/**/*.ts";
@source "../utils/**/*.ts";

/* 第三方套件 */
@source "../node_modules/@company/ui-lib/src/**/*.vue";

/* 動態 class safelist */
@source inline("
  text-red-500 bg-red-50 border-red-200
  text-yellow-500 bg-yellow-50 border-yellow-200
  text-green-500 bg-green-50 border-green-200
  text-blue-500 bg-blue-50 border-blue-200
");

/* ===== 設計 Token ===== */
@theme {
  /* ...（設計 Token 定義） */
}
```

### `@source` vs v3 `content` 對照

| 功能 | Tailwind v3 (`tailwind.config.ts`) | Tailwind v4 (CSS) |
|------|-------------------------------------|---------------------|
| 掃描路徑 | `content: ['./components/**/*.vue']` | `@source "../components/**/*.vue"` |
| 掃描第三方 | `content: ['./node_modules/...']` | `@source "../node_modules/..."` |
| Safelist | `safelist: ['text-red-500', ...]` | `@source inline("text-red-500 ...")` |
| 自動偵測 | 無（必須手動配置） | 預設啟用（`@source` 為補充） |
