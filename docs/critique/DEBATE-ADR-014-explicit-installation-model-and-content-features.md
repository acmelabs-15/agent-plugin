---
title: DEBATE-ADR-014 Explicit Installation Model and Content Features
type: critique
permalink: critique/debate-adr-014-explicit-installation-model-and-content-features-2
tags:
- adr-review
- debate
- installation
- features
- lockfile
- ADR-014
- agent-plugin
---

# DEBATE-ADR-014 Explicit Installation Model and Content Features

## ADR Under Review

ADR-014 Explicit Installation Model and Content Features (785 lines, 8 decisions)

Supersedes ADR-013. Replaces postinstall auto-wiring with explicit `agent-plugin add/remove/update` commands. Introduces features model, `.agent-lock.json` lockfile, 8 content types, and `.agent-plugin/plugin.json` manifest location.

## Round 1 Results

**Outcome**: 4 P0 issues (reclassified to 1 P0 + 3 P1), 10 P1 issues. No agent accepted as-is.

Round 1: 0 Accept, 0 D&C, 6 Needs Revision, 0 Block

### Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | NEEDS REV | 1 | 3 | 2 |
| Critic | NEEDS REV | 2 | 5 | 4 |
| Independent Thinker | NEEDS REV | 1 | 5 | 0 |
| Security | NEEDS REV | 1 | 4 | 0 |
| Analyst | NEEDS REV | 0 | 3 | 3 |
| High-Level Advisor | NEEDS REV | 1 | 4 | 0 |

## P0 Issues (Consolidated, Post-Reclassification)

### P0-4: Lockfile Lacks Version/Ref for Deterministic Reproduction (Critic P0, Architect P1)

- `.agent-lock.json` schema stores `source`, `sourceType`, and `hash` but no `version` (npm) or `ref` (git)
- The `hash` field is a content hash, not a version identifier. You cannot pass a SHA-256 hash to npm registry or git to fetch a specific version.
- Without version/ref, `agent-plugin install` (team sync) cannot reproduce the exact installation state
- The lockfile's stated purpose (deterministic team synchronization) is unfulfillable without addressing information

**[RESOLVED] Added `version` and `ref` fields to lockfile schema.**

Updated schema adds:
- `version` (string | null): Resolved npm semver (e.g., "1.2.3"). Null for local sources.
- `ref` (string | null): Resolved git commit SHA. Null for npm and local sources.
- `hash` scope clarified: SHA-256 of all content files listed in manifest, computed deterministically (sorted file paths, concatenated content).

### P0-1: Hook Merge Strategy Undefined (Architect P0, Critic P0) — RECLASSIFIED TO P1

ADR-003 Decision 2 defined overlay/recompute hook merging from `plugin-lock.json`. ADR-013 defined full-recompute from `node_modules`. ADR-014 supersedes both without specifying hook merge behavior.

**Reclassification rationale** (High-Level Advisor tie-breaker): ADR-014 uses file-based installation (direct writes to platform configs). Each plugin's hooks are independently installed as separate entries namespaced by plugin name. No cross-plugin hook merging needed. This differs from ADR-003/013's merge model but is simpler.

**[RESOLVED] Hook merge semantics added to Decision 7 Step 7.**

Hooks from multiple plugins are installed as independent entries. No overlay/recompute. Platform adapters that require a single hook entry point add one entry per plugin hook.

### P0-2: MCP Command Execution Without Confirmation (Security P0) — RECLASSIFIED TO P2

Security flagged that `mcpServers` entries specify arbitrary commands written to platform configs with no validation or user confirmation. Risk Score 8/10 (CWE-78/CWE-94).

**Reclassification rationale** (High-Level Advisor): MCP server behavior is inherited from ADR-011 and not modified by ADR-014. The concern belongs on ADR-011, not here.

**Partial resolution**: IMP-001 security constraints updated to require displaying exact `command` and `args` in installation summary and requiring explicit user confirmation for MCP entries. Command allowlist recommended.

