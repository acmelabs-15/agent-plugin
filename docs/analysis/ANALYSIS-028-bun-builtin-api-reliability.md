---
title: ANALYSIS-028-bun-builtin-api-reliability
type: analysis
permalink: analysis/analysis-028-bun-builtin-api-reliability
tags:
- bun
- semver
- archive
- reliability
- production-readiness
- api-evaluation
---

# ANALYSIS-028: Bun Built-in API Reliability Assessment

## 1. Objective and Scope

**Objective**: Determine whether Bun.semver, Bun.Archive, and Bun.write() with fetch Response streaming are production-ready for use in the @acmelabs-15/agent-plugin CLI tool.

**Scope**: API maturity, known bugs, missing features, community sentiment, and alternative recommendations for each API. Excludes Bun runtime stability in general (already established as production-viable).

## 2. Context

The agent-plugin CLI runs on Bun and needs to:

- Compare semantic versions (plugin compatibility checks, update detection)
- Extract npm tarballs and GitHub release archives (plugin installation)
- Download plugin packages from registries/URLs (fetch + write to disk)

Using Bun built-ins instead of third-party packages reduces dependency count, bundle size, and supply chain risk. The trade-off is maturity and feature completeness.

## 3. Approach

**Methodology**: Web research across Bun documentation, GitHub issues, release blogs, community articles, and API references. Cross-referenced official claims with filed bugs and community reports.

**Tools Used**: WebSearch (12 queries), WebFetch (7 pages), Bun official docs, GitHub issue tracker, community blogs.

**Limitations**: Could not access Bun source code or bun-types definitions directly due to GitHub rate limits. Some type definition details inferred from documentation rather than source.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| Bun.semver introduced in v1.0.11 (Nov 8, 2023) | Bun blog | High |
| Bun.semver passes node-semver test suite | Bun v1.0.11 release notes | High |
| Bun.semver provides only 2 methods: satisfies() and order() | Bun docs, API reference | High |
| node-semver provides 10+ methods (valid, parse, coerce, inc, diff, clean, etc.) | npm docs | High |
| Bun.semver is 20x faster than node-semver | Bun docs | Medium (vendor claim) |
| Bun.semver has no valid() method | GitHub issue #7698 (open) | High |
| Bun.semver had pre-release parsing bug (fixed) | Bun v1.1.13 release notes | High |
| Bun.Archive introduced in v1.3.6 (Jan 13, 2026) | Bun blog | High |
| Bun.Archive supports tar and tar.gz only (no zip) | Bun docs, GitHub #27077 | High |
| Bun.Archive has NO strip-components option | Bun docs, API reference | High |
| Bun.Archive requires full archive in memory for instantiation | GitHub #26664 | High |
| Bun.Archive has NO ReadableStream input support | GitHub #26664 (open) | High |
| Bun.Archive has path traversal protection (rejects .., absolute paths) | Bun docs | High |
| Bun.Archive handles symlinks on Linux/macOS, skips on Windows | Bun docs | High |
| Bun.write(path, Response) has known hanging bug with large files | GitHub #21455, #13237 | High |
| Bun.write hanging workaround: use response.body() or response.arrayBuffer() | GitHub #21455 | High |
| PR #27217 in progress to fix Bun.write Response streaming | GitHub #13237 | Medium |

### Facts (Verified)

- [fact] Bun.semver API surface consists of exactly 2 methods: `satisfies(version, range)` and `order(versionA, versionB)` #semver #api-surface
- [fact] Bun.semver passes node-semver's test suite as of v1.0.11 (Nov 2023) #semver #compatibility
- [fact] node-semver provides 10+ methods that Bun.semver lacks: valid, parse, coerce, inc, diff, clean, compare, rcompare, gt, lt, eq, gte, lte, neq #semver #gap
- [fact] Bun.Archive was introduced 2 months ago (Jan 13, 2026 in v1.3.6) #archive #maturity
- [fact] Bun.Archive has no strip-components equivalent for removing path prefixes #archive #limitation
- [fact] Bun.Archive requires entire archive buffered in memory before extraction #archive #memory
- [fact] Bun.write(path, Response) has a confirmed hanging bug on large files, closed as duplicate of #13237 which remains open #streaming #bug

