---
name: cominty-streaming
description: "The exact contract of the Cominty run stream, GET /chat/messages/{message_id}/stream, for someone writing their own stream reader or debugging one: JSON Lines and not SSE, heartbeat lines, progress events versus the terminal message, why the result event is not the end, a last line with no newline, resuming with Last-Event-Id, and cancelling. Use when someone says things like: parse the Cominty stream, my stream hangs or never ends, the stream stops too early, my parser breaks on heartbeat, is this SSE, how do I reconnect or resume a run, how do I cancel a run, why is there no token-by-token text, I am writing a Cominty client in Go, Ruby, PHP, Java, Rust or C#."
---

# The Cominty run stream

One endpoint streams a run: `GET /chat/messages/{message_id}/stream`. This
skill is its contract, rule by rule, with the reason for each rule.

## Do you need this?

In TypeScript or Python, use the SDK. `for await (const event of run)` and
`async for event in run` do the reading for you. Read on when you are
writing a client in another language, or debugging a reader that
misbehaves.

## The request

```
GET /chat/messages/{message_id}/stream
x-cominty-token: <api key>
Last-Event-Id: <event id>        only when resuming
```

The base URL is `https://ds.cominty.com`. The key goes in
`x-cominty-token`. An `Authorization: Bearer` header gets a 401.

Where the message id comes from:

- `POST /chat` returns the new thread. The reply is the last entry in
  `messages` whose `role` is `"assistant"`.
- `POST /chat/{thread_id}` returns the new assistant message itself.

Both come back as soon as the agent has started. The message is still
being produced. Its `id` is what you stream.

## What comes back

`Content-Type: application/jsonl`. One JSON value per line.

```
{"id":"...","correlation_id":1,"at":"...","name":"waiting_for_start","status":"running"}
{"id":"...","correlation_id":1,"at":"...","name":"waiting_for_start","status":"success"}
"heartbeat"
{"id":"...","correlation_id":3,"at":"...","name":"tool_call","status":"running","data":{...}}
{"id":"...","correlation_id":4,"at":"...","name":"result","status":"success","data":{"reply":"...",...}}
{"id":"...","correlation_id":2,"at":"...","name":"setting_up_sandbox","status":"success"}
{"id":"...","thread_id":"...","role":"assistant","content":"...","status":"success",...}
```

| Line | How to recognise it | What to do |
|---|---|---|
| Heartbeat | A bare JSON string, `"heartbeat"` | Skip it. |
| Progress event | An object that has `correlation_id` | Show progress. Remember its `id`. |
| Terminal message | An object with no `correlation_id`. It has `thread_id`, `role`, `content`, `status`. | The run is over. Stop reading. |
| Shutdown notice | An object with `__server_is_shutting_down: true` and `partial` | The server stopped mid-run. Not the end. |

Test each line in this order: is it an object, is it the shutdown notice,
does it have `correlation_id`. What is left is the terminal message. The
shutdown notice has no `correlation_id` either, so check for it before
you call a line terminal.

Event names and fields are in [references/events.md](references/events.md).

## The rules

**1. It is JSON Lines, not SSE.** There is no `event:` or `data:` prefix,
no `[DONE]` line and no `retry:`. An SSE client library will not parse it.
Split the body on `\n` and parse each line as JSON.

**2. Skip every line that is not a JSON object.** A heartbeat is the bare
string `"heartbeat"`, sent during long idle steps. It parses, but it is
not an object. Both SDKs also skip blank lines and lines that are not JSON
at all, instead of failing the stream. Do the same.

**3. Stop on the terminal message, not on `result`.** `result` carries the
reply, but the stream stays open and more events can follow it. A
`setting_up_sandbox` success can arrive after it, for one. The stream ends
with the terminal message: the object with no `correlation_id`.

**4. Parse what is left when the body ends.** The last line can arrive
with no trailing newline. A reader that only emits a line when it sees
`\n` never sees the terminal message. From the outside that looks like a
stream that never ends.

