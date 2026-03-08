---
title: ANALYSIS-013-input-sanitization-patterns
type: note
permalink: analysis/analysis-013-input-sanitization-patterns
tags:
- sanitization
- validation
- security
- injection
- analysis
- agent-plugin
---

# ANALYSIS-013 Input Sanitization Patterns

## 1. Objective and Scope

**Objective**: What packages, patterns, and architectural approaches should @acmelabs-15/agent-plugin use to validate and sanitize plugin-authored content before injecting it into platform configuration files (JSON, YAML frontmatter, markdown, shell command strings)?

**Scope**: Covers (1) Node.js/TypeScript validation and sanitization package landscape, (2) injection vector analysis per target format, (3) community best practices from VSCode, npm, GitHub Actions, WordPress, ESLint, and OWASP, (4) schema validation vs sanitization design decision, (5) recommended approach with package selections.

**Excluded**: Runtime AI prompt injection detection (out of scope for static sanitization layer). Browser-side sanitization (CLI tool only).

## 2. Context

ANALYSIS-009 established that instruction file content is a proven attack vector. Three CVEs in 2025 demonstrate real-world exploits through configuration/instruction file injection (CVE-2025-54135, CVE-2025-59944, CVE-2025-59536). Research paper arxiv:2509.22040v1 showed 41-84% prompt injection success rates through coding rule files. OWASP LLM Top 10 2025 ranks prompt injection as the #1 vulnerability, found in 73% of production AI deployments.

Our tool writes author-supplied content from plugin.json manifests into files that directly control AI agent behavior across 7 platforms. This creates 5 distinct injection surfaces: JSON config files, YAML frontmatter, markdown instruction content, shell command strings in hook arrays, and security-critical fields that control AI agent permissions.

Prior decisions:

- ADR-001: Plugin manifest (plugin.json) defines name, version, description, component paths
- ADR-003: Cross-platform frontmatter with platformConfig overrides; platform adapters emit supported fields
- ANALYSIS-009 RQ4: Identified sanitization layers needed (length limits, pattern detection, content fencing, schema validation)

## 3. Approach

**Methodology**: Web research across 20+ queries covering npm package ecosystems, OWASP guidelines, platform security models (VSCode, npm, GitHub Actions, WordPress, ESLint), injection vector research, and library CVE histories. Cross-referenced with existing project analyses.

**Tools Used**: WebSearch (20 queries), WebFetch (1 page), Brain MCP search, file reads of existing project analyses.

**Limitations**: Bundle size numbers are from web sources and may shift with new releases. Weekly download counts fluctuate. Some npm pages returned errors during fetch. Shescape exact download count was not available from search results.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| Zod: 65M weekly downloads, 41K GitHub stars, TypeScript-first, v4 released 2025 with 14x faster string parsing | npm trends, InfoQ | High |
| AJV: 177M weekly downloads, 14.5K stars, JSON Schema draft-06/07 compliance, CVE-2020-15366 prototype pollution (patched in 6.12.3) | npm trends, Snyk | High |
| Joi: 14M weekly downloads, 21K stars, mature but no native TypeScript type inference | npm trends | High |
| Valibot: 4.7M weekly downloads, 8.3K stars, 90% smaller than Zod, CVE-2025-66020 ReDoS in emoji regex (patched in 1.2.0) | npm trends, GitLab Advisory | High |
| validator (npm): 15.5M weekly downloads, 23.6K stars, string-only validators (isEmail, isURL, isAlphanumeric) and sanitizers (escape, trim, stripLow) | npm | High |
| DOMPurify: 28M weekly downloads, 16.7K stars, DOM-based XSS sanitizer, requires jsdom for Node.js server-side use | npm, GitHub | High |
| sanitize-html: 7.7M weekly downloads, 4.5K stars, Node.js native HTML sanitizer with allowlist config | npm | High |
| shell-quote: 22.7M weekly downloads, known CVE for incomplete escaping of redirect operators (patched in 1.6.1) | npm, Medium | High |
| shescape: smaller adoption (~20 dependents), cross-platform shell escaping (bash/powershell/cmd), CVE for symlink chain issue (patched in 2.1.9) | npm, GitHub | Medium |
| js-yaml v4: load() is safe by default (safeLoad removed), but no built-in maxAliasCount limit for billion laughs prevention | npm, GitHub issues | High |
| OWASP: Allowlist validation preferred over denylist; constrain-reject-sanitize layered approach recommended | OWASP Cheat Sheet | High |
| VSCode: Extension manifest validated via semver, trusted badge allowlist, workspace trust model for untrusted content | VSCode docs, DeepWiki | High |
| npm: Package.json validated via validate-npm-package-name, validate-npm-package-license, type checks per field | npm docs | High |
| GitHub Actions: Untrusted input must go through intermediate env vars; inline shell interpolation is the primary injection vector | GitHub Security Lab | High |
| ESLint: Plugin rule options validated against JSON Schema (meta.schema); unrecognized options rejected by default | ESLint docs | High |
| WordPress: Layered sanitization (sanitize_text_field strips HTML, wp_kses allows specific tags via allowlist, separate escaping on output) | WordPress docs | High |
| Zod strips unrecognized keys by default during parsing; z.record() with **proto** keys can cause prototype pollution | GitHub issue #2227 | High |
| Zod v4 adds global schema registry, JSON Schema import, unified error API | Zod docs, InfoQ | High |
| Prompt injection via AGENTS.MD hijacked VSCode Chat agent behavior | Prompt Security blog | High |

