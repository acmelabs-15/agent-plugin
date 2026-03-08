---
title: DEBATE-ADR-010 Installation Lifecycle and CLI Generation
type: critique
permalink: critique/debate-adr-010-installation-lifecycle-and-cli-generation
tags:
- adr-review
- debate
- installation
- cli-generation
- ADR-010
- agent-plugin
---

# DEBATE-ADR-010 Installation Lifecycle and CLI Generation

## ADR Under Review

ADR-010 Installation Lifecycle and CLI Generation

## Final Consensus: ACCEPTED

5 Accept, 0 Block, 0 Needs Revision, 1 Disagree-and-Commit (Security)

All P0 issues resolved. No P0s remaining. ADR-010 status changed to Accepted on 2026-03-07.

## Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | D&C | 2 | 5 | 5 |
| Critic | NEEDS REV | 3 | 5 | 3 |
| Independent Thinker | D&C | 0 | 4 | 4 |
| Security | D&C | 2 | 5 | 2 |
| Analyst | D&C | 0 | 4 | 3 |
| Advisor | BLOCK | 3 | 4 | 2 |

## P0 Issues (Consolidated)

### P0-1: Split CLI Generation into Separate ADR-011 (Advisor BLOCK, 4+ agree)

- CLI generation is a different concern with different stakeholders, change frequency, and risk profile
- 3 of 6 NEG items are CLI-specific
- ADR-003 was previously blocked for the same overloading problem
- CLI generation has independent utility (could be a standalone package)
**Strong consensus: Split Decision 5 into ADR-011**

### P0-2: Dependency Auto-Install Is Not MVP (Advisor BLOCK, Security P0, Analyst, Ind. Thinker)

- Security P0: dependency install commands from plugin manifests = arbitrary code execution (CWE-78, CVSS 9.1)
- No mainstream tool auto-installs system deps (npm, pip, cargo, mise all require explicit user action)
- Sanitization pipeline for install commands does not exist yet
- Cross-platform package manager complexity (brew vs apt vs winget vs choco)
- User confirmation is insufficient mitigation against social engineering
- ANALYSIS-027 originally recommended check-only for documented security reasons
**Strong consensus: Defer to check-only for MVP. Revisit when user demand proves friction matters.**

### P0-3: Command Grouping Algorithm Undefined (Critic, Architect, Advisor, Analyst)

- ADR says "prefix-based" but examples show suffix-based grouping
- ANALYSIS-030 says "prefix convention" but groups by noun, not verb
- No formal algorithm (pseudocode or decision tree)
- Failure modes: camelCase tools, single-word tools, 3+ segment tools, mixed conventions
- Algorithm must be deterministic for consistent CLI generation

### P0-4: MCP Server Lifecycle Undefined (Critic, Independent Thinker, Security)

- No crash recovery strategy
- No orphaned process handling
- No concurrent invocation behavior
- "Terminated on CLI exit" assumes clean exit
- Each invocation cold-starts server (stdio transport = per-process)

### P0-5: Dependency Rollback Is "Attempt to Reverse" (Critic, Analyst)

- Zero mainstream package managers implement reliable rollback (npm, pip, cargo, brew)
- brew uninstall may fail if other packages depend on it
- Atomicity claim (POS-004) is false if Phase 5 dependency rollback is unreliable

### P0-6: No Plugin Source Integrity Verification (Security, CWE-494, CVSS 8.6)

- No SHA-512 verification for npm packages
- No pinned commit SHA for git sources
- Hash computed AFTER install, not compared pre-install

### P0-7: MCP Namespace Not Explicitly Codified (Architect)

- ADR references ANALYSIS-029 by name but never states the separator format
- Implementer reading only ADR-010 would not know the MCP key format

## P1 Issues (Consolidated, Deduplicated)

