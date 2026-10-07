# TypeScript and JavaScript

`@cominty-ai/sdk`. Node 20+, or a server-side runtime with global `fetch`
and `ReadableStream` (Bun, Deno, edge workers). ESM only. No runtime
dependencies. Server side only.

```bash
npm install @cominty-ai/sdk   # or: pnpm add, yarn add, bun add
```

Contents: [The server module](#the-server-module) ·
[The route](#the-route) · [The three endings](#the-three-endings) ·
[What the agent may use](#what-the-agent-may-use) ·
[Capping tool rounds](#capping-tool-rounds) · [Memory](#memory) ·
[Before the SDK has them](#before-the-sdk-has-them) ·
[Cancelling and timeouts](#cancelling-and-timeouts) ·
[Other runtimes](#other-runtimes)

## The server module

One file, server side only. Put it where the project keeps server code.

```ts
import { Cominty, type Message, type Question } from '@cominty-ai/sdk'

// Your own setting. The SDK has no default agent and does not read this.
const agentId = process.env.COMINTY_AGENT_ID ?? '__cominty_agents::agent.chat'

let client: Cominty | undefined

/** One client for the whole process. It holds no persistent connection. */
export function cominty(): Cominty {
    // Reads COMINTY_API_KEY and COMINTY_USER_ID from the environment.
    // Built on first use, so a build step with no secrets does not fail.
    client ??= new Cominty()
    return client
}

export interface AgentAnswer {
    threadId: string
    status: Message['status']
    text: string
    questions: Question[]
}

function toAnswer(threadId: string, reply: Message): AgentAnswer {
    return {
        threadId,
        status: reply.status,
        text: reply.content,
        questions: reply.questions ?? [],
    }
}

/** Sends one message. Pass `threadId` to continue a conversation. */
export async function askAgent(message: string, threadId?: string): Promise<AgentAnswer> {
    if (threadId === undefined) {
        const run = await cominty().chat.start({ agentId, message })
        return toAnswer(run.thread.id, await run.result())
    }
    // The run from chat.send has no `thread`: keep the id you were given.
    const run = await cominty().chat.send(threadId, { agentId, message })
    return toAnswer(threadId, await run.result())
}
```

`run.result()` waits for the agent to finish and returns the final
message. A run can take a while, so call this from a route that is allowed
to run that long.

## The route

A handler on web-standard `Request` and `Response`: Next.js route
handlers, Hono, Bun, Deno. In Express or Fastify, keep the body of the
function and adapt the request and the response.

```ts
import { APIError, InvalidParams, RateLimitError } from '@cominty-ai/sdk'

// askAgent is the function from the server module above.
export async function POST(request: Request): Promise<Response> {
    // 1. Your own auth: who is calling?
    // 2. If the body has a threadId, check it belongs to that user.
    const body = (await request.json()) as { message?: unknown; threadId?: string }
    const { message, threadId } = body
    if (typeof message !== 'string' || message.length === 0 || message.length > 30_000) {
        return Response.json({ error: 'invalid_message' }, { status: 400 })
    }

    try {
        return Response.json(await askAgent(message, threadId))
    } catch (error) {
        if (error instanceof RateLimitError) {
            // 'concurrency' clears when a run finishes. 'user' and
            // 'organization' are quotas: a retry will not help before the reset.
            return Response.json({ error: 'rate_limited', scope: error.scope }, { status: 429 })
        }
        if (error instanceof InvalidParams) {
            return Response.json({ error: 'invalid_request' }, { status: 400 })
        }
        if (error instanceof APIError) {
            console.error('Cominty API error', error.status, error.detail)
            return Response.json({ error: 'agent_unavailable' }, { status: 502 })
        }
        throw error
    }
}
```

What can throw, and where:

| Call | Can throw |
|---|---|
| `new Cominty()` | A plain `Error`: the key or the user id is missing, the user id is malformed, or the code runs in a browser. Fix it. Do not catch it. |
| `chat.start`, `chat.send` | `InvalidParams`, `APIError`, `APIConnectionError` |
| `run.result()`, `run.text()`, `run.questions()`, iterating a run | `APIError`, `APIConnectionError`, `StreamInterrupted`, `SDKError` |
| `threads.*` | `APIError`, `APIConnectionError` |

A run can fail while it streams, after `chat.start` has succeeded. Wrap
the code that consumes the run, not only the call that creates it.

Do not wrap a send in a retry helper. An `APIConnectionError` on
`chat.start` or `chat.send` means no response arrived, and the message may
still have been accepted.

## The three endings

```ts
const answer = await askAgent(message, threadId)

if (answer.status !== 'success') {
    // The run failed or was cancelled. Nothing was thrown.
    console.error('the run ended as', answer.status)
} else if (answer.questions.length > 0) {
    // The agent asked instead of answering. Show each prompt and its
    // options. The reply is the next message in the same thread:
    //   askAgent(chosenOption, answer.threadId)
    for (const question of answer.questions) console.log(question.prompt, question.options)
} else {
    // An answer. It can be a recap that asks whether to continue, when the
    // agent reached its cap on tool rounds. Show it like any other.
    console.log(answer.text)
}
```

## What the agent may use

Tools are on by default. A call can only turn things off, or narrow
retrieval:

```ts
const run = await cominty().chat.start({
    agentId,
    message,
    disabledTools: ['web', 'mcp:*'], // no web search, no MCP servers
    sourceIds: [4, 9], // retrieval from these knowledge sources only
})
```

`disabledTools` takes `'web'`, `'company_documents'`, `'mcp:<server>'` for
one MCP server, or `'mcp:*'` for all of them. Leave an option out to mean
"everything the agent has". Do not pass an empty `sourceIds`.

## Capping tool rounds

`maxSteps` caps how many tool rounds the agent runs for one message. It is
not in `@cominty-ai/sdk` 0.1.0: see
[Before the SDK has them](#before-the-sdk-has-them).

```ts
const first = await cominty().chat.start({ agentId, message, maxSteps: 5 })
const reply = await first.result()
// At the cap the agent stops, recaps and asks whether to continue.
// reply.status is still 'success': nothing marks the cap.

// The cap is per message. Without it, this one runs with the server default.
const second = await cominty().chat.send(first.thread.id, {
    agentId,
    message: 'Yes, continue.',
    maxSteps: 5,
})
console.log(await second.text())
```

- An integer of 1 or more. There is no "unlimited" and no upper bound.
  Anything else is rejected: over the wire that is a 422.
- Leave it out, or pass `undefined`, to let the server decide: 60 today,
  and it may change.
- Do not parse the reply to find out whether the cap was reached. No
  field, event or error code says so, and the wording is the model's.
- It is an order of magnitude: the agent can run `maxSteps + 1` rounds,
  one round can hold several tool calls, and each sub-agent counts its
  own. It does not limit tokens, cost or time.
- Do not set a very large cap on an expensive model. The budget is checked
  once, when the agent starts, and the cost is deducted at the end.
- The value is not stored and not returned. Keep it on your side.

## Memory

A thread has memory only when it starts with a namespace, or when its
agent has one. Read [memory.md](memory.md) before you choose the name: a
namespace is shared by the whole organization. Not in 0.1.0 either.

```ts
// Built on the server, from the user you authenticated. Never from the request.
const memoryNamespace = `support-bot-${endUserId}`

const run = await cominty().chat.start({ agentId, message, memoryNamespace })

// A follow-up takes no namespace: the thread keeps the one it started with.
const reply = await cominty().chat.send(run.thread.id, { agentId, message: 'And in French?' })
```

`chat.send` does not accept `memoryNamespace`. Nothing on `run.thread` or
on a message echoes the namespace.

The memory files:

```ts
import { ConflictError } from '@cominty-ai/sdk'

const namespace = 'brand-voice'

await cominty().memory.create({
    path: 'tone.md',
    namespace,
    purpose: 'writing style', // why the file exists: the agent reads it
    content: 'Keep it casual.',
})

const summaries = await cominty().memory.list({ namespace }) // no content
const names = await cominty().memory.listNamespaces() // string[]

let file = await cominty().memory.get('tone.md', { namespace })
try {
    file = await cominty().memory.update('tone.md', {
        namespace,
        version: file.version, // from the last read, unchanged
        content: 'Keep it upbeat.',
    })
} catch (error) {
    if (!(error instanceof ConflictError)) throw error
    // The file changed since you read it. Read it again, then redo the change.
    file = await cominty().memory.get('tone.md', { namespace })
}

await cominty().memory.delete('tone.md', { namespace })
```

`list()` with no argument returns every file in the organization. `update`
changes only what you pass: `content`, `purpose`, or both. One is needed.
A 409 is a `ConflictError` and a 404 a `NotFoundError`, as on every call:
the cases are in [memory.md](memory.md).

## Before the SDK has them

`maxSteps`, `memoryNamespace` and `client.memory` are not in
`@cominty-ai/sdk` 0.1.0. They come in the release after it. Check what is
installed before you write them:

```bash
npm ls @cominty-ai/sdk   # or the project's own package manager
```

On 0.1.0, TypeScript rejects the two arguments, plain JavaScript drops
them without an error, and `client.memory` is `undefined`. Do not cast
past the type error: the request would go out without them.

Until the project can upgrade, send these two options over plain HTTP and
keep the SDK for the rest. `chat.stream` reads a message that any request
started:

```ts
import type { Thread } from '@cominty-ai/sdk'

const baseUrl = process.env.COMINTY_BASE_URL ?? 'https://ds.cominty.com'

/** `chat.start` by hand, with the two options the SDK does not send yet. */
async function startWithOptions(
    message: string,
    options: { max_steps?: number; memory_namespace?: string },
) {
    const response = await fetch(`${baseUrl}/chat`, {
        method: 'POST',
        headers: {
            'x-cominty-token': process.env.COMINTY_API_KEY ?? '',
            'content-type': 'application/json',
        },
        body: JSON.stringify({
            message: { content: message },
            // An option left undefined is left out of the JSON. Never send null.
            options: { agent_id: agentId, user_id: process.env.COMINTY_USER_ID, ...options },
        }),
    })
    if (!response.ok) throw new Error(`Cominty answered HTTP ${response.status}`)

    const thread = (await response.json()) as Thread
    const reply = [...thread.messages].reverse().find((m) => m.role === 'assistant')
    if (!reply) throw new Error('the new thread has no assistant message')

    // No request is made until the run is consumed.
    return { threadId: thread.id, run: cominty().chat.stream(reply.id) }
}

const { threadId, run } = await startWithOptions(message, { max_steps: 5 })
console.log(await run.text())
```

A follow-up is the same request to `/chat/{thread_id}`, with no
`memory_namespace`. It returns the new assistant message, so its `id` is
the one to pass to `chat.stream`. Errors here are plain HTTP statuses, not
SDK classes: [http.md](http.md) has them, and the memory file calls.

## Cancelling and timeouts

Pass an `AbortSignal`. It covers the request and the stream.

```ts
const controller = new AbortController()
const timer = setTimeout(() => controller.abort(), 120_000)

try {
    const run = await cominty().chat.start({ agentId, message, signal: controller.signal })
    console.log(await run.text())
} catch (error) {
    if (error instanceof Error && error.name === 'AbortError') {
        console.log('gave up waiting')
    } else {
        throw error
    }
} finally {
    clearTimeout(timer)
}
```

- Your own abort surfaces as the standard `AbortError`, not as a
  `ComintyError`.
- Cancelling stops your connection. It does not stop the agent. The run
  goes on, and the finished message will be in the thread:
  `cominty().threads.get(threadId)`.
- In a route handler, pass `request.signal` so the work stops when the
  browser goes away.
- The client's `timeout` option (60,000 ms by default) covers ordinary
  requests only. Streams have no overall timeout.

## Other runtimes

- **No `process.env`** (some edge workers): pass both values yourself.

  ```ts
  const client = new Cominty({ apiToken: env.COMINTY_API_KEY, userId: env.COMINTY_USER_ID })
  ```

- **CommonJS on Node below 22.12**: load the package with a dynamic import.

  ```ts
  const { Cominty } = await import('@cominty-ai/sdk')
  ```

- **A browser**: not supported. `dangerouslyAllowBrowser: true` exists for
  trusted environments only. It is not a way to ship the key to users.