### Facts (Verified)

- [fact] Zod v4 (released 2025) is the most popular TypeScript-first schema validator at 65M+ weekly downloads, with 14x faster string parsing than v3 and native JSON Schema import support
- [fact] AJV has the highest raw download count (177M/week) but does not natively infer TypeScript types and had a prototype pollution CVE (patched)
- [fact] OWASP explicitly recommends allowlist validation over denylist, and schema-based validation (JSON Schema/XSD) for structured data input
- [fact] js-yaml v4 made load() safe by default (removed safeLoad), preventing arbitrary code execution from YAML, but does not prevent billion laughs (alias expansion) by default
- [fact] shell-quote had incomplete escaping of redirect operators (>, <) and semicolons until version 1.6.1; shescape provides cross-platform shell escaping but has smaller adoption
- [fact] Zod strips unrecognized object keys by default (z.object().parse()), providing built-in protection against unexpected fields in plugin manifests
- [fact] WordPress uses a three-layer approach: validate (check type/format), sanitize (strip/encode dangerous content), escape (context-specific output encoding)
- [fact] ESLint validates plugin configuration options against JSON Schema defined in meta.schema, rejecting any options not matching the schema
- [fact] GitHub Actions documents that untrusted input (issue titles, PR branch names, comment bodies) must never be interpolated into shell commands directly
- [fact] DOMPurify requires a DOM (jsdom on Node.js), making it heavier for CLI use; sanitize-html works natively in Node.js without DOM dependency

### Hypotheses (Unverified)

- [hypothesis] Zod's .strip() behavior (removing unrecognized keys) combined with explicit **proto**/constructor/prototype key rejection may provide sufficient prototype pollution protection without a separate library
- [hypothesis] For our use case (CLI tool, no browser rendering), HTML/XSS sanitization libraries (DOMPurify, sanitize-html) add unnecessary weight since our injection target is AI agents consuming markdown, not browsers rendering HTML
- [hypothesis] A custom prompt injection pattern detector (regex-based) may be more effective than generic XSS sanitizers for our specific threat model (AI instruction hijacking vs browser XSS)

## 5. Results

### Package Comparison Table

