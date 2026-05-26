# Phase 2：Gateway 层源码深度解析

> 分析对象：OpenClaw 六层架构中的 **L2 Gateway 层**
> 分析目标：标准接入方式、核心功能模块、输入输出接口规范、二次开发技术细节

---

## 1. Gateway 层定位

Gateway 是 OpenClaw 的 **有状态后端中枢**，不是简单的代理层。它是一个完整的后端系统，负责：

- 接收所有 Client 的 WebSocket / HTTP 请求
- 统一认证、鉴权、限流
- 将请求路由到对应的 Handler（含 Agent 执行、会话管理、配置操作等）
- 将 Agent 运行时事件推送回 Client
- 托管 Control UI 静态资源
- 管理配置热加载、Channel 插件、Cron 定时任务等生命周期

**核心结论：Gateway 层代码复用度约 98-100%，接入新客户端不需要修改 Gateway 核心代码。**

---

## 2. Gateway 核心功能模块

### 2.1 模块全景图

```
Gateway 层内部架构
│
├── 🔌 传输层（Transport）
│   ├── server-http.ts                → HTTP 服务器（REST API + Control UI + 插件路由）
│   ├── server-runtime-state.ts       → WebSocketServer 创建 + HTTP 绑定
│   ├── server/ws-connection.ts       → WS 连接生命周期管理
│   └── server/http-listen.ts         → 端口绑定与监听
│
├── 🔐 认证层（Auth）
│   ├── auth.ts                       → 统一认证入口（WS + HTTP）
│   ├── auth-rate-limit.ts            → 暴力破解限流
│   ├── device-auth.ts                → Ed25519 设备签名校验
│   ├── role-policy.ts                → 角色策略（operator / node）
│   ├── method-scopes.ts              → 方法级权限控制
│   └── origin-check.ts              → 浏览器 Origin 校验
│
├── 🔀 路由层（Dispatch）
│   ├── server-methods.ts             → handleGatewayRequest() 总入口
│   ├── methods/registry.ts           → GatewayMethodRegistry
│   ├── methods/core-descriptors.ts   → 100+ 核心方法声明
│   └── server-methods/*.ts           → ~30 组 RPC Handler
│
├── 📡 事件推送层（Broadcast）
│   ├── server-broadcast.ts           → 事件广播器（scope-guarded fan-out）
│   ├── server-session-events.ts      → 会话事件分发
│   ├── server/presence-events.ts     → 在线状态事件
│   └── server-methods-list.ts        → 事件类型注册表
│
├── 📦 状态管理层（State）
│   ├── session-utils.ts              → 会话存储/查询
│   ├── chat-abort.ts                 → Agent 运行中止控制
│   ├── server-chat-state.ts          → 聊天状态缓冲
│   ├── server-shared.ts              → 去重（dedupe）状态
│   └── node-registry.ts             → 已连接 Node 设备注册表
│
├── ♻️ 生命周期层（Lifecycle）
│   ├── server.impl.ts                → Gateway 启动主流程（~1690 行）
│   ├── server-reload-handlers.ts     → 配置热加载
│   ├── config-reload.ts              → 文件监听 + 变更计划
│   ├── server-close.ts               → 优雅关闭
│   └── server-startup-post-attach.ts → 启动后侧车服务
│
└── 🌐 静态托管层（Control UI）
    ├── control-ui.ts                 → SPA 静态服务 + CSP 安全策略
    ├── server-control-ui-root.ts     → UI 产物路径解析
    └── control-ui-csp.ts             → Content-Security-Policy
```

### 2.2 启动流程详解

Gateway 通过 `startGatewayServer(port, opts)` 启动，依次执行以下阶段：

```
src/gateway/server.impl.ts → startGatewayServer()
│
├── 1. 网络初始化        → server-network-runtime.ts
├── 2. 配置快照 + 认证    → server-startup-config.ts, startup-auth.ts
├── 3. 插件引导          → server-startup-plugins.ts, server-plugin-bootstrap.ts
├── 4. 运行时配置解析     → server-runtime-config.ts（bind/TLS/auth/Tailscale）
├── 5. HTTP + WSS 创建   → createGatewayRuntimeState()（WebSocketServer + HTTP Server）
├── 6. 早期侧车          → server-startup-early.ts（Bonjour 发现、维护定时器）
├── 7. 事件订阅          → server-runtime-subscriptions.ts（Agent 事件 → broadcast）
├── 8. 方法注册表        → server-methods.ts（Core + Plugin + Aux handlers）
├── 9. WS Handler 挂载   → server-ws-runtime.ts → ws-connection.ts → message-handler.ts
├── 10. 端口监听          → server/http-listen.ts
├── 11. 后期侧车         → server-startup-post-attach.ts（Channels/Memory/Cron/Model预热）
└── 12. 配置热加载启动    → server-reload-handlers.ts → config-reload.ts
```

> 源码入口：`src/gateway/server.impl.ts` 第 531 行 `startGatewayServer()`

---

## 3. 标准接入方式——请求处理全链路

