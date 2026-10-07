# Troubleshooting

Symptom first, because that is how a problem arrives. The
`cominty-troubleshooting` skill covers more cases.

---

## Authentication and setup

| Symptom | Cause | Fix |
|---|---|---|
| `401` on every call | The key was sent as `Authorization: Bearer <key>` | The header is `x-cominty-token: <key>`. |
| `401` with the right header | The key is missing, mistyped or revoked | Create a new one at `platform.cominty.ai/api-keys`. |
| The key is lost | A key is shown once | Create a new one. The old one cannot be read back. |
| The TypeScript SDK throws when constructed in a browser | By design: the key would be readable in devtools | Call the API from your own server. `dangerouslyAllowBrowser` is for trusted environments only. |
| The SDK throws on a missing or malformed user id | It is checked locally, before any request | Copy it from the console's Quickstart page, or from the menu behind your avatar. It starts with `user_`. |
| The SDK says the key is required, but it is in `.env` | The SDKs do not read `.env` files | Export the variables, or load the file yourself. |

---

## The stream

| Symptom | Cause | Fix |
|---|---|---|
| The parser fails on the first line | It expects SSE framing | The stream is JSON Lines. No `event:` or `data:`, no `[DONE]`. |
| The reader breaks on a line now and then | A bare `"heartbeat"` line. It parses to a string, not an object. | Skip any line that is not a JSON object. |
| The stream "ends" but more was coming | The reader stopped on the `result` event | `result` is mid-stream. Stop on the terminal message: the object with no `correlation_id`. |
| It hangs and the last message never arrives | The terminal line has no trailing newline | Parse the buffer when the body ends. |
| Steps show twice after a reconnect | The replay overlapped what was already shown | Deduplicate by event `id`. |
| Waiting for text token by token | There is no text-delta event | Progress streams. The reply arrives whole, in `result` and in the terminal message. |
| `setting_up_sandbox` arrives after `llm` finished | Steps run in parallel, with no order across them | Do not sequence on it. Order holds only within one `correlation_id`. |
| The stream ends with a shutdown notice | The server stopped during the run | Reconnect with `Last-Event-Id` and continue. |

---

## Rate limits

A `429` is more than one problem. The SDKs expose which as `error.scope`:

| `scope` | Meaning | What to do |
|---|---|---|
| `concurrency` | Too many chat sessions at once. Transient. | Retry when a run in flight finishes. |
| `user` | This user's request quota is used up | Wait for the reset, or have an admin raise the limit. |
| `organization` | The organization's quota is used up | Wait for the reset, or have an admin raise the limit. |
| none | The response did not say | Treat it as transient. |

Retrying a quota 429 in a loop does not help.

---

## Runs that fail or behave oddly

| Symptom | Cause | Fix |
|---|---|---|
| The final message has `status: "failed"` and empty `content` | There is no error event. Failure is reported on the final message. | Read `status`, then `error_code`. |
| `error_code: "budget_exhausted"` | The workspace's budget ran out | Say that plainly. Do not report a generic failure. |
| Questions came back instead of an answer | The agent needs more input | Read `questions`. Send an option, or free text, as the next message in the same thread. |
| The agent ignores your knowledge sources | The request had `"source_ids": []` | Empty means "no source is readable". Leave the field out to allow everything. |
| A connected tool is not used | It is in `disabled_tools`, or it is not connected | Check the request. `"mcp:*"` disables every MCP server at once. |
| Costs drift when summed | Decimal strings were parsed to floats | Keep them as strings. Sum them with a decimal type. |
| The message is rejected before sending | It is over 30,000 characters | The SDK checks this locally and raises `InvalidParams`. |
| The reply stops half way and asks whether to continue | The agent reached its cap on tool rounds for that message: `max_steps`, or the server default, 60 today. `status` is `success`. | Not a failure. Send a follow-up in the same thread. Do not parse the text to detect it. |
| The cap held on the first message, not on the follow-up | `max_steps` is per message | Send it on every message that needs it. |
| `422` on a call that sends `max_steps` | It is not a JSON integer of 1 or more: `null`, `0`, a float, a string | Send an integer, or leave the field out. Nothing was created. |

---

## Memory

| Symptom | Cause | Fix |
|---|---|---|
| The agent does not remember anything | The thread started with no memory namespace, and the agent has none of its own. It then has no memory at all, and nothing says so. | Send `memory_namespace` on the call that starts the thread. It cannot be added on a follow-up. |
| It remembers in one thread and not in the next | The two threads started with different names. The name is not trimmed. | Start every thread with the same string. |
| One user gets what the agent learned from another | A namespace is shared by the whole organization. `user_id` is not part of it. | Put the user's id in the name: `support-bot-<user id>`. |
| Memory from before namespaces is gone | It sits in a namespace named `{user_id}::default`, one per former user | Pass that exact string as the namespace. |
| `400` `Missing namespace` | A memory file call with no `namespace` | In the body for `POST /memory`. In the query for read, update and delete. |
| `409` on a memory create | A file exists at that path in that namespace | Read it and update it. |
| `409` on a memory update | The `version` is stale: the file changed since you read it | Read it again, redo the change, send the new `version` unchanged. |
| `404` on a memory delete | The path is not in that namespace. Delete is not idempotent: a second delete is a `404`. | Check the path and the namespace, or treat it as gone. |
| The SDK does not know `max_steps`, `memory_namespace` or `client.memory` | Python: `cominty-sdk` is older than 0.5.0. TypeScript: 0.1.0 does not have them. | Upgrade Python. In TypeScript, send these over plain HTTP until the release after 0.1.0 is installed. |

---

## Still stuck

- https://docs.cominty.ai
- https://docs.cominty.ai/api-reference
- For an agent: https://docs.cominty.ai/llms-full.txt, or the docs' MCP
  server at https://docs.cominty.ai/mcp.
