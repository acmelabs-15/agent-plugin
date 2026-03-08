---
title: ADR-008 Source Resolution and Package Validation
type: decision
permalink: decisions/adr-008-source-resolution-and-package-validation-2
tags:
- decision
- source-resolution
- manifest-discovery
- semver
- archive
- bun-api
- package-validation
---

# ADR-008 Source Resolution and Package Validation

## Status

**Accepted**

**Date**: 2026-03-07
**Authors**: Agent Plugin Core Team
**Consulted**: Analyst team (ANALYSIS-025 source resolution patterns, ANALYSIS-028 Bun API reliability)
**Informed**: All project contributors

## Context and Problem Statement

`@acmelabs-15/agent-plugin` resolves plugin packages from three source types (npm registry, GitHub repositories, local filesystem) and must extract, validate, and stage them before installation. Three implementation questions need explicit architectural decisions:

1. How does the tool discover the plugin manifest inside downloaded packages?
2. How does the tool interpret bare `owner/repo` specifiers that could be local paths or GitHub repos?
3. Which Bun built-in APIs are production-ready for version comparison, archive extraction, and file downloads?

ADR-007 Decision 8 defines 4 source format types with validation rules. ADR-001 mandates `plugin.json` at the package root. ADR-005 establishes Bun 1.3.x as the runtime. ANALYSIS-025 researched source resolution patterns across npm, GitHub, Go, pip, Cargo, and Homebrew. ANALYSIS-028 assessed Bun.semver, Bun.Archive, and Bun.write() reliability.

## Decision Drivers

- Single source of truth for plugin metadata (ADR-001 established plugin.json as the manifest format)
- Zero ambiguity in source type detection (ADR-007 defines detection order)
- Minimal third-party dependencies (ADR-006 removed 6 unnecessary dependencies from the spec)
- Bun runtime alignment (ADR-005 selected Bun 1.3.x as sole runtime)
- Production reliability over API novelty (prefer battle-tested patterns over bleeding-edge built-ins)

## Considered Options

### Manifest Discovery

- Option A: plugin.json only, no fallback (chosen)
- Option B: plugin.json with package.json field fallback
- Option C: Auto-discovery scanning multiple file patterns

### Bare owner/repo Detection

- Option A: Filesystem-first check, GitHub fallback (chosen)
- Option B: GitHub-first, treat as local path only if GitHub 404s
- Option C: Require explicit `github:` prefix for all GitHub sources (no bare support)

### Bun API Selection

- Option A: Pragmatic mix based on API maturity (chosen)
- Option B: All Bun built-ins (Bun.semver + Bun.Archive + Bun.write direct)
- Option C: All third-party packages (node-semver + tar + node-fetch)

## Decision Outcome

Three decisions govern source resolution and package validation. Each addresses a distinct concern in the add/upgrade pipeline.

### Decision 1: Manifest Discovery -- plugin.json Only

Plugin packages MUST contain a `plugin.json` at the package root. No fallback to `package.json` fields, no auto-discovery of alternative manifest locations.

**Discovery algorithm after archive extraction:**

1. List immediate children of the extraction directory
2. If exactly 1 subdirectory and 0 files at root: enter that subdirectory (strip the archive wrapper -- `package/` for npm, `{repo}-{ref}/` for GitHub)
3. Look for `plugin.json` in the current directory
4. If not found: error with message "No plugin.json found. This package is not an agent-plugin plugin."

**In-memory pre-check (optimization):** Before extracting to disk, use the `tar` npm package's `list` command to search for `plugin.json` in the archive entry list. If absent, fail fast without writing to the filesystem. (Future consideration: when Bun.Archive matures past 1+ year and adds strip-components support per IMP-004, `Bun.Archive.files()` could replace tar's list command for this pre-check.)

| Source Type | Archive Wrapper Directory | plugin.json Location |
|---|---|---|
| npm tarball | `package/` | `package/plugin.json` |
| GitHub archive | `{repo}-{ref}/` | `{repo}-{ref}/plugin.json` |
| Local path | N/A (direct filesystem) | `{path}/plugin.json` |

**Why no package.json fallback:** A fallback creates ambiguity about which file is authoritative. Plugin authors would not know whether their `package.json` fields or `plugin.json` fields take precedence. A single mandatory manifest file eliminates this class of bugs entirely. ADR-001 already established plugin.json as the manifest format. This decision enforces that contract at the resolution layer.