### Hypotheses (Unverified)

- [insight] Bun.semver satisfies() and order() cover 90%+ of CLI plugin manager use cases (version comparison and range checking)
- [insight] Bun.Archive's lack of strip-components can be worked around with glob patterns or post-extraction file moves
- [insight] The Bun.write Response hanging bug may be fixed in a near-future release (PR #27217 exists)

## 5. Results

### Bun.semver

**Maturity**: 2+ years old (since Nov 2023). Stable.

**API Surface**:

| Method | Signature | Purpose |
|---|---|---|
| satisfies | `(version: string, range: string) => boolean` | Check if version matches range |
| order | `(a: string, b: string) => -1 \| 0 \| 1` | Compare two versions for sorting |

**What it supports**:

- Caret ranges (`^1.0.0`)
- Tilde ranges (`~1.0.0`)
- Exact versions (`1.0.0`)
- Wildcards (`1.0.x`, `1.x.x`, `x.x.x`)
- Range syntax (`1.0.0 - 2.0.0`)
- OR unions (`>=1.2.9 <2.0.0 || >=3.0.0`)
- AND intersections (space-separated comparators)
- Pre-release versions (`1.0.0-alpha.1`, `1.0.0-beta.3`)

**What it lacks (compared to node-semver)**:

| Missing Method | Purpose | Impact on agent-plugin |
|---|---|---|
| valid() | Check if string is valid semver | Low (workaround: `satisfies(v, '*')`) |
| parse() | Parse into { major, minor, patch } object | Low (can split string manually) |
| coerce() | Coerce loose strings to semver | None (we control version formats) |
| inc() | Increment version | None (not needed for plugin manager) |
| diff() | Difference between versions | None (not needed) |
| clean() | Trim/clean version strings | Low (can trim manually) |
| gt/lt/eq/gte/lte/neq() | Individual comparison operators | Low (order() covers all cases) |

**Known bugs**: Pre-release parsing bug where `1.0.0.alpha` incorrectly parsed as `1.0.0-alpha` was fixed in v1.1.13.

**Verdict**: [PASS] Safe for production use in agent-plugin.

### Bun.Archive

**Maturity**: 2 months old (since Jan 2026 in v1.3.6). Young.

**API Surface**:

| Method | Purpose |
|---|---|
| constructor(data, options?) | Create or read archive |
| extract(path, options?) | Extract to disk |
| files(glob?) | Get files as Map of File objects (in-memory) |
| blob() / bytes() | Serialize archive |

**What it supports**:

- tar and tar.gz formats
- Glob pattern filtering on extraction
- Negative glob patterns for exclusion
- Path traversal protection (rejects `..`, absolute paths)
- Symlink handling (Linux/macOS: extracted; Windows: skipped)
- Gzip compression levels 1-12

**What it lacks**:

| Missing Feature | Impact on agent-plugin | Severity |
|---|---|---|
| strip-components option | Cannot strip `package/` prefix from npm tarballs | High |
| ReadableStream input | Must buffer entire archive in memory | Medium |
| Streaming extraction | Large archives risk OOM in constrained environments | Medium |
| Zip format support | Cannot handle zip archives | Low (npm uses tar.gz) |

**Known bugs**:

- GitHub #26664 (open): No ReadableStream input support. OOM risk for large archives.
- GitHub #26665 (open): Archive creation buffers everything in memory.
- GitHub #27077 (open): No zip support.

**Critical gap for agent-plugin**: npm tarballs contain files under a `package/` prefix. GitHub release archives contain files under a `{repo}-{ref}/` prefix. Bun.Archive has no strip-components option to remove these prefixes. The only option is glob-based filtering (e.g., `{ glob: "package/**" }`) followed by manual path adjustment, or post-extraction directory rename.

**Verdict**: [WARNING] Usable but requires workarounds. The missing strip-components is a significant friction point. The 2-month age introduces risk of undiscovered bugs.

### Bun.write() with fetch Response Streaming

**Known Issues**:

- GitHub #13237 (open): `Bun.write(path, new Response(req.body))` hangs indefinitely
- GitHub #21455 (closed as duplicate of #13237): Large file downloads via `Bun.write(path, response)` download to memory successfully but writing to disk hangs
- PR #27217 in progress to fix this

**Workarounds**:

```typescript
// Instead of (hangs):
await Bun.write("file.tgz", response);

// Use one of:
await Bun.write("file.tgz", response.body);  // Pass ReadableStream
await Bun.write("file.tgz", await response.arrayBuffer());  // Buffer first
```

**Verdict**: [WARNING] The direct `Bun.write(path, response)` pattern has an open hanging bug. The workaround of passing `response.body` or `response.arrayBuffer()` is reliable. Use the workaround pattern.

## 6. Discussion

### Bun.semver is production-ready

Bun.semver has been stable for 2+ years. It passes node-semver's test suite. The 2-method API (satisfies + order) covers the exact use cases a plugin manager needs: checking version compatibility and sorting versions. The missing methods (valid, parse, inc, diff) are either unnecessary for a plugin manager or have trivial workarounds. The 20x performance claim is a bonus but irrelevant at CLI scale.

### Bun.Archive is usable but immature

At 2 months old, Bun.Archive is the youngest API under evaluation. The core extraction logic works and includes proper security protections. The missing strip-components option is the primary concern because both npm tarballs and GitHub archives use nested prefixes. Two workaround strategies exist:

1. **Glob + manual path handling**: Extract with glob pattern, then programmatically rename/move files
2. **files() + manual write**: Read archive entries via `.files()`, strip prefix from keys, write to disk manually

Both add code complexity. The `.files()` approach loads everything into memory, which is acceptable for typical plugin packages (most are under 10MB) but problematic for large packages.

### Bun.write streaming needs a workaround

The hanging bug with `Bun.write(path, response)` is well-documented and reproducible. The fix (PR #27217) is in progress. The workaround of using `response.arrayBuffer()` or `response.body` is straightforward and reliable. This is a code pattern choice, not a blocker.

### Community sentiment

Bun is broadly considered production-viable for APIs, CLIs, and internal tools as of 2026. Multiple articles from dev.to, InfoQ, and Medium confirm adoption. No "don't use Bun.X" warnings found for semver or Archive specifically. The primary community concern is Node.js compatibility gaps in edge cases, not built-in API reliability.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Use Bun.semver for all version operations | 2+ years stable, passes node-semver tests, API matches our needs exactly | None (direct use) |
| P0 | Use `response.arrayBuffer()` workaround for downloads | Avoids known Bun.write hanging bug | Minimal (pattern choice) |
| P1 | Use Bun.Archive for tar.gz extraction with manual prefix stripping | Works for our use case with a utility function to handle package/ prefix | Low (10-20 lines wrapper) |
| P1 | Add file size guard before `.files()` calls | Prevents OOM on unexpectedly large archives; fall back to `.extract()` + rename for large packages | Low |
| P2 | Monitor GitHub #13237 for Bun.write fix | Remove arrayBuffer workaround when fix ships | None (tracking) |
| P2 | Monitor GitHub #26664 for streaming Archive support | Upgrade to streaming when available | None (tracking) |
| P2 | Keep node-semver as optional fallback | If edge cases surface, swap without architecture change | Low (interface abstraction) |

### Alternative Libraries Comparison

| Library | Purpose | Weekly Downloads | Maturity | Why Not Use |
|---|---|---|---|---|
| node-semver | Semver operations | 300M+ | 12+ years | Unnecessary; Bun.semver covers our needs |
| tar (node-tar) | Tarball extraction | 30M+ | 10+ years | Adds dependency; Bun.Archive works with workaround |
| tar-stream | Streaming tar | 20M+ | 8+ years | Fallback option if Bun.Archive OOM becomes an issue |
| decompress | Multi-format extract | 3M+ | 7+ years | Overkill; we only need tar.gz |

## 8. Conclusion

**Verdict**: Proceed with Bun built-ins, with documented workarounds

**Confidence**: High for Bun.semver, Medium for Bun.Archive, Medium for Bun.write streaming

**Rationale**: Bun.semver is mature and feature-complete for our needs. Bun.Archive is young but functional with a prefix-stripping wrapper. Bun.write streaming has a known bug with a reliable workaround. Using built-ins eliminates 3 dependencies (node-semver, tar, and a download utility) from the supply chain.

### User Impact

- **What changes for you**: Zero third-party dependencies for core plugin operations (version checking, archive extraction, file downloads). Faster cold starts due to fewer imports.
- **Effort required**: Write a 10-20 line utility function to handle archive prefix stripping. Use `response.arrayBuffer()` pattern for downloads.
- **Risk if ignored**: Using Bun.Archive without the prefix-stripping wrapper will extract files into a nested `package/` directory instead of the target location. Using `Bun.write(path, response)` directly risks hanging on large downloads.

## 9. Appendices

### Bun.semver Workaround for Missing valid()

```typescript
// Workaround: Use satisfies with wildcard
function isValidSemver(version: string): boolean {
  return Bun.semver.satisfies(version, "*");
}
```

### Bun.Archive Prefix Stripping Pattern

```typescript
// Extract npm tarball with prefix stripping
async function extractTarball(archiveData: Uint8Array, targetDir: string, prefix = "package/") {
  const archive = new Bun.Archive(archiveData);
  const files = archive.files(`${prefix}**`);
  for (const [path, file] of files) {
    const strippedPath = path.slice(prefix.length);
    const targetPath = `${targetDir}/${strippedPath}`;
    await Bun.write(targetPath, file);
  }
}
```

### Safe Download Pattern

```typescript
// Use arrayBuffer() to avoid Bun.write hanging bug
async function downloadFile(url: string, outputPath: string): Promise<void> {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`Download failed: ${response.status}`);
  await Bun.write(outputPath, await response.arrayBuffer());
}
```

### Sources Consulted

- Bun Semver Documentation: <https://bun.com/docs/runtime/semver>
- Bun Semver API Reference: <https://bun.com/reference/bun/semver>
- Bun Archive Documentation: <https://bun.com/docs/runtime/archive>
- Bun v1.0.11 Release Blog: <https://bun.sh/blog/bun-v1.0.11>
- Bun v1.3.6 Release Blog: <https://bun.com/blog/bun-v1.3.6>
- GitHub Issue #7698 (Bun.semver.valid request): <https://github.com/oven-sh/bun/issues/7698>
- GitHub Issue #26664 (Archive ReadableStream): <https://github.com/oven-sh/bun/issues/26664>
- GitHub Issue #26665 (Archive memory usage): <https://github.com/oven-sh/bun/issues/26665>
- GitHub Issue #27077 (Archive zip support): <https://github.com/oven-sh/bun/issues/27077>
- GitHub Issue #21455 (Bun.write large file hang): <https://github.com/oven-sh/bun/issues/21455>
- GitHub Issue #13237 (Bun.write Response hang): <https://github.com/oven-sh/bun/issues/13237>
- node-semver npm: <https://www.npmjs.com/package/semver>
- Bun Production Readiness 2026: <https://dev.to/last9/is-bun-production-ready-in-2026-a-practical-assessment-181h>

### Data Transparency

- **Found**: Full API surface for Bun.semver and Bun.Archive, known bugs with issue numbers, release dates, community sentiment from multiple sources, workaround patterns
- **Not Found**: Bun.semver source code (to verify test suite pass independently), exact ArchiveExtractOptions TypeScript interface (GitHub rate-limited), performance benchmarks for Bun.Archive vs tar/tar-stream, Bun.write fix timeline

## Observations

- [decision] Use Bun.semver for version operations in agent-plugin #semver #production-ready
- [decision] Use Bun.Archive with prefix-stripping wrapper for tarball extraction #archive #workaround
- [decision] Use response.arrayBuffer() pattern for file downloads to avoid Bun.write hanging bug #download #workaround
- [risk] Bun.Archive is only 2 months old; undiscovered edge cases possible #archive #maturity
- [risk] Bun.Archive loads entire archive into memory; OOM possible for very large packages #archive #memory
- [fact] Bun.semver passes node-semver test suite and provides satisfies() + order() methods #semver #compatibility
- [fact] Bun.Archive has proper path traversal protection (rejects .., absolute paths, unsafe symlinks) #archive #security
- [insight] All three Bun APIs are usable with documented workarounds; zero third-party deps for core operations #dependency-reduction

## Relations

- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
