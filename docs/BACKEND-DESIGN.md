# 剧灵 StoryAi - 后端详细设计文档

版本：v1.0 | 日期：2026-03-30 | 状态：设计稿

---

## 一、架构决策（ADR）

### ADR-001：采用 Modular Monolith 架构

**背景**：剧灵目前是单团队（<10人），需要快速迭代，同时有明确的模块边界（用户、订阅、工作流、AI生成）。完全重构成微服务运维成本过高。

**决策**：采用 Modular Monolith 架构，模块内部高内聚，模块间通过清晰接口通信。未来业务增长后可平滑拆分为独立服务。

**模块划分**：
```
src/
├── modules/
│   ├── auth/           # 用户认证、登录、OAuth
│   ├── subscription/   # 订阅、套餐、积分
│   ├── project/        # 项目管理
│   ├── novel/           # 小说上传解析
│   ├── outline/         # 大纲生成
│   ├── script/          # 剧本生成
│   ├── storyboard/      # 分镜管理
│   ├── assets/          # 素材管理
│   ├── video/           # 视频生成
│   ├── workflow/        # 工作流编排（新增）
│   ├── quality/         # 质检节点（新增）
│   ├── notification/    # 通知推送（新增）
│   └── admin/           # 管理后台（新增）
│
├── shared/
│   ├── domain/          # 共享领域模型
│   ├── infrastructure/  # 数据库、缓存、存储、消息队列
│   ├── ai/              # AI 模型路由（扩展现有）
│   ├── middleware/      # 全局中间件
│   └── utils/           # 工具函数
│
└── app.ts               # 应用入口
```

**原则**：
- 模块间禁止直接 import 内部实现，只通过模块 Public API（`api/` 目录）通信
- 每个模块拥有独立的 `domain/` + `infrastructure/` + `api/`
- 共享代码只放在 `shared/`，且只能是纯工具或跨模块通用接口

---

### ADR-002：数据存储策略

**决策**：
- **PostgreSQL**：所有业务数据（用户、项目、任务、配置）
- **Redis**：Session、缓存、任务队列（短期 BullMQ / 长期 Kafka 抽象）
- **COS/S3/OSS**：所有文件（用户上传、生成素材、成品视频）

**PostgreSQL 扩展策略**：
- 现有 SQLite 表通过扩字段迁移，不重建
- 新增 SaaS 表（users, plans, subscriptions, credits 等）独立创建
- 用户数据隔离：通过 `user_id` 字段 + RLS（Row Level Security）

---

### ADR-003：任务队列抽象层

**决策**：定义 `ITaskQueue` 接口，短期用 BullMQ 实现，长期可切换 Kafka / Pulsar，切换时业务代码零改动。

---

## 二、目录结构设计

```
src/
├── main.ts                          # 应用入口
├── app.ts                           # Express 配置
│
├── modules/                         # 业务模块（模块化）
│
│   ├── auth/                       # 用户认证
│   │   ├── api/                    # 公共接口（模块对外 API）
│   │   │   ├── AuthModule.ts
│   │   │   └── index.ts
│   │   ├── domain/                  # 业务逻辑（内部）
│   │   │   ├── services/
│   │   │   └── entities/
│   │   ├── infrastructure/          # 技术实现（内部）
│   │   │   ├── repositories/
│   │   │   └── providers/
│   │   └── routes/                  # 路由注册（仅注册到 app.ts）
│   │
│   ├── subscription/              # 订阅/积分（同上结构）
│   ├── project/                    # 项目管理
│   ├── workflow/                   # 工作流编排（新增核心）
│   ├── quality/                    # 质检节点（新增）
│   ├── notification/              # 通知推送（新增）
│   └── admin/                      # 管理后台（新增）
│
├── shared/
│   ├── domain/
│   │   ├── BaseEntity.ts           # 所有实体基类
│   │   ├── ValueObject.ts           # 值对象基类
│   │   └── types/                   # 共享类型定义
│   │
│   ├── infrastructure/
│   │   ├── database/
│   │   │   ├── PostgresClient.ts     # PostgreSQL 连接（knex）
│   │   │   ├── migrations/           # 数据库迁移
│   │   │   └── repositories/         # 通用 Repository
│   │   ├── queue/
│   │   │   ├── interfaces/
│   │   │   │   └── ITaskQueue.ts     # 队列接口
│   │   │   ├── adapters/
│   │   │   │   ├── BullMQAdapter.ts  # 短期实现
│   │   │   │   └── KafkaAdapter.ts   # 长期实现（预留）
│   │   │   └── TaskQueueFactory.ts   # 工厂（按配置返回 adapter）
│   │   ├── storage/
│   │   │   ├── interfaces/
│   │   │   │   └── IStorageAdapter.ts
│   │   │   ├── adapters/
│   │   │   │   ├── TencentCosAdapter.ts
│   │   │   │   ├── AwsS3Adapter.ts
│   │   │   │   └── AliyunOssAdapter.ts
│   │   │   └── StorageFactory.ts
│   │   ├── cache/
│   │   │   └── RedisClient.ts
│   │   └── eventBus/
│   │       └── EventBus.ts           # 事件总线（模块间通信）
│   │
│   ├── ai/
│   │   ├── interfaces/
│   │   │   └── IModelAdapter.ts      # AI 模型接口
│   │   ├── adapters/
│   │   │   ├── GoogleGeminiAdapter.ts
│   │   │   ├── ByteDanceAdapter.ts
│   │   │   ├── SeedanceAdapter.ts
│   │   │   ├── OpenRouterAdapter.ts
│   │   │   ├── AwsBedrockAdapter.ts
│   │   │   ├── AliyunDashscopeAdapter.ts
│   │   │   └── SoraAdapter.ts         # 现有（保留）
│   │   └── ModelRouter.ts            # 模型路由决策
│   │
│   ├── middleware/
│   │   ├── AuthMiddleware.ts
│   │   ├── BillingMiddleware.ts
│   │   ├── RateLimitMiddleware.ts
│   │   ├── LoggerMiddleware.ts
│   │   ├── ValidateMiddleware.ts
│   │   ├── ProjectMiddleware.ts
│   │   ├── CreditMiddleware.ts
│   │   └── SubscriptionMiddleware.ts
│   │
│   └── utils/
│       ├── responseFormat.ts        # 统一响应格式（现有）
│       ├── errors.ts                 # 统一错误类
│       └── config.ts                  # 配置读取
│
└── routes/                           # 现有 82 个路由（保留，原样迁移）
    ├── novel/
    ├── outline/
    ├── script/
    ├── storyboard/
    ├── assets/
    ├── video/
    └── ...
```

