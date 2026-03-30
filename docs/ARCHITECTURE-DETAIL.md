# 剧灵 StoryAi - 技术架构图 & 中间件总览

---

## 一、整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户层                                          │
│  Web (浏览器)  │  H5 (移动端)  │  小程序  │  API (企业客户)                    │
└──────────────┬──────────────┬──────────┬───────────┬────────────────────────┘
               │              │          │           │
           HTTPS          HTTPS     HTTPS      REST API
               │              │          │           │
               ▼              ▼          ▼           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Nginx (负载均衡)                                  │
│                         限流 + 日志 + HTTPS                                   │
└──────────────┬─────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        API 网关层 (Gateway)                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  鉴权中间件  │  │  计费中间件   │  │  限流中间件   │  │  日志中间件   │   │
│  │ AuthMid     │  │ BillingMid   │  │  RateLimit   │  │  Logger      │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
└──────────────┬─────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          业务层 (Express)                                     │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                    编排层 (Workflow Orchestrator)                       │ │
│  │    一个API触发 → 小说解析 → 大纲生成 → 剧本 → 分镜 → 视频 → 后处理       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │  用户服务   │ │  项目服务   │ │  订阅服务   │ │  积分服务   │           │
│  │ UserService │ │ProjectServ  │ │SubService   │ │ CreditServ  │           │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   模型路由   │  │   质检节点   │  │  任务队列    │  │  后处理服务  │   │
│  │ModelRouter   │  │QualityCheck  │  │TaskQueue    │  │PostProcess   │   │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      现有 82 个路由（零改动）                              │ │
│  │  novel/ │ outline/ │ script/ │ storyboard/ │ assets/ │ video/ │ setting/... │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└──────────────┬─────────────────────────────────────────────────────────────┘
               │
    ┌──────────┼──────────┬────────────┐
    ▼          ▼          ▼            ▼
 ┌──────┐  ┌────────┐  ┌────────┐  ┌────────┐
 │Redis │  │SQLite  │  │  OSS   │  │ AI模型  │
 │BullMQ│  │(现有)  │  │(文件)  │  │  Router │
 │任务队│  └────────┘  └────────┘  └────────┘
 │列+缓存│
 └──────┘
```

---

## 二、中间件层次详解

### 第一层：入口中间件（Gateway）

```
┌─────────────────────────────────────────────────────────────┐
│                    Middleware Layer 1: 入口层               │
│                    （所有请求必经）                          │
└─────────────────────────────────────────────────────────────┘
```

**1. AuthMiddleware（鉴权中间件）**
```
职责：验证用户身份，多租户隔离

检查顺序：
  1. 检查 Authorization Header（Bearer Token）
  2. Token 验证 → 查询 t_user 表 → 返回 user_id + plan
  3. 未登录 → 判断路由是否需要鉴权（公开路由放行）
  4. 挂载 req.user = { userId, plan, credit }

