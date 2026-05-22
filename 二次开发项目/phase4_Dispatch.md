# Phase 4：调度层（Dispatch/Orchestration Layer）详细分析

> 本文档基于 OpenClaw 源码深度分析，梳理调度层的架构、输入输出规范、与 Gateway/Channels/Agent 层的对接关系，以及二次开发的复用性评估。

---

## 一、调度层在整体架构中的位置

```
┌─────────────────────┐     ┌─────────────────────────────────┐
│   Gateway 层        │     │       Channel Plugin 层          │
│  (WebUI / CLI / SDK)│     │ (Telegram / WhatsApp / Slack ..) │
└────────┬────────────┘     └───────────────┬─────────────────┘
         │                                  │
         │  构建 MsgContext                 │  构建 MsgContext
         │                                  │
         ▼                                  ▼
┌══════════════════════════════════════════════════════════════════┐
║                    调度层 (Dispatch Layer)                       ║
║                                                                  ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │         dispatchInboundMessage()  ← 统一入口             │   ║
║  │                    │                                      │   ║
║  │         finalizeInboundContext()  ← 标准化                │   ║
║  │                    │                                      │   ║
║  │         dispatchReplyFromConfig() ← 核心调度引擎          │   ║
║  │           ├─ 入站去重 (inbound-dedupe)                    │   ║
║  │           ├─ 命令检测 (command-detection)                  │   ║
║  │           ├─ 会话解析 (session lookup)                     │   ║
║  │           ├─ Hook 执行 (message_received / inbound_claim) │   ║
║  │           ├─ Agent & 模型选择 (harness selection)          │   ║
║  │           ├─ 队列策略 (queue policy)                       │   ║
║  │           └─ getReplyFromConfig() → runReplyAgent()       │   ║
║  └──────────────────────────────────────────────────────────┘   ║
║                    │                                             ║
║                    ▼                                             ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  Agent 执行 → ReplyPayload → dispatcher.deliver()        │   ║
║  └──────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════╝
         │                                  │
         ▼                                  ▼
   Gateway 回复推送                  Channel outbound.send()
   (WebSocket event)                (平台 API 发送)
```

**核心定位**：调度层是一个**纯粹的中间件层**，它接收标准化的 `MsgContext` 输入，通过配置驱动的策略管线，最终触发 Agent 执行并投递回复。它对消息来源（Gateway 还是 Channel）**完全无感知**。

---

## 二、源码目录结构

调度层的核心代码位于 `src/auto-reply/` 目录下，共 500+ 个文件：

```
src/auto-reply/
├── dispatch.ts                          ← 统一入口函数
├── dispatch-dispatcher.ts               ← dispatcher 包装器
├── templating.ts                        ← MsgContext / FinalizedMsgContext 类型定义
├── inbound-debounce.ts                  ← 入站消息防抖
├── command-detection.ts                 ← 命令检测
├── command-auth.ts                      ← 命令授权
├── command-turn-context.ts              ← 命令上下文
├── commands-registry.ts                 ← 命令注册表
├── heartbeat.ts                         ← 心跳定时任务
├── heartbeat-filter.ts                  ← 心跳过滤
├── model.ts                             ← 模型选择
├── reply-payload.ts                     ← ReplyPayload 类型定义
├── reply.ts                             ← 回复入口
├── tokens.ts                            ← 特殊标记 (HEARTBEAT_TOKEN 等)
├── types.ts                             ← 公共类型
├── reply/
│   ├── dispatch-from-config.ts          ← ★ 核心调度引擎 (2301 行)
│   ├── dispatch-from-config.types.ts    ← 调度参数和结果类型
│   ├── get-reply-run.ts                 ← Agent 回复准备 (1240 行)
│   ├── agent-runner.ts                  ← Agent 执行器 (2225 行)
│   ├── agent-runner-execution.ts        ← Agent 执行细节 (2564 行)
│   ├── reply-dispatcher.ts              ← 回复分发器
│   ├── reply-run-registry.ts            ← 运行注册表
│   ├── queue.ts                         ← 队列导出
│   ├── queue/
│   │   ├── types.ts                     ← 队列类型 (QueueMode / FollowupRun)
│   │   ├── enqueue.ts                   ← 入队
│   │   ├── drain.ts                     ← 出队
│   │   ├── settings-runtime.ts          ← 队列配置解析
│   │   └── ...
│   ├── session.ts                       ← 会话管理
│   ├── routing-policy.ts                ← 路由策略
│   ├── origin-routing.ts                ← 来源路由
│   ├── typing.ts                        ← typing 指示器
│   ├── commands-approve.ts              ← 审批命令
│   ├── block-reply-pipeline.ts          ← 分块回复管线
│   └── ...
└── ...

src/process/
├── command-queue.ts                     ← 命令车道队列
└── lanes.ts                             ← 车道枚举定义
```