---

## 三、核心模块设计

### 3.1 工作流编排层（Workflow Module）— 新增核心

**职责**：一个 API 触发整个 Pipeline，自动串联所有环节。

```typescript
// src/modules/workflow/api/WorkflowService.ts

export class WorkflowService {
  constructor(
    private novelService: NovelService,
    private outlineService: OutlineService,
    private scriptService: ScriptService,
    private storyboardService: StoryboardService,
    private videoService: VideoService,
    private postProcessService: PostProcessService,
    private qualityService: QualityService,
    private taskQueue: ITaskQueue,
    private eventBus: EventBus
  ) {}

  /**
   * 启动全自动工作流
   * @param userId 用户ID
   * @param projectId 项目ID
   * @param options 工作流选项（风格/分辨率等）
   * @returns jobId（任务ID，用于查询进度）
   */
  async startFullPipeline(
    userId: number,
    projectId: number,
    options: WorkflowOptions
  ): Promise<string> {
    // 1. 校验用户积分和套餐权限
    // 2. 创建任务记录（t_tasks 表）
    // 3. 入队（TaskQueue）
    // 4. 返回 jobId
    return jobId;
  }
}

// 工作流状态机
export type WorkflowStage =
  | 'pending'
  | 'parsing'        // 小说解析
  | 'outline'       // 大纲生成
  | 'script'         // 剧本生成
  | 'quality_script' // 剧本质检
  | 'storyboard'     // 分镜
  | 'quality_sboard' // 分镜质检
  | 'video'          // 视频生成
  | 'quality_video'   // 视频质检
  | 'postprocess'    // 后处理
  | 'quality_final'  // 最终质检
  | 'completed'
  | 'failed';

// 每个 stage 完成后，WorkflowEngine 自动推进到下一 stage
// 每个 stage 的失败都触发重试（最多 3 次），仍失败则终止并通知用户
```

**WorkflowEngine 执行流程**：

```
用户 POST /workflow/start
       ↓
  [入队] TaskQueue.enqueue()
       ↓
  ┌────────────────────────────────────────────────────────┐
  │              WorkflowEngine.execute(jobId)             │
  │                                                        │
  │  Stage 1: 解析小说 → 质检 → 通过则继续                  │
  │  Stage 2: 生成大纲 → 质检 → 通过则继续                 │
  │  Stage 3: 生成剧本 → 质检 → 通过则继续                 │
  │  Stage 4: 生成角色 → 用户确认（可跳过自动继续）         │
  │  Stage 5: 生成角色一致 → 质检 → 通过则继续              │
  │  Stage 6: 生成分镜 → 质检 → 通过则继续                  │
  │  Stage 7: 生成图片 → 质检 → 通过则继续                  │
  │  Stage 8: 生成视频片段 → 质检 → 通过则继续              │
  │  Stage 9: 后处理（FFmpeg拼接+字幕+音频）               │
  │  Stage 10: 最终质检 → 通过则完成                        │
  │                                                        │
  │  每个 Stage 完成后：                                    │
  │    - 推送 WebSocket 进度 (progress 0-100)               │
  │    - 写入 t_tasks 表状态                               │
  │    - 如质检失败，自动重写（最多3次）                   │
  │                                                        │
  └────────────────────────────────────────────────────────┘
       ↓
  [完成] 推送完成通知 + 保存作品记录
```

---

### 3.2 质检节点（Quality Module）— 新增

```typescript
// src/modules/quality/api/QualityService.ts

export class QualityService {
  // 剧本质检
  async checkScript(script: ScriptJSON): Promise<QualityResult> {
    const checks = [
      { rule: 'logic_gaps', description: '剧情逻辑漏洞' },
      { rule: 'character_consistency', description: '角色行为一致性' },
      { rule: 'emotion_tags', description: '情绪标签完整性' },
      { rule: 'dialogue_length', description: '对白长度合理性' }
    ];

    const results = await Promise.all(
      checks.map(c => this.runCheck(c.rule, script))
    );

    const score = this.calculateScore(results);
    const failed = results.filter(r => !r.passed);

    return { score, failed, canProceed: score >= 60 };
  }

  // 分镜质检
  async checkStoryboard(storyboard: StoryboardJSON): Promise<QualityResult> {
    const checks = [
      { rule: 'script_alignment', description: '与剧本一致性' },
      { rule: 'required_fields', description: '必填字段完整性' },
      { rule: 'prompt_quality', description: 'AI 提示词质量' }
    ];
    // ...
  }

  // 图片质检
  async checkImage(imageUrl: string, characterRefs: CharacterRef[]): Promise<QualityResult> {
    const score = await this.aiScore(imageUrl, characterRefs);
    return { score, failed: score < 70 ? ['image_quality'] : [], canProceed: score >= 70 };
  }

  // 最终成片质检
  async checkFinalVideo(videoUrl: string): Promise<QualityResult> {
    const checks = [
      { rule: 'sync_audio_video', description: '音画同步' },
      { rule: 'subtitle_alignment', description: '字幕对齐' },
      { rule: 'duration', description: '时长合理性' }
    ];
    // ...
  }

  // 自动重写（最多3次）
  async autoRewrite(stage: WorkflowStage, content: any): Promise<any> {
    for (let attempt = 1; attempt <= 3; attempt++) {
      const rewritten = await this.rewrite(stage, content, attempt);
      const quality = await this.check(stage, rewritten);
      if (quality.canProceed) return rewritten;
    }
    return null; // 3次均失败
  }
}
```

---

### 3.3 后处理服务（PostProcess Module）— 新增

