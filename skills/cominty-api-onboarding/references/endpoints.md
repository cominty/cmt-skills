# The HTTP surface

Base URL: `https://ds.cominty.com`.
Requests authenticate with the header `x-cominty-token: <api key>`.

Every request is scoped to one end user. Chat calls send it in the body as
`options.user_id`. Thread listings send it as the `user_id` query
parameter. The SDKs set it once on the client, so their methods never take
it.

An error response is JSON with a `detail` field: a string, an object, or a
list (a 422 validation error).

---

## Conversations

These seven are the endpoints on the public API reference, and the ones the
SDKs call.

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
    "user_id": "user_..."
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

---

## Agents and files

The public reference at https://docs.cominty.ai/api-reference documents
the seven endpoints above. They are also everything the two SDKs call.

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