---

## 三、统一入口函数：`dispatchInboundMessage()`

> 源码位置：`src/auto-reply/dispatch.ts`，第 246-292 行

这是所有消息（无论来自 Gateway 还是 Channel）进入调度层的统一入口。

### 3.1 函数签名

```typescript
// src/auto-reply/dispatch.ts

export async function dispatchInboundMessage(params: {
  ctx: MsgContext | FinalizedMsgContext; // 入站消息上下文
  cfg: OpenClawConfig; // 全局配置
  dispatcher: ReplyDispatcher; // 回复分发器
  replyOptions?: Omit<GetReplyOptions, "onBlockReply">;
  replyResolver?: GetReplyFromConfig;
}): Promise<DispatchInboundResult> {
  // 1. 标准化上下文
  const finalized = finalizeInboundContext(params.ctx);

  // 2. 调用核心调度引擎
  const result = await withReplyDispatcher({
    dispatcher: params.dispatcher,
    run: () =>
      dispatchReplyFromConfig({
        ctx: finalized,
        cfg: params.cfg,
        dispatcher: params.dispatcher,
        replyOptions: params.replyOptions,
        replyResolver: params.replyResolver,
      }),
  });

  return finalizeDispatchResult(result, params.dispatcher);
}
```

### 3.2 带缓冲分发器的入口

对于需要 typing 指示和消息防抖的场景（Channel 通常使用此变体）：

```typescript
// src/auto-reply/dispatch.ts

export async function dispatchInboundMessageWithBufferedDispatcher(params: {
  ctx: MsgContext | FinalizedMsgContext;
  cfg: OpenClawConfig;
  dispatcherOptions: ReplyDispatcherWithTypingOptions;
  replyOptions?: Omit<GetReplyOptions, "onBlockReply">;
  replyResolver?: GetReplyFromConfig;
}): Promise<DispatchInboundResult> {
  const finalized = finalizeInboundContext(params.ctx);

  // 前台回复防护（同一会话/目标的并发调度保护）
  const foregroundReplyFence = beginForegroundReplyFence(finalized);

  // 创建带 typing 指示的 dispatcher
  const { dispatcher, replyOptions, markDispatchIdle, markRunComplete } =
    createReplyDispatcherWithTyping({ ...params.dispatcherOptions });

  try {
    return await dispatchInboundMessage({
      ctx: finalized,
      cfg: params.cfg,
      dispatcher,
      replyResolver: params.replyResolver,
      replyOptions: { ...params.replyOptions, ...replyOptions },
    });
  } finally {
    endForegroundReplyFence(foregroundReplyFence);
    markRunComplete();
    markDispatchIdle();
  }
}
```

### 3.3 Gateway 和 Channels 如何调用此入口

**Gateway 层调用方式**（`src/gateway/server-methods/chat.ts`）：

```typescript
import { dispatchInboundMessage } from "../../auto-reply/dispatch.js";

// 在 chat.send RPC handler 中：
const msgContext: MsgContext = {
  Body: message.text,
  From: clientInfo.deviceId,
  SessionKey: sessionKey,
  Provider: "gateway",
  Surface: clientInfo.clientName,
  // ...
};
const dispatcher = createReplyDispatcher({ deliver: (payload) => { ... } });
await dispatchInboundMessage({ ctx: msgContext, cfg, dispatcher });
```

**Telegram Channel 调用方式**（`extensions/telegram/src/bot-message-dispatch.ts`）：

