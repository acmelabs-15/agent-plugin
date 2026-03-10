---
title: ADR-004 Plugin Security Model
type: note
permalink: decisions/adr-004-plugin-security-model-1
tags:
- security
- hooks
- trust-model
- permissions
- content-injection
- supply-chain
---

# ADR-004 Plugin Security Model

---
status: "draft"
date: "2026-03-09"
decision-makers: "Peter Kloss, Agent Plugin Core Team"
consulted: "Architect agent, Security agent, Analyst agent"
informed: "All project contributors"
---

## Status

**Draft** (2026-03-09) — Decisions being added incrementally as topics are resolved.

Forward-referenced by ADR-001 (IMP-006), ADR-002 (NEG-006, NEG-007), ADR-003 (D2, D6, NEG-007), ADR-006 (IMP-009), ADR-012 (D11/P0-6), ADR-014 (IMP-001). See ANALYSIS-049 for full inventory.

## Context and Problem Statement

`@acmelabs/agx` installs third-party content (skills, agents, hooks, rules, MCP servers, AGENTS.md) into AI coding agent platform configurations. This content can:

1. **Execute arbitrary commands** via hooks and MCP server entries
2. **Manipulate AI agent behavior** via injected instructions in skills, agents, rules, and AGENTS.md
3. **Access the filesystem** via paths declared in plugin.json
4. **Modify platform configs** that control how AI agents operate

There is no centralized registry or vetting process. Plugins come from npm, GitHub repos, local paths, or configless directory scans. The security model must protect users from malicious or buggy plugins while keeping the installation experience practical.

## Decision Drivers

- Users install plugins from untrusted sources (no registry vetting)
- Hook commands execute on the user's machine with the user's permissions
- AI agent behavior is directly influenced by installed content (prompt injection risk)
- CI/non-interactive mode must work without consent prompts
- Plugin authors need a clear boundary of what's allowed
- Security must not make the tool impractical to use (balance security vs UX)

## Decisions

### Decision 1: Hook Consent Model — Automated Analysis with Human Fallback

Hook scripts declared by plugins are analyzed automatically during the `add` flow before installation.

**Two-tier consent:**

1. **Automated analysis for TypeScript, Python, and bash** — Static security analysis runs against the actual script content (not just the command string). If the analysis passes clean, the hook is approved with a green status indicator in the install summary. If the analysis flags issues, the wizard shows the findings and asks for human review/approval.

2. **Human review fallback** — If the script is NOT TypeScript, Python, or bash, OR if the automated analysis is unavailable, the wizard displays the script content and requires explicit human approval before the hook is installed.

**In CI mode (`--ci` / `--yes`):** Automated analysis still runs. Clean scripts are auto-approved. Flagged scripts or unanalyzable scripts cause the install to fail with an error (not silently skip). Use `--trust-hooks` flag to override in CI.

**What the analysis checks for** (per language):

Dangerous patterns common across all languages:
- Network access (HTTP requests, sockets, DNS lookups)
- File system access outside the project/plugin directory
- Environment variable reading (credential exfiltration risk)
- Process/child process spawning
- Dynamic code execution (eval, exec, Function constructor)
- Obfuscation patterns (base64 encoding of code)
- Known malicious signatures

**Analysis tooling**: See ANALYSIS-050 and ANALYSIS-051 for package research. Scoped to TypeScript, Python, bash — three languages that cover the vast majority of hook scripts.

### Decision 2: Hook Execution Safety — execFile with Bun TypeScript Default

Hook commands are executed via `execFile` (never `exec`). No hook command string is ever passed through a shell interpreter.

**Recommended hook format**: Bun TypeScript (`.ts`). Plugin authors write hook logic in TypeScript using Bun APIs. `agx create` scaffolds TypeScript hook stubs by default.

**Execution model**:

| Hook type | Runtime call | Status |
|-----------|-------------|--------|
| `.ts` (recommended) | `execFile("bun", ["run", "scripts/hook.ts"])` | Default, best analysis coverage |
| `.py` (supported) | `execFile("python3", ["scripts/hook.py"])` | Supported, ast-grep analysis |
| `.sh` (supported) | `execFile("bash", ["scripts/hook.sh"])` | Supported, shell-quote analysis |

