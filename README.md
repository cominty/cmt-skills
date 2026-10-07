# Cominty skills

Skills for coding agents that work with Cominty: a managed AI-agent API
and its developer console.

A skill is a folder with a `SKILL.md` file. It holds instructions that a
coding agent loads when a task calls for them. These skills use the open
Agent Skills format, so the same files work in Claude Code, Cursor, Codex,
OpenCode and the other agents that support it.

## Install

With the open [`skills`](https://www.npmjs.com/package/skills) CLI:

```bash
npx skills add cominty/cmt-skills
```

```bash
pnpx skills add cominty/cmt-skills
```

The CLI asks which skills to install and for which agents. To skip the
questions:

```bash
# List the skills without installing anything
npx skills add cominty/cmt-skills --list

# Install one skill
npx skills add cominty/cmt-skills --skill cominty-api-onboarding

# Install for a given tool
npx skills add cominty/cmt-skills -a claude-code

# One skill, for two tools, with no prompts
npx skills add cominty/cmt-skills --skill cominty-streaming -a cursor -a codex -y
```

Skills are installed into the current project by default. Add `-g` to
install them for your user instead.

## The skills

| Skill | Use it when |
|---|---|
| [`cominty-api-onboarding`](skills/cominty-api-onboarding/SKILL.md) | You have an account and want your code to get a first answer back: the key, the user id, the SDK, the first call, streaming, follow-ups. |
| [`cominty-add-agent-to-app`](skills/cominty-add-agent-to-app/SKILL.md) | You want a Cominty agent inside a project that already exists, with the API key kept on the server. Also for giving that agent a memory, or a cap on its work per message. |
| [`cominty-streaming`](skills/cominty-streaming/SKILL.md) | You are writing your own reader for the run stream, or debugging one. |
| [`cominty-troubleshooting`](skills/cominty-troubleshooting/SKILL.md) | Something fails: a 401, a 429, a stream that never ends, no answer, an agent that stops half way or does not remember. |
| [`cominty-console`](skills/cominty-console/SKILL.md) | You need to know where something is in the developer console. |
| [`cominty-ideas`](skills/cominty-ideas/SKILL.md) | You want to know what an agent could do in your project. |

Once a skill is installed, ask your agent in your own words, for example
"add Cominty to this app" or "my Cominty stream never ends". It loads the
skill that fits.

## For the Cominty team

`skills/internal/` holds skills for the team's own workflow. They are
marked `metadata.internal: true`, so the installer leaves them out of its
list unless `INSTALL_INTERNAL_SKILLS=1` is set:

```bash
INSTALL_INTERNAL_SKILLS=1 npx skills add cominty/cmt-skills --list
```

```bash
INSTALL_INTERNAL_SKILLS=1 npx skills add cominty/cmt-skills --skill cominty-thread-tag
```

| Skill | What it does |
|---|---|
| `cominty-pr-review-request` | Writes the "PR ready" message for the review queue. |
| `cominty-thread-tag` | Writes the head message of a thread, with its visibility tag. |

Hidden is not private. The flag only keeps a skill out of the installer's
list. A skill named with `--skill` installs without the variable, and
anyone who can read this repository can read these files.

## Layout

```
skills/
  <skill-name>/
    SKILL.md        the skill: frontmatter, then instructions
    references/     longer material, linked from SKILL.md
  internal/
    <skill-name>/
      SKILL.md
```

## Adding a skill

1. Create `skills/<name>/SKILL.md`. The folder name and the `name` in the
   frontmatter must be the same: lowercase letters, digits and hyphens.
2. Write the frontmatter: `name` and `description`, nothing else. The
   description says what the skill does and when to use it, with the
   phrases a person would actually say. An agent decides from the
   description alone whether to load the skill.
3. Keep it portable. No `allowed-tools`, no `context: fork`, no hooks.
4. Keep `SKILL.md` under about 200 lines. Put longer material in
   `references/*.md` and link it from `SKILL.md`.
5. Do not link from one skill into another. Each skill folder is installed
   on its own. Repeat the few lines you need, or name the other skill so
   the agent can load it.
6. State only what you can point to in public: the SDK source, the docs,
   the API reference. No invented endpoints, fields, limits or
   prices. Check every snippet against the SDK.
7. No secrets. Examples read `COMINTY_API_KEY` from the environment and use
   `user_...` as the user id.
8. For a team-only skill, put the folder under `skills/internal/` and add
   this to its frontmatter:

   ```yaml
   metadata:
     internal: true
   ```

9. Check that the installer finds it, from the root of this repository:

   ```bash
   npx skills add . --list
   INSTALL_INTERNAL_SKILLS=1 npx skills add . --list
   ```

## What the skills are based on

The public skills were written from the TypeScript SDK
([`@cominty-ai/sdk`](https://github.com/cominty/js-sdk), 0.1.0), the Python
SDK ([`cominty-sdk`](https://github.com/cominty/python-sdk), 0.4.0), the
documentation and API reference at https://docs.cominty.ai, and the team's
earlier onboarding notes. `max_steps` and memory namespaces were added
from the Python SDK 0.5.0 and the API reference. Their TypeScript names
are the ones planned for the release after 0.1.0: check them against that
release when it is out. When the API or an SDK changes,
update the skills in the same change.

## License

No license has been chosen for this repository yet.
# cmt-skills
