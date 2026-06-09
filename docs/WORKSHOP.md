# Claude Code Workshop — Hands-on Guide

This guide turns the small Tasks API in this repo into a slightly bigger task
tracker. You will **not** be doing the coding by hand — Claude Code does it.
The point of each stage is to learn a Claude Code feature by using it to make
the next change easier, safer, or repeatable.

Each stage builds on the previous one, so do them in order.

> **One running task, told two ways.** As an app, you are growing a Tasks API
> into a tiny team tracker (Tasks → Projects → Comments). As a Claude Code
> user, you are building a small "toolkit" (memory → rules → a skill → a
> reviewer subagent → hooks → a shareable plugin) that makes that growth fast
> and consistent. Every feature is introduced exactly when the app gives you a
> reason to want it.

Official docs to keep open: the
[feature overview](https://code.claude.com/docs/en/features-overview) is the
single best map of how these pieces relate.

---

## Before you start

```bash
npm install
npm run dev     # confirm http://localhost:3000/health returns {"status":"ok"}
npm test        # confirm the 7 tests pass
```

Then open this repo in Claude Code (`claude` in the project root, or the VS Code
/ JetBrains extension). Stay in the project root so Claude picks up `CLAUDE.md`.

---

## Stage 0 — Warm up: drive the app with Claude Code (10 min)

Goal: get comfortable letting Claude read and run the project before you change
anything about its setup.

Try these prompts and watch what Claude reads:

- "Give me a tour of this codebase and how a request flows from route to store."
- "Run the tests and the linter and tell me if anything is wrong."
- "Add a `priority` field (`low | medium | high`, default `medium`) to tasks,
  with validation and a test for it."

Notice the friction: Claude has to re-derive the project conventions every time,
and it might guess at how you run tests or where files go. That friction is the
motivation for everything below.

> If you did the `priority` change, keep it — later stages assume tasks may have
> extra fields, and it is a fine example for the reviewer subagent to inspect.

---

## Stage 1 — CLAUDE.md (project memory)

**Concept.** `CLAUDE.md` is persistent context Claude loads at the start of
**every** session. It is the place for "always true / always do" facts: the
stack, how to run things, the folder conventions, and the definition of done.
Keep it tight — aim for **under ~200 lines**; anything longer belongs in rules
or skills (Stage 2 and 3). Docs:
[Memory & CLAUDE.md](https://code.claude.com/docs/en/memory).

**Why now.** In Stage 0 Claude kept rediscovering basics. Write them down once.

**Task.** Replace the stub `CLAUDE.md` with a real one. A good prompt:

> "Read this repo and draft a CLAUDE.md. Cover: the stack and Node version; how
> to run, test, lint, and format; the `src/resources/<name>/` pattern and the
> order routes → controller → service → validation → store; the response
> conventions (`{ data }`, `{ errors }`, `{ error }`); and our definition of
> done (validation + a passing test for every change). Keep it under 150 lines."

Then refine by hand — you own this file.

**Reference shape (facilitators):**

```md
# Tasks API

Small Express REST API. In-memory store, no database.

## Commands
- Dev: `npm run dev`  ·  Test: `npm test`  ·  Lint: `npm run lint`  ·  Format: `npm run format`
- Node 20+ required.

## Architecture
- One folder per resource under `src/resources/<plural>/`.
- Layering, in order: routes → controller → service → validation → store.
- Register new routers in `src/app.js` under `/api/<plural>`.

## Conventions
- Success responses use `{ "data": ... }`. Validation errors use `{ "errors": [...] }` with 400. Missing records use `{ "error": ... }` with 404.
- Every write endpoint validates its payload.
- Every change ships with a test. `npm test` and `npm run lint` must be green before "done".

## Never
- Never commit secrets or edit `package-lock.json` by hand.
```

**Done when:** opening a fresh Claude session and asking "how do I run the
tests and add a resource?" gets answered correctly without Claude re-reading
half the repo.

---

## Stage 2 — Rules (`.claude/rules/`)

**Concept.** Rules are like CLAUDE.md, but they can be **scoped to file paths**
via `paths` frontmatter, so they only load when Claude touches matching files.
This keeps CLAUDE.md short while still giving detailed, local guidance. Docs:
[Memory → rules](https://code.claude.com/docs/en/memory).

**Why now.** Your CLAUDE.md wants to grow — you keep wanting to add detail about
how resources and tests should look. Move that detail into rules so it loads
only when relevant.

**Task.** Create two rules and slim down CLAUDE.md accordingly:

- `.claude/rules/resources.md` (scoped to `src/resources/**`) — the exact
  resource pattern, naming, and response envelope.
- `.claude/rules/tests.md` (scoped to `tests/**`) — that you use `node:test` +
  `supertest`, and that each resource needs CRUD coverage plus a validation
  failure case.

**Reference shape (facilitators):**

```md
---
description: Conventions for API resources
paths:
  - src/resources/**
---
# Resource conventions
- Mirror `src/resources/tasks/` exactly: store, validation, service, controller, routes.
- Validation returns `{ value, errors }`; controllers turn non-empty `errors` into a 400.
- Reuse `createStore()` from `src/store/createStore.js`; do not invent a new store.
```

**Done when:** asking Claude to edit a file under `tests/` makes it follow the
test rule, while editing `src/server.js` does not pull that rule into context.

---

## Stage 3 — Skills: a `/new-resource` workflow

**Concept.** A skill is a markdown file (`SKILL.md`) of knowledge or a workflow
that Claude loads on demand — automatically when relevant, or when you type
`/<name>`. Skills are the right home for a **repeatable procedure**. Docs:
[Skills](https://code.claude.com/docs/en/skills).

**Why now.** Adding a resource is five files in a fixed pattern plus a router
registration plus tests. That is a playbook you do not want to re-explain every
time. Capture it once.

**Task.**

1. Create `.claude/skills/new-resource/SKILL.md` describing the full scaffolding
   procedure (see below).
2. **Use it** to build the **Projects** resource: a project has `name`
   (required string) and `status` (`active | archived`, default `active`).
   Mount it at `/api/projects`. Run `/new-resource` and feed it those fields.
3. Confirm `npm test` and `npm run lint` are green.

**Reference SKILL.md (facilitators):**

```md
---
name: new-resource
description: Scaffold a complete CRUD resource (store, validation, service, controller, routes, tests) following this project's resource pattern. Use when adding a new entity to the API.
---
# New resource

Inputs: the singular and plural name, and the list of fields with types/rules.

Steps:
1. Create `src/resources/<plural>/` mirroring `src/resources/tasks/`:
   - `<plural>.store.js` exporting `<plural>Store` from `createStore()`.
   - `<plural>.validation.js` exporting `validate<Singular>(body, { partial })`.
   - `<plural>.service.js` with list/get/create/update/remove.
   - `<plural>.controller.js` with the five handlers and the `{ data }` envelope.
   - `<plural>.routes.js` wiring GET / POST / GET :id / PUT :id / DELETE :id.
2. Register the router in `src/app.js` under `/api/<plural>`.
3. Add `tests/<plural>.test.js`: CRUD lifecycle + one validation-failure case.
4. Run `npm test` and `npm run lint`; fix anything red.
Match the existing style and conventions exactly.
```

**Done when:** `/new-resource` produces a working `/api/projects` with passing
tests, and the diff looks like the `tasks` resource with the names swapped.

> Talking point: compare this to just pasting the same instructions every time.
> The skill is versioned, shared, and improves as you edit it.

---

## Stage 4 — Subagents: a `code-reviewer`

**Concept.** A subagent runs in its **own isolated context** and returns only a
summary to your main conversation. It is ideal for work that would otherwise
flood your session — reading lots of files, reviewing a diff — when you only
care about the conclusion. Define one in `.claude/agents/<name>.md` with
frontmatter (name, description, optional `tools`, `model`, `skills`). Docs:
[Subagents](https://code.claude.com/docs/en/sub-agents).

**Why now.** You just generated a whole resource. You want it reviewed against
your conventions without dumping every file into your main context.

**Task.**

1. Create `.claude/agents/code-reviewer.md` (see below).
2. Ask Claude to "use the code-reviewer subagent to review the projects
   resource against our conventions." Notice that your main context stays
   clean — only the findings come back.
3. Fix anything it flags.

**Reference agent (facilitators):**

```md
---
name: code-reviewer
description: Reviews a code change against this project's conventions and returns a concise findings list. Use after implementing a feature, before committing.
tools: Read, Grep, Glob, Bash
---
You review changes for a small Express API. Inspect the current diff (`git diff`)
and check:
- The resource layering: routes → controller → service → validation → store.
- Response envelope: `{ data }` on success, `{ errors }` (400), `{ error }` (404).
- Every write endpoint validates input; tests cover CRUD + a failure case.
- No leftover console noise, secrets, or unused exports.
Return only: a one-line summary, then findings grouped as blocker / nice-to-have,
each with file:line and a suggested fix. Do not modify files.
```

**Done when:** the reviewer returns a short, useful report and your main session
transcript did **not** fill up with the contents of every reviewed file.

> Stretch: add a `test-writer` subagent that, given a resource, writes the test
> file in isolation. Talking point: subagents vs. agent teams (independent
> sessions that message each other) — see the docs comparison.

---

## Stage 5 — Hooks: enforce, don't just ask

**Concept.** A hook fires on a lifecycle event (e.g. `PostToolUse`,
`PreToolUse`, `SessionStart`, `PreCompact`) and runs a command, HTTP request,
prompt, or subagent. Unlike a CLAUDE.md instruction (a request Claude *usually*
follows), a hook is **deterministic** — it always runs, and a `PreToolUse` hook
can *block* an action. Hooks are configured in `settings.json`
(`.claude/settings.json` for project scope). Docs:
[Hooks guide](https://code.claude.com/docs/en/hooks-guide) and the
[hooks reference](https://code.claude.com/docs/en/hooks) for the exact event
names and the input/output (blocking) contract.

**Why now.** Your CLAUDE.md says "lint must be green" and "never edit
package-lock.json by hand," but those are only requests. Make the important ones
guaranteed.

**Task.**

1. Add a **`PostToolUse`** hook that runs the linter after Claude edits files,
   so problems surface immediately and the output flows back to Claude.
2. Add a **`PreToolUse`** hook that **blocks** edits to protected files
   (`package-lock.json`, anything matching `.env`). Read the hooks reference for
   exactly how a `PreToolUse` hook signals "block" (it inspects the tool input
   and returns a deny decision / non-zero exit) — wiring this yourself is part
   of the exercise.
3. Test both: ask Claude to introduce a lint error (watch the post-edit hook
   report it) and to edit `.env` (watch the pre-edit hook refuse).

**Reference starting point (facilitators):** `.claude/settings.json`

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npm run lint" }]
      }
    ]
  }
}
```

For the blocking `PreToolUse` guard, point a hook at a small script
(`.claude/hooks/protect.js`) that reads the tool input, checks the target path
against a deny-list, and exits with the blocking signal documented in the hooks
reference. Confirm the exact field names against the docs rather than guessing —
the contract is the lesson here.

**Done when:** a lint error introduced by Claude is reported automatically, and
an attempt to modify `.env` is stopped before it happens.

> Key talking point: "never do X" in a prompt is a request; a `PreToolUse` hook
> is enforcement. Put guardrails in hooks, not prose.

---

## Stage 6 — Plugins: package it for the team

**Concept.** A plugin bundles skills, subagents, hooks, and MCP servers into one
installable unit, distributed through a **marketplace** (a Git repo or local
folder). Plugin skills are namespaced (e.g. `/workshop:new-resource`) so they
never collide with local ones. Docs:
[Plugins](https://code.claude.com/docs/en/plugins) and
[Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces).

**Why now.** Everyone in the room just built the *same* skill, subagent, and
hooks by hand. That should be a one-line install for the next teammate.

**Task.**

1. Move the `new-resource` skill, the `code-reviewer` agent, and the lint hook
   into a plugin folder (structure below).
2. Register a local marketplace and install the plugin into a *fresh* checkout
   of this repo.
3. Verify the namespaced command works (e.g. `/workshop:new-resource`).

**Reference structure (facilitators):**

```
workshop-plugin/
  .claude-plugin/
    plugin.json          # name, version, description
  skills/new-resource/SKILL.md
  agents/code-reviewer.md
  hooks/hooks.json        # the PostToolUse lint hook
```

Then add it as a marketplace and install it with the `/plugin` commands
documented in the marketplace guide (`/plugin marketplace add <path>`, then
`/plugin install`). Check the docs for the exact `plugin.json` fields and
command syntax — they are short but version-specific.

**Done when:** a teammate with none of the `.claude/` files can install the
plugin and immediately use `/workshop:new-resource` and the reviewer.

---

## Stage 7 — Compaction: surviving a long task

**Concept.** Claude has a finite [context window](https://code.claude.com/docs/en/context-window).
As a session grows, you can **compact** it — summarize the history so far and
keep going — with `/compact` (optionally `/compact <instructions>` to steer what
is preserved), or start fresh with `/clear`. Claude also auto-compacts when the
window gets full. You can run something right before this happens with a
`PreCompact` hook (see the hooks reference).

**Why now.** You are about to do a genuinely long task, and you will watch the
context fill up. This is the moment to practice managing it instead of letting a
session degrade.

**Task (a deliberately big one).** Add a **Comments** resource nested under
tasks:

- `GET/POST /api/tasks/:taskId/comments`, `GET/PUT/DELETE
  /api/tasks/:taskId/comments/:id`.
- A comment has `body` (required string) and `author` (optional string), and
  must reference an existing task (404 if the task does not exist).
- Use the `/new-resource` skill as a starting point, then adapt it for nesting.
- Write the tests, run the reviewer subagent, then update `README.md` and
  `CLAUDE.md` to mention the new endpoints.

Partway through, when the context is large:

1. Run `/compact preserve the comments API design decisions and remaining TODOs`
   and continue.
2. Discuss: what got summarized away? What would you have lost with a plain
   `/clear`? When is each appropriate?

**Done when:** the Comments API works with passing tests, and the team can
articulate when to compact, when to clear, and what a `PreCompact` hook is for.

---

## Stretch goals

- **MCP:** connect an external service (e.g. a real Postgres) via an
  [MCP server](https://code.claude.com/docs/en/mcp) and migrate the store off
  in-memory. Pair it with a skill documenting the schema.
- **Agent teams:** have parallel reviewers (security, tests, style) work the
  same diff — see [agent teams](https://code.claude.com/docs/en/agent-teams).
- **More hooks:** a `SessionStart` hook that prints the current test status; a
  `Stop` hook that runs the full suite when Claude finishes.

---

## Facilitator notes

**Suggested timing (half day, ~3.5 hrs):**

| Stage | Topic | Time |
| ----- | ----- | ---- |
| 0 | Warm-up / drive the app | 15 min |
| 1 | CLAUDE.md | 20 min |
| 2 | Rules | 15 min |
| 3 | Skills + Projects | 30 min |
| 4 | Subagents | 25 min |
| 5 | Hooks | 35 min |
| 6 | Plugins | 30 min |
| 7 | Compaction + Comments | 35 min |
| — | Buffer / stretch / Q&A | 25 min |

**Format tips**

- **Pairs, not solo.** One drives Claude Code, one watches the context window
  and the docs. Swap each stage.
- **Checkpoint branches.** Tag a git commit at the end of each stage
  (`git tag stage-3`) so anyone who falls behind can reset and rejoin.
- **Reset between groups.** The in-memory store resets on restart, so demos are
  repeatable — no cleanup needed.
- **Make them feel the pain first.** Each stage works better if you do the
  *next* task the slow way once, then introduce the feature that fixes it. The
  warm-up in Stage 0 is built for this.
- **Diff review as the heartbeat.** After every Claude change, read the diff
  together. The workshop is as much about reviewing AI output as producing it.
- **Common gotcha:** if a skill or subagent does not show up, it is almost
  always a filename/location issue (`.claude/skills/<name>/SKILL.md`,
  `.claude/agents/<name>.md`) or a description Claude cannot match — good
  teaching moments about how discovery works.

**Discussion prompts to close on**

- Which belongs in CLAUDE.md vs a rule vs a skill vs a hook? (Always-true →
  CLAUDE.md; path-specific → rule; on-demand workflow → skill; must-always-run →
  hook.)
- Where would a prompt instruction have failed where a hook succeeded?
- What would you package into your *team's* plugin on Monday?
