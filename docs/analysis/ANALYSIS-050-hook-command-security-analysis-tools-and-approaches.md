---
title: ANALYSIS-050 Hook Command Security Analysis Tools and Approaches
type: analysis
permalink: analysis/analysis-050-hook-command-security-analysis-tools-and-approaches-1
tags:
- security
- hooks
- shell-analysis
- sandboxing
- npm-packages
- ADR-004
- supply-chain
---

# ANALYSIS-050 Hook Command Security Analysis Tools and Approaches

## 1. Objective and Scope

**Objective**: What npm packages, tools, and approaches exist to analyze shell/hook commands for safety during the `agent-plugin add` installation flow?

**Scope**: Static analysis of shell commands, runtime sandboxing, pattern-based detection, LLM-assisted analysis, and ecosystem precedents from VS Code, Homebrew, GitHub Actions, and Claude Code. Focused on tools available as npm packages or usable from Bun runtime.

**Out of scope**: Full threat model (deferred to ADR-004), content injection analysis (ANALYSIS-049 topic 4), path traversal (topic 10).

## 2. Context

`@acmelabs-15/agent-plugin` installs plugins that declare hook commands in `plugin.json`. These commands execute on the user's machine with the user's permissions. ANALYSIS-049 identifies 3 tightly coupled hook security topics (consent, execution safety, composability) blocking ADR-004.

The tool already has `shell-quote` (15.1M weekly downloads) in the dependency stack per ADR-006. `shell-quote` parses shell strings into token arrays, identifying operators (`|`, `>`, `<`, `&&`, `||`, `;`), glob patterns, and comments. This provides the foundation for pattern-based analysis.

ADR-003 D2 explicitly states hooks are NOT executable until ADR-004 establishes consent policy. ADR-006 IMP-009 requires deciding between `exec` vs `execFile` and handling `shell-quote` bypass at runtime.

## 3. Approach

**Methodology**: Web research across npm registry, GitHub, security tooling ecosystems, and platform documentation. Evaluated packages for download counts, maintenance status, Bun compatibility, and practical applicability.

**Tools Used**: WebSearch across npm, GitHub, Socket.dev, Snyk, Deno/Node.js/Bun documentation, Anthropic engineering blog, VS Code security docs, Homebrew docs, GitHub Actions security guides.

**Limitations**: npm download counts are point-in-time (March 2026). Bun compatibility for OS-level sandboxing tools not independently verified. No hands-on testing of any packages.

## 4. Data and Analysis

### RQ1: Shell Command Parsing and Dangerous Pattern Detection

| Package | Weekly Downloads | Last Updated | Purpose | Bun Compatible |
|---------|-----------------|--------------|---------|----------------|
| `shell-quote` | 15,137,600 | < 1 year ago | Parse shell strings into token arrays | Yes (pure JS) |
| `@mergesium/shell-quote` | Unknown | Active | Fork with additional features | Yes (pure JS) |
| `shell-parse` | Low | Unknown | Alternative shell parser | Likely (pure JS) |
| `@cronvel/shell-quote` | Low | Unknown | Fork variant | Likely (pure JS) |

**Key finding**: No dedicated npm package exists for "dangerous shell pattern detection." This capability must be built on top of `shell-quote`'s token output. `shell-quote.parse()` returns typed tokens that identify operators (`||`, `&&`, `;`, `|`, `>`, `<`, `|&`), globs, and comments. A custom analyzer can classify these tokens against a risk taxonomy.

**Proposed risk taxonomy for hook commands** (derived from npm supply chain attack patterns):