### Decision 2: Smart Detection for Bare owner/repo Specifiers

When a user provides a bare `owner/repo` string (no `github:` prefix, no `https://` prefix, no `@` prefix), agent-plugin checks the local filesystem first, then falls back to the GitHub API.

**Detection order:**

1. Check if the string resolves to an existing local path (`fs.existsSync` or `Bun.file().exists()`)
2. If local path exists: treat as local source
3. If local path does not exist: treat as GitHub shorthand (`github:{owner}/{repo}`)

**Why filesystem-first:** A string like `plugins/my-plugin` is a valid local path AND could be misinterpreted as a GitHub `owner/repo`. Checking the filesystem first prevents misinterpreting local paths as GitHub repositories. Note: HashiCorp's go-getter uses a detector chain pattern but places FileDetector last, not first. agent-plugin inverts this order intentionally because local development paths are the more common case for a plugin CLI tool, and misinterpreting a local path as a remote repo is the more dangerous failure mode.

**Why not require explicit `github:` prefix:** ANALYSIS-025 Section 6 identified this ambiguity. Option 1 (require `github:` prefix) is safer but creates friction. Users expect `owner/repo` to resolve to GitHub because npm, Go, and Homebrew all support this convention. The filesystem-first heuristic resolves the ambiguity without sacrificing usability.

**Edge case:** If a user has a local directory named `facebook/react`, the tool treats it as a local path. The user can force GitHub resolution with the explicit `github:facebook/react` prefix. This is the correct behavior: local paths take priority over remote assumptions.

**Interaction with ADR-007 detection order:** ADR-007 defines the source type detection chain as: starts with `@` = npm; starts with `https://` = git HTTPS; starts with `github:`/`gitlab:`/`bitbucket:` = git shorthand; otherwise local path. This decision refines the "otherwise local path" branch to add a GitHub fallback when the local path does not exist AND the string matches the `owner/repo` pattern (exactly one `/`, no `.` or path separators beyond the single `/`).

### Decision 3: Bun Built-in API Selection

Use a pragmatic mix of Bun built-ins and third-party packages based on each API's maturity and fitness for the task.

**Bun.semver: USE (production-ready)**

- Introduced in Bun v1.0.11 (November 2023). Over 2 years old.
- Passes node-semver's test suite. Two methods: `satisfies(version, range)` and `order(a, b)`.
- Covers the exact use cases agent-plugin needs: range matching for npm version resolution, version sorting for upgrade detection.
- 20x faster than node-semver (vendor claim). Performance is irrelevant at CLI scale but confirms optimization investment.
- Missing methods (valid, parse, coerce, inc) are either unnecessary or have trivial workarounds (`satisfies(v, '*')` for validation).

**tar npm package: USE instead of Bun.Archive (too young)**

- Bun.Archive was introduced in Bun v1.3.6 (January 13, 2026). Only 2 months old at time of decision.
- Bun.Archive has no `strip-components` option. Both npm tarballs (`package/` prefix) and GitHub archives (`{repo}-{ref}/` prefix) require prefix stripping. The workaround using `.files()` + manual path rewriting adds 10-20 lines of code.
- The `tar` npm package (node-tar) has 30M+ weekly downloads, is 10+ years old, and provides `strip: 1` for prefix removal out of the box.
- Trade-off: adding `tar` as a 14th dependency increases the dependency count by 1. This is acceptable given the maturity gap.
- Monitor Bun.Archive development. When `strip-components` support ships and the API reaches 1+ year maturity, re-evaluate switching to the built-in.

**Bun.write() with arrayBuffer() workaround: USE**