### 3.1 WS 请求处理链路

```
Client WebSocket 连接
  ↓
socket.on("message")                    [message-handler.ts:1883]
  ↓ 解析 JSON → 判断帧类型
  ↓
第一帧: connect 握手请求
  ↓ validateConnectParams()             [protocol/index.ts]
  ↓ 设备签名校验                        [ws-connection/handshake-auth-helpers.ts]
  ↓ 认证决策                            [ws-connection/auth-context.ts]
  ↓ 配对检查                            [infra/device-pairing.js]
  ↓ 构建 hello-ok 响应
  ↓ 发送 hello-ok（含 snapshot/features/auth）
  ↓
后续帧: RPC 请求
  ↓ validateRequestFrame()              [protocol/index.ts]
  ↓ handleGatewayRequest()              [server-methods.ts:179]
    ↓ authorizeGatewayMethod()          → role + scope 检查
    ↓ controlPlaneWrite 限流检查        → 3次/分钟
    ↓ methodRegistry.getHandler()       → 方法名 → handler 函数
    ↓ handler({ req, params, client, respond, context })
      ↓ 业务逻辑处理
      ↓ respond(ok, payload)            → 发送响应帧
```

### 3.2 方法路由核心代码

```typescript
// src/gateway/server-methods.ts (简化)
export async function handleGatewayRequest(opts) {
  const { req, respond, client, context } = opts;
  const methodRegistry = opts.methodRegistry;

  // 1. 角色 + scope 鉴权
  const authError = authorizeGatewayMethod(req.method, client, req.params);
  if (authError) { respond(false, undefined, authError); return; }

  // 2. 启动期间不可用方法检查
  if (context.unavailableGatewayMethods?.has(req.method)) {
    respond(false, undefined, errorShape("UNAVAILABLE", "...")); return;
  }

  // 3. 控制面写操作限流
  if (methodRegistry.isControlPlaneWrite(req.method)) {
    const budget = consumeControlPlaneWriteBudget({ client });
    if (!budget.allowed) { respond(false, ...); return; }
  }

  // 4. 查找 handler 并执行
  const handler = methodRegistry.getHandler(req.method);
  if (!handler) { respond(false, undefined, "unknown method"); return; }

  // 5. 在插件作用域内执行
  await withPluginRuntimeGatewayRequestScope({ context, client }, () =>
    handler({ req, params: req.params, client, respond, context })
  );
}
```

**关键设计：handler 的签名是统一的、客户端无关的。** `client` 参数仅提供连接元数据（role/scopes/connId），不影响业务路由。

---

## 4. 输入输出接口规范

### 4.1 WS 接口——帧格式规范

#### 4.1.1 客户端 → Gateway（请求帧）

```json
{
  "type": "req",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "agent",
  "params": {
    "message": "你好",
    "sessionKey": "main",
    "idempotencyKey": "unique-key-123"
  }
}
```

| 字段     | 类型     | 必填 | 说明                   |
| -------- | -------- | ---- | ---------------------- |
| `type`   | `"req"`  | 是   | 固定值                 |
| `id`     | `string` | 是   | 请求唯一 ID（UUID）    |
| `method` | `string` | 是   | RPC 方法名             |
| `params` | `object` | 否   | 方法参数（各方法不同） |

#### 4.1.2 Gateway → 客户端（响应帧）

```json
{
  "type": "res",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "ok": true,
  "payload": {
    "runId": "run-abc-123",
    "status": "accepted"
  }
}
```

错误响应：

```json
{
  "type": "res",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "missing scope: operator.write",
    "retryable": false
  }
}
```

#### 4.1.3 Gateway → 客户端（事件帧）

```json
{
  "type": "event",
  "event": "agent",
  "payload": {
    "runId": "run-abc-123",
    "type": "delta",
    "text": "你好！"
  },
  "seq": 42,
  "stateVersion": { "presence": 5, "health": 3 }
}
```

| 字段           | 类型      | 必填 | 说明                           |
| -------------- | --------- | ---- | ------------------------------ |
| `type`         | `"event"` | 是   | 固定值                         |
| `event`        | `string`  | 是   | 事件名称                       |
| `payload`      | `any`     | 否   | 事件数据                       |
| `seq`          | `number`  | 否   | 广播事件序列号（用于丢失检测） |
| `stateVersion` | `object`  | 否   | 状态版本（presence/health）    |

### 4.2 connect 握手接口

