---
title: CRIT-003 ADR-003 Round 2 Convergence Review
type: critique
permalink: critique/crit-003-adr-003-round-2-convergence-review
tags:
- adr-review
- critique
- ADR-003
- convergence
- round-2
---

# CRIT-003 ADR-003 Round 2 Convergence Review

## Verdict

**Position**: Accept

**Confidence**: High

**Rationale**: All 7 P0 issues from Round 1 are resolved. The always-namespace decision eliminated 4 P0s as moot (P0-2, P0-3, P0-4 and the core always-namespace gap). The remaining 3 P0s received substantive resolutions with concrete specifications: colon-is-logical-only with kebab-case regex (P0-5), overlay/recompute with strictest-wins via deepmerge customMerge (P0-6), and JSON lockfile at .agent-plugins/plugin-lock.json with 3-layer corruption recovery (P0-7). All 14 P1 issues are resolved or rendered moot. The ADR is implementation-ready.

## Summary

The rewrite is a fundamentally different document from Round 1. The original ADR-003 attempted interactive conflict resolution with rename tracking and cross-reference updates, creating 3 load-bearing subsystems with undefined scope. The rewrite adopts always-namespace, which collapses those subsystems to zero. This is the correct architectural choice. The 6 decisions are tightly coupled (namespacing determines hook merge requirements, which determines state tracking needs), justifying a single ADR rather than splitting.

## Strengths

- Always-namespace eliminates 3 subsystems (conflict resolution UI, rename tracking, cross-reference updates) that drove 4 of 7 P0 issues in Round 1
- Every decision references a specific analysis note (ANALYSIS-010 through ANALYSIS-014), creating a verifiable evidence chain
- Overlay/recompute for hooks is backed by 12 production system precedents documented in ANALYSIS-010
- The 4-level platform config resolution order has concrete examples at each level with estimated usage distribution (80/15/5)
- Reversibility assessment covers all 5 required items
- Confirmation section specifies 5 testable verification methods (unit, property-based, integration, schema, shell-quote)
- Implementation notes are specific enough to code against (IMP-001 through IMP-010 include regex patterns, file paths, package names, function signatures)
- Options section includes the rejected alternatives with specific reasons tied to project constraints (CI/Docker/MCP from ADR-002)

## Issues Found

### Critical (Must Fix)

None.

### Important (Should Fix)

- [ ] **P1-NEW-1: ADR-001 Section 9 alignment drift.** ADR-001 Section 9 "Intelligent Conflict Resolution" still describes "Namespace prefixing or user-prompted rename on conflict" for skills and "Strategy details TBD" for other types. ADR-003 supersedes this with always-namespace. ADR-001 should be updated to reference ADR-003 for conflict resolution, or Section 9 should note it is superseded. This is not a blocker for ADR-003 acceptance but creates contradictory documentation.

- [ ] **P1-NEW-2: State directory path inconsistency.** ADR-003 IMP-003 specifies hook overlays at `~/.agent-plugins/state/hooks/{platform}/{plugin-name}.json`. ADR-003 IMP-005 specifies the lockfile at `.agent-plugins/plugin-lock.json` (project-relative). ANALYSIS-010 uses `~/.agent-plugin/state/hooks/` (singular, home-relative). Three different base paths across two documents. Recommend: pick one base path convention and document the rationale (project-scoped vs user-scoped). This does not block acceptance because implementation notes are guidance, not binding specification.

- [ ] **P1-NEW-3: deepmerge customMerge field-matching scope.** IMP-004 states boolean fields "named `blocking` or `deny`" use logical OR. Claude Code hook schema uses no field named `blocking` or `deny`. The blocking semantics come from the hook type and the platform runtime evaluating return values, not from a boolean field in the configuration. The customMerge description should clarify which specific JSON keys in which platform schemas trigger strictest-wins, or defer the field mapping to platform adapter implementation. Low risk because the platform runtime handles strictest-wins regardless of our merge behavior (as noted in ANALYSIS-010 Section 5 RQ4).

### Minor (Consider)

- [ ] **P2-NEW-1: NEG-004 medium confidence.** NEG-004 states "Medium confidence" for prompt injection detection. This is an honest assessment but lacks a mitigation plan. Consider adding: "ADR-004 will address prompt injection detection strategy with higher-confidence mitigations."

- [ ] **P2-NEW-2: Overlay file naming for scoped packages.** IMP-003 uses `{plugin-name}.json` as the overlay filename. Plugin names can be scoped (`@scope/plugin-name` per ADR-001). The slash in scoped names creates subdirectories on disk. Consider documenting how scoped names map to overlay filenames (e.g., `scope__plugin-name.json`).

