# 剧灵 StoryAi - 现有代码仓库分析

创建时间：2026-03-30 | 分析人：小咕嘟

---

## 一、现有仓库总览

### 代码规模
- 总文件数：188 个（含 TS/HTML/CSS/配置等）
- TypeScript 文件：155 个
- 核心路由：82 个（已注册到 Express）
- 前端资源：Vue 3（独立仓库 Toonflow-web）

### 技术栈（已确定）
| 模块 | 技术 |
|------|------|
| 后端框架 | Express.js + TypeScript |
| AI SDK | `ai` (Vercel SDK) — 已支持多模型 |
| 数据库 | SQLite（better-sqlite3）|
| 前端 | Vue 3 + TypeScript + Vite（独立仓库）|
| 实时通信 | express-ws（WebSocket）|
| 构建 | Electron（本地客户端）+ Docker |

---

## 二、已有优势（能直接复用的）

### 1. AI 模型路由层 ✅ 强
已接入多模型商：Google (Gemini)、Anthropic (Claude)、OpenAI、DeepSeek、XAI
- 文件：`src/routes/setting/`（配置管理）、`src/types/database.d.ts`
- 已有 `t_config` 表，支持配置切换不同模型
- **复用价值**：直接沿用，增加 Seedance/豆包/聚合平台适配即可

### 2. 工作流 Pipeline ✅ 强
已有完整链路：小说 → 大纲 → 剧本 → 分镜 → 素材生成 → 视频生成

关键模块：
- `src/routes/novel/` — 小说管理（上传/解析/章节拆分）
- `src/routes/outline/` — 大纲生成（agentsOutline，调用 LLM 生成结构化大纲）
- `src/routes/script/` — 剧本生成（generateScriptApi，调用 LLM 输出剧本）
- `src/routes/storyboard/` — 分镜管理（chatStoryboard，AI 对话式分镜生成）
- `src/routes/assets/` — 素材管理（角色、道具、场景图片生成）
- `src/routes/video/` — 视频生成（generateVideo，支持多模型视频生成）

**复用价值**：核心流程已跑通，只需增强和扩展，不需要重写

### 3. Agent 逻辑 ✅ 强
`src/agents/` 已实现：
- `outlineScript/index.ts` — 大纲脚本 Agent，定义了完整的 Schema（EpisodeData，包含场次、角色、道具、情绪曲线等）
- `storyboard/index.ts` — 分镜 Agent，包含图生提示词、图片生成、图像分割
- 已有 EventEmitter 架构，支持流式输出和进度推送

**复用价值**：Agent 设计理念和 Schema 定义非常成熟，可直接扩展

### 4. 数据库设计 ✅ 中
- `src/types/database.d.ts` 定义了完整的表结构
- 已有表：t_project、t_novel、t_outline、t_script、t_storyboard、t_assets、t_video、t_videoConfig、t_config
- **复用价值**：表结构基本满足业务，扩展字段即可

### 5. 前端 Toonflow-web ✅ 中
独立仓库，已实现：
- 项目管理、原始文本编辑、角色素材库、大纲管理、剧本编辑器、分镜设计器、视频配置、任务监控
- **复用价值**：UI 框架和组件可直接复用，改造成本低

---

## 三、需要改动的部分（查缺补漏）

### 优先级 P0（必须重写/新增）

**1. 用户系统 → 多租户 SaaS**
- 现状：单机 admin 账号，无用户体系
- 改动：新增 user/auth 表 + 订阅/积分表 + 认证中间件
- 工作量：小（新增模块，不改现有逻辑）

**2. 订阅与积分系统**
- 现状：无计费系统
- 改动：新增 t_subscription、t_credit_flow 表 + 计费中间件
- 工作量：小（新增模块）

**3. 工作流编排层（串联已有模块）**
- 现状：用户手动一个个调用 API，步骤割裂
- 改动：在 `src/routes/workflow/` 新增自动编排层，一键触发整个 pipeline
- 工作量：中（新增 orchestration 层，调用现有 82 个路由）

**4. AI 模型路由扩展**
- 现状：已支持 Google/Anthropic/OpenAI/DeepSeek/XAI
- 改动：新增 Seedance 2.0、豆包视频模型、聚合平台（OpenRouter/Together AI）
- 工作量：小（扩展现有 router 层）

**5. 质检节点**
- 现状：无质检机制，AI 输出直接通过
- 改动：新增 `src/services/quality/` 服务，每个环节自动评分 + 不合格重写
- 工作量：中（新增质检模块）

### 优先级 P1（需要扩展）

**6. 任务队列（异步 + 可靠性）**
- 现状：同步调用，生成时间很长时前端会超时
- 改动：引入 BullMQ（Redis）+ 任务持久化 + WebSocket 推送
- 工作量：中（需要改写 video 等耗时路由）

**7. 后处理流水线（FFmpeg + AI 音频）**
- 现状：视频生成后只有文件存储，无剪辑加工
- 改动：新增 `src/services/postprocess/` — 字幕生成 + 音频合成 + BGM
- 工作量：中（新增服务模块）

**8. 海外平台适配**
- 现状：无 SRT/VTT 导出、无多尺寸输出
- 改动：在 video 生成后加参数（resolution/format）控制 + 字幕格式切换
- 工作量：小（在现有流程上扩展参数）

### 优先级 P2（长期优化）

**9. 角色一致性系统**
- 现状：素材生成各自独立，角色外观不连贯
- 改动：在 assets 模块中增加角色 ID 引用 + reference_image 复用逻辑
- 工作量：中（修改 assets 生成逻辑）

**10. 风格模板系统**
- 现状：ArtStyle 已有 artStyle.ts（艺术风格定义），但无模板化
- 改动：将现有 artStyle 扩展为可配置的模板系统
- 工作量：小（在现有代码上扩展）

---

## 四、重构原则（不重写）

| 原则 | 说明 |
|------|------|
| **保留 82 个路由** | 现有 API 全部保留，只是扩展和组合 |
| **保留 Agent 逻辑** | `src/agents/` 不重写，只扩展新的 Agent |
| **保留数据库表** | 在现有表上扩展字段，不新建大表 |
| **保留前端 Toonflow-web** | 直接复用，只改接口层 |
| **保留 AI SDK** | 继续用 `ai` 包，扩展新的模型适配器 |
| **新增，不修改** | 所有 SaaS 功能作为新模块叠加，老模块零改动 |

---

## 五、建议的架构演进路径

**Phase 1（复用为主）**：在现有代码上叠加 SaaS 基础层
- 新增：用户系统、订阅积分、认证中间件
- 扩展：AI 模型列表（Seedance 2.0 + 豆包 + OpenRouter）
- 新增：工作流编排层（调用现有路由，全自动 pipeline）

**Phase 2（扩展为主）**：增强核心能力
- 新增：质检节点、任务队列（BullMQ）、WebSocket 进度推送
- 扩展：FFmpeg 后处理（字幕 + 音频 + BGM）
- 扩展：海外平台适配（SRT/VTT + 多尺寸）

**Phase 3（完善为主）**：运营功能
- 新增：风格模板系统、角色一致性管理、模板市场
- 新增：数据分析面板、自定义模型接口