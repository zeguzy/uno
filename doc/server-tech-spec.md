# MBTI 人格测试服务端技术规格说明书

## 文档信息

| 项目 | 值 |
|------|-----|
| 版本 | 1.0.0 |
| 创建日期 | 2026-04-04 |
| 状态 | 最终版 |

---

## 1. 概述

### 1.1 文档目的

本文档为 MBTI 人格测试网站后端服务提供完整的技术实现规格。任何具备 Node.js/TypeScript 经验的工程师应能仅依据本文档完成服务的完整实现，无需额外澄清或补充资料。

### 1.2 范围

本文档覆盖以下内容：
- 服务端 API 设计与实现
- 数据模型与存储策略
- 外部 API 集成（16personalities-api）
- 定时任务与同步机制
- 错误处理与降级策略
- 安全防护措施

本文档不覆盖：
- 前端实现
- CI/CD 流程
- 生产环境部署配置
- 负载均衡与水平扩展

### 1.3 目标读者

- 后端开发工程师
- 技术评审人员
- 测试工程师

### 1.4 技术栈确认表

| 组件 | 技术选型 | 版本要求 | 用途 |
|------|----------|----------|------|
| Runtime | Node.js | >= 18.0.0 | JavaScript 运行时 |
| Framework | Fastify | ^5.0.0 | Web 框架 |
| ORM | Prisma | ^6.0.0 | 数据库访问层 |
| Dev Database | SQLite | ^3.0.0 | 本地开发数据库 |
| Prod Database | PostgreSQL | >= 14.0 | 生产数据库 |
| Validation | Zod | ^3.23.0 | 请求/响应校验 |
| Scheduler | node-cron | ^3.0.0 | 定时任务调度 |
| HTTP Client | undici | ^6.0.0 | 外部 API 调用 |
| Linting | ESLint | ^9.0.0 | 代码规范检查 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Browser)                            │
└─────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            FASTIFY SERVER                                │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        MIDDLEWARE LAYER                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │    │
│  │  │ Rate Limiter │  │ Error Handler│  │ Request Validator    │  │    │
│  │  │ (per-IP)     │  │              │  │ (Zod)                │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                      │                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                         ROUTES LAYER                             │    │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐ │    │
│  │  │ /questions │ │ /test      │ │ /sync      │ │ /health      │ │    │
│  │  │ (GET)      │ │ (POST/GET) │ │ (GET/POST) │ │ (GET)        │ │    │
│  │  └────────────┘ └────────────┘ └────────────┘ └──────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                      │                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        SERVICE LAYER                             │    │
│  │  ┌────────────────┐ ┌────────────────┐ ┌────────────────────┐  │    │
│  │  │ QuestionService│ │ ScoringService │ │ SyncService        │  │    │
│  │  │                │ │                │ │ + Circuit Breaker  │  │    │
│  │  └────────────────┘ └────────────────┘ └────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                      │                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                         JOBS LAYER                               │    │
│  │  ┌──────────────────────────────────────────────────────────┐   │    │
│  │  │ SyncJob (cron: 0 0 * * * - every 24h at midnight UTC)    │   │    │
│  │  └──────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                    │                                      │
                    ▼                                      ▼
        ┌───────────────────┐              ┌───────────────────────────┐
        │     PRISMA ORM    │              │  16PERSONALITIES-API.COM  │
        └───────────────────┘              │   (External API)          │
                    │                      │   - GET /api/personality/ │
                    │                      │     questions             │
                    │                      │   - POST /api/personality │
                    ▼                      └───────────────────────────┘
        ┌───────────────────┐
        │ SQLite (dev)      │
        │ PostgreSQL (prod) │
        └───────────────────┘
```

### 2.2 数据流图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        QUESTION SYNC FLOW                             │
└──────────────────────────────────────────────────────────────────────┘

[Cron Trigger / Manual POST /sync]
              │
              ▼
    ┌─────────────────┐
    │  SyncService    │ ◄─── Check Circuit Breaker State
    │  .syncQuestions │
    └─────────────────┘
              │
              ▼ (if circuit closed)
    ┌─────────────────┐
    │ GET 16p API     │ ──► https://16personalities-api.com/api/personality/questions
    │ /questions      │
    └─────────────────┘
              │
              ├──────────────────┐
              │                  │
              ▼ (success)        ▼ (failure)
    ┌─────────────────┐  ┌─────────────────┐
    │ Validate        │  │ Increment       │
    │ Response (Zod)  │  │ Failure Count   │
    └─────────────────┘  └─────────────────┘
              │                  │
              ▼                  ▼ (if failures >= 3)
    ┌─────────────────┐  ┌─────────────────┐
    │ Calculate Diff  │  │ Open Circuit    │
    │ (version bump)  │  │ (6h cooldown)   │
    └─────────────────┘  └─────────────────┘
              │
              ▼
    ┌─────────────────┐
    │ Transaction:    │
    │ 1. Deactivate   │
    │    old questions│
    │ 2. Insert new   │
    │    questions    │
    │ 3. Create       │
    │    SyncLog      │
    └─────────────────┘


┌──────────────────────────────────────────────────────────────────────┐
│                          TEST SUBMIT FLOW                             │
└──────────────────────────────────────────────────────────────────────┘

[POST /api/test/submit]
        │
        ▼
┌─────────────────┐
│ Rate Limiter    │ ──► Check: 10 req/hour per IP
│ Middleware      │     Header: X-Forwarded-For or socket.remoteAddress
└─────────────────┘
        │
        ▼ (within limit)
┌─────────────────┐
│ Validate        │ ──► Zod: SubmitTestRequestSchema
│ Request Body    │
└─────────────────┘
        │
        ▼
┌─────────────────┐
│ Verify Question │ ──► Check questionSetVersion matches active set
│ Set Version     │     Reject if mismatch (stale client)
└─────────────────┘
        │
        ▼
┌─────────────────┐
│ Validate Answer │ ──► Each answer.id exists in DB
│ IDs             │     Each answer.value in [-3, -2, -1, 0, 1, 2, 3]
└─────────────────┘
        │
        ▼
┌─────────────────┐
│ Check Circuit   │ ──► If open: return cached result or error
│ Breaker         │
└─────────────────┘
        │
        ▼ (circuit closed)
┌─────────────────┐
│ POST 16p API    │ ──► https://16personalities-api.com/api/personality
│ /personality    │     Body: { answers, gender }
└─────────────────┘
        │
        ├──────────────────┐
        │                  │
        ▼ (success)        ▼ (failure)
┌─────────────────┐  ┌─────────────────┐
│ Generate UUID   │  │ Increment       │
│ Store Result    │  │ Failure Count   │
│ in DB           │  │ May open circuit│
└─────────────────┘  └─────────────────┘
        │                  │
        ▼                  ▼
┌─────────────────┐  ┌─────────────────┐
│ Return Result   │  │ Return Error    │
│ with id         │  │ EXTERNAL_API_   │
│                 │  │ UNAVAILABLE     │
└─────────────────┘  └─────────────────┘
```

### 2.3 服务间依赖关系

```
┌────────────────────────────────────────────────────────────────────┐
│                      DEPENDENCY GRAPH                              │
└────────────────────────────────────────────────────────────────────┘

Routes Layer
    │
    ├── questions.ts ──────► QuestionService
    │                              │
    │                              └──► Prisma Client
    │
    ├── test.ts ───────────► ScoringService
    │       │                      │
    │       │                      ├──► Prisma Client
    │       │                      │
    │       │                      └──► SyncService (for circuit breaker check)
    │       │
    │       └──► SyncService (circuit breaker state)
    │
    ├── sync.ts ───────────► SyncService
    │                              │
    │                              ├──► Prisma Client
    │                              │
    │                              └──► 16personalities API (external)
    │
    └── health.ts ─────────► Prisma Client (ping)
                             SyncService (circuit breaker status)
                             QuestionService (has data?)
```

---

## 3. 数据模型设计

### 3.1 Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearchPostgreSQL"]
}

datasource db {
  provider = "sqlite"  // 开发环境
  // provider = "postgresql"  // 生产环境
  url      = env("DATABASE_URL")
}

// ==================== Question Set Versioning ====================

/// 题目集版本，用于处理同步时的竞态条件
model QuestionSetVersion {
  id          String   @id @default(uuid())
  version     Int      @unique
  description String?  // 例如: "2026-04-01 同步"
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  questions   Question[]

  @@index([isActive])
  @@index([version])
  @@map("question_set_versions")
}

// ==================== Questions ====================

/// 题目缓存表
model Question {
  id                    String              @id // Base64 encoded from 16p API
  text                  String
  questionSetVersionId  String
  questionSetVersion    QuestionSetVersion  @relation(fields: [questionSetVersionId], references: [id], onDelete: Cascade)
  options               QuestionOption[]
  order                 Int                 @default(0)
  createdAt             DateTime            @default(now())
  
  @@unique([id, questionSetVersionId])
  @@index([questionSetVersionId])
  @@index([questionSetVersionId, order])
  @@map("questions")
}

/// 题目选项
model QuestionOption {
  id          String    @id @default(uuid())
  questionId  String
  question    Question  @relation(fields: [questionId], references: [id], onDelete: Cascade)
  label       String
  value       Int       // -3 到 +3
  order       Int       @default(0)
  
  @@unique([questionId, value])
  @@index([questionId])
  @@map("question_options")
}

// ==================== Test Results ====================

/// 测试结果主表
model TestResult {
  id                  String      @id @default(uuid())
  questionSetVersionId String
  questionSetVersion  QuestionSetVersion @relation(fields: [questionSetVersionId], references: [id])
  
  // 用户提交的元数据
  gender              String      // "Male" | "Female" | "Other"
  clientIp            String?     // 脱敏存储，用于分析
  userAgent           String?     // 脱敏存储
  
  // 16p API 返回的核心字段
  niceName            String
  fullCode            String      // 例如: "ENFP-A"
  avatarSrc           String?
  avatarSrcStatic     String?
  avatarAlt           String?
  snippet             String?     @db.Text
  
  // 完整的原始响应数据 (JSON)
  rawData             String      @db.Text
  
  createdAt           DateTime    @default(now())
  
  // 关联
  scales              TestScale[]
  traits              TestTrait[]
  
  @@index([createdAt])
  @@index([fullCode])
  @@index([questionSetVersionId])
  @@map("test_results")
}