#### 4.2.1 Gateway → Client: connect.challenge

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "random-nonce-string" }
}
```

#### 4.2.2 Client → Gateway: connect 请求

```json
{
  "type": "req",
  "id": "uuid",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "openclaw-control-ui",
      "version": "1.0.0",
      "platform": "web",
      "mode": "webchat",
      "instanceId": "optional-instance-id",
      "displayName": "My Client",
      "deviceFamily": "desktop"
    },
    "role": "operator",
    "scopes": [
      "operator.admin",
      "operator.read",
      "operator.write",
      "operator.approvals",
      "operator.pairing"
    ],
    "device": {
      "id": "sha256-of-pubkey-hex",
      "publicKey": "base64url-ed25519-pubkey",
      "signature": "base64url-signature",
      "signedAt": 1716393600000,
      "nonce": "server-provided-nonce"
    },
    "caps": ["tool-events"],
    "auth": {
      "token": "gateway-shared-token",
      "deviceToken": "device-specific-token"
    },
    "userAgent": "MyApp/1.0",
    "locale": "zh-CN"
  }
}
```

#### 4.2.3 Gateway → Client: hello-ok 响应

```json
{
  "type": "res",
  "id": "uuid",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": {
      "version": "2026.5.1",
      "connId": "conn-abc-123"
    },
    "features": {
      "methods": ["health", "agent", "chat.send", "sessions.list", "..."],
      "events": ["agent", "chat", "presence", "tick", "..."]
    },
    "snapshot": {
      "presence": [{ "host": "...", "mode": "cli", "...": "..." }],
      "stateVersion": { "presence": 1, "health": 1 },
      "health": { "...": "..." },
      "sessions": [{ "key": "main", "...": "..." }]
    },
    "auth": {
      "role": "operator",
      "scopes": ["operator.admin", "operator.read", "operator.write"],
      "deviceToken": "new-device-token-if-issued",
      "issuedAtMs": 1716393600000
    },
    "policy": {
      "maxPayload": 10485760,
      "maxBufferedBytes": 16777216,
      "tickIntervalMs": 30000
    }
  }
}
```

> hello-ok 的 `features.methods` 告诉客户端当前 Gateway 支持哪些 RPC 方法，`features.events` 告诉客户端会收到哪些事件类型。

### 4.3 HTTP REST 接口

Gateway 同时暴露 HTTP 端点，支持非 WebSocket 场景：

| 路径                      | 方法 | 说明                               | 认证         |
| ------------------------- | ---- | ---------------------------------- | ------------ |
| `/v1/chat/completions`    | POST | OpenAI 兼容的 Chat Completions API | Bearer Token |
| `/v1/responses`           | POST | Open Responses API                 | Bearer Token |
| `/v1/tools/invoke`        | POST | 工具调用                           | Bearer Token |
| `/v1/models`              | GET  | 模型列表                           | Bearer Token |
| `/v1/embeddings`          | POST | 嵌入向量                           | Bearer Token |
| `/api/history`            | GET  | 会话历史                           | Bearer Token |
| `/api/sessions/:key/kill` | POST | 终止会话                           | Bearer Token |
| `/health/live`            | GET  | 存活探针                           | 无需认证     |
| `/health/ready`           | GET  | 就绪探针                           | 无需认证     |
| `/`                       | GET  | Control UI SPA                     | 可选认证     |

> 源码依据：`src/gateway/server-http.ts` 中的 `createGatewayHttpServer()` 构建了 HTTP 路由管线。

---

## 5. 核心 RPC 方法完整清单

### 5.1 按功能域分组

以下是 `methods/core-descriptors.ts` 中定义的 100+ 核心方法，按功能域分组：

#### 5.1.1 Agent 执行域

| 方法                 | Scope            | 说明                        |
| -------------------- | ---------------- | --------------------------- |
| `agent`              | `operator.write` | 发起 Agent 执行（核心入口） |
| `agent.wait`         | `operator.write` | 阻塞等待 Agent 运行结束     |
| `agent.identity.get` | `operator.read`  | 获取 Agent 身份信息         |

#### 5.1.2 会话管理域

| 方法                 | Scope            | 说明                         |
| -------------------- | ---------------- | ---------------------------- |
| `sessions.list`      | `operator.read`  | 列出所有会话                 |
| `sessions.create`    | `operator.write` | 创建新会话                   |
| `sessions.send`      | `operator.write` | 向会话发送消息（触发 Agent） |
| `sessions.abort`     | `operator.write` | 中止正在运行的会话           |
| `sessions.patch`     | `operator.admin` | 修改会话属性                 |
| `sessions.reset`     | `operator.admin` | 重置会话                     |
| `sessions.delete`    | `operator.admin` | 删除会话                     |
| `sessions.compact`   | `operator.admin` | 压缩会话历史                 |
| `sessions.subscribe` | `operator.read`  | 订阅会话变更事件             |
| `sessions.preview`   | `operator.read`  | 预览会话内容                 |
| `sessions.describe`  | `operator.read`  | 获取会话详情                 |

#### 5.1.3 聊天域

| 方法           | Scope            | 说明                            |
| -------------- | ---------------- | ------------------------------- |
| `chat.send`    | `operator.write` | 发送聊天消息（流式 Agent 交互） |
| `chat.history` | `operator.read`  | 获取聊天历史                    |
| `chat.abort`   | `operator.write` | 中止聊天                        |

#### 5.1.4 模型与工具域

| 方法                | Scope            | 说明               |
| ------------------- | ---------------- | ------------------ |
| `models.list`       | `operator.read`  | 列出可用模型       |
| `models.authStatus` | `operator.read`  | 模型认证状态       |
| `tools.catalog`     | `operator.read`  | 列出可用工具       |
| `tools.effective`   | `operator.read`  | 当前生效的工具列表 |
| `tools.invoke`      | `operator.write` | 调用工具           |

#### 5.1.5 配置域

| 方法            | Scope            | 说明                     |
| --------------- | ---------------- | ------------------------ |
| `config.get`    | `operator.read`  | 读取配置                 |
| `config.set`    | `operator.admin` | 设置配置                 |
| `config.apply`  | `operator.admin` | 应用配置（控制面写操作） |
| `config.patch`  | `operator.admin` | 修改部分配置             |
| `config.schema` | `operator.admin` | 获取配置 JSON Schema     |

#### 5.1.6 Agent 管理域

| 方法                | Scope            | 说明            |
| ------------------- | ---------------- | --------------- |
| `agents.list`       | `operator.read`  | 列出 Agent      |
| `agents.create`     | `operator.admin` | 创建 Agent      |
| `agents.update`     | `operator.admin` | 更新 Agent      |
| `agents.delete`     | `operator.admin` | 删除 Agent      |
| `agents.files.list` | `operator.read`  | 列出 Agent 文件 |
| `agents.files.get`  | `operator.read`  | 获取 Agent 文件 |
| `agents.files.set`  | `operator.admin` | 设置 Agent 文件 |

#### 5.1.7 任务与审批域

| 方法                    | Scope                | 说明         |
| ----------------------- | -------------------- | ------------ |
| `tasks.list`            | `operator.read`      | 列出任务     |
| `tasks.get`             | `operator.read`      | 获取任务详情 |
| `tasks.cancel`          | `operator.write`     | 取消任务     |
| `exec.approval.list`    | `operator.approvals` | 列出审批请求 |
| `exec.approval.resolve` | `operator.approvals` | 处理审批     |

#### 5.1.8 渠道与 Cron 域

| 方法              | Scope            | 说明             |
| ----------------- | ---------------- | ---------------- |
| `channels.status` | `operator.read`  | 渠道状态         |
| `channels.start`  | `operator.admin` | 启动渠道         |
| `channels.stop`   | `operator.admin` | 停止渠道         |
| `cron.list`       | `operator.read`  | 列出定时任务     |
| `cron.add`        | `operator.admin` | 添加定时任务     |
| `cron.run`        | `operator.admin` | 手动执行定时任务 |

#### 5.1.9 设备配对域

| 方法                  | Scope              | 说明         |
| --------------------- | ------------------ | ------------ |
| `device.pair.list`    | `operator.pairing` | 列出设备     |
| `device.pair.approve` | `operator.pairing` | 批准配对     |
| `device.pair.reject`  | `operator.pairing` | 拒绝配对     |
| `device.pair.remove`  | `operator.pairing` | 移除设备     |
| `device.token.rotate` | `operator.pairing` | 轮换设备令牌 |

#### 5.1.10 Node 设备域

| 方法                 | Scope            | 说明                 |
| -------------------- | ---------------- | -------------------- |
| `node.invoke`        | `operator.write` | 向 Node 设备发送命令 |
| `node.list`          | `operator.read`  | 列出已连接 Node      |
| `node.event`         | `node`           | Node 上报事件        |
| `node.invoke.result` | `node`           | Node 返回命令结果    |

#### 5.1.11 健康与诊断域

| 方法                    | Scope           | 说明       |
| ----------------------- | --------------- | ---------- |
| `health`                | `operator.read` | 健康检查   |
| `diagnostics.stability` | `operator.read` | 稳定性诊断 |
| `status`                | `operator.read` | 系统状态   |

---

## 6. 事件推送系统

### 6.1 事件类型完整清单

以下是 `server-methods-list.ts` 中注册的所有 Gateway 事件：

| 事件名称                  | Scope 守卫           | 说明                                     |
| ------------------------- | -------------------- | ---------------------------------------- |
| `connect.challenge`       | 无                   | 握手挑战（仅握手阶段）                   |
| `agent`                   | `operator.read`      | Agent 运行时事件（delta/tool/lifecycle） |
| `chat`                    | `operator.read`      | 聊天投影事件（delta/final）              |
| `session.message`         | `operator.read`      | 会话消息事件                             |
| `session.operation`       | `operator.read`      | 会话操作事件                             |
| `session.tool`            | `operator.read`      | 会话工具执行事件                         |
| `sessions.changed`        | `operator.read`      | 会话列表变更通知                         |
| `presence`                | 无                   | 在线状态变化（所有客户端可见）           |
| `tick`                    | 无                   | 心跳（所有客户端可见）                   |
| `health`                  | 无                   | 健康状态变化                             |
| `heartbeat`               | 无                   | 定期心跳                                 |
| `shutdown`                | 无                   | Gateway 关闭通知                         |
| `cron`                    | `operator.read`      | Cron 任务事件                            |
| `talk.event`              | `operator.read`      | 语音会话事件                             |
| `talk.mode`               | `operator.write`     | 语音模式变更                             |
| `exec.approval.requested` | `operator.approvals` | 审批请求通知                             |
| `exec.approval.resolved`  | `operator.approvals` | 审批结果通知                             |
| `plugin.approval.*`       | `operator.approvals` | 插件审批事件                             |
| `device.pair.requested`   | `operator.pairing`   | 设备配对请求                             |
| `device.pair.resolved`    | `operator.pairing`   | 设备配对结果                             |
| `node.pair.*`             | `operator.pairing`   | Node 配对事件                            |
| `voicewake.changed`       | `operator.read`      | 唤醒词变更                               |
| `update.available`        | 无                   | 版本更新通知                             |

### 6.2 事件 Scope 守卫机制

Gateway 的事件广播器内置了 **scope-guarded fan-out**，确保不同权限的客户端只收到其有权查看的事件：

```typescript
// src/gateway/server-broadcast.ts (简化)
const EVENT_SCOPE_GUARDS = {
  agent: ["operator.read"], // 需要 read 权限才能收到 agent 事件
  chat: ["operator.read"],
  health: [], // 空数组 = 所有连接都能收到
  presence: [],
  tick: [],
  "exec.approval.requested": ["operator.approvals"],
  // ...
};