**Key rules**:

1. The command string is always "run this file" — first token is a known runtime binary (`bun`, `python3`, `bash`), rest are arguments
2. All logic lives inside the script file, not in the command string
3. If `shell-quote.parse()` detects shell operators (`|`, `>`, `&&`, `;`) in the command string, static analysis flags it — the plugin author should move that logic into the script file using proper APIs (e.g., `Bun.spawn` for chaining, `Bun.write` for file output)
4. `execFile` treats shell operators as literal strings, preventing CWE-78 (OS Command Injection) even if an attacker manipulates a command string

**Combined with Layer 3**: The `execFile`'d process runs inside `@anthropic-ai/sandbox-runtime` with restricted filesystem/network access.

**What this resolves**:
- ADR-006 IMP-009: exec vs execFile decision — always execFile
- ADR-006 IMP-009: shell-quote bypass at runtime — not applicable since shell is never invoked

### Decision 3: Hook Composability — Independent Execution, Strictest-Wins Exit Codes

When multiple plugins register hooks for the same event, all hooks execute independently. No plugin can prevent another plugin's hook from running.

**Execution model**:

1. All hooks registered for the event fire — no short-circuit on first failure
2. Each hook runs in its own sandboxed process (isolated via `@anthropic-ai/sandbox-runtime`)
3. If ANY hook returns non-zero exit code, the overall event result is "fail" (strictest-wins)
4. The CLI reports which specific hook(s) failed so the user can diagnose the conflict

**Why this works**: With Decision 2 (execFile per hook), each hook is a separate process invocation. Plugin A's hook cannot interfere with Plugin B's hook at the process level. "Strictest-wins" applies only to exit code aggregation, not to runtime availability or capability restriction.

**User resolution**: If two plugins' hooks conflict (e.g., one always fails when the other is installed), the user decides whether to keep or remove the conflicting plugin. The CLI surfaces which hooks failed to inform this decision.

**What this resolves**:
- ADR-003 NEG-007: Strictest-wins composability conflict mitigation — resolved by independent execution with exit code aggregation
- ANALYSIS-049 Topic 3: Hook composability conflicts — no plugin can block another plugin's hooks

### Decision 4: Content Injection — Accepted Risk, Not Mitigated by Plugin Manager

Prompt injection via plugin instruction files (skills, agents, rules, AGENTS.md) is an accepted risk that the plugin manager does NOT attempt to detect or prevent.

**Rationale**: Plugin instruction files ARE instructions by design. Automated detection of prompt injection in natural language instruction content is an unsolved problem — ML classifiers produce false positives on legitimate instruction content because the content is indistinguishable in form from injection attempts. Regex-based pattern matching catches only trivially naive attacks that any real attacker would evade.

**What we rely on instead**:

1. **Source trust model** (Decision 5) — provenance and publisher reputation are stronger signals than content scanning
2. **Platform-level protections** — AI platforms (Claude Code, Cursor, etc.) enforce their own boundaries around tool permissions, confirmation prompts, and sandboxing. This is the platform's responsibility, not the plugin manager's
3. **Visibility and auditability** — `agx list --content` shows all installed instruction content per plugin. Users can inspect any file at any time. The `add` flow shows file counts and types being installed
4. **Community signals** — download counts, GitHub stars, known publishers provide trust indicators

**Explicitly out of scope**:
- No automated content scanning during `add` flow
- No prompt injection detection (regex, ML, or API-based)
- No content approval wizard for instruction files (hooks have their own consent model per D1)

**What this resolves**:
- ADR-002 NEG-006: Instruction file modification as attack vector — accepted risk, mitigated by trust model + platform protections + visibility
- ANALYSIS-049 Topic 4: Content injection prevention — resolved as accepted risk

### Decision 5: Source Trust Model — Informational Only, No Capability Gating

Trust level is informational metadata displayed during `add`, not a capability gate. All plugin sources receive identical security treatment regardless of origin.

**Trust signals displayed during `add`** (from data already available via source resolution):

