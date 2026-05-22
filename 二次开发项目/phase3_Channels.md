# Phase 3：Channel 层详细分析 — 统一接口规范与自定义 Channel 接入指南

> 本文档基于 OpenClaw 源码深度分析，梳理 Channel 层的完整接口规范，为接入自定义 Channel 提供详细的开发指引。

---

## 一、Channel 层在整体架构中的位置

```
┌──────────────────────────────────────────────────────────────────┐
│  外部消息平台 (Telegram / WhatsApp / Slack / Discord / ...)     │
└──────────────────────┬───────────────────────────────────────────┘
                       │ 平台原生协议 (HTTP Webhook / WebSocket / Polling)
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Channel Plugin Layer                          │
│  ┌────────┐ ┌──────────┐ ┌───────┐ ┌─────────┐ ┌────────────┐  │
│  │Telegram│ │ WhatsApp │ │ Slack │ │ Discord │ │ 自定义 ... │  │
│  └───┬────┘ └────┬─────┘ └───┬───┘ └────┬────┘ └─────┬──────┘  │
│      │           │           │          │             │          │
│      └───────────┴───────────┴──────────┴─────────────┘          │
│                       统一接口                                   │
│                  ChannelPlugin 接口                              │
└──────────────────────┬───────────────────────────────────────────┘
                       │ MsgContext (入站) / ChannelOutboundContext (出站)
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│              Dispatch / Agent 执行层                             │
│  dispatchInboundMessage() → Agent Runtime → ReplyPayload        │
└──────────────────────────────────────────────────────────────────┘
```

**核心设计哲学**：每个 Channel 是一个独立的插件（Extension），通过实现统一的 `ChannelPlugin` 接口来接入系统。不同 Channel 的内部实现可以完全不同，但对外暴露的接口完全一致。

---

## 二、Channel Plugin 的文件组织结构

每个 Channel 插件遵循统一的目录结构，位于 `extensions/<channel-id>/`：

```
extensions/
└── my-channel/                        ← 插件根目录
    ├── openclaw.plugin.json           ← 插件清单文件（必需）
    ├── index.ts                       ← 入口文件（必需）
    ├── package.json                   ← 包定义
    └── src/
        ├── channel.ts                 ← ChannelPlugin 对象定义（必需）
        ├── accounts.ts                ← 账户解析逻辑
        ├── outbound-adapter.ts        ← 出站消息适配器
        ├── normalize.ts               ← 目标地址归一化
        ├── send.ts                    ← 发送消息实现
        ├── runtime.ts                 ← 运行时依赖注入
        ├── shared.ts                  ← 共享工具函数
        └── ...                        ← 其他特定功能模块
```

**源码参考**：以 Telegram 为例

```
extensions/telegram/
├── openclaw.plugin.json               ← 15 行，声明 channel id 和环境变量
├── index.ts                           ← 24 行，defineBundledChannelEntry()
└── src/
    ├── channel.ts                     ← 1135 行，telegramPlugin 定义
    ├── outbound-adapter.ts            ← 出站适配器
    ├── send.ts                        ← Telegram Bot API 发送
    ├── bot-message-dispatch.ts        ← 入站消息分发
    ├── polling-session.ts             ← 长轮询实现
    ├── webhook.ts                     ← Webhook 接收
    └── ... (约 295 个源文件)
```

---

## 三、插件清单文件：`openclaw.plugin.json`

### 3.1 基本格式

这是系统识别和加载 Channel 插件的入口声明。

```json
{
  "id": "my-channel",
  "activation": {
    "onStartup": false
  },
  "channels": ["my-channel"],
  "channelEnvVars": {
    "my-channel": ["MY_CHANNEL_API_TOKEN", "MY_CHANNEL_SECRET"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

### 3.2 字段说明

| 字段                   | 类型                       | 必填 | 说明                          |
| ---------------------- | -------------------------- | ---- | ----------------------------- |
| `id`                   | `string`                   | 是   | 插件唯一标识符，与目录名一致  |
| `activation.onStartup` | `boolean`                  | 是   | 是否在系统启动时自动激活      |
| `channels`             | `string[]`                 | 是   | 该插件注册的 channel id 列表  |
| `channelEnvVars`       | `Record<string, string[]>` | 否   | 每个 channel 关联的环境变量名 |
| `configSchema`         | `JSON Schema`              | 否   | 该 channel 的配置校验 schema  |
| `contracts.tools`      | `string[]`                 | 否   | 该插件提供的 Agent 工具列表   |
| `toolMetadata`         | `object`                   | 否   | Agent 工具的配置信号          |
| `skills`               | `string[]`                 | 否   | 插件提供的技能目录            |

### 3.3 现有 Channel 示例对比

**Telegram** — 最简形式：

```json
{
  "id": "telegram",
  "activation": { "onStartup": false },
  "channels": ["telegram"],
  "channelEnvVars": { "telegram": ["TELEGRAM_BOT_TOKEN"] }
}
```

**Slack** — 多环境变量：

```json
{
  "id": "slack",
  "activation": { "onStartup": false },
  "channels": ["slack"],
  "channelEnvVars": { "slack": ["SLACK_BOT_TOKEN", "SLACK_APP_TOKEN", "SLACK_USER_TOKEN"] }
}
```

**飞书** — 复杂配置 + 自定义工具：

```json
{
  "id": "feishu",
  "channels": ["feishu"],
  "contracts": { "tools": ["feishu_chat", "feishu_doc", ...] },
  "channelEnvVars": { "feishu": ["FEISHU_APP_ID", "FEISHU_APP_SECRET", ...] },
  "configSchema": {
    "properties": {
      "appId": { "type": "string" },
      "connectionMode": { "enum": ["websocket", "webhook"] },
      "accounts": { "type": "object", "additionalProperties": { "$ref": "#/$defs/account" } }
    }
  }
}
```

---

## 四、入口文件：`index.ts`

### 4.1 标准格式

所有 Channel 插件的入口文件都使用 `defineBundledChannelEntry()` 工厂函数：

> 源码位置：`src/plugin-sdk/channel-entry-contract.ts`

```typescript
import { defineBundledChannelEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelEntry({
  id: "my-channel", // 必填：channel 唯一 ID
  name: "My Channel", // 必填：显示名称
  description: "My custom channel plugin", // 必填：描述
  importMetaUrl: import.meta.url, // 必填：模块 URL（用于路径解析）

  // 必填：ChannelPlugin 对象的导入引用
  plugin: {
    specifier: "./channel-plugin-api.js", // 模块路径（相对于 index.ts）
    exportName: "myChannelPlugin", // 导出名称
  },

  // 可选：独立的出站适配器
  outbound: {
    specifier: "./outbound-api.js",
    exportName: "myChannelOutbound",
  },

  // 可选：密钥管理
  secrets: {
    specifier: "./secret-contract-api.js",
    exportName: "channelSecrets",
  },

  // 可选：运行时注入器
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },

  // 可选：账户检查
  accountInspect: {
    specifier: "./account-inspect-api.js",
    exportName: "inspectMyChannelAccount",
  },
});
```

### 4.2 `defineBundledChannelEntry` 完整参数类型

```typescript
// 源码：src/plugin-sdk/channel-entry-contract.ts，第 50-64 行

