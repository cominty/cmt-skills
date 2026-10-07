# Python

`cominty-sdk`. Python 3.9+. The client is async only: `AsyncCominty`.

```bash
pip install cominty-sdk   # or: uv add cominty-sdk
```

Contents: [An async framework](#an-async-framework) ·
[A sync framework](#a-sync-framework) ·
[The three endings](#the-three-endings) · [Errors](#errors) ·
[What the agent may use](#what-the-agent-may-use) ·
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
