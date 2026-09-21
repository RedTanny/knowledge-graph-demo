---
name: memory-mcp-setup
description: >-
  Add the Memory MCP server for the knowledge-graph demo. Merges the memory
  entry into your host MCP config, sets an absolute MEMORY_FILE_PATH, and
  stages demo data under workspace data/. Use after installing the pack from
  the hub (any client) or when Memory MCP is missing.
license: Apache-2.0
user_invocable: true
model: inherit
allowed-tools:
---

# Memory MCP Setup

Wire the Memory MCP server (`@modelcontextprotocol/server-memory`) into the
current host's MCP configuration, and stage the demo notes so
`DEMO_WALKTHROUGH.md` works regardless of how this pack was installed
(private-hub install via any client, or a plain git clone).

## Prerequisites

- Node.js 18+ on `PATH` (`npx` fetches `@modelcontextprotocol/server-memory`
  on first start).
- Network access to the npm registry.
- Write access to the target MCP config file.

## When to Use This Skill

- Right after installing this pack from the hub (Claude Code's self-hosted
  marketplace, Lola, or any other client) — this is "Session 0" before the
  demo walkthrough.
- Whenever `/agent-memory` bounces you here because Memory MCP tools
  (`create_entities`, `search_nodes`, etc.) aren't available.
- After a plugin/module update, to re-sync the MCP entry and demo data.

This skill is **installer-agnostic**. It does not assume Lola, Claude Code,
or a git clone — it works out what's on disk and adapts. It is safe to
re-run: it never deletes `memory.jsonl`, and it asks before overwriting
anything that already exists and differs.

## Design Rule: Pack Markers, Not Installer Names

Never hard-code an installer's cache layout as the *definition* of success.
A directory `<dir>` **is** `$PLUGIN_ROOT` (the pack) only if it contains:

- `mcp.json` (the pack's Memory MCP entry), **and**
- `plugin.json` whose `"name"` is `knowledge-graph-demo`, when that file
  exists.

Everything about Claude plugin caches, Lola module caches, etc. below is a
**search list of hints** for finding a directory that matches this rule —
never the rule itself. If a specific cache layout ever changes, that only
means adding another glob to the search list; it does not change what
`$PLUGIN_ROOT` means.

**Anti-pattern:** treating `.cursor/skills/` or `.claude/skills/` (host
skill-flatten directories) as `$PLUGIN_ROOT`. Those directories never have
`mcp.json` or `data/` — they only ever hold `SKILL.md` files.

## Workflow

1. Detect the current host (Cursor or Claude Code only — stop otherwise).
2. Resolve `$WORKSPACE_ROOT` — the opened project folder. **Never** default
   to git toplevel.
3. If a second supported host also looks configured in this folder, ask
   "this host" vs "both".
4. Ask project-level vs user-level; default to project-level, warn hard on
   user-level.
5. Resolve `$MCP_FILE` (or files, if "both") from `$WORKSPACE_ROOT`.
6. Resolve `$PLUGIN_ROOT` by pack markers (search caches as hints; skeleton
   as last resort).
7. Load the `memory` server entry: vendor `mcp.json` → pack-root `mcp.json`
   → inline skeleton.
8. Rewrite `MEMORY_FILE_PATH` to `$WORKSPACE_ROOT/data/memory.jsonl`
   (absolute), then merge into `$MCP_FILE` without touching other servers.
9. Stage `data/sample-meeting-notes.md` (and optionally
   `memory.jsonl.example`) under `$WORKSPACE_ROOT/data/`. Never create,
   truncate, or delete `memory.jsonl`.
10. Print the post-install summary and next steps.

---

## Step 1 — Detect the host

State which agentic tool you are (you already know this from your system
context — e.g. "I am Cursor", "I am Claude Code"). This skill supports:

| Host | Vendor namespace dir | Project-level `$MCP_FILE` | User-level `$MCP_FILE` |
|------|----------------------|----------------------------|--------------------------|
| Claude Code | `com.anthropic.claude-code` | `$WORKSPACE_ROOT/.mcp.json` | `~/.claude/.mcp.json` |
| Cursor | `com.cursor.editor` | `$WORKSPACE_ROOT/.cursor/mcp.json` | `~/.cursor/mcp.json` |

If the current host is neither: tell the user v1 only merges MCP config for
Cursor and Claude Code, print `$PLUGIN_ROOT/mcp.json` (if you can find it)
so they can merge it by hand, and **stop**. Do not guess a `.mcp.json` path
for an unlisted host.

## Step 2 — Resolve `$WORKSPACE_ROOT` (opened project, not git toplevel)

`$WORKSPACE_ROOT` is the folder the user actually opened in their host —
**not** necessarily the git repository root. The pack's `mcp.json` uses a
cwd-relative `MEMORY_FILE_PATH`; getting this wrong silently breaks the
demo (setup "succeeds", Session 1 writes the graph somewhere the user never
opened).

Resolution order:

1. Prefer the **host-reported** workspace/project folder (Cursor's
   workspace root, Claude Code's project directory).
2. If this `SKILL.md` lives under a host skill-flatten directory
   (`.cursor/skills/...` or `.claude/skills/...`), walk **up** from the
   current working directory until you hit the directory that *contains*
   that flatten dir. That directory is the opened project after a
   hub-install-then-flatten.
3. If (1) and (2) disagree, prefer whichever one will actually be read as
   project-level MCP config (i.e. it contains, or will contain, `.cursor/`
   or `.mcp.json`). Tell the user if they differed.
4. Only if neither resolves: if `git rev-parse --show-toplevel` matches the
   pack markers above (i.e. the git repo itself **is** the pack — the clone
   case), use it. Otherwise use `$PWD`.
5. **Never** use git toplevel when it is a **parent** of the folder found
   in (1)/(2)/(4) above — a parent repo must not receive the demo's `data/`
   or MCP file.

Print the resolved `$WORKSPACE_ROOT` (absolute path) to the user before
continuing.

## Step 3 — Two hosts in one project?

Check whether *both* a Cursor-flavored setup (`.cursor/` or
`.cursor/skills/`) and a Claude-Code-flavored setup (`.claude/skills/`, or a
Claude project plugin install) look present under `$WORKSPACE_ROOT`. If so,
ask the user:

- **This host only** (default — the host you detected in Step 1), or
- **Both Cursor and Claude Code** — merge into both hosts' MCP files in one
  pass.

## Step 4 — Project vs. user level

Ask the user to choose:

- **Project-level (default, recommended)** — only this workspace gets the
  server, with a graph pinned to `$WORKSPACE_ROOT/data/memory.jsonl`.
- **User-level** — every workspace opened with this host loads the same
  `memory` entry. **Warn:** `MEMORY_FILE_PATH` in a user-level config is a
  single fixed path, so *every* project opened with that host would share
  one graph file unless the user edits it per-project afterward. Recommend
  project-level unless the user explicitly wants one global graph.

## Step 5 — Resolve `$MCP_FILE`

Using the table in Step 1, the level from Step 4, and `$WORKSPACE_ROOT`
from Step 2, resolve the absolute path(s) to write to. If Step 3 selected
"both hosts", resolve one `$MCP_FILE` per host.

## Step 6 — Resolve `$PLUGIN_ROOT`

Apply this rule in order; **first match wins**:

1. **Skill path, if pack-shaped** — from this `SKILL.md`'s own path, if it
   matches `.../skills/memory-mcp-setup/SKILL.md`, let `candidate` be that
   `skills/` directory's parent. If `candidate` is **not** named `.cursor`
   or `.claude` (host flatten dirs) and it satisfies the pack-marker rule
   above → `$PLUGIN_ROOT = candidate`.
2. **Opened project is the pack** — if `$WORKSPACE_ROOT` itself satisfies
   the pack-marker rule (the clone case: the user opened this repo
   directly) → `$PLUGIN_ROOT = $WORKSPACE_ROOT`.
3. **Search known caches** as hints (existence checks only; skip any root
   that doesn't exist). A hit is any directory matching the pack-marker
   rule. Prefer the shortest matching path whose `plugin.json` name is
   `knowledge-graph-demo`:

   | Client | Search roots (hints, not the rule) |
   |--------|--------------------------------------|
   | Claude Code plugin cache | `~/.claude/plugins/cache/*/*/` and `~/.claude/plugins/cache/*/*/*/` (marketplace/plugin/version layers); also `$WORKSPACE_ROOT/.claude/plugins/` if present |
   | Lola | `$WORKSPACE_ROOT/.lola/modules/**/plugins/community/knowledge-graph-demo/` (nested) **and** `$WORKSPACE_ROOT/.lola/modules/knowledge-graph-demo/` (flat); repeat both shapes under `"${LOLA_HOME:-$HOME/.lola}"/modules/` |
   | Hub clone opened as workspace | already covered by step 2 above |

4. **Inline skeleton** — if nothing matched: use the skeleton in Step 7,
   print every root you searched, and tell the user to copy
   `data/sample-meeting-notes.md` from wherever they installed the pack
   manually.

## Step 7 — Load the `memory` entry

Look for the `memory` entry to merge, in this order:

1. `$PLUGIN_ROOT/com.<vendor>/mcp.json` — the namespace dir matching the
   host from Step 1 (e.g. `com.cursor.editor`, `com.anthropic.claude-code`).
2. `$PLUGIN_ROOT/mcp.json` — the pack-root fallback.
3. If `$PLUGIN_ROOT` wasn't found or neither file exists, use this
   skeleton:

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "env": {
        "MEMORY_FILE_PATH": "./data/memory.jsonl"
      }
    }
  }
}
```

Treat whichever source is found as read-only reference — copy the entire
`memory` object (all of `command`, `args`, `env`) rather than reconstructing
it field-by-field, so you don't accidentally drop something. This server is
**stdio**, not HTTP — do not add `url` or `headers`, and do not set `cwd` to
`$PLUGIN_ROOT` (that would resolve a relative `MEMORY_FILE_PATH` against the
plugin cache instead of the workspace).

## Step 8 — Rewrite `MEMORY_FILE_PATH` and merge

1. Set `env.MEMORY_FILE_PATH` on the loaded `memory` object to the
   **absolute** path `$WORKSPACE_ROOT/data/memory.jsonl`. Do not leave it as
   the relative `./data/memory.jsonl` — that resolves against the MCP
   server's process cwd, which is host-dependent and not guaranteed to be
   `$WORKSPACE_ROOT`.
2. Create `$WORKSPACE_ROOT/data/` first if it doesn't exist.
3. Read `$MCP_FILE` if it exists (create it as `{"mcpServers": {}}` if not).
4. If `$MCP_FILE` already has a top-level `memory` entry: show the user the
   old vs. new `env.MEMORY_FILE_PATH` (and any other differences) and ask
   before overwriting. Otherwise just add it.
5. Merge — **never** remove or clobber any other server already present in
   `mcpServers`.
6. Validate the result parses as JSON before writing. Write it back to
   `$MCP_FILE`.
7. If Step 3 selected "both hosts", repeat Steps 5–8 for the second host's
   `$MCP_FILE`.

Do **not** point `MEMORY_FILE_PATH` at `$PLUGIN_ROOT/data/memory.jsonl` —
plugin/module caches are deleted on update (both Claude plugin cache and
Lola module cache), which would silently destroy the graph.

## Step 9 — Stage demo data

1. Ensure `$WORKSPACE_ROOT/data/` exists (already done in Step 8 if you
   followed it in order).
2. If `$PLUGIN_ROOT` was found, copy `$PLUGIN_ROOT/data/sample-meeting-notes.md`
   to `$WORKSPACE_ROOT/data/sample-meeting-notes.md`:
   - Missing at destination → copy.
   - Exists and byte-identical → no-op.
   - Exists and differs → ask before overwriting.
3. Optionally copy `memory.jsonl.example` the same way (skip-if-present).
4. **Never** create, truncate, or delete `$WORKSPACE_ROOT/data/memory.jsonl`
   itself — that file is the live graph, written only by the Memory MCP
   server during Session 1 of the demo.

If `$PLUGIN_ROOT` was not found, or the copy fails, print the searched
paths (or the failed source path) and ask the user to copy
`sample-meeting-notes.md` in manually before running the demo.

## Step 10 — Post-install summary

Report back to the user, filling in the resolved absolute paths (adapt if
"both hosts" was chosen — repeat the MCP file / notes lines per host):

```text
Memory MCP server added.

