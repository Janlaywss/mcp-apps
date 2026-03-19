# mcp_apps.json 完全指南

## 它是什么

`mcp_apps.json` 是连接 **Module Federation 生产者** 和 **AI Agent 工具** 的桥梁配置文件。

它的核心职责是回答两个问题：

1. **去哪里拿 UI？** — `remotes[]` 描述你的 MF 远程模块部署在哪里
2. **注册什么工具？** — `tools[]` 描述 AI Agent 可以调用哪些工具、调用时渲染哪个组件

---

## 为什么需要它

### 问题背景

MCP Apps 标准要求 MCP Server 为每个工具提供一个 HTML 资源。传统做法是把 UI 组件打包进 MCP Server，和服务端代码捆绑在一起。

这带来一个根本矛盾：**UI 的生命周期和 MCP Server 的生命周期是不同的**。

```
传统方式的困境：

MCP Server
 ├── index.js          ← 服务端逻辑
 └── static/
      ├── tool1.html   ← UI 打包在这里，与服务端耦合
      └── tool2.html   ← 每次更新 UI 都要重新打包部署整个 Server
```

而你的 MF 远程模块已经有自己独立的 CI/CD 流程，部署在 CDN 上：

```
已有的 MF 部署：

CDN (https://cdn.example.com)
 └── shop-frontend/2.1.0/
      ├── mf-manifest.json      ← MF 入口
      ├── ProductList.js        ← 你的组件
      └── OrderHistory.js
```

### mcp_apps.json 的解决思路

`mcp_apps.json` 让 MCP Server 变成一个**纯配置驱动的转发层**，不再持有任何 UI 代码：

```
MCP Server 收到工具调用
  ↓
读取 mcp_apps.json，找到对应工具的 remote + module 配置
  ↓
生成一个资源 URI：ui://mf/<remote-slug>
  ↓
AI Host（Claude）拿到 URI，打开渲染容器 iframe
  ↓
渲染容器从 CDN 动态加载 MF 远程模块，渲染组件
```

MCP Server 不需要知道组件长什么样，也不需要在发布新版 UI 时重新部署。

---

## 在整体架构中的位置

```
┌─────────────────────────────────────────────────────┐
│                   开发者写什么                        │
│                                                     │
│  module-federation.config.ts    mcp_apps.json       │
│  ┌─────────────────────────┐   ┌─────────────────┐  │
│  │ name: 'shop_frontend'   │   │ remotes[].name  │  │
│  │ exposes: {              │──▶│ = 'shop_frontend'│  │
│  │  './ProductList': ...   │   │                 │  │
│  │  './OrderHistory': ...  │   │ tools[].module  │  │
│  │ }                       │──▶│ = './ProductList'│  │
│  └─────────────────────────┘   └────────┬────────┘  │
└────────────────────────────────────────┼────────────┘
                                         │ 启动时读取
                                         ▼
┌─────────────────────────────────────────────────────┐
│                   MCP Server 运行时                   │
│                                                     │
│  loadConfig(mcp_apps.json)                          │
│    → 注册 MCP 工具 (tools[])                         │
│    → 注册渲染资源 (remotes[])                         │
│                                                     │
│  工具调用时：                                         │
│    tool.remote ──▶ 找到 remote.baseUrl               │
│    tool.module ──▶ 组装 moduleFederation config      │
│               ──▶ 返回给 AI Host                     │
└───────────────────────────┬─────────────────────────┘
                            │ MCP Protocol
                            ▼
┌─────────────────────────────────────────────────────┐
│                  AI Host (Claude)                    │
│                                                     │
│  接收工具结果，包含 ui://mf/<slug> 资源 URI            │
│  在对话中打开 iframe，加载渲染容器                       │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│              渲染容器 (mf-render-container)           │
│                                                     │
│  收到 moduleFederation config：                      │
│    remoteName: 'shop_frontend'                      │
│    remoteEntry: 'https://cdn.../mf-manifest.json'   │
│    module: './ProductList'                          │
│                                                     │
│  → 调用 createInstance() + loadRemote()             │
│  → 从 CDN 加载组件并渲染                              │
└─────────────────────────────────────────────────────┘
```

