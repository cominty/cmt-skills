---
name: cominty-api-onboarding
description: "Take a developer from a Cominty account to a first answer from the Cominty agent API, one step at a time: the API key and user id, which SDK to install (TypeScript, Python or plain HTTP), picking an agent, the first chat call, streaming progress, and continuing a thread. Use when someone says things like: get started with Cominty, set up the Cominty API, how do I authenticate, where does my API key go, which SDK do I install, how do I call my agent, show me a first request, how do I stream the reply, how do I send a follow-up."
---

# Onboarding a developer onto the Cominty API

Take someone from "I have an account" to "my code got an answer back".

This skill teaches. Work in small steps, confirm each one worked, then move
on. Do not write a whole integration at once. To wire an agent into an
existing project, load the `cominty-add-agent-to-app` skill instead.

## Say these four things early

First integrations get the same four things wrong. Say them before anyone
asks. The stream contract is in
[references/streaming.md](references/streaming.md).

| The wrong guess | What is true |
|---|---|
| `Authorization: Bearer <key>` | The header is `x-cominty-token: <key>`. A key sent in `Authorization` gets a 403. |
| The stream is SSE | It is JSON Lines (`application/jsonl`): one JSON value per line. No `event:` or `data:` framing, no `[DONE]`. |
| The `result` event ends the stream | It does not. More events can follow it. The stream ends on the terminal message: the line with no `correlation_id`. |
| Text arrives token by token | There is no text-delta event. Progress events stream. The reply arrives whole. |

## The path, in order

### 1. Ask what they are building in

TypeScript, Python or plain HTTP. Every snippet depends on the answer.

| Stack | Install | Needs |
|---|---|---|
| TypeScript or JavaScript | `npm install @cominty-ai/sdk` | Node 20+. ESM only. Server side only. |
| Python | `pip install cominty-sdk` | Python 3.9+. Async only. |
| Plain HTTP | Nothing | Any language that can send HTTPS and JSON. |

Recommend an SDK unless they have a reason to write the client by hand.
Both read the stream correctly for you. A hand-written client is where
the four mistakes above happen.

### 2. Two credentials

Both come from the developer console, https://platform.cominty.ai.

| Value | Where | Notes |
|---|---|---|
| API key | `/api-keys` | A key is shown once. If it is lost, they create a new one. |
| User id | `/quickstart` | Starts with `user_`. The page shows it with a copy button. It is also behind your avatar in the console. |

The user id names the end user every request acts on behalf of. The SDKs
take it once, on the client, and apply it to every call.

```bash
export COMINTY_API_KEY="<their key>"
export COMINTY_USER_ID="user_..."
```

- Never ask them to paste the key into the conversation. Never repeat a
  key back. Never write one into a file or a snippet. The snippets here
  read `COMINTY_API_KEY` from the environment, so they stay safe to share.
- The SDKs do not read `.env` files. The developer exports the variables,
  or loads the file with whatever the project already uses.
- The TypeScript SDK refuses to construct in a browser, because the key
  would be readable in devtools. If they want this in a front end, the
  answer is a route on their own server that holds the key.

### 3. Pick an agent

Every chat call takes an agent id.

- The console's Quickstart page (`/quickstart`) shows the first call with
  their own agent id and user id already written into the code.
- The built-in agents work before they have made their own:

| Agent id | Use for |
|---|---|
| `__cominty_agents::agent.chat` | General purpose. Start here. |
| `__cominty_agents::agent.hive` | Multi-agent orchestration for complex tasks. |
| `__cominty_agents::agent.planner` | Plans and structures multi-step work. |

### 4. The first call

Give them one snippet, in their stack, and have them run it. Plain HTTP
takes two requests: see [references/quickstart.md](references/quickstart.md).

```ts
import { Cominty } from '@cominty-ai/sdk'

const client = new Cominty() // reads COMINTY_API_KEY and COMINTY_USER_ID
const run = await client.chat.start({
    agentId: '__cominty_agents::agent.chat',
    message: 'Hello! What can you help me with?',
})
console.log(await run.text())
```

