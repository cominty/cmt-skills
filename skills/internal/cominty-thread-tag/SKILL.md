---
name: cominty-thread-tag
description: >
  Write the head message of a Slack thread about frontend work, with the
  mandatory visibility tag (PUBLIC / RESTRICTED / INTERNAL). Use when the user
  wants to open, start, post or draft a thread about the frontend changes on
  the current git branch or a given branch, or asks to tag a thread.
metadata:
  internal: true
---

# Thread visibility tag (frontend)

For threads about frontend work only: the cominty app, the platform app and
the shared packages. Every new thread must open with one visibility tag, as
the very first thing in the head message. The coloured dot makes the status
readable from the channel list without opening the thread.

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

## Language: write for non-technical readers

The thread is read by the whole team, not only engineers: customer success
managers, marketers, sales. Write so that someone who has never opened the
code understands what changed and what it means for them.

- Say what a user of the product will see or be able to do, not how it was
  built. "The app is now available in Spanish", not "added `es.json` and
  registered the locale".
- No file names, package names, function names, version numbers or commit
  hashes in the body. The branch name on the last line is the only exception.
- No jargon: avoid words like i18n, locale, deps, lockfile, refactor, aria,
  endpoint, PR. Use the everyday word (translation, language, update, review),
  or explain the term in a few words when there is no everyday one.
- Lead with the change that matters most to customers. Internal changes with
  no visible effect (tooling, cleanup, upgrades) get one short line at the
  end, with the reason in plain words ("a security update needed before we
  can release"), or are left out.
- Short sentences, a few bullets at most. Say what is still missing or not
  included when a reader could assume otherwise.
- Requests to engineers ("needs a review") stay, in one plain line, and say
  who is asked.

## Tags

| Tag | Meaning |
|---|---|
| `:large_green_circle: [PUBLIC]` | Can be shared with anyone outside, no restriction. |
| `:large_yellow_circle: [RESTRICTED]` | Can only be shared with a specific, pre-approved external organization for that occasion. |
| `:red_circle: [INTERNAL]` | Default. Stays inside the team. |

## Format

```
:red_circle: [INTERNAL][FRONTEND][TAG2] ThreadTitle
```

The visibility tag comes first, then `[FRONTEND]`, then one optional tag for
the area (`[STUDIO]`, `[CHAT]`, `[PLATFORM]`, `[UI]`, `[DEPS]`), then the
title. The body follows on the next lines.

## Steps

1. Pick the visibility. Use what the user said. If they did not say, use
   `[INTERNAL]` and tell them so. Never pick `[PUBLIC]` or `[RESTRICTED]` on
   your own; for `[RESTRICTED]`, name the approved organization in the body.
2. Add `[FRONTEND]`, the area tag that fits, and a short title in plain words.
3. Write the body from the branch's commits and diff, in the plain language
   described above: what changed for the people using the product, where
   they will see it (the Cominty app, the platform), and what is needed from
   others (a decision, a proofread, a review). End with the branch name.
4. Show the message. Post it only when the user asks, and to the channel they
   name; otherwise leave it as a draft or as text to paste.

## Rules

- Never edit the tag on an existing thread. If the status of the information
  changes, close the topic there and open a new thread with the correct tag.
- A thread without a tag is treated as `[INTERNAL]`.
- A PR review request is a thread too: it gets a tag (see `cominty-pr-review-request`).
