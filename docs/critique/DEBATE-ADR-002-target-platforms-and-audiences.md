---
title: DEBATE-ADR-002-target-platforms-and-audiences
type: note
permalink: critique/debate-adr-002-target-platforms-and-audiences
tags:
- adr-review
- debate
- ADR-002
- platforms
- audiences
---

# DEBATE-ADR-002 Target Platforms and Audiences

**ADR**: [[ADR-002-target-platforms-and-audiences]]
**Date**: 2026-03-07
**Round**: 1
**Result**: NOT CONSENSUS — 0 Accept, 4 Disagree-and-Commit, 1 Needs Revision

## Agent Verdicts

| Agent | Verdict | Confidence |
|-------|---------|------------|
| Architect | Disagree-and-Commit | High |
| Critic | Needs Revision | High |
| Independent Thinker | Disagree-and-Commit | High |
| Security | Disagree-and-Commit | High |
| Analyst | Disagree-and-Commit | High |
| High-Level Advisor | Disagree-and-Commit | High |

## Consolidated P0 Issues

### P0-1: Cross-Platform Claim Contradicts Graceful Degradation (Critic, Independent Thinker)

The `platforms` field was removed from plugin.json (all plugins are inherently cross-platform). But ADR-002 Section 3 defines 3 platforms where capabilities fail silently (Codex CLI hooks don't block, Cline can't spawn sub-agents, Gemini CLI runs agents sequentially). A plugin with a blocking hook IS NOT cross-platform — it fails silently on Codex CLI.

**Proposed Resolution**: Add a capability requirements mechanism (e.g., `"requires": { "capabilities": ["blocking-hooks"] }`) so plugins can declare what they need. The installer warns at install time when a target platform lacks a required capability. This is different from restricting platforms — it's declaring functional requirements.

### P0-2: No Discovery Mechanism (Critic, High-Level Advisor)

The ADR rejects a hosted registry but provides no alternative for plugin discovery. The Author CLI includes `publish` with no defined target. NEG-003 acknowledges this but offers no mitigation.

**Proposed Resolution**: Add a forward reference to a future discovery ADR. Define what `publish` means in v1 (e.g., publish to npm, push to GitHub). Consider federated discovery (community-maintained index.json files, Homebrew taps model) as noted by Independent Thinker.

### P0-3: Instruction File Injection Is Critical Security Risk (Security)

The tool modifies files that directly control AI agent behavior (CLAUDE.md, .cursorrules). A malicious plugin injects adversarial prompts that persist across sessions. No content policy, no user review, no revert on uninstall. Rated 9/10 severity.

**Proposed Resolution**: Add NEG-006 to Consequences acknowledging instruction file modification as a high-severity attack vector requiring content security policies in ADR-004.

### P0-4: Hook Execution Runs With Full User Privileges (Security)

Auto-merged hook collisions (ADR-003) mean a malicious hook runs alongside legitimate ones without independent approval. Access to SSH keys, cloud credentials, all project files. Rated 8/10 severity.

**Proposed Resolution**: Add NEG-007 to Consequences acknowledging the no-registry multi-source model provides no centralized vetting, requiring explicit trust model in ADR-004.

## Consolidated P1 Issues

| ID | Issue | Raised By |
|----|-------|-----------|
| P1-1 | Duplicate ADR exists (ADR-001-target-platforms-and-selection-criteria) — delete or supersede | Architect, Critic |
| P1-2 | No audience priority ordering; need Consumer CLI (P0) → Author CLI (P1) → MCP (P2) | High-Level Advisor |
| P1-3 | Need phased delivery table: Phase 1 Claude Code + Cursor, Phase 2 remaining platforms, etc. | High-Level Advisor, Independent Thinker |
| P1-4 | Instruction file paths incomplete — only 2 of 7 platforms specified | Critic, Analyst |
| P1-5 | "5.5/6" scoring conflates architecturally different degradation situations | Analyst |
| P1-6 | skills.sh competitor not analyzed — launched Jan 2026, 40+ platforms, 20K installs in 6 hours | Analyst |
| P1-7 | Self-bootstrapping Phase 4 has no acceptance criteria | Critic |
| P1-8 | Three-audience single-package bloat risk — no size budget or lazy loading commitment | Critic |
| P1-9 | Source types are trust-on-first-use with no content hashing | Security |
| P1-10 | MCP server exposes plugin metadata with no access control | Security |
| P1-11 | Instruction file management rated Low feasibility — needs PoC before committing to 7+ formats | Analyst |
| P1-12 | ADR-002 doesn't reference ADR-003 (bidirectional link missing) | Critic |

