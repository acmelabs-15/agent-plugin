---
title: DEBATE-ADR-005 Runtime and Distribution Strategy
type: critique
permalink: critique/debate-adr-005-runtime-and-distribution-strategy
tags:
- critique
- adr-review
- adr-005
- runtime
- bun
- distribution
---

# DEBATE-ADR-005 Runtime and Distribution Strategy

## Review Summary

**ADR**: ADR-005 Runtime and Distribution Strategy
**Date**: 2026-03-07
**Round**: 1
**Status**: ACCEPTED (Round 2: Unanimous Accept 6/6)

## Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|-----|-----|-----|
| Architect | Accept with P1 revisions | 0 | 3 | 3 |
| Critic | Approved | 0 | 2 | 3 |
| Independent Thinker | Concerns (Medium-High) | 0 | 2 | 2 |
| Security | Conditional | 1 | 3 | 0 |
| Analyst | Concerns | 1 | 2 | 0 |
| Advisor | Accept with conditions | 0 | 2 | 0 |

**Consensus on decision**: All 6 agents agree Bun is the correct runtime choice. No agent recommends Node.js or Deno. Issues are about ADR quality, not the decision itself.

## P0 Issues

### P0-1: Compiled Binary Startup Claim Misleading

**Raised by**: Analyst (P0), Critic (P1), Independent Thinker (P1)
**Description**: ADR headline claims 8-15ms cold start and sets sub-20ms confirmation target. This applies to `bun run`, NOT compiled binaries. Tigris CLI case study (ANALYSIS-016 Section 4.2) measured compiled binary at 104ms vs Node.js at 64ms — compiled binary was SLOWER. Evan You benchmarks confirm compiled binaries are 50-100ms+ without `--bytecode` flag.
**Impact**: Confirmation target "cold start < 20ms" may be unachievable for compiled binary distribution path. POS-001 overstates benefit.
**Resolution**: Scope startup claim to `bun run`/`bunx` path. Differentiate compiled binary startup. Update confirmation criteria.

### P0-2: No Binary Integrity Verification

**Raised by**: Security (P0)
**Description**: 50-100 MB compiled binaries distributed via GitHub Releases with zero integrity verification — no checksums, no code signing, no Sigstore attestation. CWE-494. Users cannot distinguish legitimate binaries from tampered ones.
**Impact**: All users of the secondary (binary) distribution channel.
**Assessment**: Valid security concern but scoped to release pipeline design, not runtime selection. Recommend adding as IMP note in ADR-005, full treatment in release engineering or ADR-004.

## P1 Issues

### P1-1: MADR 4.0 Structural Compliance

**Raised by**: Architect
**Description**: Missing `decision-makers`, `consulted`, `informed` frontmatter fields (uses `authors`). Missing standalone `## Considered Options` section. Missing `## Decision Outcome` with "Chosen option: X, because Y" format.
**Resolution**: Fix MADR structure.

### P1-2: @clack/prompts Test Gate with Deadline

