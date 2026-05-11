---
description: Clean up old savepoints under .claude/context-memory/. Removes files whose last-updated date is older than the given duration (default — 1 month).
argument-hint: "[duration]"
allowed-tools:
  - Read
  - Bash
---

Clean up stale savepoints under `<cwd>/.claude/context-memory/`. By default, removes any savepoint whose `last-updated` is older than **1 month** ago. Pass a natural-language duration to override the threshold.

Examples:

```
/cleanup                 # default — older than 1 month
/cleanup 7d              # older than 7 days
/cleanup 2 weeks         # older than 2 weeks
/cleanup t-3months       # older than 3 months (t- prefix is optional)
/cleanup 1 year          # older than 1 year
```

The command always shows a preview and asks for confirmation before deleting anything. Deletes are irreversible — there's no trash.

## Step 0 — Parse the duration argument

If no argument was given, set `DURATION` to "1 month".

Otherwise, strip an optional leading `t-` (or `T-`) prefix and parse the remainder as a natural-language duration. Accept these shapes (case-insensitive, with or without spaces between the number and the unit):

| Unit | Accepted forms |
| --- | --- |
| days | `d`, `day`, `days` |
| weeks | `w`, `wk`, `wks`, `week`, `weeks` |
| months | `m`, `mo`, `mos`, `month`, `months` |
| years | `y`, `yr`, `yrs`, `year`, `years` |

Examples that should all parse:
- `7d`, `7 days`, `7day`, `t-7d`
- `2w`, `2 weeks`, `2wks`
- `1m`, `1 month`, `3 months`, `t-3mo`
- `1y`, `1 year`

If the argument can't be confidently parsed into `(number, unit)`, stop and ask the user to clarify:

> "I couldn't parse `<arg>` as a duration. Try something like `7d`, `2 weeks`, or `1 month`."

Compute `THRESHOLD_DATE` = today minus the parsed duration. Use date arithmetic (e.g. `date -v -7d +%Y-%m-%d` on macOS, `date -d '7 days ago' +%Y-%m-%d` on GNU date) via the Bash tool. Today's date is available in the conversation context (`# currentDate`).

Hold:
- `DURATION_LABEL` — human-readable form to show the user (e.g. "1 month", "7 days")
- `THRESHOLD_DATE` — `YYYY-MM-DD`

## Step 1 — Inventory savepoints

Use the Bash tool to list savepoint files:

```bash
ls .claude/context-memory/*.md 2>/dev/null
```

**If the directory doesn't exist or no `.md` files are found**, tell the user:

> "No savepoints found in `<cwd>/.claude/context-memory/`. Nothing to clean up."

Then stop.

For each file, use the Read tool to read it and parse the YAML meta block at the top. Extract `subject` and `last-updated`. Bucket each file into one of three lists:

- **DELETE_CANDIDATES** — `last-updated` is a valid date AND is strictly older than `THRESHOLD_DATE`
- **KEEP** — `last-updated` is a valid date AND is on or after `THRESHOLD_DATE`
- **SKIPPED** — `last-updated` is missing, empty, or not a valid `YYYY-MM-DD` date

Never bucket a file into DELETE_CANDIDATES based on a missing or unparseable date. When in doubt, skip it.

## Step 2 — Preview

If `DELETE_CANDIDATES` is empty:

> "Nothing to clean up. No savepoints have a `last-updated` older than `<THRESHOLD_DATE>` (`<DURATION_LABEL>` ago).
>
> Kept: <N> file(s). Skipped (no valid `last-updated`): <M> file(s)."

Then stop.

Otherwise, show the user a structured preview:

> **Cleanup preview** — threshold: `<THRESHOLD_DATE>` (`<DURATION_LABEL>` ago)
>
> **Will delete (<N>):**
> - `<file>.md` — *<subject>* (last updated `<date>`)
> - `<file>.md` — *<subject>* (last updated `<date>`)
>
> **Will keep (<N>):**
> - `<file>.md` — *<subject>* (last updated `<date>`)
>
> **Skipped — no valid `last-updated` (<N>):**
> - `<file>.md` — *<subject or "(no subject)">*
>
> Proceed with deletion?
> - **yes** / **delete** — delete the files above
> - **no** / **cancel** — abort

Omit any section that's empty.

Wait for the user's response.

- **yes / delete** → proceed to Step 3
- **no / cancel / anything else** → stop. Do not delete anything.

## Step 3 — Delete

For each file in `DELETE_CANDIDATES`, delete it via the Bash tool. Quote paths to handle spaces:

```bash
rm -- ".claude/context-memory/<file>.md"
```

Prefer one `rm` call per file (or a single `rm` with all paths) over wildcards — wildcards risk matching files the preview didn't show.

## Step 4 — Confirm

Tell the user:

> "Cleanup complete. **<N> savepoint(s) deleted:**
>
> - `.claude/context-memory/<file>.md`
> - `.claude/context-memory/<file>.md`
>
> <M> kept, <K> skipped."

Omit the kept/skipped counts if both are zero.