---

## 字段详解及与 MF 配置的对应关系

### 顶层结构

```json
{
  "$schema": "./mcp_apps.schema.json",
  "version": "1.0.0",
  "remotes": [...],
  "tools": [...]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `$schema` | string | 推荐 | 指向 schema 文件，IDE 会提供字段补全和校验 |
| `version` | string | 否 | 配置文件版本，仅作标注，不影响运行 |
| `remotes` | array | ✅ | MF 远程模块声明列表 |
| `tools` | array | ✅ | MCP 工具声明列表 |

---

### `remotes[]` — MF 模块声明

每一条 remote 对应你 CDN 上部署的一个 MF 远程包。

```json
{
  "name": "shop_frontend",
  "version": "2.1.0",
  "baseUrl": "https://cdn.example.com/shop-frontend/2.1.0/mf-manifest.json",
  "manifestType": "mf",
  "csp": {
    "connectDomains": ["https://cdn.example.com", "https://api.example.com"],
    "resourceDomains": ["https://cdn.example.com"]
  }
}
```

#### `name` · string · 必填

远程模块的唯一标识符，**必须与 `module-federation.config.ts` 中的 `name` 字段完全一致**。

```typescript
// module-federation.config.ts（生产者端）
export default {
  name: 'shop_frontend',   // ← 这个值
  exposes: { ... }
};
```

```json
// mcp_apps.json
"name": "shop_frontend"   // ← 必须一致
```

这是因为 MF runtime 用这个名字去 `loadRemote('shop_frontend/ProductList')` —— 名字不对就找不到模块。

---

#### `baseUrl` · string · 必填

MF manifest 文件的完整 URL，对应 MF 构建产物里的入口文件。

```json
// 标准 MF（rsbuild / webpack / rspack）
"baseUrl": "https://cdn.example.com/shop-frontend/v2.1.0/mf-manifest.json"

// 本地开发
"baseUrl": "http://localhost:8080/mf-manifest.json"