1. **CLI command selection in wrong phase** (Architect) -- Phase 2 (Select) cannot show CLI commands before package is fetched in Phase 3 (Resolve). Move to Phase 4 (Confirm).
2. **Upgrade order should be install-first** (Architect) -- remove-then-install creates a window with no plugin. ANALYSIS-027 recommends install-then-remove.
3. **Multiselect fatigue** (Critic, Independent Thinker) -- 2-3 interactive prompts before anything happens. Need --platforms and --cli-groups flags for power users.
4. **Scope creep in dependency management** (Critic, Independent Thinker) -- detecting OS, package manager, correct package name, authentication (sudo) is a massive implementation burden.
5. **Upgrade config migration undefined** (Independent Thinker, Analyst) -- renamed groups, merged groups, removed tools within groups not handled.
6. **Symlink hijacking and binary name shadowing** (Security) -- plugin named "git" would shadow system git. Need denylist or prefix (ap-pluginname).
7. **MCP server auto-start = untrusted code execution** (Security) -- plugin MCP server runs with user's full permissions on every CLI invocation.
8. **CI mode auto-confirms dependency installation** (Security, Advisor) -- --ci silently executes install commands without human review. Must block on new deps in CI.
9. **Lockfile integrity is warn-only** (Security) -- tampered lockfile redirects MCP server execution. Should error-with-override.
10. **Uninstall flow not specified** (Advisor) -- one sentence for a flow that must reverse every install operation.
11. **6 phases should be 4** (Independent Thinker, Advisor) -- first 3 phases are read-only with identical rollback ("no changes made").
12. **~/.local/bin not on PATH on macOS** (Critic, Analyst) -- needs post-install PATH check and user guidance.
13. **Windows CLI distribution deferred** (Advisor, Analyst, Independent Thinker) -- affects plugin author contract. Must explicitly scope to Unix for v1.
14. **MADR 4.0 frontmatter fields missing** (Architect) -- third ADR with this issue.
15. **MCP schema type gaps for CLI** (Analyst) -- array, object, oneOf parameters have no CLI mapping defined.
16. **Rollback backup created too late** (Security) -- .bak created in Phase 6 but Phase 5 failures need it.

## D&C Reservations

- Architect: Commits but requires MCP namespace codification and staging directory clarification.
- Critic: Blocks on MCP lifecycle, dependency rollback, and grouping algorithm.
- Independent Thinker: Would prefer 4 phases, check-only deps, and CLI as separate package. Expects auto-install to cause a security incident within 6 months.
- Security: Commits if dependency commands come from agent-plugin code (not plugin manifests) and integrity verification is added.
- Analyst: Commits but notes dependency rollback is aspirational. Sanitization pipeline must be foregrounded as primary security control.
- Advisor: BLOCKS. "You are designing the complete system before building anything. Ship the simple version first."

## Observations

- [decision] ADR-010 needs revision with 1 BLOCK: 0 Accept, 1 Block, 1 Needs Revision, 3 D&C #adr-review
- [decision] Strong consensus to split CLI generation into ADR-011 #architecture
- [decision] Strong consensus to defer dependency auto-install to check-only for MVP #security #scope
- [fact] 7 P0 issues identified including 1 BLOCK verdict from Advisor #revision-needed
- [risk] Dependency install from plugin manifests rated CVSS 9.1 by Security agent #security
- [insight] Advisor warns against over-engineering: "10+ ADRs, 30+ analyses, zero lines of production code" #strategy
- [fact] Zero mainstream package managers implement reliable dependency rollback #feasibility

## Relations

- reviews [[ADR-010 Installation Lifecycle and CLI Generation]]
- relates_to [[ANALYSIS-027 Installation Mechanics]]
- relates_to [[ANALYSIS-029 Platform Config Registry]]
- relates_to [[ANALYSIS-030 Auto-Generated CLI from MCP Tools]]
- relates_to [[ADR-003 Sanitization Pipeline]]

## P0-1 Resolution

P0-1 resolved: CLI generation (Decision 5) extracted from ADR-010 into ADR-011 Auto-Generated CLI from MCP Tools. ADR-010 renamed to "Installation Lifecycle" and set to Proposed status for continued revision. 5 of 6 reviewers agreed on split.

- [outcome] P0-1 resolved: CLI generation split into ADR-011, ADR-010 narrowed to install lifecycle only #p0-resolution #architecture
- relates_to [[ADR-011 Auto-Generated CLI from MCP Tools]]

## P0-2 Resolution: Dependency Auto-Install Kept with Revised Security Model

P0-2 recommended reverting to check-only for MVP due to CVSS 9.1 arbitrary code execution risk from plugin manifests. The revised approach keeps auto-install but eliminates the threat vector:

1. Dependencies defined by agent-plugin package in `platforms.config.json`, NOT by plugin authors in `plugin.json`.
2. The `systemDependencies` field removed from plugin manifest schema entirely.
3. Multiselect prompt (not confirm) for missing deps, all selected by default.
4. Package manager dependencies included in chain (brew needed for claude, brew itself may be missing).
5. Uninstall shows reverse multiselect of installed deps.

- [outcome] P0-2 resolved: dependency auto-install kept. CVSS 9.1 mitigated by moving install command definitions from untrusted plugin manifests to trusted package code (platforms.config.json). Remaining risk = supply chain attack on agent-plugin npm package itself (same as any npm dependency). #p0-resolution #dependencies #security
- [decision] Security agent's condition met: "Commits if dependency commands come from agent-plugin code (not plugin manifests)" -- this is exactly what the revised approach does. #security #d-and-c

## P0-5 Resolution: Dependency Rollback Honesty

P0-5 raised that zero mainstream package managers implement reliable rollback (npm, pip, cargo, brew), making the atomicity claim (POS-004) misleading for dependency operations. Resolution:

