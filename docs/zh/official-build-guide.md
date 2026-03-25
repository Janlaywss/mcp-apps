# 构建一个 MCP App

构建可交互 UI 应用的入门指南。

---

## 前置要求

需要 [Node.js](https://nodejs.org/en/download) 18 或更高版本。建议提前了解 [MCP 工具](https://modelcontextprotocol.io/specification/latest/server/tools)和[资源](https://modelcontextprotocol.io/specification/latest/server/resources)的概念，因为 MCP Apps 同时结合了这两种原语。有 [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) 使用经验，将有助于更好地理解服务端模式。

---

## 快速开始

最快的方式是使用带有 MCP Apps skill 的 AI 编程助手。如果你希望手动搭建项目，请跳至「手动搭建」章节。

### 使用 AI 编程助手

支持 Skills 的 AI 编程助手可以为你自动搭建完整的 MCP App 项目。Skills 是指令和资源的文件夹，当相关任务出现时，助手会自动加载它们，从而掌握创建 MCP Apps 等专项任务的方法。`create-mcp-app` skill 包含了架构指导、最佳实践和可运行的示例，助手会利用这些来生成你的项目。

#### 第一步：安装 skill

如果你使用 Claude Code，可以直接通过以下命令安装：

```bash
/plugin marketplace add modelcontextprotocol/ext-apps
/plugin install mcp-apps@modelcontextprotocol-ext-apps
```

你也可以使用 [Vercel Skills CLI](https://skills.sh/) 在不同 AI 编程助手之间安装 skills：

```bash
npx skills add modelcontextprotocol/ext-apps
```

或者，手动克隆 ext-apps 仓库来安装：

```bash
git clone https://github.com/modelcontextprotocol/ext-apps.git
```

然后将 skill 复制到对应助手的目录下：

| 助手 | macOS / Linux | Windows |
|------|---------------|---------|
| Claude Code | `~/.claude/skills/` | `%USERPROFILE%\.claude\skills\` |
| VS Code / GitHub Copilot | `~/.copilot/skills/` | `%USERPROFILE%\.copilot\skills\` |
| Gemini CLI | `~/.gemini/skills/` | `%USERPROFILE%\.gemini\skills\` |
| Cline | `~/.cline/skills/` | `%USERPROFILE%\.cline\skills\` |
| Goose | `~/.config/goose/skills/` | `%USERPROFILE%\.config\goose\skills\` |
| Codex | `~/.codex/skills/` | `%USERPROFILE%\.codex\skills\` |
| Cursor | `~/.cursor/skills/` | `%USERPROFILE%\.cursor\skills\` |

> 以上列表并不完整，其他助手可能将 skills 放在不同位置，请查阅各自的文档。

以 Claude Code 为例，可以全局安装（在所有项目中可用）：

```bash
cp -r ext-apps/plugins/mcp-apps/skills/create-mcp-app ~/.claude/skills/create-mcp-app
```

或者只为当前项目安装（复制到项目目录下的 `.claude/skills/`）：

```bash
mkdir -p .claude/skills && cp -r ext-apps/plugins/mcp-apps/skills/create-mcp-app .claude/skills/create-mcp-app
```

验证 skill 是否安装成功，可以询问你的 AI 助手："你有哪些 skills？"——你应该能看到 `create-mcp-app` 出现在列表中。

#### 第二步：创建应用

直接告诉 AI 编程助手你要构建什么：

```
Create an MCP App that displays a color picker
```

助手会识别出 `create-mcp-app` skill 与此任务相关，加载其指令，然后自动搭建包含服务端、UI 和配置文件的完整项目。

#### 第三步：运行应用

```bash
npm install && npm run build && npm run serve
```

> 运行前确认你已切换到 app 文件夹目录。

#### 第四步：测试应用

按照下方「测试你的 MCP App」章节中的步骤进行测试。对于颜色选择器示例，开启一个新会话，让 Claude 给你一个颜色选择器即可。

---

### 手动搭建

如果你没有使用 AI 编程助手，或者希望理解搭建过程，请按以下步骤操作。

#### 第一步：创建项目结构

一个典型的 MCP App 项目将服务端代码与 UI 代码分开：

```
my-mcp-app/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── server.ts
├── mcp-app.html
└── src/
    └── mcp-app.ts
```

服务端负责注册工具并提供 UI 资源。UI 资源最终会在一个默认拒绝所有 CSP 配置的沙箱 iframe 中渲染。如果你的应用有 CSS 和 JS 资源，需要[配置 CSP](https://apps.extensions.modelcontextprotocol.io/api/documents/Patterns.html#configuring-csp-and-cors)，或者使用 `vite-plugin-singlefile` 这类工具将所有资源打包进 HTML，本教程采用后者。

#### 第二步：安装依赖

```bash
npm install @modelcontextprotocol/ext-apps @modelcontextprotocol/sdk
npm install -D typescript vite vite-plugin-singlefile express cors @types/express @types/cors tsx
```

`ext-apps` 包为服务端（注册工具和资源）和客户端（UI 与宿主通信用的 `App` 类）都提供了工具函数。这里使用带 `vite-plugin-singlefile` 插件的 Vite 将 UI 和资源打包进单个 HTML 文件，这是可选的——你完全可以使用其他打包工具或不打包，只要[配置好 CSP](https://apps.extensions.modelcontextprotocol.io/api/documents/Patterns.html#configuring-csp-and-cors) 即可。

#### 第三步：配置项目

**`package.json`**

`"type": "module"` 启用 ES 模块语法。`build` 脚本使用 `INPUT` 环境变量告诉 Vite 要打包哪个 HTML 文件。`serve` 脚本使用 `tsx` 运行 TypeScript 服务端。

```json
{
  "type": "module",
  "scripts": {
    "build": "INPUT=mcp-app.html vite build",
    "serve": "npx tsx server.ts"
  }
}
```

#### 第四步：实现项目

项目结构和配置就绪后，继续阅读下方「实现 MCP App」章节来实现服务端和 UI。

---

## 实现 MCP App

以下我们构建一个显示当前服务器时间的简单应用，演示完整模式：注册带 UI 元数据的工具、将打包后的 HTML 作为资源提供、构建与服务端通信的 UI。

### 服务端实现

服务端需要做两件事：注册一个包含 `_meta.ui.resourceUri` 字段的工具，以及注册一个提供打包 HTML 的资源处理器。完整的服务端文件如下：

```typescript
// server.ts
console.log("Starting MCP App server...");

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import {
  registerAppTool,
  registerAppResource,
  RESOURCE_MIME_TYPE,
} from "@modelcontextprotocol/ext-apps/server";
import cors from "cors";
import express from "express";
import fs from "node:fs/promises";
import path from "node:path";

const server = new McpServer({
  name: "My MCP App Server",
  version: "1.0.0",
});

// ui:// scheme 告诉宿主这是一个 MCP App 资源。
// 路径结构任意，按你的 app 组织即可。
const resourceUri = "ui://get-time/mcp-app.html";

// 注册返回当前时间的工具
registerAppTool(
  server,
  "get-time",
  {
    title: "Get Time",
    description: "Returns the current server time.",
    inputSchema: {},
    _meta: { ui: { resourceUri } },
  },
  async () => {
    const time = new Date().toISOString();
    return {
      content: [{ type: "text", text: time }],
    };
  },
);

// 注册提供打包 HTML 的资源
registerAppResource(
  server,
  resourceUri,
  resourceUri,
  { mimeType: RESOURCE_MIME_TYPE },
  async () => {
    const html = await fs.readFile(
      path.join(import.meta.dirname, "dist", "mcp-app.html"),
      "utf-8",
    );
    return {
      contents: [
        { uri: resourceUri, mimeType: RESOURCE_MIME_TYPE, text: html },
      ],
    };
  },
);

// 通过 HTTP 暴露 MCP 服务端
const expressApp = express();
expressApp.use(cors());
expressApp.use(express.json());

expressApp.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
    enableJsonResponse: true,
  });
  res.on("close", () => transport.close());
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