| Package | Weekly Downloads | GitHub Stars | TypeScript | Bundle Size (min+gz) | Last CVE | Our Use Case Fit |
|---|---|---|---|---|---|---|
| **zod** (v4) | 65M | 41K | Native (first-class) | ~6.9KB (mini) / ~15KB (full) | CVE-2023-4316 ReDoS (patched 3.22.3) | PRIMARY: Schema validation for plugin.json, frontmatter, platformConfig |
| **ajv** | 177M | 14.5K | Via @types | ~35KB | CVE-2020-15366 prototype pollution (patched 6.12.3) | ALTERNATIVE: If JSON Schema compliance needed for external tooling |
| **joi** | 14M | 21K | Via @types | ~45KB | None known | NOT RECOMMENDED: No TS inference, heavier than Zod |
| **valibot** | 4.7M | 8.3K | Native | ~1.4KB | CVE-2025-66020 ReDoS (patched 1.2.0) | ALTERNATIVE: If bundle size is critical constraint |
| **yup** | ~8M | ~22K | Via @types | ~20KB | None known | NOT RECOMMENDED: Less active, Formik-focused |
| **validator** | 15.5M | 23.6K | Via @types | ~25KB | None known | SUPPLEMENTARY: String-level validators (isURL, isAlphanumeric) and sanitizers (escape, stripLow) |
| **DOMPurify** | 28M | 16.7K | Via @types | ~15KB | None known | NOT NEEDED: Requires jsdom, targets browser XSS not AI prompt injection |
| **sanitize-html** | 7.7M | 4.5K | Via @types | ~40KB | None known | NOT NEEDED: HTML sanitization unnecessary for markdown-to-AI pipeline |
| **xss** | 4.1M | N/A | Via @types | ~20KB | None known | NOT NEEDED: Same rationale as DOMPurify |
| **shell-quote** | 22.7M | N/A | Via @types | ~3KB | CVE in redirect escaping (patched 1.6.1) | SUPPLEMENTARY: Parse/quote shell commands for hook validation |
| **shescape** | Small (~20 deps) | ~200 | Native | ~8KB | CVE symlink issue (patched 2.1.9) | ALTERNATIVE to shell-quote: Better cross-platform escaping |
| **js-yaml** | ~80M | ~6K | Via @types | ~25KB | CVE-2013-4660, GHSA-8j8c (patched; v4 safe by default) | ALREADY NEEDED: YAML parsing for frontmatter; use safe defaults |

### Injection Vector Analysis

#### Vector 1: JSON Injection (settings.json, opencode.json)

**Attack**: Prototype pollution via **proto**, constructor, or prototype keys in plugin config objects.

**Mitigations**:

1. Zod schema validation with .strict() rejects unexpected keys entirely
2. Explicit denial of dangerous keys: z.object().refine(obj => !('**proto**' in obj))
3. Use Object.create(null) for merge targets
4. Freeze merged objects with Object.freeze()

**Confidence**: High. Well-understood attack with well-understood mitigations.

#### Vector 2: YAML Frontmatter Injection

**Attack A**: YAML bombs (billion laughs via anchor/alias expansion) causing DoS.
**Attack B**: Type coercion attacks (yaml values like `!!python/object` or `!!js/function` executing code).
**Attack C**: Multiline string injection breaking out of frontmatter into instruction content.

**Mitigations**:

1. js-yaml v4 load() is safe by default (no code execution via type tags)
2. Set maxAliasCount option to limit alias expansion (prevents billion laughs)
3. Validate parsed frontmatter against Zod schema (reject unexpected fields/types)
4. For frontmatter we GENERATE (not parse untrusted YAML), so type coercion is not a risk on the write path
5. On the read path (reading existing files), always use js-yaml v4+ with default safe mode

**Confidence**: High. js-yaml v4 defaults eliminated the worst YAML attacks.

#### Vector 3: Shell Command Injection (Hook Commands)

**Attack**: Malicious plugin author provides hook commands containing command chaining (;, &&, ||), subshell execution ($(), backticks), or redirect operators (>, <).

**Mitigations**:

1. Allowlist approach: Define permitted command patterns (executable name + arguments only)
2. Use shell-quote to parse hook commands and inspect the AST for operators
3. Reject commands containing: pipes (|), redirects (>, <, >>), subshells ($(), backticks), command chains (;, &&, ||), environment variable references ($VAR), glob patterns (*, ?)
4. Alternatively, run hooks via execFile/spawn (no shell) with explicit argument arrays
5. Display hook commands to user for approval before first execution

**Confidence**: High. GitHub Actions and Node.js security guides converge on the same mitigations.

#### Vector 4: Markdown Instruction Injection (Prompt Injection)

**Attack**: Plugin description or instruction content contains hidden instructions that override user intent when consumed by AI agents.

**Mitigations**:

1. Length limits on all text fields (description: 500 chars, component description: 200 chars)
2. Pattern detection for known prompt injection markers (regex-based):
   - "ignore previous instructions", "you are now", "IMPORTANT:", "CRITICAL:", "OVERRIDE:"
   - System prompt references, role-switching language
   - Shell command syntax (backticks, $(), curl, wget, eval)
   - Base64-encoded content
   - URLs in description fields (flag for review)
3. Content fencing: Wrap author content in explicit data boundaries (blockquotes, clear labeling as "author-provided")
4. Structural separation: Plugin metadata rendered by tool-owned templates, not author-controlled templates
5. No executable content in text fields: Strip backtick code blocks from descriptions