1. Phase 5 rollback language changed from "attempt to reverse dependency installs" to "dependency uninstall is best-effort and may fail."
2. POS-004 scoped: atomicity applies to file writes and config merges, not to system dependency installation.
3. NEG-004 added: explicitly documents that dependency rollback is not atomic, with brew uninstall failure as a concrete example.

- [outcome] P0-5 resolved: dependency rollback language made honest. Atomicity claim scoped to file writes and config merges. NEG-004 added for best-effort dependency uninstall. #p0-resolution #rollback #honesty

## P0-6 Resolution: Plugin Source Integrity Verification

P0-6 identified CWE-494 (download of code without integrity check, CVSS 8.6). Resolution: IMP-008 added requiring pre-install integrity verification:

1. npm: verify tarball SHA-512 against registry `dist.integrity` field before extraction.
2. git: pin to commit SHA in lockfile after first install, verify on upgrade, warn on force-pushed tags.
3. local: compute and store SHA-256 content hash on first install, warn on change.

- [outcome] P0-6 resolved: pre-install integrity verification added as IMP-008. npm SHA-512, git commit SHA pinning, local content hashing. CWE-494 mitigated. #p0-resolution #security #integrity

## P0-7 Resolution: MCP Namespace Explicitly Codified

P0-7 noted that ADR-010 referenced ANALYSIS-029 for MCP namespacing but never stated the format. Resolution: explicit colon-separator statement added per ADR-009 Decision 2 with `db-tools:postgres` example.

- [outcome] P0-7 resolved: MCP namespace format explicitly stated as colon-separated per ADR-009 Decision 2 (e.g., db-tools:postgres). #p0-resolution #mcp #namespacing

## P1 Resolutions (Batch)

- P1-1 (CLI in wrong phase): verified removed. CLI generation extracted to ADR-011. No residual content.
- P1-2 (upgrade order): changed from remove-then-install to install-then-remove per ANALYSIS-027. Avoids window with no plugin.
- P1-10 (uninstall flow): full 6-step uninstall flow added (locate, remove files, remove MCP entries, recompute hooks, offer dep uninstall, update lockfile).
- P1-11 (6 phases should be 4): added note that Phases 1-3 are read-only with no rollback needed. Kept 6-phase structure for clarity.
- P1-14 (MADR frontmatter): status, date, decision-makers added to YAML frontmatter.
- P1-16 (backup timing): .bak creation moved from Phase 6 to Phase 5 (before modifications).

- [outcome] P1 batch resolved: upgrade order fixed (P1-2), uninstall flow added (P1-10), read-only phases noted (P1-11), MADR frontmatter added (P1-14), backup timing fixed (P1-16) #p1-resolution #editorial

## Acceptance Outcome (2026-03-07)

ADR-010 achieved consensus after all P0 issues were resolved and incorporated into the ADR text. Final vote: 5 Accept, 1 D&C (Security). Key resolutions: CLI generation extracted to ADR-011 (P0-1), dependency auto-install kept with revised security model moving install commands from plugin manifests to trusted package code (P0-2), dependency rollback language made honest (P0-5), pre-install integrity verification added (P0-6), MCP namespace explicitly codified (P0-7). Advisor BLOCK lifted after revisions. No P0 issues remaining. Status changed from Proposed to Accepted.

- [outcome] ADR-010 accepted: 5 Accept, 1 D&C (Security). All P0s resolved. Advisor BLOCK lifted. Status changed to Accepted. #accepted #consensus
- [decision] Security D&C condition met: dependency commands sourced from trusted package code (platforms.config.json), not plugin manifests. Integrity verification added (IMP-008). #security #d-and-c

## Round 2 Results

### Consensus: ACCEPTED (5 Accept, 1 D&C)

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | ACCEPT | 0 | 2 | 1 |
| Critic | ACCEPT | 0 | 3 | 1 |
| Independent Thinker | ACCEPT | 0 | 1 | 2 |
| Security | D&C | 0 | 4 | 1 |
| Analyst | ACCEPT | 0 | 2 | 1 |
| Advisor | ACCEPT | 0 | 2 | 1 |

### Security D&C Reservations

- P1-6: Binary name shadowing -- need reserved name denylist
- P1-7: MCP server runs with full user permissions -- document as accepted risk
- P1-8: CI mode auto-confirms deps -- add --ci --no-deps flag
- P1-9: Lockfile integrity warn-only -- add SHA-256 integrity check

### Status Change

- ADR-010 status changed from Proposed to **Accepted**

- [outcome] ADR-010 Round 2 ACCEPTED: All P0 issues resolved, 5 Accept + 1 D&C consensus achieved #round-2
- [outcome] Security D&C: 4 P1 implementation concerns documented for tracking #security