```python
import asyncio
from cominty_sdk import AsyncCominty

async def main() -> None:
    async with AsyncCominty() as client:  # reads the same two variables
        run = await client.chat.start(
            agent_id="__cominty_agents::agent.chat",
            message="Hello! What can you help me with?",
        )
        print(await run.text())

asyncio.run(main())
```

`run.text()` waits for the agent to finish. That is the whole happy path.
The TypeScript snippet needs an ES module: saved as `first-call.mjs`,
`node first-call.mjs` runs it. Do not bring up streaming until this works.

### 5. Streaming, only if they need it

Streaming shows progress: tool calls, LLM steps, notes from the agent. It
is worth it for a screen that shows what the agent is doing. It adds
nothing if they only want the answer.

```ts
import { isKnownEvent } from '@cominty-ai/sdk'

const research = await client.chat.start({
    agentId: '__cominty_agents::agent.chat',
    message: 'Research the history of the espresso machine in three lines.',
})

for await (const event of research) {
    if (!isKnownEvent(event)) continue // an event a newer server added
    if (event.name === 'tool_call') console.log('tool', event.data.name, event.status)
}
console.log(await research.text()) // already received: no second request
```

Iterate first, then ask for the text. A run is single-use: once `text()`
has drained it, a second loop throws in TypeScript and yields nothing in
Python. Python is in [references/quickstart.md](references/quickstart.md).
Someone writing the reader by hand reads
[references/streaming.md](references/streaming.md) first, not after.

### 6. Continue the conversation

A follow-up needs only the thread id. The agent keeps the thread's context.

```ts
const reply = await client.chat.send(run.thread.id, {
    agentId: '__cominty_agents::agent.chat',
    message: 'Can you go into more detail?',
})
console.log(await reply.text())
```

## Raise these before they hit them

- **The agent may ask questions instead of answering.** They arrive on the
  final message as `questions`, each with a `prompt` and `options`. The
  answer is the next message in the same thread: an option, or free text.
- **A failed run does not throw.** The request worked; the agent did not.
  Check `status` on the final message.
- **Costs are decimal strings.** Floats lose precision once they are
  summed. Use a decimal type. The Python SDK already returns `Decimal`.
- **A 429 is more than one problem.** `concurrency` is transient: retry
  when a run in flight finishes. `user` and `organization` mean a quota is
  used up: wait for the reset, or an admin raises the limit.
- **Scoping.** `sourceIds` and `documentIds` restrict retrieval.
  `disabledTools` turns tools off: `'web'`, `'company_documents'`,
  `'mcp:<server>'`, `'mcp:*'`. Leave a field out to mean "everything the
  agent has". An empty `source_ids` means "no source", not "all of them".
- **A long task can stop and ask to continue.** The tool rounds for one
  message are capped: at 60 by default today. At the cap the agent recaps
  and asks whether to go on, and the message still ends as `success`.
  Send a follow-up to continue. `maxSteps` sets the cap, per message.
- **No memory across threads unless you ask.** Start the thread with
  `memoryNamespace`, a name you choose, or it has no memory beyond itself
  and nothing says so. The name is shared by the whole organization. Both
  options are in [references/quickstart.md](references/quickstart.md).

## Reference

| File | For |
|---|---|
| [references/quickstart.md](references/quickstart.md) | First call, streaming, follow-up, `max_steps` and memory in all three stacks |
| [references/streaming.md](references/streaming.md) | The JSON Lines contract. Required reading for a hand-written client |
| [references/endpoints.md](references/endpoints.md) | The HTTP surface: threads, messages, memory files, agents, files |
| [references/troubleshooting.md](references/troubleshooting.md) | Symptom to cause, for the failures people hit first |

Related skills, when they are installed: `cominty-add-agent-to-app`,
`cominty-streaming`, `cominty-troubleshooting`, `cominty-console`.

Docs: https://docs.cominty.ai and https://docs.cominty.ai/api-reference.
For an agent: https://docs.cominty.ai/llms-full.txt, or the docs' MCP
server at https://docs.cominty.ai/mcp.

## How to behave

- One step at a time. They need the first call to work before anything
  else means anything. Ask before assuming their stack.
- Never invent an endpoint, a field or an error code. If it is not in this
  skill or the API reference, say you are not sure and point at the
  reference.
- Never handle their API key.
