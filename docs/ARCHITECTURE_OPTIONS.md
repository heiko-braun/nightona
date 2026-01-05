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

Run the Agent SDK inside the Daytona sandbox with a WebSocket relay server, enabling structured message streaming directly to the browser.

#### Why Hybrid?

| Component | Can Run SDK? | Reason |
|-----------|--------------|--------|
| Cloudflare Worker | ❌ | No filesystem, no persistent process |
| Browser | ❌ | No filesystem access, security constraints |
| Daytona Sandbox | ✅ | Full Node.js environment with filesystem |

The Claude Agent SDK requires filesystem access to read/write code, so it **must** run inside the sandbox.

#### Architecture

Daytona sandboxes can expose **multiple ports** (range 3000-9999) with public preview URLs. This enables serving both the app preview and WebSocket relay from the same sandbox:

```
                                      Daytona Sandbox (public: true)
                                 ┌─────────────────────────────────────┐
                                 │                                     │
┌─────────────┐   WebSocket      │  ┌─────────────────┐    AsyncIter   │   ┌─────────────┐
│   Browser   │◄────────────────────│ Streaming Relay │◄──────────────────►│ Claude SDK  │
│   (React)   │   :8080          │  │ (relay-server)  │                │   │  query()    │
│             │                  │  └─────────────────┘                │   └─────────────┘
│  ┌───────┐  │   HTTP (iframe)  │  ┌─────────────────┐                │
│  │Preview│◄───────────────────────│  Vite Dev Server │                │
│  │ iframe│  │   :3000          │  │  (user's app)   │                │
│  └───────┘  │                  │  └─────────────────┘                │
└─────────────┘                  │                                     │
                                 └─────────────────────────────────────┘

Preview URLs:
  - https://3000-{sandbox-id}.{runner}.daytona.work  → Vite dev server
  - https://8080-{sandbox-id}.{runner}.daytona.work  → WebSocket relay
```

#### Implementation

**1. Streaming relay server inside sandbox:**

```typescript
// /tmp/project/relay-server.ts
import { query } from "@anthropic-ai/claude-agent-sdk";
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  ws.on("message", async (data) => {
    const { prompt } = JSON.parse(data);

    for await (const message of query({ prompt, options: { cwd: "/tmp/project" } })) {
      ws.send(JSON.stringify(message));
    }
  });
});
```

**2. Browser connects directly to sandbox:**

```typescript
// Daytona provides public preview URLs
const ws = new WebSocket("wss://sandbox-abc123.daytona.io:8080");

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  // message.type: 'assistant' | 'user' | 'result' | 'system'
  renderIncrementally(message);
};

ws.send(JSON.stringify({ prompt: "Add a dark mode toggle" }));
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
  relayUrl: relayUrl.url            // For WebSocket connection
});
```

**4. Browser connects to both services:**

```typescript
// Display live preview in iframe
<iframe src={devServerUrl} />

// Stream Claude activity via WebSocket
const ws = new WebSocket(relayUrl.replace('https://', 'wss://'));
ws.onmessage = (event) => renderMessage(JSON.parse(event.data));
ws.send(JSON.stringify({ prompt: "Add dark mode" }));
```

**Key benefit**: Browser connects directly to sandbox—no Cloudflare Worker in the streaming path.

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

## Open Source Implementations

| Project | Streaming Method | Key Feature |
|---------|------------------|-------------|
| [claudecodeui](https://github.com/siteboon/claudecodeui) | WebSocket | Express.js + React, multi-CLI support |
| [cui](https://github.com/wbopan/cui) | SDK-powered | Background tasks, push notifications |
| [claude-code-webui](https://github.com/sugyan/claude-code-webui) | Real-time | Deno/Node backend, mobile-responsive |
| [claude-agent-sdk-container](https://github.com/receipting/claude-agent-sdk-container) | Character-by-character | REST API, GitHub OAuth |

**Most relevant for Nightona**: `claudecodeui` uses WebSocket streaming with an architecture adaptable to Cloudflare Workers + Durable Objects.

---

## References

### Daytona
- [TypeScript SDK](https://www.daytona.io/docs/en/typescript-sdk/)
- [Process & Code Execution](https://www.daytona.io/docs/en/process-code-execution/)
- [PTY (Pseudo Terminal)](https://www.daytona.io/docs/en/pty/)
- [Preview & Authentication](https://www.daytona.io/docs/en/preview-and-authentication/) - Multi-port exposure

### Claude Code / Agent SDK
- [Agent SDK Overview](https://docs.anthropic.com/en/docs/claude-code/sdk)
- [TypeScript SDK GitHub](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Streaming vs Single Mode](https://platform.claude.com/docs/en/agent-sdk/streaming-vs-single-mode)

### Cloudflare
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [WebSockets with Durable Objects](https://developers.cloudflare.com/durable-objects/examples/websocket-hibernation/)

---

## Recommended Next Steps

1. Create relay server script using Claude Agent SDK with WebSocket
2. Update Docker snapshot to include `@anthropic-ai/claude-agent-sdk` and `ws` packages
3. Modify `/api/initialize` to start relay server and return both URLs
4. Update React UI to connect WebSocket to relay URL and render streaming messages
5. Add PM2 ecosystem config for relay server lifecycle management
