# mcp_apps.json Complete Guide

## What It Is

`mcp_apps.json` is the bridge configuration file that connects **Module Federation remotes** to **AI Agent tools**.

Its core responsibility is to answer two questions:

1. **Where does the UI come from?** — `remotes[]` describes where your MF remote modules are deployed
2. **What tools to register?** — `tools[]` describes which tools the AI Agent can call and which component to render when called

---

## Why It's Needed

### The Problem

The MCP Apps standard requires the MCP Server to provide an HTML resource for each tool. The conventional approach bundles UI components directly into the MCP Server, coupling them with server-side code.

This creates a fundamental tension: **the UI lifecycle and the MCP Server lifecycle are independent**.

```
The conventional problem:

MCP Server
 ├── index.js          ← server logic
 └── static/
      ├── tool1.html   ← UI bundled here, tightly coupled to the server
      └── tool2.html   ← rebuilding UI means redeploying the whole server
```

But your MF remotes already have their own independent CI/CD pipeline and are deployed on a CDN:

```
Your existing MF deployment:

CDN (https://cdn.example.com)
 └── shop-frontend/2.1.0/
      ├── mf-manifest.json      ← MF entry point
      ├── ProductList.js        ← your component
      └── OrderHistory.js
```

### How mcp_apps.json Solves It

`mcp_apps.json` turns the MCP Server into a **pure config-driven forwarding layer** — it never holds any UI code:

```
MCP Server receives tool call
  ↓
Reads mcp_apps.json, finds remote + module config for the tool
  ↓
Generates a resource URI: ui://mf/<remote-slug>
  ↓
AI Host (Claude) receives the URI, opens the render container iframe
  ↓
Render container dynamically loads the MF remote from CDN, renders the component
```

The MCP Server doesn't need to know what the component looks like, and doesn't need to be redeployed when you ship a new version of your UI.

---

## Role in the Overall Architecture

```
┌─────────────────────────────────────────────────────┐
│                 What the developer writes            │
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
                                         │ read on startup
                                         ▼
┌─────────────────────────────────────────────────────┐
│                   MCP Server runtime                 │
│                                                     │
│  loadConfig(mcp_apps.json)                          │
│    → register MCP tools (tools[])                   │
│    → register render resources (remotes[])          │
│                                                     │
│  On tool call:                                      │
│    tool.remote ──▶ look up remote.baseUrl           │
│    tool.module ──▶ assemble moduleFederation config │
│               ──▶ return to AI Host                 │
└───────────────────────────┬─────────────────────────┘
                            │ MCP Protocol
                            ▼
┌─────────────────────────────────────────────────────┐
│                  AI Host (Claude)                    │
│                                                     │
│  Receives tool result containing ui://mf/<slug>     │
│  Opens iframe in the conversation, loads the shell  │
└───────────────────────────┬─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│              Render Container (mf-render-container)  │
│                                                     │
│  Receives moduleFederation config:                  │
│    remoteName: 'shop_frontend'                      │
│    remoteEntry: 'https://cdn.../mf-manifest.json'   │
│    module: './ProductList'                          │
│                                                     │
│  → calls createInstance() + loadRemote()            │
│  → loads component from CDN and renders it          │
└─────────────────────────────────────────────────────┘
```

---

## Field Reference and Mapping to MF Config

### Top-level structure

```json
{
  "$schema": "./mcp_apps.schema.json",
  "version": "1.0.0",
  "remotes": [...],
  "tools": [...]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `$schema` | string | Recommended | Points to the schema file; IDEs use it for field completion and validation |
| `version` | string | No | Config file version; informational only, does not affect runtime |
| `remotes` | array | ✅ | List of MF remote declarations |
| `tools` | array | ✅ | List of MCP tool declarations |

---

### `remotes[]` — MF module declarations

Each entry corresponds to one MF remote package deployed on your CDN.

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

#### `name` · string · required

The unique identifier for the remote module. **Must exactly match the `name` field in `module-federation.config.ts` on the producer side.**

```typescript
// module-federation.config.ts (producer side)
export default {
  name: 'shop_frontend',   // ← this value
  exposes: { ... }
};
```

```json
// mcp_apps.json
"name": "shop_frontend"   // ← must match exactly
```

This is because the MF runtime uses this name to call `loadRemote('shop_frontend/ProductList')` — the wrong name means the module can't be found.

---

#### `baseUrl` · string · required

The full URL of the MF manifest file, corresponding to the entry file in your MF build output.

```json
// Standard MF (rsbuild / webpack / rspack)
"baseUrl": "https://cdn.example.com/shop-frontend/v2.1.0/mf-manifest.json"