type DefineBundledChannelEntryOptions<TPlugin> = {
  id: string; // 插件 ID
  name: string; // 显示名称
  description: string; // 描述
  importMetaUrl: string; // import.meta.url
  plugin: BundledEntryModuleRef; // ChannelPlugin 引用（必填）
  outbound?: BundledEntryModuleRef; // 出站适配器引用
  secrets?: BundledEntryModuleRef; // 密钥适配器引用
  configSchema?: ChannelConfigSchema | (() => Schema); // 配置 Schema
  runtime?: BundledEntryModuleRef; // 运行时注入引用
  accountInspect?: BundledEntryModuleRef; // 账户检查引用
  features?: BundledChannelEntryFeatures; // 特性标志
  registerCliMetadata?: (api: OpenClawPluginApi) => void; // CLI 元数据注册
  registerFull?: (api: OpenClawPluginApi) => void; // 完整注册回调
};

type BundledEntryModuleRef = {
  specifier: string; // 模块文件路径（相对路径）
  exportName?: string; // 具名导出的名称
};
```

### 4.3 注册流程

`defineBundledChannelEntry()` 返回的对象包含一个 `register(api)` 方法，系统启动时按如下流程调用：

```
系统启动 → 扫描 extensions/ 目录
         → 读取每个 openclaw.plugin.json
         → 加载对应 index.ts 的 default 导出
         → 调用 register(api) 方法
         → api.registerChannel({ plugin: channelPlugin })
         → Channel 注册到全局插件注册表