**Raised by**: Advisor (P1), Critic (P1), Analyst (P1), Independent Thinker (P2)
**Description**: Three documented Bun stdin issues (#4835, #3099, #7033). Analyst found NEW regression in Bun 1.3.2 (issue #24615: EPERM stdin error). NEG-001 mitigation says "test early" but defines no deadline or go/no-go gate. If @clack/prompts fails Bun testing, all interactive flows are dead.
**Resolution**: Add blocking gate: "Test @clack/prompts before any interactive flow implementation. If fails, select alternative before proceeding."

### P1-3: MCP Server Runtime Unaddressed

**Raised by**: Independent Thinker (P1), Analyst (mentioned)
**Description**: ADR evaluates only CLI cold start. The embedded MCP server is a long-running process where startup is irrelevant — memory, stability under load, GC behavior matter instead. This is half the architecture with zero analysis.
**Resolution**: Add paragraph noting MCP server has different performance profile; startup benchmarks do not apply to long-running MCP process.

### P1-4: Node.js Consumer API Compatibility Gap

**Raised by**: Analyst (P1)
**Description**: ADR commits to Bun-specific APIs (Bun.file, Bun.Glob, Bun.spawn) AND npm distribution for Node.js consumers. These are contradictory without an abstraction layer. ANALYSIS-016 recommends this as P1 but ADR does not include it.
**Resolution**: Add IMP note requiring abstraction layer for Bun-specific APIs to support npx/Node.js consumers.

### P1-5: Cold Start CI Enforcement

**Raised by**: Advisor (P1)
**Description**: Sub-20ms target without automated CI enforcement is a wish, not a constraint. Without a benchmark that fails the build on regression, the target will drift within 3 months.
**Resolution**: Add IMP note for CI benchmark step.

## P2 Issues

| # | Issue | Raised By |
|---|-------|-----------|
| P2-1 | Anthropic acquisition risk not in NEG items (only in exit strategy) | Architect |
| P2-2 | Binary size lacks comparison context (Go 5-15MB, Rust 2-10MB) | Critic, Independent Thinker, Analyst |
| P2-3 | Frontmatter `type: note` should be `type: decision` | Critic |
| P2-4 | Missing relation to ADR-003 | Critic |
| P2-5 | gunshi "primary runtime" claim unsupported — should be "supports Bun" | Analyst |
| P2-6 | Node.js `--strip-types` (stable in 23.6+) not re-evaluated | Independent Thinker |

## Scope Decision

**Split recommended?** No (4 agents say keep together, 2 suggest possible split)
**Rationale**: Runtime and distribution are tightly coupled — Bun's `bun build --compile` directly enables the binary distribution path. Splitting would create two ADRs that must always be read together.

## Key Evidence Discovered During Review

- [fact] Tigris CLI compiled binary: 104ms startup vs Node.js 64ms on M4 Max — compiled binary SLOWER #benchmark
- [fact] Evan You benchmark: Node.js SEA faster than `bun --compile` without `--bytecode` flag #benchmark
- [fact] Bun 1.3.2 introduced NEW EPERM stdin regression with @clack/prompts (issue #24615) #regression
- [fact] Bun 1.3.9 with `--bytecode` flag is 25% faster than Node SEA #benchmark
- [fact] Node.js 23.6+ `--strip-types` may eliminate build-step argument against Node.js #node-evolution

## Observations

- [fact] All 6 agents agree Bun is the correct runtime choice for this project #consensus
- [fact] Primary concern is ADR documentation quality, not the decision itself #quality
- [decision] P0-1 requires scoping startup claim to bun run path, not compiled binaries #correction
- [risk] @clack/prompts Bun compatibility has a NEW regression in 1.3.2, worse than previously known #dependency-risk
- [insight] MCP server runtime performance (long-running process) is a gap in the analysis — startup benchmarks don't apply #gap

## Relations

- reviews [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ANALYSIS-016-bun-runtime-assessment]]
- relates_to [[DEBATE-ADR-001-plugin-format-and-manifest]]
- relates_to [[DEBATE-ADR-002-target-platforms-and-audiences]]
- relates_to [[DEBATE-ADR-003-conflict-resolution-and-namespacing]]

## Round 2: Convergence Check

**Date**: 2026-03-07
**Status**: CONSENSUS REACHED — UNANIMOUS ACCEPT (6/6)

### Changes Since Round 1

- P0-1: Startup claims scoped to bun run/bunx; compiled binary 50-100ms+ documented
- P0-2: IMP-007 added (SHA-256 checksums + code signing, CWE-494)
- P1-1: MADR frontmatter fields added
- P1-2: IMP-008 BLOCKING GATE for @clack/prompts with pass/fail criteria
- P1-3: IMP-009 MCP server long-running process benchmarking
- P1-4: Bun-only distribution — all Node.js/npx references removed
- P1-5: IMP-011 CI benchmark (50ms fail threshold)
- P2-3/P2-4: Frontmatter type fixed, ADR-003 relation added

### Agent Verdicts (Round 2)

| Agent | Verdict | Concerns |
|-------|---------|----------|
| Architect | **Accept** | Cosmetic IMP numbering gap (IMP-004 to IMP-006) |
| Critic | **Accept** | Cosmetic NEG numbering gap (NEG-003 to NEG-005) |
| Independent Thinker | **Accept** | None |
| Security | **Accept** | npm provenance (SLSA) belongs in release engineering, not this ADR |
| Analyst | **Accept** | Cosmetic numbering gaps only |
| High-Level Advisor | **Accept** | None |

### Consensus

All 6 agents voted Accept. No blocks, no D&C positions. ADR-005 is ready for implementation.

### P2 Notes for Implementation

- Cosmetic numbering gaps (NEG-004, IMP-005) are artifacts of Bun-only revision. Not worth renumbering.
- npm package provenance attestations (SLSA Level 1+) should be tracked as a future item when release pipeline is built.