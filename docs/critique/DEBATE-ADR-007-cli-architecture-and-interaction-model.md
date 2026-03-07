---
title: DEBATE-ADR-007 CLI Architecture and Interaction Model
type: note
permalink: critique/debate-adr-007-cli-architecture-and-interaction-model-1
tags:
- debate
- adr-007
- cli
- architecture
- review
---

# DEBATE-ADR-007 CLI Architecture and Interaction Model

**Status:** ACCEPTED (Round 2: Unanimous Accept 6/6)
**Round 1 Verdict:** 6/6 ACCEPT_WITH_CONDITIONS
**Round 2 Verdict:** 6/6 ACCEPT (3 Disagree-and-Commit positions documented)

## Round 1 Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|-----|-----|-----|
| Architect | ACCEPT_WITH_CONDITIONS | 0 | 5 | 4 |
| Critic | ACCEPT_WITH_CONDITIONS | 2 | 6 | 5 |
| Independent Thinker | ACCEPT_WITH_CONDITIONS | 0 | 4 | 5 |
| Security | ACCEPT_WITH_CONDITIONS | 1 | 4 | 3 |
| Analyst | ACCEPT_WITH_CONDITIONS | 1 | 4 | 4 |
| High-Level Advisor | ACCEPT_WITH_CONDITIONS | 1 | 3 | 3 |

## Consolidated P0 Issues (Must Resolve)

### P0-A: MCP Error Response Schema Undefined

**Raised by:** Critic (P0-1), High-Level Advisor (P0-1), Architect (P1-2), Analyst (detailed notes), Security (P1-002 sanitization)

**Consensus:** 5/6 agents flagged this. Decision 4 shows a single inline example `{ error: "missing scope", options: ["global", "project"] }` but defines no schema, no field guarantees, no error codes, no alignment with MCP SDK JSON-RPC conventions. 14+ MCP tools will have inconsistent error shapes without a contract.

**Required resolution:** Define MCP error response Zod schema or explicitly defer to MCP-specific ADR with tracked issue.

### P0-B: --json Output Envelope Undefined

**Raised by:** Critic (P0-2), High-Level Advisor (P1-1), Architect (P2-4 overlap)

**Consensus:** 3/6 agents flagged. --json outputs "JSON data" to stdout but no common envelope, no stability guarantee, no schema. CI consumers and MCP agents depend on predictable structure.

**Required resolution:** Define common JSON envelope or state explicitly that JSON shape is command-specific with no semver stability guarantee.

### P0-C: "12/12" Component Count Mismatch

**Raised by:** Analyst (P0-1), Architect (P1-3)

**Consensus:** The "12/12 components pass on Bun 1.3.8" claim has no traceable source. ANALYSIS-022 reports 16 v0.x APIs tested, 6 v1.0.0 APIs untested. Decision 7 lists 18 component categories. Numbers don't align.

**Required resolution:** Correct the number with verifiable evidence or clarify counting methodology.

### P0-D: `add <source>` Source Argument Has No Validation

**Raised by:** Security (P0-001, Risk 7/10, CVSS 8.1)

**Consensus:** Security-only P0 but critical. Source identifiers are the primary external input vector. CWE-78 command injection if source reaches subprocess calls. Three attack scenarios: shell injection via git URL, path traversal via local path, URL scheme injection.

**Required resolution:** Define source format validation rules or reference ADR that will (could be ADR-004 security policy).

## Consolidated P1 Issues (Resolve or Defer with Issue)

### Flag-Related P1s

| ID | Issue | Raised By | Recommendation |
|----|-------|-----------|----------------|
| P1-flags-1 | Flag interaction matrix (--ci + --quiet, --ci + --verbose) | Architect, Analyst | Add matrix |
| P1-flags-2 | No --color/--no-color / NO_COLOR env var | Architect, Analyst | Add or document exclusion |
| P1-flags-3 | No --dry-run flag | Critic, Ind. Thinker | Defer or add |
| P1-flags-4 | No --version flag | Critic | Document if gunshi auto-generates |
| P1-flags-5 | No --no-ci override for ci-info false positives | Critic | Add escape hatch |

### MCP-Related P1s

| ID | Issue | Raised By | Recommendation |
|----|-------|-----------|----------------|
| P1-mcp-1 | MCP return type inconsistency in pseudocode | Analyst | Fix type signature |
| P1-mcp-2 | MCP error sanitization (CWE-209) | Security | Add requirement |
| P1-mcp-3 | mcp serve lifecycle underspecified | Critic | Define transport, signals |

### ADR Scope/Detail P1s

| ID | Issue | Raised By | Recommendation |
|----|-------|-----------|----------------|
| P1-scope-1 | Extract component mapping (D7) to impl spec | Ind. Thinker, HLA | Move table out of ADR |
| P1-scope-2 | Two-tier (interactive/programmatic) vs three-tier simplification | Ind. Thinker | CI and MCP share input resolution |
| P1-scope-3 | Defer validation rules 4-12 to implementation | Ind. Thinker | Start with 6 format rules |
| P1-scope-4 | Three-tier guard abstraction strategy missing | Critic | Document factory/middleware pattern |

### Security P1s