```typescript
// src/modules/postprocess/api/PostProcessService.ts

export class PostProcessService {
  constructor(
    private ffmpeg: FFmpegWrapper,
    private tts: TTSAdapter,
    private storage: IStorageAdapter
  ) {}

  async process(job: PostProcessJob): Promise<PostProcessResult> {
    const { videoFragments, script, options } = job;

    // Step 1: 拼接视频
    const mergedVideo = await this.ffmpeg.concat(videoFragments);

    // Step 2: 生成字幕（ASR）
    const audioText = await this.extractAudio(mergedVideo);
    const subtitleJson = await this.asr(audioText); // { text, start, end }
    const srtContent = this.toSRT(subtitleJson);

    // Step 3: 字幕压制
    const videoWithSubtitle = await this.ffmpeg.burnSubtitle(
      mergedVideo,
      srtContent,
      { position: options.subtitlePosition || 'bottom', color: options.subtitleColor || '#FFFFFF' }
    );

    // Step 4: AI 配音（可选）
    let finalVideo = videoWithSubtitle;
    if (options.enableTTS) {
      const ttsAudio = await this.tts.generate(script.dialogue, options.voiceType || 'female_warm');
      const mixedAudio = await this.ffmpeg.mixAudio(videoWithSubtitle, ttsAudio, { bgmVolume: 0.3, voiceVolume: 1.0 });
      finalVideo = mixedAudio;
    }

    // Step 5: BGM（可选）
    if (options.bgmUrl) {
      finalVideo = await this.ffmpeg.addBGM(finalVideo, options.bgmUrl, { volume: 0.3 });
    }

    // Step 6: 转码（分辨率）
    const output = await this.ffmpeg.transcode(finalVideo, options.resolution || '1080p');

    // Step 7: 上传到 COS
    const url = await this.storage.upload(output, `videos/${job.projectId}/${job.episodeId}.mp4`);

    return { url, format: 'mp4', resolution: options.resolution };
  }
}
```

---

### 3.4 订阅与积分（Subscription Module）— 新增

```typescript
// src/modules/subscription/api/

export class SubscriptionService {
  // 获取用户当前套餐
  async getUserPlan(userId: number): Promise<UserPlan> {
    const sub = await this.subscriptionRepo.findActive(userId);
    if (!sub) return { plan: 'free', credits: 0 };
    return { plan: sub.plan, credits: sub.credits };
  }

  // 扣减积分（带事务）
  async consumeCredits(userId: number, amount: number, jobId: string, description: string): Promise<void> {
    await this.db.transaction(async (trx) => {
      const user = await this.userRepo.findById(userId, trx);
      if (user.credits < amount) throw new InsufficientCreditsError();

      await this.userRepo.deductCredits(userId, amount, trx);
      await this.creditFlowRepo.record({
        userId, type: 'consume', amount: -amount,
        balanceAfter: user.credits - amount,
        jobId, description, createdAt: new Date()
      }, trx);
    });
  }

  // 预锁积分（任务开始时）
  async lockCredits(userId: number, amount: number, jobId: string): Promise<void> {
    // 冻结积分（不实际扣减），任务完成后调用 consume 或 unlock
  }

  // 返还预锁积分（任务失败时）
  async unlockCredits(userId: number, amount: number, jobId: string): Promise<void> {}
}
```

---

### 3.5 通知推送（Notification Module）— 新增

```typescript
// src/modules/notification/api/NotificationService.ts

export class NotificationService {
  constructor(
    private wsManager: WebSocketManager,  // 复用 express-ws
    private pushQueue: ITaskQueue
  ) {}

  // WebSocket 推送（实时）
  async pushToUser(userId: number, event: NotificationEvent): Promise<void> {
    const socket = this.wsManager.getSocket(userId);
    if (socket?.readyState === WebSocket.OPEN) {
      socket.send(JSON.stringify(event));
    }
  }

  // 浏览器通知（需用户授权）
  async pushBrowserNotification(userId: number, event: NotificationEvent): Promise<void> {
    // 读取用户偏好，优先 WebSocket，失败则尝试 Browser Notification API
  }

  // 邮件通知（异步）
  async sendEmail(to: string, template: string, data: Record<string, any>): Promise<void> {
    await this.pushQueue.enqueue({
      type: 'notification',
      payload: { method: 'email', to, template, data }
    });
  }
}

// 通知事件类型
export type NotificationEvent =
  | { type: 'task:progress'; jobId: string; progress: number; stage: string }
  | { type: 'task:complete'; jobId: string; outputUrl: string }
  | { type: 'task:failed'; jobId: string; error: string }
  | { type: 'credit:low'; balance: number }
  | { type: 'subscription:expiry'; daysLeft: number };
```

---

## 四、AI 模型路由（扩展现有）

### 4.1 接口定义

```typescript
// src/shared/ai/interfaces/IModelAdapter.ts

export interface IModelAdapter {
  readonly name: string;
  readonly provider: string;
  readonly capabilities: ModelCapability[];

  init(): Promise<void>;
  call(prompt: string, options?: CallOptions): Promise<ModelOutput>;
  health(): Promise<ModelHealth>;
  getCostPerCall(): number;
}

export type ModelCapability =
  | 'script-generation'    // 剧本生成
  | 'outline-generation'     // 大纲生成
  | 'character-analysis'     // 角色分析
  | 'image-generation'        // 图片生成
  | 'video-generation'       // 视频生成
  | 'tts'                    // 语音合成
  | 'asr';                   // 语音识别

export interface CallOptions {
  temperature?: number;
  maxTokens?: number;
  seed?: number;
  model?: string;  // 如 'gemini-2.0-flash'
}

export interface ModelOutput {
  content: string | object;
  usage: { promptTokens: number; completionTokens: number; totalTokens: number };
  latency: number;  // ms
}
```

### 4.2 模型选择策略

```typescript
// src/shared/ai/ModelRouter.ts

export class ModelRouter {
  constructor(
    private config: ModelConfigMap,  // 从 t_config 表读取
    private adapters: Map<string, IModelAdapter>
  ) {}

  async call(
    task: ModelCapability,
    prompt: string,
    context: { userPlan: string; style?: string }
  ): Promise<ModelOutput> {
    // 1. 读取该 task 对应的模型优先级列表
    const modelList = this.config.getModelsForTask(task);

    // 2. 按优先级尝试，失败则自动切换备选
    for (const modelName of modelList) {
      const adapter = this.adapters.get(modelName);
      if (!adapter || !(task in adapter.capabilities)) continue;

      try {
        return await adapter.call(prompt, this.getOptionsForModel(modelName, context));
      } catch (err) {
        // 记录失败，尝试下一个
        this.logFailure(modelName, err);
      }
    }

    throw new NoModelAvailableError(task);
  }
}

// 配置示例（环境变量或 t_config 表）
// GEMINI=google:gemini-2.0-flash
// DOUBAN_LLM=bytedance:doubao-pro
// SEEDANCE=seedance:2.0
// VIDEO_BACKUP=openrouter:seedance
```

