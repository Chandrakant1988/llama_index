---
title: CopilotKit LlamaIndex Channels SDK
---

The same LlamaIndex agent that powers your in-app copilot can also run as a bot in messaging platforms. [CopilotKit Channels](https://docs.copilotkit.ai/slack/llamaindex) connects your LlamaIndex agent to Slack and other messaging platforms, with threads, tool calls, and rich interactive messages handled natively in the channel.

> **Note:** Full platform setup lives in the [CopilotKit Channels documentation](https://docs.copilotkit.ai/slack/llamaindex). This page shows how Channels fit together with a LlamaIndex agent.

## How it fits together

Your LlamaIndex server stays where it is, exposed over [AG-UI](https://docs.ag-ui.com/). A channel is a separate long-running Node service built with `@copilotkit/channels`: you point it at your agent and attach one or more platform adapters. `createChannel` takes an array of adapters, so a single process can serve several platforms at once. Channels run through a channel runner — the `CopilotRuntime` starts and owns each channel's lifecycle once CopilotKit Intelligence is configured (a free tier is available). The SDK and its adapters are MIT-licensed, and as an alternative to the managed runner you can build and operate your own channel runner on the SDK's lower-level primitives, with no Intelligence dependency — your team then owns lifecycle, state, persistence, concurrency, and retries.

```
Slack / Teams / Discord  ──►  channel process (Node)  ──►  LlamaIndex server (AG-UI)
```

Your agent server can keep serving a web frontend at the same time — the web app and the channel are just two clients of one AG-UI endpoint.

## Expose your agent over AG-UI

If your agent is not yet served over AG-UI, install the AG-UI protocol integration:

```bash
pip install llama-index-protocols-ag-ui llama-index-llms-openai fastapi uvicorn
```

`get_ag_ui_workflow_router` wraps a LlamaIndex agent in a FastAPI router that speaks AG-UI:

```python
# server.py
import uvicorn
from fastapi import FastAPI

from llama_index.llms.openai import OpenAI
from llama_index.protocols.ag_ui.router import get_ag_ui_workflow_router

agent_router = get_ag_ui_workflow_router(
    llm=OpenAI(model="gpt-4.1"),
    system_prompt="You are a helpful assistant.",
)

app = FastAPI(title="LlamaIndex AG-UI server")
app.include_router(agent_router, prefix="/agentic_chat")

if __name__ == "__main__":
    uvicorn.run(app, host="127.0.0.1", port=9000)
```

The router exposes the agent at `POST /agentic_chat/run` — that URL is what the channel connects to.

## Point a channel at it

Install the Channels SDK (batteries-included, every adapter) alongside the CopilotKit runtime and the AG-UI client for LlamaIndex:

```bash
npm install @copilotkit/channels @copilotkit/runtime @ag-ui/llamaindex
```

Build the agent as a per-thread factory so each conversation gets its own session. The router you mounted above speaks AG-UI over HTTP, so use `LlamaIndexAgent` from `@ag-ui/llamaindex`:

```typescript
// channel.ts
import { createChannel } from "@copilotkit/channels";
import { slack } from "@copilotkit/channels/slack";
import { CopilotRuntime, CopilotKitIntelligence } from "@copilotkit/runtime/v2";
import { createCopilotNodeListener } from "@copilotkit/runtime/v2/node";
import { LlamaIndexAgent } from "@ag-ui/llamaindex";

const channel = createChannel({
  name: "llamaindex-bot", // project-unique Intelligence Channel name
  identifyUser: "platform",
  // A fresh agent per conversation, keyed by thread.
  agent: (threadId: string) => {
    const agent = new LlamaIndexAgent({
      url: "http://localhost:9000/agentic_chat/run",
    });
    agent.threadId = threadId;
    return agent;
  },
  adapters: [
    slack({
      botToken: process.env.SLACK_BOT_TOKEN!, // xoxb-…
      appToken: process.env.SLACK_APP_TOKEN!, // xapp-… (Socket Mode)
    }),
  ],
});

// Mentions and DMs both start a turn; runAgent streams the reply into the thread.
channel.onMessage(async ({ thread }) => {
  await thread.runAgent();
});

// The runtime owns the channel's lifecycle — there is no `channel.start()`.
const runtime = new CopilotRuntime({
  agents: {}, // the channel supplies its own agent; no web-facing agents needed
  intelligence: new CopilotKitIntelligence({
    apiKey: process.env.COPILOTKIT_INTELLIGENCE_API_KEY!, // free tier available
  }),
  channels: [channel],
});

// Creating the listener starts the channel's connection.
const listener = createCopilotNodeListener({ runtime });
await listener.channels.ready(); // fail startup loudly on a broken config
```

The agent receives ordinary AG-UI input and emits ordinary AG-UI events; the platform mechanics stay behind the adapter, so the same LlamaIndex agent runs unchanged across every channel. Rich messages are written as JSX and rendered to each platform's native format (Block Kit on Slack, for example), so an interactive card degrades gracefully where a platform has no equivalent.

## Run it

Set `COPILOTKIT_INTELLIGENCE_API_KEY` (a CopilotKit Intelligence key — a free tier is available; see the [CopilotKit Channels documentation](https://docs.copilotkit.ai/slack/llamaindex)) along with the Slack tokens below, then start the channel process alongside your agent server:

```bash
python server.py     # terminal 1 — LlamaIndex agent server
npx tsx channel.ts   # terminal 2 — channel process
```

Mention or DM the bot in Slack and it runs your LlamaIndex agent, streaming the reply back into the thread.

## Slack

Slack runs over Socket Mode, which opens an outbound WebSocket to Slack, so no public URL or tunnel is required during development. Set the Slack credentials in your environment:

- **`SLACK_BOT_TOKEN`**: Bot User OAuth token (`xoxb-...`).
- **`SLACK_APP_TOKEN`**: App-level token with the `connections:write` scope (`xapp-...`).

For the full Slack app walkthrough, see the [CopilotKit Slack guide for LlamaIndex](https://docs.copilotkit.ai/slack/llamaindex).

## Other platforms

The Channels SDK is not limited to Slack. Microsoft Teams, Discord, Telegram, and WhatsApp run through the same `@copilotkit/channels` API (`@copilotkit/channels/teams`, `@copilotkit/channels/discord`, and so on): add the matching adapter to the `adapters` array — a single process can serve several platforms at once — and the rest of your agent code stays the same. See the [CopilotKit Channels documentation](https://docs.copilotkit.ai/slack/llamaindex) for the current platform list and per-platform setup.

## Message pipeline and architecture

The example above uses a **direct adapter**: your process holds the Slack tokens, and Socket Mode delivers platform events straight to it, with Intelligence acting as the channel runner (lifecycle, registration, health). Slack can also run as a **managed connection** through CopilotKit Intelligence, which keeps a deliberate credential split, useful when your LlamaIndex agent holds its own model keys and tools:

- **You keep** the agent logic, model credentials, tools, and the channel process.
- **CopilotKit Intelligence holds** the platform credentials, message delivery, registration, health, and reconnects.

In the managed path, a turn flows through five stages: a user messages the app, Intelligence receives the platform event using the credentials configured for that channel, a persistent gateway connection delivers the turn to your running channel process, your LlamaIndex agent runs and renders a reply, and Intelligence sends it back as native platform content. Platform credentials never enter the agent process.

> **Note:** By default, interactive actions live in memory and reset on restart. Back the channel with a durable action and state store (Redis or Postgres) so buttons and per-thread state survive restarts and span multiple instances.
