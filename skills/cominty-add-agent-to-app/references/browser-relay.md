# Showing the agent's progress in a browser

The browser must never hold the API key, and the TypeScript SDK refuses to
run there. So the server consumes the run and relays its events to the
browser. This is the pattern from the TypeScript SDK's streaming guide:
relay the events as server-sent events from your own route.

What streams is progress: tool calls, LLM steps, notes from the agent.
There is no token-by-token text. The reply arrives whole, at the end.

Contents: [The relay, in TypeScript](#the-relay-in-typescript) ·
[Before you ship it](#before-you-ship-it) ·
[The relay, in Python](#the-relay-in-python) ·
[The browser side](#the-browser-side) · [Notes](#notes)

## The relay, in TypeScript

This is the SDK guide's example. `client` is your one `Cominty` client and
`agentId` is your agent's id.

```ts
// Any runtime with web-standard Request/Response: Next.js, Hono, Bun, Deno…
export async function POST(request: Request): Promise<Response> {
    const { message } = await request.json()
    const run = await client.chat.start({ agentId, message, signal: request.signal })

    const encoder = new TextEncoder()
    const send = (event: string, data: unknown) =>
        encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`)

    const body = new ReadableStream<Uint8Array>({
        async start(controller) {
            try {
                for await (const event of run) controller.enqueue(send('progress', event))
                controller.enqueue(send('done', await run.result()))
            } catch (error) {
                controller.enqueue(send('error', { message: (error as Error).message }))
            } finally {
                controller.close()
            }
        },
        cancel: () => run.close(),
    })

    return new Response(body, {
        headers: { 'content-type': 'text/event-stream', 'cache-control': 'no-cache' },
    })
}
```

It sends three kinds of frame: `progress` for each event, then one `done`
with the final message, or one `error`.

`signal: request.signal` stops the work when the browser goes away. That
closes your connection to Cominty. It does not stop the agent: the run
goes on, and the finished message will be in the thread.

## Before you ship it

The example forwards everything. A product should not.

1. **Check the caller first.** Authenticate, validate the message, and
   check a thread id from the browser belongs to that user, before
   `chat.start` or `chat.send`.
2. **Support follow-ups.** When the body has a thread id, call
   `client.chat.send(threadId, { agentId, message, signal: request.signal })`
   instead of `chat.start`.
3. **Forward only what the UI needs.** Events can include tool names,
   model names and cost. Send a small object of your own:

   ```ts
   import { isKnownEvent, type AnyEvent } from '@cominty-ai/sdk'

   /** What the browser may see. No tool names, no model names, no cost. */
   function toUiEvent(event: AnyEvent) {
       if (!isKnownEvent(event) || event.name === 'result') return undefined
       return {
           id: event.id,
           step: event.correlation_id,
           name: event.name,
           status: event.status,
           // The one field the agent writes for people to read.
           note: event.name === 'intermediary_update' ? event.data.message : undefined,
       }
   }
   ```

   In the loop: `const ui = toUiEvent(event)`, then
   `if (ui) controller.enqueue(send('progress', ui))`.
4. **Send a small `done` too.** The final message carries more than the
   UI needs. Send the thread id, the status, the text and the questions:

   ```ts
   const final = await run.result()
   controller.enqueue(
       send('done', {
           threadId: final.thread_id,
           status: final.status,
           text: final.content,
           questions: final.questions ?? [],
       }),
   )
   ```
5. **Do not send error text to the browser.** Log the real error on the
   server. Send a fixed message.

## The relay, in Python

The same pattern with the Python SDK: an async generator that yields
server-sent events. `cominty()` and `AGENT_ID` come from the server module
in [python.md](python.md).

```python
import json
from typing import AsyncIterator


def sse(event: str, data: dict) -> str:
    return f"event: {event}\ndata: {json.dumps(data)}\n\n"


async def relay(message: str) -> AsyncIterator[str]:
    """Yield server-sent events for one run: progress, then done or error."""
    try:
        run = await cominty().chat.start(agent_id=AGENT_ID, message=message)
        async with run:  # releases the stream if the browser goes away
            async for event in run:
                if event.name == "result":
                    continue  # the reply comes with "done"
                yield sse("progress", {
                    "id": event.id,
                    "step": event.correlation_id,
                    "name": event.name,
                    "status": event.status,
                })
            final = await run.result()
            yield sse("done", {
                "threadId": str(final.thread_id),
                "status": final.status.value,
                "text": final.content,
                "questions": [q.model_dump() for q in final.questions or []],
            })
    except Exception:
        # A ComintyError, or anything else that broke the stream. Log the real
        # error here. The browser gets a fixed message and a clean end.
        yield sse("error", {"message": "The agent is unavailable."})
```

Return the generator from the framework's streaming response, with the
media type `text/event-stream`. In FastAPI:

```python
from fastapi.responses import StreamingResponse


@app.post("/api/agent")
async def agent(body: dict) -> StreamingResponse:
    # Your own auth and checks first.
    return StreamingResponse(
        relay(body["message"]),
        media_type="text/event-stream",
        headers={"cache-control": "no-cache"},
    )
```

## The browser side

The relay answers a `POST`, so the browser's `EventSource` cannot read it:
`EventSource` only sends `GET`. Read the response with `fetch`. This code
never talks to Cominty. It only talks to your route.

```ts
interface Progress {
    id: string
    step: number
    name: string
    status: string
    note?: string
}

interface Final {
    threadId: string
    status: string
    text: string
    questions: { prompt: string; options: string[] }[]
}

interface Handlers {
    onProgress: (event: Progress) => void
    onDone: (final: Final) => void
    onError: (error: { message: string }) => void
}

export async function askWithProgress(message: string, handlers: Handlers): Promise<void> {
    const response = await fetch('/api/agent', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ message }),
    })
    if (!response.ok || !response.body) throw new Error(`HTTP ${response.status}`)

    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    let buffer = ''

    for (;;) {
        const { value, done } = await reader.read()
        if (done) break
        buffer += decoder.decode(value, { stream: true })

        // A frame ends with a blank line: "event: <name>\ndata: <json>\n\n"
        let end = buffer.indexOf('\n\n')
        while (end !== -1) {
            const frame = buffer.slice(0, end)
            buffer = buffer.slice(end + 2)
            end = buffer.indexOf('\n\n')

            const name = /^event: (.*)$/m.exec(frame)?.[1]
            const data = /^data: (.*)$/m.exec(frame)?.[1]
            if (!name || !data) continue

            const payload = JSON.parse(data)
            if (name === 'progress') handlers.onProgress(payload)
            else if (name === 'done') handlers.onDone(payload)
            else if (name === 'error') handlers.onError(payload)
        }
    }
}
```

This reads the frames the relay above writes: one `event:` line and one
`data:` line each. It is not a general SSE parser.

In the UI:

- Group progress by `step`. A step sends `running` first, then a final
  status. Steps overlap, so do not show them as a strict sequence.
- Key the list by `id`.
- When `done` has questions, show each `prompt` with its `options`. The
  choice goes back as the next message, with the same thread id.
- When `done` has a status other than `success`, say the run failed.
- A `done` whose text is a recap that asks whether to continue is a
  normal answer: the agent reached its cap on tool rounds. Show it, and
  let the person reply with the same thread id.

## Notes

- If all the events arrive at once at the end, something between the
  route and the browser is buffering the response: a proxy, a compression
  layer, or the framework. Fix that there. The relay itself writes each
  event as it comes.
- The browser can leave before the run ends. The run still finishes on
  Cominty's side. To show the answer later, read the thread on the server
  (`threads.get`) and send the last message.
- One request relays one run. A new message is a new request.
