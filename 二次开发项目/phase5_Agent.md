# Phase 5：Agent 层源码深度解析

> 分析对象：OpenClaw 六层架构中的 **L5 Agent 层**
> 分析目标：Agent 运行时框架、模型接入机制、工具/Skills/MCP 集成、记忆系统、会话管理、二次开发指南

---

## 一、Agent 层在整体架构中的位置

```
┌─────────────────────┐     ┌─────────────────────────────────┐
│   L4 调度层          │     │       L3 Channel Plugin 层      │
│  (auto-reply/cron)   │     │  (入站 MsgContext)              │
└────────┬────────────┘     └───────────────┬─────────────────┘
         │                                  │
         │  dispatchReplyFromConfig()       │  MsgContext (进程内)
         │                                  │
         ▼                                  ▼
┌══════════════════════════════════════════════════════════════════┐
║                     L5 Agent 层                                   ║
║                                                                  ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │              Harness 选择 (pi / codex / claude-cli)      │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                    │                                             ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │           runEmbeddedAttempt() — 核心执行循环             │   ║
║  │                                                          │   ║
║  │  ① 初始化   ② 工具创建   ③ System Prompt 构建            │   ║
║  │  ④ Context Engine  ⑤ Session 管理  ⑥ LLM 调用           │   ║
║  │  ⑦ Compaction      ⑧ 结果构建                           │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                    │                                             ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  子系统：Tools / Skills / MCP / Memory / Hook / Approval  │   ║
║  └──────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════╝
         │                                  │
         ▼                                  ▼
   L6 Provider 插件                     L6 Channel 插件
   (大模型 API 调用)                    (消息发送)
```

**核心定位**：Agent 层是智能体执行内核——接收标准化消息 → 加载会话与记忆 → 调用 LLM → 解析结果（文本+工具）→ 执行工具 → 流式回复 → 保存状态。Agent 层不直接调用任何 LLM 厂商 API，而是通过 L6 Provider 插件完成模型调用。

---

## 二、源码目录结构

```
src/agents/
├── harness/                              ← Harness 抽象层（注册/选择/执行）
│   ├── registry.ts                       ← 全局 Harness 注册中心
│   ├── types.ts                          ← AgentHarness 接口定义
│   ├── selection.ts                      ← Harness 选择算法 + 执行入口
│   ├── policy.ts                         ← Harness Policy 解析
│   ├── v2.ts                             ← V2 生命周期适配器
│   ├── builtin-pi.ts                     ← 内置 PI Harness 工厂
│   ├── runtime-plugin.ts                 ← 插件 Runtime 加载
│   └── result-classification.ts          ← 结果分类
├── pi-embedded-runner/                   ← PI Agent 核心实现 (~5000 行)
│   ├── run.ts                            ← runEmbeddedPiAgent() 顶层入口
│   ├── run/attempt.ts                    ← runEmbeddedAttempt() 核心执行循环
│   ├── run/types.ts                      ← 参数与结果类型
│   ├── run/attempt-system-prompt.ts      ← System Prompt 构建
│   ├── run/context-engine-maintenance.ts ← Context Engine 维护
│   ├── run/backend.ts                    ← 后端适配
│   ├── compact.ts                        ← 会话压缩入口
│   ├── compact.runtime.ts                ← 压缩运行时
│   └── runtime.ts                        ← EmbeddedAgentRuntime 类型
├── tools/                                ← 内建 Agent 工具
│   ├── common.ts                         ← AnyAgentTool 基础类型 + 参数读取器
│   ├── message-tool.ts                   ← 消息发送工具 (30+ actions)
│   ├── sessions-send-tool.ts             ← 会话发送工具
│   ├── sessions-spawn-tool.ts            ← 子 Agent 生成工具
│   ├── gateway-tool.ts                   ← Gateway 管理工具
│   ├── cron-tool.ts                      ← 定时任务工具
│   ├── image-generate-tool.ts            ← 图像生成工具
│   └── session-status-tool.ts            ← 会话状态工具
├── openclaw-tools.ts                     ← createOpenClawTools() 工具工厂 (529 行)
├── openclaw-tools.registration.ts        ← 工具注册辅助
├── tool-catalog.ts                       ← 工具目录 (33 个工具, 4 种 profile)
├── tool-description-presets.ts           ← 工具描述预设
├── tool-policy.ts                        ← 工具策略 (allow/deny)
├── prompt-surface.ts                     ← Prompt 表面类型
├── system-prompt.ts                      ← buildAgentSystemPrompt() (1381 行)
├── bootstrap-prompt.ts                   ← Bootstrap prompt 构建
├── bootstrap-files.ts                    ← Bootstrap 文件注入
├── model-fallback.ts                     ← runWithModelFallback() 模型回退
├── model-fallback.types.ts               ← 回退类型定义
├── model-fallback-observation.ts         ← 回退可观测性
├── model-selection-resolve.ts            ← 模型选择解析
├── provider-stream.ts                    ← registerProviderStreamForModel()
├── compaction.ts                         ← 上下文压缩 (summarizeWithFallback)
├── compaction-real-conversation.ts       ← 真实对话检测
├── pi-compaction-constants.ts            ← 压缩常量
├── pi-tools.before-tool-call.ts          ← 工具执行前门控 (916 行)
├── pi-bundle-mcp-runtime.ts              ← per-session MCP 运行时
├── mcp-transport.ts                      ← MCP 传输解析
├── mcp-stdio.ts                          ← MCP stdio 配置
├── bash-tools.exec-runtime.ts            ← Shell 执行运行时 (1015 行)
├── bash-tools.exec-types.ts              ← 执行类型定义
├── bash-tools.exec-approval-request.ts   ← 审批请求 (两阶段协议)
├── bash-tools.exec-approval-followup.ts  ← 审批后续
├── auth-profiles/                        ← 多 Key 轮换与冷却
│   ├── types.ts                          ← AuthProfileStore 类型
│   ├── order.ts                          ← 轮换排序算法
│   ├── usage.ts                          ← 冷却管理
│   └── store.ts                          ← 持久化存储
├── skills/                               ← Skills 系统
│   ├── types.ts                          ← SkillEntry / SkillSnapshot 类型
│   ├── workspace.ts                      ← buildWorkspaceSkillsPrompt()
│   ├── skill-contract.ts                 ← Skill 契约与 XML 格式化
│   ├── agent-filter.ts                   ← Agent 级 Skill 过滤器
│   ├── config.ts                         ← Skill 资格检查
│   ├── local-loader.ts                   ← 本地 Skill 加载
│   ├── plugin-skills.ts                  ← 插件 Skill 发现
│   └── frontmatter.ts                    ← Skill frontmatter 解析
├── memory-search.ts                      ← 记忆搜索配置解析
└── pi-embedded-subscribe.handlers.*.ts   ← 事件处理器 (tools/compaction)

src/context-engine/                       ← 上下文引擎
├── types.ts                              ← ContextEngine 接口 + 结果类型
├── registry.ts                           ← 引擎注册表 (可插拔)
├── legacy.ts                             ← LegacyContextEngine (默认)
├── delegate.ts                           ← 压缩委托给运行时
└── init.ts                               ← 初始化

src/memory/                               ← 记忆系统核心
├── root-memory-files.ts                  ← MEMORY.md 规范路径
└── manager-runtime.ts                    ← 记忆管理器运行时

src/config/sessions/                      ← Session 会话管理 (~35 个模块)
├── types.ts                              ← SessionEntry 类型 (~100 字段)
├── store.ts                              ← Session Store CRUD + 原子写
├── store-writer.ts                       ← 写锁保护 (WriterQueue 串行化)
├── store-cache.ts                        ← 三层缓存 (对象/快照/序列化)
├── store-load.ts                         ← 存储加载 + 规范化管道
├── store-entry.ts                        ← Session Entry 解析
├── store-maintenance.ts                  ← 维护清理 (TTL/上限/磁盘预算)
├── session-key.ts                        ← Session Key 派生
├── main-session.ts                       ← 主 Session Key 解析
├── group.ts                              ← 群组 Session Key
├── reset-policy.ts                       ← 重置策略 (daily/idle)
├── metadata.ts                           ← 元数据推导
├── transcript-append.ts                  ← JSONL Transcript 追加
├── transcript-stream.ts                  ← Transcript 流式读取
├── artifacts.ts                          ← Archive/Checkpoint 命名
├── combined-store-gateway.ts             ← 多 Agent Store 合并
└── paths.ts                              ← 路径解析

src/hooks/                                ← Hook 系统
├── types.ts                              ← Hook / HookEntry 类型
├── internal-hook-types.ts                ← InternalHookEvent 类型
├── internal-hooks.ts                     ← 内部 Hook 注册与触发
├── loader.ts                             ← Hook 加载管线
├── workspace.ts                          ← Hook 目录发现
├── config.ts                             ← Hook 资格检查
├── module-loader.ts                      ← Hook 模块动态导入
└── legacy-config.ts                      ← 旧版配置兼容

extensions/memory-core/                   ← 记忆引擎插件 (SQLite + FTS + Vector)
├── src/manager.ts                        ← MemoryIndexManager (主类)
├── src/embeddings.ts                     ← EmbeddingProvider 解析
├── src/dreaming.ts                       ← Dreaming 调度 (Light/Deep/REM)
├── src/dreaming-phases.ts                ← Dreaming 阶段执行
├── src/short-term-promotion.ts           ← 短期回忆提升算法

extensions/memory-lancedb/                ← LanceDB 记忆插件
├── src/index.ts                          ← 自动捕获 + 自动召回 (before_prompt_build / agent_end hooks)
└── src/db.ts                             ← MemoryDB 类
```

---

## 三、Harness 框架 — Agent 运行时抽象

### 3.1 设计目标

Harness 是 Agent 执行运行时的抽象层，允许 OpenClaw 在不同的大模型 CLI/API backend 之间切换。核心接口极简，仅定义 `supports()` 和 `runAttempt()` 两个核心方法。

### 3.2 AgentHarness 接口

> 源码：`src/agents/harness/types.ts:68-83`

```typescript
type AgentHarness = {
  id: string;                          // 唯一标识，如 "pi"、"codex"
  label: string;                       // 可读名称
  pluginId?: string;                   // 附属的插件 ID

  deliveryDefaults?: AgentHarnessDeliveryDefaults;  // 投递默认值

  // 能力声明 — 给定 provider/model，是否支持
  supports(ctx: {
    provider: string;
    model: string;
  }): AgentHarnessSupport;
  // 返回 { supported: true, priority?: number }
  // 或   { supported: false, reason?: string }

  // 核心执行方法
  runAttempt(params: AgentHarnessAttemptParams): Promise<AgentHarnessAttemptResult>;

  // 可选：边问题（side question）— 在 Agent turn 中插入额外 LLM 查询
  runSideQuestion?(params): Promise<AgentHarnessSideQuestionResult>;

  // 可选生命周期方法
  classify?(result, ctx): AgentHarnessResultClassification | undefined;  // 结果分类
  compact?(params): Promise<...>;      // 会话压缩
  reset?(params): void | Promise<void>;  // 重置
  dispose?(): void | Promise<void>;      // 销毁
};
```