```

---

## 五、核心接口：`ChannelPlugin` 类型

> 源码位置：`src/channels/plugins/types.plugin.ts`，第 61-106 行

这是所有 Channel 插件必须实现的核心类型契约。

### 5.1 完整类型定义

```typescript
export type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  // ━━━━━━━━━━━━━━━━━━━━━ 必填字段 ━━━━━━━━━━━━━━━━━━━━━
  id: ChannelId; // Channel 唯一标识
  meta: ChannelMeta; // 元数据（显示名称、文档路径等）
  capabilities: ChannelCapabilities; // 静态能力声明
  config: ChannelConfigAdapter<ResolvedAccount>; // 配置适配器（必填）

  // ━━━━━━━━━━━━━━━━━━━━━ 可选字段 ━━━━━━━━━━━━━━━━━━━━━
  // 消息收发
  outbound?: ChannelOutboundAdapter; // 出站消息适配器
  messaging?: ChannelMessagingAdapter; // 消息路由/目标解析
  message?: ChannelMessageAdapterShape; // 消息处理能力声明
  streaming?: ChannelStreamingAdapter; // 流式输出配置
  threading?: ChannelThreadingAdapter; // 线程/话题支持

  // 生命周期与运行时
  gateway?: ChannelGatewayAdapter; // Gateway 生命周期（启动/停止账户）
  lifecycle?: ChannelLifecycleAdapter; // 配置变更/账户移除回调
  heartbeat?: ChannelHeartbeatAdapter; // 心跳/typing 指示

  // 认证与安全
  auth?: ChannelAuthAdapter; // 登录/登出
  pairing?: ChannelPairingAdapter; // 设备配对
  security?: ChannelSecurityAdapter; // DM 安全策略
  allowlist?: ChannelAllowlistAdapter; // 白名单管理
  secrets?: ChannelSecretsAdapter; // 密钥管理

  // 配置与状态
  configSchema?: ChannelConfigSchema; // 运行时配置验证 schema
  setup?: ChannelSetupAdapter; // 初始配置向导
  setupWizard?: ChannelSetupWizard; // UI 配置向导
  status?: ChannelStatusAdapter; // 状态/健康检查
  doctor?: ChannelDoctorAdapter; // 配置修复

  // 群组与目录
  groups?: ChannelGroupAdapter; // 群组策略
  mentions?: ChannelMentionAdapter; // @提及处理
  directory?: ChannelDirectoryAdapter; // 通讯录/联系人
  resolver?: ChannelResolverAdapter; // 目标解析

  // Agent 集成
  agentPrompt?: ChannelAgentPromptAdapter; // Agent 提示词定制
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[]; // Channel 专属工具
  actions?: ChannelMessageActionAdapter; // 消息动作（react、发送等）

  // Gateway 扩展
  gatewayMethods?: string[]; // 自定义 Gateway RPC 方法名
  gatewayMethodDescriptors?: ChannelGatewayMethodDescriptor[]; // 方法描述
  approvalCapability?: ChannelApprovalCapability; // 审批能力
  elevated?: ChannelElevatedAdapter; // 提权适配器

  // 会话绑定
  bindings?: ChannelConfiguredBindingProvider; // 配置级会话绑定
  conversationBindings?: ChannelConversationBindingSupport; // 会话绑定支持

  // 默认值与热重载
  defaults?: { queue?: { debounceMs?: number } }; // 消息队列默认参数
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] }; // 热重载前缀
  commands?: ChannelCommandAdapter; // 命令策略
};
```

### 5.2 必填 vs 可选适配器分级

| 级别        | 适配器                                                      | 说明               | 自定义 Channel 必要性 |
| ----------- | ----------------------------------------------------------- | ------------------ | --------------------- |
| **L0 必填** | `id`, `meta`, `capabilities`, `config`                      | 身份与基础能力     | **必须实现**          |
| **L1 核心** | `outbound`, `gateway`, `messaging`                          | 消息收发与生命周期 | **强烈建议**          |
| **L2 推荐** | `setup`, `security`, `heartbeat`, `status`                  | 配置安全与运维     | 建议实现              |
| **L3 增强** | `threading`, `groups`, `directory`, `actions`, `agentTools` | 高级功能           | 按需实现              |
| **L4 可选** | `pairing`, `doctor`, `streaming`, `commands`, `bindings` 等 | 特定场景           | 视情况而定            |

---

## 六、入站消息标准格式：`MsgContext`

> 源码位置：`src/auto-reply/templating.ts`，第 42-279 行

Channel 接收到外部平台消息后，必须将其转换为统一的 `MsgContext` 对象，这是与 Agent 层对接的核心数据结构。

### 6.1 核心字段（自定义 Channel 必须填充）

```typescript
export type MsgContext = {
  // ━━━ 消息内容（必填） ━━━
  Body?: string; // 消息原始文本
  BodyForAgent?: string; // Agent 看到的文本（可包含上下文/历史）
  CommandBody?: string; // 用于命令检测的纯净文本
  BodyForCommands?: string; // 命令解析文本（优先于 CommandBody）

  // ━━━ 路由信息（必填） ━━━
  From?: string; // 发送者标识（平台用户 ID）
  To?: string; // 接收者/目标标识（群组 ID 或 bot ID）
  SessionKey?: string; // 会话唯一标识（用于会话隔离）
  AccountId?: string; // 多账户场景下的账户 ID

  // ━━━ 消息标识（必填） ━━━
  MessageSid?: string; // 消息唯一 ID
  MessageSidFull?: string; // 完整消息 ID（当 MessageSid 是缩写时）

  // ━━━ Channel 标识（必填） ━━━
  Provider?: string; // 提供者标签（如 "telegram"）
  Surface?: string; // 平台表面标签（优先于 Provider）
  OriginatingChannel?: string; // 来源 Channel（用于回复路由）
  OriginatingTo?: string; // 来源目标 ID（用于回复路由）
  ChatType?: string; // "direct" | "group" | "channel"

  // ━━━ 发送者信息（推荐） ━━━
  SenderName?: string; // 发送者显示名称
  SenderId?: string; // 发送者唯一 ID
  SenderUsername?: string; // 发送者用户名
  Timestamp?: number; // 消息时间戳
};
```

### 6.2 媒体相关字段

```typescript
{
  MediaPath?: string;              // 本地媒体文件路径
  MediaUrl?: string;               // 媒体远程 URL
  MediaType?: string;              // MIME 类型（"image", "audio", "video", "document"）
  MediaDir?: string;               // 媒体存储目录
  MediaPaths?: string[];           // 多媒体文件路径
  MediaUrls?: string[];            // 多媒体远程 URL
  MediaTypes?: string[];           // 多媒体 MIME 类型
  Transcript?: string;             // 语音转文字结果
  Sticker?: {                      // 贴纸元数据（Telegram 特有）
    emoji?: string;
    setName?: string;
    fileId?: string;
    // ...
  };
}
```

### 6.3 引用/回复相关字段

```typescript
{
  ReplyToId?: string;              // 被引用消息 ID
  ReplyToBody?: string;            // 被引用消息文本
  ReplyToSender?: string;          // 被引用消息的发送者
  ReplyChain?: Array<{             // 完整引用链
    messageId?: string;
    sender?: string;
    body?: string;
    mediaType?: string;
    // ...
  }>;
  ForwardedFrom?: string;          // 转发来源
}
```

### 6.4 群组相关字段

```typescript
{
  GroupSubject?: string;           // 群组名称
  GroupChannel?: string;           // 频道标识（如 Slack 的 #general）
  GroupSpace?: string;             // 工作空间标识
  GroupMembers?: string;           // 群组成员信息
  GroupSystemPrompt?: string;      // 群组级系统提示词
  MemberRoleIds?: string[];       // 发送者在群组中的角色 ID
  WasMentioned?: boolean;         // 是否被 @提及
  ExplicitlyMentionedBot?: boolean; // 是否显式 @了 bot
}
```

### 6.5 线程相关字段

```typescript
{
  MessageThreadId?: string | number; // 线程/话题 ID（Telegram topic / Matrix thread）
  TransportThreadId?: string | number; // 传输层线程 ID
  NativeChannelId?: string;        // 平台原生频道 ID（Slack DM channel ID）
  ThreadStarterBody?: string;      // 线程首条消息内容
  ThreadLabel?: string;            // 线程标签
  RootMessageId?: string;          // 线程根消息 ID（飞书 root_id）
}
```

### 6.6 MsgContext → Agent 的数据流

```
Channel 接收平台消息
    │
    ▼
构建 MsgContext 对象（填充上述字段）
    │
    ▼
dispatchInboundMessage({
  ctx: msgContext,          ← 入站消息上下文
  cfg: openclawConfig,      ← 全局配置
  dispatcher: replyDispatcher  ← 回复分发器（包含 deliver 回调）
})
    │
    ▼
finalizeInboundContext(ctx)   ← 标准化上下文
    │
    ▼
dispatchReplyFromConfig()     ← 路由到 Agent 执行
    │
    ▼
Agent 生成回复 (ReplyPayload)
    │
    ▼