| Risk Category | Detection Method | Patterns |
|---------------|-----------------|----------|
| Network access | Token matching | `curl`, `wget`, `nc`, `ncat`, `ssh`, `scp`, `rsync`, `ftp` |
| File exfiltration | Token + operator | Redirect operators (`>`, `>>`) to paths outside plugin scope |
| Credential access | Token matching | `~/.npmrc`, `~/.ssh/`, `~/.aws/`, `~/.gnupg/`, env vars like `$NPM_TOKEN`, `$AWS_SECRET` |
| Reverse shells | Token + pipe | `bash -i`, `/dev/tcp/`, `mkfifo`, `nc -e`, `/bin/sh -c` piped to network |
| Crypto miners | Token matching | `xmrig`, `minerd`, `cpuminer`, `hashrate` |
| Privilege escalation | Token matching | `sudo`, `su`, `chmod 777`, `chown`, `setuid` |
| Package execution | Token matching | `npm install`, `pip install`, `gem install` (secondary payload download) |
| Environment exfiltration | Token + pipe | `env`, `printenv`, `set` piped to network commands |
| Process manipulation | Token matching | `kill`, `pkill`, `killall` targeting non-plugin processes |
| Obfuscation | Token analysis | Base64-encoded payloads (`echo ... \| base64 -d \| bash`), `eval`, `exec` |

### RQ2: Static Analysis Tools for Shell Scripts (npm/WASM)

| Package | Weekly Downloads | Approach | Bun Compatible | Practical for Use Case |
|---------|-----------------|----------|----------------|----------------------|
| `shellcheck` (npm) | 642,292 | Downloads native ShellCheck binary | Partial (binary works, npm wrapper uses Node) | Medium: good for `.sh` files, overkill for single-line hook commands |
| `node-shellcheck` | 4 | Downloads ShellCheck binary | Partial | Low: negligible adoption |

**Key finding**: No WASM version of ShellCheck exists on npm. Both npm packages download the native Haskell binary. ShellCheck excels at finding bugs in shell scripts but is designed for script files, not single-line commands. It detects quoting errors, undefined variables, deprecated syntax, and portability issues. It does NOT detect malicious intent (network exfiltration, credential theft).

**Verdict**: ShellCheck is complementary but insufficient. It catches buggy hooks (unquoted variables, portability issues) but not malicious hooks. Worth considering as an optional quality check for plugin authors, not as a security gate for consumers.

### RQ3: Runtime Sandboxing and Permission Restriction

#### OS-Level Sandboxing

| Tool | Distribution | Mechanism | Bun Compatible | Practical for Use Case |
|------|-------------|-----------|----------------|----------------------|
| `@anthropic-ai/sandbox-runtime` | npm | macOS: sandbox-exec (Seatbelt), Linux: bubblewrap + seccomp BPF | Yes (OS-level, runtime-agnostic) | High |
| `node-safe` (`@berstend/node-safe`) | npm (19 downloads/week) | macOS: sandbox-exec (Seatbelt) | macOS only, Node-specific wrapper | Low: 19 downloads, macOS-only, unmaintained (1 year) |
| `sandbox-shell` (agentic-dev3o) | GitHub | macOS Seatbelt CLI for developers | macOS only | Low: new project, macOS-only |

**`@anthropic-ai/sandbox-runtime` is the standout option.** It provides:

- Filesystem restrictions via dynamically generated Seatbelt profiles (macOS) or bubblewrap (Linux)
- Network restrictions via HTTP/SOCKS5 proxy with domain allowlist/denylist
- No container required (pure OS-level)
- Zero runtime dependencies
- Applies to ALL subprocesses (not just Node/Bun)
- Used in production by Claude Code

**Limitation**: Sandboxing is a runtime control, not a static analysis tool. It prevents damage but does not help users understand WHAT a command will do before approving it. Both approaches (static analysis + runtime sandboxing) are needed.

#### Runtime Permission Models

| Runtime | Permission Model | Child Process Control | Status |
|---------|-----------------|----------------------|--------|
| Node.js v25+ | `--permission` flag with `--allow-child-process`, `--allow-fs-read`, `--allow-fs-write` | `--allow-child-process=git,curl` or `=false` | Stable (was experimental until v23.5.0) |
| Deno | Built-in deny-by-default | `--allow-run=git,node` (allow specific executables) | Stable, but subprocesses escape the sandbox |
| Bun | None | No built-in permission model | Open issue #6617, no timeline |

**Key Deno caveat**: `--allow-run=cat` lets `cat` read ANY file regardless of `--allow-read` restrictions. Subprocesses are not sandboxed. This makes Deno's model insufficient for our use case where hooks ARE subprocesses.

**Key Node.js caveat**: The permission model has had bypass vulnerabilities (symlink bypass, UDS bypass). Adoption is low.

