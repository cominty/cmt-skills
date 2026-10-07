# Quickstart: the first call in three stacks

Every snippet reads credentials from the environment. Never write a key
into code.

```bash
export COMINTY_API_KEY="<from platform.cominty.ai/api-keys>"
export COMINTY_USER_ID="user_..."   # on the console's Quickstart page
```

Contents: [TypeScript](#typescript) · [Python](#python) ·
[Plain HTTP](#plain-http)

---

## TypeScript

```bash
npm install @cominty-ai/sdk   # or: pnpm add, yarn add, bun add
```

Node 20+, or any server-side runtime with global `fetch` and
`ReadableStream` (Bun, Deno, edge workers). No runtime dependencies.

The package is ESM only. From CommonJS, `require('@cominty-ai/sdk')` works
on Node 22.12+. On older versions use `await import('@cominty-ai/sdk')`.

### Start a conversation

```ts
import { Cominty } from '@cominty-ai/sdk'

// Reads COMINTY_API_KEY and COMINTY_USER_ID from the environment.
const client = new Cominty()

const run = await client.chat.start({
    agentId: '__cominty_agents::agent.chat',
    message: 'Hello! What can you help me with?',
})

console.log(await run.text()) // waits for the agent to finish
console.log('thread:', run.thread.id)
```

This is plain JavaScript too: save it as `first-call.mjs` and run
`node first-call.mjs`.

`await run.result()` gives the whole message: status, files, structured
output, questions. `run.text()` is shorthand for its `content`.

Explicit options win over the environment. Use them on a runtime with no
`process.env`, such as some edge workers:

```ts
const client = new Cominty({ apiToken: env.COMINTY_API_KEY, userId: env.COMINTY_USER_ID })
```

A missing or malformed `userId` throws when the client is constructed, not
later as a server error.

### Stream progress

```ts
import { isKnownEvent } from '@cominty-ai/sdk'

const run = await client.chat.start({ agentId, message: 'Research X and summarize.' })

for await (const event of run) {
    if (!isKnownEvent(event)) continue // an event this SDK version does not know

    switch (event.name) {
        case 'tool_call':
            console.log('tool', event.data.name, event.status)
            break
        case 'llm':
            console.log('llm', event.data.description, event.data.model)
            break
        case 'intermediary_update':
            console.log('note', event.data.message)
            break
        case 'result':
            console.log('cost', event.data.cost.total) // a decimal string
            break
    }
}

console.log(await run.text()) // already received: no second request
```

Call `isKnownEvent` first. It lets TypeScript narrow `event.data` in each
branch, and it skips event types a newer server has added.

A run is single-use. Iterate it once, or await its result once. If you
break out of the loop early, `run.result()` throws `SDKError`: read the
message later with `client.threads.get(threadId)`.

### Continue the thread

```ts
const reply = await client.chat.send(run.thread.id, {
    agentId: '__cominty_agents::agent.chat',
    message: 'Can you go into more detail?',
})
console.log(await reply.text())
```

The run that `chat.send` returns has no `thread`. Keep the thread id you
already hold.

### Answer the agent's questions

```ts
const run = await client.chat.start({ agentId, message: 'Book me a meeting room.' })

const questions = await run.questions() // empty when the agent answered
for (const question of questions) console.log(question.prompt, question.options)

if (questions.length > 0) {
    const reply = await client.chat.send(run.thread.id, { agentId, message: 'Tomorrow at 10am' })
    console.log(await reply.text())
}
```

### Threads

```ts
const threads = await client.threads.list({ limit: 20 }) // newest first, no messages
await client.threads.list({ terms: ['invoice'] }) // free-text search
await client.threads.list({ limit: 10, page: 1 }) // pages start at 0

const thread = await client.threads.get(threadId) // with its messages
await client.threads.update(threadId, { name: 'Renamed', starred: true })
await client.threads.archive(threadId) // a soft delete
```

### Message parameters

`chat.start` and `chat.send` both take:

| Argument | Type | Notes |
|---|---|---|
| `agentId` | `string` | Required. |
| `message` | `string` | Required. At most 30,000 characters. |
| `name` | `string` | `start` only. Names the new thread. |
| `fileIds` | `string[]` | Files uploaded earlier. At most 5. |
| `sourceIds` | `number[]` | Restrict retrieval to these knowledge sources. |
| `documentIds` | `string[]` | Restrict retrieval to these documents. |
| `disabledTools` | `DisablableTool[]` | `'web'`, `'company_documents'`, `'mcp:<server>'`, `'mcp:*'`. |
| `signal` | `AbortSignal` | Cancels the request and the run's stream. |

Bad values throw `InvalidParams` before any request is sent. It names every
bad argument at once.

### Configuration

| Option | Environment variable | Default |
|---|---|---|
| `apiToken` | `COMINTY_API_KEY` | none, required |
| `userId` | `COMINTY_USER_ID` | none, required |
| `baseUrl` | `COMINTY_BASE_URL` | `https://ds.cominty.com` |
| `timeout` | none | `60000` milliseconds. Streams are exempt. |
| `fetch` | none | the global `fetch` |
| `dangerouslyAllowBrowser` | none | `false` |

Each option resolves as explicit argument, then environment variable, then
default. The SDK does not read `.env` files.

---

## Python

```bash
pip install cominty-sdk   # or: uv add cominty-sdk
```

Python 3.9+. The client is async only.

### Start a conversation

```python
import asyncio
from cominty_sdk import AsyncCominty

async def main() -> None:
    # Reads COMINTY_API_KEY and COMINTY_USER_ID from the environment.
    async with AsyncCominty() as client:
        run = await client.chat.start(
            agent_id="__cominty_agents::agent.chat",
            message="Hello! What can you help me with?",
        )
        print(await run.text())
        print("thread:", run.thread.id)

asyncio.run(main())
```

The snippets below run inside that `async with` block, with
`AGENT_ID = "__cominty_agents::agent.chat"`.

### Stream progress

```python
from cominty_sdk import events

run = await client.chat.start(agent_id=AGENT_ID, message="Research X and summarize.")

async for event in run:
    if isinstance(event, events.ToolCall):
        print("tool", event.data.name, event.status)
    elif isinstance(event, events.LlmStep):
        print("llm", event.data.description)
    elif isinstance(event, events.Result):
        print("cost", event.data.cost.total)  # a Decimal

print(await run.text())  # already received: no second request
```

### Continue the thread

```python
reply = await client.chat.send(
    run.thread.id,
    agent_id=AGENT_ID,
    message="Can you go into more detail?",
)
print(await reply.text())
```

### Answer the agent's questions

```python
run = await client.chat.start(agent_id=AGENT_ID, message="Book me a meeting room.")

questions = await run.questions()  # empty when the agent answered
for question in questions:
    print(question.prompt, question.options)

if questions:
    reply = await client.chat.send(run.thread.id, agent_id=AGENT_ID, message="Tomorrow at 10am")
    print(await reply.text())
```

### Threads

```python
threads = await client.threads.list(limit=20)   # newest first, no messages
await client.threads.list(terms=["invoice"])    # free-text search
await client.threads.list(limit=10, page=1)     # pages start at 0

thread = await client.threads.get(thread_id)    # with its messages
await client.threads.update(thread_id, name="Renamed", starred=True)
await client.threads.archive(thread_id)         # a soft delete
```

### Configuration

| Argument | Environment variable | Default |
|---|---|---|
| `api_token` | `COMINTY_API_KEY` | none, required |
| `user_id` | `COMINTY_USER_ID` | none, required |
| `base_url` | `COMINTY_BASE_URL` | `https://ds.cominty.com` |
| `timeout` | none | `60` seconds. It also bounds each wait for data on a stream. |

`chat.start` and `chat.send` take `agent_id`, `message`, `file_ids`,
`source_ids`, `document_ids` and `disabled_tools`. `start` also takes
`name`. Bad values raise `InvalidParams` before any request is sent.

---

## Plain HTTP

No SDK. You need `curl` and `jq`.

```bash
BASE="${COMINTY_BASE_URL:-https://ds.cominty.com}"
```

### Start a conversation

```bash
curl -sS "$BASE/chat" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": { "content": "Hello! What can you help me with?" },
    "options": {
      "agent_id": "__cominty_agents::agent.chat",
      "user_id": "'"$COMINTY_USER_ID"'"
    }
  }' > thread.json

THREAD_ID=$(jq -r '.id' thread.json)
MESSAGE_ID=$(jq -r '[.messages[] | select(.role == "assistant")] | last | .id' thread.json)
```

The response is the new thread. It comes back as soon as the agent has
started, so the assistant message in it is still being produced. Its id is
what you stream.

### Read the reply

```bash
curl -sSN "$BASE/chat/messages/$MESSAGE_ID/stream" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  | jq --unbuffered -Rc 'fromjson? | select(type == "object")'
```

`-N` turns off curl's buffering. `-R` with `fromjson?` reads line by line
and skips anything that is not JSON. `select(type == "object")` drops the
bare `"heartbeat"` lines.

The last object has no `correlation_id`. That is the terminal message, and
its `content` is the reply. To print only that:

```bash
curl -sSN "$BASE/chat/messages/$MESSAGE_ID/stream" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  | jq -Rr 'fromjson?
      | select(type == "object")
      | select(has("correlation_id") or has("__server_is_shutting_down") | not)
      | .content'
```

Read [streaming.md](streaming.md) before building on this.

### Continue it

```bash
curl -sS "$BASE/chat/$THREAD_ID" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": { "content": "Can you go into more detail?" },
    "options": {
      "agent_id": "__cominty_agents::agent.chat",
      "user_id": "'"$COMINTY_USER_ID"'"
    }
  }' > reply.json

MESSAGE_ID=$(jq -r '.id' reply.json)
```

This endpoint returns the new assistant message, not the thread. Stream it
the same way.

### List threads

```bash
curl -sS -G "$BASE/chat" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  --data-urlencode "user_id=$COMINTY_USER_ID" \
  --data-urlencode "limit=20" \
  | jq -r '.[] | "\(.created_at)  \(.name)  \(.id)"'
```

Free-text search repeats the key, one per term: `terms=a&terms=b`.

### Scoping the run

These fields are optional. Leave a field out when you do not want it. Do
not send it empty: an empty `source_ids` means "no source is readable", not
"all of them".

```json
{
  "message": {
    "content": "Summarise the Q3 report",
    "source_ids": [4, 9],
    "disabled_tools": ["web", "mcp:*"]
  },
  "options": {
    "agent_id": "__cominty_agents::agent.chat",
    "user_id": "user_..."
  }
}
```