### P0-3: Pivot Rationale Contains Factual Error (Independent Thinker P0) — RECLASSIFIED TO P1

ADR-014 stated: "The npm-only model requires every plugin to be published as an npm package." This is factually incorrect. ADR-013 Decision 1 explicitly supports `github:owner/repo`, `git+https://`, and `file:../` via bun.

**Reclassification rationale** (High-Level Advisor): Factual error weakens one of three pivot drivers but does not invalidate the pivot. The other two drivers (implicit wiring opacity, no feature selection) are legitimate and sufficient.

**[RESOLVED] Pivot driver #3 rewritten.**

Changed from "npm-only" claim to accurate description: ADR-013 delegates source resolution to bun (requiring protocol prefixes like `github:`, `file:`), while ADR-014 handles resolution directly with simpler syntax (bare `owner/repo`) and without creating npm dependency entries. All 4 instances corrected (context, Option A cons, POS-004, More Information section).

## P1 Issues (Consolidated, Deduplicated)

1. **CLI content type is a placeholder** (Architect, Critic, Advisor, Analyst) — Listed as 8th content type with no schema, directory, or behavior. **[RESOLVED]** Marked as deferred. Listed for taxonomy completeness; schema/behavior will be defined in future ADR if demand emerges.

2. **Source resolution security gaps** (Architect, Critic, Security) — ADR-008 covered CWE-494, CWE-22, registry spoofing in detail. ADR-014 reinstates custom resolution with only a one-line CWE-22 mention. **[RESOLVED]** Expanded IMP-001 with comprehensive security constraints: CWE-22 path validation for all 8+ path surfaces, CWE-494 tarball integrity, CWE-78 MCP command confirmation, git clone safety flags.

3. **Features model novel/unvalidated** (Independent Thinker, Advisor, Analyst) — ANALYSIS-043 confirms no ecosystem provides cross-type per-component cherry-picking. Zero production validation. **[RESOLVED]** Added mandatory PoC validation gate to Decision 5: build markdown section extraction PoC first, test with real plugin, only proceed to other mechanisms after validation.

4. **Source resolution underspecified** (Critic, Analyst) — 4-row table is insufficient for reinstated custom resolution. Version/ref syntax in source strings not specified. **Deferred** — detailed source resolution specification will be part of implementation design, not ADR-level.

5. **Manifest location contradicts ADR-001 evidence** (Analyst, Independent Thinker) — ADR-001 chose root-level `plugin.json` citing Claude Code docs calling nested paths "error-prone". ADR-014 reverses this. **Acknowledged** — ADR-014 cites 3/3 platform convergence on wrapper directories as new evidence. The ADR-001 concern was about discoverability; the `.agent-plugin/` directory provides disambiguation.

6. **CI mode needs `--features` flag** (Critic) — `--ci` selects defaults only. No mechanism for CI to specify non-default features. **[RESOLVED]** Added `--features <list>` flag for explicit feature specification in CI/Docker environments.

7. **Hash scope undefined** (Security) — What exactly is hashed? **[RESOLVED]** Specified: SHA-256 of all content files listed in manifest, computed deterministically (sorted file paths, concatenated content).

8. **Lockfile-to-reality drift** (Independent Thinker) — Lockfile says X is installed, platform configs disagree. **Acknowledged** — inherent to any stateful model. `agent-plugin install` is the reconciliation command (re-applies lockfile state to platform configs).

9. **Source availability for team sync** (Independent Thinker, Critic) — `install` requires re-fetching from original sources. No caching. npm packages can be unpublished. **Acknowledged** — known limitation, same as npm/yarn. Content caching deferred to future enhancement.

10. **"rules" naming conflicts with Cursor `.cursorrules`** (Critic, Analyst) — Potential namespace confusion. **Acknowledged** — `.cursorrules` is Cursor's own rule format, different from plugin-distributed rules. Platform adapter handles the distinction.

