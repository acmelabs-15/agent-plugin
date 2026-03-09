---
title: DEBATE-ADR-001 Plugin Format and Manifest
type: critique
permalink: critique/debate-adr-001-plugin-format-and-manifest
tags:
- adr-review
- debate
- ADR-001
- plugin-format
---

# DEBATE-ADR-001 Plugin Format and Manifest

**ADR**: [[ADR-001-plugin-format-and-manifest]]
**Date**: 2026-03-07
**Round**: 1
**Result**: NOT CONSENSUS — 2 Accept, 3 Disagree-and-Commit, 1 Block (Critic)

---

## Phase 1: Independent Reviews

### Architect — ACCEPT

Strengths: Evidence-based corrections (root-level manifest), installMode is clean, npm-convention fields.
Issues:

- P1-1: Component path declarations lack schema contract (string vs array ambiguity)
- P1-2: `sources` and `dependencies` declared but undefined
- P1-3: Conflict resolution TBD for 3 of 5 component types
- P2-1: No `$schema` field
- P2-2: `platforms` field semantics unclear
- P2-3: No manifest schema versioning
- P2-4: Missing MADR 4.0 sections

### Critic — NEEDS REVISION (BLOCK)

Strengths: Root-level manifest, installMode concept, npm conventions.
Blocking Issues:

- P0-1: `platforms` field has no defined semantics (omission behavior, invalid platform handling, open vs closed enum)
- P0-2: `dependencies` and `conflicts` fields declared with no schema
Other Issues:
- P1-1: No schema versioning
- P1-2: `sources` field undefined
- P1-3: Component path edge cases unspecified
- P1-4: installMode binary insufficient for hybrid plugins
- P1-5: `mcp` singular vs array inconsistency

### Independent Thinker — DISAGREE-AND-COMMIT

Challenges:

1. Mandatory manifest asserted not proven — convention-based discovery works (Claude Code, Vercel)
2. Bundle model assumed correct without stress-testing flat-skill alternative
3. JSON format unjustified — why not YAML (matches frontmatter) or package.json field?
4. installMode binary too simplistic for mixed-dependency plugins
5. Format fragmentation cost not weighed — should we adopt Claude Code format instead?

Commits to direction but recommends: justify JSON, quantify bundle need, acknowledge installMode limits, state format fragmentation stance.

### Security — DISAGREE-AND-COMMIT

Critical Issues:

- SEC-P0-001: Path traversal in component path declarations (CWE-22, 9/10)
- SEC-P0-002: Hook merging combines untrusted output with trusted (CWE-94, 8/10)
- SEC-P0-003: MCP server configuration injection — full code execution (CWE-94/78, 9/10)
High Issues:
- SEC-P1-001: Plugin name squatting/typosquatting (6/10)
- SEC-P1-002: No manifest integrity verification (7/10)
- SEC-P1-003: Conflict resolution as social engineering vector (5/10)
- SEC-P1-004: Dependency confusion attack surface (6/10)

Required mitigations: path canonicalization, hook isolation boundaries, MCP approval gates, scoped names.

### Analyst — DISAGREE-AND-COMMIT

Evidence Gaps:

1. Per-type conflict resolution is novel design, not borrowed from any reference system
2. Nested path error-proneness argument slightly overstated (issue is component placement, not manifest path)
3. `prompts/` directory has zero precedent in any analyzed system
Feasibility:
4. Hook merging across 17 event types is underspecified — recursive conflict problem
5. Collection mode dependency solver not addressed
Fact-checks:
6. 5-component model is deliberate subset of Claude Code's 7 types — not documented as such
7. "Matches npm conventions" slightly inaccurate (npm requires only name+version)

### High-Level Advisor — ACCEPT

Strategic assessment: Right level of simplicity. Format is simpler than npm, VS Code, Vercel. Correct position for v1.
P0 Risk: The platform translation layer (how plugin.json maps to 7 platform runtimes) is the entire value proposition and this ADR says nothing about it. That's the ADR that hasn't been written yet.
Recommendation: Go. Ship this format. Open ADR for platform translation contract immediately.

---

## Phase 2: Consolidation

### Consensus Points (all 6 agree)

- Bundle model is directionally correct
- Root-level plugin.json is better than nested
- installMode concept addresses a real need
- npm-convention minimum fields reduce friction

### Contested Points

1. **`platforms` semantics** — Critic blocks, Architect/Analyst flag as P1/P2
2. **`dependencies`/`conflicts` schemas** — Critic blocks, Analyst flags
3. **Security: path traversal, hook merging, MCP execution** — Security flags 3 P0s
4. **installMode binary vs hybrid** — Critic, Independent Thinker both flag
5. **JSON format justification** — Independent Thinker challenges, others silent
6. **`prompts/` directory precedent** — Analyst flags, others silent
7. **Manifest schema versioning** — Architect, Critic, Security all flag

### Cross-Agent Agreement on Gaps

- Need manifest schema version field (4/6 agents)
- Need defined `platforms` semantics (3/6 agents)
- Need path validation rules (3/6 agents)
- Need defined `dependencies`/`conflicts` schemas (3/6 agents)

---

## Phase 3: Resolution Proposals

### For P0 Issues (must resolve to unblock Critic)

**P0-1: `platforms` field semantics**
Proposal: Omission means "all platforms". Invalid platform = warning at install. Open string set (future platforms allowed). Degraded platforms valid entries.

**P0-2: `dependencies`/`conflicts` schemas**
Proposal: Defer to separate ADR with explicit forward reference. Remove empty `{}` from ADR-001 examples. These are load-bearing architectural concepts that deserve their own ADR.

### For Security P0s (must address before implementation)

**SEC-P0-001-003: Path traversal, hook merging, MCP execution**
Proposal: Create ADR-004 Plugin Security Model as a companion ADR. Reference from ADR-001. These are implementation-level security controls, not format decisions.

### For P1 Issues (should resolve)

**Schema versioning**: Add `formatVersion: 1` to required fields.
**installMode hybrid**: Add "Future Considerations" section acknowledging the limitation.
**JSON justification**: Add brief rationale paragraph.
**`prompts/` rationale**: Document as an extension beyond analyzed systems.

---

## Phase 4: Convergence Vote

Pending user review of P0 resolutions. If P0-1 and P0-2 proposals are accepted, Critic unblocks → 2 Accept, 4 D&C = CONSENSUS via Disagree-and-Commit.

## Observations

- [fact] 6 agents reviewed ADR-001: 2 Accept, 3 Disagree-and-Commit, 1 Block #adr-review
- [decision] Core architecture (bundle model, mandatory manifest, root placement) validated by all 6 agents #consensus
- [risk] 3 security P0s identified: path traversal, hook output poisoning, MCP code execution #security
- [risk] `platforms` and `dependencies` fields underspecified — blocks implementation #schema
- [insight] Per-type conflict resolution is novel design with no precedent in analyzed systems — should be labeled as such #honesty
- [insight] Platform translation layer identified as the project's actual value proposition and biggest risk #strategy

## Relations

- reviews [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[ANALYSIS-003-vercel-skills-format-deep-dive]]
- relates_to [[ANALYSIS-004-tanstack-intent-deep-dive]]
- relates_to [[ANALYSIS-005-claude-code-plugin-format]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]