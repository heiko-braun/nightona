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

### Option 3: Hybrid Approach

Run the Agent SDK inside Daytona sandbox, stream via PTY or custom WebSocket bridge.

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

### Claude Code / Agent SDK
- [Agent SDK Overview](https://docs.anthropic.com/en/docs/claude-code/sdk)
- [TypeScript SDK GitHub](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Streaming vs Single Mode](https://platform.claude.com/docs/en/agent-sdk/streaming-vs-single-mode)

### Cloudflare
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [WebSockets with Durable Objects](https://developers.cloudflare.com/durable-objects/examples/websocket-hibernation/)

---

## Recommended Next Steps

1. Implement WebSocket support via Durable Objects
2. Replace `executeCommand()` with `createPty()` for streaming
3. Add SSE/WebSocket endpoint to stream `onData` events to frontend
4. Update React UI to render incremental output