**Bun has no permission model.** Issue #6617 is open with no timeline. Issue #25929 specifically requests Bun as a secure sandbox for AI agent code execution. This is a gap for our runtime.

#### JavaScript Sandboxing (NOT applicable to shell commands)

| Package | Purpose | Why Not Applicable |
|---------|---------|-------------------|
| `v8-sandbox` | Sandboxed V8 context for JS | Hooks are shell commands, not JS |
| `vm2` (deprecated) | Node VM sandbox | Critical escape vulnerability (CVSS 9.8), deprecated |
| `process-sandbox` | Node VM for child processes | Uses Node VM API, not shell isolation |

### RQ4: Ecosystem Precedents

#### VS Code Extensions

**Model**: No sandbox for extensions. The Extension Host Process has full access to filesystem, network, and host machine. Verified marketplace badge does not guarantee safety. Malware scanning occurs at publish time (sandbox environment), not at install time.

**Key finding**: VS Code's security model is widely criticized. In February 2026, critical flaws were found in 4 extensions with 128M+ installs. In October 2025, 100+ extensions were found to expose developers to supply chain risks. VS Code relies on Workspace Trust (restricted mode) to limit extension capabilities, but this is opt-in and coarse-grained.

**Lesson for agent-plugin**: Do not follow VS Code's model. Their approach of trusting extensions at install time has proven insufficient. Pre-install analysis and explicit consent are necessary.

#### Homebrew

**Model**: Human-reviewed PRs for homebrew-core/homebrew-cask. Automated `brew audit` checks for style, license, and appropriateness. GitHub Actions run security and style checks on submissions. No sandboxing at install time.

**Key finding**: Homebrew's security comes from centralized review (maintainer approval on every PR). This works for a curated registry but does not apply to agent-plugin's decentralized, no-registry model.

**Lesson for agent-plugin**: Without centralized review, the tool must compensate with stronger local analysis. Homebrew's `brew audit` pattern (automated checks that flag suspicious patterns) is worth borrowing.

#### GitHub Actions

**Model**: CodeQL-based static analysis of workflow files. Detects shell injection, untrusted input interpolation, missing permissions, and tainted data flow. 18 specialized queries for Actions security. Tracks tainted data through steps, jobs, composite actions, and reusable workflows.

**Key finding**: GitHub's approach is the closest precedent. CodeQL's Actions analysis identifies dangerous patterns in workflow YAML, including shell command injection via untrusted inputs. The taint-tracking model (source -> sink analysis) is sophisticated.

**Lesson for agent-plugin**: CodeQL is not available as an npm package for direct use. But the PATTERN of analyzing shell commands for known dangerous sinks (redirect to file, pipe to network, environment variable interpolation) is directly applicable. Build a simpler, focused version of this analysis.

#### Claude Code

**Model**: Permission-based with three tiers: auto-allowed (read-only, safe commands), ask-permission (modifications, new commands), blocked (known dangerous). Uses `@anthropic-ai/sandbox-runtime` for OS-level filesystem and network restrictions. Hooks execute automatically with no confirmation (vulnerability fixed August 2025 after disclosure).

**Key finding**: Claude Code's tiered permission model is the most relevant precedent. The combination of (1) command classification, (2) user confirmation, and (3) runtime sandboxing is exactly what agent-plugin needs. The hooks vulnerability (CVE-2025-59536) demonstrates that auto-executing hooks without confirmation is a real attack vector.

**Lesson for agent-plugin**: Adopt Claude Code's layered approach. Static classification first (show risk assessment), user confirmation second (informed consent), runtime sandboxing third (defense in depth).

#### npm (LavaMoat)

**Model**: `@lavamoat/allow-scripts` creates an allowlist of packages authorized to run lifecycle scripts. All other scripts are replaced with a trivial no-op. Configuration stored in package.json.

**Key finding**: LavaMoat's approach is relevant to our hook consent model. Rather than analyzing WHAT scripts do, it controls WHETHER scripts run at all. Binary allow/deny per package. The September 2025 Shai-Hulud attack (compromising 18+ packages via lifecycle scripts) validated this approach.