/// 测试结果 - 维度得分
model TestScale {
  id            String    @id @default(uuid())
  testResultId  String
  testResult    TestResult @relation(fields: [testResultId], references: [id], onDelete: Cascade)
  
  scaleKey      String    // 例如: "EI", "SN", "TF", "JP", "AI"
  score         Float     // 原始分数
  percentage    Float     // 百分比
  
  @@unique([testResultId, scaleKey])
  @@index([testResultId])
  @@map("test_scales")
}

/// 测试结果 - 特质详情
model TestTrait {
  id            String    @id @default(uuid())
  testResultId  String
  testResult    TestResult @relation(fields: [testResultId], references: [id], onDelete: Cascade)
  
  traitKey      String    // 例如: "MIND", "ENERGY", "NATURE", "TACTICS", "IDENTITY"
  label         String
  color         String?   // 十六进制颜色
  score         Float
  percentage    Float     // pct 字段
  isReverse     Boolean   @default(false) // reverse 字段
  
  @@unique([testResultId, traitKey])
  @@index([testResultId])
  @@map("test_traits")
}

// ==================== Sync Log ====================

/// 同步日志表，同时存储熔断器状态
model SyncLog {
  id                  String    @id @default(uuid())
  
  // 同步执行信息
  triggeredBy         String    // "cron" | "manual" | "startup"
  status              String    // "pending" | "success" | "failed" | "skipped"
  questionsCount      Int?
  previousVersion     Int?
  newVersion          Int?
  errorMessage        String?
  durationMs          Int?
  startedAt           DateTime  @default(now())
  completedAt         DateTime?
  
  // 熔断器状态 (持久化)
  circuitBreakerState     String    @default("closed") // "closed" | "open" | "half-open"
  circuitBreakerFailures  Int       @default(0)
  circuitBreakerOpenedAt  DateTime?
  circuitBreakerCooldownUntil DateTime?
  
  @@index([startedAt])
  @@index([status])
  @@index([triggeredBy])
  @@map("sync_logs")
}

// ==================== Rate Limit Log ====================

/// 速率限制日志，用于追踪和审计
model RateLimitLog {
  id              String    @id @default(uuid())
  ipHash          String    // IP 的 SHA-256 哈希，不存储明文 IP
  endpoint        String
  blocked         Boolean   @default(false)
  requestCount    Int
  windowStart     DateTime
  createdAt       DateTime  @default(now())
  
  @@unique([ipHash, endpoint, windowStart])
  @@index([ipHash, endpoint])
  @@index([windowStart])
  @@map("rate_limit_logs")
}
```

### 3.2 实体关系说明

```
┌──────────────────────┐
│ QuestionSetVersion   │
├──────────────────────┤
│ id: uuid             │
│ version: int (unique)│◄─────────────────────────────────┐
│ isActive: boolean    │                                  │
└──────────────────────┘                                  │
         │                                                │
         │ 1:N                                            │ 1:N
         ▼                                                │
┌──────────────────────┐      ┌──────────────────────┐    │
│ Question             │      │ TestResult           │    │
├──────────────────────┤      ├──────────────────────┤    │
│ id: string (Base64)  │      │ id: uuid             │    │
│ text: string         │      │ gender: string       │    │
│ questionSetVersionId │──────│ questionSetVersionId │────┘
└──────────────────────┘      │ niceName: string     │
         │                    │ fullCode: string     │
         │ 1:N                │ rawData: text        │
         ▼                    └──────────────────────┘
┌──────────────────────┐              │
│ QuestionOption       │              │ 1:N
├──────────────────────┤              │
│ id: uuid             │              ├─────────────────┐
│ questionId: string   │              ▼                 ▼
│ label: string        │      ┌──────────────┐  ┌──────────────┐
│ value: int (-3 ~ +3) │      │ TestScale    │  │ TestTrait    │
└──────────────────────┘      ├──────────────┤  ├──────────────┤
                              │ testResultId │  │ testResultId │
                              │ scaleKey     │  │ traitKey     │
                              │ score        │  │ label        │
                              │ percentage   │  │ score        │
                              └──────────────┘  └──────────────┘
```

### 3.3 索引策略

| 表名 | 索引字段 | 类型 | 用途 |
|------|----------|------|------|
| QuestionSetVersion | isActive | B-Tree | 快速查询当前激活版本 |
| QuestionSetVersion | version | B-Tree (Unique) | 版本唯一性约束 |
| Question | questionSetVersionId | B-Tree | 按版本查询题目 |
| Question | [questionSetVersionId, order] | B-Tree | 按版本和顺序查询 |
| TestResult | createdAt | B-Tree | 按时间范围查询 |
| TestResult | fullCode | B-Tree | 按人格类型统计 |
| SyncLog | startedAt | B-Tree | 同步历史查询 |
| SyncLog | status | B-Tree | 按状态筛选 |
| RateLimitLog | [ipHash, endpoint] | B-Tree | 速率限制查询 |
| RateLimitLog | [ipHash, endpoint, windowStart] | B-Tree (Unique) | 窗口唯一性 |

### 3.4 迁移策略

#### 开发环境
1. 使用 `prisma migrate dev` 创建和应用迁移
2. SQLite 不支持某些 PostgreSQL 特性，注意以下限制：
   - 无数组类型
   - 无枚举类型（使用 String 替代）
   - 无 `RETURNING` 子句（Prisma 自动处理）

#### 生产环境
1. 修改 `schema.prisma` 中的 provider 为 `postgresql`
2. 运行 `prisma migrate deploy` 应用迁移
3. PostgreSQL 专用优化：
   - 添加 `@db.Text` 替换为 `TEXT` 类型
   - 可启用全文搜索功能

```bash
# 开发环境
DATABASE_URL="file:./dev.db" prisma migrate dev --name init