- [ ] **P2-NEW-3: deepmerge staleness acknowledgment.** NEG-006 correctly flags deepmerge v4.3.1 as 3 years without a release and names deepmerge-ts as the alternative. Consider adding deepmerge-ts as the primary recommendation with deepmerge as fallback, given deepmerge-ts has native TypeScript, active maintenance, and the project is TypeScript-first.

## Questions for Planner

None. The ADR is sufficiently specified for implementation.

## Recommendations

1. Update ADR-001 Section 9 to reference ADR-003 for the authoritative conflict resolution strategy. This can happen in a separate commit.
2. Resolve the state directory path inconsistency (IMP-003 vs IMP-005 vs ANALYSIS-010) during implementation planning. The decision between project-scoped and user-scoped state has implications for monorepos and shared workstations.
3. Clarify the customMerge field mapping in IMP-004 or explicitly defer it to platform adapter implementation.

## Approval Conditions

None. The ADR is approved as written. The P1 issues above are documentation hygiene items that do not affect the architectural decisions. They can be addressed in subsequent commits without blocking implementation.

## Round 1 Resolution Verification

| Round 1 Issue | Status | Verification |
|---------------|--------|--------------|
| P0-1: ADR overloaded | [PASS] Kept as one ADR. Decisions are tightly coupled. Correct choice. | Verified: namespacing determines hook merge needs, which determines lockfile scope |
| P0-2: No non-interactive resolution | [PASS] MOOT. Always-namespace is automatic. | Verified: no user prompts in any decision |
| P0-3: Cross-reference scope undefined | [PASS] MOOT. No renames. | Verified: no rename tracking in any decision |
| P0-4: installMode interaction | [PASS] MOOT. Always-namespace works for both. | Verified: Decision 1 states "automatic and unconditional" |
| P0-5: Colon in filenames + validation | [PASS] Colon is logical-only. Kebab-case regex specified. | Verified: IMP-001, IMP-002 |
| P0-6: Hook merge semantics | [PASS] Overlay/recompute with deepmerge customMerge. Strictest-wins for blocking. | Verified: Decision 2, IMP-004 |
| P0-7: State store undefined | [PASS] JSON lockfile, atomically, 3-layer recovery, SHA-256 integrity. | Verified: Decision 3, IMP-005, IMP-006 |
| P1-1: Always-namespace not evaluated | [PASS] ADOPTED as Decision 1. | Verified |
| P1-2: Hook merge ordering fragile | [PASS] Alphabetical sort, order-independent. | Verified: Decision 2 |
| P1-3: Colon escaping specifics | [PASS] Covered by P0-5. | Verified |
| P1-4: platformConfig no schema | [PASS] 4-level resolution with 3 cross-platform concepts. | Verified: Decision 4 |
| P1-5: Rename tracking | [PASS] MOOT. | Verified |
| P1-6: Reversibility | [PASS] Reversibility assessment section added. | Verified: 5 items checked |
| P1-7: Cross-reference injection | [PASS] 6-layer sanitization pipeline. | Verified: Decision 5 |
| P1-8: State store tampering | [PASS] Re-derive from disk on corruption. | Verified: Decision 3 |
| P1-9: Frontmatter injection | [PASS] Security-critical fields never from manifests. | Verified: Decision 5 final paragraph |
| P1-10: Scale analysis | [PASS] MOOT. No prompts. | Verified |
| P1-11: platformConfig location | [PASS] 4-level resolution order. | Verified: Decision 4 table |
| P1-12: Rollback on rename failure | [PASS] MOOT. No renames. | Verified |
| P1-13: Component vs plugin.json precedence | [PASS] 4-level resolution order. | Verified: Decision 4 |
| P1-14: ADR-004 forward reference | [PASS] Decision 6. | Verified |

**All 21 issues from Round 1: 21/21 resolved.** Zero regressions.

## Observations

- [decision] ADR-003 Round 2 accepted after full rewrite resolving all 21 Round 1 issues #convergence #adr-review
- [fact] Always-namespace eliminated 4 of 7 P0 issues by making conflict resolution moot #simplification
- [fact] 3 new P1 issues identified (ADR-001 alignment, state path inconsistency, customMerge field scope) -- none blocking #hygiene
- [insight] The rewrite demonstrates that choosing the right abstraction (always-namespace vs interactive resolution) can eliminate entire categories of complexity #architecture
- [risk] State directory paths are inconsistent across ADR-003 and ANALYSIS-010; must resolve during implementation planning #consistency

## Relations

- part_of [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[DEBATE-ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-002 Target Platforms and Audiences]]
- relates_to [[ANALYSIS-010-hook-merge-unmerge-patterns]]
- relates_to [[ANALYSIS-012-json-config-merge-patterns]]
- relates_to [[ANALYSIS-013-input-sanitization-patterns]]
- relates_to [[ANALYSIS-014-platform-config-patterns]]