**Confidence**: Medium. Prompt injection is an unsolved problem. Pattern detection catches obvious attacks but sophisticated prompt injection can evade regex-based detection.

#### Vector 5: Path Traversal

**Attack**: Plugin manifest references files outside the plugin directory via ../../ sequences or absolute paths.

**Mitigations**:

1. Resolve all file paths with path.resolve() then verify they start with the plugin's root directory
2. Reject absolute paths in plugin manifests (all paths must be relative)
3. Reject paths containing .. segments
4. Apply URL decoding before path validation (attackers use %2e%2e%2f to bypass naive filters)
5. Validate file extensions against allowlist (only .md, .json, .yaml, .yml, .mdc)

**Confidence**: High. Well-understood attack with deterministic mitigations.

#### Vector 6: Security-Critical Field Manipulation

**Attack**: Plugin manifest sets permissionMode, allowed-tools, or other permission-controlling fields to escalate AI agent capabilities.

**Mitigations**:

1. Security-critical fields must NEVER come from plugin manifests
2. Zod schema for plugin.json must not include permission-related fields; any present are stripped
3. Permission fields are user-controlled only (set via CLI flags or user config, never from plugin metadata)
4. Log warnings when plugin manifests contain unrecognized fields that resemble permission fields

**Confidence**: High. This is a design-level mitigation, not a sanitization problem.

### Community Best Practice Findings

#### VSCode Extension Validation Model

- Strict manifest schema with known fields
- Trusted badge allowlist (only known badge services)
- Workspace trust model separates trusted from untrusted extension capabilities
- Extension packaging tool (vsce) validates during build, not just at runtime
- Semver validation for version fields

#### npm Package.json Validation Model

- Dedicated validators per field type (validate-npm-package-name, validate-npm-package-license)
- Type checking per field (private must be boolean, os must be string array)
- Name validation includes character restrictions and scope format
- No content sanitization (npm trusts package authors; security comes from code review and npm audit)

#### GitHub Actions Input Security Model

- Untrusted input must go through intermediate environment variables
- Never interpolate user-controlled values directly into shell commands
- CodeQL queries detect unsafe interpolation patterns
- Scorecards project audits action workflows for injection risks

#### ESLint Plugin Config Validation Model

- JSON Schema (meta.schema) defines allowed options per rule
- Unrecognized options rejected automatically
- Schema validation happens before any rule execution
- This is the closest analog to our use case: plugin authors define config, framework validates it

#### WordPress Content Sanitization Model

- Three-layer approach: validate (type/format), sanitize (strip/encode), escape (output context)
- sanitize_text_field(): strips HTML tags, extra whitespace, tabs, line breaks
- wp_kses(): allowlist-based HTML filtering (specify exactly which tags/attributes permitted)
- Context-specific escaping on output (esc_html, esc_attr, esc_url)

#### OWASP Input Validation Guidelines

- Allowlist ("what IS authorized") over denylist ("what is NOT authorized")
- Schema-based validation explicitly recommended for JSON input
- Constrain first, reject second, sanitize third (defense in depth)
- Server-side validation mandatory (client-side is UX only)
- Validation as early as possible in the data flow

### Schema Validation vs Sanitization: Design Decision

**Finding**: The community consensus is "validate-and-reject first, sanitize-and-transform second." These are complementary, not competing approaches.

For our use case:

| Layer | Approach | What It Does | Package |
|---|---|---|---|
| Layer 1: Schema Validation | Validate-and-reject | Define exact shape of plugin.json. Reject manifests with wrong types, missing required fields, unexpected fields. | Zod (.strict() mode) |
| Layer 2: Value Constraints | Validate-and-reject | Enforce length limits, character restrictions, format requirements (semver, valid identifiers). | Zod (.max(), .regex(), .refine()) |
| Layer 3: Content Sanitization | Sanitize-and-transform | Strip dangerous patterns from text fields that pass schema validation. Remove shell syntax, HTML tags, prompt injection markers. | Custom transforms in Zod (.transform()) + validator.js string functions |
| Layer 4: Context-Specific Escaping | Escape-on-output | When writing to JSON, YAML, markdown, or shell contexts, apply format-specific escaping. | Platform adapters (custom code per format) |

This mirrors the WordPress model (validate, sanitize, escape) and OWASP's constrain-reject-sanitize pattern.

## 6. Discussion

### Why Zod Over AJV