# 生产环境
DATABASE_URL="postgresql://user:pass@host:5432/mbti" prisma migrate deploy
```

---

## 4. API 接口规范

### 4.1 统一响应格式

#### 成功响应

```typescript
interface SuccessResponse<T> {
  success: true;
  data: T;
  meta?: {
    timestamp: string;      // ISO 8601
    requestId: string;      // UUID for tracing
    questionSetVersion?: number;  // 当前激活的题目集版本
  };
}
```

#### 错误响应

```typescript
interface ErrorResponse {
  success: false;
  error: {
    code: string;           // 例如: "VALIDATION_ERROR"
    message: string;        // 人类可读的错误描述
    details?: unknown;      // 额外错误详情
    requestId: string;      // UUID for tracing
  };
  meta: {
    timestamp: string;      // ISO 8601
  };
}
```

### 4.2 API 端点规格

#### 4.2.1 GET /api/health

**描述**: 健康检查端点，用于负载均衡器和监控

**请求头**: 无特殊要求

**查询参数**: 无

**成功响应** (200 OK):

```json
{
  "success": true,
  "data": {
    "status": "healthy" | "degraded" | "unhealthy",
    "checks": {
      "database": {
        "status": "ok" | "error",
        "latencyMs": 5
      },
      "questions": {
        "status": "ok" | "empty" | "error",
        "count": 120
      },
      "externalApi": {
        "status": "ok" | "circuit_open" | "unknown",
        "circuitBreakerState": "closed" | "open" | "half-open",
        "lastSuccessAt": "2026-04-04T10:30:00Z",
        "cooldownEndsAt": null
      }
    },
    "version": "1.0.0",
    "uptimeSeconds": 86400
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**降级响应** (503 Service Unavailable):

```json
{
  "success": false,
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "服务暂时不可用，请稍后重试",
    "details": {
      "reason": "initial_sync_pending"
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440001"
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z"
  }
}
```

---

#### 4.2.2 GET /api/questions

**描述**: 获取当前激活题目集的所有题目

**请求头**:

| Header | Required | Description |
|--------|----------|-------------|
| Accept | No | 建议设置为 `application/json` |

**查询参数**: 无

**成功响应** (200 OK):

```json
{
  "success": true,
  "data": {
    "questions": [
      {
        "id": "cXVlc3Rpb24tMQ==",
        "text": "You find it easy to stay relaxed and focused even when there is some pressure.",
        "order": 1,
        "options": [
          { "label": "Strongly Disagree", "value": -3, "order": 1 },
          { "label": "Disagree", "value": -2, "order": 2 },
          { "label": "Slightly Disagree", "value": -1, "order": 3 },
          { "label": "Neutral", "value": 0, "order": 4 },
          { "label": "Slightly Agree", "value": 1, "order": 5 },
          { "label": "Agree", "value": 2, "order": 6 },
          { "label": "Strongly Agree", "value": 3, "order": 7 }
        ]
      }
    ],
    "total": 120,
    "questionSetVersion": 5
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440002",
    "questionSetVersion": 5
  }
}
```

**错误响应** (503 Service Unavailable):

```json
{
  "success": false,
  "error": {
    "code": "QUESTIONS_NOT_AVAILABLE",
    "message": "题目数据暂不可用，请稍后重试",
    "details": {
      "reason": "no_active_question_set"
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440003"
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z"
  }
}
```

---

#### 4.2.3 POST /api/test/submit

**描述**: 提交测试答案，计算人格类型

**请求头**:

| Header | Required | Description |
|--------|----------|-------------|
| Content-Type | Yes | 必须为 `application/json` |
| X-Forwarded-For | No | 代理服务器传递的真实 IP |

**请求体**:

```typescript
interface SubmitTestRequest {
  questionSetVersion: number;     // 必须匹配当前激活版本
  gender: "Male" | "Female" | "Other";
  answers: Array<{
    id: string;                   // 题目 ID (Base64)
    value: number;                // -3 | -2 | -1 | 0 | 1 | 2 | 3
  }>;
}
```

**请求体示例**:

```json
{
  "questionSetVersion": 5,
  "gender": "Female",
  "answers": [
    { "id": "cXVlc3Rpb24tMQ==", "value": 2 },
    { "id": "cXVlc3Rpb24tMg==", "value": -1 }
  ]
}
```

**成功响应** (201 Created):

```json
{
  "success": true,
  "data": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "personality": {
      "niceName": "Commander",
      "fullCode": "ENTJ-A",
      "avatarSrc": "https://...",
      "avatarSrcStatic": "https://...",
      "avatarAlt": "ENTJ Commander avatar",
      "snippet": "Bold, imaginative and strong-willed leaders..."
    },
    "scales": [
      { "scaleKey": "EI", "score": -12.5, "percentage": 0.35 },
      { "scaleKey": "SN", "score": 8.0, "percentage": 0.62 },
      { "scaleKey": "TF", "score": 15.3, "percentage": 0.78 },
      { "scaleKey": "JP", "score": -5.0, "percentage": 0.42 },
      { "scaleKey": "AI", "score": 22.1, "percentage": 0.85 }
    ],
    "traits": [
      {
        "traitKey": "MIND",
        "label": "Extraverted",
        "color": "#E33B3B",
        "score": 12.5,
        "percentage": 0.65,
        "isReverse": false
      }
    ]
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440004"
  }
}
```

**错误响应**:

| Status | Code | Description |
|--------|------|-------------|
| 400 | VALIDATION_ERROR | 请求体格式错误 |
| 400 | QUESTION_SET_VERSION_MISMATCH | 题目集版本不匹配 |
| 400 | INVALID_ANSWER_COUNT | 答案数量不正确 |
| 400 | INVALID_ANSWER_VALUE | 答案值超出范围 |
| 400 | UNKNOWN_QUESTION_ID | 未知的题目 ID |
| 429 | RATE_LIMIT_EXCEEDED | 超过速率限制 |
| 503 | EXTERNAL_API_UNAVAILABLE | 外部 API 不可用 |

**错误响应示例** (400 Bad Request):

```json
{
  "success": false,
  "error": {
    "code": "QUESTION_SET_VERSION_MISMATCH",
    "message": "题目集版本不匹配，请刷新页面重新获取题目",
    "details": {
      "clientVersion": 4,
      "currentVersion": 5
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440005"
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z"
  }
}
```

**速率限制响应** (429 Too Many Requests):

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "请求过于频繁，请 42 分钟后重试",
    "details": {
      "retryAfterSeconds": 2520,
      "limit": 10,
      "window": "1h"
    },
    "requestId": "550e8400-e29b-41d4-a716-446655440006"
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z"
  }
}
```

---

#### 4.2.4 GET /api/test/result/:id

**描述**: 根据 ID 查询历史测试结果

**路径参数**:

| Parameter | Type | Description |
|-----------|------|-------------|
| id | string (UUID) | 测试结果 ID |

**查询参数**: 无

**成功响应** (200 OK):

```json
{
  "success": true,
  "data": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "personality": {
      "niceName": "Commander",
      "fullCode": "ENTJ-A",
      "avatarSrc": "https://...",
      "avatarSrcStatic": "https://...",
      "avatarAlt": "ENTJ Commander avatar",
      "snippet": "Bold, imaginative and strong-willed leaders..."
    },
    "scales": [
      { "scaleKey": "EI", "score": -12.5, "percentage": 0.35 }
    ],
    "traits": [
      {
        "traitKey": "MIND",
        "label": "Extraverted",
        "color": "#E33B3B",
        "score": 12.5,
        "percentage": 0.65,
        "isReverse": false
      }
    ],
    "createdAt": "2026-04-04T10:30:00Z"
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440007"
  }
}
```

**错误响应**:

| Status | Code | Description |
|--------|------|-------------|
| 400 | INVALID_RESULT_ID | 结果 ID 格式无效 |
| 404 | RESULT_NOT_FOUND | 结果不存在 |

**错误响应示例** (404 Not Found):

```json
{
  "success": false,
  "error": {
    "code": "RESULT_NOT_FOUND",
    "message": "未找到该测试结果",
    "requestId": "550e8400-e29b-41d4-a716-446655440008"
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z"
  }
}
```

---

#### 4.2.5 POST /api/sync

**描述**: 手动触发题目同步

**请求头**:

| Header | Required | Description |
|--------|----------|-------------|
| Content-Type | Yes | 必须为 `application/json` |
| X-Admin-Token | Yes | 管理员令牌 |

**请求体**: 无

**成功响应** (202 Accepted):

```json
{
  "success": true,
  "data": {
    "syncId": "660e8400-e29b-41d4-a716-446655440000",
    "status": "started",
    "message": "同步任务已启动"
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440009"
  }
}
```

**错误响应**:

| Status | Code | Description |
|--------|------|-------------|
| 401 | UNAUTHORIZED | 缺少或无效的管理员令牌 |
| 409 | SYNC_ALREADY_RUNNING | 已有同步任务在执行 |
| 503 | CIRCUIT_BREAKER_OPEN | 熔断器开启中 |

---

#### 4.2.6 GET /api/sync/status

**描述**: 获取同步状态和熔断器状态

**请求头**: 无特殊要求

**查询参数**: 无

**成功响应** (200 OK):

```json
{
  "success": true,
  "data": {
    "circuitBreaker": {
      "state": "closed",
      "failureCount": 0,
      "openedAt": null,
      "cooldownEndsAt": null
    },
    "lastSync": {
      "id": "660e8400-e29b-41d4-a716-446655440000",
      "status": "success",
      "triggeredBy": "cron",
      "questionsCount": 120,
      "previousVersion": 4,
      "newVersion": 5,
      "startedAt": "2026-04-04T00:00:00Z",
      "completedAt": "2026-04-04T00:00:05Z",
      "durationMs": 5000
    },
    "currentQuestionSetVersion": 5,
    "totalQuestions": 120
  },
  "meta": {
    "timestamp": "2026-04-04T12:00:00Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440010"
  }
}
```

---

### 4.3 HTTP 状态码总表

| Status Code | 使用场景 |
|-------------|----------|
| 200 OK | GET 请求成功 |
| 201 Created | POST 创建资源成功 |
| 202 Accepted | 异步任务已接收 |
| 400 Bad Request | 请求参数验证失败 |
| 401 Unauthorized | 缺少认证信息 |
| 404 Not Found | 资源不存在 |
| 409 Conflict | 状态冲突（如同步已在进行） |
| 429 Too Many Requests | 超过速率限制 |
| 500 Internal Server Error | 服务器内部错误 |
| 503 Service Unavailable | 服务不可用（如熔断器开启） |

---

### 4.4 错误码总表

| Code | HTTP Status | Description | User Message (zh-CN) |
|------|-------------|-------------|----------------------|
| VALIDATION_ERROR | 400 | 请求体验证失败 | 请求参数格式错误 |
| QUESTION_SET_VERSION_MISMATCH | 400 | 题目集版本不匹配 | 题目已更新，请刷新页面 |
| INVALID_ANSWER_COUNT | 400 | 答案数量不正确 | 答案数量不正确 |
| INVALID_ANSWER_VALUE | 400 | 答案值超出范围 | 答案值无效 |
| UNKNOWN_QUESTION_ID | 400 | 未知的题目 ID | 包含无效的题目 |
| INVALID_GENDER_VALUE | 400 | 性别值无效 | 性别选项无效 |
| INVALID_RESULT_ID | 400 | 结果 ID 格式无效 | 结果 ID 格式无效 |
| UNAUTHORIZED | 401 | 未授权访问 | 未授权的访问 |
| RESULT_NOT_FOUND | 404 | 测试结果不存在 | 未找到该测试结果 |
| SYNC_ALREADY_RUNNING | 409 | 同步任务已在运行 | 同步任务正在进行中 |
| RATE_LIMIT_EXCEEDED | 429 | 超过速率限制 | 请求过于频繁，请稍后重试 |
| SERVICE_UNAVAILABLE | 503 | 服务不可用 | 服务暂时不可用 |
| QUESTIONS_NOT_AVAILABLE | 503 | 题目数据不可用 | 题目数据暂不可用 |
| EXTERNAL_API_UNAVAILABLE | 503 | 外部 API 不可用 | 计算服务暂不可用 |
| CIRCUIT_BREAKER_OPEN | 503 | 熔断器开启 | 服务正在恢复中，请稍后重试 |
| INTERNAL_ERROR | 500 | 内部服务器错误 | 服务器内部错误 |

---

## 5. 服务层设计

### 5.1 QuestionService

**职责**: 管理题目数据的读取和查询

**文件**: `src/services/questionService.ts`

**依赖**:
- Prisma Client

**公开方法**:

```typescript
class QuestionService {
  /**
   * 获取当前激活题目集的所有题目
   * @returns 包含题目列表、总数和版本号的对象
   * @throws Error 如果没有激活的题目集
   */
  async getActiveQuestions(): Promise<{
    questions: QuestionWithOptions[];
    total: number;
    questionSetVersion: number;
  }>

  /**
   * 获取当前激活的题目集版本号
   * @returns 版本号，如果没有激活版本则返回 null
   */
  async getActiveVersion(): Promise<number | null>

  /**
   * 验证答案中的题目 ID 是否存在于指定版本中
   * @param questionSetVersion 题目集版本
   * @param answerIds 答案 ID 列表
   * @returns 验证结果，包含是否有效和无效的 ID 列表
   */
  async validateAnswerIds(
    questionSetVersion: number,
    answerIds: string[]
  ): Promise<{
    valid: boolean;
    invalidIds: string[];
  }>

  /**
   * 检查是否有激活的题目集
   * @returns 是否存在激活的题目集
   */
  async hasActiveQuestionSet(): Promise<boolean>

  /**
   * 获取激活题目集的题目数量
   * @returns 题目数量，如果没有则返回 0
   */
  async getActiveQuestionCount(): Promise<number>
}

interface QuestionWithOptions {
  id: string;
  text: string;
  order: number;
  options: Array<{
    id: string;
    label: string;
    value: number;
    order: number;
  }>;
}
```

---

### 5.2 ScoringService

**职责**: 处理测试提交，调用外部 API，存储结果

**文件**: `src/services/scoringService.ts`

**依赖**:
- Prisma Client
- SyncService (读取熔断器状态)
- QuestionService (验证题目)

**公开方法**:

```typescript
class ScoringService {
  /**
   * 提交测试答案
   * @param request 提交请求
   * @param clientIp 客户端 IP（用于速率限制日志）
   * @returns 测试结果
   */
  async submitTest(
    request: SubmitTestRequest,
    clientIp?: string
  ): Promise<TestResultResponse>

  /**
   * 根据 ID 获取测试结果
   * @param id 结果 ID
   * @returns 测试结果
   */
  async getResultById(id: string): Promise<TestResultResponse | null>
}

interface SubmitTestRequest {
  questionSetVersion: number;
  gender: 'Male' | 'Female' | 'Other';
  answers: Array<{ id: string; value: number }>;
}