### 3.3 Harness 类型对比

| 维度           | PI (pi-embedded)                    | Codex (codex)              | Claude CLI                     |
| -------------- | ----------------------------------- | -------------------------- | ------------------------------ |
| **ID**         | `"pi"`                              | `"codex"`                  | `"claude-cli"`                 |
| **来源**       | 内置 (`builtin-pi.ts`)              | 插件 (`extensions/codex/`) | 插件 (`extensions/anthropic/`) |
| **supports()** | 永远 `{supported:true, priority:0}` | 根据模型/provier 判断      | 根据模型/provier 判断          |
| **runAttempt** | `runEmbeddedAttempt()` (~5000 行)   | 通过 Codex CLI 子进程      | 通过 Claude CLI 子进程         |
| **compact**    | 内置                                | 实现                       | 实现                           |
| **reset**      | 内置 (无操作)                       | 实现                       | 实现                           |
| **classify**   | 无                                  | 实现                       | 实现                           |
| **压缩方式**   | summarizeWithFallback + 管道        | Harness 自有               | Harness 自有                   |
| **工具支持**   | 全部 33 个内建工具                  | 受 CLI 限制                | 受 CLI 限制                    |
| **适用场景**   | 默认选择，功能最全                  | OpenAI 模型优化            | Anthropic 模型优化             |
| **Hooks**      | 15 层 Stream 管道                   | CLI 自有                   | CLI 自有                       |

**PI Harness 工厂** (`builtin-pi.ts:4-11`)：

```typescript
function createPiAgentHarness(): AgentHarness {
  return {
    id: "pi",
    label: "PI embedded agent",
    supports: () => ({ supported: true, priority: 0 }),
    runAttempt: runEmbeddedAttempt, // 直接委托给核心执行函数
  };
}
```

PI harness 极简，无 classify/compact/reset/dispose 方法——这些能力直接集成在核心执行循环中。

### 3.4 Harness 选择算法

> 源码：`src/agents/harness/selection.ts:121-211`

```
selectAgentHarnessDecision({ provider, model, config })
  │
  ├─ 1. resolveConfiguredAgentHarnessPolicy() → 获取 runtime policy
  │      (从 agents.defaults.harness.runtime 或 model-level config)
  │
  ├─ 2. AgentHarnessRuntimeOverride 检查 (运行时覆盖)
  │
  ├─ 3. 内置 PI 候选 (永远在候选列表中)
  │
  ├─ 4. 根据 runtime type 路由：
  │
  │   ┌─ runtime === "pi"
  │   │   → 直接选择 PI，不检查插件
  │   │
  │   ├─ runtime === "codex" (或其他具体 ID)
  │   │   → 在插件注册表中查找该 ID
  │   │   → 找到 → 选择该插件
  │   │   → 找不到 → 抛出 MissingAgentHarnessError
  │   │
  │   └─ runtime === "auto"
  │       → 遍历所有已注册插件 harness
  │       → 调用 harness.supports({ provider, model })
  │       → 筛选 supported: true 的候选
  │       → 按 priority 降序排列
  │       → 最高优先级者胜出
  │       → 无匹配 → 回退到 PI
  │
  └─ 5. 返回 AgentHarnessSelectionDecision {
        harness, policy, selectedReason, candidates
      }
```

**Policy 解析** (`policy.ts:17-56`) 的特殊规则：

- OpenAI provider 在 `auto` 模式下默认路由到 `codex`
- 如果 codex harness 未注册，回退为 `pi`

### 3.5 PI Agent 执行入口

> 源码：`src/agents/pi-embedded-runner/run.ts:405`

```typescript
async function runEmbeddedPiAgent(params: RunEmbeddedPiAgentParams): Promise<EmbeddedPiRunResult>;
```

**执行流程**：

```
runEmbeddedPiAgent()
  │
  ├─ 1. Session Key 回溯 — 从 sessionId 反向解析 sessionKey
  │
  ├─ 2. Lane 队列调度 — 两级入队确保顺序执行
  │   ├─ resolveSessionLane()  → per-session lane
  │   ├─ resolveGlobalLane()   → 跨 session 并发控制
  │   │   (main 默认并发 4, subagent 默认并发 8)
  │   ├─ enqueueSession()      → session lane 内串行
  │   └─ enqueueGlobal()       → global lane 限额
  │
  └─ 3. runEmbeddedAttempt() → 核心执行循环 (详见下节)
```

**队列模式**（`src/auto-reply/reply/queue/types.ts`）：

| 模式        | 行为                              | 使用场景                 |
| ----------- | --------------------------------- | ------------------------ |
| `steer`     | 新消息替换队列中等待的消息        | 默认模式，适合 1:1 对话  |
| `followup`  | 新消息排队依次执行                | 需要保证每条消息都被处理 |
| `collect`   | 收集多条消息合并为一次 Agent 调用 | 减少 API 调用次数        |
| `interrupt` | 中断当前 run，立即执行新消息      | 会话重置后的首条消息     |

### 3.6 PI 核心执行循环

> 源码：`src/agents/pi-embedded-runner/run/attempt.ts` (~5000 行)

这是整个 Agent 系统最核心的函数——`runEmbeddedAttempt()`：

```
runEmbeddedAttempt()
  │
  ├─ 阶段 1: 初始化 (~1160 行)
  │   ├─ 创建工作目录 (fs.mkdir)
  │   ├─ 解析 sandbox 上下文
  │   ├─ 解析 skills 环境变量覆盖
  │   └─ 创建执行阶段计时器
  │
  ├─ 阶段 2: 工具初始化 (~1330-1820 行)
  │   ├─ mergeForcedEmbeddedAttemptToolsAllow() — 强制工具 allowlist
  │   ├─ resolveEmbeddedAttemptToolConstructionPlan() — 工具构建计划
  │   ├─ createOpenClawCodingTools() — 编码工具集 (exec/read/write/edit/web_search)
  │   ├─ 工具 allowlist 过滤
  │   ├─ 解析 bundle MCP 和 LSP 运行时
  │   └─ 应用 tool-search 和 code-mode 目录压缩
  │
  ├─ 阶段 3: System Prompt 构建 (~1970-2060 行)
  │   └─ buildAttemptSystemPrompt() → 参见第七节
  │
  ├─ 阶段 4: Context Engine & Session (~2080-2400 行)
  │   ├─ repairSessionFileIfNeeded() — 修复损坏的会话文件
  │   ├─ SessionManager.open() — 打开会话管理器
  │   ├─ runAttemptContextEngineBootstrap() — 引导上下文引擎
  │   ├─ createAgentSession() — 创建 agent session
  │   └─ 安装各种 hook (写锁/tool-only-terminal/...)
  │
  ├─ 阶段 5: Stream 函数管道 (~2610-2960 行)
  │   └─ 见下方 "Stream 管道" 详情
  │
  ├─ 阶段 6: Prompt 提交与执行 (~3500-4160 行)
  │   ├─ before_agent_run hook → 可能拦截/修改 prompt
  │   ├─ shouldPreemptivelyCompactBeforePrompt() → 抢占式压缩
  │   └─ promptActiveSession() → 提交 prompt、等待 LLM 流式响应
  │
  ├─ 阶段 7: Compaction 等待与后处理 (~4220-4480 行)
  │   ├─ waitForCompactionRetryWithAggregateTimeout() (60s 聚合超时)
  │   ├─ 更新缓存 TTL
  │   └─ finalizeAttemptContextEngineTurn()
  │
  └─ 阶段 8: 结果构建与资源清理 (~4800-4940 行)
      ├─ 构建 EmbeddedRunAttemptResult
      ├─ 运行 agent_end 和 llm_output hooks
      ├─ 清理工具搜索目录
      └─ 释放会话锁
```

**Stream 函数管道**——这是 PI Agent 最精妙的设计模式，通过函数组合将 15+ 个 wrapper 串联：

```
defaultSessionStreamFn                       ← 原始 stream 函数
  → resolveEmbeddedAgentStreamFn             ← API Key 注入 (从 auth-profiles)
  → wrapStreamFnTextTransforms               ← 文本转换 (格式修正)
  → createCodexNativeWebSearchWrapper        ← Code Mode 网络搜索
  → cacheTrace.wrapStreamFn                  ← 缓存追踪
  → dropThinkingBlocks / dropReasoningFromHistory  ← 推理块清理
  → sanitizeReplayToolCallIdsForStream       ← ToolCall ID 清理
  → downgradeOpenAIReasoningBlocks           ← OpenAI 兼容层
  → sessions_yield 检测                      ← 协作中断检测
  → wrapStreamFnSanitizeMalformedToolCalls   ← 畸形工具调用修复
  → wrapStreamFnTrimToolCallNames            ← 工具名修剪
  → wrapStreamFnRepairMalformedToolCallArguments ← 参数修复
  → wrapStreamFnDecodeXaiToolCallArguments   ← HTML 实体解码
  → anthropicPayloadLogger.wrapStreamFn      ← 日志记录
  → wrapStreamFnHandleSensitiveStopReason    ← 拒绝输出检测
  → streamWithIdleTimeout                    ← 空闲超时检测
  → wrapStreamFnWithDiagnosticModelCallEvents ← 诊断事件
```

每个 wrapper 是一个 `(StreamFn) => StreamFn` 的高阶函数，职责单一，可独立测试。

---

## 四、大模型接入方式

### 4.1 架构概览

OpenClaw 不直接调用任何 LLM 厂商 API。模型调用走三层抽象：

```
Agent 执行循环
  → runWithModelFallback()          ← 回退 + 冷却管理
    → registerProviderStreamForModel()  ← 模型字符串 → StreamFn 解析
      → Provider 插件 (extensions/<provider>/)  ← 厂商特定实现
        → HTTPS SSE 调用大模型 API
```

### 4.2 Provider 插件完整类型定义

> 源码：`src/plugins/types.ts:1222-1825` — `ProviderPlugin` 完整类型 (~50 个可选钩子)

