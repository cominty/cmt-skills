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
