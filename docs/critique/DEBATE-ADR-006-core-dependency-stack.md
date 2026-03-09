---
title: DEBATE-ADR-006 Core Dependency Stack
type: critique
permalink: critique/debate-adr-006-core-dependency-stack
tags:
- critique
- adr-review
- adr-006
- dependencies
- debate-log
---

# DEBATE-ADR-006 Core Dependency Stack

## Review Summary

**ADR**: ADR-006 Core Dependency Stack
**Date**: 2026-03-07
**Round**: 1
**Status**: ACCEPTED (Round 2: Unanimous Accept 6/6)

## Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|-----|-----|-----|
| Architect | Accept with P1 revisions | 0 | 2 | 2 |
| Critic | Needs Revision | 1 | 2 | 0 |
| Independent Thinker | Concerns | 0 | 2 | 2 |
| Security | BLOCKED | 2 | 2 | 0 |
| Analyst | P1 revisions needed | 0 | 2 | 1 |
| Advisor | Accept with P0 condition | 1 | 1 | 1 |

**Consensus on decisions**: All 6 agents agree the dependency selections are sound. No agent recommends replacing any of the 8 new dependencies. Issues are about ADR documentation quality, version pinning, security controls, and a blocking validation gate for @clack/prompts.

## Consensus Points (All or Most Agents Agree)

1. @clack/prompts Bun stdin compatibility is the highest-risk item in the stack (6/6 agents)
2. "latest" version specifiers on shell-quote, validator, atomically are unacceptable (4/6 agents)
3. ADR-003 dependencies need at least acknowledgment of governance boundary (4/6 agents)
4. Keep as single ADR, do not split (6/6 agents)
5. gray-matter rejection is justified — CVE-2025-64718 verified on Snyk by analyst and independent thinker (5/6; critic initially questioned but analyst confirmed)
6. Dependency stack is lean and well-reasoned overall (6/6 agents)

## Conflicts Resolved

### Conflict 1: CVE-2025-64718 Validity

- **Critic**: P0 — CVE format valid but ID does not appear in NVD, may be fabricated
- **Analyst**: Verified on Snyk (SNYK-JS-JSYAML-13961110), CVSS 5.3 Medium
- **Independent Thinker**: Also verified on Snyk, GitLab Advisories, and Vulert
- **Resolution**: CVE is real. Critic's concern resolved by analyst/independent-thinker evidence. gray-matter disqualification stands. ADR should add CVSS score (5.3 Medium) for accuracy.

## P0 Issues

### P0-1: validator CVE-2025-12758 (CVSS 7.5)

**Raised by**: Security (P0)
**Description**: Active CVE in validator library. `isLength()` fails to count Unicode Variation Selectors, allowing payloads to bypass length validation. Enables DoS via memory exhaustion. Fixed in validator >= 13.15.22.
**Impact**: If the ADR-003 sanitization pipeline uses isLength(), this is directly exploitable.
**Resolution**: Pin validator >= 13.15.22. Verify which validator functions are used in sanitization pipeline.

### P0-2: No Supply Chain Controls Specified

**Raised by**: Security (P0), supported by Architect (P1 version pinning), Independent Thinker (P1)
**Description**: Zero supply chain protections specified. Three dependencies use "latest" version specifiers (shell-quote, validator, atomically). The September 2025 npm supply chain attack compromised 18+ packages with 2.6B weekly downloads. "latest" resolves to attacker's version immediately on compromise.
**Impact**: All users of the tool.
**Resolution**: (1) Pin ALL dependencies to exact versions. (2) Add lockfile integrity requirement. (3) Add CI audit gate. (4) Consider Socket.dev or similar monitoring.

### P0-3: @clack/prompts Bun Validation Gate