```typescript
type ProviderPlugin<ResolvedAccount = unknown> = {
  // ━━━ 身份 (必填) ━━━
  id: string;                          // Provider 唯一标识
  label: string;                       // 可读名称
  pluginId?: string;                   // 所属插件 ID
  docsPath?: string;                   // 文档路径
  aliases?: string[];                  // 别名列表
  envVars?: string[];                  // 环境变量名列表

  // ━━━ 认证 (必填) ━━━
  auth: ProviderAuthMethod[];          // 至少一种认证方式
  // 支持 api-key / oauth / token / synthetic / custom

  // ━━━ 模型目录 ━━━
  catalog?: ProviderPluginCatalog;     // 模型目录构建器
  // catalog.order: "simple" | "grouped" | ...
  // catalog.run(ctx): 动态生成模型列表
  staticCatalog?: ModelDefinitionConfig[];
  augmentModelCatalog?: (ctx) => ModelCatalogEntry[];
  dynamicModelResolution?: ProviderDynamicModelResolution;

  // ━━━ 模型归一化 ━━━
  normalizeModelId?: (id, ctx) => string | undefined;
  normalizeConfig?: (cfg) => OpenClawConfig;
  normalizeResolvedModel?: (model) => ResolvedModel;
  normalizeTransport?: (transport) => ModelTransport;
  prepareDynamicModel?: (model, ctx) => ModelDefinitionConfig | undefined;

  // ━━━ 流式传输 (核心调用路径) ━━━
  createStreamFn?: (ctx) => StreamFn | undefined;
  wrapStreamFn?: (ctx) => StreamFn;    // 请求/响应包装 (最常用)
  resolveStreamFamily?: (ctx) => ProviderStreamFamily;

  // ━━━ 推理/思考控制 ━━━
  resolveThinkingProfile?: (ctx) => ThinkingProfile | undefined;
  resolveDefaultThinkingLevel?: (ctx) => string;
  isBinaryThinking?: (ctx) => boolean;
  isModernModelRef?: (ctx) => boolean;
  resolveFastModeToken?: (ctx) => string | undefined;

  // ━━━ 认证运行时 ━━━
  prepareRuntimeAuth?: (ctx) => ProviderPreparedRuntimeAuth;
  resolveUsageAuth?: (ctx) => ProviderUsageAuth;
  resolveSyntheticAuth?: (ctx) => ProviderAuthResult;
  refreshOAuth?: (ctx) => Promise<...>;
  resolveAuthProfileOrder?: (ctx) => string[];

  // ━━━ 工具 Schema 兼容 ━━━
  normalizeToolSchemas?: (schemas, ctx) => ToolSchema[];
  inspectToolSchemas?: (schemas, ctx) => ToolSchemaCompatibilityReport;
  resolveToolChoice?: (ctx) => ToolChoice | undefined;

  // ━━━ 错误处理 ━━━
  matchesContextOverflowError?: (err) => boolean;
  matchesRateLimitError?: (err) => boolean;
  classifyFailoverReason?: (err) => FailoverReason;
  resolveFailoverDecision?: (ctx) => FailoverDecision;

  // ━━━ 压缩/重放 ━━━
  buildReplayPolicy?: (ctx) => ReplayPolicy;
  sanitizeReplayHistory?: (msgs, ctx) => AgentMessage[];
  validateReplayTurns?: (turns, ctx) => ReplayValidationResult;
  buildReplayPrepFn?: (ctx) => ReplayPrepFn;

  // ━━━ 系统提示 ━━━
  resolveSystemPromptContribution?: (ctx) => string | undefined;
  transformSystemPrompt?: (prompt, ctx) => string;
  applySystemPromptDefaults?: (ctx) => void;

  // ━━━ 回退路由 ━━━
  followupFallbackRoute?: (ctx) => ModelCandidate | undefined;
  resolveImageFallbackRoute?: (ctx) => ModelCandidate | undefined;
};
```

**最常用的 Provider 钩子（按使用频率排序）**：

| 优先级 | 钩子                                          | 被调用时机                           |
| ------ | --------------------------------------------- | ------------------------------------ |
| P0     | `auth`                                        | 启动时解析 API Key                   |
| P0     | `catalog.buildProvider` / `catalog.run`       | 模型列表构建                         |
| P1     | `wrapStreamFn`                                | 每次 LLM 调用前包装请求/响应         |
| P1     | `resolveThinkingProfile`                      | 模型选择时获取推理级别               |
| P2     | `matchesContextOverflowError`                 | LLM 报错时判断是否上下文溢出         |
| P2     | `buildReplayPolicy` + `sanitizeReplayHistory` | Compaction 重放会话历史              |
| P3     | `normalizeToolSchemas`                        | 工具 Schema 传给 LLM 前做兼容转换    |
| P3     | `resolveSystemPromptContribution`             | System Prompt 组装时追加厂商特有指令 |

### 4.3 Provider 类型一览

| 接入方式                                            | 数量 | 示例                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defineSingleProviderPluginEntry` (OpenAI 兼容 API) | ~17  | deepseek, cerebras, deepinfra, fireworks, together, xai, qwen, moonshot, mistral, nvidia, venice, xiaomi, huggingface, kilocode, synthetic, vercel-ai-gateway                                                                                                                                                    |
| `definePluginEntry` + 自定义注册                    | ~33+ | openai, anthropic, anthropic-vertex, google, groq, ollama, vllm, amazon-bedrock, litellm, openrouter, lmstudio, sglang, minimax, zai, stepfun, tencent, volcengine, cloudflare-ai-gateway, github-copilot, microsoft-foundry, alibaba, arcee, byteplus, fal, chutes, copilot-proxy, kimi-coding, opencode, vydra |

**合计 50+ 文本推理 Provider**。此外还有图像生成（fal, openai, runway）、音乐生成、视频生成、TTS、Web 搜索等能力型 Provider。

### 4.4 模型调用的核心桥梁

> 源码：`src/agents/provider-stream.ts`

```typescript
function registerProviderStreamForModel(params: {
  model: Model;
  cfg?: OpenClawConfig;
  agentDir?: string;
  workspaceDir?: string;
  env?: NodeJS.ProcessEnv;
}): StreamFn | undefined;
```

执行流程：

```
"deepseek/deepseek-v4-pro" 输入
  │
  ├─ 1. resolveProviderStreamFn()
  │      → 检查 DeepSeek Provider 插件是否提供了自定义 createStreamFn
  │      → 若有，调用 createStreamFn(ctx) 生成 StreamFn
  │
  ├─ 2. 若无自定义 → createTransportAwareStreamFnForModel()
  │      → 基于标准 HTTP streamSimple 的通用传输
  │      → 解析 baseUrl + api + headers
  │
  └─ 3. ensureCustomApiRegistered(model.api, streamFn)
         → 注册到 pi-ai 库，使得 stream() 调用能路由到正确的传输实现
```

### 4.5 DeepSeek 作为参考实现

以 DeepSeek 为例展示一个 Provider 插件的完整结构：

```
extensions/deepseek/
  ├── index.ts              ← defineSingleProviderPluginEntry({...})
  ├── models.ts             ← 模型定义 + baseUrl
  ├── stream.ts             ← createDeepSeekV4ThinkingWrapper()
  ├── thinking.ts           ← V4_THINKING_PROFILE (7 个思考级别)
  ├── provider-catalog.ts   ← buildDeepSeekProvider()
  ├── onboard.ts            ← 初始化引导
  ├── api.ts                ← 公共 API 导出
  └── openclaw.plugin.json  ← 清单声明
```

**注册入口** (`index.ts`)：

```typescript
export default defineSingleProviderPluginEntry({
  id: "deepseek",
  name: "DeepSeek Provider",
  provider: {
    label: "DeepSeek",
    docsPath: "/providers/deepseek",
    auth: [
      {
        methodId: "api-key",
        label: "DeepSeek API key",
        envVar: "DEEPSEEK_API_KEY",
        defaultModel: "deepseek/deepseek-v4-flash",
      },
    ],
    catalog: { buildProvider: buildDeepSeekProvider },
    wrapStreamFn: (ctx) => createDeepSeekV4ThinkingWrapper(ctx.streamFn, ctx.thinkingLevel),
    resolveThinkingProfile: ({ modelId }) => resolveDeepSeekV4ThinkingProfile(modelId),
    matchesContextOverflowError: ({ errorMessage }) =>
      /\bdeepseek\b.*(?:input.*too long|context.*exceed)/i.test(errorMessage),
    ...buildProviderReplayFamilyHooks({ family: "openai-compatible" }),
    ...buildProviderToolCompatFamilyHooks("deepseek"),
  },
});
```

**思考级别** (`thinking.ts`)：DeepSeek V4 支持 7 个级别 —— `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`，默认 `high`。

### 4.6 模型回退机制

> 源码：`src/agents/model-fallback.ts:905`

```typescript
async function runWithModelFallback<T>(params: {
  primaryProvider: string;
  primaryModel: string;
  fallbacksOverride?: ModelCandidate[];
  run: (provider: string, model: string) => Promise<T>;
  // ...error classification callbacks
}): Promise<T>;
```

**执行流程**：

```
主模型调用
  │
  ├─ 成功 → 返回结果
  │
  └─ 失败
      │
      ├─ 1. resolveFallbackCandidates()
      │      → 从 agents.defaults.model.fallbacks 配置生成候选列表
      │      → 主模型 + 显式配置的回退模型
      │
      └─ 2. 遍历各候选 (结合 AuthProfile Store 冷却状态)
            │
            ├─ 跳过冷却中的 Profile (除非主模型探测期已过)
            ├─ runFallbackCandidate(provider, model)
            │   ├─ 成功 → classifyResult() 验证
            │   └─ 失败 → markAuthProfileFailure() → 下一个候选
            │
            └─ 全部失败 → FallbackSummaryError (含所有尝试详情)
```

### 4.7 Auth Profiles — 多 Key 轮换与冷却

> 源码：`src/agents/auth-profiles/order.ts:217`

支持为同一 Provider 配置多个 API Key，通过冷却机制实现自动故障转移：

```
resolveAuthProfileOrder(providerId)
  │
  ├─ 1. 清除已过期的冷却
  ├─ 2. 按配置顺序 + lastUsed 排序 (最早优先 = 轮询)
  ├─ 3. 按类型过滤: oauth > token > api_key
  ├─ 4. 冷却中的 Profile 排在末尾
  └─ 5. preferredProfile 放在最前面
```

**冷却退避策略**：

| 失败次数 | 冷却时间      |
| -------- | ------------- |
| 第 1 次  | 30 秒         |
| 第 2 次  | 1 分钟        |
| 第 3+ 次 | 5 分钟 (上限) |

- `rate_limit` 类型冷却仅影响触发限流的特定模型
- `billing` / `auth_permanent` 失败进入更长的禁用期 (5h-24h)
- 冷却期间允许探测（节流：每 30 秒每种 Provider/Agent 最多一次探测）

---

## 五、Tools / Skills / MCP 三层能力体系

### 5.1 体系总览

```
┌─────────────────────────────────────────────┐
│              暴露给 LLM 的 Tool Schema        │
│   (统一的 JSON Schema 格式)                  │
└────────────────────┬────────────────────────┘
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ 内建 Tools│  │  Skills  │  │   MCP    │
│  (33 个)  │  │(文件驱动) │  │(外部进程) │
└──────────┘  └──────────┘  └──────────┘
      │              │              │
      └──────────────┴──────────────┘
                     │
        createOpenClawTools() 统一工厂
