# Python

`cominty-sdk`. Python 3.9+. The client is async only: `AsyncCominty`.

```bash
pip install cominty-sdk   # or: uv add cominty-sdk
```

Contents: [An async framework](#an-async-framework) ·
[A sync framework](#a-sync-framework) ·
[The three endings](#the-three-endings) · [Errors](#errors) ·
[What the agent may use](#what-the-agent-may-use) ·
[Capping tool rounds](#capping-tool-rounds) · [Memory](#memory) ·
[Giving up on a run](#giving-up-on-a-run)

## An async framework

FastAPI, Starlette, aiohttp, async Django views. One client for the
process, closed when the app shuts down.

```python
from __future__ import annotations

import os
from typing import Optional

from cominty_sdk import AsyncCominty

# Your own setting. The SDK has no default agent and does not read this.
AGENT_ID = os.environ.get("COMINTY_AGENT_ID", "__cominty_agents::agent.chat")

_client: Optional[AsyncCominty] = None


def cominty() -> AsyncCominty:
    """One client for the whole process.

    Reads COMINTY_API_KEY and COMINTY_USER_ID from the environment.
    """
    global _client
    if _client is None:
        _client = AsyncCominty()
    return _client


async def close_cominty() -> None:
    """Call once, when the app shuts down."""
    global _client
    if _client is not None:
        await _client.close()
        _client = None


async def ask_agent(message: str, thread_id: Optional[str] = None) -> dict:
    """Send one message. Pass `thread_id` to continue a conversation."""
    client = cominty()
    if thread_id is None:
        run = await client.chat.start(agent_id=AGENT_ID, message=message)
        thread_id = str(run.thread.id)
    else:
        # The run from chat.send has no thread: keep the id you were given.
        run = await client.chat.send(thread_id, agent_id=AGENT_ID, message=message)

    reply = await run.result()  # waits for the agent to finish
    return {
        "thread_id": thread_id,
        "status": reply.status.value,
        "text": reply.content,
        "questions": [question.model_dump() for question in reply.questions or []],
    }
```

In the route that calls `ask_agent`:

1. Authenticate the caller with the project's own auth.
2. Check the message is a string of at most 30,000 characters.
3. If a thread id came from the browser, check it belongs to the caller.

Register `close_cominty` with the framework's shutdown hook.

## A sync framework

Flask, sync Django views, a script. The SDK has no sync client. Run the
call with `asyncio.run`, and create the client inside that call. The SDK
wraps an `httpx.AsyncClient`, and that belongs to the event loop it was
created in, so do not keep one client across calls to `asyncio.run`.

```python
import asyncio
import os

from cominty_sdk import AsyncCominty

AGENT_ID = os.environ.get("COMINTY_AGENT_ID", "__cominty_agents::agent.chat")


async def _ask(message: str) -> str:
    async with AsyncCominty() as client:
        run = await client.chat.start(agent_id=AGENT_ID, message=message)
        return await run.text()


def ask_agent_sync(message: str) -> str:
    return asyncio.run(_ask(message))
```

## The three endings

```python
answer = await ask_agent(message, thread_id)

if answer["status"] != "success":
    # The run failed or was cancelled. Nothing was raised.
    print("the run ended as", answer["status"])
elif answer["questions"]:
    # The agent asked instead of answering. Show each prompt and its
    # options. The reply is the next message in the same thread:
    #   await ask_agent(chosen_option, answer["thread_id"])
    for question in answer["questions"]:
        print(question["prompt"], question["options"])
else:
    # An answer. It can be a recap that asks whether to continue, when the
    # agent reached its cap on tool rounds. Show it like any other.
    print(answer["text"])
```

`await run.questions()` returns the same questions as a list of
`Question` objects, with `prompt` and `options`. It is empty when the
agent gave an answer.

## Errors

```python
from cominty_sdk import APIError, InvalidParams, RateLimitError

try:
    answer = await ask_agent(message, thread_id)
except RateLimitError as error:
    # "concurrency" clears when a run finishes. "user" and "organization"
    # are quotas: a retry will not help before the reset.
    print("rate limited:", error.scope, error.retry_after, error.reset_at)
except InvalidParams as error:
    for problem in error.errors:
        print(problem["param"], problem["message"])
except APIError as error:
    print("Cominty API error", error.status_code, error.detail)
```

| Raised | When |
|---|---|
| `ValueError` | `AsyncCominty()` could not be built: the key or the user id is missing, or the user id is malformed. Fix it. Do not catch it. |
| `InvalidParams` | Arguments failed the SDK's own checks. No request was sent. |
| `AuthError`, `PermissionError`, `NotFoundError`, `ConflictError`, `RateLimitError`, `ServerError` | The API answered 401, 403, 404, 409, 429 or 5xx. All are `APIError`. |
| `APIConnectionError` | No response arrived: a network failure or a timeout. |
| `StreamInterrupted` | The server shut down mid-stream. `.partial` is the message so far. |

`APIError` has `status_code`, `detail`, `body` and `headers`.

Do not wrap a send in a retry helper. An `APIConnectionError` on
`chat.start` or `chat.send` means no response arrived, and the message may
still have been accepted.

## What the agent may use

Tools are on by default. A call can only turn things off, or narrow
retrieval:

```python
run = await cominty().chat.start(
    agent_id=AGENT_ID,
    message=message,
    disabled_tools=["web", "mcp:*"],  # no web search, no MCP servers
    source_ids=[4, 9],                # retrieval from these knowledge sources only
)
```

`disabled_tools` takes `"web"`, `"company_documents"`, `"mcp:<server>"`
for one MCP server, or `"mcp:*"` for all of them. Leave an argument out to
mean "everything the agent has". Do not pass an empty `source_ids`.

## Capping tool rounds

`max_steps` caps how many tool rounds the agent runs for one message. It
needs `cominty-sdk` 0.5.0 or later: `pip show cominty-sdk` gives the
installed version.

```python
first = await cominty().chat.start(agent_id=AGENT_ID, message=message, max_steps=5)
reply = await first.result()
# At the cap the agent stops, recaps and asks whether to continue.
# reply.status is still "success": nothing marks the cap.

# The cap is per message. Without it, this one runs with the server default.
second = await cominty().chat.send(
    first.thread.id,
    agent_id=AGENT_ID,
    message="Yes, continue.",
    max_steps=5,
)
print(await second.text())
```

- An integer of 1 or more. There is no "unlimited" and no upper bound.
  `None`, `0`, a negative number, a float, a bool or a string raises
  `InvalidParams` before any request is sent.
- Leave it out to let the server decide: 60 today, and it may change.
- `None` does not mean "no cap". To pass an optional cap through your own
  function, default it to the SDK's sentinel:

  ```python
  from cominty_sdk import SERVER_DEFAULT, MaxSteps

  async def ask_agent(message: str, max_steps: MaxSteps = SERVER_DEFAULT) -> str:
      run = await cominty().chat.start(agent_id=AGENT_ID, message=message, max_steps=max_steps)
      return await run.text()
  ```

- Do not parse the reply to find out whether the cap was reached. No
  field, event or error code says so, and the wording is the model's.
- It is an order of magnitude: the agent can run `max_steps + 1` rounds,
  one round can hold several tool calls, and each sub-agent counts its
  own. It does not limit tokens, cost or time.
- Do not set a very large cap on an expensive model. The budget is checked
  once, when the agent starts, and the cost is deducted at the end.
- The value is not stored and not returned. Keep it on your side.

## Memory

A thread has memory only when it starts with a namespace, or when its
agent has one. Read [memory.md](memory.md) before you choose the name: a
namespace is shared by the whole organization. Needs `cominty-sdk` 0.5.0
or later.

```python
# Built on the server, from the user you authenticated. Never from the request.
namespace = f"support-bot-{end_user_id}"

run = await cominty().chat.start(
    agent_id=AGENT_ID,
    message=message,
    memory_namespace=namespace,
)

# A follow-up takes no namespace: the thread keeps the one it started with.
reply = await cominty().chat.send(run.thread.id, agent_id=AGENT_ID, message="And in French?")
```

`chat.send` has no `memory_namespace` argument: passing one is a
`TypeError`. Nothing on `run.thread` or on a message echoes the namespace.

The memory files:

```python
from cominty_sdk import ConflictError

namespace = "brand-voice"

await cominty().memory.create(
    path="tone.md",
    namespace=namespace,
    purpose="writing style",  # why the file exists: the agent reads it
    content="Keep it casual.",
)

summaries = await cominty().memory.list(namespace=namespace)  # no content
names = await cominty().memory.list_namespaces()              # a list of str

file = await cominty().memory.get("tone.md", namespace=namespace)
try:
    file = await cominty().memory.update(
        "tone.md",
        namespace=namespace,
        version=file.version,  # from the last read, unchanged
        content="Keep it upbeat.",
    )
except ConflictError:
    # The file changed since you read it. Read it again, then redo the change.
    file = await cominty().memory.get("tone.md", namespace=namespace)

await cominty().memory.delete("tone.md", namespace=namespace)
```

`list()` with no argument returns every file in the organization.
`update` changes only what you pass, `content` or `purpose`.

| Raised | When |
|---|---|
| `InvalidParams` | Before any request: a namespace over 128 characters, a `path` more than one folder deep, an `update` with neither `content` nor `purpose`, or with one of them set to `None`. |
| `ConflictError` | `create` on a path that exists in that namespace. `update` with a stale `version`. |
| `NotFoundError` | `get` or `delete` of a path that is not in that namespace. A second `delete` raises it too. |
| `APIError` with `status_code` 422 | A malformed `version`. |
| `TypeError` | `namespace` left out of `create`, `get`, `update` or `delete`. |

## Giving up on a run

```python
import asyncio

run = await cominty().chat.start(agent_id=AGENT_ID, message=message)
try:
    text = await asyncio.wait_for(run.text(), timeout=120)
except asyncio.TimeoutError:
    await run.aclose()  # release the connection
    text = None
```

This stops your side waiting. It does not stop the agent. The run goes on,
and the finished message will be in the thread:
`await cominty().threads.get(run.thread.id)`.