### 4.3 现有模型适配器迁移

| 现有代码 | 迁移到 | 说明 |
|---------|--------|------|
| `src/routes/setting/` (模型配置) | `t_config` 表 + `ModelRouter` | 扩展现有配置逻辑 |
| `src/agents/outlineScript/` | `OutlineAdapter` + `ScriptAdapter` | 现有 Agent 作为 Adapter 封装 |
| `src/agents/storyboard/` | `StoryboardAdapter` | 同上 |
| `src/routes/video/generateVideo.ts` | `VideoAdapter` (Seedance / Sora / 豆包) | 扩展现有视频路由 |

---

## 五、中间件设计

### 5.1 执行顺序

```
请求进入
    ↓
[Logger] 记录请求（开始时间 + 请求ID）
    ↓
[RateLimit] Redis 滑动窗口限流（per user）
    ↓
[Auth] JWT 验证 + 解析 userId/plan（公开路由跳过）
    ↓
[Subscription] 校验套餐权限（访问特定功能时）
    ↓
[Project] 校验项目归属（userId === owner）
    ↓
[Validate] Zod Schema 校验参数
    ↓
[Billing] 积分预扣（消耗性操作）
    ↓
[Handler] 业务处理
    ↓
[Logger] 记录响应（状态 + 耗时 + AI成本）
    ↓
响应返回
```

### 5.2 积分计费矩阵

| 操作 | 积分消耗 | 预扣时机 |
|------|---------|---------|
| 小说上传 | 0 | — |
| 剧本生成（Gemini） | 5 | 开始生成 |
| 分镜生成 | 5 | 开始生成 |
| 图片生成（每张） | 3 | 开始生成 |
| 视频生成（每个分镜） | 20 | 开始生成 |
| 后处理（FFmpeg+音频） | 5 | 开始处理 |
| 质检失败自动重写 | 不额外扣 | — |

---

## 六、数据库设计（PostgreSQL）

### 6.1 新增核心表

```sql
-- 用户表
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20) UNIQUE,
  password_hash VARCHAR(255),
  oauth_provider VARCHAR(20),  -- 'wechat', 'google'
  oauth_id VARCHAR(255),
  status VARCHAR(20) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 套餐表
CREATE TABLE plans (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) NOT NULL,  -- 'free', 'creator', 'pro', 'enterprise'
  display_name VARCHAR(100),
  monthly_credits INTEGER NOT NULL,
  price_monthly DECIMAL(10,2),
  features JSONB  -- { watermark: true, resolutions: ['720p'], batch: false }
);

-- 用户订阅
CREATE TABLE subscriptions (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  plan_id INTEGER REFERENCES plans(id),
  status VARCHAR(20) DEFAULT 'active',  -- 'active', 'cancelled', 'expired'
  started_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  auto_renew BOOLEAN DEFAULT false
);

-- 积分账户
CREATE TABLE credits (
  user_id INTEGER PRIMARY KEY REFERENCES users(id),
  balance INTEGER DEFAULT 0 CHECK (balance >= 0),
  locked INTEGER DEFAULT 0,  -- 预锁积分
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 积分流水
CREATE TABLE credit_flow (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  type VARCHAR(20) NOT NULL,  -- 'subscribe', 'purchase', 'consume', 'refund', 'lock', 'unlock'
  amount INTEGER NOT NULL,  -- 正数=增加，负数=减少
  balance_after INTEGER NOT NULL,
  job_id VARCHAR(100),
  description VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);

-- 任务表（扩展开源版的 task 功能）
CREATE TABLE tasks (
  id SERIAL PRIMARY KEY,
  job_id VARCHAR(100) UNIQUE NOT NULL,  -- 队列 job ID（BullMQ）
  user_id INTEGER REFERENCES users(id),
  project_id INTEGER REFERENCES t_project(id),  -- 引用现有表
  type VARCHAR(30) NOT NULL,  -- 'pipeline', 'video', 'postprocess', 'batch'
  stage VARCHAR(30) DEFAULT 'pending',  -- 当前阶段
  status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'processing', 'completed', 'failed', 'cancelled'
  progress INTEGER DEFAULT 0,  -- 0-100
  attempts INTEGER DEFAULT 0,
  error TEXT,
  result JSONB,  -- { outputUrl, metadata }
  created_at TIMESTAMP DEFAULT NOW(),
  started_at TIMESTAMP,
  completed_at TIMESTAMP
);

-- 支付记录
CREATE TABLE payments (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  type VARCHAR(20) NOT NULL,  -- 'subscription', 'credits'
  amount DECIMAL(10,2) NOT NULL,
  currency VARCHAR(10) DEFAULT 'CNY',
  status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'paid', 'failed', 'refunded'
  gateway VARCHAR(20) NOT NULL,  -- 'wechat', 'alipay', 'stripe', 'paypal'
  gateway_order_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  paid_at TIMESTAMP
);

-- 作品表
CREATE TABLE works (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  project_id INTEGER REFERENCES t_project(id),
  task_id INTEGER REFERENCES tasks(id),
  title VARCHAR(200),
  video_url VARCHAR(500),
  resolution VARCHAR(10),  -- '720p', '1080p', '4k'
  watermark BOOLEAN DEFAULT true,
  views INTEGER DEFAULT 0,
  public BOOLEAN DEFAULT false,  -- 是否公开展示
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 6.2 现有表扩展

```sql
-- t_config 表扩展字段（新增 AI 模型配置格式）
-- 现有: id, model, api_key, endpoint, config, type
-- 扩展: 添加 provider, capabilities(JSONB), cost_per_call 等