function hasEventScope(client, event) {
  const required = EVENT_SCOPE_GUARDS[event];
  if (required.length === 0) return true; // 无 scope 要求
  const scopes = client.connect.scopes;
  if (scopes.includes("operator.admin")) return true; // admin 全通
  return required.some((s) => scopes.includes(s)); // 逐个检查
}
```

> **Node 角色的客户端** 只能收到 `voicewake.changed` 和 `voicewake.routing.changed` 两类广播事件。

### 6.3 广播器实现

```typescript
// src/gateway/server-broadcast.ts (简化)
export function createGatewayBroadcaster({ clients }) {
  const broadcast = (event, payload, opts) => {
    for (const client of clients) {
      if (!hasEventScope(client, event)) continue; // scope 过滤
      if (client.socket.bufferedAmount > MAX_BUFFERED_BYTES) {
        if (opts?.dropIfSlow) continue; // 慢消费者：丢弃
        client.socket.close(1008, "slow consumer"); // 或断开
        continue;
      }
      const frame = `{"type":"event","event":${JSON.stringify(event)},...,"seq":${nextSeq}}`;
      client.socket.send(frame); // 逐客户端发送
    }
  };
  return { broadcast, broadcastToConnIds };
}
```

---

## 7. 认证体系详解

### 7.1 认证模式

Gateway 支持多种认证模式，通过配置选择：

| 认证模式  | 配置值       | 说明                        |
| --------- | ------------ | --------------------------- |
| Token     | `"token"`    | 共享令牌认证（默认）        |
| Password  | `"password"` | 密码认证                    |
| None      | `"none"`     | 无认证（不推荐）            |
| Tailscale | 自动         | 通过 Tailscale 网络身份认证 |

### 7.2 connect 握手中的认证决策流程

```
connect 请求到达
  ↓
