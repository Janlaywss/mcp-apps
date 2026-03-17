# 05 · Integrating a Custom Agent with `@module-federation/mcp-apps`

This guide is for developers who **already have their own Agent service** and want to connect it to Module Federation MCP Apps.  
If you are using a native MCP Apps client such as Claude Desktop, VS Code GitHub Copilot, or Goose, **you don't need this guide** — those hosts handle UI rendering natively. Just configure `mcp_apps.json` and register your MCP Server with the client.

> ⭐ Found this useful? Give us a star on GitHub: **[module-federation/mcp-apps](https://github.com/module-federation/mcp-apps)**

---

## When to Use This Guide

| Situation | Needed? |
|-----------|---------|
| Using Claude Desktop / VS Code / Goose (STDIO mode) | ❌ No — the host handles it natively |
| **Custom Agent (any framework) + custom web frontend** | ✅ **This guide** |

---

## Architecture Overview

Native MCP hosts (Claude Desktop, etc.) can call `resources/read` directly and render HTML.  
A custom B/S Agent faces a fundamental challenge: **the MCP Client lives on the server, while the iframe lives in the browser** — they need a bridging channel.

```
User input
    │
    ▼
[Server] Your Agent (LangGraph ReAct / any framework)
    │  LLM decides to call a UI tool based on tool.description
    ▼
[Server] MCP Client → POST /mcp → MCP Server executes the tool
    │  CallToolResult carries JSON config (includes MF remote/module info)
    ▼  on_tool_end / tool_call callback
[Server] UI Enrichment Layer (you implement this)
    │  ① Detect _meta.ui.resourceUri in the tool metadata
    │  ② Assemble iframeUrl from mcpServerBaseUrl
    │  ③ Push a render_ui_resource message to the frontend (SSE / WebSocket)
    ▼  stream
[Browser] Your frontend
    │  Parse render_ui_resource → render <iframe>
    ▼  iframe loaded
[iframe] MCP App (AppBridge built into the shell)
    │  Receive tool args + CallToolResult via postMessage
    ▼  Dynamically import MF remote component
    User sees your React component ✅
```

The core logic is just two steps: **detect UI tool** → **push to frontend**. The `iframeUrl` is assembled directly from `mcpServerBaseUrl` — no `resources/read` required.

---

## Prerequisites

- A deployed Module Federation remote component (or local `http://localhost:8080`)
- An `mcp_apps.json` config file (see below)
- Your Agent service can connect to the MCP Server (HTTP mode)
- Your web frontend can render arbitrary HTML into an `<iframe>`

---

## Step 1 — Write `mcp_apps.json`

Tell the MCP Server which MF components to expose as tools.

### Auto-generate with AI Skill (recommended)

The project ships with an AI Skill that detects your Module Federation config (`module-federation.config.ts` / `rspack.config.js`, etc.), parses `name` and `exposes`, and generates `mcp_apps.json` automatically.

Provide [`skill/generate-mcp-apps-config.md`](../../skill/generate-mcp-apps-config.md) as context to any AI coding tool (Claude, Cursor, GitHub Copilot, Windsurf, etc.) and say:

```
Following the instructions in skill/generate-mcp-apps-config.md, generate mcp_apps.json for my project.
```

### Write manually

```jsonc
{
  // remotes: declare your MF remote packages; each remote maps to a CDN-deployed MF bundle
  "remotes": [
    {
      // name: MF package name — must match the `name` field in module-federation.config.ts
      "name": "@my-org/ops-tools",
      // baseUrl: CDN root path for the MF bundle; the server loads remoteEntry.js from here
      "baseUrl": "https://cdn.example.com/ops-tools/2.4.1/mf-manifest.json",
      // csp: Content Security Policy controlling which domains the iframe may access
      "csp": {
        // connectDomains: domains allowed for fetch/XHR inside the iframe
        "connectDomains": ["cdn.example.com", "api.example.com"],
        // resourceDomains: domains allowed for script/img resources inside the iframe
        "resourceDomains": ["cdn.example.com"]
      }
    }
  ],
  // tools: declare the tools exposed to the LLM; each tool maps to an MF component
  "tools": [
    {
      // name: unique tool identifier used by the LLM to call it (snake_case recommended)
      "name": "deploy_wizard",
      // title: display name for the tool (optional)
      "title": "Deploy Wizard",
      // description: tells the LLM when to call this tool — the clearer the better
      "description": "Call this tool when the user wants to deploy an application. Shows a visual deployment workflow.",
      // inputSchema: JSON Schema for the tool's input params; the LLM generates args from this
      "inputSchema": {
        "type": "object",
        "properties": {
          "appId": { "type": "string", "description": "ID of the application to deploy" },
          "env":   { "type": "string", "enum": ["staging", "production"] }
        },
        "required": []
      },
      // remote: which remote to use — must match a remotes[].name above
      "remote": "@my-org/ops-tools",
      // module: the MF expose path, matching the key in module-federation.config.ts exposes
      "module": "./DeployWizard",
      // exportName: component export name, defaults to "default" (optional)
      "exportName": "default"
    }
  ]
}
```

> **Multi-step wizards** (step1 → step2 → step3): each step's `inputSchema` should declare the fields injected by the previous step via `sendMessage`. Add "do not call proactively" to the `description` to prevent the LLM from triggering it unexpectedly.

---

## Step 2 — Deploy the MCP Server (HTTP mode)

HTTP mode requires **running the MCP Server as a long-lived Node.js service**, unlike STDIO mode which launches it as a local subprocess. Use the ready-to-go project template:

**[module-federation-mcp-starter](../../module-federation-mcp-starter)**

Clone or copy that directory — it already includes:
- Pre-configured `package.json` and startup scripts
- A placeholder `mcp_apps.json` (replace with your config)
- `MF_MCP_BASE_URL` environment variable support (for the public domain in production)

```bash
cd module-federation-mcp-starter

# Install dependencies
pnpm install

# Local development
pnpm run start

# Production deployment (set a publicly accessible domain for browser iframe access)
MF_MCP_BASE_URL=https://your-mcp-server.example.com pnpm run start
```

After starting, the server exposes:
- `POST /mcp` — standard MCP Streamable HTTP endpoint (used by `@modelcontextprotocol/sdk`)
- `GET /static/mcp-app-shell.html` — lightweight iframe shell (~530 B, loads JS asynchronously from `/static/`)
- `GET /static/mcp-app.html` — fully inlined HTML (~686 KB, no network dependencies, fallback option)

> `MF_MCP_BASE_URL` determines the base path for JS/CSS in the shell HTML. In local development it defaults to `http://localhost:<port>`. In production it **must be set to a publicly accessible address** — otherwise the browser cannot load the iframe's static assets and you'll get a blank screen.

Verify the server is working:

```bash
curl http://localhost:3001/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

---

## Step 3 — Server: Connect the MCP Client and Register Tools with the LLM

> This step is a standard MCP + Agent integration flow **unrelated to the Module Federation MCP Apps specifics**. The example below uses LangChain + `@modelcontextprotocol/sdk`; adapt as needed for your framework.

### Connect the MCP Client

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

const client = new Client({ name: 'my-agent', version: '1.0.0' });
await client.connect(
  new StreamableHTTPClientTransport(new URL('http://localhost:3001/mcp'))
);

// ⚠️ MCP Apps specific: tool definitions carry _meta.ui metadata;
//    the next step needs it to identify UI tools.
const { tools } = await client.listTools();
```

### Register Tools with the LLM (LangChain example)

```typescript
import { DynamicStructuredTool } from '@langchain/core/tools';

// Use any MCP → LangChain adapter, e.g. @h1deya/langchain-mcp-tools,
// or manually convert tools[].inputSchema to a Zod schema and wrap in DynamicStructuredTool.
const langchainTools = tools.map((tool) => new DynamicStructuredTool({
  name: tool.name,
  description: tool.description ?? '',
  schema: jsonSchemaToZod(tool.inputSchema ?? {}),
  func: async (args) => JSON.stringify(await client.callTool({ name: tool.name, arguments: args })),
  // ⚠️ MCP Apps specific: preserve _meta so the UI enrichment layer can read resourceUri
  metadata: { _meta: (tool as any)._meta },
}));

const agent = createReactAgent({ llm, tools: langchainTools });
```

> The exact registration method depends on your Agent framework (LangGraph, AutoGen, CrewAI, etc.). Regardless of framework, **ensure the `_meta` field is forwarded** — without it the next step cannot identify UI tools.

---

## Step 4 — Server: UI Enrichment Layer (core)

> **🔑 MCP Apps specific**  
> In standard MCP, after a tool returns its result the Agent framework passes the text directly to the LLM. MCP Apps extends this with a convention: UI tools carry `_meta.ui.resourceUri` in their definition, pointing to the MCP App HTML hosted by the server. Your Agent service must **intercept these tool results** in `on_tool_end` (or equivalent callback) and push a render instruction to the frontend instead of forwarding HTML to the LLM.

This is the most critical part of a custom Agent integration. **After every tool call**, check whether the tool is a UI tool — if so, assemble the `iframeUrl` and push it to the frontend, then continue streaming any follow-up LLM text.

```typescript
// agent/uiEnrichment.ts

export interface UiRenderPayload {
  /** Publicly accessible URL of mcp-app-shell.html, used as the iframe src */
  iframeUrl: string;
  /** Serialized CallToolResult text (JSON string; the MF component uses this as its props source) */
  callToolResult: string;
  /** Arguments passed to the tool (forwarded to the MF component as props) */
  toolInput: Record<string, unknown>;
  /** Tool name */
  toolName: string;
}

/**
 * Call this after each tool invocation.
 * Returns the frontend render payload if the tool has UI metadata; otherwise null.
 *
 * @param toolName           Name of the tool that just finished
 * @param toolInput          Arguments passed to the tool
 * @param callToolResultText Serialized CallToolResult text (converted from LangGraph event.data.output)
 * @param mcpTools           Full tool list returned by tools/list
 * @param mcpServerBaseUrl   Public root address of the MCP Server, e.g. "http://localhost:3001"
 */
export function detectAndEnrichUiTool(
  toolName: string,
  toolInput: Record<string, unknown>,
  callToolResultText: string,
  mcpTools: any[],
  mcpServerBaseUrl: string,
): UiRenderPayload | null {
  // Find the tool definition and check for UI metadata
  const toolDef = mcpTools.find((t) => t.name === toolName);
  const resourceUri: string | undefined = (toolDef as any)?._meta?.ui?.resourceUri;

  if (!resourceUri) {
    return null; // Regular tool — follow the normal text stream
  }

  // Use the lightweight shell page (~530 B), which loads JS asynchronously from /static/
  const iframeUrl = `${mcpServerBaseUrl.replace(/\/$/, '')}/static/mcp-app-shell.html`;

  return {
    iframeUrl,
    callToolResult: callToolResultText,
    toolInput,
    toolName,
  };
}
```

Wire it into your Agent's streaming event loop:

```typescript
// agent/agentService.ts (LangGraph example)
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
      // LangGraph serializes CallToolResult into a string in event.data.output
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
        // Push UI render message to the frontend, then continue streaming —
        // the LLM may still emit follow-up text after the tool call.
        yield { type: 'render_ui', payload: uiPayload };
      }
    }

    // Regular text token
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

## Step 5 — Server: Push to the Frontend via SSE / WebSocket

> **🔑 MCP Apps specific**  
> Add a new message type `render_ui_resource` to your existing streaming protocol (SSE / WebSocket / any custom format). The frontend triggers iframe rendering upon receiving it. The message structure is up to you, but must include `iframeUrl` (iframe load address) and `callToolResult` (MF component load config).

Wrap the `render_ui` messages from the previous step into your streaming protocol. SSE example:

```typescript
// server/routes/chat.ts (Koa / Express / Hono all work)
async function streamChatHandler(req, res) {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  for await (const chunk of streamChat(req.body.message, agent, mcpTools)) {
    if (chunk.type === 'text') {
      res.write(`data: ${JSON.stringify({ type: 'text', text: chunk.payload })}\n\n`);
    } else if (chunk.type === 'render_ui') {
      // Push the render_ui_resource event to the frontend
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

## Step 6 — Frontend: Render the iframe

> **🔑 MCP Apps specific**  
> The MCP App runs inside an `<iframe>` and communicates with the host page via AppBridge (built on `postMessage`). After the iframe loads, the host must actively push `callToolResult` (MF component load config) and tool arguments into the iframe before the component can render. `McpAppRenderer` encapsulates this entire flow.

When the frontend receives a `render_ui_resource` message, pass `iframeUrl` and `callToolResult` to the rendering component.

### Recommended: use `McpAppRenderer` (React)

If your frontend is React, use the `McpAppRenderer` component from `@module-federation/mcp-apps`. It handles AppBridge initialization, `postMessage` communication, and automatic height adjustment:

```bash
pnpm add @module-federation/mcp-apps
```

Below is a complete React chat panel example showing the full flow from SSE message to rendered component:

```tsx
// ChatPanel.tsx
import { useState, useCallback } from 'react';
import { McpAppRenderer } from '@module-federation/mcp-apps/react';

// ─── SSE message shape from the server (Step 5 render_ui_resource event) ──────
interface RenderUiEvent {
  type: 'render_ui_resource';
  /** Public URL of mcp-app-shell.html, used directly as the iframe src */
  iframeUrl: string;
  /** Serialized CallToolResult text (raw JSON string returned by the MCP Server) */
  callToolResult: string;
  /** Tool arguments, forwarded as-is to the MF component as props */
  toolInput: Record<string, unknown>;
  /** Tool name */
  toolName: string;
  /** Unique message ID — distinguishes multiple calls to the same tool */
  messageId: string;
}

interface TextEvent {
  type: 'text';
  text: string;
}

type SseEvent = RenderUiEvent | TextEvent | { type: '[DONE]' };

// ─── Frontend message model ────────────────────────────────────────────────────
interface ChatMessage {
  id: string;
  kind: 'text' | 'ui';
  text?: string;
  ui?: RenderUiEvent;
}

// ─── Chat panel ───────────────────────────────────────────────────────────────
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
        // UI render instruction received → append a 'ui' message
        setMessages((prev) => [
          ...prev,
          { id: event.messageId, kind: 'ui', ui: event },
        ]);
      } else if (event.type === 'text' && event.text) {
        // Text token → append or merge into the last text message
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

        // ── UI message: render McpAppRenderer ─────────────────────────────────
        const { ui } = msg;
        if (!ui) return null;

        return (
          <McpAppRenderer
            key={msg.id}
            /**
             * messageId acts as React key and useEffect dependency.
             * When the same tool is called twice in a row, the identical iframeUrl
             * won't trigger a re-render on its own — messageId change forces AppBridge
             * to reinitialize.
             */
            messageId={msg.id}

            /**
             * Host application info forwarded to AppBridge.
             * Readable inside the MF component via mcpApp.getHostInfo().
             */
            hostInfo={{ name: 'my-agent', version: '1.0.0' }}

            /**
             * Initial iframe height (px).
             * McpAppRenderer listens to AppBridge onsizechange and adjusts automatically.
             * initialHeight only affects the placeholder before the component loads.
             */
            initialHeight={400}

            uiResource={{
              /**
               * resourceHttpUrl — iframe src.
               * Must be an HTTP URL accessible by the browser (not the server's localhost).
               * Typically the MCP Server's /static/mcp-app-shell.html.
               *
               * Example: "http://mcp.example.com/static/mcp-app-shell.html"
               */
              resourceHttpUrl: ui.iframeUrl,

              /**
               * callToolResult — MF component load config from the MCP Server tool result.
               * Must be wrapped in a CallToolResult object — do NOT pass a raw string.
               * Injected into the iframe via sendToolResult() once AppBridge is ready.
               *
               * ui.callToolResult is the raw JSON string pushed by the server, e.g.:
               * '{"moduleFederation":{"remoteName":"@my-org/ops-tools","module":"./DeployWizard",...}}'
               */
              callToolResult: {
                content: [{ type: 'text', text: ui.callToolResult }],
              },

              /**
               * toolResult — tool arguments (the arguments the LLM passed to the tool).
               * Injected into the iframe via sendToolInput() once AppBridge is ready.
               * The MF component receives them via mcpApp.onToolInput(cb) and renders as props.
               *
               * Example: { appId: "my-app", env: "production" }
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

**Prop reference:**

- **`messageId`** `string`  
  Unique ID for each tool call. Acts as React `key` and `useEffect` dependency, ensuring AppBridge reinitializes correctly when the same tool is called twice consecutively. Pass the `messageId` field from the server event directly.

- **`hostInfo`** `{ name: string; version: string }`  
  Host application identifier, forwarded to AppBridge. Readable inside the MF component via `mcpApp.getHostInfo()` — useful for distinguishing host environments or version compatibility checks.

- **`initialHeight`** `number` (default `400`)  
  Initial iframe placeholder height in px. `McpAppRenderer` listens to AppBridge `onsizechange` and adjusts automatically; this value only affects the placeholder before the component finishes loading.

- **`uiResource.resourceHttpUrl`** `string`  
  The iframe `src` — typically the MCP Server's `/static/mcp-app-shell.html`. **Must be an HTTP URL directly accessible by the browser.** Do not use the server's `localhost` (the browser cannot reach it).

- **`uiResource.callToolResult`** `CallToolResult`  
  MF component load config from the MCP Server tool result. **Must be wrapped as `{ content: [{ type: 'text', text: '...' }] }`** — never pass a raw string. Injected via `sendToolResult()` once AppBridge is ready; the component parses the remote config and dynamically loads itself.

- **`uiResource.toolResult`** `object`  
  Tool arguments — the `arguments` the LLM passed when calling the tool. Injected via `sendToolInput()` once AppBridge is ready. The MF component receives them via `mcpApp.onToolInput(cb)` and renders them as props.

Once AppBridge initializes, `McpAppRenderer` automatically:
1. Calls `sendToolInput({ arguments: toolResult })` to push tool arguments into the iframe
2. Calls `sendToolResult(callToolResult)` to push the `CallToolResult` into the iframe; the MF component parses the remote config and dynamically loads

### Reference implementation: plain JS (non-React)

Non-React frontends must use `AppBridge` directly from the ext-apps SDK. The MCP App shell communicates via JSON-RPC over `postMessage` — raw `postMessage` with a custom format is not supported.

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
    null, // No MCP Client — tool calls are handled directly by the host
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

  // Height adjustment: triggered when the MF component calls mcpApp.setHeight()
  bridge.onsizechange = ({ height }) => {
    if (height !== undefined) {
      iframe.style.height = `${height}px`;
    }
  };

  // External link handling: triggered when the MF component calls mcpApp.openLink()
  bridge.onopenlink = ({ url }) => {
    window.open(url, '_blank', 'noopener,noreferrer');
    return Promise.resolve({});
  };

  // ⚠️ oninitialized must be registered BEFORE connect() — the initialized event
  //    fires during connect() and will be missed if set afterward.
  bridge.oninitialized = () => {
    // Use setTimeout(0) to wait for the MF component's React effects to register
    // their ontoolresult / ontoolinput handlers before we send data.
    setTimeout(() => {
      // 1. Send tool arguments first (the MF component's props source)
      bridge.sendToolInput({ arguments: msg.toolInput });
      // 2. Then send CallToolResult (the MF component parses remote config and loads)
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

## Step 7 — Frontend: Handle `sendMessage` (Multi-step Components)

> **🔑 MCP Apps specific**  
> MCP Apps defines a `sendMessage` mechanism that lets components inside the iframe inject messages into the conversation (e.g. when the user clicks "Next" to trigger the next step's tool). This is key to multi-step wizards — each step is triggered by the previous component via `sendMessage`, not by the user typing manually.

When an MF component calls `mcpApp.sendMessage(...)`, the message is delivered to the host via the **AppBridge JSON-RPC protocol** and triggers the host's `bridge.onmessage` callback — **not** a raw `postMessage`. Do not listen to `window.message` events.

### Plain JS host: register a callback in `renderMcpApp`

Extend `renderMcpApp` from the previous step to accept an `onSendMessage` parameter:

```typescript
export async function renderMcpApp(
  msg: RenderUiEvent,
  container: HTMLElement,
  // Optional: called when the MF component calls sendMessage; text is sent to the Agent as new user input
  onSendMessage?: (text: string) => void,
) {
  // ... iframe creation, bridge initialization (same as above) ...

  // Triggered by mcpApp.sendMessage() via AppBridge JSON-RPC — not raw postMessage
  bridge.onmessage = async ({ content }) => {
    if (onSendMessage) {
      const text = Array.isArray(content)
        ? content.map((c: any) => c.text ?? '').join('')
        : String(content ?? '');
      if (text) onSendMessage(text);
    }
    return {}; // Must return a result — otherwise the MF component side receives a timeout error
  };

  // ... bridge.oninitialized, bridge.connect (same as above) ...
}
```

Pass the callback when calling:

```typescript
await renderMcpApp(msg, container, (text) => {
  // Treat the text as new user input and restart the Agent streamChat flow
  startNewChatStream(text);
});
```

### React host: `McpAppRenderer` does not currently expose an `onmessage` prop

`McpAppRenderer` currently sets `bridge.onmessage` to a no-op internally (silently discards). If your use case requires handling `sendMessage`, wrap AppBridge directly using the plain JS approach above, or open a PR to add an `onMessage` prop to the component.

---

## Integration Checklist

| Step | Where | What to do | MCP Apps specific? |
|------|-------|------------|--------------------|
| 1 | Anywhere | Write `mcp_apps.json` declaring remotes and tools | ✅ Specific |
| 2 | Server | Deploy `module-federation-mcp-starter` (HTTP mode) | ✅ Specific |
| 3 | Server | Connect MCP Client, register tools with LLM; ensure `_meta` is forwarded | ⚡ General (`_meta` forwarding is specific) |
| 4 | Server | In `on_tool_end`, detect `_meta.ui.resourceUri` and assemble `iframeUrl` from `mcpServerBaseUrl` | ✅ Specific |
| 5 | Server | Push `render_ui_resource` message to the frontend (SSE / WebSocket) | ✅ Specific |
| 6 | Frontend | Use `McpAppRenderer` or manually create `<iframe>` to render the MCP App | ✅ Specific |
| 7 | Frontend | Handle the MF component's `sendMessage` in `bridge.onmessage` and inject text into the Agent conversation | ✅ Specific |

---

## FAQ

### MF component is not rendering?

1. Check that the `baseUrl` in `mcp_apps.json` is correct and accessible
2. `curl` to verify `/static/mcp-app-shell.html` returns HTTP 200
3. Open the iframe in browser DevTools and check the console for CORS or CSP errors
4. Ensure `mcpServerBaseUrl` is set to a publicly accessible address — `localhost` is only reachable from the server, not the browser

### What if the host doesn't support UI — how does it degrade?

The MCP Server always returns a `CallToolResult` regardless of whether the host supports UI.  
If your frontend has no iframe renderer, the tool result (a JSON text string) appears in the conversation stream — the Agent still works correctly, just without the visual UI.

### How do I support multiple MCP Servers?

When you create multiple MCP Clients, track which server's `mcpServerBaseUrl` each tool belongs to. In `detectAndEnrichUiTool`, route to the correct `mcpServerBaseUrl` based on the tool name — the `iframeUrl` will then be assembled pointing to the right server.

### What is inside `callToolResult`?

It is the `CallToolResult.content[0].text` returned by the MCP Server tool handler — a JSON string containing the MF remote component's load config (`moduleFederation.remoteName`, `remoteEntry`, `module`, etc.). AppBridge inside the iframe parses this JSON and dynamically `import()`s the corresponding MF remote module. **You don't need to parse it** — pass it through to the frontend as-is.

---

## References

- [mcp_apps.json Schema](../../mcp_apps.schema.json)
- [MF Component Guide (sendMessage / callServerTool)](../../README.md#developing-mf-components)
- [ext-apps SDK docs](https://github.com/module-federation/mcp-apps/tree/main/ext-apps)