影响范围：所有 /api/* 路由（/other/login 除外）
```

**2. BillingMiddleware（计费中间件）**
```
职责：扣减积分，校验额度

检查顺序：
  1. 读取 req.user.plan 判断是否付费用户
  2. 根据路由类型计算本次消耗积分（如：生成视频 -30 积分）
  3. 查询 t_credit_flow 剩余积分
  4. 积分不足 → 返回 402 Payment Required
  5. 积分充足 → 写入 t_credit_flow（本次扣减记录）

影响范围：workflow/* / video/* / script/* 等消耗性路由
```

**3. RateLimitMiddleware（限流中间件）**
```
职责：防止单用户过度消耗 API 配额

策略：
  - 免费用户：60 req/min
  - 付费用户：300 req/min
  - 视频生成：10 req/hour（单独限制）

实现：Redis 计数滑动窗口算法
```

**4. LoggerMiddleware（日志中间件）**
```
职责：记录所有请求日志 + 审计

记录内容：
  - user_id / ip / method / path / duration / status
  - AI 调用次数和成本
  - 积分消耗明细

用于：成本分析、用户行为审计、问题排查
```

---

### 第二层：路由级中间件（Per-Route）

**5. ValidateMiddleware（参数校验）**
```
职责：Zod Schema 校验请求参数

已在 82 个路由中使用，扩展到 SaaS 新路由：

示例 - workflow/start:
{
  projectId: z.number(),
  styleId: z.string().optional(),
  resolution: z.enum(['720p', '1080p', '4k'])
}
```

**6. ProjectMiddleware（项目隔离）**
```
职责：校验用户是否有权访问该项目

检查：t_project.user_id === req.user.userId
不存在 → 403 Forbidden
```

**7. CreditMiddleware（积分预扣）**
```
职责：在任务开始前预扣积分（防止任务完成后账户为空）

仅用于：视频生成、批量任务等高消耗操作

流程：
  1. 估算本次消耗积分
  2. 预扣（锁定）积分
  3. 任务完成 → 实际扣减
  4. 任务失败 → 返还预扣积分
```

---

### 第三层：业务层服务（Services）

```
┌─────────────────────────────────────────────────────────────┐
│                   Middleware Layer 3: Services 层           │
│                 （注入到 Route Handler 中使用）             │
└─────────────────────────────────────────────────────────────┘
```

**8. ModelRouter（模型路由服务）**
```
职责：统一管理所有 AI 模型调用

现有模型（已接入）：
  - LLM: Gemini / Claude / GPT-4 / DeepSeek / XAI
  - 视频: Sora（OpenAI）

新增模型（规划）：
  - LLM: 豆包（字节）
  - 视频: Seedance 2.0 / 豆包视频 / 可灵
  - 聚合: OpenRouter / Together.ai
  - 图片: Seedream / Nano Banana Pro

接口：
  callModel(modelId: string, prompt: string, options): Promise<ModelOutput>
  getModelStatus(modelId: string): Promise<ModelHealth>

配置：/routes/setting/getAiModelMap（管理后台）
```

**9. QualityCheck（质检节点）**
```
职责：每个环节 AI 输出自动评分，不合格触发重写

质检矩阵：
  - 剧本 → AI 自检（剧情逻辑 + 角色一致性 + 情绪标签）
  - 分镜 → 规则校验（必填字段 + 与剧本一致性）
  - 图片 → AI 评分（0-100，<70 重新生成）
  - 视频 → 时长检测 + 画面分析
  - 成片 → 综合评分（<60 提示重做）

重试策略：每个环节最多自动重写 3 次，仍失败通知用户
```

**10. TaskQueue（任务队列服务）**
```
职责：异步任务执行 + 进度推送 + 中断恢复

使用：BullMQ + Redis

队列设计：
  - queue:video 生成队列（并发控制：3）
  - queue:postprocess 后处理队列（并发控制：2）
  - queue:high-priority 高优先级（实时交互任务）

Job 数据结构：
{
  id: string
  userId: number
  type: 'video' | 'postprocess' | 'batch'
  projectId: number
  status: 'pending' | 'processing' | 'completed' | 'failed'
  progress: number  // 0-100
  attempts: number  // 重试次数
  result?: { outputUrl: string }
  createdAt: Date
}

进度推送：WebSocket → 实时推送 progress 更新
中断恢复：Job 数据持久化到 SQLite，重启后自动读取并继续
```

**11. PostProcess（后处理服务）**
```
职责：FFmpeg 视频加工 + AI 音频 + 字幕

步骤：
  1. 片段拼接（FFmpeg concat）
  2. 字幕生成（ASR 语音识别 → 时间轴对齐 → SRT 压制）
  3. 音频处理（AI 音色生成 + BGM 混音 + 音效）
  4. 输出 MP4（可选分辨率：720P/1080P/4K）

调用方式：作为独立 Job 入队，由 TaskQueue 调度
```

---

## 三、数据流全景图

```
用户上传小说
    ↓
[Auth] 验证身份
    ↓
[Workflow] 编排层接收请求
    ↓
┌─────────────────────────────────────────────────────────────┐
│                  Pipeline 执行流程                            │
│                                                              │
│  ① 小说解析 → ② 大纲生成(Gemini) → ③ 剧本生成              │
│              ↓                        ↓                      │
│         [QualityCheck]          [QualityCheck]              │
│              ↓                        ↓                      │
│  ④ 角色管理 → ⑤ 分镜制作 → ⑥ 图片生成                     │
│              ↓                        ↓                      │
│         [QualityCheck]          [QualityCheck]              │
│              ↓                        ↓                      │
│  ⑦ 视频生成(Seedance) → ⑧ 后处理(FFmpeg)                   │
│              ↓                        ↓                      │
│         [QualityCheck]          [QualityCheck]              │
│              ↓                        ↓                      │
│         输出 MP4  ─────────────────────────────────→  作品管理
│                                                              │
└─────────────────────────────────────────────────────────────┘
    ↓
[Billing] 实际扣减积分
    ↓
[作品记录] 保存到数据库 + OSS
    ↓
[通知] WebSocket → 推送完成通知给用户
```

---

## 四、中间件清单总表

| # | 中间件名称 | 类型 | 作用位置 | 影响范围 |
|---|-----------|------|---------|---------|
| M1 | AuthMiddleware | 入口 | Gateway | 所有需鉴权路由 |
| M2 | BillingMiddleware | 入口 | Gateway | 所有消耗性路由 |
| M3 | RateLimitMiddleware | 入口 | Gateway | 所有 API 路由 |
| M4 | LoggerMiddleware | 入口 | Gateway | 所有 API 路由 |
| M5 | ValidateMiddleware | 路由级 | 每个路由 | 所有请求 |
| M6 | ProjectMiddleware | 路由级 | 项目路由 | project/* |
| M7 | CreditMiddleware | 路由级 | 消耗路由 | video/* / workflow/* |
| S1 | ModelRouter | 服务 | 业务层 | 所有 AI 调用 |
| S2 | QualityCheck | 服务 | 业务层 | pipeline 每环节 |
| S3 | TaskQueue | 服务 | 业务层 | 视频/后处理 |
| S4 | PostProcess | 服务 | 业务层 | 后处理队列 |
| S5 | WebSocketManager | 服务 | 业务层 | 实时推送 |

---

## 五、技术选型说明

| 组件 | 选型 | 理由 |
|------|------|------|
| API 网关 | 复用现有 Express，加 Nginx 层 | 无需引入 Kong |
| 数据库 | SQLite（现有）→ PostgreSQL（多租户扩展） | 增量迁移，先保现有稳定 |
| 缓存 + 队列 | Redis + BullMQ | 最成熟，Node 生态首选 |
| 对象存储 | 阿里云 OSS | 成本低，有国内节点 |
| WebSocket | 复用 express-ws（现有）| 无需引入 Socket.io |
| AI SDK | 复用 `ai` 包（现有）| 扩展新模型适配器即可 |
| FFmpeg | 部署在 Docker 容器或云函数 | 不占用主服务资源 |

---

*本文档用于架构决策参考，待确认后进入详细设计阶段*