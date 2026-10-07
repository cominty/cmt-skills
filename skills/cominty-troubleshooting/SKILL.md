---
name: cominty-troubleshooting
description: "Symptom to cause to fix for the Cominty agent API and its SDKs: a 401 (a bearer token instead of the x-cominty-token header), the kinds of 429, a stream that never ends or stops early, questions coming back instead of an answer, a run that failed, costs as decimal strings, a client constructed in a browser, a missing or malformed user id. Use when someone says things like: my Cominty call returns 401 or 429, unauthorized, rate limited, the stream hangs, I get no answer, the reply is empty, the agent asked me a question back, the SDK throws when I create the client, invalid userId, the costs do not add up, it works with curl but not in my app."
---

# Troubleshooting the Cominty API

Start from the symptom. Before you suggest a fix, get the evidence:

1. The HTTP status, or the error's class name and its message.
2. The stack: the TypeScript SDK, the Python SDK, or a hand-written client.
3. Where the code runs: a server, a browser, an edge runtime.

Never ask for the API key. To check how it is sent, ask for the name of
the header, not its value.

## 401 and 403

| Symptom | Cause | Fix |
|---|---|---|
| `401` on every call | The key is sent as `Authorization: Bearer <key>` | Send it as `x-cominty-token: <key>`. A bearer token gets a 401. Look for an HTTP client's bearer helper, or an API tool's auth tab, adding the wrong header. |
| `401` with the right header (`AuthError`) | The key is missing, mistyped or revoked | Check the variable is set in the process that makes the call. If in doubt, create a new key on the console's `/api-keys` page. A key is shown once: an old one cannot be read back. |
| `403` (`PermissionError`) | The key is valid but may not access this resource | Check the thread, message or agent id is the one you meant. |

## The client will not construct

These are plain `Error` in TypeScript and `ValueError` in Python, not
`ComintyError`. They are mistakes to fix, not conditions to catch.

| Message | Cause | Fix |
|---|---|---|
| `Refusing to construct a Cominty client in a browser` | The TypeScript SDK was constructed in browser code. The key would be readable by anyone who opens devtools. | Move the call to your own server and have the browser call that. Load `cominty-add-agent-to-app`. `dangerouslyAllowBrowser: true` is for trusted environments only. It is not the fix for a public page. |
| `userId is required`, `user_id is required` | No user id was passed and `COMINTY_USER_ID` is not set in that process | Set it. The id is on the console's Quickstart page (`/quickstart`), with a copy button, and in the menu behind your avatar. |
| `invalid userId`, `invalid user_id` | The value is not `user_` followed by at least 20 letters or digits. Often a cut-off copy, stray quotes or spaces, or a different id. | Copy it again from the console. |
| `apiToken is required`, `api_token is required`, yet the key is in `.env` | The SDKs do not read `.env` files | Export the variables, or load the file yourself. In Node: `node --env-file=.env app.js`. |
| The same, on an edge runtime | There is no `process.env` | Pass `apiToken` and `userId` to `new Cominty({ ... })` yourself. |
| `require()` of the SDK fails | `@cominty-ai/sdk` is ESM only | Node 22.12+ can `require` it. On older Node use `await import('@cominty-ai/sdk')`. |

Over plain HTTP there is no local check. With an API key, `user_id` is
required: as `options.user_id` when you start a chat, where a missing one
is rejected with a `400`, and as a query parameter for `GET /chat`.

## 429

A `429` is one of three problems. The SDKs tell you which in
`error.scope`.

| `scope` | Meaning | What to do |
|---|---|---|
| `concurrency` | Too many chat sessions running at once | Transient. Retry when a run in flight finishes. |
| `user` | This user's request quota is used up | Wait for the reset, or have an admin raise the limit. |
| `organization` | The organization's quota is used up | Wait for the reset, or have an admin raise the limit. |
| `null` or `None` | The response did not say | Treat it as transient. |

- `RateLimitError` also has the seconds to wait (`retryAfter` in
  TypeScript, `retry_after` in Python) and the reset time (`resetAt`,
  `reset_at`). Either can be empty. Its message already says which limit
  was hit, so it is fit for logs as it is.
- Over plain HTTP, read `detail`. A quota looks like
  `{"quota_reached": "user", "reset_at": "..."}`. The concurrency limit is
  the string `"Too many concurrent requests"`. A `Retry-After` header may
  be present.
- Retrying a quota 429 before the reset does not help.
- Plan and credits are on the console's `/settings/plan` page.

## The stream

