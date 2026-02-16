# 實戰範例：完整 CRUD 模式

## 概述

以「文章管理」為例，示範標準的 CRUD 實作模式，包含型別定義、API、composable、元件、頁面、測試。

---

## 型別定義

```typescript
// types/post.ts
export interface Post {
  id: number
  title: string
  slug: string
  content: string
  excerpt: string
  status: 'draft' | 'published' | 'archived'
  authorId: number
  author?: { id: number; name: string }
  tags: string[]
  createdAt: string
  updatedAt: string
}

export interface CreatePostInput {
  title: string
  content: string
  excerpt?: string
  status?: Post['status']
  tags?: string[]
}

export interface UpdatePostInput extends Partial<CreatePostInput> {}

export interface PostFilters {
  status?: Post['status']
  tag?: string
  search?: string
  page?: number
  perPage?: number
}
```

## Server API

```typescript
// server/api/posts/index.get.ts
import { z } from 'zod'
import type { Post } from '~/types/post'

interface PostListResponse {
  data: Post[]
  meta: { total: number; page: number; perPage: number; totalPages: number }
}

const querySchema = z.object({
  status: z.enum(['draft', 'published', 'archived']).optional(),
  tag: z.string().optional(),
  search: z.string().optional(),
  page: z.coerce.number().positive().default(1),
  perPage: z.coerce.number().min(1).max(100).default(20)
})

export default defineEventHandler(async (event): Promise<PostListResponse> => {
  const result = querySchema.safeParse(getQuery(event))
  if (!result.success) {
    throw createError({ statusCode: 400, message: '查詢參數無效', data: result.error.flatten() })
  }
  const query = result.data
  const { page, perPage, status, tag, search } = query
  const offset = (page - 1) * perPage

  const where: Record<string, unknown> = {}
  if (status) where.status = status
  if (tag) where.tags = { has: tag }
  if (search) where.OR = [
    { title: { contains: search, mode: 'insensitive' } },
    { content: { contains: search, mode: 'insensitive' } }
  ]

  const [posts, total] = await Promise.all([
    db.post.findMany({
      where, skip: offset, take: perPage,
      orderBy: { createdAt: 'desc' },
      include: { author: { select: { id: true, name: true } } }
    }),
    db.post.count({ where })
  ])

  return {
    data: posts,
    meta: { total, page, perPage, totalPages: Math.ceil(total / perPage) }
  }
})
```

```typescript
// server/api/posts/index.post.ts
import { z } from 'zod'
import type { Post } from '~/types/post'

const createSchema = z.object({
  title: z.string().min(1, '標題不可為空').max(200),
  content: z.string().min(1, '內容不可為空'),
  excerpt: z.string().max(500).optional(),
  status: z.enum(['draft', 'published']).default('draft'),
  tags: z.array(z.string()).default([])
})

export default defineEventHandler(async (event): Promise<{ data: Post }> => {
  const auth = event.context.auth
  if (!auth) throw createError({ statusCode: 401, message: '未認證' })
  const { userId } = auth
  const result = createSchema.safeParse(await readBody(event))
  if (!result.success) {
    throw createError({ statusCode: 400, message: '輸入驗證失敗', data: result.error.flatten() })
  }
  const body = result.data

  // generateSlug 定義在 server/utils/slug.ts（Nitro 自動匯入）
  const slug = generateSlug(body.title)
  const post = await db.post.create({
    data: {
      ...body,
      slug,
      excerpt: body.excerpt ?? body.content.slice(0, 150),
      authorId: userId
    },
    include: { author: { select: { id: true, name: true } } }
  })

  setResponseStatus(event, 201)
  return { data: post }
})
```

