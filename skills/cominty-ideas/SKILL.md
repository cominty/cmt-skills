---
name: cominty-ideas
description: "Propose two or three concrete things a Cominty agent could do inside the project at hand, then offer to build the first one. Reads the real codebase first, stays within what Cominty is documented to do, and says what it does not know. Use when someone asks things like: what could I build with Cominty, what can a Cominty agent do for my app, give me ideas, is Cominty a fit for this project, or when they want to add an assistant, a support bot, a help chat, a research feature, question answering over their documents, or 'chat with our docs' to their product."
---

# What could a Cominty agent do in this project?

Read the project in front of you. Propose two or three concrete things an
agent could do in it. Offer to start the first one.

Be as encouraging as a good getting-started page, and no more. The ideas
have to survive the developer trying them.

## Ground rules

- **Propose, do not promise.** Write "an agent could", "this would need".
  Do not write "Cominty will".
- **Stay inside what is documented.** The table in step 2 is the whole
  palette. If an idea needs something outside it, drop the idea, or name
  the unverified part and say how to check it.
- **No numbers you do not have.** No speed, accuracy, price or limit. No
  comparison with another product.
- **Ideas come from the project.** If you have not read the code, you are
  not ready to propose. A list of generic AI features helps nobody.
- **Plain words.** No superlatives. Say what the person using the product
  would see.

## Step 1. Read the project

Look for where an agent would save a real person real effort.

| Look at | To learn |
|---|---|
| The README and the landing copy | What the product is, and who uses it |
| Routes, pages, screens | Where people ask for help, search, or wait |
| Data models | What the product knows: orders, tickets, projects, articles |
| Docs, help pages, FAQ, policies | Text that already holds the answers |
| Support and contact flows | The questions that get asked again and again |
| Tools the code already talks to | Trackers, chat, storage: what the team works in |
| Back-office scripts and reports | Text someone rewrites or summarises by hand |

Check one thing early: does the project have a server side? The API key
must stay on a server. A front-end-only project needs a small backend
first, and that is part of the first idea's cost.

If the code does not tell you enough, ask. Three short questions at most:
who uses the product, what do they ask for help with most, and where do
the answers live today.

## Step 2. Match needs to what is documented

This is what the Cominty docs, SDKs and API show an agent doing. Build
ideas from these and nothing else.

| Building block | What is documented | What it needs |
|---|---|---|
| A conversation | Send a message to an agent and get an answer. Send follow-ups in the same thread: the agent keeps the thread's context. | A server route that holds the key |
| Questions back | When it needs more input, an agent can end its turn with questions and suggested options instead of an answer. | A way to show the options |
| Visible progress | The stream reports what the agent is doing: tool calls, LLM steps, short notes. The answer arrives whole, not token by token. | A relay from your server to the browser |
| The team's documents | Agents can search sets of uploaded files (the console's Documents page). | The files uploaded |
| Web search | On by default. A call can turn it off. | Nothing |
| The team's tools | Tools are connected through MCP (the console's Integrations page). An agent's calls to them show up as progress. | The tool connected |
| A custom agent | An agent with its own instructions and its own ordered list of models. To your code it is one more agent id. | The agent created first. How to create one is not on the public API reference, so check first. |
| Built-in agents | `agent.chat` is general purpose. `agent.hive` orchestrates several agents for complex tasks. `agent.planner` plans multi-step work. | Nothing |
| Files | A message can carry up to 5 uploaded files. A reply can carry files the agent produced. | The upload calls. They are not on the public API reference, so check them first. |
| Memory across conversations | An agent can keep memory files in a namespace: a name your code chooses when it starts a thread. Threads started with the same name share the files. Your code can read and write them too. | A namespace on every thread. It is shared by the whole organization, so one memory per end user needs the user's id in the name. |
| A cap on work per message | `max_steps` caps the tool rounds of one message. At the cap the agent stops, recaps and asks whether to continue. | A way for the person to answer "continue" |
| History | Threads can be listed, searched, renamed, starred and archived. | Nothing |
| Cost per run | A run's `result` event carries its token counts and its cost. | Nothing |

Not documented, so do not build an idea on it:

- Text that appears token by token.
- Calling Cominty straight from a browser or a mobile app.
- A separate Cominty user for each end user of the developer's product.
- Voice, images, or anything in real time.
- Any figure for answer quality, speed, price or limits.

## Step 3. Write the proposal

Two or three ideas. Put first the one that is smallest to build and
easiest to judge. Use this shape for each:

```
### <the feature, named in the product's own words>

What the user sees: <one or two sentences>
What the agent does: <one or two sentences>
Built on: <building blocks from the table>
Needs from you: <documents to upload, a tool to connect, a decision>
First version: <the smallest thing that proves it>
Not known yet: <what has to be tried or checked>
```

Then say which one you would start with, and why, in one sentence.

Shapes that often fit. Adapt them to the project. Do not paste them.

- **The repository has docs or help pages.** A question box that answers
  from those documents, and asks a question back when the request is
  vague.
- **There is a support or contact form.** An assistant that answers first.
  Handing over to a person when it cannot answer is the product's own
  logic to write.
- **The team works in a tool that can be connected through MCP.** A
  "where does this stand" action. The Python SDK's own example asks an
  agent with a project tracker connected to list one person's sprint tasks
  and summarise them.
- **Someone rewrites text for another audience.** A custom agent with
  instructions for that rewrite, with tools turned off. The Python SDK's
  own example turns a technical incident report into a short briefing for
  an executive.
- **People come back to the product.** An assistant that keeps what it
  learned about each person between conversations. The docs' own example
  is a support bot with one memory namespace per user, holding a file on
  the tone to write in.
- **People research a topic inside the product.** A research action that
  uses web search and shows its progress while it works.

"Not known yet" is the part that keeps this honest. Every idea has one.
The usual entries: how good the answers are on this team's real questions,
how long a run takes, and what it costs. Each one is settled by trying it,
not by arguing.

## Step 4. Offer to start

Ask if they want the first idea built. Then:

1. If they have not made a first call yet, load `cominty-api-onboarding`.
   Keys are created on the console's `/api-keys` page, and its Quickstart
   page (https://platform.cominty.ai/quickstart) walks through the call.
2. Suggest one real test before any building: send one question taken from
   the project through the first-call snippet. It is one snippet, and it
   tells them more than a proposal can. It calls the real API, so it
   spends credits.
3. To build it, load `cominty-add-agent-to-app` and follow it. If that
   skill is not installed:
   `npx skills add cominty/cmt-skills --skill cominty-add-agent-to-app`.

## When the answer is no

Say so when the project is not a fit. For example: it needs text streamed
token by token, or it has no server and the team will not add one, or
every idea you found leans on something from the "not documented" list.
"I did not find a good use here" is a useful answer.