| Symptom | Cause | Fix |
|---|---|---|
| The stream never ends, or hangs at the very end | The reader emits a line only when it sees `\n`, and the last line has none. Or it waits for `[DONE]` or an end event that never comes. | Parse what is left in the buffer when the body ends. Stop on the object with no `correlation_id`. |
| The stream stops too early | The reader stopped on the `result` event | `result` is mid-stream. More events can follow it. |
| An SSE client shows nothing | The stream is JSON Lines, not SSE | Split on `\n` and parse each line as JSON. |
| The reader breaks on a line now and then | A `"heartbeat"` line. It parses to a string, not an object. | Skip every line that is not a JSON object. |
| No text until the end | There is no text-delta event | Progress streams. The reply arrives whole. |
| Steps show twice after a reconnect | The replay repeated events | Deduplicate by event `id`. |
| `SDKError: stream was partially consumed` | The loop was left early, then `result()` or `text()` was called | Iterate to the end. Or read the thread with `threads.get`, or reattach with `chat.stream`. |
| `SDKError: ... already been consumed`, or a loop that yields nothing | A run was iterated twice. TypeScript throws. Python gives an empty loop. | A run is single-use. Start from its result, or from a new stream. |
| `APIConnectionError` while iterating | The connection dropped. The run goes on. | Reattach with `chat.stream`. Do not send the message again. |
| `StreamInterrupted` | The server shut down mid-stream. `.partial` is the message so far. | Reattach, or read the thread later. |
| Python: `APIConnectionError: Stream to ... timed out` | No data arrived for longer than the client's `timeout`, 60 seconds by default. In Python it bounds each wait on a stream. In TypeScript streams are exempt. | Build the client with a larger `timeout`. Reattach with `chat.stream`. |
| Python: a pydantic `ValidationError` while iterating a run | An event carried a value the SDK's model does not accept. Version 0.4.0 types `status` as `running`, `success` or `error`. The TypeScript SDK also lists `failed`. | It is not a `ComintyError`, so catch it by name. Then read the final message from the thread with `threads.get`. |

The full contract and complete readers are in the `cominty-streaming`
skill.

## No answer came back

| Symptom | Cause | Fix |
|---|---|---|
| Questions instead of an answer | The agent needs more input, so it ended its turn with `questions` | `await run.questions()`. Show each `prompt` and its `options`. Send the chosen option, or free text, with `chat.send` in the same thread. |
| The final message has `status` `failed` | The run failed. Nothing is thrown: the request worked, the agent did not. | Check `status` on the result of `run.result()`. Then check `error_code`. |
| `error_code` is `budget_exhausted` | The workspace's budget ran out during the run | Say so plainly. A workspace admin is the person to ask. |
| The agent does not retrieve from your sources | The request had an empty `source_ids`, which means "no source". Or `company_documents` is in `disabled_tools`. | Leave a field out to mean "everything the agent has". |
| A connected tool is not used | `mcp:<server>` or `mcp:*` is in `disabled_tools`, or the tool is not connected | Check the request. Tools are connected on the console's `/integrations` page. |

The Python SDK's `Message` does not carry `error_code`. Read it from
`GET /chat/{thread_id}`. In TypeScript it is on the object as
`message.error_code`.

## A request is rejected or lost

| Symptom | Cause | Fix |
|---|---|---|
| `InvalidParams`, before any request | The SDK checked the arguments: a message over 30,000 characters, more than 5 file ids, an unknown tool name, and in TypeScript an empty `agentId` | `error.errors` lists every problem at once. |
| `404` (`NotFoundError`) | Wrong thread, message or agent id | Check the id and where it came from. |
| `409` (`ConflictError`) | The request conflicts with the resource's current state | Read `detail`. |
| `422` | Plain HTTP: the body failed validation | `detail` is a list that names each bad field. |
| `5xx` (`ServerError`) | A problem on Cominty's side | Usually worth retrying, with backoff. |
| `APIConnectionError` on a send | No response arrived: network failure, or the 60 second default timeout | The message may have been accepted. Look at the thread, or the thread list, before you resend. |

The SDKs never retry on their own. Sending a message is not idempotent: a
retry of a send that did reach the server starts a second run and bills
twice. Retry reads freely. Retry sends with care.

## Costs do not add up

`cost.total`, `cost.input_cost` and `cost.output_cost` arrive as decimal
strings, on the `result` event.

- In JavaScript, `+` on two of them joins the strings.
- Turning them into floats loses precision once you sum many of them.
- Use a decimal library. The Python SDK already gives `Decimal`.

The console's Usage page is `/dashboard`. It is for workspace admins only.

## Still stuck

- The console's Agent Sessions page (`/chats`) lists the sessions made
  through the API. Look for the one your call should have made.
- Docs: https://docs.cominty.ai and https://docs.cominty.ai/api-reference.
  For an agent: https://docs.cominty.ai/llms-full.txt, or the docs' MCP
  server at https://docs.cominty.ai/mcp.
- The public docs give a contact address: hi@cominty.com.

Do not guess at an endpoint, a field or an error code that is not here or
in the docs. Say you are not sure.