```typescript
import { runInboundReplyTurn } from "openclaw/plugin-sdk/inbound-reply-dispatch";

// runInboundReplyTurn 内部封装了 dispatchReplyFromConfig
await runInboundReplyTurn({
  rawMessage: telegramUpdate,
  buildInboundContext: (raw) => ({
    Body: raw.text,
    From: String(raw.from.id),
    To: String(raw.chat.id),
    Provider: "telegram",
    Surface: "telegram",
    ChatType: raw.chat.type === "private" ? "direct" : "group",
    // ... 转换为标准 MsgContext
  }),
  dispatch: async ({ ctx, cfg }) => dispatchReplyFromConfig({ ctx, cfg, dispatcher }),
  deliver: async (payload) => {
    await sendMessageTelegram(chatId, payload.text, { cfg });
  },
});
```

**WhatsApp Channel 调用方式**（`extensions/whatsapp/src/auto-reply/monitor/inbound-dispatch.ts`）：

```typescript
import {
  dispatchReplyWithBufferedBlockDispatcher,
  finalizeInboundContext,
} from "./inbound-dispatch.runtime.js";

// 同样通过标准 MsgContext → dispatch 管线
await dispatchReplyWithBufferedBlockDispatcher({
  ctx: msgContext, // 标准 MsgContext
  cfg: loadedConfig,
  dispatcherOptions: {
    deliver: async (payload) => {
      await sendWhatsAppMessage(jid, payload);
    },
  },
});
```

**结论：三个来源（Gateway、Telegram、WhatsApp）最终都调用同一个调度管线，输入格式完全一致。**

---

## 四、调度参数与结果类型

> 源码位置：`src/auto-reply/reply/dispatch-from-config.types.ts`

### 4.1 调度输入参数

```typescript
export type DispatchFromConfigParams = {
  ctx: FinalizedMsgContext; // 标准化后的入站消息上下文
  cfg: OpenClawConfig; // 全局配置对象
  dispatcher: ReplyDispatcher; // 回复分发器（由调用方提供 deliver 回调）
  replyOptions?: Omit<GetReplyOptions, "onBlockReply">;
  replyResolver?: GetReplyFromConfig;
  fastAbortResolver?: TryFastAbortFromMessage;
  formatAbortReplyTextResolver?: FormatAbortReplyText;
  configOverride?: OpenClawConfig; // 可选的配置覆盖补丁
};
```

### 4.2 调度输出结果

```typescript
export type DispatchFromConfigResult = {
  queuedFinal: boolean; // 是否有最终回复被投递
  counts: Record<ReplyDispatchKind, number>; // 各类回复计数
  failedCounts?: Partial<Record<ReplyDispatchKind, number>>;
  sourceReplyDeliveryMode?: SourceReplyDeliveryMode;
  beforeAgentRunBlocked?: boolean;
};

// ReplyDispatchKind:
type ReplyDispatchKind = "tool" | "block" | "final";
// tool  — Agent 工具调用中间结果
// block — 流式分块输出
// final — 最终完整回复
```

### 4.3 ReplyDispatcher 接口

调度层通过 `ReplyDispatcher` 的回调将 Agent 回复投递出去，不关心投递目标是 Gateway 还是 Channel：

```typescript
type ReplyDispatcher = {
  sendToolReply(payload: ReplyPayload): boolean;
  sendBlockReply(payload: ReplyPayload): boolean;
  sendFinalReply(payload: ReplyPayload): boolean;
  getQueuedCounts(): Record<ReplyDispatchKind, number>;
  getCancelledCounts?(): Partial<Record<ReplyDispatchKind, number>>;
  getFailedCounts?(): Partial<Record<ReplyDispatchKind, number>>;
};
```

---

## 五、核心调度引擎：`dispatchReplyFromConfig()`

> 源码位置：`src/auto-reply/reply/dispatch-from-config.ts`（2301 行）

这是调度层最核心的函数，包含完整的消息处理管线。

### 5.1 执行流程

