---
permalink: analysis/analysis-048-decision-audit-reconciliation
---

# Decision Audit Reconciliation (2026-03-09)

Source: 6 parallel extraction agents read 22,969 lines of conversation log line-by-line, then 1 reconciliation agent walked through all findings chronologically.

## P0 GAPS (Critical — design decisions exist but have no ADR)

### 1. ADR-014 NOT CREATED
The entire post-pivot design has no formal ADR. ADR-013 was accepted then immediately superseded by a design pivot. Decisions needing ADR-014:
- Explicit `agent-plugin add/remove/update` commands (Vercel Skills style)
- `.agent-lock.json` in consuming project (source, sourceType, hash, features, timestamps)
- `.agent-plugin/plugin.json` manifest location (directory provides disambiguation)
- Features model (plugin-level + component-level, section-based for markdown/code, file-based for rules, none for MCP)
- No consumer-side config file
- No init/deinit commands
- Revised command tree (add/remove/update/install/list + create/validate/build + mcp serve + content-type CRUD)
- 8-step add flow
- Content placed directly to platform locations (not symlinks)
- install from lockfile for team sync

### 2. ADR-001 STALE (5/8+ amendments applied, 3+ missing)
Applied: mcpServers rename, string|string[] paths, inline objects, commands as 6th type, version removed
Missing:
- Manifest location: `.agent-plugin/plugin.json` (not root-level)
- Content type taxonomy: 8 types replacing 6 (Skills, Agents, Hooks, Commands, Rules, MCP, AGENTS.md, CLI)
- installMode replaced by features model
- `prompts/` replaced by `rules/`
- `instructions/` replaced by `AGENTS.md`

### 3. ADR-013 NOT MARKED SUPERSEDED
Status is "accepted" but contains pre-pivot design (postinstall model, init/deinit, no consumer commands). Needs supersession header pointing to ADR-014.

## P1 GAPS (Missing analysis/documentation)

1. **Features model analysis** — ANALYSIS-043 and ANALYSIS-044 exist but don't reflect final user corrections (section-based hooks/commands, standard markdown headers, no custom markers)
2. **Post-pivot lockfile design** — No analysis note. ANALYSIS-046 recommended opposite of what was decided. ANALYSIS-011 is pre-ADR-013.
3. **plugin.json schema (post-pivot)** — Schema drafted in session but no standalone note
4. **Content type taxonomy update** — Shift from 6 to 8 types only in session note
5. **Add flow documentation** — 8-step flow only in session note
6. **Vercel Skills CLI research** — Detailed research done during pivot (skills-lock.json, source types) but no formal ANALYSIS note

## P2 GAPS (Housekeeping)

1. ANALYSIS-042: Should reflect "deferred to v2" decision
2. ANALYSIS-043: Stale conclusion (recommends per-component; user chose per-feature section-level)
3. ADR-003 Decision 3 supersession note mentions "config file TBD" — should say "no config file"
4. Session note status still IN_PROGRESS, end protocol unchecked
5. ADR-008/010 supersession chain will need updating when ADR-014 supersedes ADR-013

## KEY REVERSALS (Chronological)

1. **Consumer commands**: Eliminated by ADR-013 → Reinstated post-pivot as explicit add/remove/update
2. **Config file**: .agent-plugin/config.json → agent-plugin.config.json → agents.config.json → NO CONFIG FILE
3. **Lockfile**: plugin-lock.json (ADR-003) → bun.lockb only (ADR-013) → .agent-lock.json (post-pivot)
4. **init/deinit**: Created by ADR-013 → Eliminated post-pivot
5. **installMode**: bundle/collection (ADR-001) → Features model with section-level selection
6. **Manifest location**: Root plugin.json (ADR-001) → .agent-plugin/plugin.json
7. **Content types**: 6 types with prompts/instructions → 8 types with Rules/AGENTS.md/CLI
8. **Section markers**: Custom markers → Standard markdown headers (user correction)

## FINAL DECISIONS SUMMARY

| Topic | Final Decision | Decider |
|-------|---------------|---------|
| Installation model | Explicit `agent-plugin add {source}` (Vercel style) | User |
| Lockfile | `.agent-lock.json` in consuming project | User |
| Config file | None. Lockfile + platform configs only. | User |
| Manifest location | `.agent-plugin/plugin.json` | User |
| Content types | Skills, Agents, Hooks, Commands, Rules, MCP, AGENTS.md, CLI | User |
| Feature mechanism (markdown) | Standard markdown headers via remark/unified | User correction |
| Feature mechanism (code) | `// #region feature:NAME` markers, custom parser | User confirmed |
| Feature mechanism (rules) | File-based inclusion/exclusion | User confirmed |
| Feature mechanism (MCP) | None — always installed as-is | User |
| Hooks features | Section-based (NOT file-based) | User correction |
| Commands features | Section-based | User |
| Command tree | add/remove/update/install/list + create/validate/build + mcp serve + CRUD groups | User confirmed |
| Add flow | 8-step: resolve → read manifest → validate → detect → wizard → parse → write → update lockfile | User confirmed |
| init/deinit | Eliminated | User |
| plugin.json name vs package.json name | Intentionally independent (Option C) | User |
| Native platform plugins | Deferred to v2 | User confirmed |
| ADR-013 impact | Option C: keep accepted, create ADR-014 to supersede | User |
| Dev workflow --watch | Non-issue, not needed | User |
| Content placement | Direct to platform locations (not symlinks) | User |
| Team sync | `agent-plugin install` from lockfile | User confirmed |
| Source types | npm packages, git repos, local paths (same as Vercel) | User confirmed |
| Platform metadata | In plugin.json per component `platforms` field | User confirmed |

## RECOMMENDED PRIORITY

1. [P0] Create ADR-014 capturing all post-pivot decisions
2. [P0] Update ADR-001 with remaining amendments
3. [P0] Add supersession header to ADR-013
4. [P1] Create/update analysis notes for features model, lockfile design, plugin.json schema
5. [P1] Fix ADR-003 Decision 3 supersession note
6. [P2] Close session, housekeeping