1. 检查 auth.token / auth.password / auth.bootstrapToken
  ↓
2. 检查 device 签名
  ↓ 解析 publicKey → 推导 deviceId
  ↓ 验证签名（Ed25519）
  ↓ 检查 nonce 匹配
  ↓
3. 查询设备配对状态
  ↓ 已配对 → 颁发/验证 deviceToken
  ↓ 未配对 → 检查自动配对策略
  ↓           → loopback 地址自动批准
  ↓           → 否则返回 PAIRING_REQUIRED
  ↓
4. 角色 + Scope 解析
  ↓ role: "operator" 或 "node"
  ↓ scopes: 根据配对记录或请求声明
  ↓
5. 构建 hello-ok 响应
  ↓ 含 snapshot（会话/健康/在线状态）
  ↓ 含 features（方法列表/事件列表）
  ↓ 含 auth（角色/scope/deviceToken）
```

### 7.3 角色与权限体系

```
角色体系
├── operator（操作者）
│   ├── operator.admin    → 配置修改、Agent 管理、渠道控制
│   ├── operator.read     → 状态查询、历史读取、模型列表
│   ├── operator.write    → Agent 执行、聊天、工具调用
│   ├── operator.approvals → 审批处理
│   └── operator.pairing  → 设备配对管理
│
└── node（设备节点）
    └── 专有方法            → node.event, node.invoke.result, skills.bins 等