expressApp.listen(3001, (err) => {
  if (err) {
    console.error("Error starting server:", err);
    process.exit(1);
  }
  console.log("Server listening on http://localhost:3001/mcp");
});
```

关键部分说明：

- **`resourceUri`**：`ui://` scheme 告诉宿主这是一个 MCP App 资源，路径结构任意。
- **`registerAppTool`**：注册带 `_meta.ui.resourceUri` 字段的工具。当宿主调用这个工具时，UI 会被获取并渲染，工具结果在到达后被推送给 UI。
- **`registerAppResource`**：当宿主请求 UI 资源时，提供打包后的 HTML。
- **Express 服务器**：将 MCP 服务端通过 HTTP 暴露在 3001 端口。

### UI 实现

UI 由一个 HTML 页面和一个使用 `App` 类与宿主通信的 TypeScript 模块组成。

**`mcp-app.html`**

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Get Time App</title>
  </head>
  <body>
    <p>
      <strong>Server Time:</strong>
      <code id="server-time">Loading...</code>
    </p>
    <button id="get-time-btn">Get Server Time</button>
    <script type="module" src="/src/mcp-app.ts"></script>
  </body>
</html>
```

**`src/mcp-app.ts`**

```typescript
import { App } from "@modelcontextprotocol/ext-apps";