```typescript
// server/api/posts/[id].put.ts
import { z } from 'zod'
import type { Post } from '~/types/post'

const updateSchema = z.object({
  title: z.string().min(1).max(200).optional(),
  content: z.string().min(1).optional(),
  excerpt: z.string().max(500).optional(),
  status: z.enum(['draft', 'published', 'archived']).optional(),
  tags: z.array(z.string()).optional()
})

export default defineEventHandler(async (event): Promise<{ data: Post }> => {
  const id = Number(getRouterParam(event, 'id'))
  if (Number.isNaN(id)) throw createError({ statusCode: 400, message: '無效的文章 ID' })
  const auth = event.context.auth
  if (!auth) throw createError({ statusCode: 401, message: '未認證' })
  const { userId } = auth
  const result = updateSchema.safeParse(await readBody(event))
  if (!result.success) {
    throw createError({ statusCode: 400, message: '輸入驗證失敗', data: result.error.flatten() })
  }
  const body = result.data

  const existing = await db.post.findUnique({ where: { id } })
  if (!existing) throw createError({ statusCode: 404, message: '文章不存在' })
  if (existing.authorId !== userId) throw createError({ statusCode: 403, message: '無權編輯' })

  const post = await db.post.update({
    where: { id },
    data: { ...body, updatedAt: new Date() },
    include: { author: { select: { id: true, name: true } } }
  })

  return { data: post }
})
```

```typescript
// server/api/posts/[id].delete.ts
export default defineEventHandler(async (event) => {
  const id = Number(getRouterParam(event, 'id'))
  if (Number.isNaN(id)) throw createError({ statusCode: 400, message: '無效的文章 ID' })
  const auth = event.context.auth
  if (!auth) throw createError({ statusCode: 401, message: '未認證' })
  const { userId, role } = auth

  const existing = await db.post.findUnique({ where: { id } })
  if (!existing) throw createError({ statusCode: 404, message: '文章不存在' })
  if (existing.authorId !== userId && role !== 'admin') {
    throw createError({ statusCode: 403, message: '無權刪除' })
  }

  await db.post.delete({ where: { id } })
  return sendNoContent(event)
})
```

## Toast 通知 Composable

```typescript
// composables/useToast.ts
// 簡易通知 composable（實際專案中建議搭配 UI 框架如 Headless UI 或第三方套件）
export function useToast() {
  const toast = useState<{ message: string; type: 'success' | 'error' } | null>('toast', () => null)

  function success(message: string) {
    toast.value = { message, type: 'success' }
    setTimeout(() => { toast.value = null }, 3000)
  }

  function error(message: string) {
    toast.value = { message, type: 'error' }
    setTimeout(() => { toast.value = null }, 5000)
  }

  return { toast: readonly(toast), success, error }
}
```

## Composable

```typescript
// composables/usePosts.ts
import type { Post, PostFilters, CreatePostInput, UpdatePostInput } from '~/types/post'
import type { ApiResponse } from '~/types/api'

export async function usePosts(initialFilters?: PostFilters) {
  const filters = ref<PostFilters>({
    page: 1,
    perPage: 20,
    ...initialFilters
  })

  // 使用固定 key 'posts' 作為列表資料的單一來源（singleton pattern）
  // 所有使用 usePosts() 的元件共享同一份快取資料
  // 若需要多個獨立的列表實例，可傳入自訂 key：
  // const { data } = await useAsyncData(key ?? 'posts', ...)
  const { data, status, error, refresh } = await useAsyncData(
    'posts',
    () => $fetch<ApiResponse<Post[]>>('/api/posts', { query: filters.value }),
    {
      watch: [filters],
      default: () => ({ data: [], meta: { total: 0, page: 1, perPage: 20, totalPages: 0 } })
    }
  )

  const posts = computed(() => data.value?.data ?? [])
  const meta = computed(() => data.value?.meta)
  const isLoading = computed(() => status.value === 'pending')

  function setPage(page: number) {
    filters.value = { ...filters.value, page }
  }

  function setFilter(filterKey: keyof PostFilters, value: PostFilters[keyof PostFilters]) {
    filters.value = { ...filters.value, [filterKey]: value, page: 1 }
  }

  return {
    posts,
    meta,
    isLoading,
    error,
    filters,
    refresh,
    setPage,
    setFilter
  }
}

export function usePost(id: number | string) {
  return useAsyncData(`post-${id}`,
    () => $fetch<{ data: Post }>(`/api/posts/${id}`).then(r => r.data)
  )
}

export function usePostMutations() {
  const isSubmitting = ref(false)

  async function createPost(input: CreatePostInput): Promise<Post> {
    isSubmitting.value = true
    try {
      const { data } = await $fetch<{ data: Post }>('/api/posts', {
        method: 'POST',
        body: input
      })
      await refreshNuxtData('posts')
      return data
    } finally {
      isSubmitting.value = false
    }
  }

  async function updatePost(id: number, input: UpdatePostInput): Promise<Post> {
    isSubmitting.value = true
    try {
      const { data } = await $fetch<{ data: Post }>(`/api/posts/${id}`, {
        method: 'PUT',
        body: input
      })
      await Promise.all([
        refreshNuxtData('posts'),
        refreshNuxtData(`post-${id}`)
      ])
      return data
    } finally {
      isSubmitting.value = false
    }
  }

  async function deletePost(id: number): Promise<void> {
    isSubmitting.value = true
    try {
      await $fetch(`/api/posts/${id}`, { method: 'DELETE' })
      await refreshNuxtData('posts')
    } finally {
      isSubmitting.value = false
    }
  }

  return { createPost, updatePost, deletePost, isSubmitting }
}
```

