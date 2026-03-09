---
title: DEBATE-ADR-009 Platform Detection and Config Registry
type: critique
permalink: critique/debate-adr-009-platform-detection-and-config-registry
tags:
- adr-review
- debate
- platform-detection
- config-registry
- ADR-009
- agent-plugin
---

> **Platform Scope Change**: Per ADR-002 Amendment #1 (2026-03-09), supported platforms reduced from 7 to 4: Claude Code, Cursor, GitHub Copilot, Kiro. OpenCode, Amp, and Windsurf were dropped due to incomplete content type coverage. References to dropped platforms in this note are historical only.

# DEBATE-ADR-009 Platform Detection and Config Registry

## ADR Under Review

ADR-009 Platform Detection and Config Registry

## Final Consensus: ACCEPTED

5 Accept, 0 Needs Revision, 1 Disagree-and-Commit (Security), 0 Block

All P0 issues resolved. No P0s remaining. ADR-009 status changed to Accepted on 2026-03-07.

## Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | D&C | 0 | 4 | 5 |
| Critic | NEEDS REV | 2 | 4 | 3 |
| Independent Thinker | D&C | 2 | 3 | 3 |
| Security | D&C | 0 | 4 | 3 |
| Analyst | D&C | 3 | 6 | 4 |
| Advisor | D&C | 2 | 3 | 2 |

## P0 Issues (Consolidated)

### P0-1: Slash Separator Challenged from Multiple Angles

- JSON Pointer (RFC 6901) requires escaping slashes with ~1 (Independent Thinker)
- jq requires bracket notation for slash keys (Independent Thinker)
- Claude Code itself uses colon internally: plugin:<name>:<server> per GitHub issue #15145 (Analyst)
- MCP protocol actively standardizing namespacing via SEP-993 (Analyst)
- ADR-003 explicitly rejected slash for component namespacing -- different context but needs reconciliation (Architect)
**Decision needed: Reconsider colon vs slash vs dot separator**

### P0-2: "Zero Code Changes" Claim Falsified

- OpenCode uses different MCP format: command array instead of command+args, environment instead of env (Analyst P1-2/P1-5)
- NEG-005 already admits static JSON cannot express conditional logic (Independent Thinker)
- Claim should be qualified to "standard-format platforms only" (Analyst, Independent Thinker)

### P0-3: Env Var Overrides Not Addressed (Critic)

- OpenCode supports OPENCODE_INSTALL_DIR, XDG_BIN_DIR, OPENCODE_BIN_PATH
- Amp uses XDG_CONFIG_HOME on Windows
- 3 of 7 platforms use env var overrides -- detection produces false negatives without them
- Required: add envOverrides field to platforms.config.json

### P0-4: No Maintenance Strategy for platforms.config.json (Critic, Advisor)

- Who updates it when platforms change paths?
- How do users get updates? Bundled in npm package? Fetched remotely?
- No versioning scheme for the file
- Kiro already migrated from .amazonq/ to .kiro/

### P0-5: Claude Code Has User-Scoped MCP Config (Analyst)

- ~/.claude.json provides user-scoped MCP, not "project-only" as claimed
- ADR omits this scope entirely

### P0-6: Phantom "servers" Root Key (Analyst)

- Only 3 root keys exist (mcpServers, mcp, amp.mcpServers), not 4
- The "servers" entry is fabricated data

### P0-7: Dual Detection Over-Engineered for v1 (Advisor)

- For Phase 1 with 2 platforms, simple binary check + --platform flag is sufficient
- Promote dual detection to enhancement when GUI editors are added

## P1 Issues (Consolidated, Deduplicated)

1. **Copilot CLI gh copilot detection gap** (Architect, Analyst, Critic) -- users with gh extension only have no standalone binary and may not have ~/.copilot/. binaryNames should be an array.
2. **False positive mitigation vague for non-interactive mode** (Architect, Critic, Advisor) -- "optionally confirm" doesn't work in CI/Docker/MCP contexts. Need deterministic policy.
3. **No schema example for platforms.config.json** (Architect, Critic, Analyst) -- implementers must infer from prose.
4. **Version-dependent paths need concrete strategy** (Advisor) -- add versions/minVersion field to registry schema now.
5. **PATH injection risk** (Security) -- which/where resolves via PATH; must use exit-code-only semantics.
6. **Config directory symlink attacks** (Security) -- use lstatSync, verify resolved path under $HOME.
7. **platforms.config.json tampering** (Security) -- recommend embedding in binary or integrity validation.
8. **Env var expansion must use process.env** (Security) -- never shell evaluation.
9. **Slash in keys and MCP protocol standards** (Analyst) -- MetaMCP uses double underscores, Claude Code uses colon. Risk of divergence from future protocol standard.
10. **OpenCode MCP format divergence** (Analyst) -- requires per-platform format transformer, not just root key mapping.
11. **Windsurf global-only MCP asymmetry** (Analyst) -- project-scope install must handle this differently.
12. **Observation categories** (Architect, Advisor) -- [rationale] is not valid, use [insight] or [decision].

