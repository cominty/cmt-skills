# Events and the terminal message

What each line of the run stream contains. The JSON examples use made-up
values.

## Fields every event has

| Field | Type | Notes |
|---|---|---|
| `id` | string | Opaque. Looks like `"1718000000000-0"`. Send it as `Last-Event-Id` to resume after this event. |
| `correlation_id` | number | Groups the events of one step. Its presence is what makes a line an event. |
| `at` | string | ISO-8601 timestamp. |
| `name` | string | The event type. |
| `status` | string | `running`, `success`, `failed` or `error`. |
| `data` | object | Depends on `name`. Some events have none. |

A step usually emits one event with `status: "running"` and a second one
with a final status. Both share a `correlation_id`.

`success`, `failed` and `error` are final: the step emits nothing more.
Treat `failed` and `error` both as failure.

The TypeScript SDK types all four statuses. The Python SDK (0.4.0) types
three: `running`, `success` and `error`.

## Event names

| `name` | When | `data` |
|---|---|---|
| `waiting_for_start` | The run is queued, waiting for a worker | none |
| `setting_up_sandbox` | The execution sandbox is being provisioned | none |
| `uploading_file` | A file is being uploaded | `filename` |
| `llm` | An LLM call | `description`, `model`, sometimes `error` |
| `tool_call` | A tool call | `name`, `description`, `message` on success, `error` on failure |
| `intermediary_update` | A progress note from the agent, written for people | `message` |
| `result` | The agent's answer | `reply`, `files`, `questions`, `metadata`, `cost` |

Calls to a connected MCP server show up as `tool_call` events.

The server can add event names at any time. Handle the ones you know and
pass over the rest.

### Names that never arrive

`execution_failed`, `cancelled` and `end_of_stream` exist on the server
but never reach the wire as events. The server turns them into the
terminal message. Do not wait for them.

## `result`

```json
{
  "id": "1718000000002-0",
  "correlation_id": 3,
  "at": "2026-09-14T10:00:01Z",
  "name": "result",
  "status": "success",
  "data": {
    "reply": "Hello from the agent.",
    "files": [],
    "questions": null,
    "metadata": null,
    "cost": {
      "failed": false,
      "input_tokens": 10,
      "cached_tokens": 0,
      "output_tokens": 5,
      "input_cost": "0.001",
      "output_cost": "0.002",
      "total": "0.003"
    }
  }
}
```

| Field | Notes |
|---|---|
| `reply` | The whole answer, as one string. |
| `files` | Files the agent produced. Each has `id`, `name`, `mimetype`. |
| `questions` | Clarifying questions, or null. Each has `prompt` and `options`. |
| `metadata` | Depends on the agent, or null. `agent.planner` puts a `canvas` here. |
| `cost` | What the run cost. See below. |

`result` is not the end of the stream. More events can follow it.

### Cost

`input_tokens`, `cached_tokens` and `output_tokens` are numbers.
`input_cost`, `output_cost` and `total` are decimal strings.

Keep the money fields as strings, or parse them into a decimal type.
Parsing them into a binary float loses precision once you add many of them
up. The TypeScript SDK leaves them as strings. The Python SDK parses them
into `Decimal`.

## The terminal message

The last line of the stream. It has no `correlation_id`.

```json
{
  "id": "11111111-2222-3333-4444-555555555555",
  "thread_id": "99999999-8888-7777-6666-555555555555",
  "role": "assistant",
  "content": "Hello from the agent.",
  "status": "success",
  "error_code": null,
  "live": false,
  "questions": null,
  "events": null,
  "structured_output": null,
  "files": [],
  "agent": { "id": "__cominty_agents::agent.chat", "name": "..." }
}
```

| Field | Notes |
|---|---|
| `id`, `thread_id` | The message and its thread. |
| `role` | `"assistant"` here. Messages in a thread can also be `"user"`. |
| `content` | The reply text. |
| `status` | `pending`, `running`, `success`, `failed` or `cancelled`. |
| `error_code` | `"budget_exhausted"` or null. |
| `live` | Whether the message is still being produced. |
| `questions` | Clarifying questions, if the agent asked any. Otherwise null or an empty list. |
| `files` | Files attached to the message or produced by it. Each has `id`, `name`, `size`, `mimetype`, `origin` (`user` or `agent`), `url`, `share_links`. |
| `structured_output` | Structured output, for agents that produce it. Otherwise null. |
| `events` | The stored event log, in raw form. Not the same objects as the stream's events. |
| `agent` | The agent that wrote the message: an object with `id` and `name`. It can also be the string `"ARCHIVED"`, or null. |

The API can add fields. Ignore the ones you do not use.

The Python SDK's `Message` model keeps only the fields it declares, so
`error_code` is not on it. The TypeScript SDK keeps unlisted fields on the
object, so `message.error_code` is readable there.

## The shutdown notice

```json
{
  "__server_is_shutting_down": true,
  "partial": { "id": "...", "status": "running", "content": "the reply so far" }
}
```

`partial` is a message in the shape above, as far as it got. Its `status`
reflects how much was saved. This line is not the terminal message: see
"Resuming" in the skill.