// ByteDance vmok 格式
"baseUrl": "https://lf-cdn.example.com/obj/xxx/vmok-manifest.json"
```

**这个 URL 直接作为 `remoteEntry` 传给 `createInstance()`。**

```typescript
// mf-loader.ts 内部
createInstance({
  remotes: [{
    name: remoteName,
    entry: remoteEntry,   // ← 就是 baseUrl 的值
  }]
})
```

---

#### `manifestType` · `"mf"` | `"vmok"` · 可选，默认 `"mf"`

manifest 文件的格式：

| 值 | 适用场景 | manifest 文件名 |
|----|---------|----------------|
| `"mf"` | 标准 Module Federation（rsbuild、webpack、rspack） | `mf-manifest.json` |
| `"vmok"` | ByteDance 内部 vmok 格式（字节内部项目专用） | `vmok-manifest.json` |

对于外部开发者，始终使用 `"mf"`。

---

#### `csp` · object · 必填

Content Security Policy 配置。渲染容器运行在 AI Host 的 iframe 内，受严格 CSP 约束。这里声明的域名会注入到 iframe 的 CSP header 中。

```json
"csp": {
  "connectDomains": ["https://cdn.example.com", "https://api.example.com"],
  "resourceDomains": ["https://cdn.example.com"]
}
```

| 字段 | CSP 指令 | 用途 |
|------|---------|------|
| `connectDomains` | `connect-src` | 组件内 fetch/XHR 请求的目标域名 |
| `resourceDomains` | `script-src` + `img-src` + `style-src` + `font-src` | MF 远程 JS/CSS/图片/字体资源的来源域名 |

**注意事项：**
- 必须带协议头：`"https://cdn.example.com"` ✅，`"cdn.example.com"` ❌
- `baseUrl` 所在域必须出现在 `resourceDomains` 中，否则 MF 模块无法加载
- 本地开发：`"http://localhost:8080"` ✅（localhost 需要 http，不是 https）

---

### `tools[]` — MCP 工具声明

每一条 tool 对应一个会被注册到 MCP Server 的工具，AI 可以调用它来触发 UI 渲染。

```json
{
  "name": "product_list",
  "title": "商品列表",
  "description": "展示商品列表，用户可以筛选和选择商品",
  "inputSchema": {
    "type": "object",
    "properties": {
      "category": { "type": "string", "description": "商品分类" }
    }
  },
  "remote": "shop_frontend",
  "module": "./ProductList",
  "exportName": "default",
  "visibility": ["model", "app"]
}
```

---

#### `name` · string · 必填

工具的唯一标识符，全局唯一，snake_case 格式。

AI 通过这个名字调用工具：`call_tool("product_list", { category: "electronics" })`

```json
"name": "product_list"
"name": "deploy_wizard_step1"
"name": "order_history"
```

---

#### `title` · string · 必填

工具的人类可读名称，显示在 AI Host 的工具列表 UI 中。

---

#### `description` · string · 必填

**这是整个配置中最关键的字段之一。** AI 模型读取 description 来决定什么时候、用什么参数调用这个工具。

描述需要：
- 说清楚工具做什么、何时应该用
- 说明参数的含义及来源（尤其是多步骤 wizard 场景）
- 如果工具不应被 AI 主动调用（只被其他组件触发），要明确说明

```json
// 好的描述 ✅
"description": "第一步：展示应用选择 UI。用户选择应用和环境后，组件自动触发 deploy_wizard_step2。请勿手动调用 step2。"

