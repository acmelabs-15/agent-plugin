---
title: ANALYSIS-045 Pending Work Plan and Parallelism Strategy
type: note
permalink: analysis/analysis-045-pending-work-plan-and-parallelism-strategy-1
tags:
- planning
- parallelism
- work-streams
- agent-plugin
---

# ANALYSIS-045 Pending Work Plan and Parallelism Strategy

## Work Streams

### Stream A: Research (independent, launches now)

| ID | Task | Agent Type | Output File | Status |
|----|------|-----------|-------------|--------|
| A1 | Config file naming/placement conventions research (project-scoped only, .foorc vs foo.config.js vs .foo/ vs platform dirs) | analyst | ANALYSIS-046 | PENDING |

### Stream B: ADR-013 P0 Application (agreed, launches now)

All 3 P0 resolutions edit the SAME file (ADR-013). Single agent.

| ID | Task | File | Status |
|----|------|------|--------|
| B1 | Apply P0-1: Add empirical Bun postinstall test evidence, mark claim correct | ADR-013 | PENDING |
| B2 | Apply P0-2: Add full recompute approach, reference ANALYSIS-035 | ADR-013 | PENDING |
| B3 | Apply P0-3: Add direct deps restriction, wiring summary, path validation | ADR-013 | PENDING |

### Stream C: ADR-001 Amendments (agreed, launches now)

All 5 amendments edit the SAME file (ADR-001). Single agent.

| ID | Task | File | Status |
|----|------|------|--------|
| C1 | Rename `mcp` to `mcpServers` | ADR-001 | PENDING |
| C2 | Support `string or string[]` for content paths | ADR-001 | PENDING |
| C3 | Support inline objects for hooks/mcpServers | ADR-001 | PENDING |
| C4 | Add commands as 6th content type (IMP-005 from ADR-012) | ADR-001 | PENDING |
| C5 | Remove version from required fields (package.json authoritative) | ADR-001 | PENDING |

### Stream D: ADR Supersessions (agreed, launches now)

3 different files. Single agent editing sequentially (small changes).

| ID | Task | File | Status |
|----|------|------|--------|
| D1 | Mark ADR-008 fully superseded by ADR-013 | ADR-008 | PENDING |
| D2 | Mark ADR-010 mostly superseded by ADR-013 | ADR-010 | PENDING |
| D3 | Mark ADR-003 Decision 3 lockfile superseded by ADR-013 | ADR-003 | PENDING |

### Stream E: Cleanup (independent, launches now)

| ID | Task | File | Status |
|----|------|------|--------|
| E1 | IMP-007: Remove stale gray-matter references from ANALYSIS-031 | ANALYSIS-031 | PENDING |

### Stream F: Needs User Discussion (SEQUENTIAL, cannot parallelize)

| ID | Task | Blocker | Status |
|----|------|---------|--------|
| F1 | ADR-013 P1-1: ANALYSIS-034 contradiction acknowledgment | Agreed, apply during B stream | AGREED |
| F2 | ADR-013 P1-2: Husky uses prepare not postinstall | Needs user discussion | NOT STARTED |
| F3 | ADR-013 P1-3: Missing MADR Confirmation section | Needs user discussion | NOT STARTED |
| F4 | ADR-013 P1-4: Missing Pros/Cons for rejected options | Needs user discussion | NOT STARTED |
| F5 | ADR-013 P1-5: Dev workflow gap (no file watching) | Needs user discussion | NOT STARTED |
| F6 | ADR-013 P1-6: "60% complexity reduction" unsourced | Needs user discussion | NOT STARTED |
| F7 | ADR-013 P1-7: plugin.json name vs package.json name | Needs user discussion | NOT STARTED |
| F8 | ADR-013 P1-8: Path validation (CWE-22) | Covered by P0-3 resolution | RESOLVED |
| F9 | ADR-013 P1-9: Wiring change output visibility | Covered by P0-3 wiring summary | RESOLVED |
| F10 | ADR-013 P1-10: bun update postinstall unverified | Resolved by empirical test (fires on all ops) | RESOLVED |
| F11 | ADR-013 P1-11: Isolated linker compatibility | Needs user discussion | NOT STARTED |
| F12 | Features model design finalization | Needs A1 research + user decisions | NOT STARTED |
| F13 | Config naming/placement decision | Depends on A1 research | NOT STARTED |
| F14 | New ADR for features/sections/installMode | Depends on F12 + F13 | NOT STARTED |
| F15 | Native platform plugins decision | ANALYSIS-042 recommends defer to v2 | NEEDS USER |
| F16 | ADR-013 Round 2 convergence vote | Depends on B + F1-F11 | NOT STARTED |

### Stream G: Session Note Update (after parallel work completes)

| ID | Task | File | Status |
|----|------|------|--------|
| G1 | Update session note with all completed parallel work | SESSION-2026-03-07_01 | AFTER PARALLEL |

## Parallelism Diagram

```
NOW (5 parallel agents):
├── A1: Config naming research ──────────────────────► ANALYSIS-046
├── B1-B3: ADR-013 P0 application ──────────────────► ADR-013 updated
├── C1-C5: ADR-001 amendments ──────────────────────► ADR-001 updated
├── D1-D3: ADR supersessions ──────────────────────► ADR-008/010/003 updated
└── E1: ANALYSIS-031 cleanup ──────────────────────► ANALYSIS-031 updated

AFTER PARALLEL (sequential with user):
├── F2-F7, F11: ADR-013 P1 review (one at a time)
├── F12-F14: Features model + config naming + new ADR
├── F15: Native plugins decision
├── F16: ADR-013 Round 2 vote
└── G1: Session note final update
```

## Constraints

- [constraint] No two agents edit the same file (prevents last-write-wins data loss)
- [constraint] P1 issues require one-at-a-time user discussion (user directive)
- [constraint] Features ADR depends on config naming research completing first
- [constraint] ADR-013 Round 2 vote requires all P0/P1 resolutions applied first
- [fact] P1-8, P1-9, P1-10 are already resolved by P0 resolutions and empirical testing
- [fact] P1-1 is agreed but not yet applied (will be included in Stream B)

## Observations

- [decision] 5 parallel agents for independent work streams (research, ADR-013 P0s, ADR-001 amendments, supersessions, cleanup) #parallelism
- [decision] Sequential user discussion for P1 issues and features model after parallel work completes #process
- [fact] 3 of 11 P1 issues already resolved by P0 work and empirical testing (P1-8, P1-9, P1-10) #efficiency
- [fact] Remaining P1 issues needing discussion: P1-2, P1-3, P1-4, P1-5, P1-6, P1-7, P1-11 (7 items) #scope

## Relations

- relates_to [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ADR-008 Source Resolution and Package Validation]]
- relates_to [[ADR-010 Installation Lifecycle]]
- relates_to [[ANALYSIS-031 Scaffolding Wizard Patterns]]
- relates_to [[ANALYSIS-042 Native Platform Plugin Installation Feasibility]]
- relates_to [[ANALYSIS-043 Per-Component Cherry-Picking InstallMode Design]]
- relates_to [[ANALYSIS-044 Sectioned Content Configuration Patterns]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]