## 列表頁面

```vue
<!-- pages/dashboard/posts/index.vue -->
<script setup lang="ts">
definePageMeta({ middleware: 'auth', requiresAuth: true, layout: 'dashboard' })

useSeoMeta({ title: '文章管理' })

const { posts, meta, isLoading, filters, setPage, setFilter } = await usePosts()
const { deletePost, isSubmitting } = usePostMutations()

const deleteConfirm = ref<number | null>(null)

async function handleDelete() {
  if (!deleteConfirm.value) return
  await deletePost(deleteConfirm.value)
  deleteConfirm.value = null
}
</script>

<template>
  <div class="space-y-6">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl font-bold">文章管理</h1>
      <NuxtLink to="/dashboard/posts/create">
        <BaseButton variant="primary">新增文章</BaseButton>
      </NuxtLink>
    </div>

    <!-- 篩選列 -->
    <div class="flex gap-4">
      <input
        :value="filters.search"
        @input="setFilter('search', ($event.target as HTMLInputElement).value)"
        placeholder="搜尋文章..."
        class="px-4 py-2 border rounded-lg flex-1"
      />
      <select
        :value="filters.status"
        @change="setFilter('status', ($event.target as HTMLSelectElement).value || undefined)"
        class="px-4 py-2 border rounded-lg"
      >
        <option value="">所有狀態</option>
        <option value="draft">草稿</option>
        <option value="published">已發佈</option>
        <option value="archived">已封存</option>
      </select>
    </div>

    <!-- 列表 -->
    <div v-if="isLoading" class="space-y-4">
      <div v-for="i in 5" :key="i" class="h-20 bg-gray-100 animate-pulse rounded-lg" />
    </div>

    <div v-else-if="posts.length === 0" class="text-center py-12 text-gray-500">
      尚無文章
    </div>

    <div v-else class="space-y-3">
      <div
        v-for="post in posts"
        :key="post.id"
        class="p-4 bg-white rounded-lg border hover:shadow-md transition-shadow"
      >
        <div class="flex items-start justify-between">
          <div>
            <NuxtLink :to="`/dashboard/posts/${post.id}`" class="text-lg font-semibold hover:text-blue-600">
              {{ post.title }}
            </NuxtLink>
            <p class="text-sm text-gray-500 mt-1">{{ post.excerpt }}</p>
            <div class="flex items-center gap-2 mt-2">
              <span :class="[
                'px-2 py-0.5 text-xs rounded-full',
                post.status === 'published' ? 'bg-green-100 text-green-700' :
                post.status === 'draft' ? 'bg-yellow-100 text-yellow-700' :
                'bg-gray-100 text-gray-700'
              ]">
                {{ post.status === 'published' ? '已發佈' : post.status === 'draft' ? '草稿' : '已封存' }}
              </span>
              <span class="text-xs text-gray-400">{{ post.createdAt.slice(0, 10) }}</span>
            </div>
          </div>
          <div class="flex gap-2">
            <NuxtLink :to="`/dashboard/posts/${post.id}/edit`">
              <BaseButton variant="ghost" size="sm">編輯</BaseButton>
            </NuxtLink>
            <BaseButton variant="danger" size="sm" @click="deleteConfirm = post.id">
              刪除
            </BaseButton>
          </div>
        </div>
      </div>
    </div>

    <!-- 分頁 -->
    <div v-if="meta && meta.totalPages > 1" class="flex justify-center gap-2">
      <BaseButton
        v-for="page in meta.totalPages"
        :key="page"
        :variant="page === meta.page ? 'primary' : 'secondary'"
        size="sm"
        @click="setPage(page)"
      >
        {{ page }}
      </BaseButton>
    </div>

    <!-- 刪除確認 Modal -->
    <Teleport to="body">
      <div
        v-if="deleteConfirm"
        class="fixed inset-0 bg-black/50 flex items-center justify-center z-50"
        role="dialog"
        aria-modal="true"
        aria-labelledby="delete-confirm-title"
        @click.self="deleteConfirm = null"
        @keydown.escape="deleteConfirm = null"
      >
        <div class="bg-white p-6 rounded-xl max-w-sm w-full mx-4">
          <h3 id="delete-confirm-title" class="text-lg font-bold">確認刪除</h3>
          <p class="text-gray-600 mt-2">此操作無法復原，確定要刪除這篇文章嗎？</p>
          <div class="flex justify-end gap-3 mt-6">
            <BaseButton variant="secondary" @click="deleteConfirm = null">取消</BaseButton>
            <BaseButton variant="danger" :loading="isSubmitting" @click="handleDelete">確認刪除</BaseButton>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>
```