```

### 5.2 内建 Tools 体系

> 源码：`src/agents/tool-catalog.ts`

33 个内建工具，分 11 个功能 Section：

| Section        | 工具                                                                                     | Profile           |
| -------------- | ---------------------------------------------------------------------------------------- | ----------------- |
| **fs**         | `read`, `write`, `edit`, `apply_patch`                                                   | coding, full      |
| **runtime**    | `exec`, `process`, `code_execution`                                                      | coding, full      |
| **web**        | `web_search`, `web_fetch`, `x_search`                                                    | coding, full      |
| **memory**     | `memory_search`, `memory_get`                                                            | coding, full      |
| **sessions**   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield` | coding, messaging |
| **messaging**  | `message` (30+ actions)                                                                  | messaging         |
| **media**      | `image`, `image_generate`, `video_generate`, `music_generate`                            | coding, full      |
| **agents**     | `subagents`, `session_status`, `update_plan`                                             | coding, minimal   |
| **nodes**      | `nodes.*`                                                                                | full              |
| **automation** | `cron` (8 actions)                                                                       | coding, full      |
| **ui**         | `gateway`                                                                                | full              |

**四种 Tool Profile**：

| Profile     | 包含                                       | 适用场景      |
| ----------- | ------------------------------------------ | ------------- |
| `minimal`   | 仅 `session_status`                        | 极简对话模式  |
| `coding`    | 文件 + 执行 + 搜索 + 记忆 + 会话 + 子Agent | 代码/任务模式 |
| `messaging` | 消息发送 + 会话查询                        | 纯消息转发    |
| `full`      | `["*"]` 全部                               | 完全能力      |

**工具工厂** (`src/agents/openclaw-tools.ts:529`)：

```typescript
function createOpenClawTools(options?: {
  sandboxRoot?: string;
  sandboxed?: boolean;
  config?: OpenClawConfig;
  pluginToolAllowlist?: string[];
  currentChannelId?: string;
  modelProvider?: string;
  modelId?: string;
  disableMessageTool?: boolean;
  enableHeartbeatTool?: boolean;
  // ... ~50 个上下文参数
}): AnyAgentTool[];
```

根据 model/Channel/sandbox 上下文动态决定启用/禁用哪些工具。例如：

- 非 vision 模型 → 跳过 `image` 工具
- `disableMessageTool: true` → 在嵌入模式下禁用消息发送
- 特定 Channel → 启用该 Channel 专属的 message tool actions

**核心工具类型** (`src/agents/tools/common.ts`)：

```typescript
type AnyAgentTool = {
  name: string;
  description: string;
  parameters: JsonSchema;
  execute(
    toolCallId: string,
    params: unknown,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback,
  ): Promise<AgentToolResult>;
  displaySummary?: string;
};
```

**参数读取器**（所有工具共享的基础设施）：

```typescript
readStringParam(params, key, options?)    → string | throws ToolInputError(400)
readNumberParam(params, key, options?)    → number
readStringArrayParam(params, key, options?) → string[]
readReactionParams(params, options)       → { emoji, remove }
```

### 5.3 Skills 系统 — 文件驱动的专业指令

> 源码：`src/agents/skills/workspace.ts`

Skills 是文件系统中的专业指令文件（`SKILL.md`），Agent 在需要时通过 `read` 工具加载。

**4 个加载来源**：

```
1. ~/.config/openclaw/skills/     ← 托管 Skills
2. <bundled>/skills/               ← 内置捆绑 Skills
3. <workspace>/.claude/skills/     ← 工作空间 Skills
4. 插件 Skills (resolvePluginSkillDirs())  ← 插件提供的 Skills
```

**加载管线**：

```
loadSkillEntries()  → 发现所有 Skill 目录
  → filterSkillEntries()  → 按 OS/bins/env/config 过滤 + skillFilter 过滤
    → formatSkillsForPrompt()  → 生成 XML
      → buildSkillsSection()  → 注入 System Prompt
```

**加载限制（默认值）**：

| 限制项                  | 默认值  |
| ----------------------- | ------- |
| 每目录最大候选数        | 300     |
| 最大加载 Skill 数       | 200     |
| Prompt 中最大 Skill 数  | 150     |
| Skill prompt 最大字符数 | 18,000  |
| 单 Skill 文件最大字节数 | 256,000 |

**注入到 System Prompt 的格式**：

```xml
## Skills
Scan <available_skills>. If one clearly applies, read its SKILL.md
at exact <location> with `read`, then follow it.

<available_skills>
  <skill>
    <name>pdf</name>
    <description>Extract text and tables from PDF files</description>
    <location>~/.openclaw/skills/pdf/SKILL.md</location>
  </skill>
  <skill>
    <name>big-commit</name>
    <description>Create well-structured commits from large diffs</description>
    <location>~/.openclaw/skills/big-commit/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

Skill 类型定义 (`types.ts`)：

```typescript
type SkillEntry = {
  skill: Skill; // 规范 Skill 对象
  frontmatter: ParsedSkillFrontmatter; // YAML frontmatter 解析结果
  metadata?: OpenClawSkillMetadata; // always?, skillKey, emoji, os, requires, install
  invocation?: SkillInvocationPolicy; // userInvocable, disableModelInvocation
  exposure?: SkillExposure; // 可见性控制
};

type SkillEligibilityContext = {
  remote?: {
    platforms: string[];
    hasBin: (bin: string) => boolean;
    hasAnyBin: (bins: string[]) => boolean;
  };
};
```

### 5.4 MCP 集成

> 源码：`src/mcp/plugin-tools-handlers.ts`, `src/agents/pi-bundle-mcp-runtime.ts`

**工具转换**：`createPluginToolsMcpHandlers(tools)` — 将 `AnyAgentTool[]` 转换为标准 MCP handler：

```typescript
// 输入: AnyAgentTool[]
// 输出:
{
  listTools: () => Promise<{ tools: MCPTool[] }>;
  callTool: (params: { name: string; arguments?: unknown }) => Promise<MCPCallResult>;
}
```

- `listTools()` 映射 `name + description + parameters` 到 MCP Schema
- `callTool()` 调用 `tool.execute()`，每个工具在调用前通过 `wrapToolWithBeforeToolCallHook()` 包装（审批、loop 检测、插件 hook）

**三种 MCP 传输协议** (`src/agents/mcp-transport.ts`)：

| 传输类型            | 实现                            | 适用场景              |
| ------------------- | ------------------------------- | --------------------- |
| **stdio**           | `OpenClawStdioClientTransport`  | 本地 MCP 服务器子进程 |
| **SSE**             | `SSEClientTransport` (MCP SDK)  | 远程 HTTP SSE 端点    |
| **streamable-http** | `StreamableHTTPClientTransport` | 远程 HTTP 流式端点    |

**Per-session MCP 运行时** (`pi-bundle-mcp-runtime.ts`)：

- 为每个 Agent session 创建独立的 MCP Client
- 管理生命周期：create → connect → listTools → callTool → dispose
- 定期清理过期 session（sweep）

### 5.5 Bash 沙盒与审批机制

> 源码：`src/agents/bash-tools.exec-runtime.ts`, `bash-tools.exec-approval-request.ts`

**三种执行模式**：

| 模式               | 实现                  | 隔离级别 |
| ------------------ | --------------------- | -------- |
| **PTY spawn**      | `node-pty` 伪终端     | 进程级   |
| **Child process**  | `child_process.spawn` | 进程级   |
| **Docker sandbox** | Docker exec           | 容器级   |

**Security 措施**：

```typescript
// 环境变量清理
sanitizeHostBaseEnv(env); // 阻断危险的宿主环境变量
validateHostEnv(env); // 在宿主执行时拒绝 PATH 修改

// PATH 安全
wrapPosixCommandWithPathPrepend(); // 在执行上下文中预置安全 PATH
```

**两阶段审批协议** (`bash-tools.exec-approval-request.ts`)：

```
Phase 1 — 注册审批请求:
  registerExecApprovalRequest(params)
    → Gateway RPC: exec.approval.request
    → 返回 { id, expiresAtMs, finalDecision? }

Phase 2 — 等待用户裁决:
  waitForExecApprovalDecision(id)
    → Gateway RPC: exec.approval.waitDecision
    → 阻塞等待，返回 "allow" | "deny" | null(超时)

组合入口:
  requestExecApprovalDecision(params)
    → register → 若有立即决策则返回
    → 否则 waitForExecApprovalDecision → 返回
```

**审批范围**：

- `turn` — 每次工具调用都需要审批
- `session` — 同一会话中审批一次后自动通过

### 5.6 Hook 系统 — Agent 生命周期拦截

> 源码：`src/hooks/types.ts`, `src/hooks/internal-hooks.ts`

```typescript
type Hook = {
  name: string;
  description: string;
  source: "openclaw-bundled" | "openclaw-managed" | "openclaw-workspace" | "openclaw-plugin";
  pluginId?: string;
  filePath: string; // HOOK.md 路径
  baseDir: string;
  handlerPath: string; // handler.ts/js 路径
};
```

**Agent 生命周期中的全部 Hook 切点**：

| Hook 事件           | 触发时机             | 用途                   |
| ------------------- | -------------------- | ---------------------- |
| `agent:bootstrap`   | Agent 启动 bootstrap | 启动前准备工作         |
| `message:received`  | 消息到达             | 消息预处理/过滤        |
| `before_agent_run`  | Agent 执行前         | **可拦截/修改 prompt** |
| `before_tool_call`  | 工具调用前           | 审批/参数修改          |
| `after_tool_call`   | 工具调用后           | 日志/副作用处理        |
| `agent_end`         | Agent turn 结束      | 记忆捕获/清理          |
| `llm_input`         | LLM 输入组装后       | 输入日志               |
| `llm_output`        | LLM 输出后           | 输出日志/分析          |
| `before_compaction` | 压缩前               | 压缩策略控制           |
| `after_compaction`  | 压缩后               | 压缩后处理             |
| `message:sent`      | 消息发送             | 发送通知               |
| `session:patch`     | 会话修改             | 会话变更监听           |
| `gateway:startup`   | Gateway 启动         | 启动时初始化           |

**Hook 加载管线** (`src/hooks/loader.ts`)：

```
loadInternalHooks(cfg, workspaceDir)
  │
  ├─ 1. resetLoadedInternalHooks() — 清除所有已注册 handler
  ├─ 2. 检查 hasConfiguredInternalHooks(cfg) — 若禁用则返回
  │
  ├─ 3. Phase A: 目录式 Hook (新系统)
  │   ├─ loadWorkspaceHookEntries() — 发现 bundled/managed/workspace 目录
  │   ├─ 按 configuredNames 和 shouldIncludeHook() 过滤
  │   └─ 对每个 Hook:
  │       ├─ 验证 handler path (openRootFile 边界检查)
  │       ├─ 动态 import() 处理器模块
  │       └─ 为每个 metadata.events 注册 handler
  │
  └─ 4. Phase B: Legacy config handlers (向后兼容)
      └─ getLegacyInternalHookHandlers(cfg)
```

**工具执行前门控** (`pi-tools.before-tool-call.ts:916`)：

```
runBeforeToolCallHook(toolName, tool, params, toolCallId, signal, ctx)
  │
  ├─ 1. Loop 检测 — 警告级（记录日志）/ 阻断级（阻止执行）
  ├─ 2. Trusted tool policies — 预批准的工具直接通过
  ├─ 3. before_tool_call plugin hooks — 插件拦截
  ├─ 4. Plugin approval — 两阶段审批 (plugin.approval.request + waitDecision)
  ├─ 5. Sandbox FS bridge — 沙盒文件系统策略
  │
  └─ 返回: { blocked: false, params } | { blocked: true, reason }