```
dispatchReplyFromConfig(ctx, cfg, dispatcher)
│
├─ 1. 入站去重 (inbound-dedupe)
│     检查 MessageSid 是否已处理过，避免重复分发
│
├─ 2. 会话查找 (session store lookup)
│     根据 SessionKey 查找/创建会话记录
│     解析 agentId、storePath、sessionEntry
│
├─ 3. 路由决策 (routing policy)
│     根据 OriginatingChannel / OriginatingTo 决定回复路由
│     处理 source-reply delivery mode
│
├─ 4. Hook 执行
│     ├─ message_received hook（消息到达通知）
│     ├─ inbound_claim hook（插件声明处理权）
│     └─ message_sending hook（回复发送前拦截）
│
├─ 5. 命令检测 (command detection)
│     检查是否是控制命令（/new, /reset, /model, /help 等）
│     如果是命令，执行命令处理器并返回
│
├─ 6. Agent & 模型选择
│     ├─ resolveDefaultModelForAgent() → 默认模型
│     ├─ resolveChannelModelCandidate() → Channel 级模型覆盖
│     ├─ resolveStoredModelCandidate() → 会话级模型覆盖
│     ├─ selectAgentHarness() → 选择执行引擎
│     └─ 构建完整的 Agent 执行上下文
│
├─ 7. 队列管理
│     ├─ resolveQueueSettings() → 读取队列配置
│     ├─ resolveActiveRunQueueAction() → 决定队列动作
│     │   (steer / followup / collect / interrupt)
│     └─ enqueueFollowupRun() 或直接执行
│
├─ 8. Agent 执行
│     getReplyFromConfig()
│       └─ runReplyAgent()
│             └─ runAgentTurnWithFallback()
│                   └─ runEmbeddedPiAgent()  ← 实际 Agent 执行
│
├─ 9. 回复处理
│     ├─ TTS 语音合成（如果配置了）
│     ├─ 媒体路径标准化
│     ├─ 回复格式化（Markdown 表格模式等）
│     └─ dispatcher.sendFinalReply(payload) 或 routeReplyToOriginating()
│
└─ 10. 收尾
      ├─ 更新会话状态
      ├─ 提交入站去重标记
      └─ 返回 DispatchFromConfigResult
```

### 5.2 关键代码路径

**入站去重**（防止消息重复处理）：

```typescript
// dispatch-from-config.ts
const inboundDedupeClaim = claimInboundDedupe(dedupeKey);
if (inboundDedupeClaim.status === "duplicate") {
  // 跳过已处理的消息
  return { queuedFinal: false, counts: { tool: 0, block: 0, final: 0 } };
}
```

**Agent 与模型选择**：

```typescript
// dispatch-from-config.ts
const defaultModelRef = resolveDefaultModelForAgent({ cfg, agentId });
const aliasIndex = buildModelAliasIndex({ cfg, defaultProvider });

// 优先级：turn 覆盖 > 会话存储覆盖 > channel 覆盖 > 全局默认
const turnOverride = resolveTurnModelOverride(replyOptions);
const storedOverride = resolveStoredModelCandidate({ ... });
const channelOverride = resolveChannelModelCandidate({ ... });

const harness = selectAgentHarness({ cfg, provider, model, agentId });
```

**回复路由**：

```typescript
// dispatch-from-config.ts
// 根据 OriginatingChannel 决定回复是否需要路由到特定 Channel
const routeReplyChannel = ctx.OriginatingChannel;
const routeReplyTo = ctx.OriginatingTo;

if (shouldRouteToOriginating) {
  await routeReplyToOriginating(payload);
} else {
  dispatcher.sendFinalReply(payload);
}
```

---

## 六、四大子系统详解

### 6.1 自动回复调度

这是调度层的主体功能，由 `dispatchReplyFromConfig()` 实现。

核心职责：

- 消息标准化和去重
- 会话生命周期管理（创建/恢复/重置/fork）
- 命令检测与执行（`/new`、`/reset`、`/model`、`/help` 等 30+ 个命令）
- Agent 选择与执行
- 回复投递与路由

### 6.2 车道队列（Command Lanes）

> 源码位置：`src/process/command-queue.ts` + `src/process/lanes.ts`

车道队列用于**串行化命令执行**，防止并发冲突。

**车道定义**：