interface TestResultResponse {
  id: string;
  personality: {
    niceName: string;
    fullCode: string;
    avatarSrc: string | null;
    avatarSrcStatic: string | null;
    avatarAlt: string | null;
    snippet: string | null;
  };
  scales: Array<{
    scaleKey: string;
    score: number;
    percentage: number;
  }>;
  traits: Array<{
    traitKey: string;
    label: string;
    color: string | null;
    score: number;
    percentage: number;
    isReverse: boolean;
  }>;
  createdAt?: string;
}
```

**核心流程**:

```
submitTest(request)
    │
    ├─► 1. 验证 questionSetVersion
    │       └─► 如果不匹配当前版本，抛出 QUESTION_SET_VERSION_MISMATCH
    │
    ├─► 2. 验证答案数量和值
    │       └─► 调用 QuestionService.validateAnswerIds()
    │
    ├─► 3. 检查熔断器状态
    │       └─► 调用 SyncService.getCircuitBreakerState()
    │       └─► 如果为 open，抛出 CIRCUIT_BREAKER_OPEN
    │
    ├─► 4. 调用外部 API
    │       └─► POST https://16personalities-api.com/api/personality
    │       └─► 超时设置: 30000ms
    │
    ├─► 5. 处理响应
    │       ├─► 成功: 解析并存储到数据库
    │       │         └─► 生成 UUID 作为结果 ID
    │       │         └─► 在事务中创建 TestResult, TestScale, TestTrait
    │       └─► 失败: 调用 SyncService.recordExternalApiFailure()
    │                 └─► 抛出 EXTERNAL_API_UNAVAILABLE
    │
    └─► 6. 返回结果
```

---

### 5.3 SyncService

**职责**: 同步题目数据，管理熔断器状态

**文件**: `src/services/syncService.ts`

**依赖**:
- Prisma Client
- 外部 API (16personalities)

**公开方法**:

```typescript
class SyncService {
  /**
   * 执行题目同步
   * @param triggeredBy 触发来源
   * @returns 同步结果
   */
  async syncQuestions(
    triggeredBy: 'cron' | 'manual' | 'startup'
  ): Promise<SyncResult>

  /**
   * 获取同步状态
   * @returns 同步状态信息
   */
  async getSyncStatus(): Promise<SyncStatusResponse>

  /**
   * 获取熔断器状态
   * @returns 熔断器当前状态
   */
  getCircuitBreakerState(): CircuitBreakerState

  /**
   * 记录外部 API 调用失败
   * 用于非同步场景的外部 API 调用失败
   */
  async recordExternalApiFailure(): Promise<void>

  /**
   * 检查是否可以执行同步
   * @returns 是否可以同步
   */
  canSync(): boolean
}

interface SyncResult {
  success: boolean;
  syncLogId: string;
  questionsCount?: number;
  previousVersion?: number;
  newVersion?: number;
  errorMessage?: string;
}

interface SyncStatusResponse {
  circuitBreaker: {
    state: 'closed' | 'open' | 'half-open';
    failureCount: number;
    openedAt: string | null;
    cooldownEndsAt: string | null;
  };
  lastSync: SyncLog | null;
  currentQuestionSetVersion: number | null;
  totalQuestions: number;
}

type CircuitBreakerState = 'closed' | 'open' | 'half-open';
```

**熔断器状态机**:

```
                    ┌─────────────────────────────────────────────┐
                    │                                             │
                    ▼                                             │
            ┌───────────────┐                                     │
            │    CLOSED     │                                     │
            │  (正常状态)    │                                     │
            └───────────────┘                                     │
                    │                                             │
                    │ API 调用失败                                │
                    │ failureCount++                              │
                    ▼                                             │
            ┌───────────────┐                                     │
            │ 检查失败次数   │                                     │
            │ >= 3 ?        │                                     │
            └───────────────┘                                     │
                    │                                             │
          ┌─────────┴─────────┐                                   │
          │ Yes               │ No                                │
          ▼                   │                                   │
  ┌───────────────┐           │                                   │
  │     OPEN      │           │                                   │
  │  (熔断状态)    │           │                                   │
  │  冷却期: 6h   │           │                                   │
  └───────────────┘           │                                   │
          │                   │                                   │
          │ 6h 后             │                                   │
          ▼                   │                                   │
  ┌───────────────┐           │                                   │
  │   HALF-OPEN   │           │                                   │
  │  (试探状态)    │           │                                   │
  └───────────────┘           │                                   │
          │                   │                                   │
    ┌─────┴─────┐             │                                   │
    │           │             │                                   │
  成功        失败            │                                   │
    │           │             │                                   │
    │           └─────────────┴───────────────────────────────────┘
    │                         返回 OPEN
    ▼
回到 CLOSED
failureCount = 0
```

**熔断器配置常量**:

```typescript
const CIRCUIT_BREAKER_CONFIG = {
  FAILURE_THRESHOLD: 3,        // 连续失败次数阈值
  COOLDOWN_DURATION_MS: 6 * 60 * 60 * 1000,  // 6 小时
  HALF_OPEN_MAX_ATTEMPTS: 1,   // 半开状态最大尝试次数
} as const;
```

**同步 Diff 算法**:

```typescript
async calculateDiff(newQuestions: ExternalQuestion[]): Promise<DiffResult> {
  // 1. 获取当前激活题目集
  const currentQuestions = await this.getCurrentQuestions();
  
  // 2. 比较 ID 集合
  const currentIds = new Set(currentQuestions.map(q => q.id));
  const newIds = new Set(newQuestions.map(q => q.id));
  
  const added = newQuestions.filter(q => !currentIds.has(q.id));
  const removed = currentQuestions.filter(q => !newIds.has(q.id));
  
  // 3. 检查内容变化
  const modified: ModifiedQuestion[] = [];
  for (const newQ of newQuestions) {
    if (currentIds.has(newQ.id)) {
      const currentQ = currentQuestions.find(q => q.id === newQ.id);
      if (currentQ && (currentQ.text !== newQ.text || 
          JSON.stringify(currentQ.options) !== JSON.stringify(newQ.options))) {
        modified.push({
          id: newQ.id,
          oldText: currentQ.text,
          newText: newQ.text,
          oldOptions: currentQ.options,
          newOptions: newQ.options
        });
      }
    }
  }
  
  // 4. 判断是否需要版本升级
  const needsVersionBump = added.length > 0 || removed.length > 0 || modified.length > 0;
  
  return {
    added: added.length,
    removed: removed.length,
    modified: modified.length,
    needsVersionBump,
    newQuestions
  };
}
```

---

### 5.4 服务依赖图

```
┌─────────────────────────────────────────────────────────────┐
│                      Routes Layer                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │questions │  │  test    │  │  sync    │  │  health  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌─────┐
│QuestionService│ │ScoringService │ │ SyncService   │ │ DB  │
└───────────────┘ └───────┬───────┘ └───────┬───────┘ └─────┘
        │                 │                 │
        │                 │                 │
        ▼                 │                 │
┌───────────────┐         │                 │
│ Prisma Client │◄────────┼─────────────────┘
└───────────────┘         │
                          │
                          ▼
                  ┌───────────────┐
                  │ SyncService   │ (Circuit Breaker)
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────────────────┐
                  │ 16personalities-api.com   │
                  └───────────────────────────┘
```

---

## 6. 定时任务设计

### 6.1 Cron 配置

**文件**: `src/jobs/syncJob.ts`

```typescript
import cron from 'node-cron';
import { SyncService } from '../services/syncService';

const SYNC_CRON_EXPRESSION = '0 0 * * *'; // 每天 UTC 00:00

export function startSyncJob(syncService: SyncService): cron.ScheduledTask {
  return cron.schedule(SYNC_CRON_EXPRESSION, async () => {
    const startTime = Date.now();
    console.log(`[SyncJob] Starting scheduled sync at ${new Date().toISOString()}`);
    
    try {
      const result = await syncService.syncQuestions('cron');
      const duration = Date.now() - startTime;
      console.log(`[SyncJob] Sync completed in ${duration}ms`, result);
    } catch (error) {
      console.error('[SyncJob] Sync failed:', error);
    }
  }, {
    scheduled: true,
    timezone: 'UTC'
  });
}
```

**Cron 表达式解析**:

| 字段 | 值 | 含义 |
|------|-----|------|
| 分 | 0 | 第 0 分钟 |
| 时 | 0 | 第 0 小时 (UTC) |
| 日 | * | 每天 |
| 月 | * | 每月 |
| 周 | * | 每周 |

### 6.2 执行流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNCHRONOUS SYNC FLOW                            │
└─────────────────────────────────────────────────────────────────────┘

Cron 触发 (UTC 00:00)
        │
        ▼
┌─────────────────────┐
│ 检查是否已有同步    │──► 是 ──► 跳过，记录日志
│ 正在进行            │
└─────────────────────┘
        │ 否
        ▼
┌─────────────────────┐
│ 检查熔断器状态      │──► OPEN ──► 跳过，记录日志
│                     │    (冷却期内)
└─────────────────────┘
        │ CLOSED 或 HALF-OPEN
        ▼
┌─────────────────────┐
│ 创建 SyncLog        │
│ status: pending     │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│ GET 16p API         │──► 超时/错误 ──► 记录失败
│ /questions          │                 │
└─────────────────────┘                 ▼
        │                        ┌─────────────────────┐
        │ 成功                   │ 更新 SyncLog        │
        ▼                        │ status: failed      │
┌─────────────────────┐          │ 增加失败计数        │
│ Zod 验证响应        │──► 失败 ─►│ 检查是否需要开熔断 │
└─────────────────────┘          └─────────────────────┘
        │ 通过
        ▼
┌─────────────────────┐
│ 计算 Diff           │
│ 与当前题目集比较    │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│ 有变化?             │──► 否 ──► 记录日志，跳过版本升级
└─────────────────────┘
        │ 是
        ▼
┌─────────────────────┐
│ 开启事务            │
│ ┌─────────────────┐ │
│ │ 1. 创建新的     │ │
│ │    QuestionSet  │ │
│ │    Version      │ │
│ │ 2. 插入新题目   │ │
│ │ 3. 标记旧版本   │ │
│ │    isActive=    │ │
│ │    false        │ │
│ │ 4. 更新 SyncLog │ │
│ └─────────────────┘ │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│ 重置熔断器          │
│ state: closed       │
│ failureCount: 0     │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│ 记录成功日志        │
│ 返回结果            │
└─────────────────────┘
```

### 6.3 失败处理

| 失败类型 | 处理方式 | 重试策略 |
|----------|----------|----------|
| 网络超时 | 记录失败，增加计数 | 等待下次 cron |
| 响应验证失败 | 记录失败，不增加计数 | 等待下次 cron |
| 数据库错误 | 记录失败，事务回滚 | 等待下次 cron |
| 连续 3 次失败 | 开启熔断器 | 6 小时后自动重试 |

**失败记录结构**:

