# The streaming contract

`GET /chat/messages/{message_id}/stream`

Read this before writing a stream reader by hand. Both SDKs read the
stream by these rules, so prefer them unless there is a reason not to.
Only the TypeScript SDK can resume from an event: the Python SDK (0.4.0)
reattaches from the top.

The `cominty-streaming` skill has the same contract in more depth, with
complete readers in TypeScript, Python and shell.

---

## The wire format

`application/jsonl`: one JSON value per line. This is not server-sent
events:

- no `event:` or `data:` framing
- no `[DONE]` line
- no `retry:` directive

```
{"id":"...","correlation_id":1,"at":"...","name":"waiting_for_start","status":"running"}
{"id":"...","correlation_id":1,"at":"...","name":"waiting_for_start","status":"success"}
"heartbeat"
{"id":"...","correlation_id":3,"at":"...","name":"llm","status":"success","data":{...}}
{"id":"...","correlation_id":4,"at":"...","name":"result","status":"success","data":{"reply":"...",...}}
{"id":"...","correlation_id":2,"at":"...","name":"setting_up_sandbox","status":"success"}
{"id":"...","thread_id":"...","role":"assistant","content":"...","status":"success",...}
```

### The kinds of line

| Line | How to recognise it | What to do |
|---|---|---|
| Heartbeat | A bare JSON string, `"heartbeat"` | Skip it. It appears during long idle steps. |
| Progress event | An object with `correlation_id` | Show progress. |
| Terminal message | An object with no `correlation_id`. It has `thread_id` and `role`. | The run is over. Stop reading. |
| Shutdown notice | An object with `__server_is_shutting_down` and `partial` | The server stopped mid-run. Not the end: see below. |

Skip any line that is not a JSON object. That one rule handles heartbeats
now and anything similar later.

The shutdown notice has no `correlation_id` either. Check for it before
you decide a line is the terminal message.

---

## Four things that break hand-written readers

### 1. `result` is not the end

`result` carries the reply, but the stream stays open and more events can
follow it. A late `setting_up_sandbox` success can arrive after `result`.

One `result` is emitted for a successful run, then zero or more events,
then the terminal message. Stop on the terminal message. Never on
`result`.

### 2. The last line may have no trailing newline

Parse whatever is left in your buffer when the body ends. A reader that
only emits a line when it sees `\n` drops the terminal message, and that
looks like a hang.

### 3. There is no text-delta event

This is progress streaming, not token streaming. The reply arrives whole,
in `result` (`data.reply`) and again in the terminal message (`content`).
A typewriter effect would animate text you already have.

### 4. Order holds inside a step, not across steps

- Within one `correlation_id`: `running` comes before `success`, `failed`
  or `error`. One `correlation_id` is one step: one sandbox setup, one LLM
  call, one tool call.
- Across `correlation_id`s: no guarantee. Do not assume
  `setting_up_sandbox` finishes before `llm` starts.

---

## Event names

`waiting_for_start` · `setting_up_sandbox` · `uploading_file` · `llm` ·
`intermediary_update` · `tool_call` · `result`

Each has `id`, `correlation_id`, `at`, `name` and `status`. Most also have
`data`. The server can add new names at any time: ignore the ones you do
not know.

---

## Failure

There is no error event. A run that fails ends with a terminal message
whose `status` is `"failed"` and whose `content` is empty. It can happen
at any point, even after a `result`.

Check `status` on the terminal message. Then check `error_code`:
`"budget_exhausted"` means the workspace's budget ran out during the run.
Say that plainly instead of reporting a generic failure.

---

## Resuming a dropped connection

Reconnect with the id of the last event you processed:

```
Last-Event-Id: <event id>
```

The server sends what came after it and continues live. Deduplicate by
event `id`, so a replay that overlaps cannot show a step twice.

If you receive the shutdown notice instead, treat it as "reconnect and
continue", not as the end. `partial` is the message as far as it got.

---

## A minimal correct reader

```ts
const baseUrl = process.env.COMINTY_BASE_URL ?? 'https://ds.cominty.com'

type Line = Record<string, unknown>

/** Reads one run to its end and returns the terminal message. */
async function readRun(messageId: string, onEvent: (event: Line) => void): Promise<Line> {
    const response = await fetch(`${baseUrl}/chat/messages/${messageId}/stream`, {
        headers: { 'x-cominty-token': process.env.COMINTY_API_KEY ?? '' },
    })
    if (!response.ok || !response.body) throw new Error(`HTTP ${response.status}`)

    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    const seen = new Set<string>()
    let buffer = ''

    /** Returns the terminal message when this line is it. */
    function handle(raw: string): Line | undefined {
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
        if ('__server_is_shutting_down' in line) throw new Error('server shut down: reconnect')
        if (!('correlation_id' in line)) return line // the terminal message

        const id = String(line.id)
        if (!seen.has(id)) {
            seen.add(id) // a replay after a reconnect can repeat an event
            onEvent(line)
        }
        return undefined
    }

    try {
        for (;;) {
            const { value, done } = await reader.read()
            buffer += done ? decoder.decode() : decoder.decode(value, { stream: true })

            const lines = buffer.split('\n')
            buffer = lines.pop() ?? '' // the tail is a whole line only after the next \n

            for (const raw of lines) {
                const terminal = handle(raw)
                if (terminal) return terminal
            }

            if (done) {
                // The terminal message can arrive with no trailing newline,
                // so it is still in the buffer here.
                const terminal = handle(buffer)
                if (terminal) return terminal
                throw new Error('stream ended without a terminal message')
            }
        }
    } finally {
        await reader.cancel().catch(() => {})
    }
}
```

What this does that a naive reader does not: it parses the buffer when the
body ends, it skips lines that are not objects, it stops on the terminal
message and not on `result`, and it deduplicates by event id.