const serverTimeEl = document.getElementById("server-time")!;
const getTimeBtn = document.getElementById("get-time-btn")!;

const app = new App({ name: "Get Time App", version: "1.0.0" });

// 与宿主建立通信
app.connect();

// 处理宿主推送的初始工具结果
app.ontoolresult = (result) => {
  const time = result.content?.find((c) => c.type === "text")?.text;
  serverTimeEl.textContent = time ?? "[ERROR]";
};

// 用户交互时主动调用工具
getTimeBtn.addEventListener("click", async () => {
  const result = await app.callServerTool({
    name: "get-time",
    arguments: {},
  });
  const time = result.content?.find((c) => c.type === "text")?.text;
  serverTimeEl.textContent = time ?? "[ERROR]";
});
```

关键部分说明：

- **`app.connect()`**：与宿主建立通信。在应用初始化时调用一次。
- **`app.ontoolresult`**：当宿主向应用推送工具结果时触发的回调（例如工具首次被调用且 UI 渲染时）。
- **`app.callServerTool()`**：让应用主动调用服务端工具。注意每次调用都涉及一次网络往返，设计 UI 时需要妥善处理延迟。

`App` 类还提供了日志记录、打开 URL、以及用应用中的结构化数据更新模型上下文等方法，详见[完整 API 文档](https://apps.extensions.modelcontextprotocol.io/api/)。

---

## 测试你的 MCP App

构建 UI 并启动本地服务器：

```bash
npm run build && npm run serve
```

默认配置下，服务器运行在 `http://localhost:3001/mcp`。但要看到应用渲染，你需要一个支持 MCP Apps 的 MCP 宿主。以下是几种选择。

### 使用 Claude 测试

