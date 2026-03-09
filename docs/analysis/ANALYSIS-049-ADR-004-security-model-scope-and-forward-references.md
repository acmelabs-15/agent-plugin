---
title: ANALYSIS-049 ADR-004 Security Model Scope and Forward References
type: note
permalink: analysis/analysis-049-adr-004-security-model-scope-and-forward-references
tags:
- security
- ADR-004
- scope
- forward-references
- hooks
- trust-model
---

# ANALYSIS-049 ADR-004 Security Model Scope and Forward References

## Purpose

Comprehensive inventory of all security decisions deferred to ADR-004 across active ADRs. ADR-004 does not yet exist but is forward-referenced by ADR-001, ADR-002, ADR-003, ADR-006, ADR-012, and ADR-014. This is P0-1 in the Decision Completeness Audit — 11 security-critical decisions are blocked.

## Forward References by ADR

### From ADR-001 (Plugin Format and Manifest)
- IMP-006: Path traversal prevention, hook isolation, MCP approval gates, manifest integrity verification

### From ADR-002 (Target Platforms and Audiences)
- NEG-006: Instruction file modification as attack vector (malicious AGENTS.md/SKILL.md content manipulating AI agents)
- NEG-007: No centralized vetting in multi-source model

### From ADR-003 (Conflict Resolution and Namespacing)
- D2/D6: Per-hook user consent for blocking hooks, CWE-94 mitigation for hook command execution
- NEG-007: Strictest-wins composability conflict mitigation (per-plugin hook scope, user override, or conflict notification)

### From ADR-006 (Core Dependency Stack)
- IMP-009: Hook command execution method (exec vs execFile), CWE-78 shell-quote bypass at runtime

### From ADR-012 (Scaffolding and Content Management)
- D11/P0-6: analyze/analyze --fix trust model (AI suggesting code changes, approval model)

### From ADR-014 (Explicit Installation Model)
- IMP-001: CWE-22 path traversal for all path fields in plugin.json
- IMP-001: CWE-494 download integrity verification
- IMP-001: CWE-78 command injection for MCP server entries (show command + args, require confirmation)

## Proposed Decision Topics for ADR-004

10 decision topics to work through one at a time:

1. **Hook consent model** — When a plugin installs hooks, what level of user consent? Per-hook approval? Per-plugin blanket approval? Show-and-confirm at install time?
2. **Hook execution safety** — exec vs execFile choice, runtime bypass prevention for shell-quote validated commands
3. **Hook composability conflicts** — When strictest-wins causes plugin incompatibility (e.g., security plugin blocks Bash, deployment plugin needs Bash), what's the mitigation? Per-plugin scope? User override? Notification?
4. **Content injection prevention** — Malicious SKILL.md/AGENTS.md/rules content that manipulates AI agent behavior (prompt injection via plugin content)
5. **Source trust model** — Trust hierarchy for plugin sources. npm published vs GitHub public vs local path vs unknown. Does trust level affect what a plugin can do?
6. **Supply chain verification** — Plugin provenance verification. npm SHA-512 (already in ADR-014 IMP-001). GitHub release checksums. Content hash comparison on update.
7. **MCP server access control** — MCP server entries specify arbitrary commands written to platform configs. Approval model: display exact command/args, require confirmation, allowlist of safe commands (node, bun, npx, python, deno)?
8. **Plugin permission boundaries** — What can a plugin do and not do? Can a plugin modify files outside its declared paths? Can hooks access network? Filesystem boundaries?
9. **AI-assisted modification trust** — analyze/analyze --fix approval model. AI suggests code changes to plugin content. Show diff, require confirmation, apply modes (all/one-by-one/skip)?
10. **Path traversal prevention** — Enforcement at Zod schema level AND runtime before file reads. Reject ../, absolute paths, symlinks outside plugin/target directory for all path fields.

## Sequencing Recommendation

Topics 1-3 (hooks) are tightly coupled — resolve together.
Topics 4-5 (content/trust) are related — resolve together.
Topics 6-8 (supply chain/permissions) form a group.
Topics 9-10 are relatively independent.

## Observations

- [problem] ADR-004 forward-referenced by 6 active ADRs but does not exist, blocking 11 security-critical decisions #critical-blocker
- [fact] 10 decision topics identified spanning hooks, content, trust, supply chain, MCP, permissions, AI trust, and path safety #scope
- [requirement] Hook consent model must be decided before hooks can execute — ADR-003 D2 explicitly states hooks are NOT executable until ADR-004 establishes consent policy #blocking
- [fact] Topics 1-3 (hooks) are tightly coupled and should be resolved together #sequencing
- [insight] Content injection (topic 4) is unique to AI plugin systems — malicious plugin content can manipulate agent behavior without executing code #novel-threat
- [risk] Multi-source model with no registry means no centralized vetting. Trust model must compensate. #trust

## Relations

- part_of [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- relates_to [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-002-target-platforms-and-audiences]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[ADR-006-core-dependency-stack]]
- relates_to [[ADR-012-scaffolding-and-content-management]]
- relates_to [[ADR-014-explicit-installation-model-and-content-features]]
- relates_to [[ANALYSIS-048 Decision Completeness Audit Phase 1 Depth Assessment]]


