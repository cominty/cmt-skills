# Stream readers

Complete readers for the run stream. Each one follows every rule in the
skill: it skips lines that are not objects, stops on the terminal message,
parses a last line with no newline, and deduplicates by event id.

Contents: [TypeScript](#typescript-with-fetch) ·
[Python](#python-with-httpx) · [Shell](#shell-with-curl-and-jq) ·
[Resuming with the SDKs](#resuming-with-the-sdks)

All of them read `COMINTY_API_KEY` from the environment.

---

## TypeScript, with fetch

No dependencies. Node 20+, Bun, Deno, or any runtime with `fetch`.

```ts
const baseUrl = process.env.COMINTY_BASE_URL ?? 'https://ds.cominty.com'

export type Line = Record<string, unknown>

/** State that survives a reconnect. Use one per run. */
export interface ReadState {
    lastEventId?: string
    seen: Set<string>
}

/** The connection failed, or closed before the terminal message. */
export class ConnectionDropped extends Error {}

/** The server sent its shutdown notice. `partial` is the message so far. */
export class ServerShutdown extends Error {
    readonly partial: Line
    constructor(partial: Line) {
        super('server shut down before the message completed')
        this.partial = partial
    }
}

/** Reads one run until its terminal message, and returns that message. */
export async function readRun(
    messageId: string,
    onEvent: (event: Line) => void,
    state: ReadState = { seen: new Set() },
    signal?: AbortSignal,
): Promise<Line> {
    const headers: Record<string, string> = {
        'x-cominty-token': process.env.COMINTY_API_KEY ?? '',
    }
    if (state.lastEventId) headers['last-event-id'] = state.lastEventId

    const response = await fetch(`${baseUrl}/chat/messages/${messageId}/stream`, {
        headers,
        signal,
    }).catch((cause: unknown) => {
        if (signal?.aborted) throw cause // your own cancellation
        throw new ConnectionDropped('could not connect', { cause })
    })
    if (!response.ok || !response.body) {
        // Not a dropped connection: 403, 404, 429, 5xx. This reader does
        // not retry them. Decide per status.
        throw new Error(`stream request failed: HTTP ${response.status}`)
    }

    /** Returns the terminal message when this line is it. */
    const handle = (raw: string): Line | undefined => {
        let parsed: unknown
        try {
            parsed = JSON.parse(raw)
        } catch {
            return undefined // blank, or not JSON: skip
        }
        if (parsed === null || typeof parsed !== 'object' || Array.isArray(parsed)) {
            return undefined // "heartbeat" is a bare string
        }
        const line = parsed as Line
        if ('__server_is_shutting_down' in line) throw new ServerShutdown(line.partial as Line)
        if (!('correlation_id' in line)) return line // the terminal message

        const id = String(line.id)
        state.lastEventId = id
        if (!state.seen.has(id)) {
            state.seen.add(id) // a replay after a reconnect can repeat an event
            onEvent(line)
        }
        return undefined
    }

    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    let buffer = ''
    try {
        for (;;) {
            const chunk = await reader.read().catch((cause: unknown) => {
                if (signal?.aborted) throw cause
                throw new ConnectionDropped('connection dropped mid-run', { cause })
            })
            buffer += chunk.done ? decoder.decode() : decoder.decode(chunk.value, { stream: true })

            const lines = buffer.split('\n')
            buffer = lines.pop() ?? '' // the tail is a whole line only after the next \n
            for (const raw of lines) {
                const terminal = handle(raw)
                if (terminal) return terminal
            }

            if (chunk.done) {
                // The terminal message can arrive with no trailing newline,
                // so it is still in the buffer here.
                const terminal = handle(buffer)
                if (terminal) return terminal
                throw new ConnectionDropped('stream ended without a terminal message')
            }
        }
    } finally {
        await reader.cancel().catch(() => {})
    }
}

/** Reads a run to its end, reconnecting when the connection drops. */
export async function readRunWithResume(
    messageId: string,
    onEvent: (event: Line) => void,
    maxAttempts = 5,
): Promise<Line> {
    const state: ReadState = { seen: new Set() }
    for (let attempt = 1; ; attempt++) {
        try {
            return await readRun(messageId, onEvent, state)
        } catch (error) {
            if (!(error instanceof ConnectionDropped) || attempt >= maxAttempts) throw error
            await new Promise((resolve) => setTimeout(resolve, 1000 * attempt))
        }
    }
}
```

Using it:

```ts
const message = await readRunWithResume(messageId, (event) => {
    console.log(event.name, event.status)
})

if (message.status === 'success') console.log(message.content)
else console.error('the run ended as', message.status, message.error_code)
```

Pass an `AbortSignal` to `readRun` to stop reading. That closes your
connection. It does not stop the agent.

---

## Python, with httpx

```bash
pip install httpx
```

```python
from __future__ import annotations

import json
import os
import time
from typing import Callable

import httpx

BASE_URL = os.environ.get("COMINTY_BASE_URL", "https://ds.cominty.com")


class ServerShutdown(Exception):
    """The server sent its shutdown notice. `partial` is the message so far."""

    def __init__(self, partial: dict) -> None:
        super().__init__("server shut down before the message completed")
        self.partial = partial


def read_run(message_id: str, on_event: Callable[[dict], None], state: dict | None = None) -> dict:
    """Read one run until its terminal message, and return that message.

    `state` survives a reconnect: pass the same dict again to resume.
    """
    state = {} if state is None else state
    seen: set = state.setdefault("seen", set())

    headers = {"x-cominty-token": os.environ["COMINTY_API_KEY"]}
    if state.get("last_event_id"):
        headers["last-event-id"] = state["last_event_id"]

    url = f"{BASE_URL}/chat/messages/{message_id}/stream"
    # No read timeout: a run can take minutes. Bound it yourself if you must.
    timeout = httpx.Timeout(30.0, read=None)

    with httpx.stream("GET", url, headers=headers, timeout=timeout) as response:
        # Not a dropped connection: 403, 404, 429, 5xx. This reader does not
        # retry them. Decide per status.
        response.raise_for_status()
        # iter_lines() also yields a last line that has no trailing newline.
        for raw in response.iter_lines():
            try:
                line = json.loads(raw)
            except json.JSONDecodeError:
                continue  # blank, or not JSON: skip
            if not isinstance(line, dict):
                continue  # "heartbeat" is a bare string
            if "__server_is_shutting_down" in line:
                raise ServerShutdown(line["partial"])
            if "correlation_id" not in line:
                return line  # the terminal message

            state["last_event_id"] = line["id"]
            if line["id"] not in seen:
                seen.add(line["id"])  # a replay after a reconnect can repeat an event
                on_event(line)

    raise httpx.RemoteProtocolError("stream ended without a terminal message")


def read_run_with_resume(message_id: str, on_event: Callable[[dict], None], max_attempts: int = 5) -> dict:
    """Read a run to its end, reconnecting when the connection drops."""
    state: dict = {}
    for attempt in range(1, max_attempts + 1):
        try:
            return read_run(message_id, on_event, state)
        except httpx.TransportError:  # dropped connection, refused connection, timeout
            if attempt == max_attempts:
                raise
            time.sleep(attempt)
    raise AssertionError("unreachable")
```

Using it:

```python
message = read_run_with_resume(message_id, lambda event: print(event["name"], event["status"]))

if message["status"] == "success":
    print(message["content"])
else:
    print("the run ended as", message["status"], message.get("error_code"))
```

An async reader has the same shape: `httpx.AsyncClient().stream(...)` and
`response.aiter_lines()`. That is what the Python SDK does.

---

## Shell, with curl and jq

```bash
BASE="${COMINTY_BASE_URL:-https://ds.cominty.com}"
```

Every line, as it arrives:

```bash
curl -sSN "$BASE/chat/messages/$MESSAGE_ID/stream" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  | jq --unbuffered -Rc 'fromjson? | select(type == "object")'
```

- `-N` turns off curl's buffering.
- `-R` with `fromjson?` reads line by line and skips what is not JSON. It
  also reads a last line that has no newline.
- `select(type == "object")` drops the `"heartbeat"` lines.

Only the reply text:

```bash
curl -sSN "$BASE/chat/messages/$MESSAGE_ID/stream" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  | jq -Rr 'fromjson?
      | select(type == "object")
      | select(has("correlation_id") or has("__server_is_shutting_down") | not)
      | .content'
```

Resuming after a drop:

```bash
curl -sSN "$BASE/chat/messages/$MESSAGE_ID/stream" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  -H "Last-Event-Id: $LAST_EVENT_ID" \
  | jq --unbuffered -Rc 'fromjson? | select(type == "object")'
```

---

## Resuming with the SDKs

### TypeScript

`run.lastEventId` is the id of the last event received.
`client.chat.stream(messageId, { lastEventId })` opens a new stream from
that point. It makes no request until the run is consumed. This loop is
the one from the SDK's streaming guide:

```ts
import { APIConnectionError, type AnyEvent, type Message } from '@cominty-ai/sdk'

async function streamWithResume(
    messageId: string,
    onEvent: (event: AnyEvent) => void,
    maxAttempts = 5,
): Promise<Message> {
    let lastEventId: string | undefined

    for (let attempt = 1; ; attempt++) {
        const run = client.chat.stream(messageId, { lastEventId })
        try {
            for await (const event of run) onEvent(event)
            return await run.result()
        } catch (error) {
            if (!(error instanceof APIConnectionError) || attempt >= maxAttempts) throw error
            lastEventId = run.lastEventId ?? lastEventId
            await new Promise((resolve) => setTimeout(resolve, 1000 * attempt))
        }
    }
}

const started = await client.chat.start({ agentId, message })
const final = await streamWithResume(started.messageId, (event) => console.log(event.name))
```

The loop does not deduplicate. If `onEvent` must never see an event twice,
keep a `Set` of `event.id` in it.

### Python

In version 0.4.0, `client.chat.stream(message_id)` takes only the message
id. It cannot resume from an event, so a new stream starts from the top
and repeats the events already seen. Deduplicate by `event.id`:

```python
import asyncio

from cominty_sdk import APIConnectionError


async def stream_with_reattach(client, message_id, on_event, max_attempts=5):
    """Read a run to its end, reattaching when the connection drops."""
    seen = set()
    for attempt in range(1, max_attempts + 1):
        run = client.chat.stream(message_id)  # no request until it is consumed
        try:
            async for event in run:
                if event.id not in seen:
                    seen.add(event.id)
                    on_event(event)
            return await run.result()
        except APIConnectionError:
            if attempt == max_attempts:
                raise
            await asyncio.sleep(attempt)
```

`run.message_id` is the id to pass. Store it if another process has to
pick the run up.
