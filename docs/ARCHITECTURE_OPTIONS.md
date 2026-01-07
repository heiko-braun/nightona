# Nightona: Architecture Overview & Extension Options

## Current State

Nightona is a web application that provides a browser-based interface for AI-assisted development using Claude Code within Daytona sandboxes.

### Architecture
- **Frontend**: React 19 + TypeScript + Vite with shadcn/ui components
- **Backend**: Cloudflare Worker with Durable Objects for state persistence
- **Sandbox**: Daytona SDK for persistent development environments
- **AI Integration**: Claude Code CLI executed via shell commands

### Current Limitations
- **No streaming**: Uses `sandbox.process.executeCommand()` which blocks until completion
- **Batch responses**: Users wait for full Claude Code output before seeing results
- **Limited interactivity**: Cannot interrupt or provide input during execution

---

## Extension Options

### Option 1: Daytona PTY Streaming

Use Daytona's pseudo-terminal API for real-time output streaming.

```typescript
const ptyHandle = await sandbox.process.createPty({
  id: 'claude-session',
  onData: (data) => {
    // Stream to browser via SSE/WebSocket
  },
});
await ptyHandle.sendInput('claude -p "prompt" --continue\n');
```

**Requires**: Cloudflare Durable Objects with WebSocket support for browser transport.

**Reference**: [Daytona PTY Documentation](https://www.daytona.io/docs/en/pty/)

### Option 2: Claude Agent SDK Integration

Replace CLI invocation with programmatic SDK usage for structured message streaming.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({ prompt })) {
  // Stream SDKMessage events to browser
}
```

**Benefits**: Granular message types (tool calls, results, text), better error handling.

**Reference**: [Claude Agent SDK - TypeScript](https://github.com/anthropics/claude-agent-sdk-typescript)

### Option 3: Hybrid Approach (Recommended)

Run the Agent SDK inside the Daytona sandbox with an HTTP streaming relay server, enabling structured message streaming directly to the browser via Server-Sent Events (SSE).

#### Why HTTP Streaming over WebSockets?

| Aspect | WebSocket | HTTP Streaming (SSE) |
|--------|-----------|---------------------|
| **Auth headers** | ❌ Not supported in browser | ✅ Standard HTTP headers |
| **Proxy compatibility** | ⚠️ Can be blocked | ✅ Works everywhere |
| **Firewall friendly** | ⚠️ Separate protocol | ✅ Standard HTTPS |
| **Complexity** | Higher (connection mgmt) | Lower (stateless) |

For Claude Code streaming, we need **server→client** (agent activity) with occasional **client→server** (prompts). This fits HTTP streaming perfectly: POST for prompts, SSE for responses.

#### Why Hybrid?

| Component | Can Run SDK? | Reason |
|-----------|--------------|--------|
| Cloudflare Worker | ❌ | No filesystem, no persistent process |
| Browser | ❌ | No filesystem access, security constraints |
| Daytona Sandbox | ✅ | Full Node.js environment with filesystem |

The Claude Agent SDK requires filesystem access to read/write code, so it **must** run inside the sandbox.

#### Architecture

Daytona sandboxes can expose **multiple ports** (range 3000-9999) with public preview URLs. This enables serving both the app preview and HTTP streaming relay from the same sandbox:

```
                                      Daytona Sandbox
                                 ┌─────────────────────────────────────┐
                                 │                                     │
┌─────────────┐  POST /prompt    │  ┌─────────────────┐    AsyncIter   │   ┌─────────────┐
│   Browser   │─────────────────────│ Streaming Relay │◄──────────────────►│ Claude SDK  │
│   (React)   │  SSE response    │  │ (Express.js)    │                │   │  query()    │
│             │◄────────────────────│  :8080          │                │   └─────────────┘
│  ┌───────┐  │   HTTP (iframe)  │  └─────────────────┘                │
│  │Preview│◄───────────────────────┌─────────────────┐                │
│  │ iframe│  │   :3000          │  │  Vite Dev Server │                │
│  └───────┘  │                  │  │  (user's app)   │                │
└─────────────┘                  │  └─────────────────┘                │
                                 └─────────────────────────────────────┘

Preview URLs:
  - https://3000-{sandbox-id}.{runner}.daytona.work  → Vite dev server
  - https://8080-{sandbox-id}.{runner}.daytona.work  → HTTP streaming relay
```

#### Implementation

**1. HTTP streaming relay server inside sandbox:**

```typescript
// /tmp/project/relay-server.ts
import { query } from "@anthropic-ai/claude-agent-sdk";
import express from "express";
import cors from "cors";

const app = express();
app.use(cors());
app.use(express.json());

// Auth middleware - standard HTTP headers work!
app.use((req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!isValidToken(token)) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
});

