---
title: DEBATE-ADR-003 Conflict Resolution and Namespacing
type: critique
permalink: critique/debate-adr-003-conflict-resolution-and-namespacing
tags:
- adr-review
- debate
- ADR-003
- conflict-resolution
- namespacing
---

# DEBATE-ADR-003 Conflict Resolution and Namespacing

**ADR**: [[ADR-003-conflict-resolution-and-namespacing]]
**Date**: 2026-03-07
**Round**: 2
**Result**: CONSENSUS REACHED — 3 Accept, 3 Disagree-and-Commit (Round 2)

## Agent Verdicts

### Round 1

| Agent | Verdict | Confidence |
|-------|---------|------------|
| Architect | Needs Revision | High |
| Critic | Needs Revision | High |
| Independent Thinker | Needs Revision | High |
| Security | Needs Revision | High |
| Analyst | Needs Revision | High |
| High-Level Advisor | Needs Revision | High |

### Round 2 (Convergence Check on Rewritten ADR)

| Agent | Verdict | Confidence |
|-------|---------|------------|
| Architect | Accept | High |
| Critic | Accept | High |
| Independent Thinker | Disagree-and-Commit | High |
| Security | Disagree-and-Commit | High |
| Analyst | Disagree-and-Commit | High |
| High-Level Advisor | Accept | High |

**Result**: CONSENSUS REACHED — 3 Accept, 3 Disagree-and-Commit (6/6)

## Consolidated P0 Issues

### P0-1: ADR Is Overloaded — Split Into 2-3 ADRs (High-Level Advisor, Independent Thinker, Analyst)

Decisions 1-3 (colon separator, conflict resolution, rename tracking) are tightly coupled. Decisions 4-5 (cross-platform frontmatter, platform-aware generation) are a separate concern with different stakeholders and change frequency. Recommend split:

- ADR-003A: Namespacing and Conflict Resolution (decisions 1, 2, 3)
- ADR-003B: Cross-Platform Component Format (decisions 4, 5)

### P0-2: No Non-Interactive Conflict Resolution for CI/MCP (Critic, High-Level Advisor, Independent Thinker)

Interactive prompts fail for: CI pipelines, Docker builds, AI MCP server audience, scripted installs. ADR-002 defines 3 audiences but ADR-003 only designs for terminal users. Need a --strategy flag or config default: error (fail on conflict), prefix-incoming (always namespace incoming), prefix-existing, interactive (current).

### P0-3: Cross-Reference Update Scope Undefined (Architect, Critic, High-Level Advisor, Analyst)

"Cross-references within the affected plugin's files are updated automatically" but no definition of what constitutes a cross-reference. Frontmatter requires? Wikilinks? Hook configs? Prompt template interpolations? AST-aware parsing across YAML+Markdown+JSON has no off-the-shelf solution. Analyst cites ACM TOSEM 2024: even purpose-built AST tools lack multi-mapping support.

### P0-4: installMode Interaction Unspecified (Architect, Critic)

ADR-001 defines bundle (all-or-nothing) and collection (pick components). ADR-003 never addresses: if 1 of 15 bundle components conflicts, does the entire bundle fail or just that component? Does conflict detection run before or after collection selection? Renaming inside a bundle breaks internal cross-references.

### P0-5: Colon Must Be Logical-Only + Name Validation Required (Analyst, Security)

Analyst evidence: Windows NTFS forbids colons in filenames. macOS Finder replaces colons. Claude Code uses colon as DISPLAY convention, not in filenames on disk. YAML parsers treat colons as key-value separators. ADR must explicitly state: colon is a logical namespace delimiter only, never in filenames. Security adds: plugin and component names must be validated as kebab-case (no colons allowed in names) to prevent namespace spoofing.

### P0-6: Hook Merge Semantics Undefined for Blocking Events (Analyst, Security)

Hooks "merge" but Claude Code has blocking hooks (PreToolUse can block tool execution). Two merged hooks disagreeing (one blocks, one allows) has no resolution rule. Security flags: auto-merge without per-hook user consent enables malicious code execution (CWE-94, 8/10 severity). Need per-event-type merge rules distinguishing advisory hooks (can merge) from control hooks (need resolution rule).