```typescript
interface SyncFailureRecord {
  syncLogId: string;
  timestamp: Date;
  errorType: 'NETWORK' | 'VALIDATION' | 'DATABASE' | 'UNKNOWN';
  errorMessage: string;
  stackTrace?: string;
  retryScheduledAt?: Date;
}
```

---

## 7. 中间件设计

### 7.1 速率限制策略

**文件**: `src/middleware/rateLimiter.ts`

**策略配置**:

```typescript
export const RATE_LIMIT_CONFIG = {
  // 提交端点限制
  submit: {
    windowMs: 60 * 60 * 1000,  // 1 小时窗口
    maxRequests: 10,            // 最多 10 次请求
    keyGenerator: (request: FastifyRequest) => {
      // 优先使用代理传递的真实 IP
      const forwarded = request.headers['x-forwarded-for'];
      if (typeof forwarded === 'string') {
        return forwarded.split(',')[0].trim();
      }
      return request.ip;
    },
    skipCondition: (request: FastifyRequest) => {
      // 健康检查端点不限速
      return request.url === '/api/health';
    }
  },
  
  // 其他端点限制（可选）
  general: {
    windowMs: 60 * 1000,  // 1 分钟窗口
    maxRequests: 100      // 最多 100 次请求
  }
} as const;
```

**中间件实现**:

```typescript
import { FastifyRequest, FastifyReply } from 'fastify';
import { RateLimiter } from '../utils/rateLimiter';

export async function rateLimitMiddleware(
  request: FastifyRequest,
  reply: FastifyReply,
  limiter: RateLimiter,
  config: typeof RATE_LIMIT_CONFIG.submit
): Promise<void> {
  const key = config.keyGenerator(request);
  
  if (config.skipCondition?.(request)) {
    return;
  }
  
  const result = await limiter.checkLimit(key);
  
  if (!result.allowed) {
    const retryAfter = Math.ceil(result.resetTime / 1000);
    
    reply.header('Retry-After', retryAfter);
    reply.header('X-RateLimit-Limit', config.maxRequests);
    reply.header('X-RateLimit-Remaining', 0);
    reply.header('X-RateLimit-Reset', result.resetTime);
    
    throw {
      statusCode: 429,
      code: 'RATE_LIMIT_EXCEEDED',
      message: `请求过于频繁，请 ${Math.ceil(result.retryAfter / 60000)} 分钟后重试`,
      details: {
        retryAfterSeconds: result.retryAfter / 1000,
        limit: config.maxRequests,
        window: '1h'
      }
    };
  }
  
  reply.header('X-RateLimit-Limit', config.maxRequests);
  reply.header('X-RateLimit-Remaining', result.remaining);
  reply.header('X-RateLimit-Reset', result.resetTime);
}
```

**请求指纹识别（防滥用增强）**:

```typescript
interface RequestFingerprint {
  ipHash: string;          // IP 的 SHA-256 哈希
  answerPatternHash: string; // 答案模式的哈希（用于检测异常模式）
  timestamp: number;
}

export function generateFingerprint(
  ip: string,
  answers: Array<{ id: string; value: number }>
): RequestFingerprint {
  const encoder = new TextEncoder();
  
  // IP 哈希
  const ipHash = crypto.subtle.digestSync(
    'SHA-256',
    encoder.encode(ip)
  );
  
  // 答案模式哈希（用于检测重复提交）
  const answerPattern = answers
    .map(a => `${a.id}:${a.value}`)
    .sort()
    .join('|');
  const answerPatternHash = crypto.subtle.digestSync(
    'SHA-256',
    encoder.encode(answerPattern)
  );
  
  return {
    ipHash: Buffer.from(ipHash).toString('hex'),
    answerPatternHash: Buffer.from(answerPatternHash).toString('hex'),
    timestamp: Date.now()
  };
}
```

---

### 7.2 错误处理格式

**文件**: `src/middleware/errorHandler.ts`

**错误类型定义**:

```typescript
export interface AppError {
  statusCode: number;
  code: string;
  message: string;
  details?: unknown;
  isOperational: boolean;  // 是否为可预期的操作错误
}

export class ValidationError extends Error implements AppError {
  statusCode = 400;
  code = 'VALIDATION_ERROR';
  isOperational = true;
  
  constructor(message: string, details?: unknown) {
    super(message);
    this.details = details;
  }
}

export class NotFoundError extends Error implements AppError {
  statusCode = 404;
  code = 'NOT_FOUND';
  isOperational = true;
  
  constructor(resource: string) {
    super(`${resource} 不存在`);
    this.message = this.message;
  }
}

export class ExternalApiError extends Error implements AppError {
  statusCode = 503;
  code = 'EXTERNAL_API_UNAVAILABLE';
  isOperational = true;
  
  constructor(message: string = '外部服务暂时不可用') {
    super(message);
  }
}

export class CircuitBreakerError extends Error implements AppError {
  statusCode = 503;
  code = 'CIRCUIT_BREAKER_OPEN';
  isOperational = true;
  
  constructor(cooldownEndsAt: Date) {
    const minutesLeft = Math.ceil((cooldownEndsAt.getTime() - Date.now()) / 60000);
    super(`服务正在恢复中，请 ${minutesLeft} 分钟后重试`);
    this.details = { cooldownEndsAt: cooldownEndsAt.toISOString() };
  }
}
```

**全局错误处理器**:

```typescript
import { FastifyError, FastifyRequest, FastifyReply } from 'fastify';
import { ZodError } from 'zod';
import { v4 as uuidv4 } from 'uuid';

export async function errorHandler(
  error: FastifyError | AppError | Error,
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const requestId = uuidv4();
  const timestamp = new Date().toISOString();
  
  // Zod 验证错误
  if (error instanceof ZodError) {
    const response: ErrorResponse = {
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: '请求参数验证失败',
        details: error.errors.map(e => ({
          path: e.path.join('.'),
          message: e.message
        })),
        requestId
      },
      meta: { timestamp }
    };
    
    reply.status(400).send(response);
    return;
  }
  
  // 应用错误
  if ('statusCode' in error && 'code' in error) {
    const appError = error as AppError;
    const response: ErrorResponse = {
      success: false,
      error: {
        code: appError.code,
        message: appError.message,
        details: appError.details,
        requestId
      },
      meta: { timestamp }
    };
    
    // 记录非操作错误
    if (!appError.isOperational) {
      console.error('[ErrorHandler] Non-operational error:', {
        requestId,
        error: appError
      });
    }
    
    reply.status(appError.statusCode).send(response);
    return;
  }
  
  // 未知错误
  console.error('[ErrorHandler] Unknown error:', {
    requestId,
    error: error instanceof Error ? error.message : String(error),
    stack: error instanceof Error ? error.stack : undefined
  });
  
  const response: ErrorResponse = {
    success: false,
    error: {
      code: 'INTERNAL_ERROR',
      message: '服务器内部错误',
      requestId
    },
    meta: { timestamp }
  };
  
  reply.status(500).send(response);
}
```

---

### 7.3 请求校验

**文件**: `src/schemas/validation.ts`

**Zod Schemas**:

```typescript
import { z } from 'zod';

// ==================== 基础类型 ====================

export const GenderSchema = z.enum(['Male', 'Female', 'Other']);

export const AnswerValueSchema = z.number().int().min(-3).max(3);

export const UuidSchema = z.string().uuid();

export const Base64IdSchema = z.string().min(1);

// ==================== API 请求 Schemas ====================

export const SubmitTestRequestSchema = z.object({
  questionSetVersion: z.number().int().positive(),
  gender: GenderSchema,
  answers: z.array(
    z.object({
      id: Base64IdSchema,
      value: AnswerValueSchema
    })
  ).min(1, '至少需要一个答案').max(200, '答案数量超出限制')
}).strict();

export type SubmitTestRequest = z.infer<typeof SubmitTestRequestSchema>;

// ==================== API 响应 Schemas ====================

export const QuestionOptionSchema = z.object({
  label: z.string(),
  value: z.number().int(),
  order: z.number().int()
});

export const QuestionSchema = z.object({
  id: z.string(),
  text: z.string(),
  order: z.number().int(),
  options: z.array(QuestionOptionSchema).length(7, '必须有 7 个选项')
});

export const QuestionsResponseSchema = z.object({
  success: z.literal(true),
  data: z.object({
    questions: z.array(QuestionSchema),
    total: z.number().int(),
    questionSetVersion: z.number().int()
  }),
  meta: z.object({
    timestamp: z.string(),
    requestId: z.string(),
    questionSetVersion: z.number().int()
  })
});

export const ScaleSchema = z.object({
  scaleKey: z.string(),
  score: z.number(),
  percentage: z.number()
});

export const TraitSchema = z.object({
  traitKey: z.string(),
  label: z.string(),
  color: z.string().nullable(),
  score: z.number(),
  percentage: z.number(),
  isReverse: z.boolean()
});

export const PersonalitySchema = z.object({
  niceName: z.string(),
  fullCode: z.string(),
  avatarSrc: z.string().url().nullable(),
  avatarSrcStatic: z.string().url().nullable(),
  avatarAlt: z.string().nullable(),
  snippet: z.string().nullable()
});

export const TestResultResponseSchema = z.object({
  success: z.literal(true),
  data: z.object({
    id: z.string().uuid(),
    personality: PersonalitySchema,
    scales: z.array(ScaleSchema),
    traits: z.array(TraitSchema),
    createdAt: z.string().datetime().optional()
  }),
  meta: z.object({
    timestamp: z.string(),
    requestId: z.string()
  })
});

// ==================== 外部 API 响应 Schemas ====================

export const ExternalQuestionOptionSchema = z.object({
  label: z.string(),
  value: z.number().int().min(-3).max(3)
});

export const ExternalQuestionSchema = z.object({
  id: z.string().min(1),
  text: z.string().min(1),
  options: z.array(ExternalQuestionOptionSchema).length(7, '每个问题必须有 7 个选项')
});

export const ExternalQuestionsResponseSchema = z.object({
  questions: z.array(ExternalQuestionSchema)
});

export const ExternalTraitSchema = z.object({
  key: z.string(),
  label: z.string(),
  color: z.string().optional(),
  score: z.number(),
  pct: z.number(),
  trait: z.string(),
  reverse: z.boolean().optional(),
  link: z.string().optional(),
  titles: z.record(z.string()).optional(),
  description: z.string().optional(),
  snippet: z.string().optional(),
  imageAlt: z.string().optional(),
  imageSrc: z.string().optional()
});

export const ExternalPersonalityResponseSchema = z.object({
  niceName: z.string(),
  fullCode: z.string(),
  avatarSrc: z.string().optional(),
  avatarSrcStatic: z.string().optional(),
  avatarAlt: z.string().optional(),
  snippet: z.string().optional(),
  scales: z.array(z.object({
    key: z.string(),
    score: z.number(),
    pct: z.number()
  })).optional(),
  traits: z.array(ExternalTraitSchema).optional()
});

// ==================== 验证函数 ====================

export function validateQuestions(questions: unknown): {
  success: boolean;
  data?: z.infer<typeof ExternalQuestionsResponseSchema>;
  errors?: z.ZodError;
} {
  const result = ExternalQuestionsResponseSchema.safeParse({ questions });
  return result;
}

export function validateEachQuestion(questions: unknown[]): {
  valid: boolean;
  errors: Array<{ index: number; error: string }>;
} {
  const errors: Array<{ index: number; error: string }> = [];
  
  questions.forEach((q, index) => {
    const result = ExternalQuestionSchema.safeParse(q);
    if (!result.success) {
      errors.push({
        index,
        error: result.error.errors.map(e => `${e.path.join('.')}: ${e.message}`).join('; ')
      });
    }
  });
  
  return {
    valid: errors.length === 0,
    errors
  };
}
```