// POST prompt, receive SSE stream response
app.post('/prompt', async (req, res) => {
  const { prompt } = req.body;

  // Set SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Stream SDK messages as SSE events
  for await (const message of query({ prompt, options: { cwd: "/tmp/project" } })) {
    res.write(`data: ${JSON.stringify(message)}\n\n`);
  }

  res.write('data: [DONE]\n\n');
  res.end();
});

app.listen(8080);
```

**2. Browser streams via fetch API:**

```typescript
async function streamPrompt(prompt: string, onMessage: (msg: SDKMessage) => void) {
  const response = await fetch(`${relayUrl}/prompt`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${sessionToken}`,  // Standard HTTP auth works!
    },
    body: JSON.stringify({ prompt }),
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const chunk = decoder.decode(value);
    for (const line of chunk.split('\n')) {
      if (line.startsWith('data: ') && line !== 'data: [DONE]') {
        onMessage(JSON.parse(line.slice(6)));
      }
    }
  }
}
```

**3. Cloudflare Worker initialization (`/api/initialize`):**

```typescript
// Start both services with PM2
await sandbox.process.executeCommand('pm2 start ecosystem.config.cjs');  // vite on :3000
await sandbox.process.executeCommand('pm2 start relay-server.js');       // relay on :8080

// Get public URLs for both ports
const devServerUrl = await sandbox.getPreviewLink(3000);
const relayUrl = await sandbox.getPreviewLink(8080);

return Response.json({
  success: true,
  devServerUrl: devServerUrl.url,   // For iframe preview
  relayUrl: relayUrl.url            // For HTTP streaming
});
```

**4. Browser connects to both services:**

```typescript
// Display live preview in iframe
<iframe src={devServerUrl} />

// Stream Claude activity via HTTP
streamPrompt("Add dark mode", (message) => {
  // message.type: 'assistant' | 'user' | 'result' | 'system'
  renderMessage(message);
});
```

**Key benefits**:
- Browser connects directly to sandbox—no Cloudflare Worker in the streaming path
- Standard HTTP auth headers work natively (no WebSocket workarounds)
- Better proxy/firewall compatibility

#### Comparison

| Approach | Output Format | Streaming | Structured Events |
|----------|---------------|-----------|-------------------|
| Current (CLI + executeCommand) | Raw JSON | ❌ | ❌ |
| Option 1 (PTY) | Raw terminal text | ✅ | ❌ |
| **Option 3 (Hybrid)** | SDKMessage objects | ✅ | ✅ |

#### Trade-offs

| Pros | Cons |
|------|------|
| Rich, typed message stream | More complex setup |
| Direct browser↔sandbox connection | Relay server lifecycle management |
| Lower latency (no Worker in path) | Sandbox URL exposed to browser |
| Interactive tool permissions | Requires PM2 to keep relay alive |

**Reference**: [Claude Agent SDK - Streaming Mode](https://platform.claude.com/docs/en/agent-sdk/streaming-vs-single-mode)

---

## Authentication Options

Securing preview and relay server access requires authentication at multiple layers.

### Option A: Daytona Private Sandboxes (Built-in)

Use `public: false` and pass tokens with requests:

```typescript
// Create private sandbox
const sandbox = await daytona.create({
  snapshot: CLAUDE_SNAPSHOT_NAME,
  public: false,  // Requires token for access
});

// SDK returns URL + token
const preview = await sandbox.getPreviewLink(3000);
// { url: "https://3000-...", token: "vg5c0ylmcimr8b..." }
```

**Access requires header**: `x-daytona-preview-token: <token>`

**Note**: With HTTP streaming (recommended), this header works natively in fetch requests.

### Option B: Application-Level Auth (Relay Server)

Add token validation middleware in the Express relay server:

```typescript
// relay-server.ts - HTTP streaming with standard auth
app.use((req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  // Or use Daytona token
  const daytonaToken = req.headers['x-daytona-preview-token'];

  if (!isValidToken(token || daytonaToken)) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
});
```

**Flow**: Worker issues session tokens → Browser sends token in HTTP headers → Relay validates

**Advantage**: HTTP streaming supports all standard auth mechanisms (Bearer tokens, cookies, custom headers) without workarounds.

### Option C: Customer Proxy (Recommended for Production)

Daytona's **Customer Proxy** provides enterprise-grade authentication by routing all preview traffic through your own infrastructure:

```
┌─────────┐      HTTPS       ┌──────────────────┐      Internal      ┌─────────────────┐
│ Browser │ ───────────────► │  Customer Proxy  │ ─────────────────► │ Daytona Sandbox │
│         │                  │  (your domain)   │                    │                 │
└─────────┘                  └──────────────────┘                    └─────────────────┘
                                     │
                              ┌──────┴──────┐
                              │ Your Auth   │
                              │ (SSO/OAuth) │
                              └─────────────┘
```

**Benefits**:
- Custom domain (e.g., `preview.yourcompany.com`)
- Integrate with existing auth (SSO, OAuth, SAML)
- No Daytona URLs exposed to end users
- Custom error pages and branding
- Full control over access policies

**Reference**: [Customer Proxy Documentation](https://www.daytona.io/dotfiles/sandbox-preview-urls-and-auth-customer-proxy)

### Comparison

| Approach | Complexity | Auth Integration | Custom Domain | Production Ready |
|----------|------------|------------------|---------------|------------------|
| Public sandbox | None | ❌ | ❌ | ❌ |
| Private + tokens | Low | Limited | ❌ | ⚠️ |
| App-level auth | Medium | Custom | ❌ | ⚠️ |
| **Customer Proxy** | Higher | Full (SSO/OAuth) | ✅ | ✅ |

**Recommendation**: Use **Customer Proxy** for production deployments requiring user authentication, custom domains, and enterprise security compliance.

---

## Open Source Implementations

| Project | Streaming Method | Key Feature |
|---------|------------------|-------------|
| [claudecodeui](https://github.com/siteboon/claudecodeui) | WebSocket | Express.js + React, multi-CLI support |
| [cui](https://github.com/wbopan/cui) | SDK-powered | Background tasks, push notifications |
| [claude-code-webui](https://github.com/sugyan/claude-code-webui) | Real-time | Deno/Node backend, mobile-responsive |
| [claude-agent-sdk-container](https://github.com/receipting/claude-agent-sdk-container) | Character-by-character | REST API, GitHub OAuth |

**Most relevant for Nightona**: `claude-agent-sdk-container` uses HTTP streaming with REST API, which aligns with the recommended HTTP streaming approach.

---

## References

### Daytona
- [TypeScript SDK](https://www.daytona.io/docs/en/typescript-sdk/)
- [Process & Code Execution](https://www.daytona.io/docs/en/process-code-execution/)
- [PTY (Pseudo Terminal)](https://www.daytona.io/docs/en/pty/)
- [Preview & Authentication](https://www.daytona.io/docs/en/preview-and-authentication/) - Multi-port exposure
- [Customer Proxy](https://www.daytona.io/dotfiles/sandbox-preview-urls-and-auth-customer-proxy) - Enterprise auth & custom domains

### Claude Code / Agent SDK
- [Agent SDK Overview](https://docs.anthropic.com/en/docs/claude-code/sdk)
- [TypeScript SDK GitHub](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Streaming vs Single Mode](https://platform.claude.com/docs/en/agent-sdk/streaming-vs-single-mode)

### Cloudflare
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [WebSockets with Durable Objects](https://developers.cloudflare.com/durable-objects/examples/websocket-hibernation/)

---

## Recommended Next Steps

1. Create HTTP streaming relay server using Claude Agent SDK with Express.js
2. Update Docker snapshot to include `@anthropic-ai/claude-agent-sdk`, `express`, and `cors` packages
3. Modify `/api/initialize` to start relay server and return both URLs
4. Update React UI with fetch-based streaming to render SSE messages incrementally
5. Add PM2 ecosystem config for relay server lifecycle management
6. Implement auth middleware in relay server (Bearer token or Daytona token validation)
