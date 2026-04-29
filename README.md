# savepoint

Savepoints for [Claude Code](https://docs.claude.com/en/docs/claude-code) sessions.

Two slash commands — `/savepoint` and `/respawn` — that let you end one session, walk away, and resume in a fresh session as if you never closed the window. Like a savepoint in a game: drop one before you quit, respawn exactly where you left off.

## The problem

Long Claude Code sessions accumulate a lot of context that doesn't live in the codebase: decisions you made, things you ruled out, what's half-finished, what to do next. When the session ends, that context is gone. The next session has to rediscover everything from the files, and a lot of it can't be rediscovered at all.

`savepoint` fixes this with human-readable markdown files under `.claude/context-memory/` in your repo. At the end of a session you run `/savepoint` and Claude writes everything important into a file. At the start of the next session you run `/respawn` and Claude reads it back.

If you have multiple Claude windows working on different topics in the same repo (auth in one, payments in another), give each its own savepoint with a name argument — they won't overwrite each other.

## The commands

| Command | What it does |
| --- | --- |
| `/savepoint [name]` | Self-assesses what context is in-the-head but not in the files, then writes a structured savepoint. Defaults to `last-memory.md`. Pass a name (e.g. `auth-redesign`) to use a separate file. If you don't pass a name and the existing default file is about a different topic, you'll be offered three suggested names for a new file. |
| `/respawn [name]` | Reads the savepoint, internalizes it, cross-checks it against the actual files for staleness, and confirms current state + next steps. If the file isn't found, lists the other savepoints available in this repo. |

All files live at `<cwd>/.claude/context-memory/<name>.md`. The directory is auto-created on first save. The `.md` extension is auto-appended if you leave it off.

## Install

### Option 1 — As a Claude Code plugin (recommended)

In any Claude Code session, run these two slash commands:

```
/plugin marketplace add YatharthLakhera/savepoint
/plugin install savepoint@savepoint
```

The repo is its own one-plugin marketplace, so `/plugin marketplace add` points directly at it. After installing, `/savepoint` and `/respawn` are available immediately.

To uninstall: `/plugin uninstall savepoint@savepoint`. To pull updates after a new release: `/plugin marketplace update savepoint`.

### Option 2 — Manual copy

If you'd rather drop the commands directly into your config without going through the plugin system:

```bash
git clone https://github.com/YatharthLakhera/savepoint.git
cp savepoint/commands/*.md ~/.claude/commands/
```

Open any project, run `/savepoint` or `/respawn`.

### Option 3 — Per-project install

Scope the commands to a single repo (so they only show up in that project):

```bash
mkdir -p .claude/commands
cp /path/to/savepoint/commands/*.md .claude/commands/
```

Add `.claude/context-memory/` to your `.gitignore` if you don't want to commit session memory, or *do* commit it if you want the team to share context.

## Usage

### Single context (default)

End of a working session:

```
/savepoint
```

Writes to `./.claude/context-memory/last-memory.md`.

Start of the next session, in the same project directory:

```
/respawn
```

Reads it back, flags anything stale against the current files, and tells you current state and next steps.

### Multiple parallel contexts

When you have two or more Claude windows on the same repo working on different things, give each its own savepoint:

```
/savepoint auth-redesign
/savepoint payment-flow
```

Each writes to its own file under `.claude/context-memory/`. To respawn:

```
/respawn auth-redesign
/respawn payment-flow
```

The `.md` extension is optional — `/savepoint auth-redesign` and `/savepoint auth-redesign.md` are equivalent.

### Subject-mismatch detection

If you run `/savepoint` with no name and the existing `last-memory.md` is about a different topic than this session, the command won't silently merge. It'll show you the existing subject vs. the current session's subject and offer three suggested names for a new file, plus `merge` / `override` / `cancel` options.

This only fires when no name is given. If you pass a name explicitly, you're being intentional and the command just writes.

## What gets stored

Every savepoint starts with a YAML meta block:

```yaml
---
subject: One-line description of what this savepoint is about
created: 2026-04-28
last-updated: 2026-04-28
---
```

`subject` is what the mismatch check reads to decide whether the current session aligns with the file.

After the meta block, the body has these sections:

- **Project Identity** — what this project is, who it's for, the end deliverable
- **Current State** — per-deliverable status (✅ / 🔄 / ❌ / ⏳)
- **What Was Done This Session** — concrete actions, files, decisions
- **Open Questions & Blockers** — what's stuck and what would unblock it
- **Key Decisions Made** — non-obvious calls and the reasoning
- **Next Steps (in order)** — exactly what the next session should do first
- **Critical Files & Locations** — the files that would take time to rediscover
- **Conventions & Warnings** — gotchas that are easy to get wrong

The body is written in second person ("You are working on…", "You last left off at…") so the next session reads it as instructions to itself.

## Why a file, not auto-memory?

Claude Code already has built-in memory mechanisms. `savepoint` is complementary, not a replacement. The file approach gives you:

- **Project-scoped** — lives next to the code it describes
- **Multi-context aware** — keep parallel sessions on different topics in the same repo without overwriting each other
- **Version-controllable** — commit it, diff it, share it with teammates
- **Human-readable** — you can open it yourself and see exactly what the next session will see
- **Portable** — works the same across machines, accounts, and clients
- **Inspectable** — no surprise about what was remembered or forgotten

## Contributing

PRs welcome. The two command files in `commands/` are the entire surface area — keep them simple, keep them readable.

## License

[MIT](./LICENSE)
