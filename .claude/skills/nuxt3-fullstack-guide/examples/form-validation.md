# 實戰範例：表單驗證系統

## 概述

型別安全的表單驗證 composable，搭配 zod schema，支援即時驗證、欄位級錯誤、送出狀態管理。

---

## 通用表單 Composable

```typescript
// composables/useFormValidation.ts
import { z, type ZodSchema, type ZodIssue } from 'zod'

interface UseFormOptions<T extends ZodSchema> {
  schema: T
  initialValues: z.infer<T>
  onSubmit: (values: z.infer<T>) => Promise<void>
  validateOnBlur?: boolean
}

export function useFormValidation<T extends ZodSchema>(options: UseFormOptions<T>) {
  type FormData = z.infer<T>

  const values = ref<FormData>({ ...options.initialValues })
  const errors = ref<Record<string, string>>({})
  const touched = ref<Record<string, boolean>>({})
  const isSubmitting = ref(false)
  const submitError = ref<string | null>(null)
  const submitCount = ref(0)

  // ⚠️ JSON.stringify 比較的限制：
  // 1. 屬性順序不同會被判定為不同（{a:1, b:2} vs {b:2, a:1}）
  // 2. 無法處理 undefined、Function、Symbol（會被忽略）
  // 3. Date 物件會被轉為字串，比較結果可能不如預期
  // 4. 大型物件有效能影響
  //
  // 進階替代方案（逐欄位比較）：
  // const isDirty = computed(() =>
  //   (Object.keys(formData.value) as Array<keyof typeof formData.value>).some(
  //     key => formData.value[key] !== initialSnapshot[key]
  //   )
  // )
  const isDirty = computed(() =>
    JSON.stringify(values.value) !== JSON.stringify(options.initialValues)
  )

  const isValid = computed(() => Object.keys(errors.value).length === 0)

  const hasErrors = computed(() => Object.keys(errors.value).length > 0)

  function validate(): boolean {
    const result = options.schema.safeParse(values.value)
    if (result.success) {
      errors.value = {}
      return true
    }

    const newErrors: Record<string, string> = {}
    for (const issue of result.error.issues) {
      const path = issue.path.join('.')
      if (!newErrors[path]) {
        newErrors[path] = issue.message
      }
    }
    errors.value = newErrors
    return false
  }

  function validateField(field: string) {
    const result = options.schema.safeParse(values.value)
    if (result.success) {
      delete errors.value[field]
      return
    }

    const fieldError = result.error.issues.find(
      (issue: ZodIssue) => issue.path.join('.') === field
    )

    if (fieldError) {
      errors.value[field] = fieldError.message
    } else {
      delete errors.value[field]
    }
  }

  function handleBlur(field: string) {
    touched.value[field] = true
    if (options.validateOnBlur !== false) {
      validateField(field)
    }
  }

  function getError(field: string): string | undefined {
    return touched.value[field] ? errors.value[field] : undefined
  }

  function resetForm() {
    values.value = { ...options.initialValues }
    errors.value = {}
    touched.value = {}
    submitError.value = null
    isSubmitting.value = false
  }

  function setFieldValue(field: string, value: unknown) {
    ;(values.value as Record<string, unknown>)[field] = value
    if (touched.value[field]) {
      validateField(field)
    }
  }

  async function handleSubmit() {
    submitCount.value++
    // 標記所有欄位為 touched
    for (const key of Object.keys(values.value as Record<string, unknown>)) {
      touched.value[key] = true
    }

    if (!validate()) return

    isSubmitting.value = true
    submitError.value = null

    try {
      await options.onSubmit(values.value)
    } catch (error: unknown) {
      const err = (typeof error === 'object' && error !== null)
        ? (error as { message?: string; data?: Record<string, unknown> })
        : { message: undefined, data: undefined }
      submitError.value = err.message ?? '送出失敗，請稍後再試'
      // 若伺服器回傳欄位錯誤，設定到對應欄位
      if (err.data && typeof err.data === 'object') {
        for (const [field, messages] of Object.entries(err.data)) {
          if (Array.isArray(messages) && messages.length > 0 && typeof messages[0] === 'string') {
            errors.value[field] = messages[0]
          }
        }
      }
    } finally {
      isSubmitting.value = false
    }
  }

  return {
    values,
    errors,
    touched,
    isSubmitting,
    isDirty,
    isValid,
    hasErrors,
    submitError,
    submitCount,
    validate,
    validateField,
    handleBlur,
    handleSubmit,
    getError,
    resetForm,
    setFieldValue
  }
}
```