[Claude](https://claude.ai/)（Web 版）和 [Claude Desktop](https://claude.ai/download) 都支持 MCP Apps。本地开发时，需要将你的服务器暴露到公网。可以在本地运行 MCP 服务器，并用 `cloudflared` 进行内网穿透。

在独立终端中运行：

```bash
npx cloudflared tunnel --url http://localhost:3001
```

复制生成的 URL（如 `https://random-name.trycloudflare.com`），然后在 Claude 中添加为[自定义连接器](https://support.anthropic.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp)：点击你的头像 → 设置 → Connectors → 添加自定义连接器。

> 自定义连接器需要 Claude 付费计划（Pro、Max 或 Team）。

### 使用 basic-host 测试

`ext-apps` 仓库包含一个用于开发测试的宿主。克隆仓库并安装依赖：

```bash
git clone https://github.com/modelcontextprotocol/ext-apps.git
cd ext-apps/examples/basic-host
npm install
```

从 `ext-apps/examples/basic-host/` 运行 `npm start` 启动测试界面。连接到指定服务器（如你正在开发的服务器），可以通过内联传入 `SERVERS` 环境变量：

```bash
SERVERS='["http://localhost:3001/mcp"]' npm start
```

访问 `http://localhost:8080`，你会看到一个简单的界面，可以选择工具并调用它。当你调用工具时，宿主获取 UI 资源并在沙箱 iframe 中渲染，你可以与应用交互并验证工具调用是否正常工作。

---

## 如果你已经有 Module Federation 项目

上面介绍的是官方推荐的完整构建方案——适合从零开始搭建一个独立的 MCP App。

但如果你的团队**已经在用 Module Federation**，按这套流程做会遇到一个结构性问题：

```
你的前端项目                    MCP App Server
┌──────────────────┐           ┌──────────────────────┐
│  DeployWizard    │  ← 需要   │  deploy-wizard.html   │  ← 同一个组件
│  IncidentForm    │    单独    │  incident-form.html   │    打包了两份
│  UserLookup      │    抽出、  │  user-lookup.html     │
│                  │    构建、  │                      │
└──────────────────┘    部署   └──────────────────────┘
  你的 CI/CD 流水线              与主项目完全平行的
                                 "影子流水线"
```

每次组件变更，你都要走两套构建和部署。如果有 10 个工具，就要维护 10 个独立 bundle。React 等共享依赖也会被重复打包进每个 HTML。

**`@module-federation/mcp-apps` 反转了这个方向**：Server 只持有一份轻量配置文件（`mcp_apps.json`），指向你 CDN 上已有的 MF 远程。UI 留在它本来的地方，MCP Server 零 UI 代码。

```json
// mcp_apps.json — 这就是全部的"MCP 侧"配置
{
  "remotes": [
    { "name": "@my-org/ops-tools", "baseUrl": "https://cdn.example.com/ops-tools/2.4.1" }
  ],
  "tools": [
    { "name": "deploy_wizard", "remote": "@my-org/ops-tools", "module": "./DeployWizard",
      "description": "三步交互式部署 UI" },
    { "name": "incident_form", "remote": "@my-org/ops-tools", "module": "./IncidentForm",
      "description": "打开带有预填上下文的故障报告表单" }
  ]
}
```

| | 官方手动构建方案 | `@module-federation/mcp-apps` |
|---|---|---|
| **适合场景** | 新项目，从零搭建 MCP App | 已有 MF 项目，快速接入 MCP |
| **新增工具** | 抽取组件 → 重新构建 → 重新部署 Server | 在 `mcp_apps.json` 加一行 |
| **更新 UI** | 重新构建 + 重新部署 Server | 部署到 CDN，更新版本号 |
| **共享依赖** | 每个 bundle 各自打包 React | 通过 MF shared scope 共享 |
| **Server 代码量** | 需要自己写路由、URI 生成、HTML 模板 | 零代码，`npx` 启动 |

详见：[Module Federation MCP Apps 完整指南](../blog/blog-introducing-mcp-apps.zh-CN.mdx)

---

## 延伸阅读

- [完整 API 文档](https://apps.extensions.modelcontextprotocol.io/api/)
- [MCP Apps 规范](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx)
- [CSP 与 CORS 配置](https://apps.extensions.modelcontextprotocol.io/api/documents/Patterns.html#configuring-csp-and-cors)
- [示例仓库](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples)
- [反馈与讨论](https://github.com/modelcontextprotocol/ext-apps/issues)