```

---

## 六、记忆系统设计

### 6.1 分层架构

```
┌──────────────────────────────────────────────┐
│              MEMORY.md (持久记忆)              │
│         由 Deep Dreaming 自动维护              │
└──────────────────┬───────────────────────────┘
                   │ 提升 (promotion)
┌──────────────────┴───────────────────────────┐
│      短期回忆 (Short-term Recall)             │
│   memory/.dreams/short-term-recall.json      │
│   多维度评分 (frequency/relevance/recency)    │
└──────────────────┬───────────────────────────┘
                   │ 摄取
┌──────────────────┴───────────────────────────┐
│    SQLite 索引 (FTS5 + Vector)               │
│    混合搜索 (语义 + 关键词)                    │
│    MemoryIndexManager                        │
└──────────────────┬───────────────────────────┘
                   │ 分块 + 嵌入
┌──────────────────┴───────────────────────────┐
│   记忆源文件                                   │
│   MEMORY.md + 每日记忆文件 + 会话记录          │
└──────────────────────────────────────────────┘
```

### 6.2 两种记忆后端

| 后端                   | 插件                         | 存储引擎               | 搜索方式                 | 特点                |
| ---------------------- | ---------------------------- | ---------------------- | ------------------------ | ------------------- |
| **memory-core** (默认) | `extensions/memory-core/`    | SQLite + FTS5 + Vector | 混合搜索 (语义 + 关键词) | 轻量、本地          |
| **memory-lancedb**     | `extensions/memory-lancedb/` | LanceDB                | 纯向量搜索               | 自动捕获 + 自动召回 |

### 6.3 混合搜索架构

> 源码：`extensions/memory-core/src/manager.ts` — `MemoryIndexManager.search()`

```
用户查询 "我的 Python 项目用什么 ORM"
  │
  ├─ 向量搜索 (语义相似度)
  │   → EmbeddingProvider.encode(query) → 余弦相似度 Top-K
  │
  ├─ FTS 关键词搜索 (BM25)
  │   → SQLite FTS5 MATCH → BM25 排名
  │
  ├─ mergeHybridResults()
  │   ├─ vectorWeight + textWeight 标准化 (总和=1.0)
  │   ├─ MMR (Maximum Marginal Relevance) 去重
  │   ├─ Temporal Decay (halfLifeDays 衰减曲线)
  │   └─ candidateMultiplier 控制候选数量 (最多 200)
  │
  └─ selectScoredResults()
      ├─ 严格过滤 (minScore ≥ 阈值)
      ├─ 宽松过滤 (放宽阈值)
      └─ 回退 (返回 Top-K)
```

**默认搜索参数**：

| 参数           | 默认值 |
| -------------- | ------ |
| `maxResults`   | 6      |
| `minScore`     | 0.35   |
| `chunkTokens`  | 400    |
| `chunkOverlap` | 80     |
| `vectorWeight` | 0.7    |
| `textWeight`   | 0.3    |

### 6.4 Dreaming — 后台记忆合成

> 源码：`extensions/memory-core/src/dreaming.ts`

三个 cron 驱动的 Dreaming 阶段，实现从短期回忆到持久记忆的自动提升：

| 阶段      | 默认频率        | 功能                                 | 核心参数                                            |
| --------- | --------------- | ------------------------------------ | --------------------------------------------------- |
| **Light** | 每 6 小时       | 快速日间扫描，摄取每日文件和会话信号 | 速度:`fast`, 预算:`cheap`, 限制:100, 去重相似度:0.9 |
| **Deep**  | 每天凌晨 3 点   | 深度合成，排名候选，提升到 MEMORY.md | 速度:`balanced`, 最低分:0.8, 最低回忆次数:3         |
| **REM**   | 每周日凌晨 5 点 | 每周模式识别，挖掘"持久事实"         | 速度:`slow`, 最低模式强度:0.75                      |

**短期回忆提升的多维度权重** (`short-term-promotion.ts`)：

```
综合评分 = Σ (维度分数 × 权重)

frequency:     0.24  — 回忆次数的对数比例
relevance:     0.30  — 来自记忆搜索的累积总分
diversity:     0.15  — 独特查询哈希数 (跨语境相关)
recency:       0.15  — 时间衰减 (14 天半衰期)
consolidation: 0.10  — 看到回忆的天数 (稳定性)
conceptual:    0.06  — TF-IDF 加权概念标签
```

### 6.5 Auto-Memory（memory-lancedb 插件）

> 源码：`extensions/memory-lancedb/src/index.ts`

通过 Agent 生命周期 Hook 实现全自动记忆管理：

**Auto-recall**（`before_prompt_build` hook）：

```
用户消息 → 提取文本 → 嵌入 → 语义搜索 (超时 15s)
  → 将匹配的记忆包装在 <relevant-memories> 标签中
  → 注入到上下文
```

**Auto-capture**（`agent_end` hook）：

```
扫描消息中的触发模式：
  ├─ 偏好 (preferences): "我喜欢...", "我习惯...", "我偏好..."
  ├─ 决策 (decisions): "决定...", "确认...", "最终方案是..."
  ├─ 实体 (entities): 人名、项目名、工具名
  └─ 事实 (facts): 明确的陈述性信息
→ 去重 (检查是否已存储)
→ 持久化存储
```

**安全检查**：`looksLikePromptInjection()` 拒绝提示注入文本，`shouldCapture()` 多层过滤（长度/格式/表情符号/系统内容）。

---

## 七、Session 会话管理

### 7.1 存储架构

| 工件                      | 格式                 | 位置                                                  | 用途                   |
| ------------------------- | -------------------- | ----------------------------------------------------- | ---------------------- |
| **Session Store**         | JSON map             | `~/.openclaw/agents/<agentId>/sessions/sessions.json` | 会话元数据             |
| **Transcript**            | JSONL (每行一条消息) | `<sessionDir>/<sessionId>.jsonl`                      | 对话历史               |
| **Archive**               | JSONL + 时间戳       | `<sessionId>.jsonl.bak.<ISO时间戳>`                   | 删除/重置/压缩时的备份 |
| **Compaction Checkpoint** | JSONL                | `<sessionId>.checkpoint.<uuid>.jsonl`                 | 压缩前快照             |
| **Trajectory**            | JSONL                | `<sessionId>.trajectory.jsonl`                        | 执行轨迹               |

### 7.2 SessionEntry 结构

> 源码：`src/config/sessions/types.ts:176-369` — 约 100 个字段

核心字段按功能分组：

```typescript
type SessionEntry = {
  // ━━━ 身份标识 ━━━
  sessionId: string; // UUID v4
  sessionKey: string; // 路由 Key (如 "agent:main:main")
  label?: string; // 人类可读标签 (≤512 字符)
  updatedAt: number; // 最后更新时间戳
  sessionStartedAt?: number; // 首次活跃时间
  sessionFile?: string; // JSONL Transcript 路径

  // ━━━ 模型绑定 ━━━
  model?: string; // 当前使用的模型
  modelProvider?: string; // Provider ID
  agentHarnessId?: string; // Harness 锚点
  thinkingLevel?: string; // 思考级别
  fastMode?: boolean; // 快速模式

  // ━━━ 路由信息 ━━━
  channel?: string; // 规范化 Channel ID
  chatType?: "direct" | "group" | "channel";
  groupId?: string;
  origin?: SessionOrigin; // 首条消息上下文快照
  route?: ChannelRouteRef; // 当前通道路由

  // ━━━ 子 Agent ━━━
  spawnDepth?: number; // 0=main, 1=subagent, 2=sub-subagent
  subagentRole?: "orchestrator" | "leaf";
  pluginOwnerId?: string;

  // ━━━ Token / 配额 ━━━
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  estimatedCostUsd?: number;
  quotaSuspension?: QuotaSuspension; // 限流级联保护

  // ━━━ 压缩 ━━━
  compactionCount?: number;
  compactionCheckpoints?: SessionCompactionCheckpoint[];

  // ━━━ ACP (Agent Communication Protocol) ━━━
  acp?: {
    backend: string;
    agent: string;
    mode: "persistent" | "oneshot";
    state: "idle" | "running" | "error";
  };
};
```

### 7.3 Session Key 命名规范

> 源码：`src/routing/session-key.ts`

```
agent:main:main                               — 默认主 DM
agent:main:direct:<peerId>                    — 按对等方的 DM
agent:main:<channel>:direct:<peerId>          — 按 Channel 的对等方 DM
agent:main:<channel>:<accountId>:direct:<id>  — 按账户的对等方 DM
agent:main:<channel>:group:<peerId>           — 群组会话
agent:main:<channel>:<accountId>:group:<id>   — 按账户的群组会话
agent:main:...:thread:<normalizedThreadId>    — 线程会话
global                                        — 全局单例
```

**推导流程** (`session-key.ts`)：

```
MsgContext 到达
  → ctx.SessionKey 显式指定? → 直接使用
  → ChatType === "group"? → resolveGroupSessionKey()
  → 否则 → "agent:<agentId>:<mainKey>:direct:<from>"
```

### 7.4 并发控制

| 操作                | 机制                                                | 文件                   |
| ------------------- | --------------------------------------------------- | ---------------------- |
| **Store 写入**      | `WriterQueue` 严格串行化（每个 storePath 一个队列） | `store-writer.ts`      |
| **Transcript 追加** | 跨进程文件锁 `acquireSessionWriteLock()`            | `transcript-append.ts` |
| **Store 读取**      | 无锁快照缓存（45s TTL，`deepFreeze` 不可变）        | `store-cache.ts`       |

**三层缓存** (`store-cache.ts`)：

- **对象缓存** (MUTABLE) — 写入路径使用，TTL 45s
- **快照缓存** (DeepReadonly) — 读取路径使用，deepFreeze 确保不可变
- **序列化缓存** (原始 JSON) — 写入去重（64 条目或 64MB 上限）

### 7.5 维护与清理

> 源码：`src/config/sessions/store-maintenance.ts`

| 参数             | 默认值             | 说明               |
| ---------------- | ------------------ | ------------------ |
| `pruneAfterMs`   | 30 天              | 陈旧条目的 TTL     |
| `maxEntries`     | 500                | 最大 Session 数量  |
| `maxDiskBytes`   | null               | 磁盘软限制         |
| `highWaterBytes` | maxDiskBytes × 80% | 触发清理的高水位线 |

**保护机制**（以下 Session 永不过期）：

- 群组/频道/线程 Session
- Telegram 话题 Session
- 外部平台群组/频道 Session

**清理优先级**（先删合成 Session）：

- `subagent:*` > `acp:*` > `cron:*` > `hook:*` > `node:*` > `heartbeat:*`
- 然后按 `updatedAt` 升序删除普通 DM

### 7.6 Transcript 格式

> 源码：`src/config/sessions/transcript-append.ts`

```jsonl
{"type":"session","version":6,"id":"<uuid>","timestamp":"...","cwd":"..."}
{"type":"message","id":"a1b2c3d4","parentId":null,"timestamp":"...","message":{"role":"user","content":"你好"}}
{"type":"message","id":"e5f6g7h8","parentId":"a1b2c3d4","timestamp":"...","message":{"role":"assistant","content":"你好！"}}
```

- `parentId` 指向链中前一条消息，形成 parent-linked DAG
- 旧格式（线性，无 parentId）在写入时自动迁移
- 消息 ID 为 `randomUUID().slice(0, 8)` 的 8 字符短 ID
- 写入前通过 `redactTranscriptMessage()` 清理机密信息

### 7.7 Session 重置策略

> 源码：`src/config/sessions/reset-policy.ts`

两种模式：

| 模式      | 行为             | 配置                      |
| --------- | ---------------- | ------------------------- |
| **daily** | 每天固定时间重置 | `atHour`（默认凌晨 4 点） |
| **idle**  | 不活动后重置     | `idleMinutes`             |

支持 `resetByChannel` 按 Channel 覆盖重置策略。

---

## 八、Context Engine — 可插拔的上下文管理

### 8.1 接口定义

> 源码：`src/context-engine/types.ts:208-356`

```typescript
interface ContextEngine {
  // 生命周期
  bootstrap(params): Promise<BootstrapResult>;
  dispose(): Promise<void>;