**路由中验证使用**:

```typescript
// src/routes/test.ts
import { SubmitTestRequestSchema } from '../schemas/validation';

fastify.post('/api/test/submit', {
  schema: {
    body: SubmitTestRequestSchema
  }
}, async (request, reply) => {
  // request.body 已经过 Zod 验证
  const body = request.body as SubmitTestRequest;
  // ...
});
```

---

## 8. 启动与初始化流程

### 8.1 启动序列

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SERVER STARTUP SEQUENCE                        │
└─────────────────────────────────────────────────────────────────────┘

[1] 加载环境变量
    │
    ├─► 检查必需环境变量
    │   DATABASE_URL ✓
    │   ADMIN_TOKEN ✓
    │   NODE_ENV ✓
    │
    └─► 加载可选环境变量
        PORT (default: 3000)
        HOST (default: 0.0.0.0)
        LOG_LEVEL (default: info)
        │
        ▼
[2] 初始化数据库连接
    │
    ├─► 创建 Prisma Client
    │
    ├─► 执行连接测试
    │   └─► SELECT 1
    │
    └─► 运行待执行的迁移
        prisma migrate deploy
        │
        ▼
[3] 初始化服务层
    │
    ├─► QuestionService
    │
    ├─► SyncService
    │   └─► 从 SyncLog 恢复熔断器状态
    │
    └─► ScoringService
        │
        ▼
[4] 检查初始数据
    │
    ├─► 查询是否有激活的题目集
    │   │
    │   ├─► 有: 设置 serverReady = true
    │   │
    │   └─► 无: 设置 serverReady = false
    │       触发后台同步
    │       │
    │       ▼
[5] 后台初始化同步 (如果需要)
    │
    ├─► 调用 SyncService.syncQuestions('startup')
    │
    ├─► 成功: 设置 serverReady = true
    │
    └─► 失败: 保持 serverReady = false
        记录错误日志
        设置重试定时器 (每 5 分钟)
        │
        ▼
[6] 注册路由和中间件
    │
    ├─► 全局错误处理器
    │
    ├─► 请求 ID 中间件
    │
    ├─► CORS 中间件
    │
    ├─► /api/health
    │
    ├─► /api/questions
    │
    ├─► /api/test/submit (含速率限制)
    │
    ├─► /api/test/result/:id
    │
    ├─► /api/sync (需认证)
    │
    └─► /api/sync/status
        │
        ▼
[7] 启动定时任务
    │
    └─► node-cron: 0 0 * * * (每日同步)
        │
        ▼
[8] 启动 HTTP 服务器
    │
    └─► fastify.listen({ port, host })
        │
        ▼
[9] 输出启动日志
    │
    └─► "Server started on http://0.0.0.0:3000"
        "Ready: true/false"
```

### 8.2 首次部署特殊处理

**场景**: 数据库为空，外部 API 不可用

```
┌─────────────────────────────────────────────────────────────────────┐
│                 FIRST DEPLOYMENT - API DOWN                         │
└─────────────────────────────────────────────────────────────────────┘

启动
  │
  ▼
检查题目集 ────► 无数据
  │
  ▼
serverReady = false
  │
  ▼
后台同步尝试 ────► 失败 (API 不可用)
  │
  ├─► 记录 SyncLog (status: failed)
  │
  └─► 开启熔断器
  │
  ▼
服务器启动成功 (但 serverReady = false)
  │
  ▼
GET /api/health
  │
  └─► 返回:
      {
        "status": "unhealthy",
        "checks": {
          "questions": { "status": "empty", "count": 0 }
        }
      }
  │
  ▼
负载均衡器/监控检测到 unhealthy
不将流量路由到此实例
  │
  ▼
5 分钟后重试同步
  │
  ├─► 熔断器仍在冷却? 跳过
  │
  └─► 冷却结束? 尝试同步
      │
      ├─► 成功: serverReady = true
      │
      └─► 失败: 继续重试
```

**启动代码示例**:

```typescript
// src/index.ts
import Fastify from 'fastify';
import { PrismaClient } from '@prisma/client';
import { QuestionService } from './services/questionService';
import { SyncService } from './services/syncService';
import { ScoringService } from './services/scoringService';
import { startSyncJob } from './jobs/syncJob';
import { registerRoutes } from './routes';
import { config } from './config';

async function main() {
  // [1] 初始化 Fastify
  const fastify = Fastify({
    logger: {
      level: config.logLevel,
      transport: config.nodeEnv === 'development' 
        ? { target: 'pino-pretty' }
        : undefined
    }
  });
  
  // [2] 初始化数据库
  const prisma = new PrismaClient();
  await prisma.$queryRaw`SELECT 1`;
  fastify.log.info('Database connected');
  
  // [3] 初始化服务
  const questionService = new QuestionService(prisma);
  const syncService = new SyncService(prisma);
  const scoringService = new ScoringService(prisma, syncService, questionService);
  
  // [4] 检查初始数据
  let serverReady = false;
  const hasQuestions = await questionService.hasActiveQuestionSet();
  
  if (!hasQuestions) {
    fastify.log.warn('No active question set found, starting initial sync...');
    
    // 不阻塞启动，后台执行
    syncService.syncQuestions('startup')
      .then(result => {
        if (result.success) {
          serverReady = true;
          fastify.log.info('Initial sync completed successfully');
        } else {
          fastify.log.error(`Initial sync failed: ${result.errorMessage}`);
          scheduleRetry();
        }
      })
      .catch(err => {
        fastify.log.error(err, 'Initial sync failed with error');
        scheduleRetry();
      });
  } else {
    serverReady = true;
    fastify.log.info('Active question set found, server ready');
  }
  
  // 重试调度器
  function scheduleRetry() {
    setTimeout(async () => {
      if (await questionService.hasActiveQuestionSet()) {
        serverReady = true;
        return;
      }
      
      if (syncService.canSync()) {
        fastify.log.info('Retrying initial sync...');
        try {
          const result = await syncService.syncQuestions('startup');
          if (result.success) {
            serverReady = true;
          } else {
            scheduleRetry();
          }
        } catch {
          scheduleRetry();
        }
      } else {
        scheduleRetry();
      }
    }, 5 * 60 * 1000); // 5 分钟
  }
  
  // 装饰器，供健康检查使用
  fastify.decorate('isServerReady', () => serverReady);
  fastify.decorate('prisma', prisma);
  fastify.decorate('questionService', questionService);
  fastify.decorate('syncService', syncService);
  fastify.decorate('scoringService', scoringService);
  
  // [6] 注册路由
  await registerRoutes(fastify);
  
  // [7] 启动定时任务
  startSyncJob(syncService);
  
  // [8] 启动服务器
  await fastify.listen({ port: config.port, host: config.host });
  fastify.log.info(`Server started on http://${config.host}:${config.port}`);
  fastify.log.info(`Ready: ${serverReady}`);
}

main().catch(err => {
  console.error('Fatal error during startup:', err);
  process.exit(1);
});
```

---

## 9. 错误处理规范

### 9.1 错误码表

见 [4.4 错误码总表](#44-错误码总表)

### 9.2 外部 API 故障降级策略

```
┌─────────────────────────────────────────────────────────────────────┐
│                  EXTERNAL API FAILURE HANDLING                      │
└─────────────────────────────────────────────────────────────────────┘

                    ┌───────────────────┐
                    │  API 调用失败     │
                    └───────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ 记录失败到 SyncLog│
                    │ failureCount++    │
                    └───────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ failureCount >= 3?│
                    └───────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │ No                            │ Yes
              ▼                               ▼
    ┌───────────────────┐         ┌───────────────────┐
    │ 返回错误给客户端  │         │ 开启熔断器        │
    │ EXTERNAL_API_     │         │ state: open       │
    │ UNAVAILABLE       │         │ cooldownEndsAt:   │
    └───────────────────┘         │ now + 6h          │
                                  └───────────────────┘
                                            │
                                            ▼
                                  ┌───────────────────┐
                                  │ 持久化到 SyncLog  │
                                  │ 确保重启后恢复    │
                                  └───────────────────┘


┌─────────────────────────────────────────────────────────────────────┐
│                  SUBMIT REQUEST DURING CIRCUIT OPEN                 │
└─────────────────────────────────────────────────────────────────────┘

                    ┌───────────────────┐
                    │ POST /api/test/   │
                    │ submit            │
                    └───────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ 检查熔断器状态    │
                    └───────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │ CLOSED                        │ OPEN
              ▼                               ▼
    ┌───────────────────┐         ┌───────────────────┐
    │ 正常调用外部 API  │         │ 检查是否在冷却期  │
    └───────────────────┘         └───────────────────┘
                                            │
                              ┌─────────────┴─────────────┐
                              │ 在冷却期                  │ 已过冷却期
                              ▼                           ▼
                    ┌───────────────────┐   ┌───────────────────┐
                    │ 返回 503          │   │ 切换到 HALF-OPEN  │
                    │ CIRCUIT_BREAKER_  │   │ 允许一次尝试      │
                    │ OPEN              │   └───────────────────┘
                    └───────────────────┘             │
                                                      ▼
                                            ┌───────────────────┐
                                            │ 尝试调用外部 API  │
                                            └───────────────────┘
                                                      │
                                        ┌─────────────┴─────────────┐
                                        │ 成功                      │ 失败
                                        ▼                           ▼
                              ┌───────────────────┐   ┌───────────────────┐
                              │ 切换到 CLOSED     │   │ 切换回 OPEN       │
                              │ 重置失败计数      │   │ 重置冷却计时      │
                              └───────────────────┘   └───────────────────┘