AJV has 2.7x more downloads, but those downloads come from being a transitive dependency (ESLint, webpack, etc.), not from direct adoption for application validation. For our TypeScript CLI tool, Zod offers:

1. **Type inference**: Parse a plugin.json and get a fully typed PluginManifest object with zero type assertions
2. **Transform pipeline**: Schema definition, validation, and sanitization transforms in a single pipeline
3. **Strict mode**: z.object().strict() rejects unknown keys, providing prototype pollution protection at the schema level
4. **v4 improvements**: 14x faster string parsing, JSON Schema import (if we ever need external schema compatibility), global schema registry for shared schemas across platform adapters
5. **Ecosystem**: 65M weekly downloads, active maintenance, Zod v4 released 2025

AJV would be appropriate if we needed JSON Schema draft compliance for external tooling interoperability. We do not.

### Why Not DOMPurify or sanitize-html

Our injection target is AI agents consuming markdown, not browsers rendering HTML. DOMPurify prevents browser XSS by sanitizing DOM-rendered HTML. Our content never touches a browser DOM. The relevant threat is prompt injection, not XSS.

Additionally, DOMPurify requires jsdom as a dependency for Node.js use, adding ~1.5MB to the dependency tree for a capability we do not need.

sanitize-html is Node.js-native but still targets HTML rendering contexts. Neither library detects prompt injection patterns.

### Custom Prompt Injection Detection

No npm package exists that specifically detects prompt injection in markdown content destined for AI agents. This is an emerging threat category. We need custom detection logic:

1. Regex patterns for known injection markers (ANALYSIS-009 RQ4 enumerated specific patterns)
2. Structural analysis (detecting instruction-like language in description fields)
3. Length limits as a blunt but effective constraint
4. Content fencing (template-controlled output formatting)

This custom detection layer operates inside Zod transform functions, keeping the validation pipeline unified.

### Shell Hook Security Architecture

The safest approach for hook commands is a two-tier system:

**Tier 1: Static validation at install time**. Parse hook commands with shell-quote, inspect the AST, reject commands containing operators (pipes, redirects, chains, subshells). This catches most injection attempts.

**Tier 2: User approval at first execution**. Display the exact commands to the user and require explicit approval. This handles the case where static analysis misses a sophisticated attack.

This mirrors Claude Code's existing hook approval model (CVE-2025-59536 was about bypassing this approval, not about the model itself being flawed).

### Prototype Pollution in JSON Merging

When merging platformConfig overrides from plugins into platform settings, prototype pollution is the primary risk. Mitigations:

1. Zod validates the override object shape before merging (no **proto**, constructor, prototype keys)
2. Merge into Object.create(null) targets
3. Use structuredClone() for deep copying (does not preserve prototype chain)
4. Freeze the result after merging

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Adopt Zod v4 as the primary schema validation library | TypeScript-first, 65M downloads, .strict() mode rejects unknown keys, transform pipeline supports sanitization | Low (1-2 days) |
| P0 | Define strict Zod schemas for plugin.json manifest, component frontmatter, and platformConfig shapes | Schema-first validation is the strongest defense; rejects malformed input before any processing | Medium (3-5 days) |
| P0 | Implement path traversal protection in all file path fields | Resolve paths, verify containment within plugin root, reject absolute paths and .. segments | Low (1 day) |
| P0 | Reject security-critical fields (permissionMode, allowed-tools) in plugin manifests at the schema level | Permission escalation must be impossible via plugin metadata | Low (0.5 days) |
| P1 | Build custom prompt injection pattern detector as Zod transform | No existing package covers AI prompt injection detection; custom regex patterns for known markers | Medium (2-3 days) |
| P1 | Use shell-quote for hook command parsing and validation | Parse commands into AST, reject operators (pipes, redirects, chains, subshells) | Low (1-2 days) |
| P1 | Add validator.js string functions for text field sanitization | escape(), stripLow(), trim() for cleaning text fields that pass schema validation | Low (0.5 days) |
| P1 | Implement length limits on all text fields via Zod .max() | Blunt but effective constraint against injection payloads | Low (0.5 days) |
| P2 | Add content fencing in template output | Wrap author content in blockquotes/labels to semantically separate from tool instructions | Low (1 day) |
| P2 | Implement user approval flow for hook commands | Display commands, require explicit approval before first execution | Medium (2-3 days) |
| P2 | Add prototype pollution guards to JSON merge operations | Object.create(null) targets, structuredClone(), Object.freeze() | Low (1 day) |