  // 消息摄入
  ingest(params): Promise<IngestResult>;
  ingestBatch(params): Promise<IngestResult>;

  // 上下文组装 (核心 API)
  assemble(params: {
    tokenBudget: number;
    contextProjection?: ContextProjection;
  }): Promise<AssembleResult>;
  // → { messages: AgentMessage[], estimatedTokens: number,
  //     systemPromptAddition?: string }

  // 维护
  afterTurn(params): Promise<void>;
  compact(params): Promise<CompactResult>;

  // 子 Agent
  prepareSubagentSpawn(params): Promise<...>;
  onSubagentEnded(params): Promise<void>;
}
```

### 8.2 注册与选择

> 源码：`src/context-engine/registry.ts`

```typescript
// 进程全局单例 (Symbol.for 确保跨打包块共享)
registerContextEngine(id, factory);

// 解析引擎 —— 按优先级:
resolveContextEngine(config)
  → config.plugins.slots.contextEngine (显式槽位)
  → 默认 "legacy"
```

默认的 `LegacyContextEngine` (`legacy.ts`)：

- `ingest()` → no-op (SessionManager 处理消息持久化)
- `assemble()` → 透传消息
- `compact()` → 委托给 `compactEmbeddedPiSessionDirect`

第三方可通过 `registerContextEngine()` API 完全替换上下文管理行为。

---

## 九、Compaction — 上下文压缩

### 9.1 触发条件

当 `estimatedTokens > tokenBudget × MIN_PROMPT_BUDGET_RATIO(0.5)` 时触发压缩，且压缩后至少保留 `MIN_PROMPT_BUDGET_TOKENS(8000)` 可用。

### 9.2 三级回退策略

> 源码：`src/agents/compaction.ts:402` — `summarizeWithFallback()`

```
summarizeWithFallback(messages, tokenBudget)
  │
  ├─ 1. 完全摘要
  │   → chunkMessagesByMaxTokens() 分块 (SAFETY_MARGIN=1.2)
  │   → 每块并行摘要
  │   → summarizeInStages() 多阶段合并 (默认 minMessagesForSplit=4)
  │
  ├─ 2. 部分回退
  │   → 排除超大消息 (单条 > 50% 上下文窗口)
  │   → 对剩余消息执行完全摘要
  │
  └─ 3. 最终回退
      → 返回消息计数摘要
```

### 9.3 Model Handoff 专用摘要

> 源码：`src/agents/compaction.ts:602` — `summarizeForHandoff()`

模型切换时生成专门的交接摘要：

- 4000 token 上限
- 包含"领导者等级制度强化"指令
- `pruneHistoryForContextShare()` 修剪最旧消息块（`maxHistoryShare=0.2`）

---

## 十、System Prompt 组装

### 10.1 构建函数

> 源码：`src/agents/system-prompt.ts:670-1317`

```typescript
function buildAgentSystemPrompt(params: {
  workspaceDir: string;
  sandboxConfig?: BashSandboxConfig;
  channelCapabilities?: ChannelCapabilities;
  availableTools?: AnyAgentTool[];
  skillsPrompt?: string;
  bootstrapFiles?: BootstrapFile[];
  heartbeatConfig?: HeartbeatConfig;
  // ... ~50 个参数
}): string;
```

### 10.2 结构布局

```
═══════ 稳定前缀 (Cache 友好) ═══════
  You are a personal assistant running inside OpenClaw.

  ## Tooling
  可用工具列表及摘要 (read/write/edit/exec/web_search/...)

  ## Tool Call Style
  工具调用规范、approval 流程、并行调用指南

  ## Execution Bias / Safety
  偏向执行的行为指示、安全约束

  ## OpenClaw Control
  Gateway 管理指令

  ## Skills
  <available_skills> XML 块

  ## Memory
  记忆使用指南

  ## Model Aliases / Timezone

  ## Workspace
  工作目录、sandbox 配置、文件系统访问策略

  ## Authorized Senders / Bootstrap Status

  ## Workspace Files (静态上下文)
  AGENTS.md, SOUL.md, IDENTITY.md, USER.md, TOOLS.md, BOOTSTRAP.md, MEMORY.md

  ## Silent Replies
  静默回复 token
─────── SYSTEM_PROMPT_CACHE_BOUNDARY ───────
  ═══════ 动态后缀 (不缓存) ═══════

  Dynamic Project Context (heartbeat.md 等动态文件)
  Webchat Canvas — 控制台 UI 嵌入说明
  Messaging — 消息路由指南
  Voice (TTS) — 语音合成提示
  Group Chat Context — 群组上下文
  Reactions — 反应指南
  Heartbeats — 心跳响应说明
```

### 10.3 缓存策略

稳定前缀使用 **LRU Map** 缓存（最多 64 个条目），以 `SHA-256(JSON.stringify(inputs))` 为 key。上下文文件按固定顺序加载（排序权重）：

```
AGENTS.md(10) < SOUL.md(20) < IDENTITY.md(30) < USER.md(40)
< TOOLS.md(50) < BOOTSTRAP.md(60) < MEMORY.md(70)
```

---

## 十一、执行流程全景图

### 11.1 Agent Turn 完整数据流

```
调度层 dispatchReplyFromConfig()
  │
  ▼
runEmbeddedPiAgent()                           [run.ts:405]
  │
  ├─ Lane 队列调度
  │   ├─ resolveSessionLane()  → per-session lane
  │   ├─ enqueueSession()      → session lane 串行化
  │   └─ enqueueGlobal()       → global lane 并发控制
  │
  └─ runEmbeddedAttempt()                     [attempt.ts:1155]
       │
       ├─ [初始化] workspace/sandbox/skills 环境变量
       │
       ├─ [工具] createOpenClawTools()        [openclaw-tools.ts:529]
       │   ├─ 内建 33 个工具 (按 Profile 过滤)
       │   ├─ 插件工具 (resolveOpenClawPluginToolsForOptions)
       │   └─ bundle MCP/LSP 运行时工具
       │
       ├─ [System Prompt] buildAttemptSystemPrompt()
       │   ├─ 稳定前缀: 工具列表 + Skills + Memory + Workspace
       │   ├─ Bootstrap 文件注入 (AGENTS.md/SOUL.md/...)
       │   └─ 动态后缀: heartbeat + Voice + Group Context
       │
       ├─ [Context Engine] bootstrap() → assemble()
       │   └─ 在 token 预算下组装 AgentMessage[]
       │
       ├─ [Session] SessionManager.open()
       │   ├─ 加载 SessionEntry (JSON)
       │   ├─ 加载 Transcript (JSONL)
       │   └─ 安装写锁 + 事件处理器
       │
       ├─ [Model Selection]
       │   ├─ resolveDefaultModelForAgent()      → 默认模型
       │   ├─ resolveChannelModelCandidate()     → Channel 级覆盖
       │   ├─ resolveStoredModelCandidate()      → 会话级覆盖
       │   └─ selectAgentHarness()               → PI/Codex/Claude-CLI
       │
       ├─ [Prompt 提交]
       │   ├─ before_agent_run hook (可拦截/修改 prompt)
       │   ├─ shouldPreemptivelyCompactBeforePrompt()
       │   └─ promptActiveSession()
       │       │
       │       ├─ runWithModelFallback()         [model-fallback.ts]
       │       │   ├─ resolveAuthProfileOrder()   (Key 轮换)
       │       │   └─ runFallbackCandidate()
       │       │       └─ registerProviderStreamForModel()
       │       │           └─ Provider 插件 wrapStreamFn()
       │       │               └─ HTTPS SSE → 大模型 API
       │       │
       │       └─ 15 层 Stream 管道处理响应
       │           ├─ 文本转换 / 推理块清理
       │           ├─ 工具调用修复 / 兼容层
       │           ├─ 空闲超时检测
       │           └─ 诊断事件
       │
       ├─ [Compaction]
       │   ├─ 超 token 预算 → waitForCompactionRetry (60s 聚合超时)
       │   └─ summarizeWithFallback() → 三级回退
       │
       └─ [收尾]
           ├─ 构建 EmbeddedRunAttemptResult
           ├─ agent_end / llm_output hooks (fire-and-forget)
           ├─ 更新会话状态 + 持久化
           └─ 释放会话锁
```

### 11.2 回复双路径并行

```
Agent 产出 ReplyPayload
  │
  ├─ [路径 1: 永远执行]
  │   → Gateway 事件推送 (event:agent)
  │   → 所有已连接 Client 实时收到流式事件
  │
  └─ [路径 2: 如有 Channel 出站目标]
      → Channel 插件 send
      → 外部平台 API (sendMessage / editMessage)