### P0-7: State Store Format/Location/Failure Modes Undefined (Architect, Independent Thinker, Critic)

"Likely in a JSON file" is unacceptable for an accepted ADR. State store is load-bearing: updates, removals, cross-references all depend on it. Missing: format specification, location, behavior on corruption/deletion, concurrent access handling, drift detection. Analyst notes ANALYSIS-003 recommends SQLite, contradicting JSON.

## Consolidated P1 Issues

| ID | Issue | Raised By |
|----|-------|-----------|
| P1-1 | Always-namespace (Claude Code model) not evaluated as alternative — eliminates decisions 2 and 3 entirely | Independent Thinker, High-Level Advisor |
| P1-2 | Hook merge ordering: installation order is fragile, breaks on reinstall | Architect, High-Level Advisor, Critic |
| P1-3 | Colon escaping: PowerShell drive separators, YAML key-value, URL protocols — enumerate specifics | Architect, Analyst, Critic |
| P1-4 | platformConfig has no schema, no concrete example, no override precedence rules | High-Level Advisor, Analyst, Critic |
| P1-5 | Rename tracking: no upstream rename reconciliation, no bidirectional de-namespace | Analyst, Critic |
| P1-6 | No reversibility assessment — migration cost if colon separator decision is reversed | Architect |
| P1-7 | Cross-reference injection: replacement values are author-controlled, need sanitization | Security |
| P1-8 | State store tampering: no integrity protection, no file permissions, no checksums | Security |
| P1-9 | Frontmatter injection via platformConfig: allowed-tools and permissionMode are security-critical | Security |
| P1-10 | No scale analysis: 50-component plugin with 8 conflicts = 8 sequential prompts, no batch mode | Critic |
| P1-11 | platformConfig location ambiguous: plugin.json, component frontmatter, or both? | Critic |
| P1-12 | No rollback mechanism if rename + cross-ref update fails partway | Security |
| P1-13 | Component frontmatter overlaps with plugin.json fields, no precedence defined | High-Level Advisor, Analyst, Critic |
| P1-14 | Missing forward reference to ADR-004 for hook merge security | Critic |

## Consolidated P2 Issues

| ID | Issue | Raised By |
|----|-------|-----------|
| P2-1 | Duplicate frontmatter blocks in document | Architect, Critic |
| P2-2 | Alternatives not mapped to specific decisions | Architect |
| P2-3 | No sequence diagram for conflict resolution flow | Architect |
| P2-4 | ALT numbering inconsistent (description and rejection share same prefix) | Critic |
| P2-5 | Prompt fatigue: 100-component plugin with many collisions = usability attack | Security |
| P2-6 | platformConfig validation timing (author time, install time, or both?) | Security |
| P2-7 | No precedent analysis acknowledgment — per-type conflict resolution is novel | Critic |
| P2-8 | No confirmation/verification method specified | Architect |

## Strengths Identified

- Per-type conflict resolution (hooks additive, names collision) is correct and novel — no competitor does this (all 6 agents)
- Rename tracking solves a real problem for update/removal correctness (Architect, Independent Thinker, Analyst)
- Colon separator borrows from proven Claude Code convention (Architect, Analyst, Critic)
- Platform-aware frontmatter stripping reduces attack surface vs superset emission (Architect, Security)
- User agency in conflict resolution avoids surprising automated behavior (Architect, Critic)
- Cross-platform frontmatter layering (core + platformConfig + emission filtering) is correct abstraction (Architect, Critic)

## Resolution Recommendations

### All P0 Issues -- RESOLVED