## 使用範例：註冊表單

```vue
<!-- components/feature/auth/RegisterForm.vue -->
<script setup lang="ts">
import { z } from 'zod'

const registerSchema = z.object({
  name: z.string()
    .min(2, '名稱至少 2 個字元')
    .max(50, '名稱不超過 50 個字元'),
  email: z.string()
    .email('請輸入有效的電子郵件'),
  password: z.string()
    .min(8, '密碼至少 8 個字元')
    .regex(/[A-Z]/, '密碼需含至少一個大寫字母')
    .regex(/[0-9]/, '密碼需含至少一個數字'),
  confirmPassword: z.string()
}).refine((data) => data.password === data.confirmPassword, {
  message: '密碼不一致',
  path: ['confirmPassword']
})

const {
  values,
  isSubmitting,
  isDirty,
  hasErrors,
  submitError,
  handleBlur,
  handleSubmit,
  getError
} = useFormValidation({
  schema: registerSchema,
  initialValues: {
    name: '',
    email: '',
    password: '',
    confirmPassword: ''
  },
  onSubmit: async (data) => {
    await $fetch('/api/auth/register', {
      method: 'POST',
      body: { name: data.name, email: data.email, password: data.password }
    })
    await navigateTo('/login?registered=true')
  }
})
</script>

<template>
  <form @submit.prevent="handleSubmit" class="space-y-5" novalidate>
    <div v-if="submitError" class="p-3 bg-red-50 text-red-600 rounded-lg text-sm">
      {{ submitError }}
    </div>

    <!-- 名稱 -->
    <div>
      <label for="name" class="block text-sm font-medium text-gray-700 mb-1">名稱</label>
      <input
        id="name"
        v-model="values.name"
        type="text"
        @blur="handleBlur('name')"
        :class="[
          'w-full px-4 py-3 border rounded-lg transition-colors',
          getError('name') ? 'border-red-500 focus:ring-red-500' : 'border-gray-300 focus:ring-blue-500'
        ]"
      />
      <p v-if="getError('name')" class="mt-1 text-sm text-red-600">{{ getError('name') }}</p>
    </div>

    <!-- 電子郵件 -->
    <div>
      <label for="email" class="block text-sm font-medium text-gray-700 mb-1">電子郵件</label>
      <input
        id="email"
        v-model="values.email"
        type="email"
        @blur="handleBlur('email')"
        :class="[
          'w-full px-4 py-3 border rounded-lg transition-colors',
          getError('email') ? 'border-red-500 focus:ring-red-500' : 'border-gray-300 focus:ring-blue-500'
        ]"
      />
      <p v-if="getError('email')" class="mt-1 text-sm text-red-600">{{ getError('email') }}</p>
    </div>

    <!-- 密碼 -->
    <div>
      <label for="password" class="block text-sm font-medium text-gray-700 mb-1">密碼</label>
      <input
        id="password"
        v-model="values.password"
        type="password"
        @blur="handleBlur('password')"
        :class="[
          'w-full px-4 py-3 border rounded-lg transition-colors',
          getError('password') ? 'border-red-500 focus:ring-red-500' : 'border-gray-300 focus:ring-blue-500'
        ]"
      />
      <p v-if="getError('password')" class="mt-1 text-sm text-red-600">{{ getError('password') }}</p>
    </div>

    <!-- 確認密碼 -->
    <div>
      <label for="confirmPassword" class="block text-sm font-medium text-gray-700 mb-1">確認密碼</label>
      <input
        id="confirmPassword"
        v-model="values.confirmPassword"
        type="password"
        @blur="handleBlur('confirmPassword')"
        :class="[
          'w-full px-4 py-3 border rounded-lg transition-colors',
          getError('confirmPassword') ? 'border-red-500 focus:ring-red-500' : 'border-gray-300 focus:ring-blue-500'
        ]"
      />
      <p v-if="getError('confirmPassword')" class="mt-1 text-sm text-red-600">{{ getError('confirmPassword') }}</p>
    </div>

    <BaseButton
      type="submit"
      variant="primary"
      :loading="isSubmitting"
      :disabled="!isDirty || hasErrors"
      class="w-full"
    >
      註冊帳號
    </BaseButton>
  </form>
</template>
```