```

---

## 十二、关键接口定义文件清单

| 类别               | 文件                                             | 描述                                   |
| ------------------ | ------------------------------------------------ | -------------------------------------- |
| **Harness 接口**   | `src/agents/harness/types.ts`                    | `AgentHarness` 接口                    |
| **Harness 注册表** | `src/agents/harness/registry.ts`                 | 全局注册中心                           |
| **Harness 选择**   | `src/agents/harness/selection.ts`                | 选择算法 + 执行入口                    |
| **PI 执行入口**    | `src/agents/pi-embedded-runner/run.ts`           | `runEmbeddedPiAgent()`                 |
| **PI 核心循环**    | `src/agents/pi-embedded-runner/run/attempt.ts`   | `runEmbeddedAttempt()` (~5000 行)      |
| **Provider 注册**  | `src/plugin-sdk/provider-entry.ts`               | `defineSingleProviderPluginEntry`      |
| **Provider 类型**  | `src/plugins/types.ts`                           | `ProviderPlugin` 完整类型 (~50 个钩子) |
| **模型回退**       | `src/agents/model-fallback.ts`                   | `runWithModelFallback()`               |
| **Auth Profiles**  | `src/agents/auth-profiles/order.ts`              | `resolveAuthProfileOrder()`            |
| **工具目录**       | `src/agents/tool-catalog.ts`                     | 33 个工具 + 4 种 profile               |
| **工具工厂**       | `src/agents/openclaw-tools.ts`                   | `createOpenClawTools()`                |
| **工具基础类型**   | `src/agents/tools/common.ts`                     | `AnyAgentTool`, 参数读取器             |
| **Skills 加载**    | `src/agents/skills/workspace.ts`                 | `buildWorkspaceSkillsPrompt()`         |
| **Skills 类型**    | `src/agents/skills/types.ts`                     | `SkillEntry`, `SkillSnapshot`          |
| **MCP 工具转换**   | `src/mcp/plugin-tools-handlers.ts`               | `createPluginToolsMcpHandlers()`       |
| **Bash 执行**      | `src/agents/bash-tools.exec-runtime.ts`          | `runExecProcess()`                     |
| **审批协议**       | `src/agents/bash-tools.exec-approval-request.ts` | 两阶段审批                             |
| **Hook 系统**      | `src/hooks/types.ts`                             | `Hook`, `HookEntry`                    |
| **Hook 加载**      | `src/hooks/loader.ts`                            | `loadInternalHooks()`                  |
| **记忆搜索**       | `src/agents/memory-search.ts`                    | `ResolvedMemorySearchConfig`           |
| **记忆索引**       | `extensions/memory-core/src/manager.ts`          | `MemoryIndexManager`                   |
| **Dreaming**       | `extensions/memory-core/src/dreaming.ts`         | Light/Deep/REM 调度                    |
| **Session 类型**   | `src/config/sessions/types.ts`                   | `SessionEntry` (~100 字段)             |
| **Session Store**  | `src/config/sessions/store.ts`                   | CRUD + 原子写                          |
| **Session Key**    | `src/routing/session-key.ts`                     | Key 命名与派生                         |
| **Context Engine** | `src/context-engine/types.ts`                    | `ContextEngine` 接口                   |
| **Compaction**     | `src/agents/compaction.ts`                       | `summarizeWithFallback()`              |
| **System Prompt**  | `src/agents/system-prompt.ts`                    | `buildAgentSystemPrompt()`             |
| **Bootstrap 文件** | `src/agents/bootstrap-files.ts`                  | `resolveBootstrapFilesForRun()`        |
| **Model 选择**     | `src/agents/model-selection-resolve.ts`          | 模型解析与选择                         |

---

## 十三、二次开发指南

### 13.1 接入新模型提供商

**步骤**：

1. 创建 `extensions/<provider>/` 目录
2. 编写 `openclaw.plugin.json` 清单
3. 编写 `package.json`
4. 实现 `index.ts` — 使用 `defineSingleProviderPluginEntry()` 注册
5. (可选) 实现 `stream.ts` — 自定义 `wrapStreamFn`
6. (可选) 实现 `thinking.ts` — 自定义 `resolveThinkingProfile`
7. 在 `openclaw.json` 中配置 API Key

**步骤 1-3: 创建目录与清单**：

```bash
mkdir -p extensions/my-provider/src
```

```json
// extensions/my-provider/openclaw.plugin.json
{
  "id": "my-provider",
  "activation": { "onStartup": false },
  "enabledByDefault": true,
  "providers": ["my-provider"],
  "providerAuthEnvVars": {
    "my-provider": ["MY_PROVIDER_API_KEY"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": { "type": "string" }
    }
  }
}
```

**步骤 4: 最小实现模板** (OpenAI 兼容 API)：

```typescript
// extensions/my-provider/index.ts
import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

export default defineSingleProviderPluginEntry({
  id: "my-provider",
  name: "My Provider",
  description: "Custom LLM provider plugin",
  provider: {
    label: "My Provider",
    docsPath: "/providers/my-provider",
    auth: [
      {
        methodId: "api-key",
        label: "API Key",
        envVar: "MY_PROVIDER_API_KEY",
        defaultModel: "my-provider/my-model",
      },
    ],
    catalog: {
      buildProvider: (cfg) => ({
        baseUrl: "https://api.my-provider.com/v1",
        api: "openai-completions",
        headers: {
          Authorization: `Bearer ${cfg.models?.providers?.["my-provider"]?.apiKey}`,
        },
        models: [
          { id: "my-model", name: "My Model", contextWindow: 128000 },
          { id: "my-model-fast", name: "My Model Fast", contextWindow: 128000 },
        ],
      }),
    },
    // 可选：上下文溢出检测
    matchesContextOverflowError: ({ errorMessage }) =>
      /\bcontext.*(?:too\s*long|exceed|overflow)\b/i.test(errorMessage),

    // 可选：推理级别 (如 provider 支持 thinking/reasoning)
    resolveThinkingProfile: ({ modelId }) => ({
      levels: [{ id: "off" }, { id: "medium" }, { id: "high" }],
      defaultLevel: "medium",
    }),
  },
});
```

**步骤 5: 高级 — 自定义流式包装器**：

```typescript
// extensions/my-provider/stream.ts
import type { ProviderWrapStreamFnContext } from "openclaw/plugin-sdk/provider-stream";

export function createMyProviderStreamWrapper(
  baseStreamFn: ProviderWrapStreamFnContext["streamFn"],
  options?: { thinkingLevel?: string },
): ProviderWrapStreamFnContext["streamFn"] {
  return async function* (payload, signal) {
    // 在 payload 中注入 provider 特有参数
    const enrichedPayload = {
      ...payload,
      my_provider_option: options?.thinkingLevel ?? "auto",
    };

    // 调用底层 stream
    for await (const event of baseStreamFn(enrichedPayload, signal)) {
      // 可选：转换响应事件
      yield event;
    }
  };
}
```

```typescript
// 在 index.ts 的 provider 中引用:
provider: {
  // ...
  wrapStreamFn: (ctx) =>
    createMyProviderStreamWrapper(ctx.streamFn, { thinkingLevel: ctx.thinkingLevel }),
}
```

**步骤 6: 在 openclaw.json 中配置**：

```json
{
  "models": {
    "providers": {
      "my-provider": {
        "apiKey": "sk-xxxxxxxxxxxxxxxxxxxxxxxx"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "my-provider/my-model"
    }
  }
}
```

### 13.1a 模型回退配置示例

```json
{
  "agents": {
    "defaults": {
      "model": "my-provider/my-model",
      "modelFallback": {
        "models": ["openai/gpt-4o", "deepseek/deepseek-v4-flash"],
        "strategy": "sequential"
      }
    }
  }
}
```

### 13.1b Auth Profiles 配置示例（多 Key 轮换）

````json
{
  "agents": {
    "defaults": {
      "model": "openai/gpt-4o"
    }
  },
  "auth": {
    "profiles": {
      "openai": [
        { "profileId": "key-1", "credential": { "apiKey": "sk-aaa..." } },
        { "profileId": "key-2", "credential": { "apiKey": "sk-bbb..." } }
      ]
    }
  }
}

```typescript
// extensions/my-provider/index.ts
import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

export default defineSingleProviderPluginEntry({
  id: "my-provider",
  name: "My Provider",
  description: "Custom LLM provider plugin",
  provider: {
    label: "My Provider",
    docsPath: "/providers/my-provider",
    auth: [{
      methodId: "api-key",
      label: "API Key",
      envVar: "MY_PROVIDER_API_KEY",
    }],
    catalog: {
      buildProvider: (cfg) => ({
        baseUrl: "https://api.my-provider.com/v1",
        api: "openai-completions",
        headers: { Authorization: `Bearer ${cfg.models?.providers?.["my-provider"]?.apiKey}` },
        models: [{ id: "my-model", name: "My Model", contextWindow: 128000 }],
      }),
    },
    // 可选：思考级别
    resolveThinkingProfile: ({ modelId }) => ({
      levels: [{ id: "off" }, { id: "medium" }, { id: "high" }],
      defaultLevel: "medium",
    }),
  },
});
````

**需实现的 ProviderPlugin 钩子（按优先级）**：

| 优先级 | 钩子                              | 说明                                         |
| ------ | --------------------------------- | -------------------------------------------- |
| P0     | `catalog.buildProvider`           | 模型目录（baseUrl + API 类型 + 模型列表）    |
| P0     | `auth`                            | API Key 环境变量                             |
| P1     | `resolveThinkingProfile`          | 推理级别支持                                 |
| P1     | `wrapStreamFn`                    | 请求/响应体修改（如注入 `reasoning_effort`） |
| P2     | `matchesContextOverflowError`     | 上下文溢出错误识别（用于触发回退）           |
| P2     | `buildReplayPolicy`               | 压缩重放策略                                 |
| P3     | `normalizeToolSchemas`            | 工具 Schema 兼容性转换                       |
| P3     | `resolveSystemPromptContribution` | 追加 Provider 特有的 system prompt           |

### 13.2 新增 Agent 工具

**步骤**：

1. 在 `src/agents/tool-catalog.ts` 的 `CORE_TOOL_DEFINITIONS` 中添加工具声明
2. 在 `src/agents/openclaw-tools.ts` 的 `createOpenClawTools()` 中实例化工具
3. 实现工具函数（参考现有工具模式）

**工具实现模板**：

```typescript
import { textResult, readStringParam, ToolInputError } from "./common.js";

export const myTool: AnyAgentTool = {
  name: "my_tool",
  description: "Description of what this tool does",
  parameters: {
    type: "object",
    properties: {
      input_param: { type: "string", description: "..." },
    },
    required: ["input_param"],
  },
  execute: async (toolCallId, params, signal) => {
    const input = readStringParam(params, "input_param", { required: true });
    // ... 实现逻辑 ...
    return textResult("Result text", { toolCallId });
  },
};
```

### 13.3 新增 Skill

Skills 是纯粹的文件，无需编码：

```
skills/my-skill/
  └── SKILL.md     ← Frontmatter (YAML) + Markdown 指令
```

SKILL.md 格式：

```markdown
---
name: my-skill
description: What this skill helps with
always: false
os: [linux, macos]
requires:
  bins: [ffmpeg]
  env: [MY_API_KEY]
---

# My Skill

Instructions for the agent to follow...
```

放置到以下任一目录即自动发现：

- `~/.config/openclaw/skills/my-skill/`
- `<workspace>/.claude/skills/my-skill/`

### 13.4 新增 Hook

```
hooks/my-hook/
  ├── HOOK.md       ← Frontmatter + 说明
  └── handler.ts    ← 事件处理器代码
```

HOOK.md 格式：

```markdown
---
events: [before_agent_run, agent_end]
---

# My Hook

Description of what this hook does.
```

handler.ts 格式：

```typescript
export default async function handler(event: InternalHookEvent) {
  if (event.action === "before_agent_run") {
    // 在 Agent 执行前拦截...
  }
}
```

### 13.5 替换 Context Engine

实现 `ContextEngine` 接口并通过 `registerContextEngine()` 注册：

```typescript
import { registerContextEngine } from "openclaw/plugin-sdk/context-engine";

registerContextEngine("my-engine", (ctx) => ({
  async bootstrap(params) {
    /* ... */
  },
  async assemble(params) {
    /* ... */
  },
  async compact(params) {
    /* ... */
  },
  async dispose() {
    /* ... */
  },
}));
```

然后在 `openclaw.json` 中指定：

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "my-engine"
    }
  }
}
```

---

## 十四、总结

Agent 层的核心设计遵循以下原则：

1. **Harness 抽象** — Agent Runtime 可插拔（PI/Codex/Claude CLI），通过 `supports()` + `runAttempt()` 极简接口统一
2. **Provider 隔离** — Agent 层不直接调用 LLM 厂商 API，通过 L6 Provider 插件和 `streamCompletion` 抽象接入 50+ 模型提供商
3. **三层能力体系** — 内建 Tools（33 个，4 种 Profile）+ Skills（文件驱动，XML 注入）+ MCP（外部进程，三种传输），通过统一 Tool Schema 暴露给 LLM
4. **分层记忆** — 短期工作记忆 (JSON) → Dreaming 三阶段 (Light/Deep/REM) → 持久记忆 (MEMORY.md) → SQLite 混合搜索 (FTS + Vector)
5. **文件系统会话** — JSON 元数据 + JSONL 对话历史，WriterQueue 写锁串行化，30 天 TTL + 500 上限自动维护
6. **可插拔上下文引擎** — `ContextEngine` 接口允许完全替换上下文管理策略
7. **三级压缩回退** — 完全摘要 → 部分回退 → 消息计数，确保单进程在各种上下文压力下不崩溃
8. **Pipeline 架构** — 15+ Stream Wrapper 通过函数组合形成处理管道，职责单一可测试

Agent 层的二次开发优先级排序：

- **P0**：接入新模型提供商（Provider 插件，最常用）
- **P1**：新增 Agent 工具（内建 Tool）
- **P2**：新增 Skill / Hook（文件驱动，无需修改代码）
- **P3**：替换 Context Engine（高级需求，需实现完整接口）
- **P4**：新增 Harness（极高复杂度，需实现完整 Agent 后端）

---

## 十五、ACP — Agent 间通信协议

### 15.1 设计目标

ACP (Agent Communication Protocol) 使 OpenClaw Agent 能与其他 Agent 进程进行标准化通信。它基于 `@agentclientprotocol/sdk`，实现了 Agent-to-Agent 的客户端/服务端/控制平面三层架构。

### 15.2 源码目录

```
src/acp/                                  ← ACP 协议实现 (~20 文件)
├── client.ts                             ← ACP 客户端 (spawn 对端进程)
├── client-helpers.ts                     ← 客户端辅助函数
├── commands.ts                           ← ACP 命令
├── meta.ts                               ← ACP 元数据
├── conversation-id.ts                    ← 会话 ID 规范化
├── normalize-text.ts                     ← 文本归一化
├── event-ledger.ts                       ← 事件账本
├── event-mapper.ts                       ← 事件映射 (ACP ↔ OpenClaw)
├── approval-classifier.ts                ← 审批分类
├── permission-relay.ts                   ← 权限中继
├── spawn.ts                              ← 对端进程 spawn
├── runtime-cache.ts                      ← 运行时缓存
├── session-actor-queue.ts                ← 会话 Actor 队列
├── persistent-bindings.lifecycle.ts       ← 持久绑定生命周期
├── persistent-bindings.resolve.ts         ← 持久绑定解析
└── control-plane/                        ← ACP 控制平面
    ├── manager.ts                        ← 主管理器
    ├── manager.core.ts                   ← 核心管理逻辑
    ├── manager.identity-reconcile.ts      ← 身份协调
    ├── manager.runtime-controls.ts        ← 运行时控制
    └── manager.turn-stream.ts            ← Turn 流管理

