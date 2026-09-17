---
name: agent-memory
description: >-
  Manage a persistent local knowledge graph via Memory MCP.
  Use when ingesting meeting notes, recalling past decisions,
  or updating obsolete architecture facts.
license: Apache-2.0
user_invocable: true
model: inherit
allowed-tools: >-
  create_entities create_relations add_observations
  search_nodes read_graph open_nodes
  delete_relations delete_observations
---

# Agent Memory

You are equipped with a local, persistent knowledge graph via the Memory
MCP server (`@modelcontextprotocol/server-memory`), backed by
`data/memory.jsonl`. Use it to remember facts across sessions instead of
relying on conversation history.

## When to Use This Skill

- Ingesting notes, transcripts, or docs that contain decisions, ownership,
  or technical facts worth remembering.
- Recalling past decisions, who owns what, or what technology was chosen —
  especially at the start of a fresh session with no prior context.
- Updating or correcting a previously stored decision.

## Recall Phase

Before starting a non-trivial task, check the graph first:

1. Prefer `search_nodes` for targeted queries (e.g. a specific project,
   person, or technology).
2. Use `read_graph` when the scope of what you need is unclear, or a
   targeted search comes back empty.
3. Only fall back to source documents (e.g. re-reading meeting notes) if
   the graph has no relevant entities yet.

## Ingestion & Storage Phase

Parse the source content (meeting notes, docs, transcripts) and record it
in three steps:

1. **`create_entities`** — one entity per person, project, system, or
   technology mentioned.
2. **`create_relations`** — directed, verb-like relations connecting those
   entities (e.g. `Sarah` → `approved` → `Prisma`).
3. **`add_observations`** — factual, atomic properties attached to an
   entity (e.g. "Lead Engineer", "Will provision AWS RDS PostgreSQL").

### Entity naming conventions

- **Projects:** `Project_<Name>` (e.g. `Project_Helios`)
- **People:** first name only (e.g. `Sarah`)
- **Technologies:** canonical product name (e.g. `PostgreSQL`, `Prisma`,
  `MongoDB`)

Keep names stable across ingestions so relations and observations attach to
the same entity instead of creating duplicates.

## Pruning Phase

Before recording an update that reverses or invalidates a prior decision:

1. Remove the stale relation(s) with `delete_relations`.
2. Remove the stale observation(s) with `delete_observations`.
3. Then proceed with the normal ingestion steps above for the new decision.

Do **not** delete an entity itself unless the user has explicitly said it
no longer exists or is obsolete — prune relations and observations, keep
the entity.

## Dependencies

- Memory MCP server (`memory`) configured and connected — see
  [README.md](../../README.md#mcp-host-setup) for per-host setup.