## 編輯頁面

```vue
<!-- pages/dashboard/posts/[id]/edit.vue -->
<script setup lang="ts">
definePageMeta({ middleware: 'auth', requiresAuth: true, layout: 'dashboard' })

const route = useRoute()
const rawId = Array.isArray(route.params.id) ? route.params.id[0] : route.params.id
const id = Number(rawId)

if (Number.isNaN(id)) {
  throw createError({ statusCode: 400, message: '無效的文章 ID', fatal: true })
}

const { data: post, error } = await usePost(id)

if (error.value || !post.value) {
  throw createError({ statusCode: 404, message: '文章不存在', fatal: true })
}

useSeoMeta({ title: `編輯：${post.value.title}` })

const { updatePost, isSubmitting } = usePostMutations()

const form = ref({
  title: post.value.title,
  content: post.value.content,
  excerpt: post.value.excerpt,
  status: post.value.status,
  tags: post.value.tags
})

const { success: showSuccess, error: showError } = useToast()

async function handleSubmit() {
  try {
    await updatePost(id, form.value)
    showSuccess('文章更新成功')
    await navigateTo('/dashboard/posts')
  } catch (err: unknown) {
    console.error('更新文章失敗', err)
    // isNuxtFetchError 定義於 utils/typeGuards.ts（見 auth-system 範例）
    const message = isNuxtFetchError(err) && err.data && typeof err.data === 'object'
      ? (err.data as { message?: string }).message
      : undefined
    showError(message ?? '儲存失敗，請稍後再試')
  }
}
</script>

<template>
  <form @submit.prevent="handleSubmit" class="max-w-3xl space-y-6">
    <h1 class="text-2xl font-bold">編輯文章</h1>

    <div>
      <label class="block text-sm font-medium mb-1">標題</label>
      <input v-model="form.title" required class="w-full px-4 py-3 border rounded-lg" />
    </div>

    <div>
      <label class="block text-sm font-medium mb-1">摘要</label>
      <textarea v-model="form.excerpt" rows="2" class="w-full px-4 py-3 border rounded-lg" />
    </div>

    <div>
      <label class="block text-sm font-medium mb-1">內容</label>
      <textarea v-model="form.content" rows="15" required class="w-full px-4 py-3 border rounded-lg font-mono" />
    </div>

    <div>
      <label class="block text-sm font-medium mb-1">狀態</label>
      <select v-model="form.status" class="px-4 py-3 border rounded-lg">
        <option value="draft">草稿</option>
        <option value="published">發佈</option>
        <option value="archived">封存</option>
      </select>
    </div>

    <div class="flex gap-3">
      <BaseButton type="submit" variant="primary" :loading="isSubmitting">
        儲存變更
      </BaseButton>
      <NuxtLink to="/dashboard/posts">
        <BaseButton variant="secondary" type="button">取消</BaseButton>
      </NuxtLink>
    </div>
  </form>
</template>
```
