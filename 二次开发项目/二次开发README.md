# OpenClaw 二次开发 — 部署与启动指南

## 一、项目概述

**OpenClaw** 是一个多渠道个人 AI 助手平台，采用 **pnpm workspace** 的 monorepo 架构。

### 项目结构

```
My-OpenClaw/
├── src/                  # Gateway 后端核心代码（TypeScript）
├── ui/                   # Control UI 前端（Vite + Lit）
├── extensions/           # 通道插件扩展（Discord、Telegram、WhatsApp 等）
├── packages/             # 共享内部包
├── apps/                 # 移动端应用（Android/iOS）
├── scripts/              # 构建与开发辅助脚本
├── docs/                 # 文档
├── package.json          # 根 package.json（scripts + 依赖）
├── pnpm-workspace.yaml   # pnpm workspace 配置
└── openclaw.mjs          # CLI 入口
```

### 技术栈

| 层级            | 技术                                 |
| --------------- | ------------------------------------ |
| 后端 Gateway    | Node.js + TypeScript（tsx 直接运行） |
| 前端 Control UI | Vite 8 + Lit 3（Web Components）     |
| 包管理          | pnpm 11.1.0（monorepo workspace）    |
| 构建工具        | tsdown（后端）、Vite（前端）         |
| 测试            | Vitest                               |

---

## 二、首次部署（只需执行一次）

> 首次拉取项目到本地后，按以下步骤完成环境安装和初始化。完成后，后续启动只需参考"第三节"。

### 2.1 安装 Node.js（要求 >= 22.19.0，推荐 Node 24）

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc

# 安装并使用 Node 24
nvm install 24
nvm use 24

# 验证
node --version    # 应输出 v24.x.x
```

### 2.2 安装 pnpm（必须使用 pnpm，不支持 npm install）

```bash
npm install -g pnpm@11.1.0