**5. Do not wait for text deltas.** This is progress streaming, not token
streaming. The reply arrives whole: in `result` (`data.reply`) and again
in the terminal message (`content`).

**6. Order holds inside a step, not across steps.** Events that share a
`correlation_id` belong to one step and arrive in order: `running`, then a
final status. Steps overlap, so do not expect one step to finish before
another starts.

**7. Read failure from the terminal message.** There is no error event. A
failed run ends with a terminal message whose `status` is `"failed"`.
Check `status`, then `error_code`: `"budget_exhausted"` means the
workspace's budget ran out.

**8. Ignore event names you do not know.** The server can add event types
at any time. Read `name`, handle the ones you know, pass over the rest.

A reply that stops half way and asks whether to continue is not a reader
bug. The agent reached its cap on tool rounds (`options.max_steps` on the
request, or the server default). The stream ends as usual, with `status`
`"success"`, and no event or field marks it.

## Resuming

A connection can drop while the run carries on. Keep the `id` of the last
event you handled, then reconnect to the same URL with:

```
Last-Event-Id: <that id>
```

The server sends what came after it and continues live. Leave the header
out and the stream starts from the top. Deduplicate by event `id` on your
side, so an overlap can never show a step twice.

You can also stream a message that has already finished. The stream still
ends with its terminal message, so a late reconnect ends correctly.

The shutdown notice is a different case. The server is going away, and
`partial` holds the message as far as it got. Do not treat it as the
answer. Wait a moment and reconnect, or read the thread later with
`GET /chat/{thread_id}`.

## Cancelling

Two different things are called cancelling.

**Stop reading.** Close the connection. This does not stop the agent: it
keeps running on the server, and the finished message will be in the
thread. You can stream the same message id again later.

**Stop the run.** `cancelled` is one of the statuses a message can end
with, and a cancellation reaches the stream as the terminal message, not
as an event. How to ask for one is not on the public API reference and
neither SDK does it, so do not build on it without reading the reference
first.

## The same things in the SDKs

| Need | TypeScript | Python |
|---|---|---|
| Read events | `for await (const event of run)` | `async for event in run` |
| Final message | `await run.result()`, `await run.text()` | the same |
| Stop reading | an `AbortSignal` passed as `signal`, `break`, or `await run.close()` | `await run.aclose()`, or `async with run:` |
| Last event id | `run.lastEventId` | not public in 0.4.0 |
| Reattach | `client.chat.stream(messageId, { lastEventId })` | `client.chat.stream(message_id)`, from the top |
| Connection dropped | throws `APIConnectionError` | raises `APIConnectionError` |
| Server shutdown | throws `StreamInterrupted`, with `.partial` | raises `StreamInterrupted`, with `.partial` |
| Unknown event name | `isKnownEvent(event)` is false | an `events.UnknownEvent` |
| The client's `timeout` | does not apply to streams | bounds each wait for data on a stream: 60 seconds by default |

A run is single-use in both SDKs. Iterate it once, or await its result
once. In TypeScript your own abort surfaces as the standard `AbortError`.

## Before you ship a hand-written reader

Feed it each of these and check what it does. Each one maps to a rule
above.

- A `"heartbeat"` line between two events.
- A JSON object split across two network chunks.
- A `result` event followed by another event.
- A terminal message with no trailing newline.
- A terminal message with `status: "failed"`.
- A shutdown notice in place of the terminal message.
- A reconnect that replays an event the reader has already handled.
- An event with a `name` the reader has never seen.

## References

| File | For |
|---|---|
| [references/readers.md](references/readers.md) | Complete readers in TypeScript, Python and shell, with resume. The SDK's own resume loop. |
| [references/events.md](references/events.md) | Every event name and its `data`, the cost fields, the terminal message |

For the first call and the non-streaming basics, load
`cominty-api-onboarding`. For a symptom you cannot place, load
`cominty-troubleshooting`.
