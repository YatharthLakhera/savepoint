---
description: Commit the current session's context to a memory file under .claude/context-memory/ so a future session can resume exactly where this one left off.
argument-hint: "[name]"
allowed-tools:
  - Read
  - Write
  - Bash
---

Commit the current session's context to a markdown file under `<cwd>/.claude/context-memory/` so a future session can resume exactly where this one left off.

Multiple Claude windows can work on the same repo with different focuses (auth, payments, infra, etc.). Use the optional `[name]` argument to keep each focus in its own file instead of overwriting the default.

## Step 0 — Resolve filename and target path

If the user passed an argument, use it as the filename. If it does not end in `.md`, append `.md`.

If no argument was given, the filename is `last-memory.md`.

Set:
- `FILENAME` = resolved filename (e.g. `auth-redesign.md`)
- `TARGET_PATH` = `<cwd>/.claude/context-memory/<FILENAME>`
- `IS_DEFAULT` = `true` if no argument was given, else `false`

## Step 1 — Self-assessment

Before touching any file, honestly answer this question to yourself:

> "If this chat closed right now and someone opened a brand-new session with only the project files and CLAUDE.md — what context would be **missing** that I currently hold in my head?"

Think across these dimensions:
- What is the current state of each active deliverable?
- What decisions were made in this session and why?
- What is blocked, incomplete, or has an open question?
- What were the last actions taken?
- What should the next session do first?
- Are there any conventions, constraints, or warnings that are easy to forget?
- **What is this session actually about?** Distill it into one sentence — you'll use it as the `subject` meta field below.

If the answer is "nothing is missing — the codebase and CLAUDE.md capture everything", tell the user that briefly and ask if they still want to write the file.

## Step 2 — Subject-alignment check (only if `IS_DEFAULT`)

This step runs **only when no argument was given** (i.e., we're about to write to the default `last-memory.md`). If the user explicitly passed a name, skip this entire step and go to Step 3.

Use the Read tool to check if TARGET_PATH exists.

**If TARGET_PATH does not exist:** skip to Step 3.

**If TARGET_PATH exists:**
- Read the file and parse the YAML meta block at the top to get the existing `subject`
- Compare it against the current session's focus (the one-sentence summary you produced in Step 1)
- If the topics are clearly the same → skip to Step 3
- If the topics are clearly different → do not write yet. Tell the user:

  > "The current `last-memory.md` is about:
  > **[existing subject]**
  >
  > This session looks like it's about something different:
  > **[current session focus]**
  >
  > Mashing them together would lose detail from both. I'd suggest a new context file. Three suggestions:
  > - `<suggestion-1>.md`
  > - `<suggestion-2>.md`
  > - `<suggestion-3>.md`
  >
  > What would you like to do?
  > - **new** — pick one of the suggestions, or reply with your own name
  > - **merge** — proceed with `last-memory.md` anyway and combine both topics
  > - **override** — replace `last-memory.md` entirely with this session's context
  > - **cancel** — abort"

  The three suggestions should be short, kebab-case, and derived from the current session's focus (e.g. `auth-redesign`, `jwt-migration`, `session-cleanup` for an auth refactor).

  Wait for the user's response.
  - If `new`: take the chosen suggestion (or custom name), apply the `.md` rule from Step 0, update FILENAME and TARGET_PATH, then go to Step 3.
  - If `merge`: keep TARGET_PATH as-is and treat it as a `merge` decision in Step 3 (skip the question there).
  - If `override`: keep TARGET_PATH as-is and treat it as an `override` decision in Step 3 (skip the question there).
  - If `cancel`: stop. Do not write anything.

## Step 3 — Existing-file handling

If a decision was already made in Step 2 (`merge` or `override`), use that and proceed to Step 4.

Otherwise, use the Read tool to check if TARGET_PATH exists.

**If it exists:**
- Read the full file
- Write a 3–5 sentence summary of what it contains (subject, current state, when it was last updated)
- Ask the user:
  > "A context file already exists at `<TARGET_PATH>`. Here's what it covers: [summary]
  >
  > What would you like to do?
  > - **retain** — keep it as-is (abort)
  > - **override** — replace it entirely with fresh context
  > - **merge** — combine the existing context with new context from this session"

  Wait for the user's response before continuing.

**If it does not exist:** proceed directly to Step 4.

## Step 4 — Compose the file

Every context file starts with a YAML meta block, then the session memory body.

**Meta block (always at the very top of the file):**

```yaml
---
subject: <one-line description of what this context is about>
created: <YYYY-MM-DD>
last-updated: <YYYY-MM-DD>
---
```

- `subject` — one short sentence. Same one you produced in Step 1.
- `created` — if the file already existed, **preserve the original `created` value** from the existing meta. If it did not exist, use today's date.
- `last-updated` — always today's date.

**Body (after the meta block):**

```
# Session Memory — [project name]

**Written:** [today's date]
**Session goal:** [one sentence: what were we trying to accomplish in this session?]

---

## Project Identity

[2–4 sentences: what is this project, who is the client/user, what is the end deliverable?]

## Current State

[Bullet list: one line per deliverable or work stream, marking it as ✅ done / 🔄 in progress / ❌ blocked / ⏳ not started]

## What Was Done This Session

[Bullet list of concrete actions taken — files created, decisions made, tests run, problems solved. Be specific: file names, line numbers, command outputs that mattered.]

## Open Questions & Blockers

[Numbered list. Each item: what is blocked, why, and what input or action would unblock it. Tie to specific files or gaps where relevant.]

## Key Decisions Made

[Bullet list of non-obvious decisions and the reasoning. Include things that would be easy to re-debate in a new session without this context.]

## Next Steps (in order)

[Numbered list: exactly what the next session should do first, second, third. Be specific enough that no re-discovery is needed.]

## Critical Files & Locations

[Table or bullet list: the most important files to know about — not everything, just the ones that would take time to rediscover. Path + one-line purpose.]

## Conventions & Warnings

[Any non-obvious constraints, gotchas, or patterns that are easy to get wrong.]
```

Write the file in second person ("You are working on…", "You last left off at…") so the next session reads it as instructions to itself.

**If the decision is `merge`:** intelligently combine the existing file's content with new content. Keep older "key decisions" that are still relevant, update "current state" and "next steps" to reflect the new session, and append to "what was done" rather than replacing it. The meta block's `subject` should describe the union of both topics if they're being merged.

## Step 5 — Ensure directory exists, then write

Run `mkdir -p .claude/context-memory` via the Bash tool to make sure the directory exists.

Then use the Write tool to write the composed content to TARGET_PATH.

## Step 6 — Confirm

Tell the user:

> "Context committed to `.claude/context-memory/<FILENAME>`.
>
> Subject: *<subject>*
>
> To reload in a future session: `/mem-load`<if FILENAME is not `last-memory.md`: ` <name-without-extension>`>"