通过 dispatcher.deliver() 回调 → Channel 的 outbound 适配器
```

---

## 七、出站消息标准格式：`ChannelOutboundAdapter` 与 `ChannelOutboundContext`

> 源码位置：`src/channels/plugins/outbound.types.ts`

### 7.1 `ChannelOutboundContext` — 出站消息上下文

当 Agent 生成回复后，系统会将回复转换为统一的 `ChannelOutboundContext` 传递给 Channel 的出站适配器：

```typescript
export type ChannelOutboundContext = {
  cfg: OpenClawConfig; // 全局配置
  to: string; // 发送目标（用户/群组 ID）
  text: string; // 消息文本
  mediaUrl?: string; // 媒体 URL
  audioAsVoice?: boolean; // 是否作为语音消息发送
  mediaAccess?: OutboundMediaAccess; // 媒体访问方式
  mediaLocalRoots?: readonly string[]; // 本地媒体根目录
  mediaReadFile?: (path: string) => Promise<Buffer>; // 读取媒体文件
  gifPlayback?: boolean; // GIF 播放模式
  forceDocument?: boolean; // 强制作为文件发送
  replyToId?: string | null; // 引用消息 ID
  replyToIdSource?: "explicit" | "implicit"; // 引用来源
  replyToMode?: ReplyToMode; // 引用模式
  formatting?: OutboundDeliveryFormattingOptions; // 格式化选项
  threadId?: string | number | null; // 线程 ID
  accountId?: string | null; // 账户 ID
  identity?: OutboundIdentity; // 发送者身份
  deps?: OutboundSendDeps; // 发送依赖
  silent?: boolean; // 静默发送
  gatewayClientScopes?: readonly string[]; // Gateway 权限范围
};
```

### 7.2 `ChannelOutboundAdapter` — 出站适配器接口

```typescript
export type ChannelOutboundAdapter = {
  // ━━━ 必填配置 ━━━
  deliveryMode: "direct" | "gateway" | "hybrid";  // 投递模式

  // ━━━ 核心发送方法（至少实现一个） ━━━
  sendText?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendMedia?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendPayload?: (ctx: ChannelOutboundPayloadContext) => Promise<OutboundDeliveryResult>;
  sendFormattedText?: (ctx: ChannelOutboundFormattedContext) => Promise<OutboundDeliveryResult[]>;
  sendFormattedMedia?: (ctx: ChannelOutboundFormattedContext & { mediaUrl: string }) =>
    Promise<OutboundDeliveryResult>;
  sendPoll?: (ctx: ChannelPollContext) => Promise<ChannelPollResult>;

  // ━━━ 文本处理 ━━━
  chunker?: ((text: string, limit: number, ctx?) => string[]) | null;  // 长文本分块器
  chunkerMode?: "text" | "markdown";    // 分块模式
  textChunkLimit?: number;              // 单条消息文本长度上限
  sanitizeText?: (params: { text: string; payload: ReplyPayload }) => string;  // 文本清洗

  // ━━━ 能力声明 ━━━
  presentationCapabilities?: ChannelPresentationCapabilities;  // UI 能力
  deliveryCapabilities?: ChannelDeliveryCapabilities;          // 投递能力

  // ━━━ 目标解析 ━━━
  resolveTarget?: (params: {
    cfg?: OpenClawConfig;
    to?: string;
    allowFrom?: string[];
    accountId?: string | null;
    mode?: ChannelOutboundTargetMode;
  }) => { ok: true; to: string } | { ok: false; error: Error };

  // ━━━ 生命周期钩子 ━━━
  beforeDeliverPayload?: (params: { ... }) => Promise<void> | void;
  afterDeliverPayload?: (params: { ... }) => Promise<void> | void;

  // ━━━ 高级特性 ━━━
  renderPresentation?: (params: { ... }) => Promise<ReplyPayload | null>;  // 富文本渲染
  pinDeliveredMessage?: (params: { ... }) => Promise<void>;                // 置顶消息
  extractMarkdownImages?: boolean;      // 提取 Markdown 图片
  normalizePayload?: (params: { ... }) => ReplyPayload | null;            // 标准化负载

  // ━━━ 回复可见性判断 ━━━
  shouldTreatDeliveredTextAsVisible?: (params: {
    kind: "tool" | "block" | "final";
    text?: string;
  }) => boolean;
  preferFinalAssistantVisibleText?: boolean;
  targetsMatchForReplySuppression?: (params: { ... }) => boolean;

  // ━━━ 投票 ━━━
  pollMaxOptions?: number;
  supportsPollDurationSeconds?: boolean;
  supportsAnonymousPolls?: boolean;
};
```

### 7.3 `ChannelPresentationCapabilities` — UI 能力声明

```typescript
export type ChannelPresentationCapabilities = {
  supported?: boolean; // 是否支持结构化展示
  buttons?: boolean; // 是否支持按钮
  selects?: boolean; // 是否支持选择菜单
  context?: boolean; // 是否支持低优先级上下文块
  divider?: boolean; // 是否支持分隔线
  limits?: {
    actions?: {
      maxActions?: number; // 最大按钮/选项总数
      maxActionsPerRow?: number; // 每行最大按钮数
      maxRows?: number; // 最大行数
      maxLabelLength?: number; // 按钮标签最大长度
      maxValueBytes?: number; // 回调值最大字节数
      supportsStyles?: boolean; // 是否支持样式（primary/danger）
      supportsDisabled?: boolean; // 是否支持禁用态
    };
    text?: {
      maxLength?: number; // 文本最大长度
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
    };
  };
};
```

---

## 八、关键适配器详解

### 8.1 `ChannelConfigAdapter<ResolvedAccount>` — 配置适配器（必填）

> 源码位置：`src/channels/plugins/types.adapters.ts`，第 116-155 行

管理 Channel 的多账户配置读写：

```typescript
export type ChannelConfigAdapter<ResolvedAccount> = {
  // 必填方法
  listAccountIds: (cfg: OpenClawConfig) => string[]; // 列出所有账户 ID
  resolveAccount: (cfg: OpenClawConfig, accountId?) => ResolvedAccount; // 解析账户配置

  // 可选方法
  defaultAccountId?: (cfg: OpenClawConfig) => string; // 默认账户 ID
  isEnabled?: (account: ResolvedAccount, cfg: OpenClawConfig) => boolean; // 是否启用
  isConfigured?: (account: ResolvedAccount, cfg: OpenClawConfig) => boolean | Promise<boolean>;
  setAccountEnabled?: (params: { cfg; accountId; enabled }) => OpenClawConfig; // 开关账户
  deleteAccount?: (params: { cfg; accountId }) => OpenClawConfig; // 删除账户
  resolveAllowFrom?: (params: { cfg; accountId? }) => Array<string | number> | undefined;
  resolveDefaultTo?: (params: { cfg; accountId? }) => string | undefined; // 默认发送目标
};
```

### 8.2 `ChannelGatewayAdapter<ResolvedAccount>` — Gateway 生命周期适配器

> 源码位置：`src/channels/plugins/types.adapters.ts`，第 342-359 行

管理 Channel 账户的启动、停止和登录：

```typescript
export type ChannelGatewayAdapter<ResolvedAccount> = {
  // 启动账户连接（必填）
  startAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<unknown>;

  // 停止账户连接
  stopAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<void>;

  // QR 码登录（WhatsApp 等需要设备配对的 Channel）
  loginWithQrStart?: (params: {
    accountId?;
    force?;
    timeoutMs?;
    verbose?;
  }) => Promise<ChannelLoginWithQrStartResult>;
  loginWithQrWait?: (params: {
    accountId?;
    timeoutMs?;
    currentQrDataUrl?;
  }) => Promise<ChannelLoginWithQrWaitResult>;

  // 登出账户
  logoutAccount?: (ctx: ChannelLogoutContext) => Promise<ChannelLogoutResult>;

  // Gateway 认证旁路路径
  resolveGatewayAuthBypassPaths?: (params: { cfg }) => string[];
};
```

### 8.3 `ChannelGatewayContext` — Gateway 上下文

这是 `startAccount` 接收的完整上下文：

```typescript
export type ChannelGatewayContext<ResolvedAccount> = {
  cfg: OpenClawConfig; // 全局配置
  accountId: string; // 当前账户 ID
  account: ResolvedAccount; // 解析后的账户对象
  runtime: RuntimeEnv; // 运行时环境
  abortSignal: AbortSignal; // 取消信号
  log?: ChannelLogSink; // 日志接口 { info, warn, error, debug }
  getStatus: () => ChannelAccountSnapshot; // 获取当前状态
  setStatus: (next: ChannelAccountSnapshot) => void; // 更新状态

  // 外部插件可用的运行时帮助函数
  channelRuntime?: ChannelRuntimeSurface;
  // channelRuntime.reply    — AI 回复分发
  // channelRuntime.routing  — Agent 路由解析
  // channelRuntime.text     — 文本处理
  // channelRuntime.session  — 会话管理
  // channelRuntime.media    — 媒体处理
  // channelRuntime.commands — 命令处理
  // channelRuntime.groups   — 群组策略
  // channelRuntime.pairing  — 配对管理
};
```

### 8.4 `ChannelMessagingAdapter` — 消息路由适配器

> 源码位置：`src/channels/plugins/types.core.ts`，第 479-634 行

```typescript
export type ChannelMessagingAdapter = {
  targetPrefixes?: readonly string[]; // 目标前缀（如 ["telegram", "tg"]）
  normalizeTarget?: (raw: string) => string | undefined; // 归一化目标地址

  // 会话路由
  resolveOutboundSessionRoute?: (params: {
    cfg: OpenClawConfig;
    agentId: string;
    accountId?: string | null;
    target: string;
    currentSessionKey?: string;
    resolvedTarget?: { to; kind; display?; source };
    replyToId?: string | null;
    threadId?: string | number | null;
  }) => ChannelOutboundSessionRoute | null;

  // 入站会话解析
  resolveInboundConversation?: (params: {
    from?;
    to?;
    conversationId?;
    threadId?;
    isGroup;
  }) => { conversationId?; parentConversationId? } | null;

  // 目标解析
  parseExplicitTarget?: (params: { raw: string }) => {
    to: string;
    threadId?;
    chatType?;
  } | null;
  inferTargetChatType?: (params: { to }) => ChatType | undefined;

  // 目标展示
  formatTargetDisplay?: (params: { target; display?; kind? }) => string;

  // 目标验证
  targetResolver?: {
    looksLikeId?: (raw: string, normalized?) => boolean;
    hint?: string; // 提示格式（如 "<chatId>"）
  };
};
```

### 8.5 `ChannelCapabilities` — 静态能力声明

```typescript
export type ChannelCapabilities = {
  chatTypes: Array<ChatType | "thread">; // 支持的会话类型
  polls?: boolean; // 投票
  reactions?: boolean; // 表情回应
  edit?: boolean; // 消息编辑
  unsend?: boolean; // 撤回消息
  reply?: boolean; // 引用回复
  effects?: boolean; // 消息特效
  groupManagement?: boolean; // 群组管理
  threads?: boolean; // 线程
  media?: boolean; // 媒体文件
  tts?: {
    // 语音合成
    voice?: ChannelTtsVoiceDeliveryCapabilities;
  };
  nativeCommands?: boolean; // 平台原生命令
  blockStreaming?: boolean; // 分块流式输出
};
```

### 8.6 `ChannelMeta` — 元数据

```typescript
export type ChannelMeta = {
  id: ChannelId; // Channel ID
  label: string; // 显示标签（"Telegram"）
  selectionLabel: string; // 选择列表标签
  docsPath: string; // 文档路径
  docsLabel?: string; // 文档链接文字
  blurb: string; // 一句话描述
  order?: number; // 排序权重
  aliases?: readonly string[]; // 别名（如 Telegram 的别名 "tg"）
  markdownCapable?: boolean; // 是否支持 Markdown
  exposure?: ChannelExposure; // 可见性控制
};
```

### 8.7 `ChannelLifecycleAdapter` — 生命周期适配器

```typescript
export type ChannelLifecycleAdapter = {
  onAccountConfigChanged?: (params: {
    prevCfg: OpenClawConfig;
    nextCfg: OpenClawConfig;
    accountId: string;
    runtime: RuntimeEnv;
  }) => Promise<void> | void;

  onAccountRemoved?: (params: {
    prevCfg: OpenClawConfig;
    accountId: string;
    runtime: RuntimeEnv;
  }) => Promise<void> | void;

  runStartupMaintenance?: (params: {
    cfg: OpenClawConfig;
    env?: NodeJS.ProcessEnv;
    log: { info?; warn? };
  }) => Promise<void> | void;

  detectLegacyStateMigrations?: (params: {
    cfg;
    env;
    stateDir;
    oauthDir;
  }) => ChannelLegacyStateMigrationPlan[];
};
```

### 8.8 `ChannelHeartbeatAdapter` — 心跳适配器

```typescript
export type ChannelHeartbeatAdapter = {
  checkReady?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    deps?: ChannelHeartbeatDeps;
  }) => Promise<{ ok: boolean; reason: string }>;

  sendTyping?: (params: {
    cfg: OpenClawConfig;
    to: string;
    accountId?: string | null;
    threadId?: string | number | null;
  }) => Promise<void> | void;

  clearTyping?: (params: { ... }) => Promise<void> | void;
};
```

---

## 九、使用 `createChatChannelPlugin` 工厂函数

系统提供了 `createChatChannelPlugin` 工厂函数简化 `ChannelPlugin` 的创建。

> 源码位置：`src/plugin-sdk/channel-core.ts`（导出自 `openclaw/plugin-sdk/channel-core`）

### 9.1 使用示例

以下展示一个最小化的自定义 Channel 实现模板：

```typescript
// extensions/my-channel/src/channel.ts

