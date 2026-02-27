# DevOps 與可觀測性指南

## 目錄

1. [Deployment Strategies](#deployment-strategies)
2. [Rollback Strategy](#rollback-strategy)
3. [CI/CD Architecture](#cicd-architecture)
4. [Infrastructure as Code](#infrastructure-as-code)
5. [Observability — Metrics](#observability--metrics)
6. [Observability — Distributed Tracing](#observability--distributed-tracing)
7. [Observability — Structured Logging](#observability--structured-logging)
8. [Alerting 與 SLO/SLA](#alerting-與-slosla)

---

## Deployment Strategies

### Blue-Green Deployment

透過兩組完全相同的環境實現零停機部署，流量在驗證通過後一次切換。

```typescript
// server/api/deployment/health.get.ts — 部署驗證端點
export default defineEventHandler(async () => {
  const checks = await Promise.allSettled([
    checkDatabase(), checkRedis(), checkExternalAPIs(),
  ])
  const status = checks.every(c => c.status === 'fulfilled') ? 'healthy' : 'degraded'
  return {
    status,
    version: useRuntimeConfig().public.appVersion,
    environment: process.env.DEPLOYMENT_SLOT, // 'blue' | 'green'
    timestamp: new Date().toISOString(),
  }
})
```

```yaml
# kubernetes/blue-green.yaml — Service selector 切換 slot: green <-> blue
apiVersion: v1
kind: Service
metadata: { name: nuxt-app }
spec:
  selector: { app: nuxt-app, slot: green }   # 切換時改為 blue
  ports: [{ port: 80, targetPort: 3000 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: nuxt-app-green }
spec:
  replicas: 3
  selector: { matchLabels: { app: nuxt-app, slot: green } }
  template:
    metadata: { labels: { app: nuxt-app, slot: green } }
    spec:
      containers:
        - name: nuxt
          image: registry.example.com/nuxt-app:v2.1.0
          ports: [{ containerPort: 3000 }]
          readinessProbe:
            httpGet: { path: /api/deployment/health, port: 3000 }
            initialDelaySeconds: 5
```

### Canary Release

漸進式流量切換：1% -> 10% -> 50% -> 100%，每階段觀察指標後再推進。

```yaml
# kubernetes/canary-virtualservice.yaml (Istio)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nuxt-app
spec:
  hosts: [nuxt-app.example.com]
  http:
    - route:
        - destination: { host: nuxt-app-stable, port: { number: 80 } }
          weight: 99            # 穩定版流量
        - destination: { host: nuxt-app-canary, port: { number: 80 } }
          weight: 1             # Canary (1% -> 10% -> 50% -> 100%)
```

### Rolling Update

```yaml
# kubernetes/rolling-update.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuxt-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最多同時多 1 個 Pod
      maxUnavailable: 0    # 確保零停機
  template:
    spec:
      containers:
        - name: nuxt
          image: registry.example.com/nuxt-app:v2.1.0
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30
```

### Feature Flags

```typescript
// app.config.ts — Runtime feature toggle
export default defineAppConfig({
  features: {
    newCheckoutFlow: false,
    experimentalSearch: false,
    maintenanceMode: false,
  },
})

// composables/useFeatureFlag.ts
export function useFeatureFlag(flag: keyof AppConfig['features']): boolean {
  return useAppConfig().features[flag] ?? false
}
```

```vue
<template>
  <NewCheckout v-if="useFeatureFlag('newCheckoutFlow')" />
  <LegacyCheckout v-else />
</template>
```

---

## Rollback Strategy

### 自動化回滾觸發條件（Flagger）

```yaml
# kubernetes/canary-analysis.yaml — Flagger 自動化 Canary 分析
apiVersion: flagger.app/v1beta1
kind: Canary
metadata: { name: nuxt-app }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: nuxt-app }
  analysis:
    interval: 60s
    threshold: 3            # 連續 3 次失敗即回滾
    maxWeight: 50
    stepWeight: 10           # 每次增加 10% 流量
    metrics:
      - name: request-success-rate
        thresholdRange: { min: 99 }   # 成功率 < 99% 即回滾
        interval: 60s
      - name: request-duration
        thresholdRange: { max: 500 }  # p99 > 500ms 即回滾
        interval: 60s
```

### Database Migration Rollback（Prisma）

```bash
npx prisma migrate status                                             # 查看遷移歷史
npx prisma migrate resolve --rolled-back 20240101120000_add_orders    # 標記為已回滾
npx prisma db execute --file ./prisma/rollbacks/20240101120000.sql    # 執行反向 SQL
```

```sql
-- prisma/rollbacks/20240101120000.sql
DROP TABLE IF EXISTS "Order";
ALTER TABLE "User" DROP COLUMN IF EXISTS "order_count";
```

### Docker Image Tagging Strategy

```bash
# 語義化版本 + Git SHA + 時間戳
IMAGE_TAG="v${VERSION}-${GIT_SHA:0:7}-$(date +%Y%m%d%H%M%S)"
# 範例: v2.1.0-a1b2c3d-20260215143000

docker build -t registry.example.com/nuxt-app:${IMAGE_TAG} .
docker tag registry.example.com/nuxt-app:${IMAGE_TAG} registry.example.com/nuxt-app:latest

# 快速回滾：指向前一個已知穩定版本
kubectl set image deployment/nuxt-app nuxt=registry.example.com/nuxt-app:v2.0.9-f4e5d6c
```

---

## CI/CD Architecture

### 多階段 Pipeline（GitHub Actions）

Pipeline 流程：`lint-and-typecheck` -> `unit-test` -> `build` -> `integration-test` -> `docker-build` -> `deploy-staging` -> `deploy-production`

```yaml
# .github/workflows/ci-cd.yaml
name: CI/CD Pipeline
on:
  push: { branches: [main, develop] }
  pull_request: { branches: [main] }
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile && pnpm lint && pnpm typecheck
  unit-test:
    needs: lint-and-typecheck
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile && pnpm test:unit --coverage
  build:
    needs: unit-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile && pnpm build
      - uses: actions/upload-artifact@v4
        with: { name: build-output, path: .output/ }
  integration-test:
    needs: build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: test, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - uses: actions/download-artifact@v4
        with: { name: build-output, path: .output/ }
      - run: pnpm test:e2e
        env: { DATABASE_URL: 'postgresql://test:test@localhost:5432/test' }
  docker-build:
    needs: integration-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: '${{ github.actor }}', password: '${{ secrets.GITHUB_TOKEN }}' }
      - uses: docker/build-push-action@v5
        with: { push: true, tags: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}' }
  deploy-staging:
    needs: docker-build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: kubectl set image deployment/nuxt-app nuxt=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n staging
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: { name: production, url: 'https://app.example.com' }
    steps:
      - run: kubectl set image deployment/nuxt-app nuxt=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} -n production
```

### Monorepo CI（Affected Packages Only）

使用 `dorny/paths-filter` 偵測變更範圍，僅觸發受影響的 package 測試：

```yaml
# .github/workflows/monorepo-ci.yaml — 偵測變更後條件式執行
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      app: ${{ steps.filter.outputs.app }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            app:
              - 'apps/web/**'
              - 'packages/shared/**'
  test-app:
    needs: detect-changes
    if: needs.detect-changes.outputs.app == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm --filter @mono/web test
```

---

## Infrastructure as Code

### SST v3 (Ion)

```typescript
// sst.config.ts
/// <reference path="./.sst/platform/config.d.ts" />
export default $config({
  app(input) {
    return {
      name: 'nuxt-app',
      removal: input?.stage === 'production' ? 'retain' : 'remove',
      home: 'aws',
    }
  },
  async run() {
    const vpc = new sst.aws.Vpc('AppVpc')
    const database = new sst.aws.Postgres('Database', {
      vpc, scaling: { min: '0.5 ACU', max: '4 ACU' },
    })
    const bucket = new sst.aws.Bucket('Assets', { public: true })

    const nuxt = new sst.aws.Nuxt('Web', {
      link: [database, bucket],
      vpc,
      domain: { name: 'app.example.com', dns: sst.aws.dns() },
      environment: { DATABASE_URL: database.url, S3_BUCKET: bucket.name },
    })
    return { url: nuxt.url }
  },
})
```

### Cloudflare Wrangler

```toml
# wrangler.toml
name = "nuxt-app"
compatibility_date = "2025-12-01"
main = ".output/server/index.mjs"
assets = { directory = ".output/public" }
[vars]
APP_ENV = "production"
[[d1_databases]]
binding = "DB"
database_name = "nuxt-app-db"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
[[kv_namespaces]]
binding = "CACHE"
id = "xxxxxxxx"
[env.staging]
name = "nuxt-app-staging"
vars = { APP_ENV = "staging" }
```

### Docker Compose 開發環境

```yaml
# docker-compose.yaml
services:
  app:
    build: { context: ., dockerfile: Dockerfile, target: development }
    ports: ['3000:3000']
    volumes: ['.:/app', '/app/node_modules']
    environment:
      DATABASE_URL: postgresql://dev:dev@postgres:5432/nuxtapp
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres: { condition: service_healthy }
  postgres:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: nuxtapp, POSTGRES_USER: dev, POSTGRES_PASSWORD: dev }
    ports: ['5432:5432']
    healthcheck: { test: ['CMD-SHELL', 'pg_isready -U dev'], interval: 5s, retries: 5 }
  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
  prometheus:
    image: prom/prometheus:latest
    volumes: ['./infra/prometheus.yml:/etc/prometheus/prometheus.yml']
    ports: ['9090:9090']
  grafana:
    image: grafana/grafana:latest
    ports: ['3001:3000']
    environment: { GF_SECURITY_ADMIN_PASSWORD: admin }
```

---

## Observability — Metrics

### Nuxt Server 自訂指標

```typescript
// server/utils/metrics.ts
import { Counter, Histogram, Gauge, Registry } from 'prom-client'

const register = new Registry()

export const httpRequestCount = new Counter({
  name: 'nuxt_http_requests_total', help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'] as const, registers: [register],
})
export const httpRequestDuration = new Histogram({
  name: 'nuxt_http_request_duration_seconds', help: 'Request duration (s)',
  labelNames: ['method', 'route'] as const,
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5], registers: [register],
})
export const activeConnections = new Gauge({
  name: 'nuxt_active_connections', help: 'Active connections', registers: [register],
})
// Business Metrics
export const businessDAU = new Gauge({
  name: 'business_daily_active_users', help: 'DAU', registers: [register],
})
export const businessConversion = new Gauge({
  name: 'business_conversion_rate', help: 'Checkout conversion', registers: [register],
})
export { register as metricsRegistry }
```

```typescript
// server/middleware/metrics.ts
import { httpRequestCount, httpRequestDuration, activeConnections } from '../utils/metrics'

export default defineEventHandler((event) => {
  const start = performance.now()
  activeConnections.inc()
  event.node.res.on('finish', () => {
    const duration = (performance.now() - start) / 1000
    const route = getRequestURL(event).pathname
    const method = getMethod(event)
    httpRequestCount.inc({ method, route, status_code: String(event.node.res.statusCode) })
    httpRequestDuration.observe({ method, route }, duration)
    activeConnections.dec()
  })
})

// server/api/metrics.get.ts — Prometheus scrape 端點
import { metricsRegistry } from '../utils/metrics'
export default defineEventHandler(async (event) => {
  setResponseHeader(event, 'Content-Type', metricsRegistry.contentType)
  return metricsRegistry.metrics()
})
```

### Prometheus 配置

```yaml
# infra/prometheus.yml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'nuxt-app'
    metrics_path: '/api/metrics'
    static_configs:
      - targets: ['app:3000']
        labels: { environment: 'production' }
```

---

## Observability — Distributed Tracing

### OpenTelemetry 整合 Nuxt 3

```typescript
// server/plugins/otel.ts
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'
import { Resource } from '@opentelemetry/resources'
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from '@opentelemetry/semantic-conventions'

let sdk: NodeSDK | null = null
export default defineNitroPlugin((nitro) => {
  if (sdk) return
  sdk = new NodeSDK({
    resource: new Resource({
      [ATTR_SERVICE_NAME]: 'nuxt-app',
      [ATTR_SERVICE_VERSION]: useRuntimeConfig().public.appVersion ?? '0.0.0',
    }),
    traceExporter: new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318/v1/traces',
    }),
    instrumentations: [
      getNodeAutoInstrumentations({ '@opentelemetry/instrumentation-fs': { enabled: false } }),
    ],
  })
  sdk.start()
  nitro.hooks.hook('close', () => sdk?.shutdown())
})
```

### W3C Trace Context Propagation + withSpan 工具

```typescript
// server/utils/tracing.ts
import { trace, SpanStatusCode } from '@opentelemetry/api'
const tracer = trace.getTracer('nuxt-app')

export function withSpan<T>(name: string, fn: () => Promise<T>): Promise<T> {
  return tracer.startActiveSpan(name, async (span) => {
    try {
      const result = await fn()
      span.setStatus({ code: SpanStatusCode.OK })
      return result
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: String(error) })
      span.recordException(error as Error)
      throw error
    } finally { span.end() }
  })
}

// 使用範例: server/api/orders/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')!
  return withSpan('orders.getById', async () => {
    const order = await withSpan('db.findOrder', () =>
      prisma.order.findUniqueOrThrow({ where: { id }, include: { items: true } })
    )
    return withSpan('enrich.orderDetails', () => enrichOrderWithProductInfo(order))
  })
})
```

### Sentry Performance Monitoring

```typescript
// nuxt.config.ts — 加入 @sentry/nuxt/module
export default defineNuxtConfig({
  modules: ['@sentry/nuxt/module'],
  sentry: { sourceMapsUploadOptions: { org: 'my-org', project: 'nuxt-app' } },
  sourcemap: { client: 'hidden' },
})

// sentry.client.config.ts
import * as Sentry from '@sentry/nuxt'
Sentry.init({
  dsn: import.meta.env.NUXT_PUBLIC_SENTRY_DSN ?? '',
  tracesSampleRate: 0.2,                   // 20% 的請求產生 trace
  replaysOnErrorSampleRate: 1.0,           // 100% 錯誤回放
  integrations: [Sentry.replayIntegration()],
})

// sentry.server.config.ts
import * as Sentry from '@sentry/nuxt'
Sentry.init({ dsn: process.env.NUXT_PUBLIC_SENTRY_DSN ?? '', tracesSampleRate: 0.5 })
```

---

## Observability — Structured Logging

### 結構化日誌（JSON）與 Log Levels

```typescript
// server/utils/logger.ts
import { consola, createConsola } from 'consola'

interface LogContext {
  requestId?: string; userId?: string; action?: string
  duration?: number; [key: string]: unknown
}

const logger = createConsola({
  level: process.env.LOG_LEVEL === 'debug' ? 4 : 3,
  reporters: process.env.NODE_ENV === 'production'
    ? [{
        log: (logObj) => {
          process.stdout.write(JSON.stringify({
            timestamp: new Date().toISOString(), level: logObj.type,
            message: logObj.args.join(' '), service: 'nuxt-app',
            ...logObj.tag ? { tag: logObj.tag } : {},
          }) + '\n')
        },
      }]
    : undefined,
})

export function createLogger(tag: string) { return logger.withTag(tag) }

export function logWithContext(level: 'info' | 'warn' | 'error', msg: string, ctx: LogContext) {
  const entry = { timestamp: new Date().toISOString(), level, message: msg, service: 'nuxt-app', ...ctx }
  process.env.NODE_ENV === 'production'
    ? process.stdout.write(JSON.stringify(entry) + '\n')
    : consola[level](msg, ctx)
}
```

| Level | 用途 | 範例 |
|-------|------|------|
| ERROR | 需要立即處理的異常 | 資料庫連線失敗、第三方 API 中斷 |
| WARN  | 潛在問題但不影響主要功能 | 快取未命中、重試成功、即將到期的憑證 |
| INFO  | 關鍵業務事件 | 使用者登入、訂單建立、部署完成 |
| DEBUG | 開發階段除錯（生產環境關閉） | SQL 查詢、函式入出參數 |

### Correlation ID 追蹤

```typescript
// server/middleware/correlation-id.ts
import { randomUUID } from 'node:crypto'

export default defineEventHandler((event) => {
  const requestId = getRequestHeader(event, 'x-request-id') ?? randomUUID()
  event.context.requestId = requestId
  setResponseHeader(event, 'X-Request-Id', requestId)
})

// server/api/users/index.get.ts — 搭配 Correlation ID
export default defineEventHandler(async (event) => {
  const requestId = event.context.requestId as string
  logWithContext('info', 'Fetching user list', { requestId, action: 'users.list' })
  const start = performance.now()
  const users = await prisma.user.findMany()
  logWithContext('info', 'User list fetched', {
    requestId, action: 'users.list', duration: performance.now() - start, count: users.length,
  })
  return users
})
```

---

## Alerting 與 SLO/SLA

### SLI 定義

```typescript
// server/utils/sli.ts
import { Counter, Histogram } from 'prom-client'

export const sliAvailability = new Counter({
  name: 'sli_requests_total',
  help: 'Total requests for SLI availability',
  labelNames: ['result'] as const, // 'success' | 'failure'
})

export const sliLatency = new Histogram({
  name: 'sli_request_latency_seconds',
  help: 'Request latency for SLI',
  buckets: [0.05, 0.1, 0.2, 0.5, 1.0],
})

export const sliErrors = new Counter({
  name: 'sli_errors_total',
  help: 'Total errors by type',
  labelNames: ['type'] as const, // '5xx' | '4xx' | 'timeout'
})
```

### SLO 設定（Prometheus Alerting Rules）

```yaml
# infra/slo-rules.yml
groups:
  - name: slo-alerts
    rules:
      - alert: SLO_Availability_Burn_Rate_High    # 99.9% Availability
        expr: (sum(rate(sli_requests_total{result="failure"}[5m])) / sum(rate(sli_requests_total[5m]))) > 0.001
        for: 5m
        labels: { severity: critical, slo: availability }
        annotations: { summary: '可用性 SLO 燒毀率過高 (> 0.1%)' }
      - alert: SLO_Latency_P95_High               # p95 < 200ms
        expr: histogram_quantile(0.95, sum(rate(sli_request_latency_seconds_bucket[5m])) by (le)) > 0.2
        for: 5m
        labels: { severity: warning, slo: latency }
      - alert: SLO_Latency_P99_Critical            # p99 < 500ms
        expr: histogram_quantile(0.99, sum(rate(sli_request_latency_seconds_bucket[5m])) by (le)) > 0.5
        for: 5m
        labels: { severity: critical, slo: latency }
```

### Error Budget 概念與實踐

以 99.9% SLO 為例：月度 43,200 分鐘中允許 43.2 分鐘停機（0.1%）。

| Budget 餘額 | 策略 |
|-------------|------|
| > 50% | 正常功能開發 |
| 25~50% | 減緩功能發布，優先處理穩定性 |
| < 25% | 凍結功能發布，全力修復穩定性問題 |
| 耗盡 | 停止所有非穩定性相關工作 |

```yaml
# Prometheus recording rule: 追蹤 Error Budget 消耗
groups:
  - name: error-budget
    interval: 1m
    rules:
      - record: slo:error_budget:remaining_ratio
        expr: |
          1 - (sum(increase(sli_requests_total{result="failure"}[30d]))
               / (sum(increase(sli_requests_total[30d])) * 0.001))
```

### Alert 分級與 Escalation

| 等級 | 影響範圍 | 回應時間 | 通知方式 | 範例 |
|------|----------|----------|----------|------|
| P0 | 全面服務中斷 | < 5 分鐘 | 電話 + PagerDuty | 資料庫完全不可用 |
| P1 | 核心功能異常 | < 15 分鐘 | PagerDuty + Slack | 付款流程失敗率 > 5% |
| P2 | 效能降級 | < 1 小時 | Slack | p95 延遲超標 |
| P3 | 非緊急資訊 | 下個工作日 | Slack 低優先 | 憑證將在 30 天後過期 |

```yaml
# infra/alertmanager.yml
route:
  receiver: default
  group_by: ['alertname', 'severity']
  group_wait: 30s
  routes:
    - match: { severity: critical, slo: availability }  # P0
      receiver: pagerduty-p0
      repeat_interval: 5m
    - match: { severity: critical }                       # P1
      receiver: pagerduty-p1
      repeat_interval: 15m
    - match: { severity: warning }                        # P2
      receiver: slack-engineering
      repeat_interval: 1h
    - match: { severity: info }                           # P3
      receiver: slack-alerts-low
      repeat_interval: 24h
receivers:
  - name: pagerduty-p0
    pagerduty_configs:
      - service_key: '<SERVICE_KEY>'
        severity: critical
  - name: pagerduty-p1
    pagerduty_configs:
      - service_key: '<SERVICE_KEY>'
        severity: error
  - name: slack-engineering
    slack_configs:
      - api_url: '<SLACK_WEBHOOK>'
        channel: '#engineering-alerts'
  - name: slack-alerts-low
    slack_configs:
      - api_url: '<SLACK_WEBHOOK>'
        channel: '#alerts-low-priority'
  - name: default
    slack_configs:
      - api_url: '<SLACK_WEBHOOK>'
        channel: '#alerts'
```

---

## Terraform IaC 範例

### 概述

使用 Terraform 管理 Nuxt 3 應用在 AWS 上的完整基礎設施，涵蓋網路、運算、資料庫、快取與 CDN 層。所有資源透過程式碼定義，確保環境一致性與可重現性。

### State 管理（S3 Backend）

```hcl
# terraform/backend.tf — 遠端狀態儲存與鎖定
terraform {
  required_version = ">= 1.7.0"

  backend "s3" {
    bucket         = "nuxt-app-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "ap-northeast-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"          # 狀態鎖定，防止並行操作衝突
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "nuxt-app"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

### 變數定義與 tfvars

```hcl
# terraform/variables.tf — 所有可配置變數
variable "aws_region" {
  description = "AWS 部署區域"
  type        = string
  default     = "ap-northeast-1"
}

variable "environment" {
  description = "部署環境 (staging / production)"
  type        = string
  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "environment 必須為 staging 或 production"
  }
}

variable "app_name" {
  description = "應用名稱"
  type        = string
  default     = "nuxt-app"
}

variable "container_image" {
  description = "Docker 映像完整路徑含標籤"
  type        = string
}

variable "container_port" {
  description = "容器監聽連接埠"
  type        = number
  default     = 3000
}

variable "desired_count" {
  description = "ECS 服務所需任務數量"
  type        = number
  default     = 3
}

variable "cpu" {
  description = "Fargate 任務 CPU 單位 (1 vCPU = 1024)"
  type        = number
  default     = 512
}

variable "memory" {
  description = "Fargate 任務記憶體 (MiB)"
  type        = number
  default     = 1024
}

variable "db_instance_class" {
  description = "RDS 執行個體類型"
  type        = string
  default     = "db.t4g.medium"
}

variable "db_name" {
  description = "PostgreSQL 資料庫名稱"
  type        = string
  default     = "nuxtapp"
}

variable "db_username" {
  description = "資料庫管理員帳號"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "資料庫管理員密碼"
  type        = string
  sensitive   = true
}

variable "redis_node_type" {
  description = "ElastiCache Redis 節點類型"
  type        = string
  default     = "cache.t4g.small"
}

variable "domain_name" {
  description = "應用域名"
  type        = string
  default     = "app.example.com"
}

variable "certificate_arn" {
  description = "ACM 憑證 ARN（用於 ALB HTTPS）"
  type        = string
}

variable "cloudfront_certificate_arn" {
  description = "ACM 憑證 ARN（us-east-1，用於 CloudFront）"
  type        = string
}
```

```hcl
# terraform/environments/production.tfvars — 生產環境配置
aws_region                 = "ap-northeast-1"
environment                = "production"
app_name                   = "nuxt-app"
container_image            = "123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/nuxt-app:v2.1.0"
container_port             = 3000
desired_count              = 3
cpu                        = 1024
memory                     = 2048
db_instance_class          = "db.r6g.large"
db_name                    = "nuxtapp"
redis_node_type            = "cache.r6g.large"
domain_name                = "app.example.com"
certificate_arn            = "arn:aws:acm:ap-northeast-1:123456789012:certificate/xxxxxxxx"
cloudfront_certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/yyyyyyyy"
```

```hcl
# terraform/environments/staging.tfvars — 測試環境配置（較小資源）
aws_region                 = "ap-northeast-1"
environment                = "staging"
app_name                   = "nuxt-app"
container_image            = "123456789012.dkr.ecr.ap-northeast-1.amazonaws.com/nuxt-app:develop"
container_port             = 3000
desired_count              = 1
cpu                        = 512
memory                     = 1024
db_instance_class          = "db.t4g.medium"
db_name                    = "nuxtapp_staging"
redis_node_type            = "cache.t4g.small"
domain_name                = "staging.example.com"
certificate_arn            = "arn:aws:acm:ap-northeast-1:123456789012:certificate/xxxxxxxx"
cloudfront_certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/yyyyyyyy"
```

### VPC 網路架構

```hcl
# terraform/vpc.tf — 三層網路架構（公有 / 私有 / 資料庫）
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.5"

  name = "${var.app_name}-${var.environment}"
  cidr = "10.0.0.0/16"

  azs              = local.azs
  public_subnets   = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets  = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
  database_subnets = ["10.0.21.0/24", "10.0.22.0/24", "10.0.23.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = var.environment == "staging"   # staging 節省成本用單一 NAT
  enable_dns_hostnames   = true
  enable_dns_support     = true

  create_database_subnet_group           = true
  create_database_subnet_route_table     = true
  create_database_internet_gateway_route = false   # 資料庫子網不開放外部存取

  tags = {
    Environment = var.environment
  }
}
```

### ALB（Application Load Balancer）

```hcl
# terraform/alb.tf — 應用負載均衡器
resource "aws_security_group" "alb" {
  name_prefix = "${var.app_name}-alb-"
  description = "ALB 安全群組"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle { create_before_destroy = true }
}

resource "aws_lb" "main" {
  name               = "${var.app_name}-${var.environment}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnets

  enable_deletion_protection = var.environment == "production"

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"
    enabled = true
  }
}

resource "aws_lb_target_group" "app" {
  name        = "${var.app_name}-${var.environment}"
  port        = var.container_port
  protocol    = "HTTP"
  vpc_id      = module.vpc.vpc_id
  target_type = "ip"

  health_check {
    enabled             = true
    path                = "/api/deployment/health"
    port                = "traffic-port"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 15
    matcher             = "200"
  }

  deregistration_delay = 30
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_s3_bucket" "alb_logs" {
  bucket = "${var.app_name}-${var.environment}-alb-logs"
}

resource "aws_s3_bucket_lifecycle_configuration" "alb_logs" {
  bucket = aws_s3_bucket.alb_logs.id

  rule {
    id     = "expire-logs"
    status = "Enabled"

    expiration { days = 90 }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
  }
}
```

### ECS Fargate 服務定義

```hcl
# terraform/ecs.tf — ECS Cluster + Fargate Service
resource "aws_ecs_cluster" "main" {
  name = "${var.app_name}-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name       = aws_ecs_cluster.main.name
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    capacity_provider = var.environment == "production" ? "FARGATE" : "FARGATE_SPOT"
    weight            = 1
  }
}

resource "aws_security_group" "ecs_tasks" {
  name_prefix = "${var.app_name}-ecs-"
  description = "ECS 任務安全群組"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "From ALB"
    from_port       = var.container_port
    to_port         = var.container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_iam_role" "ecs_task_execution" {
  name = "${var.app_name}-${var.environment}-ecs-exec"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role_policy" "ecs_secrets" {
  name = "secrets-access"
  role = aws_iam_role.ecs_task_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["secretsmanager:GetSecretValue"]
      Resource = [aws_secretsmanager_secret.db_credentials.arn]
    }]
  })
}

resource "aws_iam_role" "ecs_task" {
  name = "${var.app_name}-${var.environment}-ecs-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.app_name}-${var.environment}"
  retention_in_days = var.environment == "production" ? 90 : 14
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.app_name}-${var.environment}"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "nuxt"
    image = var.container_image
    portMappings = [{ containerPort = var.container_port, protocol = "tcp" }]

    environment = [
      { name = "NODE_ENV", value = "production" },
      { name = "NUXT_PUBLIC_APP_VERSION", value = split(":", var.container_image)[1] },
      { name = "REDIS_URL", value = "redis://${aws_elasticache_replication_group.main.primary_endpoint_address}:6379" },
    ]

    secrets = [
      {
        name      = "DATABASE_URL"
        valueFrom = "${aws_secretsmanager_secret.db_credentials.arn}:DATABASE_URL::"
      },
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.app.name
        "awslogs-region"        = var.aws_region
        "awslogs-stream-prefix" = "nuxt"
      }
    }

    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:${var.container_port}/api/deployment/health || exit 1"]
      interval    = 15
      timeout     = 5
      retries     = 3
      startPeriod = 30
    }
  }])
}