-- t_project 表扩展（新增多租户字段）
ALTER TABLE t_project ADD COLUMN user_id INTEGER;  -- 关联到 users.id
ALTER TABLE t_project ADD COLUMN plan VARCHAR(20) DEFAULT 'free';
ALTER TABLE t_project ADD COLUMN settings JSONB;  -- 用户个性化设置

-- 其他现有表（t_novel, t_outline, t_script, t_storyboard 等）同样加 user_id
ALTER TABLE t_novel ADD COLUMN user_id INTEGER;
ALTER TABLE t_outline ADD COLUMN user_id INTEGER;
ALTER TABLE t_script ADD COLUMN user_id INTEGER;
ALTER TABLE t_storyboard ADD COLUMN user_id INTEGER;
ALTER TABLE t_assets ADD COLUMN user_id INTEGER;
ALTER TABLE t_video ADD COLUMN user_id INTEGER;
```

---

## 七、API 设计（新增部分）

### 7.1 新增 API 清单

| 方法 | 路由 | 说明 | 鉴权 |
|------|------|------|------|
| POST | `/auth/register` | 邮箱注册 | 否 |
| POST | `/auth/login` | 邮箱登录 | 否 |
| POST | `/auth/wechat` | 微信登录 | 否 |
| POST | `/auth/refresh` | 刷新 Token | 是 |
| GET | `/auth/me` | 获取当前用户 | 是 |
| GET | `/subscription/plans` | 获取套餐列表 | 否 |
| POST | `/subscription/subscribe` | 购买订阅 | 是 |
| POST | `/subscription/cancel` | 取消订阅 | 是 |
| GET | `/credits/balance` | 查询积分余额 | 是 |
| GET | `/credits/flow` | 积分明细列表 | 是 |
| POST | `/credits/purchase` | 购买积分包 | 是 |
| POST | `/workflow/start` | 启动全自动 Pipeline | 是 |
| GET | `/workflow/status/:jobId` | 查询任务状态 | 是 |
| POST | `/workflow/cancel/:jobId` | 取消任务 | 是 |
| GET | `/tasks` | 我的任务列表（分页）| 是 |
| GET | `/tasks/:id` | 任务详情 | 是 |
| GET | `/works` | 我的作品列表 | 是 |
| GET | `/works/:id` | 作品详情 | 是 |
| DELETE | `/works/:id` | 删除作品 | 是 |
| PATCH | `/works/:id/visibility` | 修改可见性 | 是 |
| GET | `/admin/users` | 用户列表（管理）| 是 |
| GET | `/admin/stats` | 数据统计（管理）| 是 |

### 7.2 核心 API 详细设计

**POST /workflow/start**（启动全自动 Pipeline）

Request:
```json
{
  "projectId": 123,
  "styleId": "ancient-sweet",
  "resolution": "1080p",
  "enableTTS": false,
  "bgmStyle": "warm"
}
```

Response:
```json
{
  "code": 0,
  "data": {
    "jobId": "job_abc123",
    "estimatedDuration": 900,  // 秒
    "stages": [
      { "name": "解析小说", "estimatedSeconds": 30 },
      { "name": "生成剧本", "estimatedSeconds": 120 },
      { "name": "生成分镜", "estimatedSeconds": 180 },
      { "name": "生成视频", "estimatedSeconds": 600 },
      { "name": "后处理", "estimatedSeconds": 60 }
    ]
  }
}
```

**WebSocket 推送格式**：
```json
// 进度更新
{ "type": "task:progress", "jobId": "job_abc123", "stage": "video", "progress": 65, "message": "正在生成第 3/8 个分镜..." }

// 完成
{ "type": "task:complete", "jobId": "job_abc123", "outputUrl": "https://cos.xxx/videos/123/episode_01.mp4", "thumbnail": "..." }

// 失败
{ "type": "task:failed", "jobId": "job_abc123", "stage": "video", "error": "视频API超时", "canRetry": true }
```

---

## 八、技术债与分阶段计划

### 8.1 第一阶段（MVP，6-8周）— 最小可运行

**目标**：能跑通完整 Pipeline，有用户系统，有基本计费

- [ ] 数据库迁移（SQLite → PostgreSQL）
- [ ] 用户认证模块（注册/登录/OAuth）
- [ ] 订阅与积分系统
- [ ] Workflow 模块（编排层 + TaskQueue 抽象）
- [ ] BullMQ 实现（Redis）
- [ ] 质检节点（基础版）
- [ ] 后处理服务（FFmpeg 拼接 + 字幕）
- [ ] 通知推送（WebSocket）
- [ ] 现有 82 个路由迁移到新架构

**不做的**：Kafka/Pulsar 切换、海外支付、AI 配音、模板市场、管理后台

### 8.2 第二阶段（差异化，4-6周）

- [ ] AI 配音（TTS）
- [ ] 风格模板系统
- [ ] 角色一致性管理
- [ ] 任务队列长期方案评估（Kafka vs Pulsar）
- [ ] 海外平台适配（SRT/VTT、多尺寸）

### 8.3 第三阶段（运营增强，4-6周）

- [ ] 管理后台
- [ ] 模板市场
- [ ] 数据分析面板
- [ ] Kafka/Pulsar 迁移（如需要）
- [ ] Stripe/PayPal 海外支付

---

## 九、测试策略

| 类型 | 范围 | 工具 |
|------|------|------|
| 单元测试 | 每个 Service / Adapter 的纯函数 | Jest + ts-jest |
| 集成测试 | 路由 + 数据库 + 中间件 | Supertest |
| AI 生成质量测试 | 剧本/分镜/视频输出的质检评分 | 自动化评分脚本 |
| 端到端测试 | 完整 Pipeline（从上传到下载）| Playwright |
| 负载测试 | 并发视频生成场景 | k6 |

---

*本文档为后端详细设计初稿，待开发评审后迭代*
---

## 十、原有业务迁移方案（详细）

### 10.1 迁移原则

| 原则 | 说明 |
|------|------|
| **功能零丢失** | 现有 82 个路由的所有业务逻辑必须保留 |
| **渐进迁移** | 先适配层（Adapter），再业务层，最后数据库层 |
| **可回滚** | 每个迁移步骤完成后可回滚到迁移前状态 |
| **隔离测试** | 迁移完成后先用测试账号验证，确认无误再开放 |

### 10.2 现有模块分析

| 现有模块 | 路由前缀 | 核心功能 | 迁移策略 |
|---------|---------|---------|---------|
| novel | `/novel/` | 小说上传/解析/章节管理 | **直接迁移** + 扩展 user_id |
| outline | `/outline/` | 大纲生成（Agent）| **封装为 Adapter** + 接入 ModelRouter |
| script | `/script/` | 剧本生成（Agent）| **封装为 Adapter** + 接入 ModelRouter |
| storyboard | `/storyboard/` | 分镜管理/图生提示词 | **封装为 Adapter** + 接入 ModelRouter |
| assets | `/assets/` | 素材生成/角色管理 | **封装为 Adapter** + 增加一致性管理 |
| video | `/video/` | 视频生成（多模型）| **封装为 Adapter** + 扩展新视频模型 |
| setting | `/setting/` | AI 模型配置 | **接入 ModelRouter**（核心）|
| project | `/project/` | 项目管理 | **迁移** + user_id + 多租户隔离 |
| prompt | `/prompt/` | 提示词模板 | **迁移** + 扩展为风格模板 |
| task | `/task/` | 任务管理 | **扩展** + 接入 TaskQueue |

### 10.3 路由迁移步骤

**Step 1：目录重组（不动代码）**
```
原目录：
  src/routes/novel/addNovel.ts
  src/routes/novel/getNovel.ts
  src/routes/outline/agentsOutline.ts

