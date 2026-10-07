---
name: cominty-pr-review-request
description: >
  Write the "PR ready" message for the team's PR review queue for a frontend
  PR in this repo, with Priority and Review Level. Use when a frontend PR is
  opened or ready for review, when the user asks to request a review or post
  a PR to the review channel, or when taking a PR for review.
metadata:
  internal: true
---

# PR review request (frontend)

For frontend PRs only: the cominty app, the platform app and the shared
packages. When a PR is ready, post it to the review channel with two signals:
Priority and Review Level. The PR owner chooses both.

## Scope: one branch

Describe only what changed on one git branch: the branch given as argument,
else the current branch. Nothing else from the conversation, other branches or
uncommitted work belongs in the message.

```
git log --oneline main..<branch>
git diff --stat main...<branch>
```

Read the commits and the diff before writing; open a file when the stat alone
does not say what a change does. If the branch has no commits ahead of `main`,
say so and stop. Mention uncommitted or unpushed work to the user, outside the
message.

## Priority: when should it be reviewed?

| Signal | Meaning |
|---|---|
| `:large_green_circle: Normal` | Next review window. Default. |
| `:large_orange_circle: High` | Should be reviewed today. |
| `:red_circle: Urgent` | ASAP: blocking, hotfix or deadline. A context switch is justified. |

## Review level: how deep?

| Signal | Meaning |
|---|---|
| `:white_check_mark: Approval` | Low risk, the owner has the context. Sanity check and approval. |
| `:eyes: Review` | A real second pair of eyes on the implementation. Default. |
| `:microscope: Deep Review` | Understand and challenge the design. Complex or sensitive changes. |

Sensitive frontend changes always get `:eyes: Review` at least, and
`:microscope: Deep Review` when in doubt:

- Auth and access: Clerk setup, route guards, server functions, `start.ts`.
- Security: `security-headers.ts` (CSP), the widget sandbox, anything that
  renders agent or user HTML.
- Secrets and config: `env.ts`, anything server-only.
- The chat stream: `use-new-chat-send`, `chat-stream-runtime.ts`.
- Dependency upgrades of the framework (TanStack, React, Vite, Nitro).

Copy, styling, i18n keys, docs and tests are usually `:white_check_mark: Approval`.

## Template

```
:red_circle: [INTERNAL][FRONTEND][PR] <PR title>
PR ready :eyes:
:memo: <one or two plain sentences: what changes for the people using the product>
:link: <PR link>
:stopwatch: Priority: :large_green_circle: Normal
:mag_right: Review: :eyes: Review
:male-technologist::skin-tone-2: Reviewer: Anyone / @<someone>
```

The first line is the visibility tag from the `cominty-thread-tag` skill; a PR is
`[INTERNAL]` unless the user says otherwise.

## Language: write for non-technical readers

The review channel is read by the whole team, not only engineers: customer
success managers, marketers, sales. The title and the `:memo:` line must make
sense to someone who has never opened the code.

- Say what a user of the product will see or be able to do, not how it was
  built. No file names, package names, version numbers or jargon (i18n,
  locale, deps, refactor, endpoint).
- If the PR title is technical, rewrite it in plain words for the message;
  the PR itself keeps its title.
- A change with no visible effect is described by its reason: "a security
  update needed before we can release".
- The one-line reason for the proposed priority and level is for the PR
  owner, outside the message, and can be technical.

## Steps

1. Get the PR link and title for the branch (`gh pr view <branch> --json
   url,title`). If there is no PR yet, say so and leave `<PR link>` to fill.
2. Propose a priority and a level from the branch diff: size, which app or package
   it touches, and whether it hits the sensitive list above. Say why in one
   line. The user decides.
3. Reviewer is `Anyone` unless the user names someone.
4. Show the message. Post it only when the user asks, and to the channel they
   name; otherwise leave it as a draft or as text to paste.

## Taking a PR

Reply in the thread with `👀 Taking this` so nobody else picks it up.

## Notes

- The review window is capped at about 45 minutes a day. Only
  `:red_circle: Urgent` bypasses it, and it can go in the standup thread.
- Do not mark a PR Urgent to jump the queue; it has to be blocking.