resource "aws_ecs_service" "app" {
  name            = "${var.app_name}-${var.environment}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = var.environment == "production" ? "FARGATE" : null

  network_configuration {
    subnets          = module.vpc.private_subnets
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "nuxt"
    container_port   = var.container_port
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true                  # 部署失敗自動回滾
  }

  deployment_maximum_percent         = 200
  deployment_minimum_healthy_percent = 100
  health_check_grace_period_seconds  = 60

  lifecycle {
    ignore_changes = [desired_count]   # 交由 Auto Scaling 管理
  }
}

# Auto Scaling — 根據 CPU 使用率自動擴縮
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = var.environment == "production" ? 10 : 3
  min_capacity       = var.desired_count
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${var.app_name}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 65.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

### RDS PostgreSQL

```hcl
# terraform/rds.tf — PostgreSQL 資料庫
resource "aws_security_group" "rds" {
  name_prefix = "${var.app_name}-rds-"
  description = "RDS 安全群組"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "PostgreSQL from ECS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_tasks.id]
  }
}

resource "aws_db_parameter_group" "postgres" {
  name_prefix = "${var.app_name}-pg16-"
  family      = "postgres16"

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"                                  # 記錄超過 1 秒的慢查詢
  }

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }

  lifecycle { create_before_destroy = true }
}

resource "aws_db_instance" "main" {
  identifier     = "${var.app_name}-${var.environment}"
  engine         = "postgres"
  engine_version = "16.2"
  instance_class = var.db_instance_class

  allocated_storage     = 50
  max_allocated_storage = 200                       # 自動擴展上限
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  db_subnet_group_name   = module.vpc.database_subnet_group_name
  vpc_security_group_ids = [aws_security_group.rds.id]
  parameter_group_name   = aws_db_parameter_group.postgres.name

  multi_az                     = var.environment == "production"
  backup_retention_period      = var.environment == "production" ? 14 : 3
  backup_window                = "03:00-04:00"
  maintenance_window           = "mon:04:00-mon:05:00"
  auto_minor_version_upgrade   = true
  deletion_protection          = var.environment == "production"
  skip_final_snapshot          = var.environment != "production"
  final_snapshot_identifier    = var.environment == "production" ? "${var.app_name}-final-snapshot" : null
  performance_insights_enabled = true

  tags = { Name = "${var.app_name}-${var.environment}-postgres" }
}

resource "aws_secretsmanager_secret" "db_credentials" {
  name = "${var.app_name}/${var.environment}/db-credentials"
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    DATABASE_URL = "postgresql://${var.db_username}:${var.db_password}@${aws_db_instance.main.endpoint}/${var.db_name}?sslmode=require"
  })
}
```