## Agent Conflict: Scope Split

| Agent | Position |
|-------|----------|
| Architect | Keep bundled (weaker linkage acceptable for greenfield) |
| Critic | Split 2-way (installation + features vs taxonomy) |
| Independent Thinker | Split 2-way (same as Critic) |
| Security | Keep bundled (causally linked trust domains) |
| Analyst | Soft split suggestion (3-4 way) |
| **High-Level Advisor** | **Split 2-way (revised from Phase 1 3-way)** |

**Resolution** (High-Level Advisor ruling): 2-way split recommended as P1. ADR-014A: Installation Model (D1, D2, D6, D7). ADR-014B: Content Model, Features, and Taxonomy (D3, D4, D5, D8). The content taxonomy decisions are orthogonal to the installation model pivot. Splitting enables independent review and unblocks content taxonomy from features model controversy.

**Current status**: Split recommended but not applied. ADR remains as single document pending user decision.

## Agent Conflict: Features Model Priority

| Agent | Position |
|-------|----------|
| Architect | Acceptable (strongest contribution) |
| Critic | Well-designed |
| Independent Thinker | P1 — untested, needs validation |
| Analyst | P1 — untested, needs validation |
| Security | No specific position |
| **High-Level Advisor** | **P1 — keep with mandatory PoC gate** |

**Resolution** (High-Level Advisor ruling): Features model stays in ADR but requires PoC validation before implementation. Deferring entirely would gut the pivot rationale (feature selection is WHY the user pivoted from ADR-013). Accepting as-is ignores zero production validation. Middle path: mandate proof-of-concept for markdown section extraction before building other mechanisms.

## Process Observation: Immediate Supersession

ADR-013 was accepted on 2026-03-09 (Round 2: 5 Accept + 1 D&C) and superseded the same day by ADR-014. Independent Thinker raised a process concern: the Vercel Skills model was already analyzed in ANALYSIS-034 before ADR-013 was written. The ADR-013 debate evaluated it. The decision to reject Vercel in favor of TanStack was made with full awareness of the alternative.

**High-Level Advisor assessment**: The pivot rationale's factual error (P0-3, reclassified P1) supports the skeptical interpretation that the decision process did not adequately test assumptions. However, for a greenfield project with no users, fast pivoting is rational. The ADR-014 corrections address the factual inaccuracy.

## Changes Applied (Phase 3)

| Change | Lines Affected | Issue Resolved |
|--------|---------------|----------------|
| Added `version` and `ref` fields to lockfile schema | D2 schema + fields table | P0-4 |
| Clarified `hash` scope (sorted file paths, concatenated content) | D2 fields table | P1-7 |
| Rewritten pivot driver #3 (source resolution UX, not npm-only) | Context section, Option A cons, POS-004, More Information | P0-3 (reclassified P1) |
| Added hook merge semantics (independent per-plugin entries) | D7 Step 7 | P0-1 (reclassified P1) |
| Marked CLI content type as deferred | D4 table + taxonomy changes section | P1-1 |
| Added features model PoC validation gate | D5 new "Validation strategy" paragraph | P1-3 |
| Added `--features <list>` flag for CI | D5 feature wizard specification | P1-6 |
| Expanded IMP-001 security constraints (CWE-22, CWE-494, CWE-78, git safety) | Implementation Notes | P1-2 |
| Added MCP command confirmation requirement | IMP-001 security constraints | P0-2 (reclassified P2) |

## Round 2 Results (Phase 4 Convergence)

**Outcome**: CONSENSUS REACHED — 5 Accept + 1 D&C. ADR-014 status changed to Accepted.