### Recommended Package Set

```
Production dependencies:
  zod@^3.24 (or zod/v4 subpath)  -- Schema validation + transform pipeline
  shell-quote@^1.8               -- Shell command parsing for hook validation
  validator@^13                   -- String-level sanitization (escape, stripLow, isURL)

Already needed (not new):
  js-yaml@^4                     -- YAML parsing (safe by default in v4)

NOT needed:
  ajv                            -- No JSON Schema compliance requirement
  joi                            -- No TypeScript inference, heavier
  DOMPurify                      -- Browser XSS, not AI prompt injection
  sanitize-html                  -- HTML context, not markdown-to-AI context
  xss                            -- Same rationale as DOMPurify
  shescape                       -- shell-quote sufficient; shescape adds cross-platform but smaller ecosystem
```

### Validation Pipeline Architecture

```
Plugin Author Input (plugin.json)
        |
        v
[Layer 1: Zod Schema Validation]
  - Strict mode: reject unknown keys
  - Required fields: name, version, description
  - Type enforcement: string, number, array, object per field
  - Reject: __proto__, constructor, prototype keys
        |
        v
[Layer 2: Zod Value Constraints]
  - Length limits: name(64), description(500), component descriptions(200)
  - Format validation: semver for version, identifier regex for name
  - Character restrictions: alphanumeric + hyphens for identifiers
  - URL validation for repository/homepage (validator.isURL)
        |
        v
[Layer 3: Zod Content Transforms]
  - Strip HTML tags from description fields
  - Remove shell syntax (backticks, $(), etc.) from text fields
  - Detect prompt injection patterns (flag or reject)
  - Normalize whitespace (validator.trim, validator.stripLow)
        |
        v
[Layer 4: Path Validation]
  - Resolve all file paths
  - Verify containment within plugin root
  - Reject absolute paths and .. segments
  - Validate file extensions against allowlist
        |
        v
[Layer 5: Hook Command Validation]
  - Parse with shell-quote
  - Inspect AST for operators (reject pipes, redirects, chains, subshells)
  - Validate executable names against allowlist (optional)
  - Queue for user approval on first execution
        |
        v
[Layer 6: Context-Specific Output Escaping]
  - JSON: JSON.stringify() handles escaping
  - YAML frontmatter: js-yaml dump() with safe schema
  - Markdown: Content fencing (blockquotes, labels)
  - Shell: shell-quote.quote() for any runtime command construction
        |
        v
Sanitized, Validated Plugin Data
```

## 8. Conclusion

**Verdict**: Proceed with Zod v4 + shell-quote + validator.js as the sanitization stack.

**Confidence**: High for schema validation and format-specific injection prevention. Medium for prompt injection detection (unsolved problem, regex-based detection is necessary but not sufficient).

**Rationale**: The three-package approach covers all 5 injection surfaces identified in our threat model. Zod provides the primary defense (schema validation with strict mode, transform pipeline for sanitization). shell-quote handles the shell injection surface. validator.js provides battle-tested string sanitization primitives. No HTML sanitization library is needed because our output targets are AI agents, not browsers.

### User Impact

- **What changes for you**: Plugin manifests will be validated against strict schemas. Invalid or suspicious content is rejected with clear error messages. Hook commands require user approval.
- **Effort required**: 3 new production dependencies (zod, shell-quote, validator). Estimated 10-15 days of implementation across all validation layers.
- **Risk if ignored**: Plugin authors can inject arbitrary instructions into AI agent configuration files, enabling data exfiltration, unauthorized command execution, and behavior hijacking. Three CVEs and 41-84% injection success rates in research demonstrate this is not theoretical.

## 9. Appendices

### Sources Consulted

