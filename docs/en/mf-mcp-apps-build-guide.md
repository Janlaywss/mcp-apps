# Integrating MCP Apps (Module Federation Approach)

Turn your Module Federation components into interactive MCP tools inside AI hosts like Claude Desktop, VS Code, and more.

---

## Prerequisites

Requires [Node.js](https://nodejs.org/en/download) 22 or higher. It helps to have a basic understanding of [MCP tools](https://modelcontextprotocol.io/specification/latest/server/tools) and the [MCP Apps standard](https://modelcontextprotocol.io/docs/extensions/apps). Teams with an existing Module Federation or Vmok project can jump straight to **Step 2**.

---

## Quick Start

The fastest path is to use an AI coding assistant to complete the integration automatically. If you prefer to do it manually, skip to the "Manual Integration" section.

### Using an AI Coding Assistant

`@module-federation/mcp-apps` ships a set of Skills that let your AI coding assistant automatically handle project-type detection, `mcp_apps.json` generation, and MCP Server registration — no MCP code to write yourself.

#### Step 1: Install the skill

If you use Claude Code, you can install directly with:

```bash
/plugin marketplace add module-federation/mcp-apps
/plugin install integrate-mcp-apps@module-federation-mcp-apps
```

Or clone the repo and copy the skill directory to your AI assistant's config path:

```bash
git clone https://github.com/module-federation/mcp-apps.git
```

Copy the files under `mcp-apps/module-federation-mcp/skill/` into your assistant's skills folder:

| Assistant | macOS / Linux | Windows |
|-----------|---------------|---------|
| Claude Code | `~/.claude/skills/` | `%USERPROFILE%\.claude\skills\` |
| VS Code / GitHub Copilot | `~/.copilot/skills/` | `%USERPROFILE%\.copilot\skills\` |
| Cursor | `~/.cursor/skills/` | `%USERPROFILE%\.cursor\skills\` |
| Goose | `~/.config/goose/skills/` | `%USERPROFILE%\.config\goose\skills\` |

To verify the skill is installed, ask your assistant: "What skills do you have?" — you should see `integrate-mcp-apps` in the list.

#### Step 2: Tell the assistant to start the integration

From your frontend project root, tell your AI assistant:

```
Please follow the integrate-mcp-apps skill to integrate this project with MCP Apps
```

The assistant will detect your project type (Vmok / standard MF / no MF), then automatically walk through everything from config to launch, ending with your tools appearing in Claude Desktop.

#### Step 3: Test

Follow the "Testing Your MCP App" section below to verify.

---

### Manual Integration

If you are not using an AI coding assistant, or want to understand each step, follow the flow below.

---

## Step 1: Detect Your Project Type

Different frontend frameworks use different Module Federation config files, which affects the integration path. Run the following in your project root:

```bash
# Check for a Vmok project
ls vmok.config.ts edenx.config.ts 2>/dev/null
cat package.json | grep -E '"@edenx/plugin-vmok-v3"|"@edenx/plugin-vmok"|"@vmok/kit"'

# Check for a standard MF project
ls module-federation.config.ts module-federation.config.js 2>/dev/null
cat package.json | grep -E '"@module-federation/enhanced"|"@module-federation/core"|"@module-federation/modern-js-v3"'
```

Based on the results:

- `vmok.config.ts` present, or deps include `@edenx/plugin-vmok-v3` / `@vmok/kit` → **Vmok project** → go to [Step 2A](#step-2a-vmok-project)
- `module-federation.config.ts` present, or deps include `@module-federation/enhanced` / `@module-federation/modern-js-v3` → **Standard MF project** → go to [Step 2B](#step-2b-standard-mf-project)
- Neither → **No MF yet** → go to [Step 2C](#step-2c-no-mf-yet)

---

## Step 2: Prepare the Module Federation Config

### Step 2A: Vmok Project

#### 2A.1 Confirm `exposes` in `vmok.config.ts`

```ts
// vmok.config.ts
import { createVmokConfig } from '@edenx/plugin-vmok-v3';

export default createVmokConfig({
  name: '@demo/ops-tools',   // ← remote name, needed later
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

If `exposes` is empty, add the components you want to expose to AI here.

#### 2A.2 Determine your `baseUrl`

- **Local dev**: `http://localhost:{devPort}/vmok-manifest.json` (check your project's dev port, default `8080`)
- **Production CDN**: Look it up in the Vmok Module Center. Format: `https://{cdn}/{scope}/{pkg}/{version}/vmok-manifest.json`

> **Important**: For Vmok, `baseUrl` must end with `/vmok-manifest.json`.

Note the `name` value from `vmok.config.ts` and proceed to [Step 3](#step-3-generate-mcp_appsjson).

---

### Step 2B: Standard MF Project

#### 2B.1 Confirm `exposes` in `module-federation.config.ts`

**Rspack / Webpack (`@module-federation/enhanced`):**

```ts
import { createModuleFederationConfig } from '@module-federation/enhanced/rspack';

export default createModuleFederationConfig({
  name: '@my-org/ops-tools',   // ← remote name, needed later
  exposes: {
    './IncidentForm': './src/components/IncidentForm',
    './UserLookup': './src/components/UserLookup',
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});
```

**Modern.js (`@module-federation/modern-js-v3`):**

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

If `exposes` is empty, add the components you want to expose.

#### 2B.2 Determine your `baseUrl`

- **Local dev**: `http://localhost:{devPort}/mf-manifest.json`
- **Production CDN**: `https://cdn.example.com/ops-tools/2.4.1/mf-manifest.json`

> **Important**: For standard MF, `baseUrl` must end with `/mf-manifest.json`.

Note the `name` value from `module-federation.config.ts` and proceed to [Step 3](#step-3-generate-mcp_appsjson).

---

### Step 2C: No MF Yet

Choose your build tool:

**Rspack / Webpack:**

```bash
pnpm add @module-federation/enhanced
```

Create `module-federation.config.ts`:

```ts
import { createModuleFederationConfig } from '@module-federation/enhanced/rspack';

export default createModuleFederationConfig({
  name: '@my-org/app-name',   // ← fill in your package name
  exposes: {
    './MyComponent': './src/components/MyComponent',  // ← fill in your component
  },
  shared: { react: { singleton: true }, 'react-dom': { singleton: true } },
});
```

**Modern.js:**

```bash
pnpm add @module-federation/modern-js-v3
```

Register the plugin in `modern.config.ts`:

```ts
import { appTools, defineConfig } from '@modern-js/app-tools';
import { moduleFederationPlugin } from '@module-federation/modern-js-v3';

export default defineConfig({
  plugins: [appTools(), moduleFederationPlugin()],
});
```

Then create `module-federation.config.ts` in the same format as above (change the import to `@module-federation/modern-js-v3`).

**Vite:** See the [module-federation.io Vite quick-start guide](https://module-federation.io/guide/start/quick-start.html).

Once configured, re-run [Step 1](#step-1-detect-your-project-type) to confirm detection, then proceed to [Step 3](#step-3-generate-mcp_appsjson).

---

## Step 3: Generate `mcp_apps.json`

`mcp_apps.json` is the single configuration entry point for the whole solution — it tells the Server which remotes exist and which component each tool maps to.

### 3.1 Copy the schema file

```bash
cp node_modules/@module-federation/mcp-apps/mcp_apps.schema.json ./mcp_apps.schema.json
```

If the package isn't installed yet, download directly from the repo:

```bash
curl -o mcp_apps.schema.json \
  https://raw.githubusercontent.com/module-federation/mcp-apps/main/mcp_apps.schema.json
```

### 3.2 Create the config file

**Vmok project** (`baseUrl` ends with `/vmok-manifest.json`):

```json
{
  "$schema": "./mcp_apps.schema.json",
  "remotes": [
    {
      "name": "@demo/ops-tools",
      "version": "local",
      "baseUrl": "http://localhost:8080/vmok-manifest.json",
      "locale": "en",
      "csp": {
        "connectDomains": ["http://localhost:8080"],
        "resourceDomains": ["http://localhost:8080"]
      }
    }
  ],
  "tools": [
    {
      "name": "incident_form",
      "title": "Incident Form",
      "description": "Create and submit a production incident report including severity, impact scope, and resolution notes",
      "inputSchema": { "type": "object", "properties": {} },
      "remote": "@demo/ops-tools",
      "module": "./IncidentForm"
    }
  ]
}
```

**Standard MF project** (`baseUrl` ends with `/mf-manifest.json`):

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
      "title": "Incident Form",
      "description": "Create and submit a production incident report including severity, impact scope, and resolution notes",
      "inputSchema": { "type": "object", "properties": {} },
      "remote": "@my-org/ops-tools",
      "module": "./IncidentForm"
    },
    {
      "name": "user_lookup",
      "title": "User Lookup",
      "description": "Look up a user by ID or email and return their profile and permission status",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "User ID or email address" }
        },
        "required": ["query"]
      },
      "remote": "@my-org/ops-tools",
      "module": "./UserLookup"
    }
  ]
}
```

### Naming conventions

- `tools[].name`: Drop the `./` prefix and convert PascalCase to snake_case — e.g. `./IncidentForm` → `incident_form`
- `tools[].description`: Written for the AI model — describe what the tool does and when to call it. The more specific, the more reliably the AI will pick the right tool
- `csp` domains must include the full origin (scheme + host + port) of `baseUrl`

### Validate the JSON

```bash
node -e "JSON.parse(require('fs').readFileSync('mcp_apps.json', 'utf8')); console.log('✅ valid JSON')"
```

> For the full field reference, see the [mcp_apps.json Complete Guide](./mcp_apps_json.md).

---

## Step 4: Start the MCP Server and Register with Your AI Host

### 4.1 Start the frontend dev server (if not already running)

```bash
pnpm run dev
```

Confirm the manifest is reachable:

```bash
# Vmok
curl http://localhost:8080/vmok-manifest.json

# Standard MF
curl http://localhost:8080/mf-manifest.json
```

### 4.2 Register with Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "my-mcp-app": {
      "command": "npx",
      "args": [
        "-y",
        "@module-federation/mcp-apps@latest",
        "--config",
        "/absolute/path/to/mcp_apps.json",
        "--stdio"
      ]
    }
  }
}
```

> **nvm users**: Claude Desktop does not inherit your shell's PATH, so `npx` may resolve to the wrong Node version. Use `which npx` to get the full path:
>
> ```json
> {
>   "mcpServers": {
>     "my-mcp-app": {
>       "command": "/Users/you/.nvm/versions/node/v22.x.x/bin/npx",
>       "args": ["-y", "@module-federation/mcp-apps@latest", "--config", "/absolute/path/to/mcp_apps.json", "--stdio"],
>       "env": {
>         "PATH": "/Users/you/.nvm/versions/node/v22.x.x/bin:/usr/local/bin:/usr/bin:/bin"
>       }
>     }
>   }
> }
> ```

Other MCP-compatible clients (VS Code GitHub Copilot, Goose, Cursor, etc.) use the same `command` and `args` format.

### 4.3 Restart and verify

1. Fully quit and relaunch Claude Desktop
2. Start a new conversation and ask: "What MCP tools do you have?"
3. Confirm the tool names declared in `mcp_apps.json` appear in the response
4. Ask the AI to invoke one of the tools and see the component render inside the chat window

---

## Step 5: Let Components Communicate with the AI (Optional)

If the component only needs to render — no interaction with the AI required — skip this step.

If you want user actions to **advance the conversation** — for example, telling the AI what to do next after a form submission — use the `mcpApp` prop.

`mcpApp` is **automatically injected as a prop** by the framework at render time. No package to install, no import needed. Just declare the interface in your component's props and use it:

```tsx
interface McpApp {
  /** Call another MCP tool and use the result inside this component */
  callServerTool: (params: {
    name: string;
    arguments?: Record<string, unknown>;
  }) => Promise<{
    content: Array<{ type: string; text?: string }>;
    isError?: boolean;
  }>;

  /** Inject a message into the AI conversation to trigger the next agent turn */
  sendMessage?: (params: {
    role: string;
    content: Array<{ type: string; text: string }>;
  }) => Promise<{ isError?: boolean }>;
}

export default function IncidentForm({ mcpApp }: { mcpApp?: McpApp }) {
  const handleSubmit = async (incident) => {
    // Your existing business logic
    await fetch('/api/incidents', { method: 'POST', body: JSON.stringify(incident) });

    // Tell the AI what happened so the conversation can continue
    await mcpApp?.sendMessage({
      role: 'user',
      content: [{ type: 'text', text: `Incident #${incident.id} created, severity: ${incident.severity}` }],
    });
  };

  return <IncidentFormUI onSubmit={handleSubmit} />;
}
```

When the component runs outside an MCP host (e.g. your regular app), `mcpApp` is `undefined` and the optional chain `?.` silently skips the call — **zero impact on existing behavior**.

### `callServerTool` vs `sendMessage`

The two methods serve different purposes:

- **`callServerTool`**: Call another MCP tool from within the component and use the return value directly (e.g. populating a dropdown, checking status)
- **`sendMessage`**: Inject a message into the conversation context so the AI reads it and automatically triggers the next step (e.g. notifying the AI that a form was submitted, advancing a multi-step wizard to the next tool)

### Multi-step wizard pattern

Chain multiple MCP tools together as a wizard: the component in Step N calls `sendMessage` after the user confirms, instructing the AI to call Step N+1 and passing the current step's data as arguments. For a detailed implementation pattern, see [develop-mcp-app-component.md](../../skill/develop-mcp-app-component.md).

---

## Testing Your MCP App

### Testing with Claude

Both [Claude.ai](https://claude.ai/) (web) and [Claude Desktop](https://claude.ai/download) support MCP Apps. For local development with the web version, you need to expose your local server to the internet:

```bash
npx cloudflared tunnel --url http://localhost:3001
```

Copy the generated URL and add it in Claude via **Profile → Settings → Connectors → Add custom connector**.

For Claude Desktop, configure as shown in [Step 4](#step-4-start-the-mcp-server-and-register-with-your-ai-host) and test directly.

### Verification checklist

- [ ] Frontend dev server is running and the manifest URL is reachable (`curl` test passes)
- [ ] `mcp_apps.json` is valid JSON with the correct `baseUrl` suffix (Vmok: `/vmok-manifest.json`; standard MF: `/mf-manifest.json`)
- [ ] The path in the Claude Desktop config is an absolute path
- [ ] After restart, the AI lists the expected tool names
- [ ] Invoking a tool renders the component inside the conversation
- [ ] (If implemented) User actions correctly advance the conversation

---

## Troubleshooting

**Tools don't appear after restart**

- Confirm the path in the Claude Desktop config is absolute, not relative
- Confirm `npx` resolves to the correct Node version (`node --version`)
- Run the command manually to check for startup errors:
  ```bash
  npx -y @module-federation/mcp-apps@latest --config /path/to/mcp_apps.json --stdio
  ```

**Component loads but shows a blank screen / CSP error**

- Open DevTools in the iframe and check the Console for blocked URLs
- Add any missing origins to `csp.connectDomains` and `csp.resourceDomains` in `mcp_apps.json`
- After editing `mcp_apps.json`, restart Claude Desktop (the Server re-reads config on startup)

**`remoteEntry` / manifest returns 404**

- Vmok: confirm the `baseUrl` path is `/vmok-manifest.json`, not `/remoteEntry.js`
- Standard MF: confirm the path is `/mf-manifest.json`, not `/remoteEntry.js`
- Check that `baseUrl` has no extra `/` immediately before the manifest filename

**nvm / node not found in Claude Desktop**

Use `which npx` to get the absolute path and pin the Node path via `env.PATH` in your Claude Desktop config (see the nvm example in [Step 4.2](#42-register-with-claude-desktop)).

---

## Deploying to Production

Once local testing passes, deploy your components to a CDN, then update `baseUrl` and `version` in `mcp_apps.json` and restart the Server to switch to the production build:

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

The Server itself **does not need to be rebuilt** — `mcp_apps.json` is the only thing that changes.

---

## Further Reading

- [mcp_apps.json Complete Guide](./mcp_apps_json.md) — semantics, constraints, and CSP rules for every field
- [Component Development Guide](./02-component-guide.md) — full `mcpApp` prop API, multi-step wizard pattern, dual-channel communication
- [HTTP Mode / Custom Agent Integration](./05-custom-agent-integration.md) — running the Server in HTTP mode for self-hosted Agent frameworks
- [MCP Apps Specification](https://modelcontextprotocol.io/docs/extensions/apps)
- [Module Federation Quick Start](https://module-federation.io/guide/start/quick-start.html)