**Lesson for agent-plugin**: The allow/deny-per-plugin model for hooks is a valid baseline. But agent-plugin can do better by also analyzing hook content before the allow/deny decision.

### RQ5: LLM/AI-Assisted Analysis

| Approach | Feasibility | Reliability | Practical for Use Case |
|----------|------------|-------------|----------------------|
| Send hook commands to LLM for risk assessment | High (MCP server already embedded) | Medium: 45% of AI-generated code contains flaws (Veracode 2025). Hallucination risk. | Medium |
| Aardvark (OpenAI) for code scanning | Low (proprietary, not embeddable) | Unknown | Not applicable |
| Custom prompt with risk taxonomy | High | Medium-High with constrained output | High for augmentation |

**Key finding**: Using the embedded MCP server to send hook commands to an LLM for analysis is technically feasible and could provide natural-language risk explanations. However, LLM analysis should NEVER be the sole security gate.

**Risks of LLM-only analysis**:

- Hallucination: LLM may declare a dangerous command safe (false negative)
- Adversarial evasion: Attacker can craft commands that look benign to LLM but execute maliciously
- Latency: API call adds delay to installation flow
- Cost: Each analysis requires API tokens
- Availability: Requires network connectivity

**Recommended role for LLM**: Supplementary analysis AFTER deterministic pattern matching. Use LLM to generate human-readable explanations of detected risks, not to detect risks. Example: pattern matcher flags `curl ... | bash` as network access + pipe execution. LLM explains: "This command downloads and immediately executes a remote script, which could contain any code."

## 5. Results

### Package Recommendations

| Priority | Package | Purpose | Action |
|----------|---------|---------|--------|
| P0 | `shell-quote` (already in stack) | Parse hook commands into tokens for pattern analysis | Build custom risk analyzer on top |
| P1 | `@anthropic-ai/sandbox-runtime` | Runtime sandbox for hook execution | Evaluate for hook execution phase |
| P2 | `shellcheck` (npm) | Optional quality check for plugin authors | Consider for `agent-plugin lint` command |
| P3 | LLM via MCP | Supplementary risk explanations | Implement after deterministic analysis |

### No Existing Package Solves This Problem

No npm package exists that analyzes shell commands for malicious intent. The security tooling ecosystem focuses on:

- Supply chain (Socket.dev, Snyk): Analyze packages, not commands
- Static analysis (ShellCheck): Finds bugs, not malice
- Sandboxing (sandbox-runtime, node-safe): Prevents damage at runtime, does not inform users pre-execution
- Permissions (Node.js, Deno): Restrict capabilities, do not analyze commands

**The hook command risk analyzer must be custom-built.** It will use `shell-quote` for tokenization and apply a curated risk taxonomy.

## 6. Discussion

### Recommended Architecture: Three-Layer Security

```text
Layer 1: Static Analysis (pre-approval)
  shell-quote tokenization -> risk pattern matching -> risk report
  Deterministic, fast, no external dependencies
  Shows user exactly what each hook does and what risks it carries

Layer 2: User Consent (at install time)
  Display risk report with per-hook detail
  Require explicit approval for each risk category
  Support --yes flag for CI with documented risk acceptance

Layer 3: Runtime Sandboxing (at execution time)
  @anthropic-ai/sandbox-runtime for filesystem + network restrictions
  Restrict hooks to plugin directory + declared paths only
  Block network access unless explicitly declared and approved
```

### Custom Risk Analyzer Design

**Input**: Hook command string from `plugin.json`

**Process**:

1. `shell-quote.parse(command)` produces token array
2. Classify each token against risk taxonomy (see RQ1 table)
3. Classify operators (pipe, redirect, background, chaining)
4. Check for obfuscation patterns (base64, eval, backtick substitution)
5. Resolve first token to known command categories (safe: echo/cat/test, risky: curl/wget/nc, dangerous: rm -rf/dd/mkfs)
6. Generate structured risk report with severity levels

**Output**: Risk report per hook with:

- Command summary (what it does in plain language)
- Risk level (none/low/medium/high/critical)
- Risk categories triggered (network/filesystem/credentials/escalation)
- Specific tokens that triggered each risk
- Whether the command can be sandboxed safely

### Bun Compatibility Summary