## 可複用表單欄位元件

```vue
<!-- components/base/BaseFormField.vue -->
<script setup lang="ts">
interface Props {
  label: string
  name: string
  error?: string
  required?: boolean
  hint?: string
}

defineProps<Props>()
</script>

<template>
  <div>
    <label :for="name" class="block text-sm font-medium text-gray-700 mb-1">
      {{ label }}
      <span v-if="required" class="text-red-500">*</span>
    </label>
    <slot />
    <p v-if="hint && !error" class="mt-1 text-xs text-gray-400">{{ hint }}</p>
    <p v-if="error" class="mt-1 text-sm text-red-600" role="alert">{{ error }}</p>
  </div>
</template>
```

使用方式：

```vue
<BaseFormField label="電子郵件" name="email" :error="getError('email')" required>
  <input
    id="email"
    v-model="values.email"
    type="email"
    @blur="handleBlur('email')"
    class="w-full px-4 py-3 border rounded-lg"
  />
</BaseFormField>
```

## 測試

```typescript
// tests/unit/composables/useFormValidation.test.ts
import { z } from 'zod'
import { useFormValidation } from '~/composables/useFormValidation'

const testSchema = z.object({
  name: z.string().min(2, '至少 2 字元'),
  email: z.string().email('無效的 email')
})

describe('useFormValidation', () => {
  function createForm(onSubmit = vi.fn()) {
    return useFormValidation({
      schema: testSchema,
      initialValues: { name: '', email: '' },
      onSubmit
    })
  }

  it('初始狀態正確', () => {
    const form = createForm()
    expect(form.isDirty.value).toBe(false)
    expect(form.isSubmitting.value).toBe(false)
    expect(form.hasErrors.value).toBe(false)
  })

  it('修改值後 isDirty 為 true', () => {
    const form = createForm()
    form.values.value.name = '測試'
    expect(form.isDirty.value).toBe(true)
  })

  it('blur 後觸發欄位驗證', () => {
    const form = createForm()
    form.handleBlur('name')
    expect(form.getError('name')).toBe('至少 2 字元')
  })

  it('值正確時無錯誤', () => {
    const form = createForm()
    form.values.value.name = '測試名稱'
    form.values.value.email = 'test@test.com'
    form.handleBlur('name')
    form.handleBlur('email')
    expect(form.getError('name')).toBeUndefined()
    expect(form.getError('email')).toBeUndefined()
  })

  it('submit 時全部驗證', async () => {
    const onSubmit = vi.fn()
    const form = createForm(onSubmit)
    await form.handleSubmit()
    expect(onSubmit).not.toHaveBeenCalled()
    expect(form.hasErrors.value).toBe(true)
  })

  it('驗證通過後呼叫 onSubmit', async () => {
    const onSubmit = vi.fn()
    const form = createForm(onSubmit)
    form.values.value.name = '測試名稱'
    form.values.value.email = 'test@test.com'
    await form.handleSubmit()
    expect(onSubmit).toHaveBeenCalledWith({
      name: '測試名稱',
      email: 'test@test.com'
    })
  })

  it('resetForm 重設所有狀態', () => {
    const form = createForm()
    form.values.value.name = '測試'
    form.handleBlur('name')
    form.resetForm()
    expect(form.values.value.name).toBe('')
    expect(form.isDirty.value).toBe(false)
    expect(form.hasErrors.value).toBe(false)
  })
})
```