### ElastiCache Redis

```hcl
# terraform/redis.tf — ElastiCache Redis 叢集
resource "aws_security_group" "redis" {
  name_prefix = "${var.app_name}-redis-"
  description = "Redis 安全群組"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "Redis from ECS"
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_tasks.id]
  }
}

resource "aws_elasticache_subnet_group" "main" {
  name       = "${var.app_name}-${var.environment}"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_elasticache_parameter_group" "main" {
  name   = "${var.app_name}-${var.environment}-redis7"
  family = "redis7"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"                     # 快取滿時自動淘汰最少使用的 key
  }
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${var.app_name}-${var.environment}"
  description          = "Nuxt App Redis — ${var.environment}"

  node_type            = var.redis_node_type
  num_cache_clusters   = var.environment == "production" ? 3 : 1
  port                 = 6379
  parameter_group_name = aws_elasticache_parameter_group.main.name
  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  automatic_failover_enabled = var.environment == "production"

  snapshot_retention_limit = var.environment == "production" ? 7 : 0
  snapshot_window          = "04:00-05:00"
  maintenance_window       = "tue:05:00-tue:06:00"

  apply_immediately = var.environment != "production"

  tags = { Name = "${var.app_name}-${var.environment}-redis" }
}
```

### CloudFront CDN Distribution

