# 05 · 自建 Agent 接入 `@module-federation/mcp-apps`

本文面向**已有自己 Agent 服务**、希望接入 Module Federation MCP Apps 的开发者。  
如果你使用的是 Claude Desktop、VS Code GitHub Copilot、Goose 等原生支持 MCP Apps 的客户端，**无需阅读本文** — 那些宿主直接处理 UI 渲染，只需配置 `mcp_apps.json` 后将 MCP Server 注册到客户端即可。

> ⭐ 觉得有用？欢迎在 GitHub 给我们点个 Star：**[module-federation/mcp-apps](https://github.com/module-federation/mcp-apps)**

---

## 适用场景

| 情况 | 是否需要本文 |
|------|-------------|
| 使用 Claude Desktop / VS Code / Goose（STDIO 模式） | ❌ 不需要，宿主原生支持 |
| **自建 Agent（任意框架）+ 自建 Web 前端** | ✅ **本文** |

---

## 架构概览

原生 MCP 宿主（Claude Desktop 等）可以直接调用 `resources/read` 并渲染 HTML。  
自建 B/S 架构的 Agent 面临一个根本问题：**MCP Client 在服务端，iframe 在浏览器端**，二者需要一条桥接通道。

```
用户输入
    │
    ▼
[Server] 你的 Agent（LangGraph ReAct / 任意框架）
    │  LLM 根据 tool.description 决定调用 UI 工具
    ▼
[Server] MCP Client → POST /mcp → MCP Server 执行工具
    │  CallToolResult 携带 JSON 配置（已含 MF remote/module 信息）
    ▼  on_tool_end / tool_call 回调
[Server] UI 增强层（你来实现）
    │  ① 检测 tool 元数据中的 _meta.ui.resourceUri
    │  ② 用 mcpServerBaseUrl 拼装 iframeUrl
    │  ③ 包装成 render_ui_resource 消息推送给前端（SSE / WebSocket）
    ▼  推流
[Browser] 你的前端
    │  解析 render_ui_resource → 渲染 <iframe>
    ▼  iframe 加载完毕
[iframe] MCP App（框架内置 AppBridge）
    │  postMessage 接收 tool args + CallToolResult
    ▼  动态 import MF 远程组件
    用户看到你的 React 组件 ✅
```

核心逻辑只有两步：**检测 UI 工具** → **推给前端**。iframeUrl 直接由 `mcpServerBaseUrl` 拼装，无需 `resources/read`。

---

## 前置条件

- 部署好的 Module Federation 远程组件（或本地 `http://localhost:8080`）
- `mcp_apps.json` 配置文件（见下文）
- 你的 Agent 服务可以连接 MCP Server（HTTP 模式）
- 你的 Web 前端可以渲染任意 HTML 到 `<iframe>`

---

## 第一步 — 编写 `mcp_apps.json`

告诉 MCP Server 暴露哪些 MF 组件作为工具。

### 使用 AI Skill 自动生成（推荐）

项目提供了 AI Skill，可以自动检测你的 Module Federation 配置文件（`module-federation.config.ts` / `rspack.config.js` 等），解析 `name` 和 `exposes`，自动生成 `mcp_apps.json`。

将 [`skill/generate-mcp-apps-config.md`](../../skill/generate-mcp-apps-config.md) 作为上下文提供给任意 AI 编码工具（Claude、Cursor、GitHub Copilot、Windsurf 等），然后说：

```
按照 skill/generate-mcp-apps-config.md 的指引，为我的项目生成 mcp_apps.json
```

### 手动编写

```jsonc
{
  // remotes：声明你的 MF 远程包，每个 remote 对应一个 CDN 部署的 MF 包
  "remotes": [
    {
      // name：MF 包名，需与 module-federation.config.ts 中的 name 字段一致
      "name": "@my-org/ops-tools",
      // baseUrl：MF 包的 CDN 根路径，Server 会从这里加载 remoteEntry.js
      "baseUrl": "https://cdn.example.com/ops-tools/2.4.1/mf-manifest.json",
      // csp：Content Security Policy 配置，控制 iframe 内允许访问的域名
      "csp": {
        // connectDomains：iframe 内 fetch/XHR 允许的域名（包含 CDN 和 API 域名）
        "connectDomains": ["cdn.example.com", "api.example.com"],
        // resourceDomains：iframe 内 script/img 等资源允许加载的域名
        "resourceDomains": ["cdn.example.com"]
      }
    }
  ],
  // tools：声明暴露给 LLM 的工具列表，每个工具对应一个 MF 组件
  "tools": [
    {
      // name：工具唯一标识，LLM 通过此名称调用工具（建议使用 snake_case）
      "name": "deploy_wizard",
      // title：工具的展示名称（可选）
      "title": "部署向导",
      // description：告诉 LLM 什么场景下应该调用这个工具，越清晰越好
      "description": "当用户想要部署应用时调用此工具，展示可视化部署流程。",
      // inputSchema：工具的输入参数定义（JSON Schema），LLM 根据此生成调用参数
      "inputSchema": {
        "type": "object",
        "properties": {
          "appId": { "type": "string", "description": "要部署的应用 ID" },
          "env":   { "type": "string", "enum": ["staging", "production"] }
        },
        "required": []
      },
      // remote：此工具使用哪个 remote，需与上方 remotes[].name 对应
      "remote": "@my-org/ops-tools",
      // module：MF expose 路径，对应 module-federation.config.ts 中 exposes 的 key
      "module": "./DeployWizard",
      // exportName：组件的导出名称，默认为 "default"（可选）
      "exportName": "default"
    }
  ]
}
```

> 多步骤向导（step1 → step2 → step3）：每个步骤的 `inputSchema` 需声明上一步通过 `sendMessage` 传入的字段，并在 `description` 中标注"请勿主动调用"防止 LLM 随意触发。

---

## 第二步 — 部署 MCP Server（HTTP 模式）

HTTP 模式需要**将 MCP Server 部署为一个长运行的 Node.js 服务**，而不是像 STDIO 模式那样作为本地子进程启动。我们提供了开箱即用的项目模板：

**[module-federation-mcp-starter](../../module-federation-mcp-starter)**

克隆或复制该目录，它已包含：
- 预配置的 `package.json` 和启动脚本
- `mcp_apps.json` 占位文件（替换为你的配置）
- `MF_MCP_BASE_URL` 环境变量支持（用于生产环境的公网域名）

```bash
cd module-federation-mcp-starter

# 安装依赖
pnpm install

# 本地开发
pnpm run start

# 生产部署（指定公网域名，供前端 iframe 访问）
MF_MCP_BASE_URL=https://your-mcp-server.example.com pnpm run start
```

启动后暴露：
- `POST /mcp` — 标准 MCP Streamable HTTP 端点（`@modelcontextprotocol/sdk` 默认使用）
- `GET /static/mcp-app-shell.html` — 极简 iframe 壳（~530 B，从 `/static/` 异步加载 JS）
- `GET /static/mcp-app.html` — 全量内联 HTML（~686 KB，无网络依赖的备用方案）

> `MF_MCP_BASE_URL` 决定了 shell HTML 中 JS/CSS 的加载基路径。本地开发时默认为 `http://localhost:<port>`，生产部署时**必须设置为公网可访问地址**，否则浏览器加载 iframe 时会因为找不到静态资源而白屏。

验证是否可用：

```bash
curl http://localhost:3001/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

---

## 第三步 — 服务端：连接 MCP Client 并注册工具到 LLM

> 本步骤是标准的 MCP + Agent 集成流程，**与 Module Federation MCP Apps 方案无关**。以 LangChain + `@modelcontextprotocol/sdk` 为例说明，其他框架参照各自的 MCP 适配方式即可。

### 连接 MCP Client

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

const client = new Client({ name: 'my-agent', version: '1.0.0' });
await client.connect(
  new StreamableHTTPClientTransport(new URL('http://localhost:3001/mcp'))
);

// ⚠️ 与 MCP Apps 方案相关：工具定义中携带 _meta.ui 元数据，后续步骤需要用它判断是否为 UI 工具
const { tools } = await client.listTools();
```

### 将工具注册到 LLM（LangChain 示例）

```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';

// 使用任意 MCP → LangChain 适配方案，例如 @h1deya/langchain-mcp-tools
// 或手动将 tools[].inputSchema 转换为 Zod schema 后包装为 DynamicStructuredTool
const langchainTools = tools.map((tool) => new DynamicStructuredTool({
  name: tool.name,
  description: tool.description ?? '',
  schema: jsonSchemaToZod(tool.inputSchema ?? {}),
  func: async (args) => JSON.stringify(await client.callTool({ name: tool.name, arguments: args })),
  // ⚠️ 与 MCP Apps 方案相关：保留 _meta，下一步 UI 增强层需要从中读取 resourceUri
  metadata: { _meta: (tool as any)._meta },
}));

const agent = createReactAgent({ llm, tools: langchainTools });
```

> 注册方式取决于你的 Agent 框架，LangGraph / AutoGen / CrewAI 等各有不同。无论使用哪种框架，**需要确保工具的 `_meta` 字段被透传**，否则下一步无法识别 UI 工具。

---

## 第四步 — 服务端：UI 增强（核心）

> **🔑 MCP Apps 方案特有**  
> 标准 MCP 协议中工具返回结果后，Agent 框架会直接将文本交给 LLM 继续生成。MCP Apps 在此基础上扩展了一个约定：UI 工具的定义里携带 `_meta.ui.resourceUri`，指向服务端托管的 MCP App HTML。你的 Agent 服务需要在 `on_tool_end`（或等价回调）中**拦截这类工具的结果**，改为向前端推送渲染指令，而不是把 HTML 内容透传给 LLM。

这是自建 Agent 最关键的部分。在 **每次工具调用结束后**，检查该工具是否是 UI 工具，如果是——拼装 iframeUrl 并推给前端，继续流式输出 LLM 的后续文字。

```typescript
// agent/uiEnrichment.ts

export interface UiRenderPayload {
  /** mcp-app-shell.html 的公网可访问 URL，用于 iframe src */
  iframeUrl: string;
  /** CallToolResult 文本（JSON 字符串，MF 组件以此为 props 来源） */
  callToolResult: string;
  /** 传给工具的参数（透传给 MF 组件 props） */
  toolInput: Record<string, unknown>;
  /** 工具名称 */
  toolName: string;
}

/**
 * 在工具调用完毕后调用。
 * 如果该工具带有 UI 元数据，返回前端渲染所需的 payload；否则返回 null。
 *
 * @param toolName  刚执行完的工具名称
 * @param toolInput 调用该工具时传入的参数
 * @param callToolResultText  CallToolResult 的文本序列化（LangGraph event.data.output 转换后）
 * @param mcpTools  tools/list 返回的完整工具定义列表
 * @param mcpServerBaseUrl  MCP Server 的公网根地址，如 "http://localhost:3001"
 */
export function detectAndEnrichUiTool(
  toolName: string,
  toolInput: Record<string, unknown>,
  callToolResultText: string,
  mcpTools: any[],
  mcpServerBaseUrl: string,
): UiRenderPayload | null {
  // 找到工具定义，检查是否带 UI 元数据
  const toolDef = mcpTools.find((t) => t.name === toolName);
  const resourceUri: string | undefined = (toolDef as any)?._meta?.ui?.resourceUri;

  if (!resourceUri) {
    return null; // 普通工具，走正常文本流
  }

  // 使用轻量壳页面（~530 B），从 /static/ 异步加载 JS
  const iframeUrl = `${mcpServerBaseUrl.replace(/\/$/, '')}/static/mcp-app-shell.html`;

  return {
    iframeUrl,
    callToolResult: callToolResultText,
    toolInput,
    toolName,
  };
}
```

在你的 Agent 流式事件循环中接入：

```typescript
// agent/agentService.ts（LangGraph 示例）
import { HumanMessage } from '@langchain/core/messages';
import { detectAndEnrichUiTool } from './uiEnrichment';

async function* streamChat(
  userInput: string,
  agent: CompiledStateGraph<any, any>,
  mcpTools: any[],
) {
  const eventStream = agent.streamEvents(
    { messages: [new HumanMessage(userInput)] },
    { version: 'v2' },
  );

  for await (const event of eventStream) {
    if (event.event === 'on_tool_end') {
      const toolName: string = event.name;
      const toolInput = event.data.input ?? {};
      // LangGraph 将 CallToolResult 序列化为字符串放入 event.data.output
      const rawOutput = event.data.output;
      const callToolResultText =
        typeof rawOutput === 'string'
          ? rawOutput
          : typeof rawOutput?.content === 'string'
            ? rawOutput.content
            : JSON.stringify(rawOutput);

      const uiPayload = detectAndEnrichUiTool(
        toolName,
        toolInput,
        callToolResultText,
        mcpTools,
        'http://localhost:3001',
      );

      if (uiPayload) {
        // 推送 UI 渲染消息给前端，然后继续流式输出——LLM 在工具调用后可能还有跟进文字
        yield { type: 'render_ui', payload: uiPayload };
      }
    }

    // 正常文本 token
    if (event.event === 'on_chat_model_stream') {
      const chunk = event.data.chunk;
      const text = chunk?.content ?? '';
      if (text) {
        yield { type: 'text', payload: text };
      }
    }
  }
}
```

---

## 第五步 — 服务端：通过 SSE / WebSocket 推送给前端

> **🔑 MCP Apps 方案特有**  
> 你需要在现有的流式协议（SSE / WebSocket / 任意自定义格式）中新增一种消息类型 `render_ui_resource`，前端收到后触发 iframe 渲染。消息结构由你自行定义，但需包含 `iframeUrl`（iframe 加载地址）和 `callToolResult`（MF 组件的加载配置）两个核心字段。

将上一步的 `render_ui` 消息包装成你的流协议。以 SSE 为例：

```typescript
// server/routes/chat.ts（Koa / Express / Hono 均可）
async function streamChatHandler(req, res) {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  for await (const chunk of streamChat(req.body.message, agent, mcpTools)) {
    if (chunk.type === 'text') {
      res.write(`data: ${JSON.stringify({ type: 'text', text: chunk.payload })}\n\n`);
    } else if (chunk.type === 'render_ui') {
      // 推送 render_ui_resource 事件给前端
      res.write(`data: ${JSON.stringify({
        type: 'render_ui_resource',
        iframeUrl:       chunk.payload.iframeUrl,
        callToolResult:  chunk.payload.callToolResult,
        toolInput:       chunk.payload.toolInput,
        toolName:        chunk.payload.toolName,
      })}\n\n`);
    }
  }

  res.write('data: [DONE]\n\n');
  res.end();
}
```

---

## 第六步 — 前端：渲染 iframe

> **🔑 MCP Apps 方案特有**  
> MCP App 运行在 `<iframe>` 内，通过 AppBridge（基于 `postMessage`）与宿主页面通信。iframe 加载完毕后，宿主需主动将 `callToolResult`（MF 组件加载配置）和工具入参推入 iframe，组件才能正确渲染。`McpAppRenderer` 封装了这一完整流程。

前端收到 `render_ui_resource` 消息后，将 `iframeUrl` 和 `callToolResult` 传给渲染组件。

### 推荐：使用 `McpAppRenderer`（React）

如果你的前端是 React，可以直接使用 `@module-federation/mcp-apps` 提供的 `McpAppRenderer` 组件，它封装了 AppBridge 初始化、`postMessage` 通信、高度自适应等所有细节：

```bash
pnpm add @module-federation/mcp-apps
```

下面是一个完整的 React 对话面板示例，展示了从 SSE 接收消息到渲染组件的完整链路：

```tsx
// ChatPanel.tsx
import { useState, useEffect, useCallback } from 'react';
import { McpAppRenderer } from '@module-federation/mcp-apps/react';

// ─── 服务端推送的 SSE 消息结构（对应第五步的 render_ui_resource 事件）─────────
interface RenderUiEvent {
  type: 'render_ui_resource';
  /** mcp-app-shell.html 的公网 URL，直接作为 iframe src */
  iframeUrl: string;
  /** CallToolResult 的文本序列化（MCP Server 返回的原始 JSON 字符串） */
  callToolResult: string;
  /** 调用工具时传入的参数，原样透传给 MF 组件作为 props */
  toolInput: Record<string, unknown>;
  /** 工具名称 */
  toolName: string;
  /** 消息唯一 ID，区分同一工具的多次调用 */
  messageId: string;
}

interface TextEvent {
  type: 'text';
  text: string;
}

type SseEvent = RenderUiEvent | TextEvent | { type: '[DONE]' };

// ─── 前端消息模型 ──────────────────────────────────────────────────────────────
interface ChatMessage {
  id: string;
  kind: 'text' | 'ui';
  text?: string;
  ui?: RenderUiEvent;
}

// ─── 对话面板 ─────────────────────────────────────────────────────────────────
export function ChatPanel() {
  const [messages, setMessages] = useState<ChatMessage[]>([]);

  const sendUserMessage = useCallback((userText: string) => {
    const es = new EventSource(`/api/chat?q=${encodeURIComponent(userText)}`);

    es.onmessage = (e: MessageEvent<string>) => {
      const event: SseEvent = JSON.parse(e.data);

      if (event.type === '[DONE]') {
        es.close();
        return;
      }

      if (event.type === 'render_ui_resource') {
        // 收到 UI 渲染指令 → 追加一条类型为 'ui' 的消息
        setMessages((prev) => [
          ...prev,
          { id: event.messageId, kind: 'ui', ui: event },
        ]);
      } else if (event.type === 'text' && event.text) {
        // 普通文本 token → 追加或合并到最后一条文本消息
        setMessages((prev) => {
          const last = prev[prev.length - 1];
          if (last?.kind === 'text') {
            return [...prev.slice(0, -1), { ...last, text: last.text + event.text }];
          }
          return [...prev, { id: crypto.randomUUID(), kind: 'text', text: event.text }];
        });
      }
    };

    return () => es.close();
  }, []);

  return (
    <div className="chat-panel">
      {messages.map((msg) => {
        if (msg.kind === 'text') {
          return <p key={msg.id}>{msg.text}</p>;
        }

        // ── UI 消息：渲染 McpAppRenderer ──────────────────────────────────────
        const { ui } = msg;
        if (!ui) return null;

        return (
          <McpAppRenderer
            key={msg.id}
            /**
             * messageId 作为 React key 和 effect 依赖。
             * 同一工具被连续调用两次时，相同 iframeUrl 不会触发 re-render，
             * 但 messageId 变化会强制重新初始化 AppBridge。
             */
            messageId={msg.id}

            /**
             * 宿主信息，透传给 AppBridge，可在 MF 组件内通过
             * mcpApp.getHostInfo() 读取。
             */
            hostInfo={{ name: 'my-agent', version: '1.0.0' }}

            /**
             * 初始 iframe 高度（px）。
             * McpAppRenderer 会监听 AppBridge onsizechange 事件自动更新高度，
             * initialHeight 只影响首次渲染前的占位高度。
             */
            initialHeight={400}

            uiResource={{
              /**
               * resourceHttpUrl — iframe 的 src。
               * 必须是前端浏览器可访问的 HTTP URL（不能是服务端 localhost）。
               * 通常为 MCP Server 的 /static/mcp-app-shell.html。
               *
               * 示例值："http://mcp.example.com/static/mcp-app-shell.html"
               */
              resourceHttpUrl: ui.iframeUrl,

              /**
               * callToolResult — MF 组件加载配置，来自 MCP Server 工具返回值。
               * 必须包装成 CallToolResult 对象，不能直接传裸字符串。
               * AppBridge 就绪后通过 sendToolResult() 注入 iframe。
               *
               * ui.callToolResult 是服务端推送的原始 JSON 字符串，内容示例：
               * '{"moduleFederation":{"remoteName":"@my-org/ops-tools","module":"./DeployWizard",...}}'
               */
              callToolResult: {
                content: [{ type: 'text', text: ui.callToolResult }],
              },

              /**
               * toolResult — 工具入参（即 LLM 调用该工具时传入的 arguments）。
               * AppBridge 就绪后通过 sendToolInput() 注入 iframe，
               * MF 组件可通过 mcpApp.onToolInput(cb) 接收并渲染为 props。
               *
               * ui.toolInput 示例：{ appId: "my-app", env: "production" }
               */
              toolResult: ui.toolInput,
            }}
          />
        );
      })}
    </div>
  );
}
```

**各参数说明：**

- **`messageId`** `string`  
  每次工具调用的唯一 ID。作为 React `key` 和 `useEffect` 依赖，确保同一工具被连续调用两次时 AppBridge 也能正确重置。服务端推送的 `messageId` 字段直接传入即可。

- **`hostInfo`** `{ name: string; version: string }`  
  宿主应用的标识信息，透传给 AppBridge。MF 组件内可通过 `mcpApp.getHostInfo()` 读取，用于区分不同宿主环境或做版本兼容判断。

- **`initialHeight`** `number`（默认 `400`）  
  iframe 的初始占位高度（px）。`McpAppRenderer` 会监听 AppBridge `onsizechange` 事件并自动跟随内容调整高度，此值仅影响组件加载完成前的占位区域大小。

- **`uiResource.resourceHttpUrl`** `string`  
  iframe 的 `src` 地址，通常为 MCP Server 的 `/static/mcp-app-shell.html`。**必须是浏览器可直接访问的 HTTP URL**，不能是服务端的 `localhost`（浏览器无法访问）。

- **`uiResource.callToolResult`** `CallToolResult`  
  MF 组件的加载配置，来自 MCP Server 工具的返回值。**必须包装成 `{ content: [{ type: 'text', text: '...' }] }` 对象**，不能直接传裸字符串。AppBridge 就绪后通过 `sendToolResult()` 注入 iframe，组件从中解析 remote 配置并动态加载。

- **`uiResource.toolResult`** `object`  
  工具入参，即 LLM 调用该工具时传入的 `arguments`。AppBridge 就绪后通过 `sendToolInput()` 注入 iframe，MF 组件通过 `mcpApp.onToolInput(cb)` 接收并渲染为组件 props。

`McpAppRenderer` 初始化 AppBridge 后，会在 MCP App 就绪时自动：
1. 调用 `sendToolInput({ arguments: toolResult })` 将工具入参推入 iframe
2. 调用 `sendToolResult(callToolResult)` 将 `CallToolResult` 推入 iframe，MF 组件从中解析 remote 配置并动态加载组件

### 参考实现：纯 JS（非 React 环境）

非 React 前端需要直接使用 `AppBridge`（ext-apps SDK）—— mcp-app.tsx 通过 JSON-RPC over postMessage 与宿主通信，不支持裸 `postMessage` 自定义格式。

```typescript
// frontend/renderUi.ts
import { AppBridge, PostMessageTransport } from '@modelcontextprotocol/ext-apps/app-bridge';

interface RenderUiEvent {
  type: 'render_ui_resource';
  iframeUrl: string;
  callToolResult: string;
  toolInput: Record<string, unknown>;
  toolName: string;
  messageId?: string;
}

export async function renderMcpApp(msg: RenderUiEvent, container: HTMLElement) {
  const iframe = document.createElement('iframe');
  iframe.src = msg.iframeUrl;
  iframe.style.cssText = 'width:100%;border:none;min-height:200px;';
  iframe.sandbox.value = 'allow-scripts allow-same-origin allow-forms allow-popups';
  container.appendChild(iframe);

  await new Promise<void>((resolve) => {
    iframe.addEventListener('load', () => resolve(), { once: true });
  });

  const bridge = new AppBridge(
    null, // 无 MCP Client，工具调用由宿主直接处理
    { name: 'my-agent', version: '1.0.0' },
    { openLinks: {} },
    {
      hostContext: {
        theme: window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light',
        platform: 'web',
        displayMode: 'inline',
        availableDisplayModes: ['inline'],
        containerDimensions: { maxHeight: 6000 },
      },
    },
  );

  // 高度自适应：MF 组件通过 mcpApp.setHeight() 触发
  bridge.onsizechange = ({ height }) => {
    if (height !== undefined) {
      iframe.style.height = `${height}px`;
    }
  };

  // 外链跳转：MF 组件通过 mcpApp.openLink() 触发
  bridge.onopenlink = ({ url }) => {
    window.open(url, '_blank', 'noopener,noreferrer');
    return Promise.resolve({});
  };

  // ⚠️ oninitialized 必须在 connect() 之前注册，否则会错过初始化事件
  bridge.oninitialized = () => {
    // 用 setTimeout(0) 等待 MF 组件的 React effects 注册完毕，再发送数据
    setTimeout(() => {
      // 1. 先发工具入参（MF 组件的 props 来源）
      bridge.sendToolInput({ arguments: msg.toolInput });
      // 2. 再发 CallToolResult（MF 组件从中解析 remote 配置并动态加载组件）
      bridge.sendToolResult({ content: [{ type: 'text', text: msg.callToolResult }] });
    }, 0);
  };

  await bridge.connect(
    new PostMessageTransport(iframe.contentWindow!, iframe.contentWindow!),
  );

  return { iframe, bridge };
}
```

---

## 第七步 — 前端：处理 `sendMessage`（多步骤组件）

> **🔑 MCP Apps 方案特有**  
> MCP Apps 定义了 `sendMessage` 机制，允许 iframe 内的组件主动向对话注入消息（例如用户点击「下一步」时触发下一个工具）。这是实现多步骤向导的关键——每一步的 UI 工具调用都由前一步组件通过 `sendMessage` 触发，而不是由用户手动输入。

MF 组件调用 `mcpApp.sendMessage(...)` 时，消息通过 **AppBridge JSON-RPC 协议**传递给宿主，触发宿主的 `bridge.onmessage` 回调——**不是**裸 `postMessage`，不要监听 `window.message` 事件。

### 纯 JS 宿主：在 `renderMcpApp` 中注册回调

在上一步的 `renderMcpApp` 基础上，接收一个 `onSendMessage` 参数：

```typescript
export async function renderMcpApp(
  msg: RenderUiEvent,
  container: HTMLElement,
  // 可选：MF 组件调用 sendMessage 时触发，文本会作为新的用户输入发给 Agent
  onSendMessage?: (text: string) => void,
) {
  // ... iframe 创建、bridge 初始化（同上）...

  // MF 组件通过 mcpApp.sendMessage() 触发，走 AppBridge JSON-RPC，非裸 postMessage
  bridge.onmessage = async ({ content }) => {
    if (onSendMessage) {
      const text = Array.isArray(content)
        ? content.map((c: any) => c.text ?? '').join('')
        : String(content ?? '');
      if (text) onSendMessage(text);
    }
    return {}; // 必须返回结果，否则 MF 组件侧会收到超时错误
  };

  // ... bridge.oninitialized、bridge.connect（同上）...
}
```

调用时传入回调：

```typescript
await renderMcpApp(msg, container, (text) => {
  // 把文字当做新的用户输入，重新走 Agent streamChat 流程
  startNewChatStream(text);
});
```

### React 宿主：`McpAppRenderer` 暂不透出 `onmessage` prop

目前 `McpAppRenderer` 内部将 `bridge.onmessage` 设为空实现（静默丢弃）。如果你的场景需要处理 `sendMessage`，可以按纯 JS 方式手动封装 AppBridge，或向组件提 PR 添加 `onMessage` prop。

---

## 完整集成清单

| 步骤 | 位置 | 你需要做的 | 是否为 MCP Apps 特有 |
|------|------|-----------|--------------------|
| 1 | 任意 | 编写 `mcp_apps.json`，声明 remotes 和 tools | ✅ 特有 |
| 2 | 服务端 | 部署 `module-federation-mcp-starter`（HTTP 模式） | ✅ 特有 |
| 3 | 服务端 | 连接 MCP Client，将工具注册到 LLM；确保 `_meta` 字段被透传 | ⚡ 通用（_meta 透传是特有要求） |
| 4 | 服务端 | 在 `on_tool_end` 中检测 `_meta.ui.resourceUri`，用 `mcpServerBaseUrl` 拼装 `iframeUrl` | ✅ 特有 |
| 5 | 服务端 | 将 `render_ui_resource` 消息推送给前端（SSE / WebSocket） | ✅ 特有 |
| 6 | 前端 | 使用 `McpAppRenderer` 或手动创建 `<iframe>` 渲染 MCP App | ✅ 特有 |
| 7 | 前端 | 在 `bridge.onmessage` 中处理 MF 组件的 `sendMessage`，把文字注入 Agent 对话 | ✅ 特有 |

---

## 常见问题

### MF 组件没有渲染出来？

1. 检查 MCP Server 的 `mcp_apps.json` 中 `baseUrl` 是否正确且可访问
2. curl 验证 `/static/mcp-app-shell.html` 可访问（HTTP 200）
3. 在浏览器 DevTools 中打开 iframe，看控制台是否有 CORS 或 CSP 报错
4. 确保 `mcpServerBaseUrl` 设置为前端可访问的公网地址，不能是 `localhost`（浏览器无法访问服务端 localhost）

### 宿主不支持 UI，组件怎么降级？

MCP Server 无论宿主是否支持 UI 都正常返回 CallToolResult。  
如果你的前端没有实现 iframe 渲染，工具调用结果（JSON 文本）会直接出现在对话流中 — Agent 仍可正常工作，只是不展示 UI。

### 如何支持多个 MCP Server？

在创建多个 MCP Client 后，为每个工具记录它属于哪个 server 的 `mcpServerBaseUrl`。在调用 `detectAndEnrichUiTool` 时，根据工具名路由到对应 server 的 `mcpServerBaseUrl`，`iframeUrl` 会自动拼装到正确的 server 地址。

### `callToolResult` 里是什么？

它是 MCP Server 工具处理函数的返回值 `CallToolResult.content[0].text`，内容是一个 JSON 字符串，包含 MF 远程组件的加载配置（`moduleFederation.remoteName`、`remoteEntry`、`module` 等）。iframe 内的 AppBridge 会解析这个 JSON 并动态 `import()` 对应的 MF 远程模块。**你不需要解析它** — 直接原样传给前端即可。

---

## 参考

- [mcp_apps.json Schema](../../mcp_apps.schema.json)
- [MF 组件开发指南（sendMessage / callServerTool）](../../README.md#developing-mf-components)
- [ext-apps SDK 文档](https://github.com/module-federation/mcp-apps/tree/main/ext-apps)
