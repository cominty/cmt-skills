---
name: cominty-console
description: "Where things are in the Cominty developer console (platform.cominty.ai) and what each page is for: API keys, the user id, the Quickstart, Set up with AI, Agent Sessions, Skills, Documents, Integrations, Usage, Team, settings, plan and credits. For a developer or a non-developer on the team. Use when someone asks things like: where do I get an API key, where is my user id, how do I see what my calls cost, how do I give my agent our documents, where do I connect a tool, where do I see the sessions my code made, where do I upload a skill, who can see usage, where is the team page, where is the plan or the credits."
---

# The Cominty developer console

The console is at https://platform.cominty.ai. This skill knows which page
is for what. It does not know what the pages look like.

## Rules

- Say only what is in this file. If someone asks what is on a page, which
  button to press, or about a page that is not listed here, say you do not
  know and suggest they open the page. Do not describe layouts, menus,
  labels or buttons.
- Give the full address, for example
  `https://platform.cominty.ai/api-keys`. Not everyone asking is a
  developer.
- Two pages are for admins only. Say so before you send someone there.
- Never ask anyone to paste an API key into the conversation.

## The pages

Every address is a path on https://platform.cominty.ai.

| Address | Page | What it is for |
|---|---|---|
| `/` | Home | The home page. |
| `/quickstart` | Quickstart | The first API call, step by step, with your own agent id and user id written into the code. It shows your user id, with a copy button. |
| `/agent-setup` | Set up with AI | Builds a briefing for a coding agent. It may open as a step of the Quickstart. |
| `/api-keys` | API keys | Create and revoke keys. A key is shown once. |
| `/chats` | Agent Sessions | The sessions made through the API. |
| `/playground` | Playground | Its request settings include "Max steps" and "Memory namespace", and its Code tab shows the same call in TypeScript, Python and cURL. |
| `/skills` | Skills | Skills uploaded for the workspace's agents. |
| `/knowledge` | Documents | Sets of uploaded files that agents can search. |
| `/integrations` | Integrations | Connect tools through MCP. |
| `/dashboard` | Usage | Usage. Workspace admins only. |
| `/admin` | Team | The team. Admins only. |
| `/settings/general` | Settings, general | This skill knows the page exists and no more. |
| `/settings/account` | Settings, account | This skill knows the page exists and no more. |
| `/settings/organization` | Settings, organization | This skill knows the page exists and no more. |
| `/settings/plan` | Settings, plan | Plan and credits. |

## Common questions

**Where do I get an API key?**
On `/api-keys`. Create one there. It is shown once, so copy it when it
appears and keep it somewhere safe, such as a secret manager or your
server's environment. If a key is lost or leaked, create a new one and
revoke the old one on the same page.

For developers: the key goes in the `x-cominty-token` header. It never goes
in `Authorization: Bearer`, and it never goes into front-end code.

**Where is my user id?**
On the Quickstart page (`/quickstart`), with a copy button, and in the menu
behind your avatar. The id starts with `user_`. The API needs it next to the
key: it names the user every request acts on behalf of.

**How do I make my first call?**
Open `/quickstart`. It walks through the first call with your own agent id
and user id already in the code. If a coding agent is doing the setup,
`/agent-setup` builds a briefing for it. In this conversation, load the
`cominty-api-onboarding` skill.

**How do I see what my calls cost?**
The place to look is the Usage page, `/dashboard`. It is for workspace
admins only: someone who is not an admin has to ask one. The plan and the
credits are on `/settings/plan`. This skill does not know which figures
either page shows.

For developers: the API also reports a run's cost, in the `result` event
of its stream. The `cominty-streaming` skill has the fields.

**How do I give my agent our documents?**
On `/knowledge`, the Documents page. It holds sets of uploaded files that
agents can search.

For developers: the API has parameters that restrict retrieval for one
call (`sourceIds`, `documentIds`), and a `company_documents` tool that a
call can turn off. This skill does not know how those ids relate to the
sets on this page. The `cominty-api-onboarding` skill has the parameters.

**Where do I connect a tool to my agents?**
On `/integrations`. Tools are connected there through MCP.

**Where do I see the conversations my code made?**
On `/chats`, the Agent Sessions page. It lists the sessions made through
the API.

**Where do I upload a skill for my agents?**
On `/skills`. It holds the skills uploaded for the workspace's agents.

**Where is the team page?**
`/admin`. It is for admins only.

**Where is the plan, or the credits?**
`/settings/plan`. This skill knows nothing about invoices, payment methods
or prices.

**Where are the docs?**
https://docs.cominty.ai. For an agent, the docs also publish
https://docs.cominty.ai/llms.txt and https://docs.cominty.ai/llms-full.txt,
and answer as an MCP server at https://docs.cominty.ai/mcp.

## What this skill does not know

- What any page looks like, or what is on it beyond the table above.
- Any page that is not in the table.
- What the Usage and Team pages show, or what an admin can do there.
- Prices, plan limits, quotas and invoices.

When a question lands there, say so. Suggest they open the page, read the
docs, or ask a workspace admin. A wrong direction costs more than "I do
not know".