## Consolidated P2 Issues

| ID | Issue | Raised By |
|----|-------|-----------|
| P2-1 | npm org registration is operational, not architectural — remove from ADR | Architect, Critic |
| P2-2 | Source type list omits Bitbucket, generic git URLs, tarballs — state if closed or extensible | Critic |
| P2-3 | No success metrics for audiences | High-Level Advisor |
| P2-4 | Parallel agents as hard requirement questionable — near-zero ecosystem adoption | Independent Thinker |
| P2-5 | Consider pluggable platform adapters (community can write adapters for new platforms) | Independent Thinker |
| P2-6 | Self-bootstrapping circular trust problem | Security |
| P2-7 | GitHub shorthand typosquatting risk | Security |
| P2-8 | Quantify testing surface: 7 platforms × 6 capabilities = 42+ test combinations | Critic |

## Strengths Identified

- Evidence-based platform selection from 16-platform analysis (Architect, Analyst)
- MCP server as genuine differentiator — no competitor offers AI-native plugin discovery (Analyst, High-Level Advisor)
- 5-component bundle (skills + agents + hooks + prompts + MCP) is unique — skills.sh covers skills only (Analyst)
- Coherent trilogy with ADR-001 and ADR-003 (Architect)
- Self-bootstrapping Phase 1/Phase 4 split avoids both premature complexity and retrofit risk (Architect)

## Resolution Recommendations

The 4 P0 issues are additive — they require new sections/fields, not structural changes:

1. Add capability requirements mechanism to plugin.json (resolves P0-1)
2. Add forward reference to discovery/registry ADR; define what `publish` means in v1 (resolves P0-2)
3. Add NEG-006 and NEG-007 to Consequences with security forward refs to ADR-004 (resolves P0-3, P0-4)

## Observations

- [fact] 6 agents reviewed ADR-002: 0 Accept, 4 D&C, 1 Needs Revision #debate-result
- [decision] P0-1 requires capability requirements mechanism for degraded platforms #cross-platform
- [decision] P0-2 requires discovery strategy forward reference #distribution
- [decision] P0-3 and P0-4 require security consequence entries with ADR-004 forward refs #security
- [insight] skills.sh is a direct competitor launched Jan 2026 with 40+ platform support — must be analyzed #competitive
- [insight] Instruction file management is the highest technical risk area, rated Low feasibility by analyst #risk
- [insight] High-level-advisor recommends explicit audience priority ordering: Consumer P0, Author P1, MCP P2 #strategy
- [risk] No agent gave a clean Accept — all see execution risk in the current scope #consensus

## Relations

- part_of [[ADR-002-target-platforms-and-audiences]]
- relates_to [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[DEBATE-ADR-001-plugin-format-and-manifest]]
- relates_to [[REVIEW-ARCHITECT-ADR-002-target-platforms-and-audiences]]
- relates_to [[REVIEW-CRITIC-ADR-002-target-platforms-and-audiences]]
- relates_to [[REVIEW-INDEPENDENT-THINKER-ADR-002-target-platforms-and-audiences]]
- relates_to [[REVIEW-SECURITY-ADR-002-target-platforms-and-audiences]]
- relates_to [[REVIEW-ANALYST-ADR-002-target-platforms-and-audiences]]
- relates_to [[REVIEW-HIGH-LEVEL-ADVISOR-ADR-002-target-platforms-and-audiences]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