| Component | Bun Status |
|-----------|-----------|
| `shell-quote` (pure JS) | Works |
| Custom risk analyzer (pure JS) | Works |
| `@anthropic-ai/sandbox-runtime` (OS-level) | Works (runtime-agnostic, wraps OS primitives) |
| `shellcheck` npm wrapper | Needs testing (downloads native binary) |
| Node.js `--permission` model | Not applicable (Bun has no equivalent) |
| LLM via MCP | Works (HTTP calls) |

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Build custom risk analyzer using `shell-quote` tokenization | No existing package; this is the core of pre-approval analysis | 2-3 days |
| P0 | Define risk taxonomy with severity levels and detection patterns | Foundation for consistent, auditable risk assessment | 1 day |
| P1 | Evaluate `@anthropic-ai/sandbox-runtime` for hook execution | Production-proven, OS-level, works with Bun, covers macOS + Linux | 1-2 days |
| P1 | Implement per-hook consent flow showing risk report before approval | Directly addresses ADR-004 topic 1 (hook consent model) | 2-3 days |
| P2 | Add LLM-powered risk explanation as supplement to pattern matching | Better UX for non-technical users; leverage existing MCP server | 1 day |
| P2 | Consider `shellcheck` integration for `agent-plugin lint` | Helps plugin authors write correct hooks; not a security gate | 0.5 day |
| P3 | Monitor Bun permission model progress (issue #6617) | Long-term: native sandboxing would simplify architecture | Ongoing |

## 8. Conclusion

**Verdict**: Proceed with custom risk analyzer + runtime sandboxing

**Confidence**: High

**Rationale**: No off-the-shelf npm package exists for shell command security analysis. The problem is well-scoped: `shell-quote` provides tokenization, a curated risk taxonomy provides detection rules, and `@anthropic-ai/sandbox-runtime` provides defense-in-depth at execution time. This three-layer approach (static analysis, informed consent, runtime sandboxing) matches the pattern proven by Claude Code and addresses the specific gaps identified in ANALYSIS-049 topics 1-3.

### User Impact

- **What changes for you**: Every hook command in a plugin gets a risk assessment before you approve installation. You see exactly what commands will run and what they can access.
- **Effort required**: 5-8 days to build the risk analyzer, consent flow, and sandbox integration.
- **Risk if ignored**: Hooks execute arbitrary commands with user permissions. Without analysis, a malicious plugin hook can exfiltrate credentials, install malware, or establish persistence. The September 2025 npm supply chain attack compromised 18+ packages with 2.6B weekly downloads using this exact vector (lifecycle scripts executing malicious commands).

## 9. Appendices

### Sources Consulted

- [shell-quote on npm](https://www.npmjs.com/package/shell-quote) -- 15.1M weekly downloads, parse/quote shell commands
- [shell-quote security analysis (Socket.dev)](https://socket.dev/npm/package/shell-quote)
- [shellcheck on npm](https://www.npmjs.com/package/shellcheck) -- 642K weekly downloads, native binary wrapper
- [ShellCheck on GitHub](https://github.com/koalaman/shellcheck) -- static analysis for shell scripts
- [@anthropic-ai/sandbox-runtime on npm](https://www.npmjs.com/package/@anthropic-ai/sandbox-runtime) -- OS-level sandboxing
- [Anthropic sandbox-runtime on GitHub](https://github.com/anthropic-experimental/sandbox-runtime)
- [Anthropic engineering blog: Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Claude Code security docs](https://code.claude.com/docs/en/security)
- [CVE-2025-59536: Claude Code hooks RCE](https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/)
- [Node.js v25 permissions documentation](https://nodejs.org/docs/latest/api/permissions.html)
- [Node.js permission model changes 2026](https://dev.to/1xapi/5-nodejs-permission-model-changes-every-api-developer-should-know-in-2026-3hh8)
- [node-safe on GitHub](https://github.com/berstend/node-safe) -- Deno-like permissions for Node.js (19 downloads/week, macOS-only)
- [Bun sandboxing permissions issue #6617](https://github.com/oven-sh/bun/issues/6617)
- [Bun as AI agent sandbox issue #25929](https://github.com/oven-sh/bun/issues/25929)
- [VS Code extension runtime security](https://code.visualstudio.com/docs/configure/extensions/extension-runtime-security)
- [VS Code extensions: 128M installs with critical flaws (Feb 2026)](https://thehackernews.com/2026/02/critical-flaws-found-in-four-vs-code.html)
- [Homebrew formula cookbook](https://docs.brew.sh/Formula-Cookbook)
- [Homebrew security and contribution model](https://workbrew.com/blog/security-and-the-homebrew-contribution-model)
- [GitHub Actions CodeQL workflow security (GA April 2025)](https://github.blog/changelog/2025-04-22-github-actions-workflow-security-analysis-with-codeql-is-now-generally-available/)
- [GitHub Actions security: untrusted input](https://securitylab.github.com/resources/github-actions-untrusted-input/)
- [@lavamoat/allow-scripts on npm](https://www.npmjs.com/package/@lavamoat/allow-scripts) -- lifecycle script allowlist
- [LavaMoat allow-scripts guide](https://lavamoat.github.io/guides/allow-scripts/)
- [Deno security and permissions](https://docs.deno.com/runtime/fundamentals/security/)
- [Socket.dev npm supply chain protection](https://www.developer-tech.com/news/socket-security-analysis-on-npm-shield-for-supply-chains/)
- [OpenAI Aardvark: agentic security researcher](https://openai.com/index/introducing-aardvark/)
- [npm supply chain attack September 2025 (CISA)](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem)
- [NodeShield: runtime enforcement for Node.js](https://arxiv.org/html/2508.13750v1)
- [Veracode 2025 GenAI Code Security Report](https://www.sciencedirect.com/science/article/abs/pii/S1566253525010036)
- [v8-sandbox on npm](https://www.npmjs.com/package/v8-sandbox)
- [vm2 sandbox escape (2026)](https://semgrep.dev/blog/2026/calling-back-to-vm2-and-escaping-sandbox/)

### Data Transparency

- **Found**: Specific npm packages with download counts, maintenance status, and Bun compatibility indicators. Ecosystem security models for VS Code, Homebrew, GitHub Actions, Claude Code, and LavaMoat. Runtime permission model status for Node.js, Deno, and Bun. CVE data for Claude Code hooks vulnerability.
- **Not Found**: No WASM version of ShellCheck. No npm package for shell command malicious intent detection. No Bun permission model timeline. No independent benchmarks for `@anthropic-ai/sandbox-runtime` performance overhead. No data on false positive rates for pattern-based shell command analysis.

## Observations

- [fact] No npm package exists for shell command malicious intent detection; custom analyzer required #gap
- [fact] shell-quote (15.1M weekly downloads) provides tokenization foundation already in dependency stack per ADR-006 #existing-dependency
- [fact] @anthropic-ai/sandbox-runtime provides OS-level sandboxing via npm, used by Claude Code in production #tool-option
- [technique] Three-layer security model: static analysis, informed consent, runtime sandboxing matches Claude Code pattern #architecture
- [insight] VS Code's trust-at-install model has failed repeatedly (128M installs affected Feb 2026); pre-install analysis is necessary #precedent
- [risk] Bun has no permission model (issue #6617 open, no timeline), making runtime sandboxing the only execution-time control #bun-gap
- [insight] LLM analysis should supplement but never replace deterministic pattern matching due to hallucination and adversarial evasion risks #ai-limitation
- [fact] Claude Code hooks vulnerability (CVE-2025-59536) demonstrates that auto-executing hooks without confirmation is a real attack vector #precedent
- [decision] Risk taxonomy should cover 10 categories: network, exfiltration, credentials, reverse shells, crypto miners, privilege escalation, secondary payloads, env exfiltration, process manipulation, obfuscation #taxonomy
- [requirement] Hook analysis must work without network connectivity (no LLM dependency for core security gate) #offline-capable

## Relations

- part_of [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- relates_to [[ADR-004 Plugin Security Model]]
- relates_to [[ANALYSIS-049 ADR-004 Security Model Scope and Forward References]]
- relates_to [[ADR-006-core-dependency-stack]]
- relates_to [[ANALYSIS-013-input-sanitization-patterns]]