```typescript
// src/process/lanes.ts
export const enum CommandLane {
  Main = "main", // 主消息处理车道
  Cron = "cron", // 定时任务车道
  CronNested = "cron-nested", // 定时任务嵌套车道
  Subagent = "subagent", // 子 Agent 车道
  Nested = "nested", // 嵌套执行车道
}
```

**队列模式**：

```typescript
// src/auto-reply/reply/queue/types.ts
export type QueueMode = "steer" | "followup" | "collect" | "interrupt";

export type QueueSettings = {
  mode: QueueMode; // 队列模式
  debounceMs?: number; // 防抖延迟
  cap?: number; // 队列上限
  dropPolicy?: QueueDropPolicy; // 丢弃策略
};

export type QueueDropPolicy = "old" | "new" | "summarize";
```

| 模式        | 行为                              | 使用场景                 |
| ----------- | --------------------------------- | ------------------------ |
| `steer`     | 新消息替换队列中等待的消息        | 默认模式，适合 1:1 对话  |
| `followup`  | 新消息排队依次执行                | 需要保证每条消息都被处理 |
| `collect`   | 收集多条消息合并为一次 Agent 调用 | 减少 API 调用次数        |
| `interrupt` | 中断当前 run，立即执行新消息      | 会话重置后的首条消息     |

**FollowupRun 数据结构**（队列中的待处理项）：

```typescript
// src/auto-reply/reply/queue/types.ts
export type FollowupRun = {
  prompt: string; // 消息文本
  transcriptPrompt?: string; // 用于会话记录的文本
  abortSignal?: AbortSignal; // 取消信号
  messageId?: string; // 用于去重的消息 ID
  enqueuedAt: number; // 入队时间戳
  originatingChannel?: OriginatingChannelType; // 来源 Channel（用于回复路由）
  originatingTo?: string; // 来源目标
  images?: Array<{ type; data; mimeType }>; // 图片附件
  // ...
};
```

### 6.3 心跳定时任务（Heartbeat）

> 源码位置：`src/auto-reply/heartbeat.ts`

心跳系统定期唤醒 Agent 执行周期性检查。

**核心配置**：

```typescript
// src/auto-reply/heartbeat.ts
export const DEFAULT_HEARTBEAT_EVERY = "30m"; // 默认每 30 分钟
export const DEFAULT_HEARTBEAT_ACK_MAX_CHARS = 300; // 回复最大字符数

export const HEARTBEAT_PROMPT =
  "Read HEARTBEAT.md if it exists (workspace context). " +
  "Follow it strictly. Do not infer or repeat old tasks from prior chats. " +
  "If nothing needs attention, reply HEARTBEAT_OK.";
```

**工作机制**：

```
定时器触发（每 N 分钟）
    │
    ▼
检查 HEARTBEAT.md 是否有内容
    │  (isHeartbeatContentEffectivelyEmpty)
    │
    ├─ 空/无任务 → 跳过，不调用 API
    │
    └─ 有任务内容 → 构建 MsgContext
         │
         ├─ Body = HEARTBEAT_PROMPT
         ├─ isHeartbeat = true
         ├─ 可选：heartbeatModelOverride（独立模型）
         │
         ▼
    进入标准 dispatchInboundMessage() 流程
         │
         ▼
    Agent 执行 → 检查 HEARTBEAT.md → 回复
         │
         ├─ "HEARTBEAT_OK" → 无需通知用户
         └─ 有实质内容 → 通知用户
```

**Heartbeat 任务类型定义**：

```typescript
export type HeartbeatTask = {
  name: string; // 任务名称
  interval: string; // 间隔（如 "30m", "1h"）
  prompt: string; // 自定义提示词
};
```

**关键设计**：心跳也走标准的 `dispatchInboundMessage()` 管线，与普通消息完全相同的处理流程，只是 `replyOptions.isHeartbeat = true` 作为标记。

### 6.4 审批循环（Approval Loop）

> 源码位置：`src/auto-reply/reply/commands-approve.ts` + `src/infra/exec-approvals.ts`

当 Agent 需要执行敏感操作（如 shell 命令、文件写入）时，审批循环会暂停执行并请求用户确认。

**审批流程**：