src/agents/                               ← Agent 侧的 ACP 集成
├── acp-spawn.ts                          ← Agent spawn ACP 会话
├── acp-spawn-parent-stream.ts            ← 父 Agent 流管理
├── acp-runtime-overlay.ts                ← ACP 运行时覆盖
└── acp-binding-*.ts                      ← ACP 绑定相关

src/config/types.acp.ts                   ← ACP 配置类型
src/cli/acp-cli.ts                        ← ACP CLI 命令
```

### 15.3 三层架构

```
┌─────────────────────────────────────────────┐
│              ACP Client (Agent 侧)           │
│  client.ts — spawn ACP 对端进程             │
│  spawn.ts — 子进程生命周期                   │
│  event-mapper.ts — 事件双向转换              │
└──────────────────┬──────────────────────────┘
                   │ ACP 协议 (stdio/SSE/HTTP)
┌──────────────────┴──────────────────────────┐
│          ACP Server (对端进程)               │
│  独立进程运行，接收 ACP 指令                  │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────┴──────────────────────────┐
│        ACP Control Plane (控制平面)          │
│  manager.ts — 会话/Turn 流管理               │
│  identity-reconcile.ts — 身份协调            │
│  persistent-bindings — 持久绑定              │
│  session-actor-queue — 会话队列              │
└─────────────────────────────────────────────┘
```

### 15.4 关键类型

```typescript
// ACP 会话模式
type AcpSessionMode = "persistent" | "oneshot";

// ACP 会话状态
type AcpSessionState = "idle" | "running" | "error";

// ACP 会话元数据 (存储在 SessionEntry.acp 中)
type SessionAcpMeta = {
  backend: string; // ACP 后端标识
  agent: string; // 对端 Agent 名称
  mode: AcpSessionMode; // 持久 / 一次性
  state: AcpSessionState; // 当前状态
  identity?: SessionAcpIdentity;
  runtimeOptions?: AcpSessionRuntimeOptions;
};
```

### 15.5 与其他层的交互

```
L5 Agent 层
  ├─ acp-spawn.ts → 启动 ACP 子 Agent 会话
  ├─ Agent turn → 通过 ACP 与子 Agent 通信
  └─ acp-binding → 建立持久 Agent 间绑定

L2 Gateway 层
  └─ SessionEntry.acp → 会话级 ACP 状态管理

L6 Plugin 层
  └─ extensions/acpx/ → ACP 扩展插件
```

---

## 十六、媒体管线 — 理解与生成

### 16.1 体系总览

媒体管线是 OpenClaw 中一个 **200+ 文件的独立子系统**，拥有自己的 Provider 抽象、能力建模和运行时调度。它处理从"理解"（image→text, audio→transcript）到"生成"（text→image/video/music/speech）到"实时交互"（Talk 语音对话）的完整链路。

### 16.2 源码目录与分层

```
┌─────────────────────────────────────────────┐
│         媒体基础层 (src/media/)              │
│  文件 I/O · 格式检测 · MIME 嗅探             │
│  FFmpeg 执行 · 音频转码 · 图像操作           │
│  PDF 提取 · QR 生成 · 字节流限制             │
└──────────────────┬──────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ 理解管线  │ │ 生成管线  │ │ 实时交互  │
│ src/     │ │ src/     │ │ src/     │
│ media-   │ │ image-   │ │ talk/    │
│ under-   │ │ gener-   │ │          │
│ standing │ │ ation    │ │ 语音会话  │
│          │ │ video-   │ │ 双向音频  │
│ 图像→文本 │ │ gener-   │ │ WebRTC/  │
│ 音频→转录 │ │ ation    │ │ PCM      │
│ 视频→分析 │ │ music-   │ │          │
│          │ │ gener-   │ │          │
│ 多Provider│ │ ation    │ │          │
│ 路由     │ │ tts/     │ │          │
│          │ │          │ │          │
│          │ │ 文本→    │ │          │
│          │ │ 图像/    │ │          │
│          │ │ 视频/    │ │          │
│          │ │ 音乐/    │ │          │
│          │ │ 语音     │ │          │
└──────────┘ └──────────┘ └──────────┘
    │              │              │
    └──────────────┴──────────────┘
                   │
       L6 Plugin 层 (Provider 实现)
       extensions/fal/, comfy/, runway/,
       elevenlabs/, azure-speech/, ...
```

### 16.3 统一的 Provider 抽象

每种媒体类型都遵循相同的 Provider 注册模式：

```
types.ts → provider-registry.ts → runtime.ts → provider-specific/
```

**示例 — TTS Provider**：

```typescript
// src/tts/types.ts
type SpeechProvider = {
  id: string;
  label: string;
  synthesize(params: {
    text: string;
    voice: string;
    format: "mp3" | "wav" | "opus";
  }): Promise<{ audio: Buffer; format: string }>;
};

// Agent 层通过 Plugin SDK 调用:
api.registerSpeechProvider(myTtsProvider);
```

### 16.4 Agent 与媒体管线的交互

```
Agent turn 执行中
  │
  ├─ LLM 返回 tool_call: image_generate { prompt: "..." }
  │   → createOpenClawTools() → image-generate-tool.ts
  │   → resolveImageGenerationProvider() → 选择 Provider
  │   → Provider.generate() → 返回图片
  │
  ├─ 用户发送语音消息
  │   → MediaUnderstanding → audio→transcript
  │   → transcript 作为消息文本传入 Agent
  │
  └─ Agent 回复 → TTS synthesize → 语音输出
      → Talk session → 实时双向语音对话
```

### 16.5 媒体管线关键文件清单

| 层级     | 目录                       | 核心文件                                                     | 说明                     |
| -------- | -------------------------- | ------------------------------------------------------------ | ------------------------ |
| 基础     | `src/media/`               | `mime.ts`, `ffmpeg.ts`, `image-ops.ts`, `audio-transcode.ts` | 底层媒体操作             |
| 理解     | `src/media-understanding/` | `runtime.ts`, `providers/`, `attachment-normalize.ts`        | image/audio/video → text |
| 图像生成 | `src/image-generation/`    | `runtime.ts`, `provider-registry.ts`, `types.ts`             | text → image             |
| 视频生成 | `src/video-generation/`    | `runtime.ts`, `provider-registry.ts`                         | text → video             |
| 音乐生成 | `src/music-generation/`    | `runtime.ts`, `provider-registry.ts`                         | text → music             |
| 语音合成 | `src/tts/`                 | `runtime.ts`, `provider-registry.ts`                         | text → speech            |
| 实时语音 | `src/talk/`                | `session.ts`, `agent-consult.ts`, `audio-codec.ts`           | 双向语音对话             |
| 链接理解 | `src/link-understanding/`  | `detect.ts`, `format.ts`, `runtime.ts`                       | URL 预览/解析            |
| 代理生成 | `src/media-generation/`    | `catalog.ts`, `normalize.ts`                                 | 统一生成目录             |

---

> 本文档基于 OpenClaw 源码静态分析得出。具体函数级行为请以源码与单元测试为最终参考。