- [npm trends: ajv vs joi vs valibot vs yup vs zod](https://npmtrends.com/ajv-vs-joi-vs-valibot-vs-yup-vs-zod)
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [OWASP Prototype Pollution Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Prototype_Pollution_Prevention_Cheat_Sheet.html)
- [GitHub Security Lab: Keeping GitHub Actions Secure - Untrusted Input](https://securitylab.github.com/resources/github-actions-untrusted-input/)
- [GitHub Blog: Four Tips for Secure GitHub Actions Workflows](https://github.blog/security/supply-chain-security/four-tips-to-keep-your-github-actions-workflows-secure/)
- [VSCode Extension Manifest Documentation](https://code.visualstudio.com/api/references/extension-manifest)
- [WordPress Sanitizing Data - Developer APIs](https://developer.wordpress.org/apis/security/sanitizing/)
- [ESLint Custom Rules - Meta Schema](https://eslint.org/docs/latest/extend/custom-rules)
- [Zod v4 Release Notes](https://zod.dev/v4)
- [Zod v4 InfoQ Coverage](https://www.infoq.com/news/2025/08/zod-v4-available/)
- [CVE-2023-4316: Zod ReDoS in Email Validation](https://security.snyk.io/vuln/SNYK-JS-ZOD-5925617)
- [CVE-2020-15366: AJV Prototype Pollution](https://security.snyk.io/vuln/SNYK-JS-AJV-584908)
- [CVE-2025-66020: Valibot ReDoS in Emoji Regex](https://advisories.gitlab.com/pkg/npm/valibot/CVE-2025-66020/)
- [js-yaml v4: safeLoad Removed, load() Safe by Default](https://github.com/jaredly/hexo-admin/issues/222)
- [shell-quote Injection Vulnerability](https://medium.com/node-security/shell-quote-module-leads-to-potential-command-injection-c1b3d502bb45)
- [shescape: Shell Escape Library](https://github.com/ericcornelissen/shescape)
- [Prompt Security: AGENTS.MD Injection in VSCode](https://prompt.security/blog/when-your-repo-starts-talking-agents-md-and-agent-goal-hijack-in-vs-code-chat)
- [Auth0: Preventing Command Injection in Node.js](https://auth0.com/blog/preventing-command-injection-attacks-in-node-js-apps/)
- [Security Innovation: Constrain, Reject, and Sanitize Input](https://blog.securityinnovation.com/blog/2010/12/constrain-reject-and-sanitize-input.html)
- [Zod GitHub Discussion #2227: z.record Prototype Pollution](https://github.com/colinhacks/zod/issues/2227)
- [Zod GitHub Discussion #1358: Sanitization with Zod](https://github.com/colinhacks/zod/discussions/1358)
- [Node.js Path Traversal Guide](https://www.stackhawk.com/blog/node-js-path-traversal-guide-examples-and-prevention/)
- [WordPress VIP: Validating, Sanitizing, and Escaping](https://docs.wpvip.com/security/validating-sanitizing-and-escaping/)

### Data Transparency

- **Found**: Download counts, GitHub stars, CVE histories, and API documentation for all 12 packages evaluated. Community best practices from 6 platform ecosystems. OWASP guidelines for input validation and prototype pollution prevention.
- **Not Found**: Exact shescape weekly download count (npm page did not expose it in search results). No npm package exists for AI prompt injection detection in markdown content. No published benchmark comparing Zod vs AJV for our specific validation patterns (object shape validation with transforms). Zod v4's exact maxAliasCount behavior for YAML integration was not documented.

## Observations

- [decision] Zod v4 selected as primary validation library over AJV (TypeScript inference, transform pipeline, strict mode) and over Valibot (larger ecosystem, more battle-tested despite larger bundle) #validation #architecture
- [fact] Three production dependencies needed: zod, shell-quote, validator; HTML sanitization libraries (DOMPurify, sanitize-html, xss) are not needed for AI-targeted output #dependencies
- [decision] Layered defense model adopted: constrain (schema), reject (value constraints), sanitize (transforms), escape (output context) following OWASP and WordPress patterns #security #architecture
- [fact] No npm package exists for AI prompt injection detection; custom regex-based detection is necessary but not sufficient for sophisticated attacks #security #gap
- [risk] Prompt injection detection has medium confidence; regex patterns catch obvious attacks but can be evaded by sophisticated adversaries #security #risk
- [technique] Zod .strict() mode combined with explicit **proto**/constructor/prototype key rejection provides prototype pollution protection without a separate library #validation #technique
- [insight] The ESLint model (JSON Schema for plugin config, reject unrecognized options) is the closest community analog to our plugin manifest validation problem #patterns
- [fact] js-yaml v4 made load() safe by default, eliminating the most dangerous YAML attack (code execution via type tags), but billion laughs prevention requires explicit maxAliasCount configuration #yaml #security

## Relations

- extends [[ANALYSIS-009-instruction-file-update-patterns]]
- implements [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[ANALYSIS-012-json-config-merge-patterns]]
- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