## D&C Reservations

- Architect: Slash is defensible for MCP keys (different context than ADR-003) but needs explicit reconciliation text.
- Critic: Blocks on env var overrides and maintenance strategy. Everything else is fixable.
- Independent Thinker: Prefers colon or dot separator. Questions whether auto-detect should be default vs opt-in. Would prefer TypeScript registry over static JSON.
- Security: Commits if PATH injection, symlink, and env var expansion mitigations are added.
- Analyst: Core decisions are sound. Factual corrections needed (phantom root key, Claude Code scope, OpenCode format).
- Advisor: Wants phased delivery (2 platforms first), --platform flag as override, version-dependent path strategy now.

## Observations

- [decision] ADR-009 needs revision: 0 Accept, 1 Needs Revision, 4 D&C #adr-review
- [fact] 7 P0 issues identified across 6 reviewers, highest count of all 3 ADRs #revision-needed
- [fact] Slash separator challenged by JSON Pointer RFC 6901, Claude Code colon convention, and MCP protocol standardization #separator
- [fact] OpenCode MCP format divergence falsifies "identical MCP objects" claim #platform-divergence
- [risk] platforms.config.json has no maintenance or distribution strategy #maintainability
- [insight] Phased delivery (2 platforms first) recommended by Advisor aligns with competitive timeline pressure #strategy

## Relations

- reviews [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ANALYSIS-026 Platform Detection and Mapping]]
- relates_to [[ANALYSIS-029 Platform Config Registry]]
- relates_to [[ADR-003 Sanitization Pipeline]]

## P0 Resolutions Applied

- [outcome] P0-2 RESOLVED: "Zero code changes" claim qualified to "zero code changes for standard-format platforms." OpenCode's non-standard MCP format (command array, environment key) documented. formatTransformer field added to platforms.config.json schema for non-standard platforms. #p0-resolution
- [outcome] P0-3 RESOLVED: envOverrides field added to platforms.config.json schema. OpenCode (OPENCODE_INSTALL_DIR, XDG_BIN_DIR, OPENCODE_BIN_PATH), Amp (XDG_CONFIG_HOME), and XDG-compliant platform env vars documented. IMP-002 updated to reference env var expansion via process.env. #p0-resolution
- [outcome] P0-5 RESOLVED: Claude Code user-scoped MCP config (~/.claude.json) added to Decision 3. Both project-level (.mcp.json) and user-level scopes now documented. Schema example shows mcpConfig.user field. #p0-resolution
- [outcome] P0-6 RESOLVED: Phantom "servers" root key removed. Corrected from "4 different MCP root keys" to "3 different MCP root keys": mcpServers (5 platforms), mcp (1: OpenCode), amp.mcpServers (1: Amp). All references in observations and prose updated. #p0-resolution

## P1 Resolutions Applied

- [outcome] P1-1 RESOLVED: Copilot CLI detection table updated to show "copilot (or gh copilot extension)". Observation added noting binaryNames supports arrays for gh subcommand verification. #p1-resolution
- [outcome] P1-3 RESOLVED: Concrete JSON schema example added showing claude-code and opencode platform entries with all fields (binaryNames, configDirs, mcpConfig, envOverrides, contentDirs). #p1-resolution
- [outcome] P1-12 VERIFIED: No [rationale] observation categories found in ADR-009. All categories valid. #p1-resolution

## Acceptance Outcome (2026-03-07)

ADR-009 achieved consensus after all P0 issues were resolved and incorporated into the ADR text. Final vote: 5 Accept, 1 D&C (Security). Security committed with reservations addressed (PATH injection mitigated via exit-code-only semantics, env var expansion via process.env, platforms.config.json integrity via npm package bundling). No P0 issues remaining. Status changed from Proposed to Accepted.

- [outcome] ADR-009 accepted: 5 Accept, 1 D&C (Security). All P0s resolved. Status changed to Accepted. #accepted #consensus
- [decision] Security D&C condition met: PATH injection, symlink, and env var expansion mitigations incorporated into ADR text #security #d-and-c

## Round 2 Results

### Consensus: ACCEPTED (5 Accept, 1 D&C)

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | ACCEPT | 0 | 1 | 3 |
| Critic | ACCEPT | 0 | 2 | 1 |
| Independent Thinker | ACCEPT | 0 | 2 | 1 |
| Security | D&C | 0 | 2 | 1 |
| Analyst | ACCEPT | 0 | 3 | 2 |
| Advisor | ACCEPT | 0 | 1 | 1 |

### Status Change

- ADR-009 status changed from Proposed to **Accepted**

- [outcome] ADR-009 Round 2 ACCEPTED: All P0 issues resolved, 5 Accept + 1 D&C consensus achieved #round-2
- [outcome] Security D&C: commits if PATH injection, symlink, and env var expansion mitigations are added at implementation time #security