| P0 | Issue | Resolution |
|----|-------|------------|
| Core | Always-namespace not evaluated | ADOPTED -- always-prefix with plugin:component (Claude Code model). Eliminates decisions 2+3. |
| P0-1 | ADR overloaded, split | KEEP as one ADR -- concerns are tightly coupled |
| P0-2 | No non-interactive conflict resolution | MOOT -- always-namespace is automatic, no user prompts |
| P0-3 | Cross-reference update scope | MOOT -- no renames with always-namespace |
| P0-4 | installMode interaction | MOOT -- always-namespace works for both bundle and collection |
| P0-5 | Colon in filenames + name validation | RESOLVED -- colon is logical-only (never in filenames), kebab-case validation for plugin/component names |
| P0-6 | Hook merge semantics | RESOLVED -- strictest wins for blocking hooks, overlay/recompute pattern (ANALYSIS-010), deepmerge with customMerge (ANALYSIS-012) |
| P0-7 | State store format/location | RESOLVED -- JSON lockfile at project root (plugin-lock.json) or user home (~/.config/agent-plugin/), hook overlays in lockfile sections, re-derive on corruption, atomically for writes (ANALYSIS-011) |

### All P1 Issues -- RESOLVED

| P1 | Issue | Resolution |
|----|-------|------------|
| P1-1 | Always-namespace not evaluated | ADOPTED as core decision |
| P1-2 | Hook merge ordering fragile | RESOLVED -- overlay/recompute pattern, order-independent (ANALYSIS-010) |
| P1-3 | Colon escaping specifics | COVERED by P0-5 (logical-only) |
| P1-4 | platformConfig no schema | RESOLVED -- hybrid D+C pattern with cross-platform concepts (loadingStrategy, filePatterns, approvedTools) + per-platform blocks (ANALYSIS-014) |
| P1-5 | Rename tracking | MOOT -- no renames with always-namespace |
| P1-6 | Reversibility of colon | COVERED -- logical-only, low migration cost |
| P1-7 | Cross-reference injection | COVERED -- ANALYSIS-009 sanitization mandate + ANALYSIS-013 (Zod v4 + shell-quote + validator) |
| P1-8 | State store tampering | COVERED by P0-7 (re-derive from disk) |
| P1-9 | Frontmatter injection via platformConfig | COVERED -- ANALYSIS-013 layered validation pipeline |
| P1-10 | Scale analysis (50 components) | MOOT -- no prompts with always-namespace |
| P1-11 | platformConfig location ambiguous | RESOLVED -- 4-level resolution: standard fields → adapter → plugin.json → per-component (ANALYSIS-014) |
| P1-12 | Rollback on rename failure | MOOT -- no renames |
| P1-13 | Component vs plugin.json precedence | RESOLVED -- 4-level resolution order (ANALYSIS-014) |
| P1-14 | Missing ADR-004 forward ref | RESOLVED -- forward reference to ADR-004 for hook security |

### ADR-003 Rewrite Required

The ADR must be rewritten to incorporate:

1. Always-namespace as the core decision (replaces user-choice conflict resolution)
2. Remove decisions 2+3 (rename tracking, cross-reference updates) -- moot
3. Colon-is-logical-only constraint + kebab-case name validation
4. Overlay/recompute pattern for hook merge with deepmerge engine
5. JSON lockfile with atomically, re-derive on corruption
6. Hybrid D+C platformConfig pattern with 4-level resolution
7. Sanitization pipeline reference (ANALYSIS-013)
8. Forward reference to ADR-004 for hook merge security
9. Updated consequences and alternatives sections

### Structural

1. Split ADR into ADR-003A (Namespacing + Conflict Resolution) and ADR-003B (Component Format)
2. Fix duplicate frontmatter block

### Must Address (P0)

1. Add non-interactive conflict resolution strategy with --strategy flag
2. Define cross-reference scope explicitly (which formats, which fields)
3. Specify installMode interaction for both bundle and collection modes
4. State colon is logical-only, add kebab-case name validation constraint
5. Define hook merge rules per event type (advisory vs blocking)
6. Specify state store format, location, and failure modes

### Should Address (P1)

1. Evaluate always-namespace as a serious alternative (may simplify dramatically)
2. Add concrete platformConfig schema with example
3. Add ADR-004 forward reference for security concerns

## Observations