```
Agent 调用敏感工具（如 exec）
    │
    ▼
exec-approvals 检查是否需要审批
    │
    ├─ 自动批准（配置允许）→ 继续执行
    │
    └─ 需要审批 → 暂停 Agent 执行
         │
         ├─ 向用户发送审批请求
         │   (通过 Channel 的 approvalCapability 适配器)
         │
         ├─ 等待用户响应
         │   /approve 或 /deny 命令
         │
         └─ 用户响应后恢复/终止 Agent 执行
```

**审批范围配置**：

```typescript
// src/auto-reply/reply/agent-runner-execution.ts
function readApprovalScopeValue(value: unknown): "turn" | "session" | undefined {
  return value === "turn" || value === "session" ? value : undefined;
}
// "turn"    — 每次工具调用都需要审批
// "session" — 同一会话中审批一次后自动通过
```

---

## 七、入站消息防抖（Inbound Debounce）

> 源码位置：`src/auto-reply/inbound-debounce.ts`

当用户快速连续发送多条消息时，防抖机制会将它们合并为一次处理。

```typescript
// src/auto-reply/inbound-debounce.ts
export function resolveInboundDebounceMs(params: {
  cfg: OpenClawConfig;
  channel: string; // Channel 标识
  overrideMs?: number; // 手动覆盖值
}): number {
  const inbound = params.cfg.messages?.inbound;
  const override = resolveMs(params.overrideMs);
  const byChannel = resolveChannelOverride({
    byChannel: inbound?.byChannel,
    channel: params.channel,
  });
  const base = resolveMs(inbound?.debounceMs);
  // 优先级：手动覆盖 > Channel 级配置 > 全局配置 > 默认 0
  return override ?? byChannel ?? base ?? 0;
}
```

**配置示例**：

```json
{
  "messages": {
    "inbound": {
      "debounceMs": 1500,
      "byChannel": {
        "telegram": 2000,
        "whatsapp": 1000
      }
    }
  }
}
```

---

## 八、调度层与 Gateway/Channels 的对接证明

### 8.1 Gateway 调用路径

> 源码位置：`src/gateway/server-methods/chat.ts`

```
WebSocket RPC: chat.send
    │
    ▼
chat.ts handler
    │  构建 MsgContext {
    │    Body, From, SessionKey,
    │    Provider: "gateway",
    │    Surface: clientInfo.clientName,
    │    ...
    │  }
    │
    ▼
dispatchInboundMessage({ ctx, cfg, dispatcher })
    │
    ▼
标准调度管线 → Agent 执行 → ReplyPayload
    │
    ▼
dispatcher.deliver(payload)
    │
    ▼
Gateway WebSocket 推送 agent 事件给客户端
```

### 8.2 Channel 调用路径（以 Telegram 为例）

> 源码位置：`extensions/telegram/src/bot-message-dispatch.ts`

```
Telegram Bot API polling/webhook → 收到 Update
    │
    ▼
bot-message-dispatch.ts
    │  将 Telegram 消息转换为 MsgContext {
    │    Body: update.message.text,
    │    From: String(update.message.from.id),
    │    To: String(update.message.chat.id),
    │    Provider: "telegram",
    │    Surface: "telegram",
    │    ChatType: "direct" | "group",
    │    ...
    │  }
    │
    ▼
runInboundReplyTurn({ ... })  // Plugin SDK 封装
    │  内部调用 dispatchReplyFromConfig()
    │
    ▼
标准调度管线 → Agent 执行 → ReplyPayload
    │
    ▼
deliver(payload)
    │
    ▼
sendMessageTelegram(chatId, payload.text, { cfg })
```

### 8.3 Channel 调用路径（以 WhatsApp 为例）

> 源码位置：`extensions/whatsapp/src/auto-reply/monitor/inbound-dispatch.ts`

```
Baileys WebSocket → 收到消息事件
    │
    ▼
inbound-dispatch.ts
    │  将 WhatsApp 消息转换为 MsgContext {
    │    Body: msg.message.conversation,
    │    From: msg.key.remoteJid,
    │    Provider: "whatsapp",
    │    Surface: "whatsapp",
    │    ...
    │  }
    │
    ▼
dispatchReplyWithBufferedBlockDispatcher({ ctx, cfg, ... })
    │  内部调用 dispatchReplyFromConfig()
    │
    ▼
标准调度管线 → Agent 执行 → ReplyPayload
    │
    ▼
deliver(payload)
    │
    ▼
sendWhatsAppMessage(jid, payload)
```