| Source type | Signals shown | Data source |
|-------------|--------------|-------------|
| npm (published) | Publisher name, weekly downloads, SHA-512 integrity | npm registry API (already fetched during resolution) |
| GitHub repo | Stars, last commit date, contributor count | GitHub API (already fetched during resolution) |
| Local path | "Local source — you control this" | Filesystem |
| Configless scan | "Project-local — discovered from directory structure" | Filesystem |

**Key principle**: All sources go through the same security checks (hook static analysis D1, execFile D2, sandbox D3). Trust level does NOT gate capabilities. A local plugin's hooks get the same analysis as an npm plugin's hooks.

**No code for trust-based behavior**: There is no trust tier system, no restricted mode for untrusted sources, no elevated privileges for trusted sources. The signals are displayed as context to help the user make an informed install decision — nothing more.

**Rationale**: Trust-based capability gating creates false confidence. A "trusted" npm package can be compromised (supply chain attacks). An "untrusted" local path might be the user's own code. Equal treatment is both simpler and more honest.

**What this resolves**:
- ADR-002 NEG-007: No centralized vetting in multi-source model — resolved by equal treatment with informational trust signals
- ANALYSIS-049 Topic 5: Source trust model — resolved as informational only

**What this resolves:**
- ADR-003 D2/D6: Per-hook user consent for blocking hooks
- ADR-003 D2: "Hook commands are NOT executable until ADR-004 establishes the consent policy" — this decision establishes that policy

## Observations

- [problem] ADR-004 forward-referenced by 6 active ADRs but did not exist until now, blocking 11 security-critical decisions #critical-blocker
- [fact] 10 decision topics identified: hook consent, hook execution safety, hook composability, content injection, source trust, supply chain, MCP access control, plugin permissions, AI modification trust, path traversal #scope
- [requirement] Hook consent model must be decided before hooks can execute — ADR-003 explicitly states hooks are NOT executable until ADR-004 establishes consent policy #blocking

## Relations

- relates_to [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-002-target-platforms-and-audiences]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[ADR-006-core-dependency-stack]]
- relates_to [[ADR-012-scaffolding-and-content-management]]
- relates_to [[ADR-014-explicit-installation-model-and-content-features]]
- relates_to [[ANALYSIS-049 ADR-004 Security Model Scope and Forward References]]

## Decision 5: Source Trust Model — Informational Only, No Capability Gating
### Decision 5: Source Trust Model — No Trust System

There is no trust model. Source type (npm, GitHub, local path, configless scan) is irrelevant to the security model. All sources receive identical treatment — same hook analysis (D1), same execFile execution (D2), same sandboxing (D3). No trust signals are displayed, no trust tiers exist, no capability gating based on source origin.

**Rationale**: Trust-based systems create false confidence. A "trusted" npm package can be compromised. An "untrusted" local path might be the user's own code. Rather than building an unreliable trust hierarchy, we apply uniform security checks to everything.

**What this resolves**:
- ADR-002 NEG-007: No centralized vetting in multi-source model — resolved by uniform security treatment regardless of source
- ANALYSIS-049 Topic 5: Source trust model — resolved as no trust system needed

### Decision 6: Supply Chain Verification — Rely on npm's Built-in Integrity

No custom supply chain verification. npm packages already receive SHA-512 integrity checks as part of npm's native package resolution. No additional content hashing, signature verification, or provenance tracking is implemented by the plugin manager.

**Rationale**: Consistent with D4 (accepted risk on content) and D5 (no trust system). Building custom verification on top of npm's existing integrity checks adds complexity without meaningful security improvement for our use case. npm's integrity verification is battle-tested and covers the primary supply chain vector (package tampering).

**What this resolves**:
- ADR-014 IMP-001: CWE-494 download integrity verification — covered by npm's native SHA-512 integrity
- ANALYSIS-049 Topic 6: Supply chain verification — resolved by deferring to npm

### Decision 7: MCP Server Access Control — Platform Responsibility, Not Ours

MCP server entries declared by plugins are written to platform configuration files. The plugin manager does NOT analyze, gate, or sandbox MCP server commands. Runtime security of MCP server processes is entirely the platform's responsibility.