// 模糊的描述 ❌
"description": "部署向导第一步"
```

---

#### `inputSchema` · object · 可选，默认 `{}`

工具的参数 Schema，遵循 JSON Schema draft-07 格式。AI 按照这个 Schema 构造调用参数，参数会作为 `args` 传给渲染的远程组件：

```tsx
// 渲染容器内部
<RemoteComponent {...args} mcpApp={app} />
//               ^^^^^ 就是 inputSchema 定义的参数
```

```json
"inputSchema": {
  "type": "object",
  "properties": {
    "category": {
      "type": "string",
      "description": "商品分类，如 electronics、clothing"
    },
    "pageSize": {
      "type": "number",
      "default": 20
    }
  },
  "required": ["category"]
}
```

---

#### `remote` · string · 必填

引用 `remotes[]` 中的哪个远程模块，填写对应的 `name` 值。

```json
"remotes": [{ "name": "shop_frontend", ... }],
"tools": [{
  "remote": "shop_frontend"   // ← 必须与上面的 name 对应
}]
```

Server 启动时会校验：所有 tool 的 `remote` 值都必须能在 `remotes[]` 中找到对应的 `name`。

---

#### `module` · string · 必填

**与 `module-federation.config.ts` 中 `exposes` 的 key 完全对应。**

```typescript
// module-federation.config.ts（生产者端）
export default {
  name: 'shop_frontend',
  exposes: {
    './ProductList':  './src/components/ProductList.tsx',  // ← key
    './OrderHistory': './src/components/OrderHistory.tsx', // ← key
  }
};
```

```json
// mcp_apps.json
{ "module": "./ProductList"  }   // ← 必须与 exposes key 一致
{ "module": "./OrderHistory" }
```

这个值最终会传给 `mf.loadRemote('shop_frontend/ProductList')`：

```typescript
// mf-loader.ts 内部的转换逻辑
const normalizedPath = modulePath.replace(/^\.\//, '');  // 'ProductList'
const fullPath = `${remoteName}/${normalizedPath}`;       // 'shop_frontend/ProductList'
mf.loadRemote(fullPath);
```

| `exposes` key | `module` 字段 | `loadRemote` 调用 |
|--------------|--------------|-----------------|
| `'./ProductList'` | `"./ProductList"` | `shop_frontend/ProductList` |
| `'./ui/Button'` | `"./ui/Button"` | `shop_frontend/ui/Button` |
| `'.'` (根暴露) | `"."` | `shop_frontend` |

---

#### `exportName` · string · 可选，默认 `"default"`

从模块中取哪个导出。

```typescript
// 生产者组件

// 默认导出 → exportName: "default"（或省略）
export default function ProductList() { ... }

// 具名导出 → exportName: "ProductList"
export function ProductList() { ... }

// 两者都有 → 用 exportName 指定取哪个
export default function App() { ... }
export function ProductList() { ... }
```

```json
{ "exportName": "default" }     // export default
{ "exportName": "ProductList" } // export { ProductList }
```

---

#### `visibility` · `("model" | "app")[]` · 可选，默认 `["model", "app"]`

控制工具对谁可见：

| 值 | 含义 |
|----|------|
| `"model"` | AI 模型可以看到并主动调用这个工具 |
| `"app"` | 其他 MCP App 组件（iframe 内的 React 组件）可以通过 `mcpApp.callTool()` 调用 |

多步骤 wizard 场景中，中间步骤设置为 `["app"]` — 只由前一个组件触发，防止 AI 误调：

```json
{ "name": "deploy_wizard_step1", "visibility": ["model", "app"] },
{ "name": "deploy_wizard_step2", "visibility": ["app"] },
{ "name": "deploy_wizard_step3", "visibility": ["app"] }
```

---

## 字段与 MF 配置的完整对应关系

```
module-federation.config.ts          mcp_apps.json
────────────────────────────         ─────────────────────────────
name: 'shop_frontend'           ──▶  remotes[].name: 'shop_frontend'
                                     tools[].remote: 'shop_frontend'

exposes: {                      
  './ProductList': './src/...'  ──▶  tools[].module: './ProductList'
}

CDN 部署产物                    
mf-manifest.json URL            ──▶  remotes[].baseUrl
```

一句话总结：**`remotes[].name` 对应 MF 包的 `name`，`tools[].module` 对应 MF 包的 `exposes` key，`remotes[].baseUrl` 对应构建产物的 manifest URL。**

---

## 完整示例

```json
{
  "$schema": "./mcp_apps.schema.json",
  "version": "1.0.0",
  "remotes": [
    {
      "name": "shop_frontend",
      "version": "2.1.0",
      "baseUrl": "https://cdn.example.com/shop-frontend/2.1.0/mf-manifest.json",
      "manifestType": "mf",
      "csp": {
        "connectDomains": [
          "https://cdn.example.com",
          "https://api.example.com"
        ],
        "resourceDomains": [
          "https://cdn.example.com"
        ]
      }
    }
  ],
  "tools": [
    {
      "name": "product_list",
      "title": "商品列表",
      "description": "展示商品列表 UI，用户可以按分类筛选并选择商品。选择后自动触发 order_form 工具。",
      "inputSchema": {
        "type": "object",
        "properties": {
          "category": {
            "type": "string",
            "description": "商品分类，如 electronics、clothing、food"
          }
        }
      },
      "remote": "shop_frontend",
      "module": "./ProductList",
      "exportName": "default",
      "visibility": ["model", "app"]
    },
    {
      "name": "order_form",
      "title": "下单表单",
      "description": "由 product_list 自动触发，勿手动调用。展示下单表单，接收商品信息并完成订单提交。",
      "inputSchema": {
        "type": "object",
        "properties": {
          "productId": { "type": "string" },
          "productName": { "type": "string" },
          "price": { "type": "number" }
        },
        "required": ["productId", "productName", "price"]
      },
      "remote": "shop_frontend",
      "module": "./OrderForm",
      "exportName": "default",
      "visibility": ["app"]
    }
  ]
}
```