**三条路径的调度入口完全相同**。差异仅在于：

1. `MsgContext` 的字段值不同（`Provider`、`Surface`、`ChatType` 等）
2. `dispatcher.deliver()` 的实现不同（Gateway 推送 WebSocket / Telegram 调用 Bot API / WhatsApp 调用 Baileys）

---

## 九、模型选择机制

调度层内置了多级模型选择策略，所有策略都是配置驱动的。

### 9.1 模型选择优先级

```
优先级从高到低：
1. Turn-level override     → replyOptions.modelOverride（单次覆盖）
2. Session stored override → 会话中 /model 命令切换的模型
3. Channel model override  → channels.modelByChannel 配置
4. Agent default model     → agents.defaults.model 配置
```

### 9.2 模型 Fallback 机制

```typescript
// src/auto-reply/reply/agent-runner-execution.ts
import { runWithModelFallback } from "../../agents/model-fallback.js";

// 当主模型不可用时，自动尝试备用模型
const result = await runWithModelFallback({
  primaryProvider, primaryModel,
  fallbackOptions: resolveModelFallbackOptions(cfg),
  run: (provider, model) => runEmbeddedPiAgent({ provider, model, ... }),
});
```

### 9.3 配置示例

```json
{
  "agents": {
    "defaults": {
      "provider": "openai",
      "model": "gpt-4o"
    }
  },
  "channels": {
    "modelByChannel": {
      "telegram": "anthropic/claude-sonnet-4-20250514",
      "whatsapp": "openai/gpt-4o-mini"
    }
  }
}
```

---

## 十、Plugin SDK 提供的调度封装

> 源码位置：`src/plugin-sdk/inbound-reply-dispatch.ts`

Plugin SDK 为 Channel 插件提供了更高层次的封装，简化调度对接。

### 10.1 `runInboundReplyTurn()`

这是大多数内置 Channel 使用的入口：

```typescript
// src/plugin-sdk/inbound-reply-dispatch.ts

export async function runInboundReplyTurn<TRaw, TDispatchResult>(
  params: RunChannelTurnParams<TRaw, TDispatchResult>,
): Promise<ChannelTurnResult<TDispatchResult>> {
  return await runPreparedChannelTurn(params);
}
```

它封装了：

- 入站消息会话记录
- Bot 循环检测保护
- 消息确认（ACK）策略
- `dispatchReplyFromConfig()` 调用
- 回复投递和错误处理

### 10.2 `dispatchReplyWithBufferedBlockDispatcher()`

这是外部 Channel 插件（通过 `channelRuntime`）使用的入口：

```typescript
// 通过 ctx.channelRuntime.reply 访问
await ctx.channelRuntime.reply.dispatchReplyWithBufferedBlockDispatcher({
  ctx: msgContext,
  cfg: config,
  dispatcherOptions: {
    deliver: async (payload) => {
      /* 发送回复 */
    },
  },
});
```

### 10.3 `hasVisibleInboundReplyDispatch()`

判断调度结果是否有可见回复：

```typescript
import { hasVisibleInboundReplyDispatch } from "openclaw/plugin-sdk/inbound-reply-dispatch";

const result = await runInboundReplyTurn({ ... });
if (hasVisibleInboundReplyDispatch(result)) {
  // 有回复发出
}
```

---

## 十一、对二次开发的影响评估

### 11.1 开发自定义 Channel 时

| 维度            | 是否涉及调度层 | 说明                                       |
| --------------- | -------------- | ------------------------------------------ |
| 消息接收与转换  | **不涉及**     | Channel 内部实现，与调度层无关             |
| 构建 MsgContext | **声明式涉及** | 需要正确填充字段，但不需要理解调度内部逻辑 |
| 调用调度入口    | **模板化调用** | 使用 SDK 封装，一行代码完成                |
| 命令处理        | **不涉及**     | 调度层自动处理所有内置命令                 |
| 会话管理        | **不涉及**     | 调度层自动创建/恢复/管理会话               |
| 队列策略        | **不涉及**     | 由配置驱动，对 Channel 透明                |
| 模型选择        | **不涉及**     | 由配置驱动，对 Channel 透明                |
| 回复投递        | **回调式涉及** | 需要提供 `deliver` 回调函数                |