**Rationale**: The plugin manager writes config entries (command, args, env). The platform (Claude Code, Cursor, etc.) launches and manages MCP server processes. We have no control over how or when the platform starts these processes, what sandboxing it applies, or what permissions it grants. Attempting to analyze MCP server commands at install time provides no meaningful security since the platform controls execution.

**Consistent with**: D4 (content injection — accepted risk, platform responsibility) and D5 (no trust system).

**What this resolves**:
- ADR-014 IMP-001: CWE-78 command injection for MCP server entries — resolved as platform responsibility
- ANALYSIS-049 Topic 7: MCP server access control — resolved by deferring to platform

### Decision 8: Plugin Permission Boundaries — Platform Responsibility, No Sandbox or Permissions

The plugin manager does NOT implement sandboxing, permission declarations, or filesystem/network restrictions for plugin content. All runtime restrictions (filesystem access, network access, process permissions) are the platform's responsibility.

**No permission system**: There is no `"permissions"` field in `plugin.json`. No sandbox configuration. No `@anthropic-ai/sandbox-runtime` integration. Platforms like Claude Code, Cursor, Copilot, etc. already provide their own settings for controlling what AI agents and their tools can access. These platform-level controls are more reliable and better maintained than anything the plugin manager could implement.

**Rationale**: The plugin manager installs files and writes config entries. It does not execute AI agents, run MCP servers, or control how platforms use the installed content. Runtime security belongs to the platform layer, which already has purpose-built permission and sandboxing systems. Reimplementing these controls in the plugin manager would be less reliable and create a false sense of security.

**Consistent with**: D4 (content injection — platform responsibility), D7 (MCP access control — platform responsibility).

**What this resolves**:
- ADR-001 IMP-006: Hook isolation — resolved as platform responsibility
- ANALYSIS-049 Topic 8: Plugin permission boundaries — resolved by deferring to platform

### Decision 9: AI-Assisted Modification Trust — Platform Responsibility

The `analyze`/`analyze --fix` commands (ADR-012 D11) invoke the platform's AI to review and suggest improvements to plugin content. The plugin manager does NOT implement a custom approval model for AI-suggested changes.

**Rationale**: The platform's AI already has its own approval flow for code changes (e.g., Claude Code shows diffs and asks for confirmation). Adding a second approval layer in the plugin manager would be redundant and confusing. The platform controls the AI, the AI suggests changes, the platform's UX handles user consent.

**What this resolves**:
- ADR-012 D11/P0-6: analyze/analyze --fix trust model — resolved by deferring to platform's existing AI approval UX
- ANALYSIS-049 Topic 9: AI modification trust — resolved as platform responsibility

### Decision 10: Path Traversal Prevention — Zod Schema Validation

All path fields in `plugin.json` are validated at parse time via Zod schema refinements. Paths containing `..`, absolute paths, or symlinks outside the plugin/target directory are rejected during the `add` flow before any files are read or written.

**Implementation**: Zod `.refine()` checks on every path field in the manifest schema:
- Reject `..` path segments
- Reject absolute paths (must be relative to plugin root)
- Reject paths that resolve outside the plugin directory after normalization (`path.resolve` + `startsWith` check)

**When it runs**: During manifest parsing in the `add` flow — before any file I/O occurs. A malformed path fails validation and aborts the install with a clear error message.

**What this resolves**:
- ADR-001 IMP-006: Path traversal prevention
- ADR-014 IMP-001: CWE-22 path traversal for all path fields in plugin.json
- ANALYSIS-049 Topic 10: Path traversal prevention — resolved by Zod schema validation at parse time

## Decision 10: Path Traversal Prevention — Zod Schema Validation
### Decision 10: Path Traversal Prevention — Not Needed

No path traversal prevention is implemented. Plugin sources are already scoped by their distribution mechanism: npm tarballs cannot contain traversal paths, git clones are in isolated directories, and local paths are explicitly trusted by the user. Adding validation would be solving a problem that doesn't exist in practice.

**What this resolves**:
- ADR-001 IMP-006: Path traversal prevention — resolved as unnecessary given scoped sources
- ADR-014 IMP-001: CWE-22 path traversal — resolved as unnecessary given scoped sources
- ANALYSIS-049 Topic 10: Path traversal prevention — resolved as not needed