| Agent | Verdict | Rationale |
|-------|---------|-----------|
| Architect | Accept | All P0/P1 resolved. No new issues. Cross-ADR coherence maintained. |
| Critic | Accept (92%) | All 7 prior concerns resolved. Minor AGENTS.md schema gap noted (non-blocking). |
| Independent Thinker | Accept (with dissent) | Factual error corrected, PoC gate adequate. Dissent: immediate supersession is a process gap; hash scope should include plugin.json. |
| Security | Accept | Risk score 2.8/10. All 5 CWE concerns resolved. R2-001: code region parser safety (impl-phase). R2-002: recommend UTC-only timestamps. |
| Analyst | D&C | Architecture sound. Residual P2: manifest location reverses ADR-001 ALT-002 rejection; 3-platform convergence claim unverified. Commits because greenfield project, manifest location changeable later. |
| High-Level Advisor | Accept | All blocking concerns resolved. Watch: PoC accuracy, feature adoption rate, source resolution LOC. |

### Recorded Dissent (D&C — Analyst)

- Manifest location (Decision 3) reverses ADR-001's rejection of nested paths (ALT-002) without addressing the "error-prone" rationale from Claude Code docs. Defensible on disambiguation grounds but the reversal rationale is thin.
- The "3/3 platform convergence" claim is stated as fact but unverified by any analysis note. Documentation quality issue, not architectural risk.
- Commits because: greenfield project, no users, manifest location is changeable without breakage.

### Recorded Dissent (Accept with reservations — Independent Thinker)

- Immediate supersession pattern (ADR-013 accepted then superseded same day) is a process gap. Suggests the review process lacks a cooling period or implementation checkpoint. Process concern, not content concern.
- Hash scope should include `plugin.json` itself, not just content files. Local sources with manifest-only changes would go undetected by hash comparison.

### Implementation-Phase Security Notes (Security Agent)

- R2-001 (Medium, CWE-94): Code region parser must validate balanced region markers, reject nested same-name regions, reject overlapping regions.
- R2-002 (Low): Lockfile timestamps should specify UTC-only constraint.

## Observations

- [outcome] ADR-014 ACCEPTED in Round 2: 5 Accept + 1 D&C (Analyst). Consensus reached. #adr-review #adr-014 #accepted
- [fact] Round 1 produced unanimous NEEDS_REVISION with 4 P0 issues across 6 agents, reclassified to 1 true P0 after advisor ruling #adr-review #adr-014
- [decision] Only true P0: lockfile lacks version/ref for deterministic reproduction. Resolved by adding version and ref fields. #lockfile #determinism
- [decision] Hook merge reclassified P0 to P1: file-based installation means independent per-plugin hooks, no cross-plugin merge needed #hooks #simplification
- [decision] MCP command execution reclassified P0 to P2: concern belongs on ADR-011, not ADR-014. Partial mitigation added. #security #scope
- [decision] Pivot rationale factual error reclassified P0 to P1: ADR-013 did support git/local via bun. Rewritten to describe UX distinction, not capability gap. #accuracy #pivot
- [decision] Features model: keep with mandatory PoC validation gate. Not deferred (would gut pivot rationale), not accepted as-is (zero production validation). #features #validation
- [decision] Scope split: 2-way recommended (installation model vs content model) but not applied pending user decision #scope #split
- [insight] Independent Thinker found factual error that 5 other agents missed, validating multi-agent review pattern #process #review-quality
- [risk] Features model is novel with zero production precedent. PoC gate mitigates but does not eliminate risk. #features #novelty
- [fact] All 6 agents flagged CLI content type as underspecified. Deferred to future ADR. #content-types #cli

## Relations

- reviews [[ADR-014 Explicit Installation Model and Content Features]]
- relates_to [[DEBATE-ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ANALYSIS-033 Consumer and Author Commands]]
- relates_to [[ANALYSIS-034 Skill Versioning Models Comparison]]
- relates_to [[ANALYSIS-043 Per-Component Cherry-Picking InstallMode Design]]
- relates_to [[ANALYSIS-044 Sectioned Content Configuration Patterns]]