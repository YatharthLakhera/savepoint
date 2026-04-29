---
description: Rewind through the session and split mixed context into multiple savepoints. Use when you forgot to /save along the way.
argument-hint: ""
allowed-tools:
  - Read
  - Write
  - Bash
---

Rewind through the entire current session and split mixed context into multiple savepoint files under `<cwd>/.claude/context-memory/`. Use this when you've worked on several different topics in one session and forgot to `/save` along the way — `/rewind` finds the natural boundaries and writes one savepoint per topic.

The model already has the full conversation in context, so this is pure analysis — no chat-history API needed.

## Step 0 — Inventory existing savepoints

Use the Bash tool:

```bash
ls .claude/context-memory/*.md 2>/dev/null
```

For each file found, use the Read tool to read it and parse the YAML meta block at the top to extract `subject` and `last-updated`.

Hold this inventory as `KNOWN_SAVEPOINTS` — a list of `{filename, subject, last-updated}`. If the directory doesn't exist or is empty, `KNOWN_SAVEPOINTS` is an empty list.

## Step 1 — Analyze the session

Reread the entire conversation in this session and identify distinct **work threads**. A thread is a coherent set of turns about one focus area — a feature, a bug, a refactor, a question.

Use these signals to find boundaries between threads:
- Explicit topic switches from the user ("ok now let's…", "moving on to…", "different topic:")
- Long stretches focused on a different file or module
- New goals or deliverables stated by the user
- Closure markers ("done with that", "shipped it") followed by something new

For each identified thread, produce:

- `subject` — one short sentence describing what this thread was about
- `suggested_filename` — kebab-case, derived from the subject, ending in `.md` (e.g. `auth-jwt-migration.md`, `payment-retry-bug.md`)
- `summary` — 2–3 sentences of what was accomplished in this thread
- `overlaps_with` — if any entry in `KNOWN_SAVEPOINTS` is about the same topic as this thread, set this to that filename. Else null.

Be honest about granularity:
- Don't propose 10 tiny shards — group small adjacent exchanges into one thread
- Don't bundle clearly different work just to keep file count low
- A reasonable session usually has 1–4 threads worth saving

## Step 2 — Single-thread fallback

If only **one** thread is identified, do NOT proceed with the multi-file flow. Tell the user:

> "This session looks like one cohesive topic: **[subject]**.
>
> `/rewind` is for splitting mixed context across multiple topics. For a single topic, just run `/save` (or `/save <name>` to use a specific filename)."

Stop.

## Step 3 — Present proposals

Show the user the proposed splits as a numbered list:

```
Found <N> distinct threads in this session:

1. <suggested_filename> — NEW
   Subject: <subject>
   Summary: <summary>

2. <suggested_filename> — ⚠️  overlaps with existing savepoint
   Subject: <subject>
   Summary: <summary>
   → would merge into: <KNOWN_SAVEPOINTS[i].filename> (existing subject: "<existing subject>", last updated <date>)

3. <suggested_filename> — NEW
   Subject: <subject>
   Summary: <summary>

What would you like to do?
- **accept** — write all proposals (merging where flagged)
- **pick <numbers>** — write only some (e.g. "pick 1,3")
- **edit** — rename one or more proposals before writing
- **cancel** — abort
```

Wait for the user's response.

- **accept** → proceed to Step 4 with all proposals
- **pick `<numbers>`** → filter proposals to the chosen subset, proceed
- **edit** → ask "Which numbers to rename, and what to?" Take the renames, auto-append `.md` to any name that doesn't end in it. Then re-show the updated list and ask the same question again (the user can then accept, pick, edit again, or cancel).
- **cancel** → stop. Do not write anything.

## Step 4 — Per-file handling

For each accepted proposal:

- If `overlaps_with` is set → use **merge** mode automatically. The user already saw the warning in Step 3 and chose to proceed.
- If the filename collides with a `KNOWN_SAVEPOINTS` entry but does NOT overlap (different subject, same name) → ask the standard prompt for that one file:

  > "Filename `<name>` already exists with a different subject (`<existing subject>`). What would you like to do for this file?
  > - **retain** — skip this proposal, keep existing
  > - **override** — replace existing with this thread's content
  > - **merge** — combine both into one file
  > - **rename** — pick a new filename for this proposal"

  Apply the choice. If `rename`, ask for the new name and re-check for collision.

- Else → fresh write.

## Step 5 — Write files

Run `mkdir -p .claude/context-memory` via the Bash tool to make sure the directory exists.

For each accepted proposal, compose the file content using the **same schema as `/save`** — but scoped to that thread only, not the whole conversation:

**Meta block (top of file):**

```yaml
---
subject: <subject from the proposal>
created: <YYYY-MM-DD>
last-updated: <YYYY-MM-DD>
---
```

- `created` — if merging into an existing file, preserve the original `created` value from that file. Else today.
- `last-updated` — always today.

**Body (after the meta block):**

```
# Session Memory — [project name]

**Written:** [today's date]
**Session goal:** [one sentence: what was this thread about?]

---

## Project Identity

[2–4 sentences: what is this project, who is the user, what is the end deliverable? Same across all files written in this /rewind run.]

## Current State

[Bullet list scoped to THIS thread: status of work done in this thread (✅ / 🔄 / ❌ / ⏳).]

## What Was Done This Session

[Bullet list of concrete actions FROM THIS THREAD only — files created, decisions made, problems solved. Be specific: file names, line numbers, command outputs that mattered.]

## Open Questions & Blockers

[Numbered list scoped to this thread.]

## Key Decisions Made

[Bullet list of non-obvious decisions FROM THIS THREAD and the reasoning.]

## Next Steps (in order)

[Numbered list: what should the next session pick up first for this thread?]

## Critical Files & Locations

[Files relevant to THIS thread.]

## Conventions & Warnings

[Gotchas relevant to THIS thread.]
```

Write the file in second person ("You are working on…", "You last left off at…").

**For merges:** intelligently combine the existing file's content with this thread's content. Keep older "key decisions" that are still relevant, update "current state" and "next steps", and append (don't replace) "what was done". The `subject` should reflect the union if the topics meaningfully extended.

Use the Write tool for each file.

## Step 6 — Confirm with summary

After all files are written, tell the user:

> "Rewind complete. **<N> savepoints written:**
>
> - `.claude/context-memory/<file1>` — *<subject1>* (new)
> - `.claude/context-memory/<file2>` — *<subject2>* (merged into existing)
> - `.claude/context-memory/<file3>` — *<subject3>* (new)
>
> Respawn any of them with `/respawn <name>`."

(Annotate each line with `(new)`, `(merged into existing)`, or `(overrode existing)` so the user knows what happened.)