```hcl
# terraform/cloudfront.tf — CDN 分發靜態資源 + 動態回源
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  http_version        = "http2and3"
  price_class         = "PriceClass_200"               # 亞洲 + 歐美節點
  aliases             = [var.domain_name]
  comment             = "${var.app_name} ${var.environment} CDN"
  default_root_object = ""

  # 動態內容回源 — ALB
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "alb"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }

    custom_header {
      name  = "X-Forwarded-Host"
      value = var.domain_name
    }
  }

  # 靜態資源快取行為 — /_nuxt/* (build 輸出的 hashed assets)
  ordered_cache_behavior {
    path_pattern     = "/_nuxt/*"
    target_origin_id = "alb"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    compress         = true

    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 86400          # 1 天
    default_ttl            = 2592000        # 30 天
    max_ttl                = 31536000       # 365 天（hashed 檔案可長期快取）
  }

  # API 路徑 — 不快取
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    target_origin_id = "alb"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    compress         = true

    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Accept", "X-Request-Id"]
      cookies { forward = "all" }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 0
    max_ttl                = 0
  }

  # 預設行為 — SSR 頁面
  default_cache_behavior {
    target_origin_id = "alb"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    compress         = true

    forwarded_values {
      query_string = true
      headers      = ["Host", "Accept", "Accept-Language"]
      cookies { forward = "all" }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 0
    max_ttl                = 60              # SSR 頁面最多快取 60 秒
  }

  viewer_certificate {
    acm_certificate_arn      = var.cloudfront_certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  tags = { Name = "${var.app_name}-${var.environment}-cdn" }
}

# Route53 — 域名指向 CloudFront
data "aws_route53_zone" "main" {
  name = join(".", slice(split(".", var.domain_name), 1, length(split(".", var.domain_name))))
}

resource "aws_route53_record" "app" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}
```

### Outputs

```hcl
# terraform/outputs.tf — 部署後輸出重要資訊
output "alb_dns_name" {
  description = "ALB DNS 名稱"
  value       = aws_lb.main.dns_name
}

output "cloudfront_domain" {
  description = "CloudFront 分發域名"
  value       = aws_cloudfront_distribution.main.domain_name
}

output "rds_endpoint" {
  description = "RDS 連線端點"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

output "redis_endpoint" {
  description = "Redis 主要端點"
  value       = aws_elasticache_replication_group.main.primary_endpoint_address
  sensitive   = true
}

output "ecs_cluster_name" {
  description = "ECS 叢集名稱"
  value       = aws_ecs_cluster.main.name
}

output "ecs_service_name" {
  description = "ECS 服務名稱"
  value       = aws_ecs_service.app.name
}
```

### 部署指令

```bash
# 初始化（首次）
cd terraform && terraform init

# 規劃變更（對照 staging）
terraform plan -var-file=environments/staging.tfvars -out=staging.tfplan

# 套用變更
terraform apply staging.tfplan

# 生產部署
terraform plan -var-file=environments/production.tfvars -out=prod.tfplan
terraform apply prod.tfplan

# 查看目前狀態
terraform show

# 匯入已存在的資源
terraform import aws_db_instance.main nuxt-app-production
```

---

## Grafana Dashboard 定義

### 概述

採用 Dashboard-as-Code 方式，將 Grafana Dashboard 定義為 JSON 檔案並透過 provisioning 自動載入。每個面板使用 PromQL 查詢既有的 Nuxt 3 metrics 指標（對應本文件 Observability — Metrics 章節定義的 `nuxt_http_requests_total`、`nuxt_http_request_duration_seconds` 等指標）。

### Grafana Provisioning 配置

```yaml
# infra/grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: 'nuxt-app'
    orgId: 1
    folder: 'Nuxt Application'
    type: file
    disableDeletion: false
    editable: false                    # 防止 UI 手動修改，強制以程式碼為唯一來源
    updateIntervalSeconds: 30
    allowUiUpdates: false
    options:
      path: /var/lib/grafana/dashboards/nuxt-app
      foldersFromFilesStructure: true
```

