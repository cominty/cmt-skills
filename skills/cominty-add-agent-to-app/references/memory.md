# Memory: namespaces and memory files

An agent's memory is a set of files. A memory namespace is the name of
one set: a string your code chooses. A file is identified by its namespace
and its path. Threads started with the same namespace share the same
files. Two namespaces never see each other's.

A thread keeps its own context from message to message without any of
this. Memory is for what has to outlast the thread: a user's preferences,
a brand's tone.

Contents: [Which memory a thread gets](#which-memory-a-thread-gets) ·
[Who shares a namespace](#who-shares-a-namespace) ·
[Memory files](#memory-files) · [Errors](#errors) ·
[Memory written before namespaces](#memory-written-before-namespaces) ·
[Check it works](#check-it-works) · [Availability](#availability)

The calls are in the reference for the stack:
[typescript.md](typescript.md), [python.md](python.md), [http.md](http.md).

## Which memory a thread gets

It is decided once, by the call that starts the thread (`POST /chat`).

| The first message | The agent | The thread uses |
|---|---|---|
| Sends a namespace | Has one of its own, or not | The one that was sent. It wins over the agent's. |
| Sends none | Has one of its own | The agent's. |
| Sends none | Has none | No memory at all. No error, field or event says so. |

- Send the namespace on every start where you expect memory. A thread
  with no memory looks exactly like one that has it.
- The namespace is fixed for the thread's life. A follow-up cannot send
  one: `send` does not take it in the SDKs, and over HTTP
  `POST /chat/{thread_id}` ignores it. To switch, start a new thread.
- Thread and message responses do not echo the namespace. Store the name
  next to the thread id if you need it later.
- "No memory" cannot be forced on an agent that has a namespace of its
  own. Leaving the field out uses the agent's.

## Who shares a namespace

Through an API key, a namespace is shared by the whole organization. The
`user_id` on the call is not part of its scope. Every thread started with
`support-bot` reads and writes the same files, whoever the end user is.

| You want | Name it |
|---|---|
| One memory for everyone: a brand's tone, a team's glossary | A fixed name: `brand-voice` |
| One memory per end user | With the user's id in it: `support-bot-<user id>` |

For one memory per end user:

- Build the name on your server, from the user you authenticated. Never
  take a namespace from the browser: whoever chooses the name reads that
  memory.
- Use an id that does not change. A different name is a different, empty
  memory.
- At most 128 characters, and not trimmed: `"acme "` and `"acme"` are two
  namespaces.

A namespace is not a security boundary between agents. Any agent of the
organization started with that name can read and write it. Do not store a
secret that only one agent should see.

## Memory files

Your code can read and write the files the agent uses: to fill a memory
before the first conversation, to show what the agent keeps, or to delete
it. No `user_id` is sent on these calls.

| Call | Method and path | Where things go | Returns |
|---|---|---|---|
| List files | `GET /memory` | Query: `namespace`, optional. Without it, every file in the organization. | Summaries, newest update first |
| List namespaces | `GET /memory/namespaces` | Nothing | The names that hold at least one file, in no order |
| Create | `POST /memory` | Body: `path`, `namespace`, `purpose`, `content` | `201` and the file |
| Read | `GET /memory/file` | Query: `path`, `namespace` | The file |
| Update | `PUT /memory/file` | Query: `path`, `namespace`, `version`. Body: only what changes, `content` or `purpose`. | The file, with a new `version` |
| Delete | `DELETE /memory/file` | Query: `path`, `namespace` | `204`, no body |

A file:

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

| Field | Rules |
|---|---|
| `path` | At most one folder: `tone.md` and `preferences/tone.md` are fine, `a/b/tone.md` is a 422. |
| `namespace` | With an API key, required on create, read, update and delete. At most 128 characters. |
| `purpose` | Why the file exists. The agent reads it. Trimmed, then 1 to 100 characters. |
| `content` | May be empty. At most 1 MiB. A file named `User.md` has a shorter limit. |
| `version` | An opaque token. Send back the one from your last read, unchanged. Never parse or compare it. |

- There is no call to create or delete a namespace. The first file
  creates it. Deleting its last file removes it from the list.
- Update is partial: send at least one of `content` and `purpose`. A JSON
  `null` is ignored: the API answers 200 and changes nothing. A field
  cannot be cleared.
- After an update, use the `version` of the file it returned for the next
  one.
- Delete is not idempotent: a second delete of the same path is a 404.
- Without the `namespace` filter, a list can hold the same `path` several
  times, once per namespace. Key a file by namespace and path.

## Errors

| Status | SDK class | When |
|---|---|---|
| `400` `Missing namespace` | `APIError` | Create, read, update or delete with no `namespace`. |
| `404` | `NotFoundError` | Read or delete of a path that is not in that namespace. |
| `409` | `ConflictError` | Create on a path that exists in that namespace. Update with a stale `version`: the file changed since you read it. |
| `422` | `APIError` | A `path` more than one folder deep (`Maximum folder depth is 1`). A malformed `version`. |

On a 409 from an update, read the file again, redo your change on what
you read, and send the new `version`. On a 409 from a create, the file is
there already: read it and update it.

## Memory written before namespaces

Memory written before namespaces existed is not lost. It sits in a
namespace named `{user_id}::default`, one per former user: for the user
`user_abc`, that is `user_abc::default`. Pass that exact string as the
namespace to reach it, on a new thread or on the memory file calls. A new
thread does not use it unless you pass it. `GET /memory/namespaces` lists
the names that hold files.

## Check it works

A thread with no memory raises nothing, so test it on purpose.

1. Create a file in the namespace, with a line you will recognise.
2. Start a new thread with that namespace. Ask the agent what its memory
   says.
3. If it does not know, compare the namespace on the file with the one
   the thread was started with, character by character. Check it was sent
   on the call that started the thread, not on a follow-up.

After a real conversation, list the namespace's files to see what the
agent kept.

## Availability

| Stack | Memory namespaces and memory files |
|---|---|
| Python | `cominty-sdk` 0.5.0 or later. |
| TypeScript | Not in `@cominty-ai/sdk` 0.1.0. They come in the release after it. Check the installed version, and use plain HTTP until it has them. |
| Plain HTTP | Works today. |
