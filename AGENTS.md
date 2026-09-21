# Knowledge Graph Demo Pack

You are an assistant equipped with a persistent local knowledge graph,
backed by the Memory MCP server (`@modelcontextprotocol/server-memory`).
The graph survives across sessions, so decisions, ownership, and technical
facts don't need to be re-pasted into every conversation.

## Skill-First Rule

ALWAYS use the `agent-memory` skill for reading from or writing to the
knowledge graph. Do NOT call Memory MCP tools (`create_entities`,
`search_nodes`, etc.) directly outside the skill — the skill enforces
correct phase ordering (recall → ingest → prune) and entity naming
conventions.

To invoke the skill, use the Skill tool with the skill name: `/agent-memory`.

A second skill, `/memory-mcp-setup`, wires the Memory MCP server itself into
your host's MCP config — run it once after installing this pack (from the
hub via any client, or by cloning the repo) before `/agent-memory` will have
any tools to call.

## Intent Routing

Match the user's request to the correct skill:

| When the user asks about… | Use skill |
|---------------------------|-----------|
| Configure Memory MCP, set up the knowledge graph server, first-time hub install, Memory MCP not connected | `/memory-mcp-setup` |
| Ingest notes/docs into memory, store decisions in the knowledge graph — **and Memory MCP tools are available** | `/agent-memory` |
| Recall past decisions, who owns what, what was decided — **and Memory MCP tools are available** | `/agent-memory` |
| Update or correct a stored decision — **and Memory MCP tools are available** | `/agent-memory` |

If the request doesn't clearly involve persistent memory, handle it
normally without invoking either skill.

## Missing Memory MCP — Bounce, Don't Improvise

If the user asks for anything in the `/agent-memory` rows above but Memory
MCP tools (`create_entities`, `search_nodes`, etc.) are **not** available:

- **Stop.** Do not fabricate a graph, keep facts only in conversation
  memory, or fall back to manually re-reading source documents as a
  substitute for the knowledge graph.
- Tell the user Memory MCP isn't connected and have them run
  `/memory-mcp-setup` first.
- Do not treat "manually copy `mcp.json`" instructions in the README as the
  primary path — that's a fallback for when the setup skill itself can't
  find the pack; `/memory-mcp-setup` is the default route.

## MCP Servers

- **memory** — local knowledge graph (entities, relations, observations)
  backed by `data/memory.jsonl`. Must be configured in your host's MCP
  settings — run `/memory-mcp-setup` to wire it up (see
  [README.md](README.md#mcp-host-setup) for the manual fallback). No
  credentials or env vars beyond `MEMORY_FILE_PATH` are required.

## Global Rules

1. **Skill-first** — always route memory operations through `/agent-memory`
   rather than calling Memory MCP tools directly; route setup through
   `/memory-mcp-setup` rather than hand-editing MCP config.
2. **Bounce, don't improvise** — if Memory MCP tools are missing, stop and
   point the user at `/memory-mcp-setup` instead of working around the gap.
3. **Recall before ingest** — check the graph (`search_nodes` /
   `read_graph`) before assuming information isn't already known.
4. **Fresh-session correctness** — when asked something the graph should
   know, query the graph; do not ask the user to re-paste prior context.
5. **No silent data loss** — prune stale relations/observations explicitly
   via the skill's pruning phase; never delete an entity without explicit
   confirmation that it's obsolete.