迁移后：
  src/modules/novel/routes/addNovel.ts
  src/modules/novel/routes/getNovel.ts
  src/modules/outline/routes/agentsOutline.ts

  src/modules/novel/adapters/NovelAdapter.ts        ← 新增：适配 ModelRouter
  src/modules/outline/adapters/OutlineAdapter.ts   ← 新增：封装现有 Agent
  src/modules/script/adapters/ScriptAdapter.ts     ← 新增：封装现有 Agent
```

**Step 2：路由层面加中间件（不改 Handler）**
- 所有现有路由在 `app.ts` 注册时外层包裹 `AuthMiddleware`（从 admin 账号改为 JWT 认证）
- 所有消耗性路由（outline/script/storyboard/video）包裹 `CreditMiddleware`
- **预期**：现有路由在加这两层中间件后，无需改动 Handler 代码即可支持多用户

**Step 3：封装为 Adapter（扩展 ModelRouter）**
```typescript
// src/modules/outline/adapters/OutlineAdapter.ts

export class OutlineAdapter implements IModelAdapter {
  readonly name = 'toonflow-outline';
  readonly provider = 'toonflow';
  readonly capabilities = ['outline-generation'] as const;

  // 复用现有 Agent 逻辑，包装为标准接口
  async call(prompt: string, options?: CallOptions): Promise<ModelOutput> {
    // 调用现有的 agents/outlineScript/index.ts
    const agent = new OutlineScript(projectId);
    const result = await agent.generate(prompt);
    return {
      content: result,
      usage: { promptTokens: 0, completionTokens: 0, totalTokens: 0 }, // 估算
      latency: 0
    };
  }
}
```

**Step 4：数据层迁移（user_id 关联）**
- 所有现有表（t_project / t_novel / t_outline 等）通过 `user_id` 字段关联到 `users` 表
- 迁移脚本：创建 `user_id` 列，存量数据暂时设为 `1`（对应 admin 用户），新数据从认证获取

### 10.4 迁移检查清单

每个模块迁移完成后验证：

- [ ] 路由响应格式不变（与迁移前一致）
- [ ] 数据库写入正常（数据可查询）
- [ ] JWT 认证生效（非 admin 账号可正常访问）
- [ ] 积分扣减生效（消耗性操作正确扣积分）
- [ ] WebSocket 推送正常（进度通知）
- [ ] 现有前端（Toonflow-web）调用不受影响

---

## 十一、数据库改造详细方案

### 11.1 现有表改造（Migration）

**原则**：现有数据不丢失，通过 ALTER TABLE 扩展，不DROP/DELETE任何列。

#### t_project（项目管理）

```sql
-- 新增字段
ALTER TABLE t_project ADD COLUMN user_id INTEGER NOT NULL DEFAULT 1;
ALTER TABLE t_project ADD COLUMN plan VARCHAR(20) NOT NULL DEFAULT 'free';
ALTER TABLE t_project ADD COLUMN settings JSONB;
ALTER TABLE t_project ADD COLUMN created_by INTEGER REFERENCES users(id);
ALTER TABLE t_project ADD COLUMN updated_by INTEGER REFERENCES users(id);
ALTER TABLE t_project ADD COLUMN deleted_at TIMESTAMP;  -- 软删除

-- 创建索引（支持多租户查询）
CREATE INDEX idx_project_user_id ON t_project(user_id);

-- 迁移数据：存量项目关联到 admin 用户（id=1）
UPDATE t_project SET user_id = 1 WHERE user_id IS NULL;
```

#### t_novel（小说管理）

```sql
ALTER TABLE t_novel ADD COLUMN user_id INTEGER NOT NULL DEFAULT 1;
ALTER TABLE t_novel ADD COLUMN title VARCHAR(500);
ALTER TABLE t_novel ADD COLUMN word_count INTEGER;
ALTER TABLE t_novel ADD COLUMN status VARCHAR(20) DEFAULT 'active';  -- active/draft/archived
ALTER TABLE t_novel ADD COLUMN created_by INTEGER REFERENCES users(id);
ALTER TABLE t_novel ADD COLUMN deleted_at TIMESTAMP;

