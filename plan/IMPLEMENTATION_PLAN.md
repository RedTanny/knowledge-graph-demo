# Knowledge Graph Demo — Implementation Plan

> **Status:** Revised after review — ready for implementation  
> **Source idea:** [`github-demo.md`](../../github-demo.md) in the AI5 workspace  
> **Target repo:** [`knowledge-graph-demo`](https://github.com/RedTanny/knowledge-graph-demo)  
> **Primary goal:** A **Compass-registered agentic pack** that the existing publication pipeline can discover, fetch, and publish to the **private hub** (`private-official-plugins`), with a hands-on demo payload inside the pack (meeting notes → knowledge graph → fresh-session recall).

---

## 1. What Changed After Review

The first draft optimized for a **local Cursor demo** (`.cursor/mcp.json`, `.cursor/skills/`, clone-and-run). That is useful **content**, but not the **distribution** path.

This revision optimizes for four product goals:

| # | Goal | How this plan addresses it |
|---|------|---------------------------|
| 1 | **Register in Compass** | Backstage `Location` + `AiResource` manifests (`plugin`, `skill`, optionally `MCPServer`) — same model as `rh-basic` |
| 2 | **Route through the existing pipeline to the private hub** | P1 discover → P2 fetch → P3 scorecard → P4 assemble → Konflux `plugin-pipeline-private-hub-release` (`HUB_TARGET=internal`) |
| 3 | **Match `rh-basic` / `agentic-plugins` pack shape** | Pack-root `AGENTS.md`, `catalog-info.yaml`, `mcp.json`, `skills/<name>/SKILL.md` — not `.cursor/` as source of truth |
| 4 | **`development` lifecycle** | `spec.lifecycle: development` on **both** plugin and skill; private-hub only (no `Marketplace-Public`) |

The Helios meeting-notes demo and two-session walkthrough are **pack content**, not a substitute for the pack.

---

## 2. Problem Statement

AI agents lose context between sessions. Meeting decisions, ownership, and technical choices buried in notes are hard to retrieve unless the user re-pastes them every time.

This pack demonstrates a lightweight alternative: a **local knowledge graph** (`data/memory.jsonl`) managed through the Memory MCP server, with an `agent-memory` skill governing recall, ingest, and prune.

**Demo "wow" moment:** Session 2 opens with zero prior conversation context; the agent still answers correctly by querying the graph.

**Product "wow" moment:** The same pack is installable from the **private hub** after Compass discovery and pipeline publish — not only from a raw git clone.

---

## 3. Scope

### In scope

| Item | Description |
|------|-------------|
| **Pack scaffold** | `rh-basic`-shaped layout at repo root (this repo **is** the pack; `agentic.redhat.com/path: .`) |
| **Compass manifests** | `Location`, plugin `AiResource`, skill `AiResource`, Memory MCP modeling |
| **Agent skill** | `agent-memory` with `rh-basic` frontmatter + `allowed-tools` for Memory MCP |
| **MCP config** | Pack-root `mcp.json` (stdio / npx) — not `.cursor/mcp.json` as source of truth |
| **Demo payload** | Helios sample notes, expected graph shape, `DEMO_WALKTHROUGH.md` |
| **Private hub publish** | Trigger existing `plugin-pipeline-private-hub-release`; expect `plugins/<pack-id>/` on `private-official-plugins` |
| **Hub install validation** | Install from private hub (Lola or equivalent), then run Session 1 + Session 2 |

### Out of scope (v1)

- This repo's own Konflux catalog pipeline (uses the existing `plugin-pipeline` ReleasePlan)
- Public / external hub (`official-plugins`) — requires `beta`+ lifecycle and `Marketplace-Public`
- Custom MCP server (use `@modelcontextprotocol/server-memory`)
- Graph visualization UI
- Automated CI in this repo

### Explicitly deferred

| Item | Notes |
|------|-------|
| GitHub fetch credentials in Konflux | Pipeline prerequisite; see §8 |
| `Marketplace-Public: "true"` | Private-only intent for v1 |
| Session 3 (prune/update) | Stretch; not blocking publish |

---

## 4. Repository Layout

Single-pack GitHub repo — flattened `rh-basic` shape at repo root:

```text
knowledge-graph-demo/
├── catalog-info.yaml                      # Location → plugin + skill (+ MCP) targets
├── knowledge-graph-demo-plugin.yaml       # AiResource spec.type: plugin
├── AGENTS.md                              # Skill-first routing (required by P2/P4)
├── README.md
├── mcp.json                               # Memory MCP (stdio / npx)
├── mcps/
│   └── memory-mcp-server.yaml             # MCPServer entity (optional but recommended)
├── skills/
│   └── agent-memory/
│       ├── SKILL.md                       # rh-basic frontmatter + allowed-tools
│       └── catalog-info.yaml              # AiResource spec.type: skill
├── data/
│   ├── sample-meeting-notes.md            # Demo input (Project Helios)
│   ├── memory.jsonl.example              # Reference graph after Session 1
│   └── memory.jsonl                       # Live graph — gitignored
├── plan/
│   └── IMPLEMENTATION_PLAN.md             # This document
├── DEMO_WALKTHROUGH.md                    # Presenter script (Sessions 1 + 2)
└── .gitignore
```

### What we drop from the first draft

- **`.cursor/` as source of truth** — host-specific MCP paths (Cursor `.cursor/mcp.json`, Claude `.mcp.json`) belong in README, documenting setup instead of assuming the IDE auto-loads pack MCP config.
- **LOLA / marketplace as "out of scope"** — private hub publish **is** in scope; LOLA install is the validation path.

### Repo vs monorepo fork

**Decision (locked):** Keep this as its own GitHub repo (`github.com/RedTanny/knowledge-graph-demo`), set `agentic.redhat.com/path: .`, and treat GitHub clone credentials as an explicit pipeline prerequisite — not as a reason to hide the pack inside GitLab `agentic-plugins`.

Alternative (easier fetch today, **not** chosen): new folder in GitLab `agentic-plugins` skips the GitHub credential gap but does not match "register this repo."

---

## 5. Compass Registration

Compass does **not** ingest `.cursor/` trees. It ingests Backstage manifests the same way `rh-basic` does.

### Required manifest files

| File | Kind | Role |
|------|------|------|
| `catalog-info.yaml` | `Location` | Points Compass at plugin + skill (+ MCP) manifests |
| `knowledge-graph-demo-plugin.yaml` | `AiResource` `spec.type: plugin` | Pack identity, owner, **lifecycle**, clone URL |
| `skills/agent-memory/catalog-info.yaml` | `AiResource` `spec.type: skill` | Per-skill identity + `dependsOn` |
| `mcps/memory-mcp-server.yaml` | `MCPServer` (recommended) | Memory MCP primitives + lifecycle |

### Clone annotations (pipeline P2)

P2 clones from these annotations on the plugin `AiResource`:

```yaml
annotations:
  agentic.redhat.com/repository: https://github.com/RedTanny/knowledge-graph-demo
  agentic.redhat.com/path: .                    # pack IS the repo root — cannot be empty
  agentic-plugins.redhat.com/version: "0.1.0"
```

Until Compass has a `type: plugin` entity for this repo, **P1 never sees it → P2 never clones it.**

### MCP modeling

Skills in `rh-basic` declare `dependsOn: [mcpserver:...]`. A skill that calls Memory MCP tools without that relation is incomplete Compass modeling.

**Recommended approach:**

1. **`mcp.json`** at pack root — golden source for Lola / pipeline fetch.
2. **`mcps/memory-mcp-server.yaml`** — `MCPServer` entity listing Memory MCP tool primitives (`create_entities`, `create_relations`, `add_observations`, `search_nodes`, `read_graph`, `delete_relations`, `delete_observations`, `open_nodes`).
3. **Plugin + skill `dependsOn`** — both reference `mcpserver:ai5-marketplace/memory-mcp-server` (or chosen namespace/name).

Pack-local `mcp.json` alone is insufficient for Compass relation graphs; the `MCPServer` entity completes the model.

### Lifecycle — `development` on everything

Set **`spec.lifecycle: development`** on plugin, skill, **and** MCPServer.

| Surface | `development` eligibility |
|---------|---------------------------|
| Internal catalog / **private hub** | ✓ Eligible (L1 quality bar per ADR) |
| `official-plugins` / public | ✗ Not eligible (`beta`+ required) |

**Do not** copy `rh-basic`'s mismatch (plugin `beta`, skills `beta` with mixed ceilings). **Do not** add `Marketplace-Public: "true"` — private-only intent.

Plugin lifecycle sets the **ceiling** for child skills. Skill lifecycle must not exceed plugin lifecycle (see `validate_lifecycle_ceiling.py`).

---

## 6. Component Specifications

### 6.1 MCP Configuration (`mcp.json`)

Pack-root golden source (not `.cursor/mcp.json`):

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

**Behaviors:**

- `MEMORY_FILE_PATH` pins storage to `./data/memory.jsonl` inside the installed project.
- `npx -y` avoids global install; Node.js 18+ is the only runtime prerequisite.
- Path is relative to workspace root — verify during implementation; document absolute-path workaround in README if Cursor resolves env relative to a different cwd.

**Host setup (documented in README, not committed as source of truth):**

| Host | Config path | Notes |
|------|-------------|-------|
| Cursor | `.cursor/mcp.json` | Copy or symlink from installed `mcp.json` |
| Claude Code | `.mcp.json` | Same |
| Copilot | VS Code MCP settings | Per README |

### 6.2 Agent Memory Skill (`skills/agent-memory/SKILL.md`)

Follow `rh-basic` skill format — YAML frontmatter with `allowed-tools` listing Memory MCP tool names:

```yaml
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
```

**Three phases (unchanged from content draft):**

#### Recall Phase
- Before non-trivial tasks, call `search_nodes` or `read_graph`.
- Prefer `search_nodes` for targeted queries; use `read_graph` when scope is unclear.

#### Ingestion & Storage Phase
1. **`create_entities`** — people, projects, systems, technologies
2. **`create_relations`** — directed links with verb-like relation types
3. **`add_observations`** — factual properties on entities

**Entity naming conventions:**
- Projects: `Project_<Name>` (e.g. `Project_Helios`)
- People: first name (e.g. `Sarah`)
- Technologies: canonical name (e.g. `PostgreSQL`, `Prisma`)

#### Pruning Phase
- Before updating changed decisions, remove stale data with `delete_relations` and `delete_observations`.
- Do not delete entities unless explicitly obsolete.

### 6.3 AGENTS.md (skill-first routing)

Required by P2/P4. Minimal routing table:

| When the user asks about… | Use skill |
|---------------------------|-----------|
| Ingest notes/docs into memory, store decisions in the knowledge graph | `/agent-memory` |
| Recall past decisions, who owns what, what was decided | `/agent-memory` |
| Update or correct a stored decision | `/agent-memory` |

Include MCP dependency note: Memory MCP must be configured locally (see README).

### 6.4 Sample Input (`data/sample-meeting-notes.md`)

Keep Project Helios verbatim from `github-demo.md`:

- **Attendees:** Sarah (Lead Engineer), David (DevOps), Marcus (Product)
- **Decisions:** MongoDB → PostgreSQL migration; Prisma ORM approved; mobile MVP postponed
- **Action items:** David → RDS provisioning; Marcus → Jira roadmap update

### 6.5 Graph file strategy (decided)

**Option C:** commit `data/memory.jsonl.example` as reference output; gitignore `data/memory.jsonl`.

### 6.6 Expected Graph Shape (validation reference)

After Session 1 ingestion:

| Entity | Type | Example observations |
|--------|------|----------------------|
| `Project_Helios` | project | "Migrating user database from MongoDB to PostgreSQL" |
| `Sarah` | person | "Lead Engineer"; "Approved Prisma as ORM" |
| `David` | person | "DevOps"; "Will provision AWS RDS PostgreSQL" |
| `Marcus` | person | "Product"; "Will update Jira roadmap" |
| `PostgreSQL` | technology | "Target database for migration" |
| `Prisma` | technology | "Approved ORM" |
| `MongoDB` | technology | "Legacy database being replaced" |

Example relations:
- `Sarah` → `leads` → `Project_Helios`
- `Sarah` → `approved` → `Prisma`
- `David` → `owns_infrastructure` → `Project_Helios`
- `Project_Helios` → `migrating_to` → `PostgreSQL`

Walkthrough validates **semantic correctness**, not exact node IDs.

---

## 7. Publication Pipeline Path

This repo does **not** need its own Konflux pipeline. It plugs into the existing flow:

```text
Compass (AiResource registered)
        │
        ▼
P1 Discover ──► returns plugin with GitHub clone URL + path .
        │
        ▼
P2 Fetch ──► clones github.com/RedTanny/knowledge-graph-demo
        │      extracts AGENTS.md, skills/*/SKILL.md, mcp.json
        ▼
P3 Scorecard ──► SoundCheck per skill (L1 bar for development)
        │
        ▼
P4 Assemble ──► enriched catalog artifact
        │
        ▼
Konflux ReleasePlan: plugin-pipeline-private-hub-release
        │  HUB_TARGET=internal
        ▼
private-official-plugins/plugins/<pack-id>/
```

### Private hub routing (secure default)

- No `Marketplace-Public` / `marketplace.public.enabled` annotation → **internal hub only**.
- `development` lifecycle is **allowed** on the internal hub (pipeline blocks `development` only on the external hub).
- Matches "private publishing repo" intent.

### Fetch blocker: GitHub credentials

Konflux fetch credentials today are a GitLab project token on `ai5-marketplace/agentic-plugins`. GitHub fetch was explicitly deferred in the integration plan.

| Repo visibility | P2 fetch behavior |
|-----------------|-------------------|
| **Public** GitHub | Anonymous clone may work without new credentials |
| **Private** GitHub | P2 fails until GitHub App / org token is added to Konflux |

**Prerequisite (track explicitly):** Before private publish succeeds, confirm repo visibility **or** add GitHub fetch credential to the pipeline. This is a pipeline ops task, not a pack-layout task.

---

## 8. Architecture

### Runtime (demo payload)

```text
┌─────────────────────┐     ┌──────────────────────────┐
│  Meeting notes      │     │  Agent + agent-memory    │
│  sample-meeting-    │────▶│  skill (from hub install)│
│  notes.md           │     │                          │
└─────────────────────┘     └───────────┬──────────────┘
                                        │ Memory MCP tools
                                        ▼
                            ┌──────────────────────────┐
                            │ @modelcontextprotocol/   │
                            │ server-memory (npx)      │
                            └───────────┬──────────────┘
                                        │ MEMORY_FILE_PATH
                                        ▼
                            ┌──────────────────────────┐
                            │ data/memory.jsonl        │
                            │ (local knowledge graph)  │
                            └──────────────────────────┘
```

### Distribution (product path)

```text
knowledge-graph-demo (GitHub)
        │ Compass Location + AiResource manifests
        ▼
plugin-pipeline (P1–P4)
        │ plugin-pipeline-private-hub-release
        ▼
private-official-plugins
        │ lola install (or equivalent)
        ▼
User project (skills + mcp.json + demo data)
```

---

## 9. Demo Walkthrough (`DEMO_WALKTHROUGH.md`)

Two-session script. **Validation path:** install from **private hub**, not only raw git clone.

### Prerequisites

- Pack installed from private hub (or dev override: clone repo + manual MCP setup)
- Memory MCP connected in the IDE
- Node.js 18+

### Session 1 — Knowledge Ingestion

1. Confirm Memory MCP is connected.
2. Start a **new** agent chat.
3. Prompt:
   > Read `data/sample-meeting-notes.md` and ingest all technical decisions, ownership, and tasks into persistent memory.
4. **Observe:** `create_entities`, `create_relations`, `add_observations`.
5. **Verify:** `data/memory.jsonl` populated.

### Session 2 — Fresh Context Recall ("Wow" Phase)

1. **Close** chat / start a completely new agent thread.
2. Prompt:
   > What database transition was decided for Project Helios, and who owns the infrastructure setup?
3. **Observe:** `search_nodes` or `open_nodes` — **not** reading the markdown file.
4. **Expected answer:** PostgreSQL migration; David owns infrastructure (RDS provisioning).

### Session 3 (stretch, deferred)

Follow-up note reversing a decision → prune stale graph entries → re-query.

---

## 10. Implementation Phases

### Phase 0 — Demo content (from first draft, kept)

- [ ] `data/sample-meeting-notes.md` (Project Helios)
- [ ] Expected graph shape documented (§6.6)
- [ ] `data/memory.jsonl.example` + gitignore live `memory.jsonl`
- [ ] `DEMO_WALKTHROUGH.md` (Sessions 1 + 2)

### Phase 1 — Pack scaffold

- [ ] Create directory structure (§4)
- [ ] `catalog-info.yaml` (Location)
- [ ] `knowledge-graph-demo-plugin.yaml` (`lifecycle: development`)
- [ ] `skills/agent-memory/SKILL.md` + `catalog-info.yaml` (`lifecycle: development`, `dependsOn` plugin + MCPServer)
- [ ] `mcp.json` (Memory MCP)
- [ ] `mcps/memory-mcp-server.yaml` (`lifecycle: development`)
- [ ] `AGENTS.md` (skill-first routing)
- [ ] `README.md` (install, MCP host setup, troubleshooting)
- [ ] `.gitignore`

### Phase 2 — Compass registration

- [ ] Register `catalog-info.yaml` Location in Compass
- [ ] Confirm P1 discover returns this plugin with GitHub clone URL and `path: .`
- [ ] Verify plugin appears with `spec.type: plugin`, `lifecycle: development`
- [ ] Verify skill + MCPServer entities resolve in Compass graph

### Phase 3 — Fetch unblock

- [ ] Confirm repo visibility (public → anonymous clone) **or** add GitHub fetch credential to Konflux
- [ ] Run P2 fetch manually / via pipeline; confirm `AGENTS.md`, `skills/`, `mcp.json` extracted
- [ ] Fix any path or annotation issues blocking clone

### Phase 4 — Private hub publish

- [ ] Run P3 scorecard (L1 pass for `development`)
- [ ] Run P4 assemble
- [ ] Trigger `plugin-pipeline-private-hub-release` (`HUB_TARGET=internal`)
- [ ] Confirm `plugins/knowledge-graph-demo/` (or chosen pack-id) on `private-official-plugins`

### Phase 5 — Demo validation (from hub)

- [ ] Install pack from private hub (Lola or equivalent)
- [ ] Configure Memory MCP per README
- [ ] Run Session 1 ingestion → `memory.jsonl` created
- [ ] Run Session 2 recall in fresh thread → PostgreSQL + David without reading notes
- [ ] Capture any cwd / MCP path issues into docs

---

## 11. Prerequisites & Environment

| Requirement | Phase | Notes |
|-------------|-------|-------|
| Node.js 18+ | Demo | For `npx @modelcontextprotocol/server-memory` |
| Compass registration | Phase 2 | Backstage `AiResource` entities |
| Pipeline access | Phase 4 | Konflux ReleasePlan trigger |
| GitHub clone (public or credentialed) | Phase 3 | See §7 fetch blocker |
| Lola (or hub install path) | Phase 5 | Validate end-to-end from private hub |
| Supported IDE with MCP | Demo | Cursor, Claude Code, etc. |

No API keys for the Memory MCP itself. No cloud services. No database install.

---

## 12. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| P2 fails on private GitHub repo | Make repo public for POC, or add GitHub App token to Konflux fetch |
| `MEMORY_FILE_PATH` wrong cwd | Test early; document absolute-path workaround |
| Agent reads markdown in Session 2 | Skill recall-first; walkthrough requires fresh thread |
| Lifecycle ceiling mismatch | Set `development` on plugin **and** skill (and MCPServer) |
| Accidental public publish | Omit `Marketplace-Public`; use private ReleasePlan only |
| MCP not auto-loaded after hub install | Document per-host setup in README |
| MCP package version drift | Pin in `mcp.json` args if needed |

---

## 13. Success Criteria

### Distribution (primary)

- [ ] Compass P1 returns `knowledge-graph-demo` plugin with correct GitHub URL and `path: .`
- [ ] P2 fetch succeeds and extracts pack golden sources
- [ ] Private hub publish produces installable pack on `private-official-plugins`
- [ ] Hub install + MCP setup documented and reproducible

### Demo (content, still required)

- [ ] Session 1 produces populated `data/memory.jsonl`
- [ ] Session 2 (fresh thread) answers PostgreSQL + David **without** reading `sample-meeting-notes.md`
- [ ] `DEMO_WALKTHROUGH.md` sufficient for a non-author presenter

### Quality bar

- [ ] Plugin + skill + MCPServer all `lifecycle: development`
- [ ] Skill `dependsOn` includes MCPServer relation
- [ ] No `Marketplace-Public` annotation
- [ ] Lifecycle ceiling validator passes (skill ≤ plugin)

---

## 14. Decisions Log

| Decision | Resolution |
|----------|------------|
| Repo location | Own GitHub repo (`RedTanny/knowledge-graph-demo`), `path: .` |
| Layout | `rh-basic` pack shape at repo root; drop `.cursor/` as source of truth |
| Lifecycle | `development` on plugin, skill, and MCPServer |
| Hub target | Private only (`plugin-pipeline-private-hub-release`) |
| Graph files | `memory.jsonl.example` committed; live `memory.jsonl` gitignored |
| Sample scenario | Keep Project Helios |
| MCP modeling | `mcp.json` + `MCPServer` entity + skill `dependsOn` |
| GitHub fetch | Explicit pipeline prerequisite; public repo OR credential |
| Session 3 (prune) | Deferred |

---

## 15. Future Extensions (post-v1)

- Promote lifecycle to `beta` + opt into `official-plugins` with `Marketplace-Public`
- Second sample input (ADR) showing graph merge across ingestions
- `scripts/validate-graph.sh` checking expected entities in `memory.jsonl`
- GitHub fetch credential automation in Konflux (production fetch shape)
- SoundCheck eval coverage beyond L1 for `development`
