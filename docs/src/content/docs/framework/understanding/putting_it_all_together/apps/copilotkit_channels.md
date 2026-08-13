---
title: CopilotKit LlamaIndex Channels SDK
---

The same LlamaIndex agent that powers your in-app copilot can also run as a bot in messaging platforms. [CopilotKit Channels](https://docs.copilotkit.ai/slack/llamaindex) connects your LlamaIndex agent to Slack and other messaging platforms, with threads, tool calls, and rich interactive messages handled natively in the channel.

> **Note:** Full platform setup lives in the [CopilotKit Channels documentation](https://docs.copilotkit.ai/slack/llamaindex). This page shows how Channels fit together with a LlamaIndex agent.

## How it fits together

Your LlamaIndex server stays where it is, exposed over [AG-UI](https://docs.ag-ui.com/). A channel is a separate long-running Node service built with `@copilotkit/channels`, connected through CopilotKit Intelligence — a required surface for Channels, by design (a free tier is available). Intelligence holds the platform connection and credentials, receives each platform event, and delivers the turn to your channel process over a persistent gateway connection; the `CopilotRuntime` starts and owns each channel's lifecycle.

```
Slack  ──►  CopilotKit Intelligence  ──►  channel process (Node)  ──►  LlamaIndex server (AG-UI)
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

Install the Channels SDK alongside the CopilotKit runtime and the AG-UI client for LlamaIndex:

```bash
npm install @copilotkit/channels @copilotkit/runtime @ag-ui/llamaindex
```

Build the agent as a per-thread factory so each conversation gets its own session. The router you mounted above speaks AG-UI over HTTP, so use `LlamaIndexAgent` from `@ag-ui/llamaindex`:

```typescript
// channel.ts
import { createChannel } from "@copilotkit/channels";
import { CopilotRuntime, CopilotKitIntelligence } from "@copilotkit/runtime/v2";
import { createCopilotNodeListener } from "@copilotkit/runtime/v2/node";
import { LlamaIndexAgent } from "@ag-ui/llamaindex";

const required = (name: string) => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const channel = createChannel({
  // The ID of the Channel you created in Intelligence; Slack credentials live
  // there, never in this process.
  name: required("INTELLIGENCE_CHANNEL_ID"),
  identifyUser: "platform",
  // A fresh agent per conversation, keyed by thread.
  agent: (threadId: string) => {
    const agent = new LlamaIndexAgent({
      url: "http://localhost:9000/agentic_chat/run",
    });
    agent.threadId = threadId;
    return agent;
  },
});

// A mention subscribes the thread and runs the agent; afterwards every message
// in a subscribed thread runs it without needing another mention.
channel.onMention(async ({ thread }) => {
  await thread.subscribe();
  await thread.runAgent();
});
channel.onMessage(async ({ thread }) => {
  if (await thread.isSubscribed()) await thread.runAgent();
});

// The runtime owns the channel's lifecycle — there is no `channel.start()`.
const runtime = new CopilotRuntime({
  agents: {}, // the channel supplies its own agent; no web-facing agents needed
  intelligence: new CopilotKitIntelligence({
    apiKey: required("INTELLIGENCE_API_KEY"), // free tier available
  }),
  channels: [channel],
});

// Creating the listener starts the channel's connection.
const listener = createCopilotNodeListener({ runtime });
await listener.channels?.ready({ timeoutMs: 15_000 }); // fail startup loudly on a broken config
```

The agent receives ordinary AG-UI input and emits ordinary AG-UI events; the platform mechanics stay behind the channel, so the same LlamaIndex agent runs unchanged across every platform. Rich messages are written as JSX and rendered to each platform's native format (Block Kit on Slack, for example), so an interactive card degrades gracefully where a platform has no equivalent.

## Run it

Set `INTELLIGENCE_API_KEY` and `INTELLIGENCE_CHANNEL_ID` (both from the CopilotKit Intelligence dashboard — a free tier is available; see the [CopilotKit Channels documentation](https://docs.copilotkit.ai/slack/llamaindex)), then start the channel process alongside your agent server:

```bash
python server.py     # terminal 1 — LlamaIndex agent server
npx tsx channel.ts   # terminal 2 — channel process
```

Mention the bot in Slack and it runs your LlamaIndex agent, streaming the reply back into the thread; the thread stays subscribed, so follow-up messages run without another mention.

## Slack

The Slack connection is configured in CopilotKit Intelligence, which walks you through creating the Slack app and holds its credentials. That leaves two environment variables for the channel process, both from the Intelligence dashboard:

- **`INTELLIGENCE_API_KEY`**: authenticates the runtime with Intelligence (free tier available).
- **`INTELLIGENCE_CHANNEL_ID`**: the ID of the Channel you created in Intelligence, matched by `createChannel({ name })`.

The channel process holds a persistent connection to the Intelligence gateway, which delivers each turn to it. It needs a long-running host — a serverless request handler cannot own that connection.

For the full walkthrough, see the [CopilotKit Slack guide for LlamaIndex](https://docs.copilotkit.ai/slack/llamaindex).

## Other platforms

Other messaging platforms connect the same way — a managed connection configured in Intelligence, with your channel code unchanged. See the [CopilotKit Channels documentation](https://docs.copilotkit.ai/slack/llamaindex) for the current platform list and per-platform setup.

## Message pipeline and architecture

Channels keeps a deliberate credential split, useful when your LlamaIndex agent holds its own model keys and tools:

- **You keep** the agent logic, model credentials, tools, and the channel process.
- **CopilotKit Intelligence holds** the platform credentials, message delivery, registration, health, and reconnects.

A turn flows through five stages: a user messages the app, Intelligence receives the platform event using the credentials configured for that channel, a persistent gateway connection delivers the turn to your running channel process, your LlamaIndex agent runs and renders a reply, and Intelligence sends it back as native platform content. Platform credentials never enter the agent process.

> **Note:** By default, interactive actions live in memory and reset on restart. Back the channel with a durable action and state store (Redis or Postgres) so buttons and per-thread state survive restarts and span multiple instances.