CREATE INDEX idx_novel_user_id ON t_novel(user_id);
UPDATE t_novel SET user_id = 1 WHERE user_id IS NULL;
```

#### t_outline / t_script / t_storyboard / t_assets / t_video

```sql
-- 统一操作（每个表）
ALTER TABLE {TABLE} ADD COLUMN user_id INTEGER NOT NULL DEFAULT 1;
ALTER TABLE {TABLE} ADD COLUMN created_by INTEGER REFERENCES users(id);
ALTER TABLE {TABLE} ADD COLUMN deleted_at TIMESTAMP;
CREATE INDEX idx_{TABLE}_user_id ON {TABLE}(user_id);
UPDATE {TABLE} SET user_id = 1 WHERE user_id IS NULL;
```

#### t_config（AI 模型配置）

```sql
-- 扩展字段支持多厂商
ALTER TABLE t_config ADD COLUMN provider VARCHAR(50);  -- 'google', 'bytedance', 'seedance', 'openrouter'...
ALTER TABLE t_config ADD COLUMN capabilities JSONB;   -- ['script-generation', 'character-analysis']
ALTER TABLE t_config ADD COLUMN cost_per_call DECIMAL(10,2) DEFAULT 0;
ALTER TABLE t_config ADD COLUMN priority INTEGER DEFAULT 100;  -- 优先级，数字越小越优先
ALTER TABLE t_config ADD COLUMN is_enabled BOOLEAN DEFAULT true;

-- 保留现有 config 字段（JSON 格式兼容新字段）
-- 迁移时：provider = 'google'（根据 model 字段推断），capabilities 从 config.JSONB 解析
```

### 11.2 新增表（Create）

详见 6.1 节，核心新增 8 张表：
- users（用户）
- plans（套餐）
- subscriptions（订阅）
- credits（积分）
- credit_flow（积分流水）
- tasks（任务）
- payments（支付）
- works（作品）

### 11.3 迁移执行计划

```
Phase 1（数据准备）：
  1. 创建 t_users 表，插入 admin 用户（id=1，作为存量数据默认用户）
  2. 创建 plans 表，插入 4 个初始套餐
  3. 批量 ALTER 所有现有表（加 user_id + created_by + deleted_at）

Phase 2（功能适配）：
  4. 所有现有表存量数据 user_id 设为 1（admin）
  5. 修改 src/routes/ 所有 Handler，自动从 JWT 解密 userId，写入 user_id 字段

Phase 3（积分关联）：
  6. 新增 credits / credit_flow 表
  7. 给 admin 用户预置积分（测试用）
  8. CreditMiddleware 接入所有消耗性路由
```

---

## 十二、第一阶段任务详细说明

> 以下为 MVP 阶段（6-8 周）必须完成的全部任务，按优先级排序。

### 任务列表

| # | 任务名称 | 工作量 | 依赖 | 说明 |
|---|---------|-------|------|------|
| T1 | 数据库迁移 | 中 | 无 | SQLite → PostgreSQL，迁移脚本 |
| T2 | 用户认证模块 | 大 | T1 | 注册/登录/微信/JWT |
| T3 | 订阅积分系统 | 大 | T1+T2 | 套餐/积分/扣减/充值 |
| T4 | 中间件层 | 中 | T2 | Auth/Billing/RateLimit/Logger |
| T5 | AI 模型路由（扩展）| 中 | 无 | 接入豆包/Seedance/OpenRouter |
| T6 | Workflow 编排层 | 大 | T4+T5 | Pipeline 串联 + 状态机 |
| T7 | 任务队列（BullMQ）| 中 | T1 | 入队/进度/中断恢复 |
| T8 | 质检节点 | 中 | T5 | 剧本/分镜/图片/视频质检 |
| T9 | 后处理服务 | 大 | T7 | FFmpeg拼接+字幕+配音+BGM |
| T10 | 通知推送（WebSocket）| 中 | T7 | 实时进度 + 完成通知 |
| T11 | 原有路由迁移 | 大 | T1+T4 | 82个路由加中间件，user_id 关联 |
| T12 | 前端适配 | 中 | T2+T11 | 登录/订阅/工作台改造 |

### T9 详细说明：AI 配音（TTS）— 第一阶段任务

**为什么放在第一阶段**：配音对完播率影响极大，是成品质量的关键差异点，且火山引擎 TTS 接入简单，ROI 高。

**技术方案**：
```typescript
// src/shared/ai/adapters/ByteDanceTTSAdapter.ts

export class ByteDanceTTSAdapter implements IModelAdapter {
  readonly name = 'bytedance-tts';
  readonly provider = 'bytedance';
  readonly capabilities = ['tts'] as const;

  async call(prompt: string, options?: TTSOptions): Promise<TTSOutput> {
    // 调用火山引擎 TTS API
    // 输入：对白文本列表 [{ text, speaker }]
    // 输出：音频 Buffer 列表
  }
}

// 支持音色（按风格）：
// - female_warm（温暖女声）
// - male_warm（温暖男声）
// - female_bright（知性女声）
// - male_deep（低沉男声）
// - female_lively（活泼女声）
```

**接入位置**：PostProcessService（Step 4: AI 配音）

**计费**：每1000字 = 10 积分（成本约 ¥0.01/千字）

---

## 十三、可观测性 & DFX 设计

> DFX（Design for X）：包括可维护性、可排障性、可扩展性。

### 13.1 日志体系

**日志级别**：
```
DEBUG：  详细调试信息（开发环境自动开启，生产环境按需开启）
INFO：   正常业务流程（请求入口/出口、任务开始/完成）
WARN：   异常但不阻断（积分不足自动补、模型调用失败自动切换）
ERROR：  错误需关注（数据库写入失败、AI API 超时、质检连续失败）
FATAL：  系统不可用（服务崩溃、连接池耗尽）
```

**日志格式（JSON 结构化）**：
```json
{
  "timestamp": "2026-03-30T14:30:00.000Z",
  "level": "INFO",
  "requestId": "req_abc123",
  "userId": 42,
  "jobId": "job_xyz789",
  "module": "workflow",
  "stage": "video-generation",
  "message": "分镜 3/8 视频生成完成",
  "duration": 12500,
  "meta": {
    "model": "seedance-2.0",
    "fragmentIndex": 3,
    "totalFragments": 8
  }
}
```

**关键日志埋点（必须）**：
- 每个中间件入口/出口
- 每个 Workflow Stage 开始/结束
- AI 模型调用（请求 + 响应 + 耗时 + token 消耗）
- 积分扣减（扣了多少、剩余多少）
- 错误发生时的完整上下文

### 13.2 异常自动处理

#### 分层异常处理策略

```
┌─────────────────────────────────────────────────────────────┐
│                      异常分类与处理                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  L1: AI 模型调用异常                                        │
│  ├─ 超时（>30s）→ 自动切换备选模型 + 重试（最多3次）          │
│  ├─ 限流（429）→ 等待指数退避（1s, 2s, 4s）后重试            │
│  ├─ 额度不足 → 切换下一个模型，记录切换原因                  │
│  └─ 网络错误 → 记录 + 重试（3次），仍失败则标记任务失败        │
│                                                             │
│  L2: 质检未通过                                              │
│  ├─ 自动重写（最多3次，参考 QualityService.autoRewrite）    │
│  └─ 3次仍失败 → 暂停任务，通知用户手动确认                  │
│                                                             │
│  L3: 任务执行异常                                            │
│  ├─ 未捕获异常 → 写入 t_tasks.error + 推送 task:failed       │
│  ├─ 任务中断（服务重启）→ BullMQ 持久化 job，重启后自动恢复   │
│  └─ 重复任务检测（同一 userId + projectId 同时只能有1个任务） │
│                                                             │
│  L4: 系统资源异常                                            │
│  ├─ Redis 连接失败 → 降级为内存队列（单实例有限支持）         │
│  ├─ PostgreSQL 连接失败 → 服务不可用，返回 503               │
│  └─ COS 上传失败 → 重试3次，仍失败则任务标记失败并通知用户   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 重试策略配置