```

### 9.3 错误日志格式

```typescript
interface ErrorLog {
  timestamp: string;           // ISO 8601
  level: 'error' | 'warn' | 'info';
  requestId: string;
  error: {
    code: string;
    message: string;
    stack?: string;
  };
  context: {
    endpoint: string;
    method: string;
    ip?: string;
    userAgent?: string;
  };
  metadata?: Record<string, unknown>;
}

// 示例
const exampleLog: ErrorLog = {
  timestamp: '2026-04-04T12:00:00.000Z',
  level: 'error',
  requestId: '550e8400-e29b-41d4-a716-446655440000',
  error: {
    code: 'EXTERNAL_API_UNAVAILABLE',
    message: 'Connection timeout after 30000ms',
    stack: 'Error: Connection timeout...'
  },
  context: {
    endpoint: '/api/test/submit',
    method: 'POST',
    ip: '192.168.1.100',
    userAgent: 'Mozilla/5.0...'
  },
  metadata: {
    externalApiUrl: 'https://16personalities-api.com/api/personality',
    timeout: 30000,
    attemptNumber: 2
  }
};
```

---

## 10. 安全设计

### 10.1 输入校验

| 字段 | 校验规则 | 实现方式 |
|------|----------|----------|
| questionSetVersion | 正整数 | Zod: `z.number().int().positive()` |
| gender | 枚举值 | Zod: `z.enum(['Male', 'Female', 'Other'])` |
| answers.id | 非空字符串 | Zod: `z.string().min(1)` |
| answers.value | 整数 -3 到 3 | Zod: `z.number().int().min(-3).max(3)` |
| resultId | UUID 格式 | Zod: `z.string().uuid()` |
| adminToken | 与环境变量匹配 | 字符串严格相等比较 |

**额外校验**:

```typescript
// 答案数量校验
const MIN_ANSWERS = 1;
const MAX_ANSWERS = 200;

// 答案去重校验（同一问题不能重复作答）
function checkDuplicateAnswers(answers: Array<{ id: string }>): boolean {
  const ids = answers.map(a => a.id);
  return new Set(ids).size === ids.length;
}
```

### 10.2 速率限制

**限制策略**:

| 端点 | 限制 | 窗口 | 识别方式 |
|------|------|------|----------|
| POST /api/test/submit | 10 次 | 1 小时 | IP 地址 |
| POST /api/sync | 5 次 | 1 分钟 | IP + Token |
| GET /api/* | 100 次 | 1 分钟 | IP 地址 |

**IP 获取优先级**:

1. `X-Forwarded-For` 头的第一个 IP
2. `X-Real-IP` 头
3. `request.ip` (socket remote address)

**防绕过措施**:

```typescript
// 检测异常请求模式
function detectAnomalousPattern(request: FastifyRequest): boolean {
  const userAgent = request.headers['user-agent'];
  
  // 空或可疑 User-Agent
  if (!userAgent || userAgent.length < 10) {
    return true;
  }
  
  // 已知的自动化工具签名
  const suspiciousPatterns = [
    /curl/i,
    /wget/i,
    /python-requests/i,
    /postman/i,
    /insomnia/i
  ];
  
  return suspiciousPatterns.some(p => p.test(userAgent));
}
```

### 10.3 CORS 配置

```typescript
// src/config/index.ts
export const corsConfig = {
  origin: (origin: string, callback: (err: Error | null, allow?: boolean) => void) => {
    const allowedOrigins = config.nodeEnv === 'production'
      ? [config.clientUrl]  // 生产环境：仅允许客户端域名
      : ['http://localhost:3000', 'http://localhost:5173'];  // 开发环境
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'), false);
    }
  },
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'X-Admin-Token'],
  exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining', 'X-RateLimit-Reset', 'Retry-After'],
  credentials: false,
  maxAge: 86400  // 24 小时预检缓存
};
```

### 10.4 敏感信息处理

| 信息类型 | 存储方式 | 日志处理 |
|----------|----------|----------|
| IP 地址 | SHA-256 哈希后存储 | 仅记录前 3 段 (192.168.x.xxx) |
| User-Agent | 原文存储，定期清理 | 截断至 100 字符 |
| 管理员令牌 | 环境变量，不存储 | 不记录到日志 |
| 外部 API 密钥 | 不使用（16p API 无需认证） | N/A |
| 测试答案 | 不存储答案详情 | 不记录 |

**IP 脱敏函数**:

```typescript
function anonymizeIp(ip: string): string {
  const parts = ip.split('.');
  if (parts.length === 4) {
    return `${parts[0]}.${parts[1]}.x.xxx`;
  }
  // IPv6 处理
  if (ip.includes(':')) {
    const segments = ip.split(':');
    return `${segments[0]}:${segments[1]}:x:x:x:x:x:x`;
  }
  return 'unknown';
}
```

**日志敏感信息过滤**:

```typescript
const sensitiveFields = ['password', 'token', 'secret', 'key', 'credential'];

