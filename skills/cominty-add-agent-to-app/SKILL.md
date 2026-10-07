---
name: cominty-add-agent-to-app
description: "Put a Cominty agent into a project that already exists: find the stack, install the right SDK (TypeScript or Python) or use plain HTTP for a language with no SDK, keep the API key on the server, write the server-side call, and relay progress events to the browser when the UI has to show them. Use when someone says things like: add Cominty to my app, integrate the Cominty API into this project, use Cominty to add an assistant or a chat to my Next.js, Express, FastAPI or Django app, call my Cominty agent from my backend, wire the Cominty agent into this codebase, show the agent's progress in my UI."
---

# Add a Cominty agent to an existing project

The goal is one working path: the product's own server sends a message to
a Cominty agent and gets the answer back. Build the UI after that works.

This skill changes the project. To learn the API step by step instead,
load `cominty-api-onboarding`.

## 1. Read the project before you write

| You find | The stack | Use |
|---|---|---|
| `package.json` | Node, TypeScript or JavaScript | `@cominty-ai/sdk`, on the server side |
| `pyproject.toml`, `requirements.txt`, `uv.lock`, `Pipfile` | Python | `cominty-sdk` |
| `go.mod`, `Gemfile`, `composer.json`, `pom.xml`, `build.gradle`, `Cargo.toml`, `*.csproj` | Go, Ruby, PHP, Java, Kotlin, Rust, .NET | Plain HTTP |

Then find:

- The package manager, from the lockfile. Use the one the project uses.
- Where server code lives: route handlers, API routes, server functions,
  controllers, views.
- How the project loads secrets and environment variables.
- How the project is run and tested, so you can prove the change works.

Stop and tell the developer when one of these is true:

- **There is no server side.** A static site, a single-page app or a
  mobile app has nowhere safe to keep the key. It needs a small backend
  route first. Propose the smallest one that fits how the project is
  hosted, and ask before adding it.
- **The Node setup does not fit.** The TypeScript SDK needs Node 20+ and is
  ESM only. A CommonJS project on Node below 22.12 has to load it with
  `await import('@cominty-ai/sdk')`.
- **The Python setup does not fit.** The Python SDK needs Python 3.9+ and
  is async only. A sync framework needs the pattern in
  [references/python.md](references/python.md).

## 2. Three values, and no key in the conversation

| Value | Where the developer gets it | Environment variable |
|---|---|---|
| API key | `https://platform.cominty.ai/api-keys`. A key is shown once. | `COMINTY_API_KEY` |
| User id | The console's Quickstart page (`/quickstart`), or the menu behind your avatar. It starts with `user_`. | `COMINTY_USER_ID` |
| Agent id | The Quickstart page writes their own agent id into its code. `__cominty_agents::agent.chat` is the built-in general-purpose agent. | Your choice, for example `COMINTY_AGENT_ID` |

The SDKs read the first two by themselves. The agent id is your code's own
setting: the SDKs have no default agent.

Handle the key like this:

- Ask the developer to set it in the server's environment themselves. Do
  not ask for its value. Do not print it, log it or commit it.
- Add the variable names, with empty values, to the project's example env
  file. Check the real env file is ignored by git.
- The SDKs do not read `.env` files. Rely on the loader the project
  already has, or load the file yourself.
- Do not give the variable a prefix that the framework ships to the
  browser, such as `NEXT_PUBLIC_` or `VITE_`.

## 3. Rules that hold in every stack

1. **The key stays on the server.** The browser calls your route. Your
   route calls Cominty. The TypeScript SDK refuses to construct in a
   browser for this reason.
2. **One client, built once.** Create it once and reuse it. In Python,
   close it when the app shuts down.
3. **One user id per client.** Every call acts on behalf of that Cominty
   user, and its threads are scoped to that user. The docs show how to get
   your own id, and nothing about creating one for each of your product's
   users. So a server set up with your id makes every call as you, and
   every thread it creates is yours, whoever was typing. Record on your
   side which thread belongs to which of your users. Check it before you
   continue or read a thread. Never trust a thread id from the browser
   without that check.
4. **Never resend a message blindly.** Sending is not idempotent. A retry
   of a send that did reach the server starts a second run and bills
   twice. The SDKs never retry on their own.
5. **A failed run does not throw.** The request worked; the agent did not.
   Check `status` on the final message.
6. **Do not pass raw events to end users.** Events can carry tool names,
   model names and cost. Forward only what the UI needs.

## 4. Build it

1. **Install the SDK** with the project's package manager.
   `npm install @cominty-ai/sdk`, or `pnpm add`, `yarn add`, `bun add`.
   `pip install cominty-sdk`, or `uv add cominty-sdk`.
2. **Add one server module** that owns the client and exposes one
   function: send a message, optionally in an existing thread, and return
   the thread id, the status, the text and any questions.
3. **Add the route the UI calls.** Authenticate the caller. Check the
   message is a string of at most 30,000 characters. Check the thread id,
   if there is one, belongs to the caller.
4. **Handle the three endings.**
   - An answer: `status` is `success` and there are no questions.
   - Questions: the agent needs more input. Show each `prompt` with its
     `options`. The reply to a question is the next message in the same
     thread.
   - A failure: `status` is anything but `success`, such as `failed` or
     `cancelled`. Tell the user plainly.
5. **Map the errors you can act on.** A 429 has a `scope`: `concurrency`
   is transient, `user` and `organization` are quotas. Everything else is
   a failure to report, not to retry in a loop.
6. **Only if the UI has to show progress,** add the relay in
   [references/browser-relay.md](references/browser-relay.md). There is no
   token-by-token text: progress events stream, and the reply arrives
   whole.

The code for each step is in the reference for the stack:

| Stack | Read |
|---|---|
| TypeScript, JavaScript | [references/typescript.md](references/typescript.md) |
| Python | [references/python.md](references/python.md) |
| Any other language | [references/http.md](references/http.md) |
| Progress in a browser | [references/browser-relay.md](references/browser-relay.md) |

Follow the project's own conventions for file names, error handling and
tests. The references show the Cominty part. They do not decide the
project's structure.

## 5. Prove it works

1. Call the new route once with a real message. This spends credits.
2. Open the console's Agent Sessions page,
   `https://platform.cominty.ai/chats`. It lists the sessions made through
   the API. Look for the one this call made.
3. Check the key did not leak: search the client-side build output for the
   variable name and for `x-cominty-token`. Neither should appear.
4. Run the project's own checks: type-check, lint, tests.

## 6. Hand over

Tell the developer, in a few lines:

- The files you added or changed.
- The environment variables they have to set, and where.
- Which of the three endings the UI handles, and which it does not yet.
- Anything you assumed. The thread ownership check in rule 3 is the one to
  say out loud if you left it as a comment.

## When something goes wrong

Load `cominty-troubleshooting` for a symptom, and `cominty-streaming` for
the stream's contract. For where things are in the console, load
`cominty-console`. If one of them is not installed:
`npx skills add cominty/cmt-skills --skill <name>`.

Do not invent an endpoint, a field or an option. If it is not in this
skill or in https://docs.cominty.ai, say you are not sure.