**Raised by**: Advisor (P0), supported by all 6 agents at P1-P2
**Description**: @clack/prompts has 4 documented Bun stdin bugs (#4835, #3099, #7033, #24615) including a NEW EPERM regression in Bun 1.3.2. ADR says "test during implementation" with no deadline, no go/no-go criteria. If @clack/prompts fails on Bun, all interactive flows (scaffolding, migration wizards, collection chooser) are dead.
**Impact**: All interactive CLI functionality.
**Resolution**: Add blocking gate with hard deadline: test @clack/prompts on Bun 1.3.x BEFORE any interactive flow implementation. Define pass/fail criteria. If it fails, switch to Inquirer.js before writing wizard code.

## P1 Issues

### P1-1: Pin ALL Dependencies to Exact Versions

**Raised by**: Security (P1), Architect (P1)
**Description**: shell-quote, validator, atomically, @gunshi/plugin-completion listed as "latest" or no version. Contradicts ADR's own security-first decision driver. gunshi is correctly pinned (0.29.2) but others are not.
**Resolution**: Specify exact version or tight semver range for every dependency in the summary table.

### P1-2: MADR 4.0 Frontmatter Non-compliance

**Raised by**: Architect (P1)
**Description**: status, date, decision-makers, consulted, informed appear in body not frontmatter. Breaks machine parsing.
**Resolution**: Move MADR-mandated fields to frontmatter.

### P1-3: Cold Start Target Inconsistency

**Raised by**: Critic (P1)
**Description**: ADR-006 Confirmation says "cold start under 200ms." ADR-005 says "sub-20ms target" and "CI fails at 50ms." 200ms is 10x the ADR-005 target. Cannot both be correct.
**Resolution**: Align with ADR-005: "under 50ms via bunx (per ADR-005 CI threshold); compiled binary under 150ms."

### P1-4: ADR-003 Dependencies Governance Boundary

**Raised by**: Critic (P1), Architect (P2), Analyst (mentioned)
**Description**: 4 dependencies (deepmerge, shell-quote, validator, atomically) listed in summary but receive zero analysis. ADR-003 NEG-006 flags deepmerge 3-year maintenance gap. ADR-006 does not acknowledge this.
**Resolution**: Add explicit statement: "ADR-003 dependencies governed by ADR-003. deepmerge staleness risk acknowledged (ADR-003 NEG-006)."

### P1-5: MCP SDK Pin >= 1.24.0

**Raised by**: Security (P1)
**Description**: CVE-2025-66414 (DNS rebinding, CVSS 7.6) fixed in 1.24.0. ADR lists "1.x" which could resolve to vulnerable version. Even though stdio transport is not directly affected, the vulnerable code is loaded.
**Resolution**: Pin @modelcontextprotocol/sdk >= 1.24.0.

### P1-6: Zod v4 / MCP SDK Peer Dependency Compatibility

**Raised by**: Analyst (P1)
**Description**: MCP SDK has peer dependency on "zod (v3.25+ or v4)." ADR does not confirm Zod v4 satisfies this without conflicts. Runtime errors possible if SDK imports v3 API surface.
**Resolution**: Verify and document tested combination.

### P1-7: Missing Dependency Governance Policy

**Raised by**: Advisor (P1)
**Description**: ADR defines initial 12 dependencies but no rule for how #13 gets approved. Without policy, dependency discipline erodes within 3 months.
**Resolution**: Add section: "New runtime dependencies require ADR amendment with Bun compatibility, security audit, and single-maintainer risk assessment."

### P1-8: MCP SDK Download Count Inaccurate

**Raised by**: Independent Thinker (P1)
**Description**: ADR claims 22.8M weekly downloads. Web research shows 5.65M. "82x more than fastmcp" drops to ~20x. Conclusion still holds but data is wrong.
**Resolution**: Correct the download count.

### P1-9: shell-quote Minimum Version >= 1.7.3

**Raised by**: Second Security Review (P1)
**Description**: CVE-2021-42740 (CVSS 9.8, CWE-78 command injection) patched in shell-quote 1.7.3. ADR lists "latest" with no minimum. If an older version resolves, OS command injection is exploitable.
**Resolution**: Pin shell-quote >= 1.7.3.

### P1-10: Hook Execution Method Unspecified

**Raised by**: Second Security Review (P1)
**Description**: ADR-003/006 do not specify whether hook commands use exec() (shell interpretation) or execFile() (direct execution). If exec() is used, shell-quote install-time parsing is bypassable at runtime. CWE-78 risk.
**Resolution**: Scope to ADR-004 but add forward reference noting this must be resolved.

### P1-11: gunshi Migration Estimate Unverified

**Raised by**: Independent Thinker (P1), Critic (P2)
**Description**: "2-3 days" citty migration estimate has no basis. citty ALSO supports lazy loading (arrow function subcommands), weakening gunshi's primary differentiator. API shapes differ (command definition, handler signatures, plugin model). For 20+ commands, realistic estimate is 1-2 weeks.
**Resolution**: Revise estimate to "1-2 weeks (unverified)" or validate with prototype spike.

### P1-1: Pin ALL Dependencies to Exact Versions

**Raised by**: Security (P1), Architect (P1)
**Description**: shell-quote, validator, atomically, @gunshi/plugin-completion listed as "latest" or no version. Contradicts ADR's own security-first decision driver. gunshi is correctly pinned (0.29.2) but others are not.
**Resolution**: Specify exact version or tight semver range for every dependency in the summary table.

### P1-2: MADR 4.0 Frontmatter Non-compliance

**Raised by**: Architect (P1)
**Description**: status, date, decision-makers, consulted, informed appear in body not frontmatter. Breaks machine parsing.
**Resolution**: Move MADR-mandated fields to frontmatter.

### P1-3: Cold Start Target Inconsistency

**Raised by**: Critic (P1)
**Description**: ADR-006 Confirmation says "cold start under 200ms." ADR-005 says "sub-20ms target" and "CI fails at 50ms." 200ms is 10x the ADR-005 target. Cannot both be correct.
**Resolution**: Align with ADR-005: "under 50ms via bunx (per ADR-005 CI threshold); compiled binary under 150ms."

### P1-4: ADR-003 Dependencies Governance Boundary

**Raised by**: Critic (P1), Architect (P2), Analyst (mentioned)
**Description**: 4 dependencies (deepmerge, shell-quote, validator, atomically) listed in summary but receive zero analysis. ADR-003 NEG-006 flags deepmerge 3-year maintenance gap. ADR-006 does not acknowledge this.
**Resolution**: Add explicit statement: "ADR-003 dependencies governed by ADR-003. deepmerge staleness risk acknowledged (ADR-003 NEG-006)."

### P1-5: MCP SDK Pin >= 1.24.0

**Raised by**: Security (P1)
**Description**: CVE-2025-66414 (DNS rebinding, CVSS 7.6) fixed in 1.24.0. ADR lists "1.x" which could resolve to vulnerable version. Even though stdio transport is not directly affected, the vulnerable code is loaded.
**Resolution**: Pin @modelcontextprotocol/sdk >= 1.24.0.

### P1-6: Zod v4 / MCP SDK Peer Dependency Compatibility

**Raised by**: Analyst (P1)
**Description**: MCP SDK has peer dependency on "zod (v3.25+ or v4)." ADR does not confirm Zod v4 satisfies this without conflicts. Runtime errors possible if SDK imports v3 API surface.
**Resolution**: Verify and document tested combination.

### P1-7: Missing Dependency Governance Policy

**Raised by**: Advisor (P1)
**Description**: ADR defines initial 12 dependencies but no rule for how #13 gets approved. Without policy, dependency discipline erodes within 3 months.
**Resolution**: Add section: "New runtime dependencies require ADR amendment with Bun compatibility, security audit, and single-maintainer risk assessment."

### P1-8: MCP SDK Download Count Inaccurate

**Raised by**: Independent Thinker (P1)
**Description**: ADR claims 22.8M weekly downloads. Web research shows 5.65M. "82x more than fastmcp" drops to ~20x. Conclusion still holds but data is wrong.
**Resolution**: Correct the download count.

## P2 Issues

| # | Issue | Raised By |
|---|-------|-----------|
| P2-1 | Decisions 3/8 lack two genuine alternatives (MADR requires this) | Architect |
| P2-2 | deepmerge staleness risk not explicitly acknowledged | Architect, Independent Thinker |
| P2-3 | chokidar Bun compatibility should be "likely but unverified" not "verified" | Analyst, Independent Thinker |
| P2-4 | Migration effort estimates (gunshi-to-citty, @clack-to-Inquirer) unverified | Independent Thinker, Critic |
| P2-5 | Upgrade thresholds (50+ plugins, 1000+ items) lack evidence basis | Advisor |
| P2-6 | watcher fallback is dead (528 downloads, last commit April 2024) | Advisor |
| P2-7 | Transitive dependency count unknown — "lean" claim unvalidated | Independent Thinker, Analyst |
| P2-8 | picocolors is transitive, inflates "12 dependencies" count | Architect, Independent Thinker |
| P2-9 | CVE-2025-64718 CVSS score (5.3 Medium) should be stated | Analyst, Independent Thinker |
| P2-10 | Frontmatter parser complexity understated (10 lines vs 50-100 production) | Independent Thinker |
| P2-11 | Multi-decision ADR needs partial supersession convention | Independent Thinker |
| P2-12 | gray-matter ^3.13.1 includes patched 3.14.2 — CVE argument slightly weaker (maintenance gap still valid) | Independent Thinker |
| P2-13 | validator.js function-to-context mapping unspecified (CWE-116) | Second Security Review |

## Key Evidence Discovered During Review

- [fact] CVE-2025-64718 verified: prototype pollution in js-yaml < 3.14.2, CVSS 5.3 Medium (Snyk SNYK-JS-JSYAML-13961110) #security
- [fact] validator CVE-2025-12758: Unicode length bypass in isLength(), CVSS 7.5, fixed >= 13.15.22 #security #new-cve
- [fact] MCP SDK CVE-2025-66414: DNS rebinding, CVSS 7.6, fixed >= 1.24.0 #security #new-cve
- [fact] @modelcontextprotocol/sdk actual downloads: ~5.65M/week (not 22.8M claimed) #correction
- [fact] September 2025 npm supply chain attack compromised 18+ packages with 2.6B weekly downloads #supply-chain
- [fact] Bun 1.3.2 introduced EPERM stdin regression (issue #24615) breaking @clack/prompts #regression
- [fact] chokidar Bun issues: #3978 (process exits, fixed) and #9031 (SIGABRT with bun --watch) #compatibility
- [fact] deepmerge 4.x: no known CVEs, but deepmerge-ts/ts-deepmerge/@75lb/deep-merge have CVEs #security

## Scope Decision

**Split recommended?** No (6/6 agents agree)
**Rationale**: Decisions are interdependent (gunshi drives shell completions, @clack drives colors, gray-matter CVE drives yaml choice). Single document provides complete dependency posture. Splitting creates overhead without governance benefit.

## Observations

- [fact] All 6 agents agree dependency selections are sound — issues are documentation quality and security controls #consensus
- [decision] CVE-2025-64718 conflict resolved: analyst and independent thinker verified on Snyk, critic's concern addressed #resolved
- [risk] @clack/prompts is unanimously flagged as highest-risk dependency — 4 Bun stdin bugs including new EPERM regression #dependency-risk
- [risk] validator has an active CVSS 7.5 CVE not mentioned in the ADR — immediate version pinning required #security
- [insight] "latest" version specifiers on 3 dependencies contradict the ADR's own security-first driver #consistency
- [insight] No supply chain controls in an ADR that cites supply chain as a decision driver is a credibility gap #consistency

## Relations

- reviews [[ADR-006 Core Dependency Stack]]
- relates_to [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[DEBATE-ADR-005-runtime-and-distribution-strategy]]
- relates_to [[ANALYSIS-017-cli-framework-comparison]]
- relates_to [[ANALYSIS-018-interactive-prompts-and-colors]]
- relates_to [[ANALYSIS-019-mcp-framework-and-file-watching]]
- relates_to [[ANALYSIS-020-frontmatter-and-markdown-processing]]
- relates_to [[ANALYSIS-021-data-storage-and-search]]

## @clack/prompts Bun Compatibility Test Results

**Date**: 2026-03-07
**Bun Version**: 1.3.8
**@clack/prompts Version**: 1.1.0
**Platform**: macOS arm64

| Test | Result |
|------|--------|
| Module import (16 APIs) | PASS |
| setRawMode(true/false) on TTY | PASS |
| sisteransi import | PASS |
| @clack/core import | PASS |
| spinner() | PASS |
| log.info/warn/error/success/message | PASS |
| note() | PASS |
| intro/outro | PASS |

- [fact] setRawMode() EPERM regression from Bun 1.3.2 (issue #24615) is FIXED in Bun 1.3.8 #bun-compat #verified
- [fact] All non-interactive @clack/prompts APIs work on Bun 1.3.8 #bun-compat #verified
- [decision] P0-3 validation gate retained as safety net but immediate risk is lower than feared. Pin Bun >= 1.3.8 #risk-mitigation

## Round 2: Convergence Check

**Date**: 2026-03-07
**Status**: CONSENSUS REACHED — UNANIMOUS ACCEPT (6/6)

### Changes Since Round 1

- P0-1: validator pinned >= 13.15.22 (CVE-2025-12758, CVSS 7.5)
- P0-2: "Dependency Security Controls" section added (version pinning, lockfile integrity, CI audit, governance, monitoring)
- P0-3: IMP-007 BLOCKING GATE for @clack/prompts with Bun 1.3.8 test results
- P1-1: All versions pinned in summary table
- P1-2: MADR frontmatter fields added
- P1-3: Cold start aligned with ADR-005 (50ms bunx, 150ms compiled)
- P1-4: ADR-003 governance boundary stated
- P1-5: MCP SDK pinned >= 1.24.0
- P1-6: IMP-008 Zod v4 / MCP SDK peer dep verification
- P1-7: Dependency governance policy added
- P1-8: MCP download count corrected to 5.65M
- P1-9: shell-quote pinned >= 1.7.3
- P1-10: IMP-009 hook exec deferred to ADR-004
- P1-11: gunshi migration estimate revised to 1-2 weeks

### Agent Verdicts (Round 2)

| Agent | Verdict | Concerns |
|-------|---------|----------|
| Architect | **Accept** | @gunshi/plugin-completion "latest" vs pinning policy (P2) |
| Critic | **Accept** | Same P2 |
| Independent Thinker | **Accept** | Same P2 + quarterly audit ownership (P2) |
| Security | **Accept** | None |
| Analyst | **Accept** | None |
| High-Level Advisor | **Accept** | None |

### Consensus

All 6 agents voted Accept. No blocks, no D&C positions. ADR-006 is ready for implementation.

### P2 Notes for Implementation

- @gunshi/plugin-completion should receive exact version pin when gunshi 0.29.2 is installed (compatible version will be known at that time)
- Quarterly dependency audit needs an owner and calendar trigger — create backlog item during implementation planning