// Local development
"baseUrl": "http://localhost:8080/mf-manifest.json"

// ByteDance vmok format
"baseUrl": "https://lf-cdn.example.com/obj/xxx/vmok-manifest.json"
```

**This URL is passed directly as `remoteEntry` to `createInstance()`.**

```typescript
// Inside mf-loader.ts
createInstance({
  remotes: [{
    name: remoteName,
    entry: remoteEntry,   // ← this is the baseUrl value
  }]
})
```

---

#### `manifestType` · `"mf"` | `"vmok"` · optional, default `"mf"`

The format of the manifest file:

| Value | Use case | Manifest filename |
|-------|----------|------------------|
| `"mf"` | Standard Module Federation (rsbuild, webpack, rspack) | `mf-manifest.json` |
| `"vmok"` | ByteDance internal vmok format (ByteDance projects only) | `vmok-manifest.json` |

For external developers, always use `"mf"`.

---

#### `csp` · object · required

Content Security Policy configuration. The render container runs inside an AI Host iframe subject to strict CSP. The domains declared here are injected into the iframe's CSP header.

```json
"csp": {
  "connectDomains": ["https://cdn.example.com", "https://api.example.com"],
  "resourceDomains": ["https://cdn.example.com"]
}
```

| Field | CSP directive | Purpose |
|-------|--------------|---------|
| `connectDomains` | `connect-src` | Target domains for fetch/XHR requests made inside the component |
| `resourceDomains` | `script-src` + `img-src` + `style-src` + `font-src` | Source domains for MF remote JS/CSS/image/font assets |

**Important notes:**
- Must include the protocol: `"https://cdn.example.com"` ✅, `"cdn.example.com"` ❌
- The domain of `baseUrl` must appear in `resourceDomains`, otherwise the MF module won't load
- Local development: `"http://localhost:8080"` ✅ (localhost requires http, not https)

---

### `tools[]` — MCP tool declarations

Each entry corresponds to one tool registered with the MCP Server that the AI can call to trigger UI rendering.

```json
{
  "name": "product_list",
  "title": "Product List",
  "description": "Renders the product list UI so users can filter by category and select items.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "category": { "type": "string", "description": "Product category" }
    }
  },
  "remote": "shop_frontend",
  "module": "./ProductList",
  "exportName": "default",
  "visibility": ["model", "app"]
}
```

---

#### `name` · string · required

Unique tool identifier, globally unique, snake_case format.

The AI calls the tool by this name: `call_tool("product_list", { category: "electronics" })`

---

#### `title` · string · required

Human-readable tool name, displayed in the AI Host's tool list UI.

---

#### `description` · string · required

**One of the most critical fields in the entire config.** The AI model reads `description` to decide when and with what arguments to call the tool.

Good descriptions:
- Clearly explain what the tool does and when it should be used
- Describe what arguments mean and where they come from (especially for multi-step wizard scenarios)
- If the tool should not be called directly by the AI (only triggered by other components), explicitly say so

```json
// Good ✅
"description": "Step 1: shows the app selection UI. After the user confirms, the component automatically triggers deploy_wizard_step2. Do NOT call step2 manually."

