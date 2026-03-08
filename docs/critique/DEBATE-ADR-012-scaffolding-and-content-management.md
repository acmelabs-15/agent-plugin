---
title: DEBATE-ADR-012 Scaffolding and Content Management
type: critique
permalink: critique/debate-adr-012-scaffolding-and-content-management-1
tags:
- adr-review
- debate
- scaffolding
- content-management
- ADR-012
- agent-plugin
---

# DEBATE-ADR-012 Scaffolding and Content Management

## ADR Under Review

ADR-012 Scaffolding and Content Management

## Round 1 Results

**Outcome**: Unanimous NEEDS_REVISION (0/6 Accept)

Round 1: 0 Accept, 0 D&C, 6 Needs Revision, 0 Block

### Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | NEEDS REV | 3 | 5 | 4 |
| Critic | NEEDS REV | 2 | 6 | 5 |
| Independent Thinker | NEEDS REV | 1 | 5 | 3 |
| Security | NEEDS REV | 1 | 6 | 1 |
| Analyst | NEEDS REV | 1 | 4 | 2 |
| Advisor | NEEDS REV | 1 | 4 | 2 |

## P0 Issues (Consolidated)

### P0-1: MCP Command Group Collision (Architect)

- `agent-plugin mcp start/stop/restart/status` (author dev-time) collides with `plugin-name mcp start/stop/restart/status` (consumer daemon, ADR-011)
- Same verbs, different contexts creates confusion
**Decision needed: Separate author dev-time MCP commands from content-type group**

### P0-2: content.* Array Naming (Architect)

- ADR-012 uses `content.skills[]` nesting not present in ADR-001's flat `skills[]` arrays
- Inconsistency between ADRs on manifest structure
**Decision needed: Align manifest array naming with ADR-001**

### P0-3: Commands Contradicts ADR-001 (Architect + Critic)

- ADR-001 explicitly excludes commands ("legacy slash-command pattern being replaced by skills")
- ADR-012 reintroduces `command` content type without justification
**Decision needed: Drop commands or justify reintroduction with ADR-001 amendment**

### P0-4: No Phasing Strategy (Advisor)

- 4 creator skills at 3-4 weeks scope with no MVP/deferral plan
- Instruction-evaluator has no scoring rubric (IMP-004)
**Decision needed: Define MVP scope and phasing for creator skills**

### P0-5: gray-matter/yaml Inconsistency (Critic + Independent Thinker)

- ANALYSIS-031 recommends gray-matter.stringify() but ADR-012/ADR-006 disqualified it (CVE-2025-64718)
- Implementers following reference chain get contradictory guidance
**Decision needed: Update ANALYSIS-031 to remove gray-matter references**

### P0-6: eval/improve Trust Model (Security)

- No documentation of what eval/improve actually execute
- Whether generated code is run, what --apply does without review
- HTTP server binding for eval viewer undocumented
**Decision needed: Document trust model and execution boundaries for eval/improve**

### P0-7: Zod v4 + MCP SDK Compatibility (Analyst)

- Flagged as unverified by both ANALYSIS-031 and ADR-006 IMP-008
- ADR-012 treats compatibility as settled when it's not verified
**Decision needed: Acknowledge as unverified, add verification gate**

## P1 Issues (Consolidated, Deduplicated)

1. **Missing `prompts` content type** (Multiple) -- ADR-001 has 5 types including prompts, ADR-012 omits it
2. **No partial failure/rollback strategy** (Critic) -- Create commands can fail mid-operation leaving inconsistent state
3. **Marker comment syntax undefined** (Architect) -- MCP create-tool patching relies on undefined marker format
4. **eval/improve undefined for non-AI path** (Independent Thinker) -- What happens when no LLM is available?
5. **Agent create group names differ** (Analyst) -- ADR-012 and ANALYSIS-031 use different group names
6. **Instruction merge prompt injection risk** (Security) -- CWE-74/LLM01 via malicious instruction content
7. **Template injection in generated TypeScript** (Security) -- CWE-94 via unsanitized descriptions in tagged templates
8. **Creator skills supply chain provenance** (Security) -- No verification of Anthropic source integrity
9. **Concurrent manifest writes race condition** (Analyst) -- Multiple create commands could corrupt plugin.json
10. **Remove cross-reference detection undefined** (Critic) -- No specification for detecting dependent content before removal

