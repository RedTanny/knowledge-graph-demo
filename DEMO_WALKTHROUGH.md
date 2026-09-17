# Demo Walkthrough — Knowledge Graph Memory

A two-session presenter script showing an agent that persists knowledge from
meeting notes into a local graph, then recalls it in a completely fresh
session — no re-pasted context required.

**Validation path:** install this pack from the **private hub** (Lola or
equivalent) before running the script below. A raw git clone with manual MCP
setup also works for local development — see [README.md](README.md).

## Prerequisites

- Pack installed from the private hub, or cloned locally with `mcp.json`
  configured for your host (see [README.md](README.md#mcp-host-setup)).
- Memory MCP server connected in the IDE (`memory` in your MCP server list).
- Node.js 18+ available on `PATH` (needed for `npx` to fetch
  `@modelcontextprotocol/server-memory`).

## Session 1 — Knowledge Ingestion

1. Confirm the Memory MCP server is connected in your IDE.
2. Start a **new** agent chat.
3. Prompt:

   > Read `data/sample-meeting-notes.md` and ingest all technical decisions,
   > ownership, and tasks into persistent memory.

4. **Observe:** the agent invokes the `agent-memory` skill, which calls
   `create_entities`, `create_relations`, and `add_observations` against the
   Memory MCP server.
5. **Verify:** `data/memory.jsonl` now exists and is populated. Its shape
   should match `data/memory.jsonl.example` (see [§6.6 of the implementation
   plan](plan/IMPLEMENTATION_PLAN.md#66-expected-graph-shape-validation-reference)
   for the full expected entity/relation list).

## Session 2 — Fresh Context Recall (the "Wow" phase)

1. **Close** the chat from Session 1, or start a completely new agent thread
   with no prior conversation history.
2. Prompt:

   > What database transition was decided for Project Helios, and who owns
   > the infrastructure setup?

3. **Observe:** the agent invokes `search_nodes` or `open_nodes` against the
   Memory MCP server — **not** re-reading `sample-meeting-notes.md`.
4. **Expected answer:** PostgreSQL migration (from MongoDB); David owns the
   infrastructure setup (provisioning AWS RDS PostgreSQL).

This is the "wow" moment: the agent has zero conversation context from
Session 1, yet answers correctly because the knowledge graph persisted to
disk.

## Session 3 (stretch, deferred)

Not part of v1. The idea: introduce a follow-up note that reverses a
decision (e.g. Project Helios reverts to MongoDB), have the agent prune the
stale graph entries with `delete_relations` / `delete_observations`, then
re-query to confirm the graph reflects the update.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `data/memory.jsonl` never appears | `MEMORY_FILE_PATH` resolved relative to an unexpected cwd | Use an absolute path in `mcp.json` env, or verify your host's working directory (see README) |
| Session 2 reads the markdown file instead of querying memory | Agent skipped the `agent-memory` skill | Re-prompt asking explicitly to "check memory" or "recall from the knowledge graph" |
| MCP server not listed in IDE | `mcp.json` not loaded for this host | Copy/symlink into the host-specific location per [README.md](README.md#mcp-host-setup) and reload/restart |