import { createChatChannelPlugin } from "openclaw/plugin-sdk/channel-core";
import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";

// 1. 定义账户解析类型
type ResolvedMyChannelAccount = {
  accountId: string;
  token: string;
  enabled: boolean;
  name?: string;
};

// 2. 账户解析函数
function resolveMyChannelAccount(params: {
  cfg: OpenClawConfig;
  accountId?: string | null;
}): ResolvedMyChannelAccount {
  const channelCfg = params.cfg.channels?.["my-channel"] as Record<string, unknown> | undefined;
  return {
    accountId: params.accountId ?? "default",
    token: (channelCfg?.token as string) ?? process.env.MY_CHANNEL_TOKEN ?? "",
    enabled: channelCfg?.enabled !== false,
    name: channelCfg?.name as string | undefined,
  };
}

// 3. 创建 ChannelPlugin
export const myChannelPlugin = createChatChannelPlugin<ResolvedMyChannelAccount>({
  base: {
    id: "my-channel",
    meta: {
      id: "my-channel",
      label: "My Channel",
      selectionLabel: "My Channel",
      docsPath: "/docs/channels/my-channel",
      blurb: "Connect to My Custom Platform",
      markdownCapable: true,
    },
    capabilities: {
      chatTypes: ["direct", "group"],
      media: true,
      reply: true,
    },
    config: {
      listAccountIds: (cfg) => {
        const accounts = (cfg.channels?.["my-channel"] as any)?.accounts;
        return accounts ? Object.keys(accounts) : ["default"];
      },
      resolveAccount: (cfg, accountId) => resolveMyChannelAccount({ cfg, accountId }),
    },

    // 消息路由
    messaging: {
      targetPrefixes: ["my-channel"],
      normalizeTarget: (raw) => raw.replace(/^my-channel:/i, "").trim() || undefined,
      targetResolver: {
        looksLikeId: (raw) => /^\d+$/.test(raw),
        hint: "<userId or groupId>",
      },
    },

    // 心跳
    heartbeat: {
      sendTyping: async ({ cfg, to, accountId }) => {
        // 调用平台 API 发送 typing 状态
      },
    },

    // Gateway 生命周期
    gateway: {
      startAccount: async (ctx) => {
        ctx.log?.info(`[${ctx.accountId}] starting my-channel provider`);
        // 启动消息接收（WebSocket / Webhook / Polling 等）
        // 接收到消息后构建 MsgContext 并调用 dispatchInboundMessage
      },
      stopAccount: async (ctx) => {
        ctx.log?.info(`[${ctx.accountId}] stopping my-channel provider`);
        // 关闭连接
      },
    },
  },

  // 出站适配器
  outbound: {
    deliveryMode: "direct",
    textChunkLimit: 4096,
    sendText: async (ctx) => {
      // 调用平台 API 发送文本消息
      const result = await myPlatformApi.sendMessage(ctx.to, ctx.text, {
        token: resolveMyChannelAccount({ cfg: ctx.cfg, accountId: ctx.accountId }).token,
        replyToId: ctx.replyToId,
        threadId: ctx.threadId,
      });
      return { messageId: result.id, ok: true };
    },
    sendMedia: async (ctx) => {
      // 调用平台 API 发送媒体消息
      const result = await myPlatformApi.sendMedia(ctx.to, ctx.mediaUrl!, {
        token: resolveMyChannelAccount({ cfg: ctx.cfg, accountId: ctx.accountId }).token,
      });
      return { messageId: result.id, ok: true };
    },
  },

  // 配对
  pairing: {
    idLabel: "myChannelUserId",
    normalizeAllowEntry: (entry) => entry.replace(/^my-channel:/i, ""),
  },

  // 安全
  security: {
    resolveDmPolicy: (ctx) => ({
      policy: ctx.account.enabled ? "allow" : "deny",
      allowFrom: null,
      allowFromPath: "channels.my-channel.allowFrom",
      approveHint: "Add user ID to channels.my-channel.allowFrom",
    }),
  },
});
```

---

## 十、Telegram vs WhatsApp 实现对比

通过对比两个成熟 Channel 的实现，可以清晰看到**相同接口、不同实现**的设计模式：

| 维度                           | Telegram                                  | WhatsApp                               |
| ------------------------------ | ----------------------------------------- | -------------------------------------- |
| **源文件数**                   | ~295                                      | ~144                                   |
| **认证方式**                   | Bot Token（字符串）                       | Baileys + QR 码配对                    |
| **消息接收**                   | HTTP Polling 或 Webhook                   | Baileys WebSocket 连接                 |
| **`gateway.startAccount`**     | 验证 token → getMe → 启动 polling/webhook | 读取本地 auth → 启动 Baileys WebSocket |
| **`outbound.sendText`**        | `sendMessage` → Telegram Bot API          | `sendMessage` → Baileys WA API         |
| **`auth`**                     | 无（Token 即认证）                        | `loginWeb()` 通过 QR 码                |
| **`heartbeat.sendTyping`**     | `sendChatAction("typing")` → Telegram API | Baileys presence update                |
| **`messaging.targetPrefixes`** | `["telegram", "tg"]`                      | `["whatsapp"]`                         |
| **`threading`**                | 支持 Forum Topic（`topic:$id`）           | 基于 `replyToMode` 配置                |
| **`agentTools`**               | 无                                        | `createWhatsAppLoginTool()`            |
| **`capabilities.chatTypes`**   | `["direct", "group"]`                     | `["direct", "group"]`                  |

两者都使用完全相同的 `createChatChannelPlugin()` 工厂函数组装，最终导出的对象都符合 `ChannelPlugin` 接口。

---

## 十一、`openclaw.json` 配置节

每个 Channel 在全局配置文件 `openclaw.json` 中有一个专属配置节：

```json
{
  "channels": {
    "my-channel": {
      "enabled": true,
      "token": "YOUR_API_TOKEN",
      "allowFrom": ["user_id_1", "user_id_2"],
      "accounts": {
        "default": {
          "enabled": true,
          "token": "TOKEN_FOR_DEFAULT_ACCOUNT"
        },
        "secondary": {
          "enabled": true,
          "token": "TOKEN_FOR_SECONDARY_ACCOUNT"
        }
      }
    }
  }
}
```

`ChannelConfigAdapter.resolveAccount()` 方法负责从该配置节解析出强类型的账户对象。

---

## 十二、自定义 Channel 接入步骤清单

### 步骤 1：创建插件目录

```bash
mkdir -p extensions/my-channel/src
```

### 步骤 2：编写 `openclaw.plugin.json`

```json
{
  "id": "my-channel",
  "activation": { "onStartup": false },
  "channels": ["my-channel"],
  "channelEnvVars": {
    "my-channel": ["MY_CHANNEL_API_TOKEN"]
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "token": { "type": "string" },
      "enabled": { "type": "boolean" },
      "allowFrom": {
        "type": "array",
        "items": { "type": "string" }
      }
    }
  }
}
```

### 步骤 3：编写入口文件 `index.ts`

```typescript
import { defineBundledChannelEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Custom channel plugin for My Platform",
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./src/channel.js",
    exportName: "myChannelPlugin",
  },
});
```

### 步骤 4：实现 `ChannelPlugin`（`src/channel.ts`）

参照第九节的完整模板，实现以下核心适配器：

| 优先级 | 适配器                 | 要做的事                                                                                 |
| ------ | ---------------------- | ---------------------------------------------------------------------------------------- |
| **P0** | `config`               | 实现 `listAccountIds` 和 `resolveAccount`                                                |
| **P0** | `gateway.startAccount` | 启动消息接收（轮询/WebSocket/Webhook），构建 `MsgContext`，调用 `dispatchInboundMessage` |
| **P0** | `outbound.sendText`    | 调用平台 API 发送文本消息                                                                |
| **P1** | `outbound.sendMedia`   | 调用平台 API 发送媒体消息                                                                |
| **P1** | `messaging`            | 实现目标地址解析和归一化                                                                 |
| **P1** | `heartbeat.sendTyping` | 发送 typing 状态                                                                         |
| **P2** | `setup`                | 实现 `applyAccountConfig` 配置写入                                                       |
| **P2** | `security`             | 实现 DM 安全策略                                                                         |
| **P2** | `status`               | 实现健康检查和账户快照                                                                   |

### 步骤 5：`startAccount` 中的消息接收与分发

这是最核心的部分——在 `gateway.startAccount` 中启动消息接收循环，将平台消息转换为 `MsgContext` 并交给系统分发：

```typescript
gateway: {
  startAccount: async (ctx) => {
    const account = ctx.account;
    ctx.log?.info(`[${account.accountId}] starting my-channel provider`);

    // 启动消息接收（示例：WebSocket）
    const ws = new WebSocket(`wss://api.my-platform.com/ws?token=${account.token}`);

    // 监听取消信号
    ctx.abortSignal.addEventListener("abort", () => ws.close());

    ws.on("message", async (data) => {
      const msg = JSON.parse(data);

      // 构建标准 MsgContext
      const msgContext: MsgContext = {
        Body: msg.text,
        BodyForAgent: msg.text,
        From: String(msg.sender_id),
        To: String(msg.chat_id),
        SessionKey: `my-channel:${msg.chat_id}`,
        AccountId: account.accountId,
        MessageSid: String(msg.message_id),
        Provider: "my-channel",
        Surface: "my-channel",
        OriginatingChannel: "my-channel",
        OriginatingTo: String(msg.chat_id),
        ChatType: msg.is_group ? "group" : "direct",
        SenderName: msg.sender_name,
        SenderId: String(msg.sender_id),
        Timestamp: msg.timestamp,
      };

      // 如果有媒体附件
      if (msg.media_url) {
        msgContext.MediaUrl = msg.media_url;
        msgContext.MediaType = msg.media_type;  // "image", "audio", "video", "document"
      }

      // 如果是引用回复
      if (msg.reply_to) {
        msgContext.ReplyToId = String(msg.reply_to.message_id);
        msgContext.ReplyToBody = msg.reply_to.text;
        msgContext.ReplyToSender = msg.reply_to.sender_name;
      }

      // 分发到 Agent 执行
      // 使用 channelRuntime（外部插件）或直接 import（内置插件）
      if (ctx.channelRuntime?.reply) {
        await ctx.channelRuntime.reply.dispatchReplyWithBufferedBlockDispatcher({
          ctx: msgContext,
          cfg: ctx.cfg,
          dispatcherOptions: {
            deliver: async (payload) => {
              // 通过 outbound 发送回复
              await myChannelPlugin.outbound!.sendText!({
                cfg: ctx.cfg,
                to: msgContext.From!,
                text: payload.text ?? "",
                accountId: account.accountId,
              });
            },
          },
        });
      }
    });

    // 更新状态
    ctx.setStatus({
      accountId: account.accountId,
      running: true,
      connected: true,
    });
  },
},
```

### 步骤 6：配置 `openclaw.json`

```json
{
  "channels": {
    "my-channel": {
      "enabled": true,
      "token": "YOUR_API_TOKEN",
      "allowFrom": ["allowed_user_1"]
    }
  }
}
```

### 步骤 7：验证

1. 启动 OpenClaw 服务
2. 检查日志确认 Channel 启动成功
3. 从外部平台发送消息验证入站流程
4. 验证 Agent 回复能正确送达外部平台

---

## 十三、关键注意事项

### 13.1 `MsgContext` 字段填充优先级

| 字段                                   | 重要性   | 影响                         |
| -------------------------------------- | -------- | ---------------------------- |
| `Body`                                 | 必填     | Agent 无法看到消息内容       |
| `From` / `To`                          | 必填     | 无法路由消息                 |
| `SessionKey`                           | 必填     | 无法创建/恢复会话            |
| `OriginatingChannel` + `OriginatingTo` | 必填     | 回复无法路由回正确的 Channel |
| `ChatType`                             | 强烈建议 | 影响群组策略和 DM 策略       |
| `Surface` / `Provider`                 | 强烈建议 | 影响日志和诊断               |
| `SenderName` / `SenderId`              | 推荐     | 影响安全策略和白名单检查     |
| `MediaPath` / `MediaUrl`               | 按需     | 仅当消息包含媒体时           |

### 13.2 `OriginatingChannel` 是回复路由的关键

系统使用 `OriginatingChannel` 和 `OriginatingTo` 来决定 Agent 回复应该发送到哪个 Channel 的哪个目标。如果不设置这两个字段，回复可能无法正确路由回你的自定义 Channel。

### 13.3 `SessionKey` 的命名规范

推荐格式：`<channel-id>:<chat_id>` 或 `<channel-id>:<account_id>:<chat_id>`

SessionKey 用于：

- 会话隔离（不同聊天窗口的对话独立）
- 会话历史关联
- 模型/Agent 绑定

### 13.4 多账户支持

所有 Channel 都支持多账户模式。`ChannelConfigAdapter` 的 `listAccountIds` 和 `resolveAccount` 方法需要正确处理多账户场景。推荐在 `openclaw.json` 中支持如下结构：

```json
{
  "channels": {
    "my-channel": {
      "token": "DEFAULT_TOKEN",
      "accounts": {
        "bot-1": { "token": "TOKEN_1", "name": "Bot 1" },
        "bot-2": { "token": "TOKEN_2", "name": "Bot 2" }
      }
    }
  }
}
```

### 13.5 错误处理

- `gateway.startAccount` 抛出的错误会阻止该账户启动
- `outbound.sendText` 等发送方法的错误会被 ReplyDispatcher 捕获并记录
- 建议在 `startAccount` 中使用 `ctx.abortSignal` 监听取消事件，确保资源清理

---

## 十四、现有内置 Channel 列表（部分）

| Channel ID      | 平台            | 连接方式                | 认证方式                      |
| --------------- | --------------- | ----------------------- | ----------------------------- |
| `telegram`      | Telegram        | HTTP Polling / Webhook  | Bot Token                     |
| `whatsapp`      | WhatsApp        | Baileys WebSocket       | QR 码配对                     |
| `slack`         | Slack           | Socket Mode / Web API   | Bot Token + App Token         |
| `discord`       | Discord         | Discord.js Gateway      | Bot Token                     |
| `feishu`        | 飞书/Lark       | WebSocket / Webhook     | App ID + App Secret           |
| `line`          | LINE            | Webhook                 | Channel Access Token + Secret |
| `msteams`       | Microsoft Teams | Bot Framework           | App ID + Password             |
| `matrix`        | Matrix          | Matrix SDK              | Access Token                  |
| `signal`        | Signal          | signal-cli              | Phone Number                  |
| `imessage`      | iMessage        | AppleScript / Shortcuts | macOS 系统集成                |
| `nostr`         | Nostr           | Relay WebSocket         | Private Key                   |
| `irc`           | IRC             | IRC Protocol            | Server + Nick                 |
| `googlechat`    | Google Chat     | Webhook / API           | Service Account               |
| `mattermost`    | Mattermost      | WebSocket + REST        | Token                         |
| `synology-chat` | Synology Chat   | Webhook                 | Token                         |

系统目前内置了 **50+ 个 Channel 插件**，覆盖了主流即时通讯平台。

---

## 十五、总结

Channel 层的核心设计遵循**适配器模式（Adapter Pattern）**：

1. **统一接口**：所有 Channel 实现同一个 `ChannelPlugin` 接口
2. **统一入站格式**：所有平台消息转换为 `MsgContext`
3. **统一出站格式**：所有回复通过 `ChannelOutboundContext` 分发
4. **内部自由**：每个 Channel 内部的连接方式、认证方式、API 调用完全自主
5. **零侵入接入**：新增 Channel 不需要修改 Gateway、Agent 或任何其他 Channel 的代码

接入自定义 Channel 的核心工作量在于：

- 实现与外部平台的消息收发（入站消息转 `MsgContext`，出站 `ChannelOutboundContext` 转平台 API）
- 注册插件清单和入口文件
- 配置账户管理逻辑
