# 剧灵 StoryAi - 最终技术架构 v1.0

> 已确认决策 | 2026-03-30 | 负责人：Bruce + 小咕嘟

---

## 一、已确认技术决策

| 决策项 | 选择 | 说明 |
|--------|------|------|
| 数据库 | **PostgreSQL** | 多租户、高并发写入、事务支持更强 |
| 文件存储 | **统一 OSS（腾讯云 COS）** | 国内节点，CDN 加速，SDK 成熟 |
| 任务队列 | **抽象层设计** | 短期：BullMQ + Redis；长期：Kafka / Pulsar，接口隔离预留切换空间 |
| 前端 | **全新设计** | 科技感 + 高体验要求，基于 UI 设计最佳实践重新设计 |
| 部署 | 东南亚优先 | 暂定东南亚（延迟低、成本低、覆盖广）|

---

## 二、整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户层                                          │
│  Web (浏览器)  │  H5 (移动端)  │  小程序  │  API (企业客户)                  │
└──────────────┬──────────────┬──────────┬───────────┬────────────────────────┘
               │              │          │           │
           HTTPS          HTTPS     HTTPS      REST API
               │              │          │           │
               ▼              ▼          ▼           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Nginx (负载均衡 + HTTPS)                          │
└──────────────┬─────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        API 网关层 (Express)                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  鉴权中间件  │  │  计费中间件   │  │  限流中间件   │  │  日志中间件   │   │
│  │ AuthMid     │  │ BillingMid   │  │  RateLimit   │  │  Logger      │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
└──────────────┬─────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          业务层 (Express + TypeScript)                       │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                    工作流编排层 (Workflow Orchestrator)                │ │
│  │         一个 API 触发 → 全自动 Pipeline → 实时进度推送                   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │  用户服务   │ │  项目服务   │ │  订阅服务   │ │  积分服务   │          │
│  │UserService  │ │ProjectServ  │ │SubService   │ │ CreditServ  │          │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   模型路由   │  │   质检节点   │  │  后处理服务  │  │   通知服务   │   │
│  │ModelRouter   │  │QualityCheck  │  │PostProcess   │  │ NotifyServ   │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                    现有 82 个业务路由（零改动）                         │ │
│  │  novel/ │ outline/ │ script/ │ storyboard/ │ assets/ │ video/        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└──────────────┬─────────────────────────────────────────────────────────────┘
               │
    ┌──────────┼──────────┬──────────────┐
    ▼          ▼          ▼              ▼
 ┌──────┐  ┌────────┐  ┌────────┐    ┌────────┐
 │Redis │  │PostgreSQL│ │  OSS   │    │ AI模型  │
 │缓存   │  │(多租户) │  │(统一)  │    │  Router │
 │Session│  └────────┘  └────────┘    └────────┘
 └──────┘
    │
    ▼
 ┌────────────────┐
 │   任务队列抽象层   │◄──────── 短期: BullMQ + Redis
 │ TaskQueueAdapter │          长期: Kafka / Pulsar
 └────────────────┘
        │
        ▼
 ┌──────────────────┐
 │  腾讯云 COS (OSS) │
 │  (统一文件存储)   │
 └──────────────────┘
```

---

## 三、任务队列抽象层设计（关键）

### 设计原则

```
┌─────────────────────────────────────────────────────────────┐
│                   TaskQueue Adapter Pattern                  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              TaskQueueInterface (接口层)               │  │
│  │  - enqueue(job)                                        │  │
│  │  - dequeue()                                           │  │
│  │  - getStatus(jobId)                                    │  │
│  │  - cancel(jobId)                                       │  │
│  │  - onProgress(callback)                                 │  │
│  │  - onComplete(callback)                                 │  │
│  │  - onFailed(callback)                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│              ▲                    ▲                    ▲     │
│              │                    │                    │     │
│   ┌───────────┴───┐  ┌────────────┴───┐  ┌───────────┴────┐ │
│   │ BullMQAdapter │  │ KafkaAdapter    │  │ PulsarAdapter │ │
│   │ (短期实现)    │  │ (长期可选)      │  │ (长期可选)     │ │
│   └───────────────┘  └────────────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 接口定义