```typescript
// src/shared/infrastructure/retry.ts

export const retryConfig = {
  aiModelCall: {
    maxAttempts: 3,
    backoffMs: [1000, 2000, 4000],  // 指数退避
    retryableErrors: ['ETIMEDOUT', 'ECONNRESET', '429', '503']
  },
  databaseWrite: {
    maxAttempts: 2,
    backoffMs: [100, 500],
    retryableErrors: ['ECONNREFUSED', 'lock_timeout']
  },
  storageUpload: {
    maxAttempts: 3,
    backoffMs: [500, 1000, 2000],
    retryableErrors: ['NetworkingError', 'RequestTimeout']
  },
  taskQueue: {
    maxAttempts: 3,
    backoffMs: [2000, 5000, 10000],
    retryableErrors: ['Worker crashed', 'Job expired']
  }
};
```

### 13.3 分布式追踪

**Request ID 全链路传递**：
```typescript
// 每个请求生成唯一 requestId
const requestId = req.headers['x-request-id'] || uuid();

// 挂载到 req，后续所有日志和子操作都携带
req.requestId = requestId;

// 中间件中自动传播
ctx.requestId = requestId;

// AI 调用时注入到 meta
const modelCall = await modelRouter.call(task, prompt, {
  requestId,
  userId: req.user.id,
  jobId
});
```

**Span 结构**（与 OpenTelemetry 兼容）：
```
Request: /workflow/start
  └─ Stage: video-generation [2.3s]
       ├─ Model: seedance-2.0 [1.8s]
       │   └─ HTTP: POST api.seedance.io [1.7s]
       ├─ Quality: image-check [0.3s]
       └─ Storage: upload [0.2s]
  └─ Stage: postprocess [1.1s]
       ├─ FFmpeg: concat [0.5s]
       ├─ TTS: generate [0.4s]
       └─ Storage: upload [0.2s]
```

### 13.4 监控指标（SLO）

| 指标 | 目标 | 告警阈值 |
|------|------|---------|
| API 响应时间 P99 | < 2s | > 5s |
| Pipeline 成功率 | > 80% | < 70% |
| AI 模型调用成功率 | > 95% | < 90% |
| 任务完成平均时长 | < 15min（不含排队）| > 20min |
| 系统可用性 | > 99.5% | < 99% |

### 13.5 告警规则

```yaml
# alert-rules.yml（接入 Prometheus AlertManager）

alerts:
  - name: high_error_rate
    condition: rate(errors_total[5m]) > 0.05
    severity: critical
    message: "5分钟内错误率超过 5%，请立即检查"

  - name: pipeline_success_rate_low
    condition: pipeline_success_rate < 0.70
    severity: warning
    message: "Pipeline 成功率低于 70%，AI 生成质量可能有问题"

  - name: task_queue_backlog
    condition: queue_size > 100
    severity: warning
    message: "任务队列积压超过 100 个，请检查 AI 服务商状态"

  - name: credits_depleted
    condition: credits_balance == 0
    severity: info
    message: "用户积分耗尽"

  - name: ai_model_degraded
    condition: model_success_rate{provider="seedance"} < 0.90
    severity: warning
    message: "Seedance 模型成功率低于 90%，建议切换备选"
```

### 13.6 故障排查指南

**常见问题快速定位**：

| 症状 | 检查路径 |
|------|---------|
| Pipeline 卡住不动 | t_tasks 表看 stage + error → 查看日志 requestId |
| AI 生成失败 | 搜索 requestId + "Model call failed" → 定位具体模型 |
| 积分扣减不对 | credit_flow 表查 userId + 时间范围 → 对照任务记录 |
| 视频无声音 | 检查 PostProcessService Step 4（TTS）日志 |
| WebSocket 不推送 | 检查用户在线状态 + Redis pub/sub 连接 |
| 数据库写入慢 | 检查索引 idx_user_id 是否存在 + 查询计划 |

**日志查询示例**（假设用 ELK）：
```
# 查某个任务的所有日志
requestId:req_abc123

# 查某个用户的所有操作
userId:42 AND _timestamp:[2026-03-30T14:00 TO 2026-03-30T15:00]

# 查 AI 模型调用失败
module:ai AND level:ERROR AND message:"Model call failed"

# 查积分异常
module:billing AND level:WARN
```

### 13.7 健康检查接口

```typescript
// GET /health

{
  "status": "ok",  // 'ok' | 'degraded' | 'down'
  "version": "1.0.0",
  "uptime": 86400,
  "checks": {
    "database": { "status": "ok", "latency": 5 },
    "redis": { "status": "ok", "latency": 2 },
    "storage": { "status": "ok", "latency": 50 },
    "queue": { "status": "ok", "size": 12 },
    "ai_models": {
      "google-gemini": { "status": "ok", "latency": 200 },
      "seedance-2.0": { "status": "degraded", "latency": 8000 }
    }
  }
}

// 当任一 check 失败时，status 变为 'degraded'
// 当 database 或 queue 失败时，status 变为 'down'（K8s 会重启 Pod）
```

---

*本文档持续更新中*
