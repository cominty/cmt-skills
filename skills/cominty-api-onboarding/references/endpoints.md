# The HTTP surface

Base URL: `https://ds.cominty.com`.
Requests authenticate with the header `x-cominty-token: <api key>`.

Every chat and thread request is scoped to one end user. Chat calls send
it in the body as `options.user_id`. Thread listings send it as the
`user_id` query parameter. The SDKs set it once on the client, so their
methods never take it. Memory file calls send no `user_id`: see
[Memory](#memory).

An error response is JSON with a `detail` field: a string, an object, or a
list (a 422 validation error).

---

## Conversations

These seven are on the public API reference, and they are the
conversation calls the SDKs make.

| Method | Path | Purpose | Returns |
|---|---|---|---|
| `POST` | `/chat` | Start a thread with a first message. | The thread, with its messages. |
| `POST` | `/chat/{thread_id}` | Send a follow-up. | The new assistant message. |
| `GET` | `/chat/messages/{message_id}/stream` | Stream a reply's progress as JSON Lines. See [streaming.md](streaming.md). | Events, then the message. |
| `GET` | `/chat` | List threads. | Summaries, without messages. |
| `GET` | `/chat/{thread_id}` | One thread with all its messages. | The thread. |
| `PUT` | `/chat/{thread_id}` | Rename, star or unstar. Body: `name`, `starred`. | A summary, without messages. |
| `DELETE` | `/chat/{thread_id}` | Archive a thread. | Nothing to read. |

Response shapes differ per endpoint. `POST /chat` returns a whole thread,
but `POST /chat/{thread_id}` returns a bare message. The OpenAPI document
also lets `POST /chat/{thread_id}` answer with the shutdown notice in
place of the message: an object with `__server_is_shutting_down` and
`partial`.

`GET /chat` query parameters:

| Parameter | Notes |
|---|---|
| `user_id` | Required with an API key. |
| `limit` | Default 50, at most 100. |
| `page` | Starts at 0. |
| `terms` | Free-text search. Repeat the key: `?terms=invoice&terms=refund`. At most 10. |

### Request body

This is what the SDKs send to `POST /chat` and `POST /chat/{thread_id}`:

```json
{
  "name": "an optional thread name, POST /chat only",
  "message": {
    "content": "...",
    "file_ids": ["..."],
    "source_ids": [4, 9],
    "document_ids": ["..."],
    "disabled_tools": ["web"]
  },
  "options": {
    "agent_id": "__cominty_agents::agent.chat",
    "user_id": "user_...",
    "max_steps": 5,
    "memory_namespace": "an optional memory namespace, POST /chat only"
  }
}
```

- `message.content` and `options.agent_id` are always required.
- `options.user_id`: with an API key, `POST /chat` rejects a request
  without it with a `400`. The SDKs send it on the follow-up too. The
  OpenAPI document does not list it there.
- `content` is at most 30,000 characters. `file_ids` holds at most 5.
- `disabled_tools` values: `"web"`, `"company_documents"`,
  `"mcp:<server-name>"`, or `"mcp:*"` for every MCP server at once.
- Leave `source_ids` and `disabled_tools` out when you do not want them.
  An empty `disabled_tools` means "nothing is disabled". An empty
  `source_ids` means "no source is readable", which is almost never what
  you meant.
- `options.max_steps` is optional: see
  [Capping tool rounds](#capping-tool-rounds).
- `options.memory_namespace` is optional, and `POST /chat` only: see
  [Memory](#memory).

### Capping tool rounds

`options.max_steps` caps how many tool rounds the agent runs for that one
message. One round is one call to the model plus the tools it picks.

- A JSON integer of 1 or more. No upper bound, no "unlimited". `null`, a
  value below 1, a float or a string is a `422`, and nothing is created.
- Left out, the server default applies: 60 today, for the built-in agents
  and every custom one. It may change.
- It is per message, not per thread. A follow-up without it runs with the
  server default again, whatever the first message used.
- Reaching it is not an error. The agent stops using tools, writes a short
  recap and asks whether to continue. The message ends with `status`
  `success` and `error_code` null. No field, event or error code says the
  cap was reached, and the wording is the model's, so do not parse it.
  Continue with an ordinary follow-up in the same thread.
- It is an order of magnitude: it is checked after each round, so the
  agent can run `max_steps + 1` rounds, one round can hold several tool
  calls, and each sub-agent counts its own. It limits rounds, not tokens,
  cost or time.
- The value is not stored and not returned. Agents have no such setting.
- Do not send a very large cap for an expensive model. The budget is
  checked once, when the agent starts, and the cost is deducted at the
  end, so a long run can spend well past the budget before anything stops
  it.

### Message shape

```jsonc
{
  "id": "...",
  "thread_id": "...",
  "role": "assistant",        // or "user"
  "content": "...",
  "status": "success",        // or "pending" / "running" / "failed" / "cancelled"
  "error_code": null,         // "budget_exhausted" when the budget ran out
  "live": false,              // true while the message is still being produced
  "questions": null,          // clarifying questions instead of an answer
  "events": null,             // the stored event log, not the stream's events
  "files": [],
  "structured_output": null,
  "agent": { "id": "...", "name": "..." }   // or "ARCHIVED", or null
}
```

Nothing in this shape says that a run stopped at its `max_steps` cap, or
which memory namespace the thread uses.

---

## Memory

An agent's memory is a set of files. A memory namespace is the name of one
set: a string the caller chooses. A file is identified by its namespace
and its path. These six endpoints are on the public API reference. The
Python SDK calls them from 0.5.0. The TypeScript SDK 0.1.0 does not: they
come in the release after it.

### On a thread

`options.memory_namespace` on `POST /chat` attaches the new thread to a
namespace. A string of at most 128 characters, not trimmed.

- It is fixed for the thread's life. `POST /chat/{thread_id}` has no such
  field, and ignores a value sent there. To switch, start a new thread.
- Left out, the thread uses the namespace set on the agent, if it has one.
  Otherwise it runs with no memory at all, and no error, field or event
  says so.
- A value on the request wins over the agent's. "No memory" cannot be
  forced on an agent that has a namespace.
- Thread and message responses do not echo it. Keep the value you sent.
- Through an API key a namespace is shared by the whole organization.
  `user_id` is not part of its scope. To keep end users apart, put an
  identifier in the name: `support-bot-<user id>`.
- It is not a security boundary between agents: any agent of the
  organization started with that name can read and write it.
- Memory written before namespaces existed sits in a namespace named
  `{user_id}::default`, one per former user. Pass that exact string to
  reach it.

### Memory files

No `user_id` is sent on these calls.

| Method | Path | Purpose | Where things go | Returns |
|---|---|---|---|---|
| `GET` | `/memory` | List files. | Query: `namespace`, optional. Without it, every file in the organization. | Summaries, newest update first. |
| `GET` | `/memory/namespaces` | List namespaces. | Nothing. | The names that hold at least one file, in no order. |
| `POST` | `/memory` | Create a file. | Body: `path`, `namespace`, `purpose`, `content`. | `201` and the file. |
| `GET` | `/memory/file` | Read a file. | Query: `path`, `namespace`. | The file. |
| `PUT` | `/memory/file` | Update a file. | Query: `path`, `namespace`, `version`. Body: only what changes, `content` or `purpose`. | The file. |
| `DELETE` | `/memory/file` | Delete a file. | Query: `path`, `namespace`. | `204`, no body. |

```json
{
  "path": "tone.md",
  "namespace": "brand-voice",
  "purpose": "writing style",
  "content": "Keep it casual.",
  "created_at": "...",
  "updated_at": "...",
  "version": "..."
}
```

A summary is the same without `content`.

- `namespace`: with an API key it is required on create, read, update and
  delete. Without it the API answers `400` `Missing namespace`.
- `path` has at most one folder: `tone.md` and `preferences/tone.md` are
  fine, `a/b/tone.md` is a `422` (`Maximum folder depth is 1`).
- `purpose` is why the file exists. The agent reads it. Trimmed, then 1 to
  100 characters.
- `content` may be empty, and is at most 1 MiB. A file named `User.md` has
  a shorter limit.
- `version` is an opaque token from the last read. Send it back unchanged
  on an update, percent-encoded like any query value. Stale is a `409`.
  Malformed is a `422`. Never parse or compare it.
- An update is partial: at least one of `content` and `purpose`. A JSON
  `null` is ignored: `200`, and nothing changed. A field cannot be
  cleared.
- A create on a path that exists in that namespace is a `409`. A read or
  a delete of a missing path is a `404`. Delete is not idempotent.
- There is no call to create or delete a namespace. The first file creates
  it, and deleting its last file removes it from the list.

---

## Agents and files

The public reference at https://docs.cominty.ai/api-reference documents
the conversation and memory endpoints above. They are also everything the
two SDKs call.

- **Agents.** A call names its agent by id. The three built-in ones work
  in every workspace: `__cominty_agents::agent.chat`,
  `__cominty_agents::agent.hive`, `__cominty_agents::agent.planner`. A
  workspace's own agents have ids of their own. How to list or create them
  over HTTP is not on the public reference: read it before you try.
- **Files.** `message.file_ids` attaches files that were uploaded before,
  at most 5. How to upload one is not on the public reference either.

---

If this file disagrees with the SDK's types or with the API reference,
trust them.