# 验证
pnpm --version    # 应输出 11.1.0
```

### 2.3 安装项目依赖

```bash
cd /root/autodl-tmp/OpenClaw/My-OpenClaw
pnpm install
```

> 首次安装耗时约 5-10 分钟，会安装根目录、`ui/`、`packages/*`、`extensions/*` 所有 workspace 包的依赖。

### 2.4 初始化本地配置

```bash
pnpm openclaw setup
```

该命令会在 `~/.openclaw/` 下创建配置文件和工作空间目录。

### 2.5 构建前端 Control UI

```bash
pnpm ui:build
```

> 前端产物输出到 `dist/control-ui/`，Gateway 启动后从此目录托管 WebUI 页面。

---

## 三、后续启动（日常使用）

> 首次部署完成后，每次启动只需以下步骤。按顺序执行即可。

### 第 1 步：加载 Node.js 环境

每次新开终端时需要先加载 nvm：

```bash
export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

验证 Node 可用：

```bash
node --version    # 应输出 v24.x.x
```

> 如果已在 `~/.bashrc` 中自动加载 nvm（首次安装时会自动写入），可跳过此步。

### 第 2 步：配置 AI 模型和 API Key

编辑配置文件：

```bash
vi ~/.openclaw/openclaw.json
```

#### 接入 DeepSeek 模型（推荐）

OpenClaw **内建支持 DeepSeek**（`deepseek` 是内置 provider），默认 baseUrl 已预配置为 `https://api.deepseek.com`，无需额外填写。

**DeepSeek 可用模型：**

| model 值                     | 说明                                                       |
| ---------------------------- | ---------------------------------------------------------- |
| `deepseek/deepseek-v4-flash` | DeepSeek V4 Flash，速度快、成本低                          |
| `deepseek/deepseek-v4-pro`   | DeepSeek V4 Pro，推理能力更强                              |
| `deepseek/deepseek-chat`     | 旧版名称，将于 2026/07/24 弃用，等同于 v4-flash 非思考模式 |
| `deepseek/deepseek-reasoner` | 旧版名称，将于 2026/07/24 弃用，等同于 v4-flash 思考模式   |

> 推荐使用 `deepseek-v4-flash` 或 `deepseek-v4-pro`，旧版模型名即将弃用。

**获取 API Key：** 前往 [DeepSeek 开放平台](https://platform.deepseek.com/) 注册并创建 API Key。

**配置方式 A：使用环境变量（推荐）**

在启动 Gateway 之前，导出 API Key：

```bash
export DEEPSEEK_API_KEY="sk-xxxxxxxxxxxxxxxxxxxxxxxx"
```

然后在 `~/.openclaw/openclaw.json` 中指定模型为 DeepSeek：

```json
{
  "agents": {
    "defaults": {
      "model": "deepseek/deepseek-v4-flash",
      "workspace": "/root/.openclaw/workspace"
    }
  },
  "gateway": {
    "mode": "local",
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "<你的Gateway令牌>"
    },
    "controlUi": {
      "dangerouslyDisableDeviceAuth": true
    }
  },
  "meta": {
    "lastTouchedVersion": "2026.5.19"
  }
}
```

**配置方式 B：在配置文件中直接写入 API Key（推荐，免去每次导出环境变量）**

将 API Key 直接写入配置文件：

```json
{
  "models": {
    "providers": {
      "deepseek": {
        "apiKey": "sk-xxxxxxxxxxxxxxxxxxxxxxxx"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "deepseek/deepseek-v4-flash",
      "workspace": "/root/.openclaw/workspace"
    }
  },
  "gateway": {
    "mode": "local",
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "<你的Gateway令牌>"
    },
    "controlUi": {
      "dangerouslyDisableDeviceAuth": true
    }
  },
  "meta": {
    "lastTouchedVersion": "2026.5.19"
  }
}
```

> **注意：** 配置文件中包含真实 API Key 时，不要将其提交到 Git 仓库。

#### 其他模型提供商参考

OpenClaw 也支持以下内建提供商，通过对应的环境变量自动识别 API Key：

| 模型提供商 | 环境变量名          | model 值示例                         |
| ---------- | ------------------- | ------------------------------------ |
| DeepSeek   | `DEEPSEEK_API_KEY`  | `deepseek/deepseek-v4-flash`         |
| OpenAI     | `OPENAI_API_KEY`    | `openai/gpt-4o`                      |
| Anthropic  | `ANTHROPIC_API_KEY` | `anthropic/claude-sonnet-4-20250514` |
| Google     | `GOOGLE_API_KEY`    | `google/gemini-2.5-flash`            |
| Cerebras   | `CEREBRAS_API_KEY`  | `cerebras/llama-4-scout-17b`         |
| DashScope  | `DASHSCOPE_API_KEY` | `dashscope/qwen-max`                 |
| MiniMax    | `MINIMAX_API_KEY`   | `minimax/minimax-text-01`            |

### 第 3 步：配置 Gateway 令牌

Gateway 令牌用于保护 WebUI 和 API 访问。打开浏览器访问 WebUI 时会要求输入此令牌。

**生成一个随机令牌：**

```bash
openssl rand -hex 32
```

会输出类似：`18d6c3268e7889f436d59045b68c85f644ecb35e63117ca2c7b7c47d915665fc`

**将令牌写入配置文件：** 确保 `~/.openclaw/openclaw.json` 中 `gateway.auth` 部分如下：

```json
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "这里粘贴你生成的令牌"
    }
  }
}
```

> **请记住你设置的令牌**，后面打开 WebUI 时需要输入。

**认证模式说明：**

| mode         | 说明                                     |
| ------------ | ---------------------------------------- |
| `"token"`    | 令牌认证（推荐），WebUI 访问时需输入令牌 |
| `"none"`     | 无认证，仅限本地开发                     |
| `"password"` | 密码认证                                 |

### 第 4 步：启动 Gateway 后端

进入项目目录，启动 Gateway：

```bash
cd /root/autodl-tmp/OpenClaw/My-OpenClaw
pnpm openclaw gateway --port 18789 --verbose
```

看到以下输出表示启动成功：

```
[gateway] http server listening (8 plugins: ...)
[gateway] ready
```

> Gateway 启动后会在当前终端持续运行，**不要关闭这个终端窗口**。

### 第 5 步：打开 WebUI

在浏览器中访问：

```
http://localhost:18789
```

如果是远程服务器，使用服务器 IP：

```
http://<服务器IP>:18789
```

打开后会出现登录页面：

- **WebSocket URL**：默认 `ws://localhost:18789`，无需修改
- **网关令牌**：输入第 3 步设置的 Gateway 令牌
- **密码**：标注为"可选"，token 模式下**留空即可**，无需填写

点击"连接"即可进入 WebUI 控制台。

#### 常见问题：提示"需要设备配对"

如果连接后提示 **"需要设备配对 — device pairing required"**，这是 OpenClaw 的设备身份认证机制，浏览器首次连接需要被 Gateway 批准。

**解决方案 A：关闭设备认证（本地开发推荐）**

在 `~/.openclaw/openclaw.json` 的 `gateway` 中添加：

```json
{
  "gateway": {
    "controlUi": {
      "dangerouslyDisableDeviceAuth": true
    }
  }
}
```

添加后重启 Gateway 即可。上方的配置示例中已包含此字段。

**解决方案 B：手动批准设备（生产环境推荐）**

在 Gateway 运行的终端中，新开一个终端执行：

```bash
cd /root/autodl-tmp/OpenClaw/My-OpenClaw

# 查看待批准的设备列表
pnpm openclaw devices list

# 批准页面上显示的 requestId
pnpm openclaw devices approve <requestId>

# 或直接批准最新的请求
pnpm openclaw devices approve --latest
```

批准后在浏览器中重新点击"连接"。

### 第 6 步：验证（可选）

新开一个终端窗口，运行诊断命令检查状态：

```bash
cd /root/autodl-tmp/OpenClaw/My-OpenClaw
pnpm openclaw doctor
```

或发送一条测试消息：

```bash
pnpm openclaw agent --message "你好" --thinking high
```

---

## 四、停止服务

在 Gateway 运行的终端窗口中按 `Ctrl + C` 即可停止后端服务。

如需强制关闭残留进程：

```bash
# 查看占用端口的进程
ps aux | grep -E "gateway|openclaw" | grep -v grep

# 按 PID 强制关闭
kill -9 <PID>
```

---

## 五、常用命令速查

| 命令                                           | 说明                                          |
| ---------------------------------------------- | --------------------------------------------- |
| `pnpm openclaw gateway --port 18789 --verbose` | 启动 Gateway（后端 + WebUI）                  |
| `pnpm ui:build`                                | 重新构建前端（修改 ui/ 后执行）               |
| `pnpm ui:dev`                                  | 启动前端 Vite 开发服务器（端口 5173，热重载） |
| `pnpm gateway:watch`                           | Gateway 开发模式（后端代码热重载）            |
| `pnpm openclaw doctor`                         | 诊断检查                                      |
| `pnpm openclaw setup`                          | 重新初始化配置（一般不需要）                  |

---

## 六、配置文件参考

### 6.1 完整配置文件示例（DeepSeek）

文件位置：`~/.openclaw/openclaw.json`

```json
{
  "models": {
    "providers": {
      "deepseek": {
        "apiKey": "sk-你的DeepSeek API Key"
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "deepseek/deepseek-v4-flash",
      "workspace": "/root/.openclaw/workspace"
    }
  },
  "gateway": {
    "mode": "local",
    "port": 18789,
    "auth": {
      "mode": "token",
      "token": "你的Gateway令牌"
    },
    "controlUi": {
      "dangerouslyDisableDeviceAuth": true
    }
  },
  "meta": {
    "lastTouchedVersion": "2026.5.19"
  }
}
```

### 6.2 Agent 工作空间

```
~/.openclaw/workspace/
```

关键文件：

- `AGENTS.md` — Agent 指令注入
- `SOUL.md` — 人格与风格定义
- `TOOLS.md` — 工具说明注入
- `skills/<skill>/SKILL.md` — 技能定义

### 6.3 Gateway 日志

```bash
cat /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

---

## 七、注意事项

1. **必须使用 pnpm**：`npm install` 不受支持。
2. **Node 版本**：确保 `>=22.19.0`，推荐 Node 24。
3. **新终端需加载 nvm**：每次打开新终端，确保 `node` 命令可用。
4. **前端修改后需重新构建**：修改 `ui/` 目录代码后需执行 `pnpm ui:build`，或使用 `pnpm ui:dev` 开发模式。
5. **API Key 安全**：不要将包含真实 API Key 的配置文件提交到 Git。
6. **模型配置字段是 `agents.defaults.model`**：不要写成顶层的 `"agent": {"model": ...}`，这会导致 `Invalid input` 启动报错。配置文件使用 `.strict()` 校验，不允许未定义的顶层字段。
7. **配置文件必须包含 `meta` 字段**：建议保留 `"meta": {"lastTouchedVersion": "2026.5.19"}`，缺少可能触发 `missing-meta-vs-last-good` 警告。

---

## 八、官方文档

- [Getting Started](https://docs.openclaw.ai/start/getting-started)
- [Configuration](https://docs.openclaw.ai/gateway/configuration)
- [Models](https://docs.openclaw.ai/concepts/models)
- [Security](https://docs.openclaw.ai/gateway/security)
- [Channels](https://docs.openclaw.ai/channels)
- [Architecture](https://docs.openclaw.ai/concepts/architecture)
- [FAQ](https://docs.openclaw.ai/help/faq)