```

> 源码依据：`src/gateway/role-policy.ts` 仅定义了 `operator` 和 `node` 两种角色；`src/gateway/methods/core-descriptors.ts` 为每个方法指定了 scope。

---

## 8. 不同客户端的接口差异

### 8.1 Operator 客户端（WebUI/CLI/SDK）

所有 operator 客户端使用 **完全相同的接口**，差异仅在于连接参数：

| 参数          | WebUI                 | CLI                | SDK                |
| ------------- | --------------------- | ------------------ | ------------------ |
| `client.id`   | `openclaw-control-ui` | `cli`              | `gateway-client`   |
| `client.mode` | `webchat`             | `cli`              | 可配置             |
| `role`        | `operator`            | `operator`         | 可配置             |
| `scopes`      | 全部 5 个             | 由配置决定         | 可配置             |
| `caps`        | `["tool-events"]`     | `["tool-events"]`  | 可配置             |
| 认证方式      | token + device        | token + device     | token              |
| 可调用方法    | 所有 operator 方法    | 所有 operator 方法 | 所有 operator 方法 |
| 可接收事件    | 根据 scope            | 根据 scope         | 根据 scope         |

**关键结论：Gateway 对所有 operator 客户端一视同仁，不存在硬编码的客户端类型判断。**

### 8.2 Node 客户端（macOS/iOS/Android）

Node 客户端的角色为 `node`，具有不同的方法和事件权限：

| 维度       | Node 客户端                                                             |
| ---------- | ----------------------------------------------------------------------- |
| `role`     | `node`                                                                  |
| 可调用方法 | `node.*`, `skills.bins`, `node.pending.*`                               |
| 可接收事件 | `voicewake.changed`, `voicewake.routing.changed`, `node.invoke.request` |
| 特殊能力   | 接收 Gateway 发起的 invoke 命令（如 camera.shoot）                      |

### 8.3 HTTP REST 客户端

通过 HTTP REST API 接入的客户端不需要 WebSocket 握手：

```bash
# OpenAI 兼容接口
curl -X POST http://127.0.0.1:18789/v1/chat/completions \
  -H "Authorization: Bearer your-token" \
  -H "Content-Type: application/json" \
  -d '{"model": "default", "messages": [{"role": "user", "content": "你好"}]}'

# 健康探针（无需认证）
curl http://127.0.0.1:18789/health/ready
```

---

## 9. 二次开发技术细节

### 9.1 接入新 Operator 客户端——零改动清单

如果你要接入一个新的 operator 客户端（如自定义 Web App、桌面应用、移动端），**Gateway 端零改动**。你只需要在客户端侧实现：

| #   | 必须实现               | 技术要求                                     |
| --- | ---------------------- | -------------------------------------------- |
| 1   | WebSocket 连接         | 连接到 `ws[s]://host:18789`                  |
| 2   | 等待 connect.challenge | 监听 event 帧，提取 nonce                    |
| 3   | 发送 connect 请求      | 包含 protocol/client/role/scopes/caps/auth   |
| 4   | 解析 hello-ok          | 存储 deviceToken、获取 features/snapshot     |
| 5   | 请求-响应关联          | 用 UUID 作为请求 id，匹配响应                |
| 6   | 事件监听               | 处理 event 帧（agent/chat/presence/tick 等） |
| 7   | 序列号跟踪             | 跟踪 seq 检测事件丢失                        |

可选但推荐：

| #   | 推荐实现            | 说明                           |
| --- | ------------------- | ------------------------------ |
| 8   | Ed25519 设备身份    | 实现设备配对，获得 deviceToken |
| 9   | 自动重连 + 指数退避 | 网络断线恢复                   |
| 10  | deviceToken 持久化  | 后续免重复配对                 |

### 9.2 核心交互场景代码示例

#### 场景 1：发起 Agent 对话

```
Client → Gateway: { type:"req", id:"1", method:"agent", params:{
  message: "帮我写一个Python脚本",
  sessionKey: "main",
  idempotencyKey: "uuid-xxx"
}}

Gateway → Client: { type:"res", id:"1", ok:true, payload:{
  runId: "run-abc", status: "accepted"
}}

Gateway → Client: { type:"event", event:"agent", payload:{
  runId: "run-abc", type: "delta", text: "好的，"
}}

Gateway → Client: { type:"event", event:"agent", payload:{
  runId: "run-abc", type: "delta", text: "我来帮你写..."
}}

Gateway → Client: { type:"res", id:"1", ok:true, payload:{
  runId: "run-abc", status: "ok", summary: "completed"
}}
```

#### 场景 2：查询会话列表

```
Client → Gateway: { type:"req", id:"2", method:"sessions.list", params:{} }

Gateway → Client: { type:"res", id:"2", ok:true, payload:{
  sessions: [
    { key: "main", agentId: "default", updatedAt: "...", ... },
    { key: "agent:coder", agentId: "coder", ... }
  ]
}}
```

#### 场景 3：中止运行中的 Agent

```
Client → Gateway: { type:"req", id:"3", method:"sessions.abort", params:{
  key: "main", runId: "run-abc"
}}

Gateway → Client: { type:"res", id:"3", ok:true, payload:{} }
```

### 9.3 配置要点

Gateway 侧配置文件 `~/.openclaw/openclaw.json`：

```json
{
  "gateway": {
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "your-shared-token"
    },
    "controlUi": {
      "enabled": true,
      "dangerouslyDisableDeviceAuth": false
    },
    "http": {
      "endpoints": {
        "chatCompletions": { "enabled": true },
        "responses": { "enabled": true }
      }
    }
  }
}
```

