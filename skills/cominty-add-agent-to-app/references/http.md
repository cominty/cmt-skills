# Plain HTTP, for a language with no SDK

Go, Ruby, PHP, Java, Kotlin, Rust, .NET, or anything else that can send
HTTPS and JSON. Write a small client inside the project's server code.

A message takes two requests: one starts the run, one reads its stream.

## What every request needs

| Thing | Value |
|---|---|
| Base URL | `https://ds.cominty.com` |
| Header | `x-cominty-token: <the API key>`. Not `Authorization: Bearer`: that gets a 401. |
| User id | In the body, as `options.user_id`, for chat calls. As the `user_id` query parameter for `GET /chat`. |

Read the key, the user id and the agent id from the server's
configuration. Never from the request, never from the client.

## 1. Start the run

`POST /chat` starts a new thread. `POST /chat/{thread_id}` continues one.
The SDKs send the same body to both, except that only `POST /chat` takes
a `name`.

```bash
BASE="${COMINTY_BASE_URL:-https://ds.cominty.com}"

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
```

Both calls return as soon as the agent has started. What they return
differs:

| Call | Returns | The message id to stream |
|---|---|---|
| `POST /chat` | The new thread, with `id` and `messages` | The `id` of the last entry in `messages` whose `role` is `"assistant"` |
| `POST /chat/{thread_id}` | The new assistant message | Its `id` |

```bash
THREAD_ID=$(jq -r '.id' thread.json)
MESSAGE_ID=$(jq -r '[.messages[] | select(.role == "assistant")] | last | .id' thread.json)
```

Store the thread id with the user it belongs to. You need it for
follow-ups, and for the ownership check before any later read.

`message` can also carry `file_ids` (at most 5), `source_ids`,
`document_ids` and `disabled_tools`. Leave them out when you do not need
them. `content` is at most 30,000 characters.

## 2. Read the stream until the terminal message

```bash
curl -sSN "$BASE/chat/messages/$MESSAGE_ID/stream" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  | jq --unbuffered -Rc 'fromjson? | select(type == "object")'
```

The response is JSON Lines: one JSON value per line. It is not SSE. In
your language, the reader is this:

```text
buffer = ""
for each chunk of the response body:
    buffer = buffer + chunk
    while buffer contains "\n":
        line, buffer = split buffer at the first "\n"
        if handle(line) is "done": stop reading
handle(buffer)                    # the last line can have no newline

handle(line):
    value = parse line as JSON; if it does not parse, skip it
    if value is not an object: skip it            # "heartbeat"
    if value has "__server_is_shutting_down": the server stopped; reconnect later
    if value has "correlation_id": a progress event; show it or ignore it
    otherwise: the terminal message; return "done"
```

On the terminal message:

| Field | Meaning |
|---|---|
| `status` | `success`, `failed` or `cancelled`. A failed run still ends this way: there is no error event. |
| `content` | The reply text |
| `questions` | A list of `{ "prompt", "options" }` when the agent asked for more input. Otherwise null or an empty list. |
| `error_code` | `"budget_exhausted"` when the workspace's budget ran out. Otherwise null. |

Five rules, each one a way hand-written readers fail:

1. Skip every line that is not a JSON object.
2. Do not stop on the `result` event. More events can follow it.
3. Parse what is left in the buffer when the body ends.
4. There is no text token by token. The reply arrives whole.
5. If the connection drops, reconnect to the same URL with
   `Last-Event-Id: <the id of the last event you handled>`, and skip any
   event id you have already seen.

The `cominty-streaming` skill has the full contract, the event fields, and
complete readers to port.

## 3. Answer questions and follow up

A follow-up, and the reply to a question, are the same call:

```bash
curl -sS "$BASE/chat/$THREAD_ID" \
  -H "x-cominty-token: $COMINTY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": { "content": "Tomorrow at 10am" },
    "options": {
      "agent_id": "__cominty_agents::agent.chat",
      "user_id": "'"$COMINTY_USER_ID"'"
    }
  }' > reply.json

MESSAGE_ID=$(jq -r '.id' reply.json)
```

Then read that message's stream as in step 2.

## 4. Errors

An error response is JSON with a `detail` field.

| Status | Meaning | What to do |
|---|---|---|
| `400` | A bad request. A missing `user_id` is one cause: it is required with an API key. | Send `options.user_id` |
| `401` | The key is missing, mistyped or revoked, or it was sent as a bearer token | Send it in `x-cominty-token` |
| `403` | The key is valid but may not access this resource | Check the id |
| `404` | Wrong thread, message or agent id | Check the id |
| `409` | The request conflicts with the resource's current state | Read `detail` |
| `422` | The body failed validation. `detail` is a list that names each field. | Fix the body |
| `429` | A limit was hit. See below. | Depends on which |
| `5xx` | A problem on Cominty's side | Retry with backoff. Mind the note on sends. |

A `429` is one of these:

- `detail` is the string `"Too many concurrent requests"`: too many chat
  sessions at once. Transient. Retry when a run in flight finishes.
- `detail` is an object with `quota_reached` set to `"user"` or
  `"organization"`, and a `reset_at` time: a quota is used up. Retrying
  before the reset does not help.
- A `Retry-After` header, when present, gives the seconds to wait.

Sends are not idempotent. If a `POST` times out or the connection breaks,
the message may still have been accepted. Resending it can start a second
run and bill twice. Look at the thread before you resend.

## Other calls

`GET /chat` lists threads, `GET /chat/{thread_id}` returns one with its
messages, `PUT /chat/{thread_id}` renames or stars it, and
`DELETE /chat/{thread_id}` archives it. The `cominty-api-onboarding`
skill's endpoint reference has them all, and so does
https://docs.cominty.ai/api-reference.