| ID | Issue | Raised By | Recommendation |
|----|-------|-----------|----------------|
| P1-sec-1 | ReDoS risk in hook matcher regex (Rule 8) | Security | safe-regex2 or glob restriction |
| P1-sec-2 | ADR-004 forward reference tracking | Security | Confirm ADR-004 is next security ADR |
| P1-sec-3 | Credential storage policy for p.password() | Security | Clarify or defer |

### Other P1s

| ID | Issue | Raised By | Recommendation |
|----|-------|-----------|----------------|
| P1-other-1 | Exit codes beyond 0/1 (0=success, 1=runtime, 2=usage) | Critic | Define ranges |
| P1-other-2 | picocolors transitive dep correction | Architect, Analyst | Note v1.1.0 change |
| P1-other-3 | Remove update alias (npm naming confusion) | Ind. Thinker | Pick one name |
| P1-other-4 | Testing matrix acknowledgment | HLA | Add test count formula |

## Consolidated P2 Issues (Document for Future)

1. Accessibility considerations (screen readers, reduced motion)
2. Validation rule registration pattern per command
3. `complete` command not in component mapping
4. --json + MCP mode redundancy
5. Content type extensibility mechanism for `new <type>`
6. --json + no subcommand behavior
7. Help output in MCP mode
8. --quiet + --verbose conflict: error vs silent tiebreaker
9. Interactive fallback discoverability vs --help
10. Three-level nesting usability testing
11. --json + interactive mode interaction
12. --quiet flag ROI (defer to user request)
13. CLI-MCP command sync mechanism (generate from shared Zod schemas)
14. tasks() API evaluation during implementation
15. groupMultiselect and selectKey unmapped
16. --plain flag for grep-parseable output
17. --verbose output boundaries (no secrets)
18. Shell completion sanitization
19. mcp serve process isolation

## Strengths (Consensus)

All 6 agents agreed on these strengths:
1. Three-tier input resolution is architecturally sound (6/6)
2. --ci vs --yes distinction is precise and correct (6/6)
3. ci-info three-layer detection with correct precedence (5/6)
4. Clean command tree with good category grouping (5/6)
5. Strong ADR cross-references and dependency chain (5/6)
6. Reversibility assessment is realistic (4/6)
7. Validation at input time, not write time (4/6)
8. Evidence-based: backed by ANALYSIS-022/023/024 (4/6)

## Observations
- [fact] Round 1: unanimous ACCEPT_WITH_CONDITIONS (6/6) #adr-007 #debate
- [fact] Round 2: unanimous ACCEPT (6/6) — all conditions resolved or D&C #adr-007 #debate
- [fact] 4 consolidated P0 issues resolved: MCP error schema (D9), JSON envelope (D10), component count corrected, source validation (rules 13-16) #p0
- [fact] 8 P1 issues resolved: flag matrix, glob hook matchers, MCP sanitization, exit codes, picocolors note, update alias dropped, testing matrix, p.password() removed #p1
- [fact] 3 Disagree-and-Commit positions from independent thinker: two-tier simplification, defer validation rules, extract D7 — all accepted user's over-specification rationale #d-and-c
- [decision] ADR-007 status: ACCEPTED (Round 2: Unanimous Accept 6/6) #adr-007 #accepted

## Relations

- reviews [[ADR-007 CLI Architecture and Interaction Model]]
- depends_on [[ADR-005 Runtime and Distribution Strategy]]
- depends_on [[ADR-006 CLI Dependency Stack]]
- relates_to [[DEBATE-ADR-005 Runtime and Distribution Strategy]]
- relates_to [[DEBATE-ADR-006 Core Dependency Stack]]


## Round 2 Convergence Results

| Agent | Verdict | Key Notes |
|-------|---------|-----------|
| Architect | ACCEPT | All 3 conditions resolved. No new concerns. |
| Critic | ACCEPT | P0-1, P0-2, P1-1 all resolved (95% confidence). D9/D10 schema asymmetry noted as intentional. |
| Independent Thinker | ACCEPT | P1-4 resolved. 3 D&C: two-tier (user's conceptual clarity argument defensible), defer rules (over-spec creates negotiable ceiling), extract D7 (ADR governance benefit outweighs staleness). |
| Security | ACCEPT | Risk down to 2/10. Source validation closes primary attack surface. Glob eliminates ReDoS. CWE-209 sanitization explicit. |
| Analyst | ACCEPT | Component count corrected accurately. Flag matrix comprehensive. picocolors note prevents false assumptions. |
| High-Level Advisor | ACCEPT | Changed stance on D7: in agent-team codebase, ADR IS the shared truth. Over-specification prevents drift. |

### Disagree-and-Commit Positions (3)

1. **Independent Thinker on two-tier simplification**: CI and MCP share input resolution logic. Three-tier adds code volume. But the MCP behavioral difference (error with options for self-correction) justifies the third tier at protocol level. Commits to three-tier.

2. **Independent Thinker on deferring validation rules**: 16 rules before any code risks premature commitment. But over-specification creates a ceiling implementers can negotiate down with evidence. Cost of changing Zod schemas is low. Commits to 16 rules.

3. **Independent Thinker on extracting D7**: Component mapping reads like an implementation checklist. But in agent-team execution where each teammate starts with zero context, the ADR IS the shared truth. Commits to keeping D7 in ADR.
