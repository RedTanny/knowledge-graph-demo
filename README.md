# Knowledge Graph Demo

A hands-on demo of a persistent local knowledge graph for AI agents. Meeting
notes go in; a queryable graph comes out; a **completely fresh** agent
session can still answer correctly by querying the graph instead of relying
on conversation history.

**Persona:** Any AI-assisted developer or knowledge worker who wants an
agent to remember decisions across sessions.

## Overview

This pack ships:

- **`agent-memory`** — a skill governing recall, ingest, and prune
  operations against a local knowledge graph.
- **Memory MCP** integration (`@modelcontextprotocol/server-memory`) —
  stores the graph as JSONL on disk, no database or cloud service required.
- **A demo payload** — sample Project Helios meeting notes and a two-session
  walkthrough that shows the "wow" moment: a fresh session recalling facts
  it was never told directly.

## Quick Start

### Prerequisites

- An MCP-capable agentic host (Cursor, Claude Code, etc.)
- Node.js 18+ (used by `npx` to fetch the Memory MCP server on first run)

### Installation (private hub / Lola)

```bash
lola install -f knowledge-graph-demo
```

### Installation (local clone, for development)

```bash
git clone https://github.com/RedTanny/knowledge-graph-demo.git
cd knowledge-graph-demo
```

Then configure the Memory MCP server for your host — see
[MCP host setup](#mcp-host-setup) below.

## MCP Host Setup

The pack ships a golden-source `mcp.json` at the repo root:

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

`mcp.json` itself is **not** auto-loaded by any host — copy or symlink its
`memory` entry into your host's own MCP config:

| Host | Config path | Notes |
|------|-------------|-------|
| Cursor | `.cursor/mcp.json` | Project-level; copy the `memory` entry in, don't overwrite other servers |
| Claude Code | `.mcp.json` | Same |
| GitHub Copilot | VS Code MCP settings | See Copilot docs for the MCP settings location |

After adding the entry, restart the host (or reload MCP servers) so the
`memory` server is picked up.

### Troubleshooting `MEMORY_FILE_PATH`

`./data/memory.jsonl` is resolved **relative to the process working
directory** the MCP server is launched from, which is host-dependent — not
always the repo root. If `data/memory.jsonl` never appears after Session 1
of the demo, switch to an absolute path in your host's MCP config, e.g.:

```json
"env": { "MEMORY_FILE_PATH": "/absolute/path/to/knowledge-graph-demo/data/memory.jsonl" }
```

## Skills

### `agent-memory` — Persistent Knowledge Graph Management

Manages a local knowledge graph via Memory MCP: recall before non-trivial
tasks, ingest facts from notes/docs, and prune stale entries when decisions
change.

**Use when:**
- "Ingest these meeting notes into memory."
- "What did we decide about the database migration?"
- "Who owns infrastructure for Project Helios?"

**What it does:**
- **Recall phase** — `search_nodes` / `read_graph` before answering or
  starting a task, so prior decisions aren't lost or contradicted.
- **Ingestion phase** — `create_entities`, `create_relations`,
  `add_observations` to record people, projects, technologies, and their
  relationships.
- **Pruning phase** — `delete_relations` / `delete_observations` when a
  decision is explicitly reversed or corrected.

## Try the Demo

See [DEMO_WALKTHROUGH.md](DEMO_WALKTHROUGH.md) for the full two-session
presenter script (ingest Project Helios notes, then recall them in a fresh
thread).

Expected graph shape after Session 1 is documented in
[plan/IMPLEMENTATION_PLAN.md §6.6](plan/IMPLEMENTATION_PLAN.md#66-expected-graph-shape-validation-reference)
and mirrored in [`data/memory.jsonl.example`](data/memory.jsonl.example).

## MCP Server Integration

### `memory` — Memory MCP Server

Local knowledge graph server: entities, relations, and observations
persisted as JSONL.

- **Transport:** stdio (`npx -y @modelcontextprotocol/server-memory`)
- **Storage:** `MEMORY_FILE_PATH`, pinned to `./data/memory.jsonl`
- **Authentication:** none — fully local, no API keys or cloud services

## Security Model

- **No hardcoded credentials** — the Memory MCP server is fully local and
  requires no API keys or tokens.
- **No credential echo** — N/A; this pack has no secrets to leak.
- **Data locality** — the knowledge graph is a plain JSONL file inside the
  project (`data/memory.jsonl`, gitignored); it never leaves the local
  filesystem unless you commit or copy it yourself.

## Architecture

```
knowledge-graph-demo/
├── catalog-info.yaml                 # Compass Location → plugin + skill + MCP targets
├── system.yaml                       # Compass System entity for this repo
├── knowledge-graph-demo-plugin.yaml  # Compass AiResource (type: plugin)
├── AGENTS.md                         # Skill-first routing
├── README.md
├── mcp.json                          # Memory MCP (stdio / npx)
├── mcps/
│   └── memory-mcp-server.yaml        # Compass MCPServer entity
├── skills/
│   └── agent-memory/
│       ├── SKILL.md
│       └── catalog-info.yaml         # Compass AiResource (type: skill)
├── data/
│   ├── sample-meeting-notes.md       # Demo input (Project Helios)
│   ├── memory.jsonl.example          # Reference graph after Session 1
│   └── memory.jsonl                  # Live graph — gitignored
├── plan/
│   └── IMPLEMENTATION_PLAN.md
├── DEMO_WALKTHROUGH.md               # Presenter script (Sessions 1 + 2)
└── .gitignore
```

## Distribution

This pack is registered in Compass and published through the existing
`plugin-pipeline` (P1 discover → P2 fetch → P3 scorecard → P4 assemble →
Konflux `plugin-pipeline-private-hub-release`) to the **private hub**
(`private-official-plugins`). It is not published to the public hub in v1
— see [plan/IMPLEMENTATION_PLAN.md](plan/IMPLEMENTATION_PLAN.md) for the
full distribution design, including the GitHub-fetch-credential
prerequisite.

## References

- [Memory MCP Server (upstream)](https://github.com/modelcontextprotocol/servers/tree/main/src/memory)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Lola Package Manager](https://github.com/LobsterTrap/lola)
- [Implementation Plan](plan/IMPLEMENTATION_PLAN.md)
