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

## Intent Routing

Match the user's request to the correct skill:

| When the user asks about… | Use skill |
|---------------------------|-----------|
| Ingest notes/docs into memory, store decisions in the knowledge graph | `/agent-memory` |
| Recall past decisions, who owns what, what was decided | `/agent-memory` |
| Update or correct a stored decision | `/agent-memory` |

If the request doesn't clearly involve persistent memory, handle it
normally without invoking the skill.

## MCP Servers

- **memory** — local knowledge graph (entities, relations, observations)
  backed by `data/memory.jsonl`. Must be configured in your host's MCP
  settings — see [README.md](README.md#mcp-host-setup). No credentials or
  env vars beyond `MEMORY_FILE_PATH` are required.

## Global Rules

1. **Skill-first** — always route memory operations through `/agent-memory`
   rather than calling Memory MCP tools directly.
2. **Recall before ingest** — check the graph (`search_nodes` /
   `read_graph`) before assuming information isn't already known.
3. **Fresh-session correctness** — when asked something the graph should
   know, query the graph; do not ask the user to re-paste prior context.
4. **No silent data loss** — prune stale relations/observations explicitly
   via the skill's pruning phase; never delete an entity without explicit
   confirmation that it's obsolete.