- [fact] All 6 agents unanimously voted Needs Revision with High confidence #consensus
- [fact] 7 P0 issues, 14 P1 issues, 8 P2 issues identified across all agents #scope
- [insight] Always-namespace (Claude Code model) could eliminate decisions 2 and 3, dramatically simplifying the ADR #simplification
- [insight] Colon is display-only in Claude Code, never in filenames — ADR must match this pattern #evidence
- [risk] Hook auto-merge without consent enables malicious code execution (CWE-94, 8/10 severity) #security
- [decision] Non-interactive conflict resolution is mandatory for CI/MCP audiences #three-audiences

## Relations

- part_of [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-002-target-platforms-and-audiences]]
- relates_to [[DEBATE-ADR-001-plugin-format-and-manifest]]
- relates_to [[DEBATE-ADR-002-target-platforms-and-audiences]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]

## Round 2: New Issues (Non-Blocking)

### P1 Issues (6 unique)

| ID | Issue | Raised By | Resolution Path |
|----|-------|-----------|-----------------|
| P1-R2-1 | ADR-001 Section 9 contradicts ADR-003 always-namespace — needs update to reference ADR-003 as authoritative | Critic, High-Level Advisor | Update ADR-001 Section 9 |
| P1-R2-2 | State directory path inconsistent: project-relative `.agent-plugins/` (ADR) vs user-home `~/.agent-plugin/` (ANALYSIS-010/011) | Independent Thinker, Critic | Resolve during implementation planning |
| P1-R2-3 | `deepmerge` customMerge dispatch by field name (`blocking`, `deny`) should be path-context-aware, not global name matching | Architect, Critic | Tighten IMP-004 during implementation |
| P1-R2-4 | Strictest-wins makes plugins non-composable when security postures conflict — restrictive plugin can break permissive plugin | Independent Thinker | Document as NEG-007, ADR-004 addresses mitigation |
| P1-R2-5 | Overlay file permissions unspecified (CWE-732) — state directory needs mode 700, files mode 600 | Security | Add IMP note for file permissions |
| P1-R2-6 | No explicit prohibition on hook execution before ADR-004 consent model (CWE-862) — syntactically safe but semantically malicious commands could execute | Security | Add constraint: hooks stored but not executable until ADR-004 |

### P2 Issues (7 unique)

| ID | Issue | Raised By |
|----|-------|-----------|
| P2-R2-1 | atomically has 2 same-author deps (stubborn-fs, when-exit), not "0 third-party deps" | Analyst |
| P2-R2-2 | 80/15/5 platformConfig split is hypothesis, not measured (zero plugins exist) | Independent Thinker, Analyst |
| P2-R2-3 | deepmerge download count (64.1M/week) is point-in-time data, current varies | Analyst |
| P2-R2-4 | No migration trigger defined for deepmerge→deepmerge-ts | Independent Thinker, Analyst |
| P2-R2-5 | Glob patterns (`*`, `?`) missing from hook command rejection list in IMP-009 | Security |
| P2-R2-6 | ADR status says "Accepted" before Round 2 review concludes | High-Level Advisor |
| P2-R2-7 | Re-derive-from-disk fails if overlay state directory also lost (double corruption) | Independent Thinker |

### Dissent Record

**Independent Thinker**: Commits despite P1-R2-4 (composability) and P1-R2-2 (path inconsistency). Neither requires architectural redesign. Both addressable with targeted edits.

**Security**: Commits despite P1-R2-5 (overlay permissions) and P1-R2-6 (hook execution gap). Both addressable with single-sentence additions to implementation notes. CVSS 5.3 for hook execution gap (CWE-862).

**Analyst**: Commits despite 4 P2 accuracy issues. All are documentation corrections, not architectural concerns.

## Round 2 Consensus

**Status**: CONSENSUS REACHED
**Round**: 2 of 10
**Verdict**: ADR-003 ACCEPTED with recorded dissent
**P0 remaining**: 0
**P1 remaining**: 6 (all non-blocking, addressable with targeted edits)
**P2 remaining**: 7 (documentation accuracy)