Workspace:  <absolute $WORKSPACE_ROOT>
MCP file:   <absolute $MCP_FILE>
Storage:    MEMORY_FILE_PATH → <absolute $WORKSPACE_ROOT/data/memory.jsonl>
Notes:      <absolute $WORKSPACE_ROOT/data/sample-meeting-notes.md>
Pack:       <absolute $PLUGIN_ROOT, or "skeleton (pack not found)">

Prerequisites: Node.js 18+ on PATH, and network access to the npm registry
(npx fetches @modelcontextprotocol/server-memory on first start).

1. Reload MCP servers (or restart the host).
2. If the host shows an Enable / Trust prompt for a new server, approve
   `memory`.
3. Confirm `memory` shows as connected in the host's MCP server list — not
   only that the JSON file exists.
4. Next: Session 1 in DEMO_WALKTHROUGH.md.
```

If a second supported host is also present in this folder and the user
chose "this host only" in Step 3, add:

```text
If you also use the other host in this folder, run /memory-mcp-setup there
too (or choose "both hosts" next time you're asked).
```

## Dependencies

- Read/write access to the target MCP config file(s) (project- or
  user-level, per the user's choice in Step 4).
- Read access to `$PLUGIN_ROOT` (if found), to locate `com.<vendor>/mcp.json`,
  pack-root `mcp.json`, and `data/sample-meeting-notes.md`.

## Notes

- This skill is **not** self-destructing — keep it around so it can be
  re-run after a plugin/module update (e.g. to re-sync `data/` or repair a
  merged entry).
- Never guess a target MCP file path for a host not in the Step 1 table —
  say so explicitly and stop, rather than silently writing to a made-up
  location.
- If `/agent-memory` reports Memory MCP tools are unavailable, that's the
  signal to run this skill — see `AGENTS.md`.