| 配置项                                           | 说明                 | 二次开发建议                            |
| ------------------------------------------------ | -------------------- | --------------------------------------- |
| `gateway.port`                                   | 监听端口             | 默认 18789，可按需修改                  |
| `gateway.auth.mode`                              | 认证模式             | 开发阶段可用 `"none"`，生产用 `"token"` |
| `gateway.auth.token`                             | 共享令牌             | 所有客户端使用相同令牌                  |
| `gateway.controlUi.dangerouslyDisableDeviceAuth` | 禁用设备配对         | 开发阶段设 `true`，免去设备签名实现     |
| `gateway.http.endpoints.chatCompletions.enabled` | 启用 OpenAI 兼容 API | 如需 REST 接入设 `true`                 |

### 9.4 错误码参考

| 错误码                          | 含义                   | 处理建议               |
| ------------------------------- | ---------------------- | ---------------------- |
| `INVALID_REQUEST`               | 请求格式错误或权限不足 | 检查参数和 scope       |
| `UNAVAILABLE`                   | 方法暂不可用（启动中） | 延迟重试               |
| `AUTH_TOKEN_MISSING`            | 缺少认证令牌           | 提供 auth.token        |
| `AUTH_TOKEN_MISMATCH`           | 令牌不匹配             | 检查令牌配置           |
| `PROTOCOL_MISMATCH`             | 协议版本不兼容         | 升级客户端协议版本     |
| `PAIRING_REQUIRED`              | 需要设备配对           | 完成配对或禁用设备认证 |
| `DEVICE_AUTH_SIGNATURE_INVALID` | 设备签名校验失败       | 检查 Ed25519 签名实现  |

### 9.5 两种接入路线对比

| 维度         | 路线 A：WebSocket JSON-RPC  | 路线 B：HTTP REST API             |
| ------------ | --------------------------- | --------------------------------- |
| **协议**     | WebSocket 长连接            | HTTP 请求-响应                    |
| **实时事件** | ✅ 服务端推送 event 帧      | ❌ 需轮询或 SSE                   |
| **流式输出** | ✅ agent delta 事件实时到达 | ✅ SSE 流（/v1/chat/completions） |
| **完整功能** | ✅ 100+ RPC 方法            | ⚠️ 仅部分 HTTP 端点               |
| **会话管理** | ✅ 完整                     | ⚠️ 有限                           |
| **设备配对** | ✅ 支持                     | ❌ 不支持                         |
| **复杂度**   | 较高（需实现握手协议）      | 较低（标准 HTTP）                 |
| **适用场景** | 完整客户端、长期交互        | 简单集成、脚本调用                |

**建议：如果需要完整功能（会话管理、实时事件、设备能力调用），选路线 A（WebSocket）。如果只需要简单的聊天/模型调用，选路线 B（HTTP REST）。**

---

## 10. 架构层面的关键设计原则

### 10.1 客户端无关性

Gateway 通过 **role + scope** 体系而非 **client ID** 来控制权限。这意味着：

- 无论 `client.id` 是什么，只要 role/scope 正确就能使用全部功能
- Gateway 代码中没有 `if (clientId === "xxx")` 的硬编码分支
- 唯一的微小感知是 `isWebchatConnect()`（影响消息来源标记）和 `mode === "backend"`（影响内部运行时交接权限），均不影响核心功能

### 10.2 方法注册表可扩展

方法注册表支持三种来源，热插拔：

- **Core handlers**：编译时确定的核心方法
- **Plugin handlers**：插件运行时注册的方法
- **Aux handlers**：辅助扩展方法

### 10.3 事件推送基于权限

广播器内置 scope 守卫，自动过滤无权限的客户端，无需业务代码显式检查。

### 10.4 配置热加载

配置变更通过文件监听 + diff 分析 + reload plan 实现热更新，无需重启 Gateway。涉及文件：

- `config-reload.ts` — 监听 + 构建 reload plan
- `config-reload-plan.ts` — 分类变更（热更新 vs 需重启）
- `server-reload-handlers.ts` — 执行热更新（auth 轮换、插件重载、Cron 重建）

---

## 11. 守护进程与进程管理

Gateway 不仅是 WS/HTTP 服务，还包含完整的守护进程生命周期管理和子进程控制。

### 11.1 源码目录

