# Schema probe

Dummy Compass entities. They do not replace `knowledge-graph-demo-plugin.yaml`.
Entity names are `schema-probe*`, namespace `ai5-marketplace`.

Analyze one file. Stop on the preview. Do not click Finish.

Replace `main` with the branch name after you push. These files are not on GitHub until then.

| File | Check | Expected |
| ---- | ----- | -------- |
| `plugin.yaml` | `spec.skills` strings plus structured `spec.includes`, MCP in `includes`, `dependsOn`, `dependencyOf` | Pass, same as `rh-virt` on `fix-schema`. Then look for `dependsOn`, `dependencyOf`, and `hasPart` / `partOf` on the preview. |
| `plugin-publishing.yaml` | `distribution`, `support_level: developer-preview`, license, attribution, version, and `publishing.redhat.com/<market>: "true"` for `redhat-only`, `ecosystem`, `RHOAI`, `oci`, `public` | Record pass or the field named in the error. These ids are our routing markets, not a confirmed Compass vocabulary. |
| `plugin-includes-only.yaml` | `includes` and no `spec.skills` | `missingProperty: skills` |
| `plugin-skills-only.yaml` | `spec.skills` and no `includes` | Pass. Legacy roster is `spec.skills`. |
| `plugin-empty-includes.yaml` | `includes: []` and a nonempty `spec.skills` | Do not copy. ADR says membership is empty. Record whether validation passes. |
| `skills/recall/catalog-info.yaml` | Skill `dependsOn` the plugin, another skill, and the MCP | Pass. `spec.skills` is not on the skill. |
| `skills/note/catalog-info.yaml` | Skill `dependsOn` the MCP only | Pass. No plugin edge. |
| `mcps/probe-mcp.yaml` | MCP document has no `includes` | Pass on `type`, `lifecycle`, `owner`. |
| `system.yaml` | System for `spec.system: default/schema-probe` | Pass. |

```text
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/plugin.yaml
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/plugin-publishing.yaml
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/plugin-includes-only.yaml
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/plugin-skills-only.yaml
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/plugin-empty-includes.yaml
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/skills/recall/catalog-info.yaml
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/skills/note/catalog-info.yaml
https://github.com/RedTanny/knowledge-graph-demo/blob/main/schema-probe/mcps/probe-mcp.yaml
```

Start with `plugin.yaml`. The other plugin files are one question each.
