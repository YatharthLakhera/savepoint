---
description: Load session context from a memory file under .claude/context-memory/ into the current session so Claude is fully up to speed.
argument-hint: "[name]"
allowed-tools:
  - Read
  - Bash
---

Load session context from a markdown file under `<cwd>/.claude/context-memory/` into the current session so Claude is fully up to speed without re-reading the whole conversation history.

## Step 0 — Resolve filename and target path

If the user passed an argument, use it as the filename. If it does not end in `.md`, append `.md`.

If no argument was given, the filename is `last-memory.md`.

Set:
- `FILENAME` = resolved filename
- `TARGET_PATH` = `<cwd>/.claude/context-memory/<FILENAME>`

## Step 1 — Read the file

Use the Read tool to read TARGET_PATH.

**If the file does not exist:**

Use the Bash tool to list any other context files in the directory:

```bash
ls .claude/context-memory/*.md 2>/dev/null
```

If files exist there, read each one's YAML meta block (`subject`, `last-updated`) so you can show a useful list, then tell the user:

> "No `<FILENAME>` found at `<TARGET_PATH>`.
>
> Available context files in this repo:
> - `<file-1>.md` — *<subject>* (last updated <date>)
> - `<file-2>.md` — *<subject>* (last updated <date>)
>
> Run `/mem-load <name>` with one of those, or `/mem-commit` first to save the current session's context."

If `.claude/context-memory/` does not exist or is empty, simpler message:

> "No context memory found in this repo. Run `/mem-commit` first to save the current session's context, or `/mem-load <name>` if you stored it under a different name."

Then stop.

## Step 2 — Internalize the content

Parse the YAML meta block at the top first — `subject`, `created`, `last-updated`. Use these in the confirmation reply.

Read every section carefully. Treat this file as ground truth for:
- What the project is and what the goal is
- What has already been done (do not redo it)
- What is blocked and why
- What the conventions and constraints are
- What should happen next

Cross-reference the memory against the actual project files if any paths or claims seem stale. If you spot a conflict between what the memory says and what the files show, note it.

## Step 3 — Confirm to the user

Reply with a structured confirmation that proves you've loaded the context. Use this format:

> **Memory loaded** from `<TARGET_PATH>`
>
> **Subject:** [subject from meta block]
> **Project:** [project name and one-line description from memory body]
> **Last updated:** [last-updated from meta block]
>
> **Current state:**
> [Reproduce the current-state bullet list from the file, unchanged]
>
> **Next steps:**
> [Reproduce the next-steps list from the file, unchanged]
>
> **Stale items noticed (if any):**
> [List any conflicts between memory and actual files — e.g., "memory says X file exists but it's not there", "memory says step Y is in progress but it looks complete based on the files". If nothing stale, omit this section.]
>
> Ready to continue. What would you like to work on?