- `Bun.write(path, response)` has a confirmed hanging bug on large files (GitHub issues #21455, #13237). The bug is open with PR #27217 in progress.
- Workaround: use `Bun.write(path, await response.arrayBuffer())` instead of passing the Response directly. This buffers the response body first, then writes. Reliable and tested.
- Plugin packages are typically under 10 MB. Buffering the full response body is acceptable at this scale.
- When the Bun.write fix ships (PR #27217), the workaround can be removed. No architectural change required.

```typescript
// CORRECT: use arrayBuffer() workaround
const response = await fetch(tarballUrl);
await Bun.write(outputPath, await response.arrayBuffer());

// INCORRECT: hangs on large files (GitHub #13237)
// await Bun.write(outputPath, response);
```

## Consequences

### Positive

- **POS-001**: plugin.json-only discovery eliminates manifest ambiguity. Plugin authors have one file to maintain. Tooling has one file to parse. Zero interpretation conflicts.
- **POS-002**: Filesystem-first detection for bare `owner/repo` prevents misinterpreting local development paths as GitHub repositories, the more dangerous failure mode.
- **POS-003**: Bun.semver eliminates the node-semver dependency (300M+ weekly downloads, 43KB) with a built-in that passes the same test suite.
- **POS-004**: The `tar` package provides battle-tested archive extraction with strip-components support, avoiding 10-20 lines of manual prefix-stripping workaround code.
- **POS-005**: The Bun.write arrayBuffer() workaround is a 1-line change that avoids a confirmed hanging bug. No architectural complexity added.

### Negative

- **NEG-001**: plugin.json-only means npm packages that are also agent-plugin plugins cannot use `package.json` as their manifest. Authors must maintain both files. Mitigated by the scaffolding wizard (`agent-plugin init`) which generates `plugin.json` alongside `package.json`.
- **NEG-002**: Filesystem-first detection adds a filesystem stat call for every bare `owner/repo` input. Performance impact is negligible (sub-millisecond) but the code path is more complex than a pure string-matching detector.
- **NEG-003**: Adding the `tar` npm package increases the dependency count from 13 to 14. Adds supply chain surface. Mitigated by tar's 10+ year track record and 30M+ weekly downloads.
- **NEG-004**: The arrayBuffer() workaround buffers the entire response body in memory before writing. For typical plugin packages (under 10 MB), this is acceptable. Packages over 100 MB would consume significant memory. Mitigated by the fact that plugin packages are small by nature (mostly markdown, JSON, and small scripts).

## Alternatives Considered

### Manifest Discovery: package.json Field Fallback

- **Description**: If `plugin.json` is absent, check `package.json` for an `agentPlugin` field containing manifest data.
- **Good**: Reduces friction for npm package authors who already have `package.json`.
- **Bad**: Creates two sources of truth. Which file wins when both exist? Every consumer of manifest data must handle both locations.
- **Bad**: Violates ADR-001 which established plugin.json as the single manifest format.
- **Rejection Reason**: Ambiguity in multi-source resolution is a class of bug, not a feature. One file, one truth.

### Manifest Discovery: Auto-Discovery Scanning

- **Description**: Scan the package root for any of: `plugin.json`, `plugin.yaml`, `plugin.toml`, `.agent-plugin/manifest.json`.
- **Good**: Flexible for plugin authors with different preferences.
- **Bad**: Every additional format requires a parser. Increases test surface. Creates "which format should I use?" decision fatigue.
- **Rejection Reason**: ADR-001 already decided JSON format. Scanning for alternatives contradicts that decision.

### Bare Specifier: GitHub-First Detection

- **Description**: When receiving `owner/repo`, query the GitHub API first. Only treat as local path if GitHub returns 404.
- **Good**: Matches user intent in most cases (users type `owner/repo` meaning GitHub).
- **Bad**: Makes a network request for every bare specifier, including local paths. Adds latency. Fails offline.
- **Bad**: A typo in a local path triggers a GitHub API call instead of an immediate filesystem error.
- **Rejection Reason**: Network-first detection penalizes the common case (local development) to optimize the less common case (GitHub shorthand without prefix).

### Bare Specifier: No Bare Support (Require github: Prefix)

- **Description**: Bare `owner/repo` is always a local path. GitHub requires explicit `github:owner/repo`.
- **Good**: Zero ambiguity. Pure string matching. No filesystem check needed.
- **Bad**: Breaks user expectations from npm, Go, and Homebrew where bare `owner/repo` resolves to GitHub.
- **Rejection Reason**: ANALYSIS-025 found that all major package managers support bare GitHub shorthand. Requiring the prefix creates unnecessary friction.

### API Selection: All Bun Built-ins

- **Description**: Use Bun.semver + Bun.Archive + Bun.write(path, response) with no third-party packages.
- **Good**: Zero additional dependencies. Smallest possible supply chain.
- **Bad**: Bun.Archive is 2 months old with missing strip-components. Requires manual workaround code.
- **Bad**: Bun.write has a confirmed hanging bug requiring workaround anyway.
- **Rejection Reason**: API maturity matters more than dependency count. A 2-month-old extraction API handling untrusted archive content is a reliability risk.

### API Selection: All Third-Party Packages

- **Description**: Use node-semver + tar + got/node-fetch for all operations. No Bun-specific APIs.
- **Good**: Maximum maturity. Every library is 5+ years old.
- **Bad**: Adds 3 dependencies (node-semver, tar, HTTP client) when Bun provides viable built-ins for 2 of 3.
- **Bad**: Ignores ADR-005's rationale for selecting Bun (built-in APIs reduce dependency count).
- **Rejection Reason**: Bun.semver is 2+ years old and passes node-semver's test suite. Using node-semver alongside Bun.semver is redundant. The pragmatic approach uses built-ins where mature and third-party where immature.

## Confirmation

Implementation compliance will be confirmed via:

- [ ] `add` command rejects packages without `plugin.json` with clear error message
- [ ] `add` command never falls back to reading `package.json` for manifest data
- [ ] Bare `owner/repo` input resolves to local path when the path exists on disk
- [ ] Bare `owner/repo` input resolves to GitHub when the path does not exist and matches the pattern
- [ ] Explicit `github:owner/repo` always resolves to GitHub regardless of local filesystem
- [ ] Bun.semver.satisfies() and Bun.semver.order() handle all npm version range formats in the test suite
- [ ] tar package extracts npm tarballs with `strip: 1` to remove `package/` prefix
- [ ] tar package extracts GitHub archives with `strip: 1` to remove `{repo}-{ref}/` prefix
- [ ] File downloads use `Bun.write(path, await response.arrayBuffer())` pattern, never `Bun.write(path, response)` directly
- [ ] Integration test: `add @scope/package` from npm registry discovers and validates plugin.json
- [ ] Integration test: `add owner/repo` from GitHub downloads archive and discovers plugin.json
- [ ] Archive extraction rejects entries containing `../` path segments, absolute paths, or symlinks outside target directory
- [ ] npm tarball SHA-512 checksum is verified against registry `dist.integrity` field before extraction
- [ ] GitHub download checksum is verified against release asset SHA-256 when available

## Reversibility Assessment

- [x] **Rollback capability**: Each decision is independently reversible. Adding package.json fallback, changing detection order, or swapping tar for Bun.Archive requires no data migration.
- [x] **Vendor lock-in**: Bun.semver has a direct replacement (node-semver). tar is a standard npm package. No proprietary APIs introduced.
- [x] **Exit strategy**: If Bun.semver develops edge-case bugs, drop in node-semver as replacement. If tar adds unwanted complexity, switch to Bun.Archive when it matures. Both swaps are under 1 day of effort.
- [x] **Legacy impact**: No existing codebase. Greenfield decisions.
- [x] **Data migration**: No persistent data formats affected. Source resolution is stateless per invocation.

## Implementation Notes

- **IMP-001**: The manifest discovery algorithm should be a standalone function (`discoverManifest(extractionDir): PluginManifest`) reusable across npm, GitHub, and local source resolvers.
- **IMP-002**: The bare `owner/repo` detection heuristic (exactly one `/`, no `.` or additional path separators) should be implemented as a regex: `/^[a-zA-Z0-9_-]+\/[a-zA-Z0-9_.-]+$/`.
- **IMP-003**: Add `tar` to the dependency list in package.json. This brings the total from 13 (ADR-006) to 14 dependencies.
- **IMP-004**: When Bun.Archive adds strip-components support and reaches 1+ year maturity, create a follow-up ADR to re-evaluate replacing tar with the built-in.
- **IMP-005**: The arrayBuffer() workaround should be encapsulated in a `downloadFile(url, outputPath)` utility function so the workaround is applied consistently and can be removed in one place when the Bun.write fix ships.
- **IMP-006**: In-memory manifest pre-check via archive entry inspection should log a debug message when skipping extraction of an invalid package, aiding troubleshooting.
- **IMP-007**: Archive extraction MUST prevent zip slip / path traversal attacks (CWE-22, CVSS 8.6). The `tar` npm package has had CVE-2021-32803 and CVE-2021-32804. Implementation MUST: (1) reject archive entries containing `../` path segments, (2) reject entries with absolute paths, (3) reject symlinks pointing outside the target extraction directory. Use tar's `filter` option to inspect and reject malicious entries before extraction. Use tar's `strip` option (already required for prefix removal) which also mitigates some traversal vectors.
- **IMP-008**: Package integrity verification MUST be performed before extraction (CWE-494, CVSS 7.5). For npm tarballs: verify the downloaded tarball's SHA-512 checksum against the `dist.integrity` field from the npm registry metadata response. For GitHub downloads: verify against the release asset's SHA-256 checksum if the release provides one. Fail with a clear error message if the checksum does not match. Use Node.js `crypto.createHash()` (available in Bun) to compute checksums.

## References

- **REF-001**: ANALYSIS-025 Source Resolution Patterns (npm registry API, GitHub archive URLs, manifest discovery, semver comparison, content staging)
- **REF-002**: ANALYSIS-028 Bun Built-in API Reliability Assessment (Bun.semver maturity, Bun.Archive gaps, Bun.write hanging bug)
- **REF-003**: ADR-001 Plugin Format and Manifest (plugin.json as mandatory manifest format)
- **REF-004**: ADR-005 Runtime and Distribution Strategy (Bun 1.3.x as sole runtime)
- **REF-005**: ADR-006 Core Dependency Stack (13 dependencies, dependency governance policy)
- **REF-006**: ADR-007 CLI Architecture and Interaction Model (source type detection order, validation rules)
- **REF-007**: HashiCorp go-getter detector chain pattern (detector chain precedent; note: go-getter places FileDetector last, agent-plugin intentionally inverts this order)

## Observations

- [decision] plugin.json is the only manifest discovery target; no fallback to package.json fields or alternative file patterns #manifest #single-source-of-truth
- [decision] Bare owner/repo specifiers use filesystem-first detection with GitHub API fallback, intentionally inverting go-getter's detector chain order #source-detection #disambiguation
- [decision] Bun.semver adopted for version comparison: 2+ years old, passes node-semver test suite, satisfies() and order() cover all plugin manager use cases #semver #bun-builtin
- [decision] tar npm package chosen over Bun.Archive for extraction: Bun.Archive is 2 months old and lacks strip-components support #archive #tar #maturity
- [decision] Bun.write() used with arrayBuffer() workaround to avoid confirmed hanging bug on direct Response writes (GitHub #13237) #download #workaround
- [fact] npm tarballs nest content under package/ prefix; GitHub archives nest under {repo}-{ref}/ prefix; both require stripping #archive #extraction
- [fact] Bun.Archive introduced January 2026 (v1.3.6), has no strip-components option (GitHub #26664) #bun-archive #limitation
- [fact] Bun.semver provides exactly 2 methods (satisfies, order) vs node-semver's 10+ methods; the 2 methods cover plugin manager needs #semver #api-surface
- [insight] Filesystem-first detection prevents misinterpreting local development paths as GitHub repos, the more dangerous failure mode #source-detection
- [insight] API maturity (years in production) is the selection criterion, not dependency count minimization; 2-month-old APIs handling untrusted content are a reliability risk #api-selection
- [constraint] Adding tar increases dependency count from 13 to 14; acceptable given 10+ year maturity and 30M+ weekly downloads #dependency
- [risk] Archive extraction must prevent zip slip / path traversal (CWE-22, CVSS 8.6); tar has had CVE-2021-32803 and CVE-2021-32804 #security #archive
- [risk] Package integrity must be verified before extraction (CWE-494, CVSS 7.5); npm tarballs verified via dist.integrity SHA-512, GitHub via release asset SHA-256 #security #integrity

## Relations

- depends_on [[ADR-001 Plugin Format and Manifest]]
- depends_on [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ADR-006 Core Dependency Stack]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ANALYSIS-025-source-resolution-patterns]]
- relates_to [[ANALYSIS-028-bun-builtin-api-reliability]]

## Post-Review Updates (2026-03-07)

- [outcome] P0/P1 editorial fixes applied from DEBATE-ADR-008 review (3 Accept, 3 D&C consensus) #adr-review
- [fact] IMP-007 added: zip slip / path traversal prevention (CWE-22, CVSS 8.6) using tar filter option #security
- [fact] IMP-008 added: package integrity verification (CWE-494, CVSS 7.5) via SHA-512 for npm, SHA-256 for GitHub #security
- [fact] go-getter citation corrected: FileDetector runs last in go-getter, agent-plugin intentionally inverts order #accuracy
- [fact] Bun.Archive.files() pre-check replaced with tar list command; Bun.Archive noted as future consideration only #consistency
- [fact] Memory buffering argument dropped from Bun.Archive rejection; 2-month maturity is the primary justification #consistency