// Vague ❌
"description": "Deploy wizard step 1"
```

---

#### `inputSchema` · object · optional, default `{}`

The tool's parameter schema, following JSON Schema draft-07 format. The AI constructs call arguments from this schema, and the arguments are passed as `args` to the rendered remote component:

```tsx
// Inside the render container
<RemoteComponent {...args} mcpApp={app} />
//               ^^^^^ these are the parameters defined in inputSchema
```

```json
"inputSchema": {
  "type": "object",
  "properties": {
    "category": {
      "type": "string",
      "description": "Product category, e.g. electronics, clothing"
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

#### `remote` · string · required

Which remote module from `remotes[]` to use. Must match the corresponding `name` value.

```json
"remotes": [{ "name": "shop_frontend", ... }],
"tools": [{
  "remote": "shop_frontend"   // ← must correspond to the name above
}]
```

The server validates on startup: every tool's `remote` value must match a `name` in `remotes[]`.

---

#### `module` · string · required

**Directly corresponds to the key in the `exposes` map of `module-federation.config.ts`.**

```typescript
// module-federation.config.ts (producer side)
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
{ "module": "./ProductList"  }   // ← must match the exposes key
{ "module": "./OrderHistory" }
```

This value is ultimately passed to `mf.loadRemote('shop_frontend/ProductList')`:

```typescript
// Path transformation logic inside mf-loader.ts
const normalizedPath = modulePath.replace(/^\.\//, '');  // 'ProductList'
const fullPath = `${remoteName}/${normalizedPath}`;       // 'shop_frontend/ProductList'
mf.loadRemote(fullPath);
```

| `exposes` key | `module` field | `loadRemote` call |
|--------------|---------------|------------------|
| `'./ProductList'` | `"./ProductList"` | `shop_frontend/ProductList` |
| `'./ui/Button'` | `"./ui/Button"` | `shop_frontend/ui/Button` |
| `'.'` (root expose) | `"."` | `shop_frontend` |

---

#### `exportName` · string · optional, default `"default"`

Which export to take from the module.

```typescript
// Producer component

// Default export → exportName: "default" (or omit the field)
export default function ProductList() { ... }

// Named export → exportName: "ProductList"
export function ProductList() { ... }

// Both present → use exportName to specify which one
export default function App() { ... }
export function ProductList() { ... }
```

```json
{ "exportName": "default" }     // export default
{ "exportName": "ProductList" } // export { ProductList }
```

---

#### `visibility` · `("model" | "app")[]` · optional, default `["model", "app"]`

Controls who can see and call the tool:

| Value | Meaning |
|-------|---------|
| `"model"` | The AI model can see and proactively call this tool |
| `"app"` | Other MCP App components (React components inside iframes) can call it via `mcpApp.callTool()` |

In multi-step wizard scenarios, intermediate steps are set to `["app"]` — triggered only by the previous component, preventing the AI from calling them directly:

```json
{ "name": "deploy_wizard_step1", "visibility": ["model", "app"] },
{ "name": "deploy_wizard_step2", "visibility": ["app"] },
{ "name": "deploy_wizard_step3", "visibility": ["app"] }
```

---

## Complete Field-to-MF Config Mapping

```
module-federation.config.ts          mcp_apps.json
────────────────────────────         ─────────────────────────────
name: 'shop_frontend'           ──▶  remotes[].name: 'shop_frontend'
                                     tools[].remote: 'shop_frontend'

exposes: {                      
  './ProductList': './src/...'  ──▶  tools[].module: './ProductList'
}

CDN build artifacts             
mf-manifest.json URL            ──▶  remotes[].baseUrl
```

In one sentence: **`remotes[].name` maps to the MF package's `name`, `tools[].module` maps to the MF package's `exposes` key, and `remotes[].baseUrl` maps to the build artifact's manifest URL.**

---

## Full Example

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
      "title": "Product List",
      "description": "Renders the product list UI. Users can filter by category and select items. Selection automatically triggers the order_form tool.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "category": {
            "type": "string",
            "description": "Product category, e.g. electronics, clothing, food"
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
      "title": "Order Form",
      "description": "Triggered automatically by product_list — do NOT call directly. Renders the order form, receives product info, and completes order submission.",
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