### 11.2 接入不同模型时

| 维度                  | 是否影响调度层   | 说明                                 |
| --------------------- | ---------------- | ------------------------------------ |
| 切换模型提供商        | **不影响**       | 修改 `openclaw.json` 配置即可        |
| 新增模型提供商        | **不影响调度层** | 在 `extensions/` 新增提供商插件      |
| 按 Channel 配不同模型 | **不影响**       | 配置 `channels.modelByChannel`       |
| 模型 Fallback         | **不影响**       | 配置 `agents.defaults.modelFallback` |
| 调整上下文窗口        | **不影响**       | 配置 `agents.defaults.contextTokens` |

### 11.3 复用性评估

**调度层代码 100% 可复用**。原因：

1. **输入标准化**：所有消息源统一为 `MsgContext`，调度层对来源无感知
2. **配置驱动**：队列策略、模型选择、命令行为等全部由 `openclaw.json` 配置控制
3. **回调式输出**：回复通过 `ReplyDispatcher.deliver()` 回调投递，调度层不关心投递目标
4. **无硬编码依赖**：调度层代码中没有任何特定 Channel 或特定模型的硬编码逻辑

---

## 十二、总结

### 12.1 调度层的核心设计原则

1. **统一入口**：`dispatchInboundMessage()` 是唯一的消息处理入口，Gateway 和所有 Channel 使用相同接口
2. **配置驱动**：所有策略行为（队列模式、模型选择、命令、审批）均由配置控制，无需修改代码
3. **回调投递**：回复通过调用方提供的 `deliver` 回调发出，调度层不关心目标平台
4. **中立中间件**：对消息来源、投递目标、底层模型完全无感知

### 12.2 自定义 Channel 的接入姿势

```typescript
// 自定义 Channel 只需要做两件事：

// 1. 构建 MsgContext
const msgContext: MsgContext = {
  Body: platformMessage.text,
  From: platformMessage.senderId,
  To: platformMessage.chatId,
  SessionKey: `my-channel:${platformMessage.chatId}`,
  Provider: "my-channel",
  Surface: "my-channel",
  OriginatingChannel: "my-channel",
  OriginatingTo: platformMessage.chatId,
  ChatType: platformMessage.isGroup ? "group" : "direct",
};

// 2. 调用调度入口，提供 deliver 回调
await ctx.channelRuntime.reply.dispatchReplyWithBufferedBlockDispatcher({
  ctx: msgContext,
  cfg: config,
  dispatcherOptions: {
    deliver: async (payload) => {
      await myPlatformApi.sendMessage(chatId, payload.text);
    },
  },
});
// 其他所有事情（命令、会话、队列、Agent、模型）由调度层自动处理
```

### 12.3 数据流全景

```
  ┌──────────┐   ┌───────────┐   ┌───────────┐
  │ Gateway  │   │ Telegram  │   │ WhatsApp  │   ... 更多来源
  │ (WebUI)  │   │ Channel   │   │ Channel   │
  └────┬─────┘   └─────┬─────┘   └─────┬─────┘
       │               │               │
       │  MsgContext    │  MsgContext    │  MsgContext
       └───────────────┼───────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ dispatchInbound │  ← 统一入口
              │   Message()     │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │ dispatchReply   │  ← 核心调度
              │ FromConfig()    │
              │ ┌─────────────┐ │
              │ │ 去重/命令/  │ │
              │ │ 会话/Hook/  │ │
              │ │ 队列/模型   │ │
              │ └──────┬──────┘ │
              └────────┼────────┘
                       │
              ┌────────▼────────┐
              │ runReplyAgent() │  ← Agent 执行
              │ runEmbeddedPi() │
              └────────┬────────┘
                       │
                  ReplyPayload
                       │
              dispatcher.deliver()
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
  Gateway 推送    Telegram API    WhatsApp API
  WebSocket       sendMessage     sendMessage
```
