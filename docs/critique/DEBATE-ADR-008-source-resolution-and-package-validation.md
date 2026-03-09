---
title: DEBATE-ADR-008 Source Resolution and Package Validation
type: critique
permalink: critique/debate-adr-008-source-resolution-and-package-validation
tags:
- adr-review
- debate
- source-resolution
- ADR-008
- agent-plugin
---

# DEBATE-ADR-008 Source Resolution and Package Validation

## ADR Under Review

ADR-008 Source Resolution and Package Validation

## Final Consensus: ACCEPT

3 Accept, 3 Disagree-and-Commit, 0 Block, 0 Needs Revision

## Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | ACCEPT | 0 | 5 | 5 |
| Critic | ACCEPT | 0 | 5 | 3 |
| Independent Thinker | D&C | 1 | 3 | 3 |
| Security | D&C | 2 | 5 | 2 |
| Analyst | D&C | 0 | 3 | 7 |
| Advisor | ACCEPT | 0 | 3 | 3 |

## P0 Issues (Consolidated)

### P0-1: Memory Argument Contradiction (Independent Thinker)

ADR-008 argues against Bun.Archive because it "loads the entire archive into memory" but the ADR's own download pattern uses arrayBuffer() which also buffers everything in memory. Logical inconsistency in the justification, not the decision itself. Fix: drop the memory argument, rely on the 2-month maturity argument alone.

### P0-2: Zip Slip / Path Traversal in Archive Extraction (Security, CWE-22, CVSS 8.6)

The tar npm package has had CVE-2021-32803 and CVE-2021-32804. ADR does not mention path traversal prevention during extraction. Required: reject archive entries with ../, absolute paths, symlinks outside target directory.

### P0-3: No Package Integrity Verification (Security, CWE-494, CVSS 7.5)

No SHA-512 checksum validation against npm registry integrity field. No verification for GitHub downloads. Required: verify npm tarball integrity before extraction.

## P1 Issues (Consolidated, Deduplicated)

1. **go-getter citation factually inverted** (Analyst) -- go-getter puts FileDetector LAST, not first. Remove or correct the citation.
2. **ANALYSIS-025/028 override not transparent** (Analyst, Critic, Advisor) -- both analyses recommended the opposite (Bun.Archive, explicit github: prefix). ADR should explicitly acknowledge overriding them.
3. **ADR-007 amendment needed** (Architect, Advisor) -- Decision 2 modifies ADR-007's detection order without formal amendment.
4. **Bun.Archive reference inconsistency** (Architect, Critic) -- Decision 1 references Bun.Archive.files() for pre-check but Decision 3 rejects Bun.Archive. Clarify.
5. **owner/repo regex too permissive** (Critic, Analyst, Advisor) -- missing dots in owner segment, matches short local paths like src/utils.
6. **No GitHub API error handling** (Critic) -- rate limiting, offline mode, auth token handling not specified.
7. **tar dependency governance incomplete** (Architect) -- lacks ADR-006-style governance assessment.
8. **MADR 4.0 frontmatter fields missing** (Architect) -- status, date, decision-makers not in YAML frontmatter.
9. **No archive size guard** (Independent Thinker, Advisor, Security) -- no Content-Length check before arrayBuffer() buffering.
10. **Local path symlink validation missing** (Security, CWE-59) -- no realpathSync boundary check.
11. **plugin.json schema validation not security-scoped** (Security) -- no size limit, no Zod validation at discovery boundary.
12. **Observation categories** (Architect) -- [rationale] is not a valid category, use [insight].

## D&C Reservations

- Independent Thinker: Would prefer deferring bare owner/repo detection to v1.1. Would prefer Bun.Archive over tar (lower supply chain surface). Both positions are reversible.
- Security: Commits if P0 IMP notes for zip slip and integrity verification are added.
- Analyst: Commits but notes the go-getter citation is factually wrong and both analyses were overridden without transparency.

## Observations

- [decision] ADR-008 accepted by consensus: 3 Accept, 3 Disagree-and-Commit #adr-review
- [fact] 3 P0 issues identified: memory argument contradiction, zip slip, no integrity verification #security
- [fact] 12 P1 issues consolidated from 6 agent reviews #completeness
- [insight] All P0 issues are justification/implementation gaps, not design flaws. No decision changes required #architecture
- [risk] go-getter citation is factually inverted and must be corrected before implementation #accuracy

## Relations

- reviews [[ADR-008 Source Resolution and Package Validation]]
- relates_to [[ANALYSIS-025 Source Resolution Patterns]]
- relates_to [[ANALYSIS-028 Bun Builtin API Reliability]]

## P0 Resolution Log (2026-03-07)

- [outcome] P0-1 RESOLVED: Dropped memory buffering argument from Bun.Archive rejection rationale; arrayBuffer() workaround has same buffering pattern, making the argument contradictory. 2-month maturity remains the primary justification. #p0-resolution
- [outcome] P0-2 RESOLVED: Added IMP-007 requiring zip slip / path traversal prevention (CWE-22, CVSS 8.6). Mandates rejecting entries with ../, absolute paths, and symlinks outside target directory. References tar's filter and strip options. #p0-resolution #security
- [outcome] P0-3 RESOLVED: Added IMP-008 requiring package integrity verification (CWE-494, CVSS 7.5). npm tarballs verified via dist.integrity SHA-512; GitHub downloads verified via release asset SHA-256 when available. #p0-resolution #security
- [outcome] P1-1 RESOLVED: go-getter citation corrected. go-getter places FileDetector last, not first. ADR now states agent-plugin intentionally inverts this order. #p1-resolution
- [outcome] P1-4 RESOLVED: Bun.Archive pre-check reference in Decision 1 replaced with tar list command. Bun.Archive.files() noted as future consideration when API matures per IMP-004. #p1-resolution
- [outcome] P1-12 RESOLVED: Changed [rationale] observation categories to [insight] (valid category). #p1-resolution
- [fact] 3 new confirmation checklist items added: path traversal rejection, npm SHA-512 verification, GitHub SHA-256 verification #confirmation
- [fact] 2 new observations added to ADR-008: zip slip risk and integrity verification risk #observations