function filterSensitiveData(obj: Record<string, unknown>): Record<string, unknown> {
  const filtered: Record<string, unknown> = {};
  
  for (const [key, value] of Object.entries(obj)) {
    if (sensitiveFields.some(f => key.toLowerCase().includes(f))) {
      filtered[key] = '[REDACTED]';
    } else if (typeof value === 'object' && value !== null) {
      filtered[key] = filterSensitiveData(value as Record<string, unknown>);
    } else {
      filtered[key] = value;
    }
  }
  
  return filtered;
}
```

---

## 11. 依赖清单

### 11.1 运行时依赖

```json
{
  "dependencies": {
    "@fastify/cors": "^10.0.0",
    "@fastify/sensible": "^6.0.0",
    "@prisma/client": "^6.0.0",
    "fastify": "^5.0.0",
    "node-cron": "^3.0.3",
    "pino": "^9.0.0",
    "pino-pretty": "^11.0.0",
    "undici": "^6.21.0",
    "uuid": "^10.0.0",
    "zod": "^3.23.0"
  }
}
```

| 包名 | 版本范围 | 用途 |
|------|----------|------|
| fastify | ^5.0.0 | Web 框架核心 |
| @fastify/cors | ^10.0.0 | CORS 中间件 |
| @fastify/sensible | ^6.0.0 | HTTP 错误工具 |
| @prisma/client | ^6.0.0 | 数据库 ORM 客户端 |
| node-cron | ^3.0.3 | 定时任务调度 |
| pino | ^9.0.0 | 日志框架 |
| pino-pretty | ^11.0.0 | 开发环境日志格式化 |
| undici | ^6.21.0 | HTTP 客户端（比 fetch 更快） |
| uuid | ^10.0.0 | UUID 生成 |
| zod | ^3.23.0 | 运行时类型验证 |

### 11.2 开发依赖

```json
{
  "devDependencies": {
    "@types/node": "^22.0.0",
    "@types/node-cron": "^3.0.11",
    "@types/uuid": "^10.0.0",
    "eslint": "^9.0.0",
    "prisma": "^6.0.0",
    "typescript": "^5.4.0",
    "tsx": "^4.10.0",
    "@typescript-eslint/eslint-plugin": "^8.0.0",
    "@typescript-eslint/parser": "^8.0.0"
  },
  "type": "module"
}
```

| 包名 | 版本范围 | 用途 |
|------|----------|------|
| typescript | ^5.4.0 | TypeScript 编译器 |
| @types/* | * | 类型定义 |
| prisma | ^6.0.0 | Prisma CLI（迁移、生成） |
| tsx | ^4.10.0 | TypeScript 直接执行（开发） |
| eslint | ^9.0.0 | 代码检查 |
| @typescript-eslint/* | ^8.0.0 | TypeScript ESLint 插件 |

### 11.3 package.json 完整示例

```json
{
  "name": "@mbti-test/server",
  "version": "1.0.0",
  "description": "MBTI personality test backend server",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "start": "node dist/index.js",
    "build": "tsc",
    "db:generate": "prisma generate",
    "db:push": "prisma db push",
    "db:migrate": "prisma migrate dev",
    "db:migrate:prod": "prisma migrate deploy",
    "db:seed": "tsx prisma/seed.ts",
    "db:studio": "prisma studio",
    "lint": "eslint src --ext .ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@fastify/cors": "^10.0.0",
    "@fastify/sensible": "^6.0.0",
    "@prisma/client": "^6.0.0",
    "fastify": "^5.0.0",
    "node-cron": "^3.0.3",
    "pino": "^9.0.0",
    "pino-pretty": "^11.0.0",
    "undici": "^6.21.0",
    "uuid": "^10.0.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "@types/node-cron": "^3.0.11",
    "@types/uuid": "^10.0.0",
    "@typescript-eslint/eslint-plugin": "^8.0.0",
    "@typescript-eslint/parser": "^8.0.0",
    "eslint": "^9.0.0",
    "prisma": "^6.0.0",
    "tsx": "^4.10.0",
    "typescript": "^5.4.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

---

## 12. 实施计划

### 12.1 实施阶段

```
┌─────────────────────────────────────────────────────────────────────┐
│                     IMPLEMENTATION PHASES                           │
└─────────────────────────────────────────────────────────────────────┘

Phase 1: 基础设施 (Day 1)
│
├─► [1.1] 初始化项目结构
│   ├── 创建目录结构
│   ├── 配置 package.json
│   ├── 配置 tsconfig.json
│   └── 验收: npm install 无错误
│
├─► [1.2] 配置 Prisma
│   ├── 创建 schema.prisma
│   ├── 运行 prisma generate
│   └── 验收: 类型生成成功
│
├─► [1.3] 创建配置模块
│   ├── 环境变量加载
│   ├── 常量定义
│   └── 验收: config 模块可导入
│
└─► [1.4] 创建基础 Fastify 应用
    ├── 最小可运行服务器
    ├── 健康检查端点
    └── 验收: 服务器启动，/api/health 返回 200

Phase 2: 数据层 (Day 1-2)
│
├─► [2.1] 实现数据库迁移
│   ├── 创建初始迁移
│   ├── 测试迁移可逆性
│   └── 验收: prisma migrate dev 成功
│
├─► [2.2] 实现 QuestionService
│   ├── getActiveQuestions
│   ├── getActiveVersion
│   ├── validateAnswerIds
│   └── 验收: 单元测试通过
│
└─► [2.3] 实现 SyncService (基础)
    ├── 熔断器状态机
    ├── 外部 API 调用
    └── 验收: 可调用 16p API 并解析响应

Phase 3: 核心功能 (Day 2-3)
│
├─► [3.1] 实现同步逻辑
│   ├── diff 算法
│   ├── 事务处理
│   ├── SyncLog 记录
│   └── 验收: 同步可创建新版本
│
├─► [3.2] 实现题目查询端点
│   ├── GET /api/questions
│   ├── Zod 验证
│   └── 验收: 端点返回正确格式
│
├─► [3.3] 实现 ScoringService
│   ├── submitTest
│   ├── getResultById
│   └── 验收: 可提交测试并存储结果
│
└─► [3.4] 实现测试端点
    ├── POST /api/test/submit
    ├── GET /api/test/result/:id
    └── 验收: 完整测试流程可用

Phase 4: 安全与稳定 (Day 3-4)
│
├─► [4.1] 实现速率限制
│   ├── RateLimiter 工具类
│   ├── 中间件集成
│   └── 验收: 超限返回 429
│
├─► [4.2] 实现全局错误处理
│   ├── 错误类型定义
│   ├── 响应格式统一
│   └── 验收: 所有错误返回正确格式
│
├─► [4.3] 实现 CORS
│   ├── 开发/生产配置
│   └── 验收: 跨域请求正常
│
└─► [4.4] 实现定时任务
    ├── cron 调度
    ├── 错误处理
    └── 验收: 定时同步执行

Phase 5: 初始化与部署 (Day 4)
│
├─► [5.1] 实现启动流程
│   ├── 启动序列
│   ├── 首次同步
│   └── 验收: 空数据库启动成功
│
├─► [5.2] 实现手动同步端点
│   ├── POST /api/sync
│   ├── GET /api/sync/status
│   └── 验收: 手动触发可用
│
└─► [5.3] 集成测试
    ├── 完整流程测试
    ├── 错误场景测试
    └── 验收: 所有测试通过
```

### 12.2 详细实施步骤

#### Step 1: 项目初始化

```bash
# 1.1 创建目录结构
mkdir -p server/{src/{routes,services,jobs,middleware,config,schemas,utils},prisma}

# 1.2 初始化 package.json
cd server
npm init -y

# 1.3 安装依赖
npm install fastify @fastify/cors @fastify/sensible @prisma/client node-cron pino pino-pretty undici uuid zod
npm install -D typescript @types/node @types/node-cron @types/uuid prisma tsx eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser

# 1.4 初始化 TypeScript
npx tsc --init

# 1.5 初始化 Prisma
npx prisma init
```

**验收标准**:
- [ ] `npm install` 无错误
- [ ] `npx tsc --version` 输出版本号
- [ ] `npx prisma --version` 输出版本号

---

#### Step 2: 配置 Prisma Schema

```bash
# 2.1 创建 schema.prisma (见 3.1 节完整内容)
# 复制 Prisma Schema 到 prisma/schema.prisma

# 2.2 生成客户端
npx prisma generate

# 2.3 创建初始迁移
npx prisma migrate dev --name init
```

**验收标准**:
- [ ] `prisma generate` 成功
- [ ] `prisma migrate dev` 创建迁移文件
- [ ] `node -e "require('@prisma/client')"` 无错误

---

#### Step 3: 实现配置模块

创建 `src/config/index.ts`:

```typescript
import { config as dotenvConfig } from 'dotenv';
import { z } from 'zod';

// 开发环境加载 .env
if (process.env.NODE_ENV !== 'production') {
  dotenvConfig();
}

const configSchema = z.object({
  nodeEnv: z.enum(['development', 'production', 'test']).default('development'),
  port: z.coerce.number().default(3000),
  host: z.string().default('0.0.0.0'),
  databaseUrl: z.string().url(),
  adminToken: z.string().min(16),
  logLevel: z.enum(['trace', 'debug', 'info', 'warn', 'error', 'fatal']).default('info'),
  clientUrl: z.string().url().optional(),
  
  // 外部 API 配置
  externalApi: z.object({
    baseUrl: z.string().url().default('https://16personalities-api.com'),
    timeout: z.coerce.number().default(30000),
  }),
  
  // 熔断器配置
  circuitBreaker: z.object({
    failureThreshold: z.coerce.number().default(3),
    cooldownDurationMs: z.coerce.number().default(6 * 60 * 60 * 1000),
  }),
  
  // 速率限制配置
  rateLimit: z.object({
    submit: z.object({
      windowMs: z.coerce.number().default(60 * 60 * 1000),
      maxRequests: z.coerce.number().default(10),
    }),
  }),
});

export const config = configSchema.parse({
  nodeEnv: process.env.NODE_ENV,
  port: process.env.PORT,
  host: process.env.HOST,
  databaseUrl: process.env.DATABASE_URL,
  adminToken: process.env.ADMIN_TOKEN,
  logLevel: process.env.LOG_LEVEL,
  clientUrl: process.env.CLIENT_URL,
  externalApi: {
    baseUrl: process.env.EXTERNAL_API_BASE_URL,
    timeout: process.env.EXTERNAL_API_TIMEOUT,
  },
  circuitBreaker: {
    failureThreshold: process.env.CIRCUIT_BREAKER_THRESHOLD,
    cooldownDurationMs: process.env.CIRCUIT_BREAKER_COOLDOWN_MS,
  },
  rateLimit: {
    submit: {
      windowMs: process.env.RATE_LIMIT_WINDOW_MS,
      maxRequests: process.env.RATE_LIMIT_MAX,
    },
  },
});

export type Config = z.infer<typeof configSchema>;
```

**验收标准**:
- [ ] 环境变量缺失时抛出明确错误
- [ ] 所有配置有合理默认值
- [ ] TypeScript 类型推断正确

---

#### Step 4: 实现服务层

按顺序实现:
1. `src/services/questionService.ts`
2. `src/services/syncService.ts`
3. `src/services/scoringService.ts`

**验收标准**:
- [ ] 每个服务可独立导入
- [ ] 构造函数正确接收依赖
- [ ] 公开方法签名与规格一致

---

#### Step 5: 实现路由层

按顺序实现:
1. `src/routes/health.ts`
2. `src/routes/questions.ts`
3. `src/routes/test.ts`
4. `src/routes/sync.ts`

**验收标准**:
- [ ] 每个路由可独立测试
- [ ] 请求/响应格式符合规格
- [ ] 错误返回正确的状态码

---

#### Step 6: 实现中间件

按顺序实现:
1. `src/middleware/errorHandler.ts`
2. `src/middleware/rateLimiter.ts`

**验收标准**:
- [ ] 错误格式统一
- [ ] 速率限制生效

---

#### Step 7: 实现定时任务

实现 `src/jobs/syncJob.ts`

**验收标准**:
- [ ] cron 表达式正确
- [ ] 执行时记录日志

---

#### Step 8: 整合与启动

实现 `src/index.ts`

**验收标准**:
- [ ] 服务器启动成功
- [ ] 健康检查通过
- [ ] 首次同步执行

---

### 12.3 验收检查清单

#### 功能验收

| 检查项 | 预期结果 | 通过标准 |
|--------|----------|----------|
| GET /api/health (有数据) | status: "healthy" | HTTP 200 |
| GET /api/health (无数据) | status: "unhealthy" | HTTP 503 |
| GET /api/questions | 返回题目列表 | HTTP 200, 包含 questionSetVersion |
| POST /api/test/submit (有效) | 创建结果 | HTTP 201, 返回 id |
| POST /api/test/submit (版本不匹配) | 返回错误 | HTTP 400, code: QUESTION_SET_VERSION_MISMATCH |
| POST /api/test/submit (超限) | 返回错误 | HTTP 429, code: RATE_LIMIT_EXCEEDED |
| GET /api/test/result/:id (存在) | 返回结果 | HTTP 200 |
| GET /api/test/result/:id (不存在) | 返回错误 | HTTP 404, code: RESULT_NOT_FOUND |
| POST /api/sync (无 token) | 返回错误 | HTTP 401 |
| POST /api/sync (有 token) | 启动同步 | HTTP 202 |
| GET /api/sync/status | 返回状态 | HTTP 200 |
| 熔断器开启时提交 | 返回错误 | HTTP 503, code: CIRCUIT_BREAKER_OPEN |

#### 非功能验收

| 检查项 | 预期结果 | 测试方法 |
|--------|----------|----------|
| 启动时间 | < 5 秒 | 测量 node 启动到监听 |
| 内存占用 | < 200MB | 进程监控 |
| 并发处理 | 100 req/s | 压力测试工具 |
| 响应时间 (P95) | < 500ms | 性能监控 |

---

## 附录 A: 环境变量模板

```bash
# .env.example

# 服务器配置
NODE_ENV=development
PORT=3000
HOST=0.0.0.0
LOG_LEVEL=debug

# 数据库
DATABASE_URL="file:./dev.db"

# 安全
ADMIN_TOKEN=your-secure-admin-token-min-16-chars

# 客户端 (生产环境必需)
# CLIENT_URL=https://your-client-domain.com

# 外部 API
# EXTERNAL_API_BASE_URL=https://16personalities-api.com
# EXTERNAL_API_TIMEOUT=30000

# 熔断器
# CIRCUIT_BREAKER_THRESHOLD=3
# CIRCUIT_BREAKER_COOLDOWN_MS=21600000

# 速率限制
# RATE_LIMIT_WINDOW_MS=3600000
# RATE_LIMIT_MAX=10
```

---

## 附录 B: TypeScript 配置

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 附录 C: API 请求示例

### 获取题目

```bash
curl -X GET http://localhost:3000/api/questions \
  -H "Accept: application/json"
```

### 提交测试

```bash
curl -X POST http://localhost:3000/api/test/submit \
  -H "Content-Type: application/json" \
  -d '{
    "questionSetVersion": 1,
    "gender": "Male",
    "answers": [
      {"id": "cXVlc3Rpb24tMQ==", "value": 2},
      {"id": "cXVlc3Rpb24tMg==", "value": -1}
    ]
  }'
```

### 获取结果

```bash
curl -X GET http://localhost:3000/api/test/result/f47ac10b-58cc-4372-a567-0e02b2c3d479 \
  -H "Accept: application/json"
```

### 手动同步

```bash
curl -X POST http://localhost:3000/api/sync \
  -H "Content-Type: application/json" \
  -H "X-Admin-Token: your-admin-token"
```

### 查看同步状态

```bash
curl -X GET http://localhost:3000/api/sync/status \
  -H "Accept: application/json"
```

---

**文档结束**
