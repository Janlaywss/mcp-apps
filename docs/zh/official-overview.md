# MCP Apps 概览

> 完整 API 文档、高级模式说明及完整规范，请访问 [MCP Apps 官方文档](https://apps.extensions.modelcontextprotocol.io/)。

纯文字回应的能力是有限的。有时用户需要与数据进行交互，而不仅仅是阅读它。MCP Apps 让服务端能够返回可交互的 HTML 界面——数据可视化、表单、仪表盘——直接渲染在对话窗口中。

---

## 为什么不直接做一个独立的 Web App？

你完全可以做一个独立的 Web 应用，然后给用户发链接。但 MCP Apps 有几个独立页面无法匹配的核心优势：

**上下文保留**
应用就在对话里。用户不需要切换 Tab、找不到上次的分析位置、也不用回想"哪个对话里有那个仪表盘"。UI 就在那里，紧挨着引出它的那段讨论。

**双向数据流**
应用可以调用 MCP 服务端上的任意工具，宿主也可以向应用推送最新结果。一个独立的 Web 应用需要自己维护 API、认证体系和状态管理；MCP Apps 通过现有的 MCP 机制天然获得这些能力。

**与宿主能力的集成**
应用可以将操作委托给宿主，由宿主调用用户已经连接的能力和工具（须用户授权）。每个应用无需自己实现和维护各类直接集成（如邮件服务提供商），而是直接请求一个结果（如"安排这个会议"），宿主通过用户已有的连接能力来路由执行。

**安全保障**
MCP Apps 运行在宿主控制的沙箱 iframe 中。它们无法访问父页面、窃取 Cookie 或逃出容器。这意味着宿主可以安全地渲染第三方应用，而不需要完全信任服务端作者。

如果你的用例不需要这些特性，普通 Web 应用或许更简单。但如果你需要与 LLM 对话紧密集成，MCP Apps 是更合适的工具。

---

## MCP Apps 如何工作

传统 MCP 工具返回文本、图片、资源或结构化数据，宿主将其作为对话内容展示。MCP Apps 扩展了这个模式：工具可以在其描述中声明一个指向可交互 UI 的引用，宿主将其直接渲染在对话中。

其核心模式结合了两个 MCP 原语：**一个声明了 UI 资源的工具** + **一个将数据渲染为可交互 HTML 界面的 UI 资源**。

当 LLM 决定调用一个支持 MCP Apps 的工具时，流程如下：

1. **UI 预加载**：工具描述中包含一个 `_meta.ui.resourceUri` 字段，指向一个 `ui://` 资源。宿主可以在工具被调用之前就预加载这个资源，从而支持将工具输入流式传输给应用等特性。

2. **获取资源**：宿主从服务端获取 UI 资源，该资源是一个 HTML 页面，通常已将 JavaScript 和 CSS 打包在一起。应用也可以从 `_meta.ui.csp` 中指定的 origin 加载外部脚本和资源。

3. **沙箱渲染**：Web 宿主通常在对话中以沙箱 iframe 的形式渲染 HTML。沙箱限制应用访问父页面，保证安全性。资源的 `_meta.ui` 对象可以包含 `permissions`（请求麦克风、摄像头等额外能力）和 `csp`（控制应用可以从哪些外部 origin 加载资源）。

4. **双向通信**：应用与宿主通过一套基于 JSON-RPC 的协议通信，该协议构成了 MCP 自己的一个方言。部分请求和通知与核心 MCP 协议共享（如 `tools/call`），部分类似（如 `ui/initialize`），大多数是以 `ui/` 为前缀的新方法。应用可以发起工具调用、发送消息、更新模型上下文、并接收来自宿主的数据。

```
MCP Server  MCP App iframe  Agent  User

"show me analytics"
               ────────────────────────→  tools/call
               ←────────────────────────  tool input/result
← tool result pushed to app ────────────
user interacts ─────────────────────────→
               ────────────────────────→  tools/call request
               ────────────────────────→  tools/call (forwarded)
               ←────────────────────────  fresh data
← fresh data ───────────────────────────
context update ─────────────────────────→
```

应用与宿主保持隔离，但仍可通过安全的 postMessage 通道调用 MCP 工具。

---

## 什么时候该用 MCP Apps

以下场景适合使用 MCP Apps：

**探索复杂数据**
用户问"按地区显示销售额"。文字回应只能列出数字，而 MCP App 可以渲染一张可交互的地图——用户点击区域下钻、悬停查看详情、在不同指标之间切换，无需追加任何提示词。

**需要配置多个选项**
设置一个部署环境涉及数十个相互依赖的选项。与其来回对话（"哪个区域？""实例规格？""开启自动扩缩容？"），MCP App 可以直接呈现一个表单，用户一次看到所有选项，还有校验和默认值。

**展示富媒体内容**
当用户想要查阅 PDF、查看 3D 模型或预览生成的图片时，文字描述远远不够。MCP App 可以在对话里直接嵌入真正的查看器（平移、缩放、旋转）。

**实时监控**
显示实时指标、日志或系统状态的仪表盘需要持续更新。MCP App 维护一个持久连接，数据变化时自动更新显示，无需用户反复问"现在状态怎么样？"。

**多步骤工作流**
审批报销单、审查代码变更或对问题进行分类，需要逐一检查每个条目。MCP App 可以提供导航控件、操作按钮，以及在多次交互之间持久保存的状态。

---

## 安全模型

MCP Apps 运行在沙箱 iframe 中，与宿主应用之间有强隔离。沙箱阻止你的应用：

- 访问父窗口的 DOM
- 读取宿主的 Cookie 或 localStorage
- 导航父页面
- 在父页面上下文中执行脚本

应用与宿主之间的所有通信都通过 [postMessage API](https://developer.mozilla.org/docs/Web/API/Window/postMessage) 进行。宿主控制应用可以访问哪些能力——例如，宿主可以限制应用能调用哪些工具，或禁用 `sendOpenLink` 能力。

沙箱的设计目的是防止应用逃出容器访问宿主或用户数据。

---

## 框架支持

MCP Apps 使用自己的 MCP 方言，和核心协议一样基于 JSON-RPC。部分消息与常规 MCP 共享（如 `tools/call`），部分是应用特有的（如 `ui/initialize`）。传输层使用 postMessage，而非 stdio 或 HTTP。由于都是标准的 Web 原语，你可以使用任意框架，甚至不用框架。

`@modelcontextprotocol/ext-apps` 中的 `App` 类只是一个便捷封装，并非必须使用。如果你希望避免依赖或需要更细粒度的控制，可以直接实现 [postMessage 协议](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx)。

[examples 目录](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples)包含 React、Vue、Svelte、Preact、Solid 和原生 JavaScript 的入门模板，展示了各框架推荐的使用模式，但这些仅供参考，并非必须遵循。

---

## 客户端支持

> MCP Apps 是[核心 MCP 规范](https://modelcontextprotocol.io/specification)的扩展。各客户端的支持情况不同。

目前支持 MCP Apps 的客户端包括：[Claude](https://claude.ai/)、[Claude Desktop](https://claude.ai/download)、[VS Code GitHub Copilot](https://code.visualstudio.com/)、[Goose](https://block.github.io/goose/)、[Postman](https://postman.com/)、[MCPJam](https://www.mcpjam.com/)。完整的扩展支持矩阵请参阅[客户端矩阵](https://modelcontextprotocol.io/extensions/client-matrix)。

如果你正在构建一个 MCP 客户端并希望支持 MCP Apps，有两种选择：

1. **使用框架**：[@mcp-ui/client](https://github.com/MCP-UI-Org/mcp-ui) 包提供了在宿主应用中渲染和交互 MCP Apps 的 React 组件，详见 [MCP-UI 文档](https://mcpui.dev/)。

2. **基于 AppBridge 构建**：SDK 包含一个 [App Bridge](https://apps.extensions.modelcontextprotocol.io/api/modules/app-bridge.html) 模块，负责处理沙箱 iframe 中的应用渲染、消息传递、工具调用代理以及安全策略执行。[basic-host 示例](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/basic-host)展示了如何集成它。

---

## 示例

[ext-apps 仓库](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples)包含多个可以直接运行的示例，涵盖不同使用场景：

**3D 与可视化**
[map-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/map-server)（CesiumJS 地球仪）、[threejs-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/threejs-server)（Three.js 场景）、[shadertoy-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/shadertoy-server)（Shader 特效）

**数据探索**
[cohort-heatmap-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/cohort-heatmap-server)、[customer-segmentation-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/customer-segmentation-server)、[wiki-explorer-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/wiki-explorer-server)

**业务应用**
[scenario-modeler-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/scenario-modeler-server)、[budget-allocator-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/budget-allocator-server)

**媒体**
[pdf-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/pdf-server)、[video-resource-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/video-resource-server)、[sheet-music-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/sheet-music-server)、[say-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/say-server)（文字转语音）

**工具类**
[qr-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/qr-server)、[system-monitor-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/system-monitor-server)、[transcript-server](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/transcript-server)（语音转文字）

**入门模板**
[React](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/basic-server-react)、[Vue](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/basic-server-vue)、[Svelte](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/basic-server-svelte)、[Preact](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/basic-server-preact)、[Solid](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/basic-server-solid)、[原生 JavaScript](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples/basic-server-vanillajs)

如需开始构建自己的 MCP App，请参阅[构建指南](https://modelcontextprotocol.io/extensions/apps/build)。