```typescript
// src/services/queue/interfaces/ITaskQueue.ts

export interface QueueJob {
  id: string;
  type: 'video' | 'postprocess' | 'batch' | 'script' | 'storyboard';
  userId: number;
  projectId: number;
  payload: Record<string, unknown>;
  status: 'pending' | 'processing' | 'completed' | 'failed' | 'cancelled';
  progress: number;      // 0-100
  attempts: number;      // 重试次数
  maxAttempts: number;   // 最大重试次数
  result?: {
    outputUrl?: string;
    error?: string;
    metadata?: Record<string, unknown>;
  };
  createdAt: Date;
  startedAt?: Date;
  completedAt?: Date;
}

export interface ITaskQueue {
  // 初始化
  init(): Promise<void>;

  // 入队
  enqueue(job: Omit<QueueJob, 'id' | 'status' | 'progress' | 'attempts' | 'createdAt'>): Promise<string>;

  // 获取状态
  getStatus(jobId: string): Promise<QueueJob | null>;

  // 取消任务
  cancel(jobId: string): Promise<boolean>;

  // 进度回调
  onProgress(callback: (jobId: string, progress: number) => void): void;

  // 完成回调
  onComplete(callback: (jobId: string, result: QueueJob['result']) => void): void;

  // 失败回调
  onFailed(callback: (jobId: string, error: string) => void): void;

  // 健康检查
  health(): Promise<{ status: 'ok' | 'degraded'; queueSize: number }>;
}
```

### 短期实现（BullMQ + Redis）

```typescript
// src/services/queue/adapters/BullMQAdapter.ts

import { Queue, Worker, Job } from 'bullmq';
import { ITaskQueue, QueueJob } from '../interfaces/ITaskQueue';

export class BullMQAdapter implements ITaskQueue {
  private queue: Queue;
  private worker: Worker;

  async init() {
    // Redis 连接从环境变量读取
    this.queue = new Queue('drama-ai-tasks', {
      connection: { host: process.env.REDIS_HOST, port: 6379 }
    });
    this.worker = new Worker('drama-ai-tasks', async (job: Job) => {
      // 处理任务...
    }, { connection: { host: process.env.REDIS_HOST, port: 6379 } });
  }

  async enqueue(job: Omit<QueueJob, 'id' | 'status' | 'progress' | 'attempts' | 'createdAt'>): Promise<string> {
    return (await this.queue.add('task', job)).id!;
  }

  async getStatus(jobId: string): Promise<QueueJob | null> {
    const job = await this.queue.getJob(jobId);
    return job ? this.mapJobToQueueJob(job) : null;
  }

  async cancel(jobId: string): Promise<boolean> {
    const job = await this.queue.getJob(jobId);
    if (job) { await job.remove(); return true; }
    return false;
  }

  // ... 其他方法实现
}
```

### 长期切换（Kafka / Pulsar）

```typescript
// src/services/queue/adapters/KafkaAdapter.ts
// 使用相同的 ITaskQueue 接口，只需替换 adapter

export class KafkaAdapter implements ITaskQueue {
  // Kafka Consumer + Producer 实现
  // 保持接口完全一致，切换时只需 DI 注入不同的 adapter
}
```

### 业务层使用（透明）

```typescript
// src/routes/workflow/start.ts
// 业务代码完全不感知底层是 BullMQ 还是 Kafka

import { container } from '@/services/queue';

const taskQueue = container.resolve<ITaskQueue>('taskQueue');

const jobId = await taskQueue.enqueue({
  type: 'video',
  userId: req.user.id,
  projectId,
  payload: { scriptId, resolution },
  maxAttempts: 3
});

// 实时进度通过 WebSocket 推送，不影响业务逻辑
taskQueue.onProgress((jobId, progress) => {
  wsManager.send(req.user.id, { type: 'progress', jobId, progress });
});
```

---

## 四、数据库设计（PostgreSQL）

### 迁移原则
- 现有 SQLite 数据**增量迁移**，不影响现有逻辑
- 新增 SaaS 功能以**扩展表**方式添加，不修改现有表结构

### 新增核心表

```sql
-- 用户认证
CREATE TABLE t_users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20) UNIQUE,
  password_hash VARCHAR(255),
  oauth_provider VARCHAR(20),  -- 'wechat' / 'google'
  oauth_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  status VARCHAR(20) DEFAULT 'active'  -- active / suspended
);

-- 订阅套餐
CREATE TABLE t_plans (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) NOT NULL,  -- free / creator / pro / enterprise
  monthly_credit INTEGER NOT NULL,
  price_monthly DECIMAL(10,2),
  features JSONB,  -- { watermark: false, resolution: ['1080p'], batch: true }
  created_at TIMESTAMP DEFAULT NOW()
);

-- 用户订阅
CREATE TABLE t_subscriptions (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES t_users(id),
  plan_id INTEGER REFERENCES t_plans(id),
  status VARCHAR(20) DEFAULT 'active',  -- active / cancelled / expired
  started_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 积分账户
CREATE TABLE t_credits (
  id SERIAL PRIMARY KEY,
  user_id INTEGER INTEGER REFERENCES t_users(id) UNIQUE,
  balance INTEGER DEFAULT 0,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 积分流水
CREATE TABLE t_credit_flow (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES t_users(id),
  type VARCHAR(20) NOT NULL,  -- 'subscribe' / 'purchase' / 'consume' / 'refund'
  amount INTEGER NOT NULL,  -- 正负数
  balance_after INTEGER NOT NULL,
  job_id VARCHAR(100),  -- 关联任务
  description VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 任务记录（扩展 SQLite 的 task 表）
CREATE TABLE t_tasks (
  id SERIAL PRIMARY KEY,
  job_id VARCHAR(100) UNIQUE NOT NULL,  -- 队列 job ID
  user_id INTEGER REFERENCES t_users(id),
  project_id INTEGER REFERENCES t_project(id),
  type VARCHAR(30) NOT NULL,  -- video / postprocess / batch
  status VARCHAR(20) DEFAULT 'pending',
  progress INTEGER DEFAULT 0,
  error TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  started_at TIMESTAMP,
  completed_at TIMESTAMP
);
```