```
src/daemon/                               ← 守护进程管理
├── gateway-entrypoint.ts                 ← Gateway 守护进程入口
├── launchd-plist.ts                      ← macOS launchd plist 生成
├── launchd-recovery.ts                   ← launchd 崩溃恢复
├── launchd-restart-handoff.ts            ← 重启时的状态交接
├── lifecycle.ts                          ← 守护进程生命周期 (install/start/stop/uninstall)
├── container-context.ts                  ← 容器环境感知 (Docker/K8s)
├── diagnostics.ts                        ← 守护进程诊断
├── inspect.ts                            ← 运行时检查
├── cmd-argv.ts                           ← 命令行参数构建
├── cmd-set.ts                            ← launchd 命令集
├── exec-file.ts                          ← 可执行文件定位
└── future-config-guard.ts                ← 未来配置兼容守卫

src/process/                              ← 进程管理
├── exec.ts                               ← 子进程执行
├── command-queue.ts                      ← 命令车道队列 (跨 L4 调度层共享)
├── kill-tree.ts                          ← 进程树终止 (含子进程)
├── spawn-utils.ts                        ← spawn 工具函数
├── lanes.ts                              ← 车道枚举 (main/cron/subagent/nested)
├── linux-oom-score.ts                    ← Linux OOM 分数调节
├── child-process-bridge.ts               ← 子进程通信桥
├── windows-command.ts                    ← Windows 命令兼容
└── supervisor/                           ← 进程监管
```

### 11.2 守护进程生命周期

```
openclaw gateway --daemon
  │
  ├─ 1. daemon/lifecycle.ts → install (写入 launchd plist)
  ├─ 2. daemon/lifecycle.ts → start (launchctl load)
  ├─ 3. daemon/gateway-entrypoint.ts → 入口
  │     ├─ container-context.ts → 检测容器环境
  │     └─ future-config-guard.ts → 配置兼容检查
  ├─ 4. Gateway 运行中
  │     ├─ launchd-recovery.ts → 崩溃自动重启
  │     └─ launchd-restart-handoff.ts → 重启时保持状态
  └─ 5. daemon/lifecycle.ts → stop (launchctl unload)
```

### 11.3 进程管理关键接口

```typescript
// src/process/exec.ts — 子进程执行
spawnChildProcess(command, args, opts): ChildProcess

// src/process/kill-tree.ts — 进程树终止
killTree(pid, signal): Promise<void>

// src/process/linux-oom-score.ts — OOM 保护
adjustOomScore(pid, score): void  // score: -1000(禁用kill) ~ 1000(优先kill)
```

---

## 12. 设备配对系统

### 12.1 设计目标

设备配对提供了比 token 更强的安全保证——每个设备拥有 Ed25519 密钥对，通过挑战-应答协议证明身份。

### 12.2 源码目录

```
src/pairing/                              ← 配对核心
├── pairing-challenge.ts                  ← 挑战-应答协议
├── pairing-messages.ts                   ← 配对消息格式
├── pairing-store.ts                      ← 配对存储 (持久化)
├── pairing-store.types.ts                ← 配对存储类型
├── pairing-labels.ts                     ← 设备标签
├── setup-code.ts                         ← 设置码生成/验证
├── allow-from-store-file.ts              ← allow-from 持久化
└── allow-from-store-read.ts              ← allow-from 读取

extensions/device-pair/                   ← 配对插件 (CLI + UI)
├── index.ts                              ← 插件入口
├── pair-command-auth.ts                  ← 配对命令认证
├── pair-command-approve.ts               ← 配对批准命令
├── qr-image.ts                           ← QR 码生成
└── notify.ts                             ← 配对通知
```

### 12.3 配对流程

```
新设备首次连接
  │
  ├─ 1. Gateway 发送 connect.challenge { nonce }
  │
  ├─ 2. 设备发送 connect 请求
  │     ├─ device.id = SHA-256(publicKey).hex
  │     ├─ device.publicKey = Ed25519 公钥 (base64url)
  │     └─ device.signature = sign(v3|deviceId|clientId|...|nonce)
  │
  ├─ 3. Gateway 验证签名
  │     ├─ 通过 → 检查配对状态
  │     │   ├─ 已配对 → 颁发/验证 deviceToken
  │     │   └─ 未配对 → 返回 PAIRING_REQUIRED
  │     └─ 失败 → DEVICE_AUTH_SIGNATURE_INVALID
  │
  ├─ 4. 用户批准配对
  │     ├─ WebUI: devices.approve(requestId)
  │     ├─ CLI: pnpm openclaw devices approve --latest
  │     └─ setup-code: 输入预设的配对码
  │
  └─ 5. Gateway 颁发 deviceToken → 持久化到 pairing-store
        → 后续连接凭 deviceToken 免重复配对
```

### 12.4 跨端签名一致性

> 源码：`src/gateway/device-auth.ts`

签名 payload 格式在 TS/Swift/Kotlin 三端严格一致：

```
v3|deviceId|clientId|clientMode|role|scopes|signedAtMs|token|nonce|platform|deviceFamily
```

各端实现：

| 端         | 源文件                                                | 函数                                 |
| ---------- | ----------------------------------------------------- | ------------------------------------ |
| TypeScript | `src/gateway/device-auth.ts`                          | `buildDeviceAuthPayloadV3()`         |
| Swift      | `apps/shared/OpenClawKit/.../DeviceAuthPayload.swift` | `GatewayDeviceAuthPayload.buildV3()` |
| Kotlin     | `apps/android/.../DeviceAuthPayload.kt`               | `DeviceAuthPayload.buildV3()`        |

---

> 本文档基于源码静态分析得出。具体函数级行为请以源码与单元测试为最终参考。
