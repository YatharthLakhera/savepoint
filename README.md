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

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed.
- **For Option 1 (plugin install):** a version of Claude Code that supports the [plugin marketplace system](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces). Type `/plugin` in any Claude Code session — if you see `marketplace` and `install` subcommands, you're good. If not, use Option 2 or 3.
- **For Options 2 and 3:** any version of Claude Code with custom slash command support.

## Install

### Option 1 — As a Claude Code plugin (recommended)

In any Claude Code session, run these two slash commands:

```
/plugin marketplace add YatharthLakhera/savepoint
/plugin install savepoint@savepoint
```

The repo is its own one-plugin marketplace, so `/plugin marketplace add` points directly at it. After installing, `/savepoint` and `/respawn` are available immediately.

**Verify it worked:**

Type `/` in your Claude Code prompt. You should see `savepoint` and `respawn` in the slash-command menu. You can also run:

```
/plugin list
```

…and you'll see `savepoint@savepoint` listed as enabled. If the new commands don't show up after install, fully close and reopen your Claude Code session — some versions only refresh the command menu on session start.

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

### Example savepoint

To make this concrete, here's what a real savepoint looks like in practice — saved partway through an auth migration:

<details>
<summary>Click to expand <code>last-memory.md</code></summary>

````markdown
---
subject: Migrating session-cookie auth to JWT-based auth on the API gateway
created: 2026-04-15
last-updated: 2026-04-28
---

# Session Memory — Acme API gateway

**Written:** 2026-04-28
**Session goal:** Wire up the JWT verifier behind a feature flag and confirm the dual-mode middleware doesn't break existing browser clients.

---

## Project Identity

You are working on Acme's API gateway, which sits in front of all internal services and currently authenticates users via opaque session cookies. The migration moves to short-lived JWTs (RS256) so mobile and CLI clients can stop tunneling through a fake browser flow.

## Current State

- ✅ `JWTVerifier` class with JWKS caching (`gateway/auth/jwt.py:42`)
- ✅ 14 unit tests covering expiry, signature, audience, and clock-skew edges
- 🔄 Dual-mode middleware (cookie + JWT) wired up but not tested under load
- ❌ Mobile SDK update blocked on Apple cert renewal
- ⏳ Cookie deprecation timeline not announced to teams yet

## What Was Done This Session

- Added the `JWTVerifier` class and finished its tests
- Updated `auth_middleware.py:88` to fall through to JWT when no cookie is present
- Drafted `docs/auth-migration-rollback.md` covering the staged rollback path

## Open Questions & Blockers

1. **JWKS endpoint URL** — auth service team hasn't confirmed the prod URL. Slack thread with @sarah.k stalled Friday. Blocking deploy.
2. **Cache TTL** — currently 10min; need a load test to confirm this is right under burst traffic.

## Key Decisions Made

- RS256 over HS256: services don't share a secret-distribution channel, public-key verification is simpler operationally.
- Dual-mode (not strict JWT-only) during rollout to avoid a hard cutover for browser clients.

## Next Steps (in order)

1. Ping @sarah.k again Monday for the JWKS prod URL.
2. Run the load test against the dual-mode middleware in staging.
3. Send the deprecation timeline draft to `#eng-platform` for review.

## Critical Files & Locations

- `gateway/auth/jwt.py` — the new verifier
- `gateway/auth/middleware.py` — dual-mode integration
- `docs/auth-migration-rollback.md` — rollback playbook

## Conventions & Warnings

- **Do NOT log the raw JWT** — it contains PII in the `email` claim.
- The JWKS cache must use the existing Redis instance (not in-process) — gateway runs multiple pods and needs a shared cache.
````

</details>

When you `/respawn` from this file in a fresh session, Claude reads it, cross-checks it against the actual repo (e.g. does `gateway/auth/jwt.py` still exist?), flags any drift, and confirms the next steps before continuing.

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
