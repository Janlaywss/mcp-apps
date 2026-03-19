# 接入 MCP Apps（Module Federation 方案）

将你的 Module Federation 组件变成 AI 宿主（Claude Desktop、VS Code 等）中可交互的 MCP 工具。

---

## 前置要求

需要 [Node.js](https://nodejs.org/en/download) 22 或更高版本。建议先了解 [MCP 工具](https://modelcontextprotocol.io/specification/latest/server/tools)和 [MCP Apps 标准](https://modelcontextprotocol.io/docs/extensions/apps)的基本概念。已有 Module Federation 或 Vmok 项目的团队可以从**第二步**直接开始。

---

## 快速开始

最快的方式是使用 AI 编程助手自动完成接入。如果你希望手动操作，请跳至「手动接入」章节。

### 使用 AI 编程助手

`@module-federation/mcp-apps` 提供了一组 Skills，能让 AI 编程助手自动完成项目类型检测、`mcp_apps.json` 生成、MCP Server 注册等全部操作，无需手写任何 MCP 代码。

#### 第一步：安装 skill

如果你使用 Claude Code，可以直接通过以下命令安装：

```bash
/plugin marketplace add module-federation/mcp-apps
/plugin install integrate-mcp-apps@module-federation-mcp-apps
```

也可以手动克隆仓库，将 skill 目录复制到对应助手的配置路径下：

```bash
git clone https://github.com/module-federation/mcp-apps.git
```

然后将 `mcp-apps/module-federation-mcp/skill/` 下的文件复制到你的 AI 助手对应目录：

| 助手 | macOS / Linux | Windows |
|------|---------------|---------|
| Claude Code | `~/.claude/skills/` | `%USERPROFILE%\.claude\skills\` |
| VS Code / GitHub Copilot | `~/.copilot/skills/` | `%USERPROFILE%\.copilot\skills\` |
| Cursor | `~/.cursor/skills/` | `%USERPROFILE%\.cursor\skills\` |
| Goose | `~/.config/goose/skills/` | `%USERPROFILE%\.config\goose\skills\` |

验证 skill 是否安装成功，可以询问你的 AI 助手："你有哪些 skills？"——你应该能看到 `integrate-mcp-apps` 出现在列表中。

#### 第二步：告诉助手开始接入

在你的前端项目根目录，直接告诉 AI 助手：

```
请按照 integrate-mcp-apps skill 帮我把这个项目接入 MCP Apps
```

助手会识别你的项目类型（Vmok / 标准 MF / 无 MF），自动完成从配置到启动的全流程，最终在 Claude Desktop 里出现你的工具。

#### 第三步：测试

按照下方「测试你的 MCP App」章节验证接入效果。

---

### 手动接入

如果你没有使用 AI 编程助手，或希望理解每个步骤，请按以下流程操作。

---

## 第一步：检查项目类型

不同的前端框架使用不同的 Module Federation 配置文件，接入方式略有差异。在项目根目录运行：

```bash
# 检查是否是 Vmok 项目
ls vmok.config.ts edenx.config.ts 2>/dev/null
cat package.json | grep -E '"@edenx/plugin-vmok-v3"|"@edenx/plugin-vmok"|"@vmok/kit"'

# 检查是否是标准 MF 项目
ls module-federation.config.ts module-federation.config.js 2>/dev/null
cat package.json | grep -E '"@module-federation/enhanced"|"@module-federation/core"|"@module-federation/modern-js-v3"'
```

根据检查结果：

- `vmok.config.ts` 存在 或 deps 中包含 `@edenx/plugin-vmok-v3` / `@vmok/kit` → **Vmok 项目**，进入 [步骤 2A](#步骤-2a-vmok-项目)
- `module-federation.config.ts` 存在 或 deps 中包含 `@module-federation/enhanced` / `@module-federation/modern-js-v3` → **标准 MF 项目**，进入 [步骤 2B](#步骤-2b-标准-mf-项目)
- 两者均无 → **尚未接入 MF**，进入 [步骤 2C](#步骤-2c-尚未接入-mf)

---

## 第二步：准备 Module Federation 配置

### 步骤 2A：Vmok 项目

#### 2A.1 确认 `vmok.config.ts` 中有 `exposes`

```ts
// vmok.config.ts
import { createVmokConfig } from '@edenx/plugin-vmok-v3';

export default createVmokConfig({
  name: '@demo/ops-tools',   // ← remote 名称，后续要用
  exposes: {
    './IncidentForm': './src/components/IncidentForm',
    './DeployWizard': './src/components/DeployWizard',
  },
  shared: {
    react: { singleton: true },
    'react-dom': { singleton: true },
  },
});
```

如果 `exposes` 为空，在这里添加你想暴露给 AI 的组件。

#### 2A.2 确认 `baseUrl`

- **本地开发**：`http://localhost:{devPort}/vmok-manifest.json`（问你的项目 dev 端口，默认 `8080`）
- **生产 CDN**：从 Vmok Module Center 获取，格式为 `https://{cdn}/{scope}/{pkg}/{version}/vmok-manifest.json`

> **注意**：Vmok 的 `baseUrl` 必须以 `/vmok-manifest.json` 结尾。

记下 `vmok.config.ts` 里的 `name` 字段值，进入[第三步](#第三步生成-mcp_appsjson)。

---

### 步骤 2B：标准 MF 项目

#### 2B.1 确认 `module-federation.config.ts` 中有 `exposes`

**Rspack / Webpack（`@module-federation/enhanced`）：**

```ts
import { createModuleFederationConfig } from '@module-federation/enhanced/rspack';

export default createModuleFederationConfig({
  name: '@my-org/ops-tools',   // ← remote 名称，后续要用
  exposes: {
    './IncidentForm': './src/components/IncidentForm',
    './UserLookup': './src/components/UserLookup',
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});
```

**Modern.js（`@module-federation/modern-js-v3`）：**

```ts
import { createModuleFederationConfig } from '@module-federation/modern-js-v3';

export default createModuleFederationConfig({
  name: '@my-org/ops-tools',
  exposes: {
    './IncidentForm': './src/components/IncidentForm',
    './UserLookup': './src/components/UserLookup',
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});
```

如果 `exposes` 为空，在这里添加需要暴露的组件。

#### 2B.2 确认 `baseUrl`

- **本地开发**：`http://localhost:{devPort}/mf-manifest.json`
- **生产 CDN**：`https://cdn.example.com/ops-tools/2.4.1/mf-manifest.json`

> **注意**：标准 MF 的 `baseUrl` 必须以 `/mf-manifest.json` 结尾。

记下 `module-federation.config.ts` 里的 `name` 字段值，进入[第三步](#第三步生成-mcp_appsjson)。

---

### 步骤 2C：尚未接入 MF

根据你的构建工具选择：

**Rspack / Webpack：**

```bash
pnpm add @module-federation/enhanced
```

创建 `module-federation.config.ts`：

```ts
import { createModuleFederationConfig } from '@module-federation/enhanced/rspack';

export default createModuleFederationConfig({
  name: '@my-org/app-name',   // ← 填入你的包名
  exposes: {
    './MyComponent': './src/components/MyComponent',  // ← 填入要暴露的组件
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});
```

**Modern.js：**

```bash
pnpm add @module-federation/modern-js-v3
```

在 `modern.config.ts` 中注册插件：

```ts
import { appTools, defineConfig } from '@modern-js/app-tools';
import { moduleFederationPlugin } from '@module-federation/modern-js-v3';

export default defineConfig({
  plugins: [appTools(), moduleFederationPlugin()],
});
```

然后创建与 Rspack 格式相同的 `module-federation.config.ts`（引用改为 `@module-federation/modern-js-v3`）。

**Vite：** 参考 [module-federation.io](https://module-federation.io/zh/guide/start/quick-start.html) 的 Vite 快速上手文档。

配置完成后重新执行[第一步](#第一步检查项目类型)确认检测结果，再进入[第三步](#第三步生成-mcp_appsjson)。

---

## 第三步：生成 `mcp_apps.json`

`mcp_apps.json` 是整个方案的唯一配置入口，告诉 Server 有哪些远程、每个工具对应哪个组件。

### 3.1 复制 schema 文件

```bash
cp node_modules/@module-federation/mcp-apps/mcp_apps.schema.json ./mcp_apps.schema.json
```

如果还没有安装包，可以直接从仓库下载：

```bash
curl -o mcp_apps.schema.json \
  https://raw.githubusercontent.com/module-federation/mcp-apps/main/mcp_apps.schema.json
```

### 3.2 创建配置文件

**Vmok 项目**（`baseUrl` 以 `/vmok-manifest.json` 结尾）：

```json
{
  "$schema": "./mcp_apps.schema.json",
  "remotes": [
    {
      "name": "@demo/ops-tools",
      "version": "local",
      "baseUrl": "http://localhost:8080/vmok-manifest.json",
      "locale": "zh",
      "csp": {
        "connectDomains": ["http://localhost:8080"],
        "resourceDomains": ["http://localhost:8080"]
      }
    }
  ],
  "tools": [
    {
      "name": "incident_form",
      "title": "故障工单",
      "description": "创建和提交线上故障工单，包含严重级别、影响范围和处理记录",
      "inputSchema": { "type": "object", "properties": {} },
      "remote": "@demo/ops-tools",
      "module": "./IncidentForm"
    }
  ]
}
```

**标准 MF 项目**（`baseUrl` 以 `/mf-manifest.json` 结尾）：

```json
{
  "$schema": "./mcp_apps.schema.json",
  "remotes": [
    {
      "name": "@my-org/ops-tools",
      "version": "2.4.1",
      "baseUrl": "https://cdn.example.com/ops-tools/2.4.1/mf-manifest.json",
      "csp": {
        "connectDomains": ["https://cdn.example.com"],
        "resourceDomains": ["https://cdn.example.com"]
      }
    }
  ],
  "tools": [
    {
      "name": "incident_form",
      "title": "故障工单",
      "description": "创建和提交线上故障工单，包含严重级别、影响范围和处理记录",
      "inputSchema": { "type": "object", "properties": {} },
      "remote": "@my-org/ops-tools",
      "module": "./IncidentForm"
    },
    {
      "name": "user_lookup",
      "title": "用户查询",
      "description": "按用户 ID 或邮箱查询用户信息和权限状态",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "用户 ID 或邮箱地址" }
        },
        "required": ["query"]
      },
      "remote": "@my-org/ops-tools",
      "module": "./UserLookup"
    }
  ]
}
```

### 命名规则

- `tools[].name`：去掉 `./` 前缀，PascalCase 转 snake_case，例如 `./IncidentForm` → `incident_form`
- `tools[].description`：写给 AI 看的——描述工具做什么、何时调用，越具体 AI 越容易选中它
- `csp` 域名必须包含 `baseUrl` 的完整 origin（协议 + 主机 + 端口）

### 验证 JSON 格式

```bash
node -e "JSON.parse(require('fs').readFileSync('mcp_apps.json', 'utf8')); console.log('✅ valid JSON')"
```

> **完整字段说明**见 [mcp_apps.json 完全指南](./mcp_apps_json.md)。

---

## 第四步：启动 MCP Server 并注册到 AI 宿主

### 4.1 启动前端 dev server（如果还没有运行）

```bash
pnpm run dev
```

确认 manifest 可访问：

```bash
# Vmok
curl http://localhost:8080/vmok-manifest.json

# 标准 MF
curl http://localhost:8080/mf-manifest.json
```

### 4.2 注册到 Claude Desktop

编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`（macOS）或 `%APPDATA%\Claude\claude_desktop_config.json`（Windows）：

```json
{
  "mcpServers": {
    "my-mcp-app": {
      "command": "npx",
      "args": [
        "-y",
        "@module-federation/mcp-apps@latest",
        "--config",
        "/绝对路径/mcp_apps.json",
        "--stdio"
      ]
    }
  }
}
```

> **nvm 用户注意**：Claude Desktop 不继承 shell 的 PATH，`npx` 可能找不到正确的 Node 版本。用 `which npx` 获取完整路径：
>
> ```json
> {
>   "mcpServers": {
>     "my-mcp-app": {
>       "command": "/Users/you/.nvm/versions/node/v22.x.x/bin/npx",
>       "args": ["-y", "@module-federation/mcp-apps@latest", "--config", "/绝对路径/mcp_apps.json", "--stdio"],
>       "env": {
>         "PATH": "/Users/you/.nvm/versions/node/v22.x.x/bin:/usr/local/bin:/usr/bin:/bin"
>       }
>     }
>   }
> }
> ```

### 4.3 重启并验证

1. 完全退出并重启 Claude Desktop
2. 新建对话，询问："你有哪些 MCP 工具？"
3. 确认 `mcp_apps.json` 里声明的工具名出现在列表中
4. 让 AI 调用其中一个工具，在聊天窗口里看到组件渲染

---

## 第五步：让组件与 AI 通信（可选）

如果组件只需要渲染，不需要和 AI 交互，跳过此步骤。

如果你希望用户操作后**推进对话**——比如表单提交后告诉 AI 下一步该做什么——可以使用 `mcpApp` prop。

`mcpApp` 由框架在渲染时**自动注入为 prop**，不需要安装任何依赖，也不需要任何 import。在组件 props 里声明接口即可使用：

```tsx
interface McpApp {
  /** 调用另一个 MCP 工具，在组件内使用返回结果 */
  callServerTool: (params: {
    name: string;
    arguments?: Record<string, unknown>;
  }) => Promise<{
    content: Array<{ type: string; text?: string }>;
    isError?: boolean;
  }>;

  /** 向 AI 对话注入一条消息，触发下一轮 Agent 推理 */
  sendMessage?: (params: {
    role: string;
    content: Array<{ type: string; text: string }>;
  }) => Promise<{ isError?: boolean }>;
}

export default function IncidentForm({ mcpApp }: { mcpApp?: McpApp }) {
  const handleSubmit = async (incident) => {
    // 正常的业务逻辑
    await fetch('/api/incidents', { method: 'POST', body: JSON.stringify(incident) });

    // 告诉 AI 发生了什么，让对话继续
    await mcpApp?.sendMessage({
      role: 'user',
      content: [{ type: 'text', text: `故障单 #${incident.id} 已创建，严重级别：${incident.severity}` }],
    });
  };

  return <IncidentFormUI onSubmit={handleSubmit} />;
}
```

`mcpApp` 在组件运行于 MCP 宿主之外时为 `undefined`，可选链 `?.` 静默跳过，**现有行为零侵入**。

### `callServerTool` vs `sendMessage`

两个方法用途不同：

- **`callServerTool`**：在组件内部调用另一个 MCP 工具并直接使用返回值（例如：枚举下拉选项、查询状态）
- **`sendMessage`**：把消息注入对话上下文，让 AI 读到后自动触发下一步（例如：提交后告诉 AI 什么完成了、触发多步骤向导的下一个工具）

### 多步骤向导模式

使用多个 MCP 工具串联为向导：Step N 的组件在用户确认后，通过 `sendMessage` 告诉 AI 调用 Step N+1，并把当前步骤的数据通过 `arguments` 传递下去。详细实现模式见 [develop-mcp-app-component.md](../../skill/develop-mcp-app-component.md)。

---

## 测试你的 MCP App

### 使用 Claude 测试

[Claude.ai](https://claude.ai/)（Web 版）和 [Claude Desktop](https://claude.ai/download) 均支持 MCP Apps。本地开发时，如果使用 Web 版 Claude，需要把本地服务器通过内网穿透暴露到公网：

```bash
npx cloudflared tunnel --url http://localhost:3001
```

复制生成的 URL，在 Claude 中通过**设置 → Connectors → 添加自定义连接器**接入。

使用 Claude Desktop 时，按[第四步](#第四步启动-mcp-server-并注册到-ai-宿主)配置后直接测试即可。

### 验证清单

- [ ] 前端 dev server 运行中，manifest URL 可访问（`curl` 测试通过）
- [ ] `mcp_apps.json` 是合法 JSON，`baseUrl` 后缀正确（Vmok：`/vmok-manifest.json`；标准 MF：`/mf-manifest.json`）
- [ ] Claude Desktop 配置里的路径是绝对路径
- [ ] 重启后 AI 列出了预期的工具名
- [ ] 调用工具后在对话中看到组件渲染
- [ ] （如已实现）用户操作后对话正常推进

---

## 常见问题

**工具重启后不出现**

- 确认 Claude Desktop 配置里的路径是绝对路径，不是相对路径
- 确认 `npx` 能解析到正确的 Node 版本（`node --version`）
- 手动运行命令检查报错：
  ```bash
  npx -y @module-federation/mcp-apps@latest --config /path/to/mcp_apps.json --stdio
  ```

**组件加载但显示空白 / CSP 报错**

- 在 iframe 的 DevTools Console 里查看被拦截的 URL
- 把缺少的 origin 加到 `mcp_apps.json` 里的 `csp.connectDomains` 和 `csp.resourceDomains`
- 修改 `mcp_apps.json` 后需要重启 Claude Desktop（Server 在启动时读取配置）

**`remoteEntry` / manifest 404**

- Vmok：确认 `baseUrl` 路径是 `/vmok-manifest.json`，不是 `/remoteEntry.js`
- 标准 MF：确认路径是 `/mf-manifest.json`，不是 `/remoteEntry.js`
- 检查 `baseUrl` 在 manifest 文件名前没有多余的 `/`

**nvm / node 在 Claude Desktop 中找不到**

使用 `which npx` 获取完整路径，并在配置中通过 `env.PATH` 固定 Node 路径（参考[第四步](#42-注册到-claude-desktop)的 nvm 示例）。

---

## 部署到生产

本地验证通过后，将组件部署到 CDN，更新 `mcp_apps.json` 里的 `baseUrl` 和 `version` 字段，重启 Server 即可切换到生产版本：

```json
{
  "remotes": [
    {
      "name": "@my-org/ops-tools",
      "version": "2.4.1",
      "baseUrl": "https://cdn.example.com/ops-tools/2.4.1/mf-manifest.json",
      "csp": {
        "connectDomains": ["https://cdn.example.com"],
        "resourceDomains": ["https://cdn.example.com"]
      }
    }
  ]
}
```

Server 本身**不需要重新构建**，也不需要动任何代码——`mcp_apps.json` 是唯一需要更新的地方。

---

## 延伸阅读

- [mcp_apps.json 完全指南](./mcp_apps_json.md) — 所有字段的语义、约束条件和 CSP 规则
- [组件开发指南](./02-component-guide.md) — `mcpApp` prop 完整 API、多步骤向导模式、双渠道通信
- [HTTP 模式 / 自定义 Agent 接入](./05-custom-agent-integration.md) — 在自建 Agent 框架中使用 HTTP 模式
- [MCP Apps 规范](https://modelcontextprotocol.io/docs/extensions/apps)
- [Module Federation 快速上手](https://module-federation.io/zh/guide/start/quick-start.html)