```yaml
# infra/grafana/provisioning/datasources/datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

### Dashboard JSON 定義 — Nuxt 3 應用總覽

```json
{
  "dashboard": {
    "id": null,
    "uid": "nuxt-app-overview",
    "title": "Nuxt 3 應用總覽",
    "description": "請求速率、錯誤率、延遲分布與業務指標",
    "tags": ["nuxt", "production"],
    "timezone": "Asia/Taipei",
    "refresh": "30s",
    "time": { "from": "now-1h", "to": "now" },
    "templating": {
      "list": [
        {
          "name": "environment",
          "type": "custom",
          "current": { "text": "production", "value": "production" },
          "options": [
            { "text": "production", "value": "production" },
            { "text": "staging", "value": "staging" }
          ]
        },
        {
          "name": "route",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(nuxt_http_requests_total{environment=\"$environment\"}, route)",
          "refresh": 2,
          "includeAll": true,
          "allValue": ".*"
        }
      ]
    },
    "panels": [
      {
        "id": 1,
        "title": "Request Rate (req/s)",
        "description": "每秒請求數，依 HTTP 狀態碼分類",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(nuxt_http_requests_total{environment=\"$environment\", route=~\"$route\"}[5m])) by (status_code)",
            "legendFormat": "{{status_code}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "smooth",
              "fillOpacity": 15,
              "stacking": { "mode": "none" }
            }
          },
          "overrides": [
            {
              "matcher": { "id": "byRegexp", "options": "^5.." },
              "properties": [{ "id": "color", "value": { "fixedColor": "red", "mode": "fixed" } }]
            },
            {
              "matcher": { "id": "byRegexp", "options": "^4.." },
              "properties": [{ "id": "color", "value": { "fixedColor": "orange", "mode": "fixed" } }]
            },
            {
              "matcher": { "id": "byRegexp", "options": "^2.." },
              "properties": [{ "id": "color", "value": { "fixedColor": "green", "mode": "fixed" } }]
            }
          ]
        }
      },
      {
        "id": 2,
        "title": "Error Rate (%)",
        "description": "5xx 錯誤率佔總請求的百分比",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "100 * sum(rate(nuxt_http_requests_total{environment=\"$environment\", route=~\"$route\", status_code=~\"5..\"}[5m])) / sum(rate(nuxt_http_requests_total{environment=\"$environment\", route=~\"$route\"}[5m]))",
            "legendFormat": "Error Rate %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "custom": { "drawStyle": "line", "fillOpacity": 20 },
            "color": { "mode": "fixed", "fixedColor": "red" },
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": null, "color": "green" },
                { "value": 0.1, "color": "yellow" },
                { "value": 1, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "Latency p50 / p95 / p99",
        "description": "請求延遲百分位數分布",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(nuxt_http_request_duration_seconds_bucket{environment=\"$environment\", route=~\"$route\"}[5m])) by (le))",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(nuxt_http_request_duration_seconds_bucket{environment=\"$environment\", route=~\"$route\"}[5m])) by (le))",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(nuxt_http_request_duration_seconds_bucket{environment=\"$environment\", route=~\"$route\"}[5m])) by (le))",
            "legendFormat": "p99"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "smooth",
              "fillOpacity": 10
            },
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": null, "color": "green" },
                { "value": 0.2, "color": "yellow" },
                { "value": 0.5, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "Latency Heatmap",
        "description": "請求延遲分布熱力圖",
        "type": "heatmap",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 8 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(increase(nuxt_http_request_duration_seconds_bucket{environment=\"$environment\", route=~\"$route\"}[5m])) by (le)",
            "legendFormat": "{{le}}",
            "format": "heatmap"
          }
        ],
        "options": {
          "calculate": false,
          "yAxis": { "unit": "s" },
          "color": { "scheme": "Oranges" }
        }
      },
      {
        "id": 5,
        "title": "Active Connections",
        "description": "當前活躍連線數",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 16 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(nuxt_active_connections{environment=\"$environment\"})",
            "legendFormat": "Connections"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "short",
            "thresholds": {
              "steps": [
                { "value": null, "color": "green" },
                { "value": 100, "color": "yellow" },
                { "value": 500, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "id": 6,
        "title": "Daily Active Users (DAU)",
        "description": "每日活躍使用者數",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 6, "y": 16 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "business_daily_active_users{environment=\"$environment\"}",
            "legendFormat": "DAU"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "short",
            "color": { "mode": "fixed", "fixedColor": "blue" }
          }
        }
      },
      {
        "id": 7,
        "title": "Conversion Rate (%)",
        "description": "結帳轉換率",
        "type": "gauge",
        "gridPos": { "h": 4, "w": 6, "x": 12, "y": 16 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "business_conversion_rate{environment=\"$environment\"} * 100",
            "legendFormat": "Conversion"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                { "value": null, "color": "red" },
                { "value": 1, "color": "yellow" },
                { "value": 3, "color": "green" }
              ]
            }
          }
        }
      },
      {
        "id": 8,
        "title": "Error Budget Remaining",
        "description": "SLO 99.9% 下的 Error Budget 剩餘百分比",
        "type": "gauge",
        "gridPos": { "h": 4, "w": 6, "x": 18, "y": 16 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "slo:error_budget:remaining_ratio * 100",
            "legendFormat": "Error Budget"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                { "value": null, "color": "red" },
                { "value": 25, "color": "orange" },
                { "value": 50, "color": "yellow" },
                { "value": 75, "color": "green" }
              ]
            }
          }
        }
      },
      {
        "id": 9,
        "title": "Top 10 Slowest Routes",
        "description": "延遲最高的 API 路徑排名",
        "type": "table",
        "gridPos": { "h": 8, "w": 24, "x": 0, "y": 20 },
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "topk(10, histogram_quantile(0.95, sum(rate(nuxt_http_request_duration_seconds_bucket{environment=\"$environment\"}[5m])) by (le, route)))",
            "legendFormat": "{{route}}",
            "instant": true,
            "format": "table"
          }
        ],
        "transformations": [
          { "id": "organize", "options": { "renameByName": { "Value": "p95 (s)", "route": "Route" } } }
        ]
      }
    ],
    "schemaVersion": 39,
    "version": 1
  }
}
```

### PromQL 查詢速查表

| 指標 | PromQL | 說明 |
|------|--------|------|
| 總請求速率 | `sum(rate(nuxt_http_requests_total[5m]))` | 全應用每秒請求數 |
| 5xx 錯誤率 | `sum(rate(nuxt_http_requests_total{status_code=~"5.."}[5m])) / sum(rate(nuxt_http_requests_total[5m]))` | 伺服器端錯誤比率 |
| p50 延遲 | `histogram_quantile(0.50, sum(rate(nuxt_http_request_duration_seconds_bucket[5m])) by (le))` | 中位數延遲 |
| p95 延遲 | `histogram_quantile(0.95, sum(rate(nuxt_http_request_duration_seconds_bucket[5m])) by (le))` | 95th 百分位延遲 |
| p99 延遲 | `histogram_quantile(0.99, sum(rate(nuxt_http_request_duration_seconds_bucket[5m])) by (le))` | 99th 百分位延遲 |
| 各路徑請求量 | `sum(rate(nuxt_http_requests_total[5m])) by (route)` | 依路徑分群的 QPS |
| DAU | `business_daily_active_users` | 每日活躍用戶（由應用推送） |
| 轉換率 | `business_conversion_rate * 100` | 結帳轉換百分比 |
| Error Budget 剩餘 | `slo:error_budget:remaining_ratio * 100` | 本月剩餘 error budget |
| 活躍連線 | `sum(nuxt_active_connections)` | 當前進行中的 HTTP 請求數 |

### Docker Compose 整合 Grafana Dashboard

```yaml
# 加入 docker-compose.yaml 的 grafana 服務掛載
services:
  grafana:
    image: grafana/grafana:latest
    ports: ['3001:3000']
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH: /var/lib/grafana/dashboards/nuxt-app/overview.json
    volumes:
      - ./infra/grafana/provisioning:/etc/grafana/provisioning
      - ./infra/grafana/dashboards:/var/lib/grafana/dashboards/nuxt-app
```

---

## Log Aggregation Pipeline

### 概述

本節定義以 Loki + Promtail 為主的日誌聚合管線，銜接本文件 Structured Logging 章節產生的 JSON 結構化日誌與 Correlation ID。所有日誌在寫入 Loki 後可透過 LogQL 查詢，並支援基於日誌的告警。

### Loki 部署配置

```yaml
# infra/loki/loki-config.yaml — Loki 伺服器配置
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: warn

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: "2024-01-01"
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  tsdb_shipper:
    active_index_directory: /loki/tsdb-index
    cache_location: /loki/tsdb-cache

limits_config:
  retention_period: 30d                       # 日誌保留 30 天
  max_query_series: 5000
  max_query_parallelism: 2
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  per_stream_rate_limit: 5MB
  per_stream_rate_limit_burst: 15MB

compactor:
  working_directory: /loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h

ruler:
  storage:
    type: local
    local:
      directory: /loki/rules
  rule_path: /loki/rules-temp
  alertmanager_url: http://alertmanager:9093
  ring:
    kvstore:
      store: inmemory
  enable_api: true
```

### Promtail 日誌收集器

```yaml
# infra/promtail/promtail-config.yaml — 收集 Nuxt 容器的 JSON 日誌
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Docker 容器日誌收集
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      # 只收集 nuxt-app 容器的日誌
      - source_labels: ['__meta_docker_container_name']
        regex: '.*nuxt.*'
        action: keep
      - source_labels: ['__meta_docker_container_name']
        target_label: container
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: service
    pipeline_stages:
      # 解析 Nuxt 結構化 JSON 日誌
      - json:
          expressions:
            level: level
            message: message
            service: service
            requestId: requestId
            userId: userId
            action: action
            duration: duration
            timestamp: timestamp
      - labels:
          level:
          service:
          action:
      - timestamp:
          source: timestamp
          format: RFC3339
      # 提取 requestId 作為可搜尋的標籤
      - structured_metadata:
          requestId:
          userId:

  # Kubernetes 環境日誌收集
  - job_name: kubernetes
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: ['__meta_kubernetes_namespace']
        regex: 'nuxt-.*'
        action: keep
      - source_labels: ['__meta_kubernetes_pod_label_app']
        target_label: app
      - source_labels: ['__meta_kubernetes_namespace']
        target_label: namespace
      - source_labels: ['__meta_kubernetes_pod_name']
        target_label: pod
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            service: service
            requestId: requestId
            action: action
            duration: duration
      - labels:
          level:
          service:
          action:
      - structured_metadata:
          requestId:
```

### LogQL 查詢範例

```promql
# 基本查詢 — 查看特定服務的錯誤日誌
{service="nuxt-app", level="error"}

# 依 Correlation ID 追蹤完整請求鏈路（銜接 Structured Logging 章節的 requestId）
{service="nuxt-app"} | json | requestId="550e8400-e29b-41d4-a716-446655440000"

# 查看特定 API 路徑的慢請求（duration > 1000ms）
{service="nuxt-app", action=~".*\\.list|.*\\.getById"} | json | duration > 1000

# 統計每分鐘各 level 的日誌數量
sum by (level) (count_over_time({service="nuxt-app"}[1m]))

# 統計每分鐘錯誤率
sum(count_over_time({service="nuxt-app", level="error"}[5m]))
  / sum(count_over_time({service="nuxt-app"}[5m])) * 100

# 查看特定使用者的操作記錄
{service="nuxt-app"} | json | userId="user-12345"

# 使用 line_format 格式化輸出
{service="nuxt-app", level="error"}
  | json
  | line_format "{{.timestamp}} [{{.requestId}}] {{.message}} ({{.action}}, {{.duration}}ms)"

# 分析最常出現的錯誤訊息（Top 10）
topk(10, sum by (message) (count_over_time({service="nuxt-app", level="error"}[1h])))

# p95 請求耗時（從日誌中的 duration 欄位計算）
quantile_over_time(0.95, {service="nuxt-app"} | json | unwrap duration [5m]) by (action)
```

### 基於日誌的告警規則（Loki Ruler）

```yaml
# infra/loki/rules/nuxt-app/alerts.yaml
groups:
  - name: nuxt-log-alerts
    rules:
      # 高錯誤率告警 — 5 分鐘內錯誤日誌超過 50 條
      - alert: HighErrorLogRate
        expr: |
          sum(count_over_time({service="nuxt-app", level="error"}[5m])) > 50
        for: 2m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Nuxt 應用錯誤日誌量異常升高"
          description: "過去 5 分鐘共產生 {{ $value }} 條錯誤日誌"

      # 資料庫連線錯誤
      - alert: DatabaseConnectionErrors
        expr: |
          sum(count_over_time({service="nuxt-app", level="error"} |~ "database|prisma|postgresql|connection refused"[5m])) > 5
        for: 1m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "偵測到資料庫連線錯誤"
          description: "過去 5 分鐘有 {{ $value }} 條資料庫相關錯誤"

      # 慢請求告警 — p95 超過 2 秒
      - alert: SlowRequestsDetected
        expr: |
          quantile_over_time(0.95,
            {service="nuxt-app"} | json | unwrap duration [5m]
          ) > 2000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "請求延遲 p95 超過 2 秒"
          description: "日誌分析顯示 p95 延遲為 {{ $value }}ms"

      # 異常重啟偵測
      - alert: ApplicationRestart
        expr: |
          count_over_time({service="nuxt-app"} |= "Nitro server started" [10m]) > 3
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Nuxt 應用在 10 分鐘內重啟超過 3 次"
```

### ELK 替代方案對比

| 面向 | Loki + Promtail | ELK (Elasticsearch + Logstash + Kibana) |
|------|-----------------|----------------------------------------|
| 索引策略 | 只索引標籤（label），日誌內容不索引 | 全文索引所有欄位 |
| 儲存成本 | 極低（壓縮原始日誌） | 高（倒排索引佔用大量空間） |
| 查詢效能 | 標籤過濾快，全文搜尋較慢 | 全文搜尋極快 |
| 運維複雜度 | 低（單一二進位檔，配置簡單） | 高（JVM 調優、叢集管理、分片策略） |
| 與 Grafana 整合 | 原生整合（同一 UI） | 需額外配置或使用 Kibana |
| 記憶體需求 | 低（~256MB 起跳） | 高（建議 8GB+ heap） |
| 適用場景 | 雲原生、Kubernetes、中小規模 | 大規模全文搜尋、合規審計 |
| 建議 | Nuxt 3 應用首選，搭配 Prometheus + Grafana 生態 | 需要跨系統日誌關聯與複雜全文搜尋時選用 |

### Docker Compose 整合日誌管線

```yaml
# 加入 docker-compose.yaml 的日誌管線服務
services:
  loki:
    image: grafana/loki:3.0.0
    ports: ['3100:3100']
    volumes:
      - ./infra/loki/loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:3.0.0
    volumes:
      - ./infra/promtail/promtail-config.yaml:/etc/promtail/config.yml
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    depends_on: [loki]

volumes:
  loki-data:
```

---

## GitOps 工作流程

### 概述

採用 ArgoCD 實現 GitOps 工作流程，以 Git 儲存庫作為基礎設施與應用狀態的唯一事實來源（Single Source of Truth）。所有部署變更透過 Git commit 觸發，回滾透過 `git revert` 實現，確保完整的變更審計軌跡。

### Git 儲存庫結構

```
nuxt-app-gitops/                          # GitOps 專用儲存庫（與應用程式碼分離）
├── apps/
│   ├── nuxt-app/
│   │   ├── base/                         # 基礎 Kubernetes 資源定義
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── hpa.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── ingress.yaml
│   │   └── overlays/                     # 環境特定覆寫
│   │       ├── staging/
│   │       │   ├── kustomization.yaml
│   │       │   ├── replica-patch.yaml
│   │       │   └── env-configmap.yaml
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           ├── replica-patch.yaml
│   │           ├── env-configmap.yaml
│   │           └── resource-patch.yaml
├── infrastructure/                       # 共用基礎設施元件
│   ├── cert-manager/
│   ├── ingress-nginx/
│   └── monitoring/
├── argocd/                               # ArgoCD 自身的設定
│   ├── application-sets/
│   │   └── nuxt-app.yaml
│   ├── projects/
│   │   └── nuxt-app.yaml
│   └── image-updater/
│       └── nuxt-app-annotations.yaml
└── scripts/
    ├── promote.sh                        # 將 staging 映像升級到 production
    └── rollback.sh                       # 快速回滾腳本
```

### Kustomize 基礎資源

```yaml
# apps/nuxt-app/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
commonLabels:
  app.kubernetes.io/name: nuxt-app
  app.kubernetes.io/managed-by: argocd

resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml
  - configmap.yaml
  - ingress.yaml
```

```yaml
# apps/nuxt-app/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuxt-app
  annotations:
    argocd-image-updater.argoproj.io/image-list: "nuxt=ghcr.io/myorg/nuxt-app"
    argocd-image-updater.argoproj.io/nuxt.update-strategy: semver
    argocd-image-updater.argoproj.io/nuxt.allow-tags: "regexp:^v\\d+\\.\\d+\\.\\d+$"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: nuxt-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nuxt-app
    spec:
      serviceAccountName: nuxt-app
      containers:
        - name: nuxt
          image: ghcr.io/myorg/nuxt-app:v2.1.0     # ArgoCD Image Updater 會自動更新此標籤
          ports:
            - containerPort: 3000
              protocol: TCP
          envFrom:
            - configMapRef:
                name: nuxt-app-config
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /api/deployment/health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /api/deployment/health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: nuxt-app
```

```yaml
# apps/nuxt-app/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
bases:
  - ../../base

patches:
  - path: replica-patch.yaml
  - path: resource-patch.yaml

configMapGenerator:
  - name: nuxt-app-config
    behavior: merge
    literals:
      - NODE_ENV=production
      - LOG_LEVEL=info
      - NUXT_PUBLIC_API_BASE=https://api.example.com
```

```yaml
# apps/nuxt-app/overlays/production/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuxt-app
spec:
  replicas: 5
```

```yaml
# apps/nuxt-app/overlays/production/resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nuxt-app
spec:
  template:
    spec:
      containers:
        - name: nuxt
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 2Gi
```

### ArgoCD Application 定義

```yaml
# argocd/application-sets/nuxt-app.yaml — 使用 ApplicationSet 管理多環境
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nuxt-app
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - environment: staging
            namespace: staging
            cluster: https://kubernetes.default.svc
            autoSync: "true"
            prune: "true"
          - environment: production
            namespace: production
            cluster: https://kubernetes.default.svc
            autoSync: "false"           # 生產環境需手動同步
            prune: "false"
  template:
    metadata:
      name: "nuxt-app-{{environment}}"
      namespace: argocd
      labels:
        app.kubernetes.io/part-of: nuxt-app
        environment: "{{environment}}"
    spec:
      project: nuxt-app
      source:
        repoURL: https://github.com/myorg/nuxt-app-gitops.git
        targetRevision: main
        path: "apps/nuxt-app/overlays/{{environment}}"
      destination:
        server: "{{cluster}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          selfHeal: true                # 叢集狀態偏移時自動修正
          prune: "{{prune}}"            # 刪除 Git 中不存在的資源
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
          - PruneLast=true
        retry:
          limit: 3
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
      ignoreDifferences:
        - group: apps
          kind: Deployment
          jsonPointers:
            - /spec/replicas            # 忽略 HPA 調整的 replica 數量差異
```

### ArgoCD Project 定義

```yaml
# argocd/projects/nuxt-app.yaml — 限制存取範圍
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: nuxt-app
  namespace: argocd
spec:
  description: "Nuxt 3 應用專案"
  sourceRepos:
    - https://github.com/myorg/nuxt-app-gitops.git
  destinations:
    - namespace: staging
      server: https://kubernetes.default.svc
    - namespace: production
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: ''
      kind: '*'
    - group: apps
      kind: '*'
    - group: autoscaling
      kind: '*'
    - group: networking.k8s.io
      kind: '*'
  roles:
    - name: developer
      description: "開發者 — 可查看與同步 staging"
      policies:
        - "p, proj:nuxt-app:developer, applications, get, nuxt-app/*, allow"
        - "p, proj:nuxt-app:developer, applications, sync, nuxt-app/nuxt-app-staging, allow"
      groups:
        - dev-team
    - name: sre
      description: "SRE — 完整操作權限"
      policies:
        - "p, proj:nuxt-app:sre, applications, *, nuxt-app/*, allow"
      groups:
        - sre-team
```

### ArgoCD Image Updater 自動化部署

```yaml
# argocd/image-updater/install.yaml — Image Updater 配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  registries.conf: |
    registries:
      - name: GitHub Container Registry
        prefix: ghcr.io
        api_url: https://ghcr.io
        credentials: pullsecret:argocd/ghcr-credentials
        default: true
  log.level: info
  git.commit-message-template: |
    chore: update {{.AppName}} image to {{.NewImage}}

    Updated by ArgoCD Image Updater
    Old image: {{.OldImage}}
    New image: {{.NewImage}}
```

Image Updater 運作流程：

1. 定期掃描容器 registry 中符合 `v*.*.*` 語義化版本的新標籤
2. 偵測到新版本時，自動更新 GitOps 儲存庫中 Deployment 的 image tag
3. ArgoCD 偵測到 Git 變更後觸發同步
4. Staging 環境自動同步（`autoSync: true`）
5. Production 環境需 SRE 手動確認後同步

### 升級與回滾腳本

```bash
#!/bin/bash
# scripts/promote.sh — 將 staging 驗證通過的映像升級到 production
set -euo pipefail

IMAGE_TAG="${1:?用法: promote.sh <image-tag>}"
GITOPS_REPO="nuxt-app-gitops"
OVERLAY_PATH="apps/nuxt-app/overlays/production"

echo "==> 升級 production 至映像版本: ${IMAGE_TAG}"

# 更新 production overlay 中的映像標籤
cd "/tmp/${GITOPS_REPO}" || git clone git@github.com:myorg/${GITOPS_REPO}.git "/tmp/${GITOPS_REPO}"
cd "/tmp/${GITOPS_REPO}"
git checkout main && git pull origin main

# 使用 kustomize edit 更新映像
cd "${OVERLAY_PATH}"
kustomize edit set image "ghcr.io/myorg/nuxt-app:${IMAGE_TAG}"

# 提交並推送
git add -A
git commit -m "chore(production): promote nuxt-app to ${IMAGE_TAG}

Promoted from staging after validation.
Triggered by: $(whoami)
Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"

git push origin main

echo "==> 已推送至 GitOps 儲存庫。請在 ArgoCD UI 確認並手動同步 production。"
```

```bash
#!/bin/bash
# scripts/rollback.sh — 透過 git revert 回滾到前一個版本
set -euo pipefail

COMMIT_TO_REVERT="${1:?用法: rollback.sh <commit-sha>}"
GITOPS_REPO="nuxt-app-gitops"

echo "==> 回滾 commit: ${COMMIT_TO_REVERT}"

cd "/tmp/${GITOPS_REPO}"
git checkout main && git pull origin main

# 使用 git revert 產生反向 commit（保留完整歷史）
git revert --no-edit "${COMMIT_TO_REVERT}"

git push origin main

echo "==> 回滾 commit 已推送。ArgoCD 將自動偵測變更並同步。"
echo "==> 請在 ArgoCD UI 確認同步狀態。"
```

### CI/CD 與 GitOps 整合流程

```yaml
# .github/workflows/gitops-update.yaml — 應用 CI 完成後更新 GitOps 儲存庫
name: Update GitOps Repository
on:
  workflow_run:
    workflows: ["CI/CD Pipeline"]
    types: [completed]
    branches: [main]

jobs:
  update-gitops:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: myorg/nuxt-app-gitops
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops

      - name: Update staging image tag
        run: |
          cd gitops/apps/nuxt-app/overlays/staging
          kustomize edit set image "ghcr.io/myorg/nuxt-app:${{ github.event.workflow_run.head_sha }}"

      - name: Commit and push
        run: |
          cd gitops
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add -A
          git diff --staged --quiet || git commit -m "chore(staging): update nuxt-app to ${{ github.event.workflow_run.head_sha }}"
          git push
```

---

## Multi-Window Burn Rate Alerting

### 概述

實作 Google SRE 手冊推薦的多視窗燃燒率（Multi-Window Burn Rate）告警策略。相較於簡單的錯誤率閾值告警，多視窗方法能同時偵測快速故障與緩慢衰退，在靈敏度與精確度之間取得最佳平衡，避免誤報與漏報。

### 燃燒率原理

以 SLO 99.9%（每月允許 43.2 分鐘停機）為例：

| 燃燒率 | 意義 | Error Budget 消耗速度 | 耗盡時間 |
|--------|------|----------------------|----------|
| 1x | 正常消耗 | 0.1% / 月 | 30 天 |
| 2x | 雙倍消耗 | 0.2% / 月 | 15 天 |
| 6x | 6 倍消耗 | 0.6% / 月 | 5 天 |
| 14.4x | 14.4 倍消耗 | 1.44% / 月 | ~2 天 |
| 36x | 36 倍消耗 | 3.6% / 月 | ~20 小時 |
| 720x | 全面故障 | 72% / 月 | 1 小時 |

### 多視窗告警策略

Google SRE 推薦使用長短兩個視窗搭配，只有在兩個視窗都超過閾值時才觸發告警：

| 告警等級 | 長視窗 | 短視窗 | 燃燒率閾值 | Budget 消耗 | 回應 |
|---------|--------|--------|-----------|-------------|------|
| P0 — 快速燃燒 (嚴重) | 1h | 5m | 14.4x | 1h 耗 2% | 立即呼叫 on-call |
| P1 — 快速燃燒 (警告) | 6h | 30m | 6x | 6h 耗 5% | 15 分鐘內回應 |
| P2 — 慢速燃燒 | 3d | 6h | 1x | 3d 耗 10% | 工作時段處理 |

短視窗的作用：確認問題仍在持續（而非歷史殘留），避免已自行恢復的暫態問題產生誤報。

### Prometheus Recording Rules — 燃燒率計算

```yaml
# infra/prometheus/rules/burn-rate-recording.yml
# Recording rules 預先計算不同時間視窗的錯誤率，降低告警查詢的運算負擔
groups:
  - name: nuxt-slo-burn-rate
    interval: 30s
    rules:
      # === 可用性 SLI — 各時間視窗的錯誤率 ===

      # 5 分鐘視窗
      - record: nuxt:sli_error_rate:5m
        expr: |
          sum(rate(sli_requests_total{result="failure"}[5m]))
          / sum(rate(sli_requests_total[5m]))

      # 30 分鐘視窗
      - record: nuxt:sli_error_rate:30m
        expr: |
          sum(rate(sli_requests_total{result="failure"}[30m]))
          / sum(rate(sli_requests_total[30m]))

      # 1 小時視窗
      - record: nuxt:sli_error_rate:1h
        expr: |
          sum(rate(sli_requests_total{result="failure"}[1h]))
          / sum(rate(sli_requests_total[1h]))

      # 6 小時視窗
      - record: nuxt:sli_error_rate:6h
        expr: |
          sum(rate(sli_requests_total{result="failure"}[6h]))
          / sum(rate(sli_requests_total[6h]))

      # 3 天視窗
      - record: nuxt:sli_error_rate:3d
        expr: |
          sum(rate(sli_requests_total{result="failure"}[3d]))
          / sum(rate(sli_requests_total[3d]))

      # === 各視窗的燃燒率（相對於 SLO 0.1% 的倍率）===

      - record: nuxt:sli_burn_rate:5m
        expr: nuxt:sli_error_rate:5m / 0.001          # 0.001 = 1 - 0.999 (SLO 99.9%)

      - record: nuxt:sli_burn_rate:30m
        expr: nuxt:sli_error_rate:30m / 0.001

      - record: nuxt:sli_burn_rate:1h
        expr: nuxt:sli_error_rate:1h / 0.001

      - record: nuxt:sli_burn_rate:6h
        expr: nuxt:sli_error_rate:6h / 0.001

      - record: nuxt:sli_burn_rate:3d
        expr: nuxt:sli_error_rate:3d / 0.001

      # === 延遲 SLI — p99 延遲 ===

      - record: nuxt:sli_latency_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(sli_request_latency_seconds_bucket[5m])) by (le)
          )

      - record: nuxt:sli_latency_p99:1h
        expr: |
          histogram_quantile(0.99,
            sum(rate(sli_request_latency_seconds_bucket[1h])) by (le)
          )

      - record: nuxt:sli_latency_p99:6h
        expr: |
          histogram_quantile(0.99,
            sum(rate(sli_request_latency_seconds_bucket[6h])) by (le)
          )

      # === Error Budget 剩餘比率（30 天滾動視窗）===

      - record: nuxt:error_budget:remaining
        expr: |
          1 - (
            sum(increase(sli_requests_total{result="failure"}[30d]))
            / (sum(increase(sli_requests_total[30d])) * 0.001)
          )
```

### Prometheus Alerting Rules — 多視窗燃燒率告警

```yaml
# infra/prometheus/rules/burn-rate-alerts.yml
groups:
  - name: nuxt-multi-window-burn-rate
    rules:
      # ===== P0：快速燃燒（嚴重） =====
      # 長視窗 1h + 短視窗 5m，燃燒率 > 14.4x
      # 意義：以此速度，1 小時即消耗 2% 的月度 Error Budget
      - alert: NuxtBurnRateCritical
        expr: |
          nuxt:sli_burn_rate:1h > 14.4
          and
          nuxt:sli_burn_rate:5m > 14.4
        for: 2m
        labels:
          severity: critical
          slo: availability
          window: fast
          team: backend
        annotations:
          summary: "Nuxt 應用可用性快速燃燒 — P0"
          description: |
            1 小時燃燒率: {{ $value | printf "%.1f" }}x（閾值: 14.4x）
            以此速度，Error Budget 正以 {{ $value | printf "%.1f" }} 倍速度消耗。
            影響：每月 43.2 分鐘的 Error Budget 正以 {{ $value | printf "%.1f" }} 倍速度消耗。
          dashboard: "https://grafana.example.com/d/nuxt-app-overview"
          runbook: "https://wiki.example.com/runbooks/nuxt-burn-rate-critical"

      # ===== P1：快速燃燒（警告） =====
      # 長視窗 6h + 短視窗 30m，燃燒率 > 6x
      # 意義：以此速度，6 小時消耗 5% 的月度 Error Budget
      - alert: NuxtBurnRateWarning
        expr: |
          nuxt:sli_burn_rate:6h > 6
          and
          nuxt:sli_burn_rate:30m > 6
        for: 5m
        labels:
          severity: warning
          slo: availability
          window: fast
          team: backend
        annotations:
          summary: "Nuxt 應用可用性快速燃燒 — P1"
          description: |
            6 小時燃燒率: {{ $value | printf "%.1f" }}x（閾值: 6x）
            以此速度，Error Budget 正以 {{ $value | printf "%.1f" }} 倍速度消耗。
          dashboard: "https://grafana.example.com/d/nuxt-app-overview"

      # ===== P2：慢速燃燒 =====
      # 長視窗 3d + 短視窗 6h，燃燒率 > 1x
      # 意義：Error Budget 消耗速度超過正常水準，3 天消耗 10%
      - alert: NuxtBurnRateSlow
        expr: |
          nuxt:sli_burn_rate:3d > 1
          and
          nuxt:sli_burn_rate:6h > 1
        for: 30m
        labels:
          severity: info
          slo: availability
          window: slow
          team: backend
        annotations:
          summary: "Nuxt 應用可用性慢速燃燒 — P2"
          description: |
            3 天燃燒率: {{ $value | printf "%.1f" }}x（閾值: 1x）
            Error Budget 正在加速消耗，建議在工作時段調查。
          dashboard: "https://grafana.example.com/d/nuxt-app-overview"

      # ===== 延遲 SLO 燃燒率告警 =====

      # p99 延遲快速燃燒 — 1h 視窗 + 5m 視窗都超過 500ms 閾值
      - alert: NuxtLatencyBurnRateCritical
        expr: |
          nuxt:sli_latency_p99:1h > 0.5
          and
          nuxt:sli_latency_p99:5m > 0.5
        for: 2m
        labels:
          severity: critical
          slo: latency
          window: fast
          team: backend
        annotations:
          summary: "Nuxt 應用延遲 SLO 快速燃燒"
          description: |
            p99 延遲 (1h): {{ $value | printf "%.3f" }}s（閾值: 0.5s）
            延遲持續超標，影響使用者體驗。

      # p99 延遲慢速燃燒
      - alert: NuxtLatencyBurnRateSlow
        expr: |
          nuxt:sli_latency_p99:6h > 0.5
          and
          nuxt:sli_latency_p99:1h > 0.3
        for: 15m
        labels:
          severity: warning
          slo: latency
          window: slow
          team: backend
        annotations:
          summary: "Nuxt 應用延遲 SLO 慢速燃燒"
          description: "p99 延遲 (6h): {{ $value | printf \"%.3f\" }}s，趨勢持續升高。"

      # ===== Error Budget 耗盡預警 =====

      # Error Budget 剩餘不足 25%
      - alert: NuxtErrorBudgetLow
        expr: nuxt:error_budget:remaining < 0.25
        for: 5m
        labels:
          severity: warning
          slo: error-budget
          team: backend
        annotations:
          summary: "Nuxt 應用 Error Budget 剩餘不足 25%"
          description: |
            Error Budget 剩餘: {{ $value | printf "%.1f" }}%
            建議暫停功能發布，優先處理穩定性問題。
          action: "凍結功能發布，全力修復穩定性相關問題"

      # Error Budget 已耗盡
      - alert: NuxtErrorBudgetExhausted
        expr: nuxt:error_budget:remaining < 0
        for: 1m
        labels:
          severity: critical
          slo: error-budget
          team: backend
        annotations:
          summary: "Nuxt 應用 Error Budget 已耗盡"
          description: |
            Error Budget 已超支: {{ $value | printf "%.1f" }}%
            SLO 已違反，必須立即停止所有非穩定性相關工作。
          action: "停止所有功能開發，啟動事故回顧流程"
```

### 告警路由 — 依燃燒率等級分流

```yaml
# infra/alertmanager-burn-rate.yml — 燃燒率告警的路由策略
route:
  receiver: default
  group_by: ['alertname', 'slo', 'window']
  group_wait: 30s
  group_interval: 5m
  routes:
    # P0：快速燃燒嚴重 — 立即電話通知 on-call
    - match:
        severity: critical
        window: fast
      receiver: pagerduty-p0
      repeat_interval: 5m
      continue: true                        # 同時發送到 Slack

    # P0/P1：所有 critical 告警同步到 Slack
    - match:
        severity: critical
      receiver: slack-critical
      repeat_interval: 10m

    # P1：快速燃燒警告 — PagerDuty 低優先
    - match:
        severity: warning
        window: fast
      receiver: pagerduty-p1
      repeat_interval: 30m

    # P2：慢速燃燒 — Slack 通知
    - match:
        severity: info
        window: slow
      receiver: slack-engineering
      repeat_interval: 4h

    # Error Budget 告警
    - match:
        slo: error-budget
        severity: critical
      receiver: pagerduty-p0
      repeat_interval: 15m
    - match:
        slo: error-budget
        severity: warning
      receiver: slack-engineering
      repeat_interval: 2h

receivers:
  - name: pagerduty-p0
    pagerduty_configs:
      - service_key: '<P0_SERVICE_KEY>'
        severity: critical
        description: '{{ .CommonAnnotations.summary }}'
        details:
          description: '{{ .CommonAnnotations.description }}'
          dashboard: '{{ .CommonAnnotations.dashboard }}'
          runbook: '{{ .CommonAnnotations.runbook }}'

  - name: pagerduty-p1
    pagerduty_configs:
      - service_key: '<P1_SERVICE_KEY>'
        severity: error

  - name: slack-critical
    slack_configs:
      - api_url: '<SLACK_WEBHOOK>'
        channel: '#incidents'
        color: 'danger'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        actions:
          - type: button
            text: 'Dashboard'
            url: '{{ .CommonAnnotations.dashboard }}'
          - type: button
            text: 'Runbook'
            url: '{{ .CommonAnnotations.runbook }}'

  - name: slack-engineering
    slack_configs:
      - api_url: '<SLACK_WEBHOOK>'
        channel: '#engineering-alerts'
        color: 'warning'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'

  - name: default
    slack_configs:
      - api_url: '<SLACK_WEBHOOK>'
        channel: '#alerts'
```

### Error Budget 耗盡預測

```typescript
// server/utils/error-budget-forecast.ts — Error Budget 消耗趨勢預測
interface BudgetForecast {
  currentRemaining: number            // 目前剩餘百分比 (0-100)
  burnRate: number                     // 目前燃燒率百分比 (0-100+)
  exhaustionDate: Date | null          // 預計耗盡日期
  daysRemaining: number | null         // 距離耗盡天數
  recommendation: string              // 行動建議
}

export function forecastErrorBudget(
  totalRequests30d: number,
  failedRequests30d: number,
  sloTarget: number = 0.999,          // 99.9%
): BudgetForecast {
  // 防止除以零：無流量時回傳安全預設值
  if (totalRequests30d === 0) {
    return {
      currentRemaining: 1,
      burnRate: 0,
      exhaustionDate: null,
      daysRemaining: null,
      recommendation: '過去 30 天無流量資料，無法計算 Error Budget',
    }
  }

  const errorBudgetTotal = totalRequests30d * (1 - sloTarget)
  const currentRemaining = 1 - (failedRequests30d / errorBudgetTotal)
  const burnRate = failedRequests30d / errorBudgetTotal

  let exhaustionDate: Date | null = null
  let daysRemaining: number | null = null

  if (burnRate >= 1) {
    // 預算已耗盡
    daysRemaining = 0
    exhaustionDate = new Date()
  } else if (burnRate > 0) {
    // 以線性趨勢預測耗盡時間
    const remainingBudget = errorBudgetTotal - failedRequests30d
    const dailyBurnRate = failedRequests30d / 30
    daysRemaining = remainingBudget / dailyBurnRate
    exhaustionDate = new Date(Date.now() + daysRemaining * 24 * 60 * 60 * 1000)
  }

  let recommendation: string
  if (currentRemaining > 0.75) {
    recommendation = '正常：繼續功能開發，維持當前品質標準'
  } else if (currentRemaining > 0.50) {
    recommendation = '注意：開始關注穩定性，評估近期變更的影響'
  } else if (currentRemaining > 0.25) {
    recommendation = '警告：減緩功能發布速度，優先處理穩定性改善'
  } else if (currentRemaining > 0) {
    recommendation = '嚴重：凍結功能發布，全力修復穩定性問題'
  } else {
    recommendation = '耗盡：停止所有非穩定性相關工作，啟動事故回顧'
  }

  return {
    currentRemaining: Math.round(currentRemaining * 10000) / 100,
    burnRate: Math.round(burnRate * 10000) / 100,
    exhaustionDate,
    daysRemaining: daysRemaining !== null ? Math.round(daysRemaining * 10) / 10 : null,
    recommendation,
  }
}
```

```typescript
// server/api/slo/budget.get.ts — Error Budget 狀態 API
import { forecastErrorBudget } from '../../utils/error-budget-forecast'

export default defineEventHandler(async () => {
  // 從 Prometheus 查詢過去 30 天的請求數據
  const promUrl = process.env.PROMETHEUS_URL ?? 'http://prometheus:9090'

  const [totalRes, failedRes] = await Promise.all([
    $fetch<{ data: { result: Array<{ value: [number, string] }> } }>(
      `${promUrl}/api/v1/query`,
      { query: { query: 'sum(increase(sli_requests_total[30d]))' } },
    ),
    $fetch<{ data: { result: Array<{ value: [number, string] }> } }>(
      `${promUrl}/api/v1/query`,
      { query: { query: 'sum(increase(sli_requests_total{result="failure"}[30d]))' } },
    ),
  ])

  const total = Number(totalRes.data.result[0]?.value[1] ?? 0)
  const failed = Number(failedRes.data.result[0]?.value[1] ?? 0)

  const forecast = forecastErrorBudget(total, failed)

  return {
    slo: '99.9%',
    period: '30d rolling',
    totalRequests: total,
    failedRequests: failed,
    ...forecast,
  }
})
```

### 多視窗燃燒率視覺化 — Grafana 面板

```json
{
  "id": 10,
  "title": "Burn Rate — Multi-Window",
  "description": "各時間視窗的 Error Budget 燃燒率",
  "type": "timeseries",
  "gridPos": { "h": 8, "w": 24, "x": 0, "y": 28 },
  "datasource": "Prometheus",
  "targets": [
    {
      "expr": "nuxt:sli_burn_rate:5m",
      "legendFormat": "5m 燃燒率"
    },
    {
      "expr": "nuxt:sli_burn_rate:30m",
      "legendFormat": "30m 燃燒率"
    },
    {
      "expr": "nuxt:sli_burn_rate:1h",
      "legendFormat": "1h 燃燒率"
    },
    {
      "expr": "nuxt:sli_burn_rate:6h",
      "legendFormat": "6h 燃燒率"
    },
    {
      "expr": "nuxt:sli_burn_rate:3d",
      "legendFormat": "3d 燃燒率"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "short",
      "custom": {
        "drawStyle": "line",
        "lineInterpolation": "smooth",
        "fillOpacity": 5
      },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "value": null, "color": "green" },
          { "value": 1, "color": "yellow" },
          { "value": 6, "color": "orange" },
          { "value": 14.4, "color": "red" }
        ]
      }
    }
  },
  "options": {
    "tooltip": { "mode": "multi" },
    "legend": { "displayMode": "table", "placement": "right", "calcs": ["lastNotNull", "max"] }
  }
}