---

## 五、OSS 统一存储方案

### 文件分类

| 类型 | 前缀 | 示例 |
|------|------|------|
| 用户上传 | `/uploads/{userId}/` | `/uploads/123/novel.txt` |
| 生成素材 | `/assets/{projectId}/` | `/assets/456/char_001.png` |
| 生成视频 | `/videos/{projectId}/` | `/videos/456/episode_01.mp4` |
| 字幕文件 | `/subtitles/{projectId}/` | `/subtitles/456/episode_01.srt` |
| 临时文件 | `/temp/{userId}/` | `/temp/123/frame_001.jpg`（24h 自动清理）|

### 上传流程（腾讯云 COS）

```typescript
// src/services/storage/CosUploader.ts
// 使用腾讯云 COS SDK (cos-nodejs-sdk-v5)

import COS from 'cos-nodejs-sdk-v5';

export class CosUploader {
  private cos: COS;

  constructor() {
    this.cos = new COS({
      SecretId: process.env.TENCENT_SECRET_ID,
      SecretKey: process.env.TENCENT_SECRET_KEY,
      Bucket: process.env.TENCENT_BUCKET,
      Region: process.env.TENCENT_REGION  // 如 'ap-shanghai'
    });
  }

  // 统一上传接口，支持断点续传
  async upload(file: Buffer, key: string, options?: {
    contentType?: string;
    maxSize?: number;  // MB，默认 100MB
  }): Promise<string> {
    // 1. 验证文件大小
    // 2. 生成唯一文件名（UUID）
    // 3. 上传到 COS
    // 4. 返回访问 URL（COS 自带 CDN 加速）
  }

  // 生成签名 URL（私有文件访问，有效期可调）
  async signedUrl(key: string, expiresIn: number = 3600): Promise<string>;

  // 删除文件
  async delete(key: string): Promise<void>;

  // 清理临时文件（定时任务，Node cron）
  async cleanupTemp(olderThanHours: number = 24): Promise<void>;
}
```

---

## 六、中间件完整清单

| # | 中间件 | 类型 | 职责 |
|---|--------|------|------|
| M1 | AuthMiddleware | 入口 | JWT 验证 + 多租户隔离 + req.user 挂载 |
| M2 | BillingMiddleware | 入口 | 积分预扣 + 实际扣减 + 不足拦截 |
| M3 | RateLimitMiddleware | 入口 | Redis 滑动窗口限流（per user + per IP）|
| M4 | LoggerMiddleware | 入口 | 请求日志 + 积分流水 + AI 调用成本 |
| M5 | ValidateMiddleware | 路由级 | Zod Schema 校验（扩展到所有新路由）|
| M6 | ProjectMiddleware | 路由级 | 项目归属校验（userId === owner）|
| M7 | CreditMiddleware | 路由级 | 积分消耗预估 + 预锁机制 |
| M8 | SubscriptionMiddleware | 路由级 | 套餐权限校验（高清输出/批量任务等）|

---

## 七、AI 模型扩展计划

### 当前已接入（可直接复用）
- LLM：Gemini / Claude / GPT-4 / DeepSeek / XAI（`ai` SDK）
- 视频：Sora（OpenAI）

### 新增接入计划
| 模型 | 类型 | 优先级 | 用途 |
|------|------|--------|------|
| 豆包（字节）| LLM + 视频 | P0 | 文本理解 + 视频生成 |
| Seedance 2.0 | 视频 | P0 | 主力视频生成 |
| OpenRouter | 聚合平台 | P0 | 统一接入多模型，按需切换 |
| Together.ai | 聚合平台 | P1 | 同上 |
| Seedream | 图片 | P1 | 角色/分镜图生成 |
| 可灵 | 视频 | P1 | 备选视频生成 |
| 火山引擎 TTS | 音频 | P1 | AI 配音 |

---

## 八、前端重构要求（高优先级）

详见：docs/FRONTEND-DESIGN.md（待创建）

核心要求：
- 科技感 UI（深色主题 + 霓虹渐变 + 动态粒子/网格背景）
- 高体验（流畅动画 + 即时反馈 + 渐进式披露）
- 功能不减（82 个现有功能全部保留）
- 新增 SaaS 专属界面（登录/订阅/积分/管理后台）

---

*本文档为最终确认版架构，所有决策已落地，可进入详细设计阶段*