## D&C Reservations

None. All 6 agents voted NEEDS REVISION.

## Round 2
## Round 2 Results

### Consensus: ACCEPTED (5 Accept, 1 D&C)

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | ACCEPT | 0 | 0 | 3 |
| Critic | ACCEPT | 0 | 6 | 3 |
| Independent Thinker | D&C | 0 | 3 | 3 |
| Security | ACCEPT | 0 | 5 | 1 |
| Analyst | ACCEPT | 0 | 2 | 2 |
| Advisor | ACCEPT | 0 | 2 | 3 |

### P0 Resolutions Verified

All 7 P0 issues from Round 1 resolved:

- P0-1: MCP lifecycle commands removed from author CLI command tree
- P0-2: Flat manifest array naming aligned with ADR-001
- P0-3: Commands kept as content type, IMP-005 tracks ADR-001 amendment
- P0-4: IMP-006 adds phasing note, creator skills independently shippable
- P0-5: IMP-007 tracks ANALYSIS-031 gray-matter cleanup
- P0-6: Decision 11 documents trust model, eval viewer, faithfulness principle
- P0-7: Verified compatible with minimum versions documented, ANALYSIS-032 created

### D&C Reservations (Independent Thinker)

- Faithfulness principle (Decision 11) needs explicit subordination to ADR-001 plugin format when they conflict
- IMP-007 scope too narrow: ANALYSIS-031 has stale content beyond gray-matter (fastmcp, content.* naming, command names, group names)
- Marker comment failure behavior unspecified: what does create-tool do when markers are missing?

### Common P1 Themes

1. ANALYSIS-031 cleanup scope broader than IMP-007 covers (4 agents flagged)
2. Bun.serve() binding should specify 127.0.0.1 and ephemeral port (1 agent)
3. Missing prompts content type from ADR-001 not addressed (2 agents)
4. Marker comment syntax and error handling undefined (2 agents)
5. Partial failure/rollback for create commands unspecified (1 agent)
6. Template literal injection cross-reference to ADR-003 Layer 6 missing (1 agent)

### Status Change

- ADR-012 status changed from Proposed to **Accepted**
## Observations

- [fact] Round 1 produced unanimous NEEDS_REVISION with 7 consolidated P0 issues across 6 agents #adr-review #adr-012
- [risk] MCP command group collision between author dev-time and consumer daemon commands #mcp #cli-architecture
- [risk] content.* array naming inconsistency between ADR-012 and ADR-001 manifest structure #manifest
- [risk] Commands content type contradicts ADR-001 exclusion of legacy slash-command pattern #commands
- [risk] No phasing strategy for 3-4 week creator skills implementation scope #planning
- [risk] ANALYSIS-031 has stale gray-matter references contradicting ADR-012 and ADR-006 #documentation
- [risk] eval/improve trust model and security boundaries undocumented #security
- [insight] Zod v4 and MCP SDK compatibility treated as settled but remains unverified #dependencies

## Relations

- reviews [[ADR-012 Scaffolding and Content Management]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ADR-011 Auto-Generated CLI from MCP Tools]]
- relates_to [[ADR-006 Core Dependency Stack]]
- relates_to [[ANALYSIS-031 Scaffolding Wizard Patterns]]

- [outcome] ADR-012 Round 2 ACCEPTED: All 7 P0 issues resolved, 5 Accept + 1 D&C consensus achieved #round-2
- [outcome] Independent Thinker D&C: faithfulness principle should be subordinate to ADR-001 format conformance #reservation
- [fact] 4 of 6 agents flagged ANALYSIS-031 cleanup scope as broader than IMP-007 covers #analysis-031