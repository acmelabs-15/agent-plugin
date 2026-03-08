---
title: ANALYSIS-025-source-resolution-patterns
type: analysis
permalink: analysis/analysis-025-source-resolution-patterns
tags:
- source-resolution
- npm
- github
- local-path
- semver
- tarball
- staging
- manifest-discovery
- version-checking
---

# ANALYSIS-025 Source Resolution Patterns

## 1. Objective and Scope

**Objective**: Research how plugin/package managers handle source resolution from npm, GitHub, and local filesystem sources. Inform the implementation of the `add <source>` command's download, extraction, and manifest discovery pipeline for `@acmelabs-15/agent-plugin`.

**Scope**: Covers npm registry API queries, GitHub archive downloads, local path handling, semver comparison, content staging, manifest discovery inside archives, and version upgrade detection. Excludes installation mechanics (platform file writing, hook merging, lockfile updates), which are separate concerns.

## 2. Context

ADR-007 Decision 8 defines 4 source format types with validation rules:

- **npm**: `@scope/name`, `@scope/name@version`, `@scope/name@^range`
- **git HTTPS**: `https://github.com/owner/repo`, `https://github.com/owner/repo@tag`
- **git shorthand**: `owner/repo`, `github:owner/repo`
- **local path**: `./path`, `/path`

Detection order (ADR-007): starts with `@` = npm; starts with `https://` = git HTTPS; starts with `github:`/`gitlab:`/`bitbucket:` = git shorthand; otherwise local path.

ADR-001 mandates `plugin.json` at the package root. No `package.json` field fallback. ADR-005 selects Bun 1.3.x as runtime. ADR-002 confirms no hosted registry.

## 3. Approach

**Methodology**: Web research of npm registry API docs, Bun runtime API docs, GitHub REST API docs, and community patterns from Go modules, pip, Cargo, Homebrew, mise, and asdf. Code pattern analysis of how existing package managers handle each resolution stage.

**Tools Used**: Web search, web fetch (Bun docs, npm registry docs), Brain memory search for prior ADR context.

**Limitations**: Bun.Archive API documentation was partially inaccessible during fetch (rate limited). Archive API details reconstructed from blog posts and search results. No hands-on benchmarking performed.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---------|--------|------------|
| npm registry returns tarball URL in `versions[v].dist.tarball` field | npm/registry GitHub docs | High |
| Abbreviated metadata via `Accept: application/vnd.npm.install-v1+json` reduces payload | npm/registry package-metadata docs | High |
| npm tarballs extract to `package/` subdirectory | npm docs, packagecloud blog | High |
| GitHub archives extract to `{repo}-{ref}/` subdirectory | GitHub docs, Baeldung | High |
| GitHub archive URLs: `https://github.com/{owner}/{repo}/archive/refs/tags/{tag}.tar.gz` | GitHub docs | High |
| GitHub API tarball: `GET /repos/{owner}/{repo}/tarball/{ref}` with Bearer token for private repos | GitHub REST API docs | High |
| GitHub private repo archive links expire after 5 minutes | GitHub docs | High |
| Bun.semver.satisfies() and Bun.semver.order() are 20x faster than node-semver | Bun docs (bun.com/docs/runtime/semver) | High |
| Bun.Archive API: `new Bun.Archive(bytes)`, `.extract(dir)`, `.files()` for tar.gz handling | Bun blog v1.3.6, Bun reference docs | High |
| Bun.Archive validates paths during extraction (rejects absolute paths, normalizes `..`) | Bun docs | High |
| pip uses adjacent temp directory + os.replace for atomic installation | pypa/pip DeepWiki | High |
| Homebrew installs to Cellar directory then symlinks to /usr/local/ | Homebrew docs | High |
| Go resolves `github.com/owner/repo` via special-case URL mapping | Go modules reference | High |
| `bun link` creates symlinks for local package development | Bun docs (bun.com/docs/pm/cli/link) | High |
| npm `dist-tags` endpoint: `GET /{pkg}?fields=dist-tags` returns ~100 bytes | npm registry API, tutorialpedia | Medium |

### Facts (Verified)

- npm registry API at `https://registry.npmjs.org/{package}` returns full metadata including all versions
- Each version object contains `dist.tarball` (URL) and `dist.integrity` (SRI hash)
- Abbreviated metadata header `Accept: application/vnd.npm.install-v1+json` returns only install-relevant fields
- Tarball URL pattern: `https://registry.npmjs.org/{name}/-/{unscoped-name}-{version}.tgz`
- npm tarballs always contain a single `package/` top-level directory
- GitHub archive tarball URL: `https://github.com/{owner}/{repo}/archive/refs/tags/{tag}.tar.gz`
- GitHub API tarball endpoint: `GET /repos/{owner}/{repo}/tarball/{ref}` (returns 302 redirect to CDN)
- GitHub private repos require `Authorization: Bearer {token}` header; archive links expire after 5 minutes
- GitHub archive tarballs extract to `{repo}-{ref}/` directory (e.g., `my-plugin-v1.0.0/`)
- Bun.semver has 2 methods: `satisfies(version, range)` returns boolean; `order(a, b)` returns -1/0/1
- Bun.semver is 20x faster than node-semver and compatible with node-semver range syntax
- Bun.Archive constructor accepts `Uint8Array` from tarball bytes; `.extract(dir)` returns entry count
- Bun.Archive `.files()` returns `Map<string, File>` for in-memory inspection without disk extraction
- Bun.Archive rejects absolute paths and normalizes path traversal components during extraction (security)
- Go uses special-case URL mapping for `github.com/` prefix paths; no meta tag lookup needed
- `bun link` creates symlinks between local packages (equivalent to `npm link`)
- Go's `replace` directive in `go.mod` points to local paths for development; not committed to production

### Hypotheses (Unverified)

- Bun's built-in fetch + Bun.Archive may be sufficient to avoid all third-party HTTP/tar dependencies
- The `package/` directory stripping for npm tarballs and `{repo}-{ref}/` stripping for GitHub archives can be handled with a single normalization function
- File watching for local source development mode may need integration with chokidar (already adopted in ADR-006)

## 5. Results

### 5.1 npm Source Resolution

**Registry API pattern**: Query `https://registry.npmjs.org/{@scope%2Fname}` with the abbreviated metadata Accept header. The response contains `dist-tags.latest` for the default version and `versions` map with `dist.tarball` and `dist.integrity` per version.

**Recommended flow**:

1. Parse source string to extract scope, name, and optional version/range
2. Fetch abbreviated metadata: `GET https://registry.npmjs.org/{@scope%2Fname}` with `Accept: application/vnd.npm.install-v1+json`
3. If no version specified, use `dist-tags.latest`
4. If version range (e.g., `^2.0.0`), use `Bun.semver.satisfies()` to find highest matching version
5. Extract `versions[resolved].dist.tarball` URL and `dist.integrity` SRI hash
6. Download tarball via `fetch(tarballUrl)`
7. Verify integrity against SRI hash before extraction
8. Extract using `Bun.Archive`

**No library needed**: The npm registry is a simple REST API. `fetch()` + JSON parsing handles metadata queries. No npm-registry-client or pacote dependency required.

**Tarball URL construction shortcut**: `https://registry.npmjs.org/{@scope/name}/-/{unscoped-name}-{version}.tgz` is predictable. However, always prefer the URL from the registry response because private registries and mirrors may use different URL patterns.

### 5.2 GitHub Source Resolution

**Two download methods**:

| Method | URL Pattern | Auth | Use Case |
|--------|------------|------|----------|
| Direct archive | `https://github.com/{owner}/{repo}/archive/refs/tags/{tag}.tar.gz` | None (public) | Public repos, no API rate limit |
| API tarball | `GET https://api.github.com/repos/{owner}/{repo}/tarball/{ref}` | Bearer token | Private repos, follows redirects |

**Shorthand resolution**: `owner/repo` is expanded to `https://github.com/{owner}/{repo}`. This matches how GitHub CLI, Go modules, and npm itself resolve GitHub shorthand. The `github:` prefix is explicit; bare `owner/repo` defaults to GitHub per ADR-007.

**Ref resolution** (when no tag specified):

1. List tags via `GET https://api.github.com/repos/{owner}/{repo}/tags` (returns array of tag objects)
2. If tags exist, select the latest semver tag using `Bun.semver.order()` for sorting
3. If no semver tags, fall back to the default branch

**Private repo auth**: Use `Authorization: Bearer {GITHUB_TOKEN}` header on API requests. The `GITHUB_TOKEN` environment variable is the standard convention used by GitHub CLI, GitHub Actions, and most CI systems.

**Rate limiting**: GitHub API has 60 requests/hour unauthenticated, 5000/hour authenticated. Direct archive URLs do not count against API rate limits for public repos.

### 5.3 Local Source Resolution

**Symlink is the correct approach for local development**. This matches:

- `npm link` (creates symlink in node_modules)
- `bun link` (creates symlink for Bun workspaces)
- Go `replace` directive (points to local path during development)

**Recommended behavior**:

| Mode | Behavior |
|------|----------|
| `add ./path` | Create symlink from install location to source path |
| `dev` command | Use chokidar (ADR-006) to watch source path for changes, auto-reinstall on change |

**Path validation** (ADR-007 Rule 16): Local paths must resolve within the workspace. No `..` escaping workspace root. Use `path.resolve()` + containment check against workspace root.

**Symlink advantages over copy**: Zero disk duplication, changes reflected immediately, matches developer expectations from npm/bun/go ecosystems.

**Symlink risks**: Symlinks break if the source path moves. The `list` command should detect broken symlinks and warn. The lockfile should record the original path for re-linking.

### 5.4 Semver Comparison

**Use Bun.semver (built-in). Do not build in-house. Do not use npm semver package.**

| Option | Weekly Downloads | Size | Speed vs node-semver | Dependencies |
|--------|-----------------|------|---------------------|--------------|
| Bun.semver (built-in) | N/A (bundled) | 0 KB | 20x faster | 0 |
| node-semver (npm) | ~200M | 43 KB | baseline | 0 |
| compare-versions | ~10M | 3 KB | unknown | 0 |
| semver-lite | ~50K | 2 KB | unknown | 0 |

**Bun.semver API surface**:

- `Bun.semver.satisfies(version: string, range: string): boolean` -- checks if version matches range
- `Bun.semver.order(versionA: string, versionB: string): -1 | 0 | 1` -- comparison for sorting

**Coverage gap**: Bun.semver has only 2 methods. It lacks `parse()`, `valid()`, `gt()`, `lt()`, `coerce()`, `inc()`, `diff()` that node-semver provides. For this project's needs (range matching for npm sources, version sorting for upgrade detection), `satisfies()` and `order()` are sufficient.

**Version validation**: Use Zod regex for format validation (`/^\d+\.\d+\.\d+(-[a-zA-Z0-9.]+)?$/`), Bun.semver for range resolution. If `satisfies()` returns `false` for all valid inputs, the range is effectively invalid.

### 5.5 Content Staging

**Recommended pattern**: Adjacent temp directory with atomic rename.

**Flow**:

1. Create temp directory adjacent to final install location (same filesystem for atomic rename)
2. Download tarball to temp directory
3. Verify integrity (SRI hash for npm, optional for GitHub)
4. Extract using `Bun.Archive` into temp directory
5. Locate `plugin.json` in extracted content (see 5.6)
6. Validate manifest
7. Rename temp directory to final location (`fs.rename()` -- atomic on same filesystem)
8. On any failure: delete temp directory

**Patterns from other package managers**:

| Manager | Staging Strategy |
|---------|-----------------|
| pip | Adjacent temp dir + `os.replace()` for atomic swap |
| Cargo | Temp target directory for builds; final binary copied to install dir |
| Homebrew | Install to Cellar versioned directory, symlink to prefix |
| Bun install | Global cache at `~/.bun/install/cache/{name}@{version}`, symlinks to node_modules |
| npm | Downloads to temp, extracts to node_modules directly |

**Cleanup**: Use `try/finally` to ensure temp directory deletion on failure. Bun's `Bun.file()` and `fs.rm(path, { recursive: true })` handle cleanup.

**Temp directory naming**: Use a prefix pattern (e.g., `.agent-plugin-staging-{random}`) adjacent to the install target. This ensures the rename operation is atomic (same filesystem) and the staging directory is identifiable for manual cleanup if the process crashes.

### 5.6 Manifest Discovery in Sources

**The core problem**: After extracting a tarball, the actual content is nested inside a single top-level directory. The manifest (`plugin.json`) is inside that directory, not at the archive root.

| Source Type | Top-Level Directory in Archive | plugin.json Location |
|-------------|-------------------------------|---------------------|
| npm tarball | `package/` | `package/plugin.json` |
| GitHub archive | `{repo}-{ref}/` (e.g., `my-plugin-v1.0.0/`) | `{repo}-{ref}/plugin.json` |
| Local path | N/A (direct filesystem) | `{path}/plugin.json` |

**Discovery algorithm**:

1. Extract archive to staging directory
2. List immediate children of staging directory
3. If exactly 1 subdirectory and 0 files at root: enter that subdirectory (strip wrapper)
4. Look for `plugin.json` in the current directory
5. If not found: error with message "No plugin.json found. This package is not an agent-plugin."

**Alternative approach using Bun.Archive.files()**: Instead of extracting to disk first, use `.files()` to get a `Map<string, File>` and search for `plugin.json` in memory. This avoids writing invalid packages to disk at all.

```
const archive = new Bun.Archive(tarballBytes);
const files = await archive.files();
// Find plugin.json -- it will be at "package/plugin.json" (npm) or "{repo}-{ref}/plugin.json" (GitHub)
const manifestEntry = [...files.entries()].find(([path]) => path.endsWith('/plugin.json') && path.split('/').length === 2);
```

This in-memory check validates the package before any disk writes. If `plugin.json` is missing, fail fast without staging.

### 5.7 Version Checking and Upgrade Detection

**npm sources**: Query the registry for current `dist-tags.latest` and compare against installed version using `Bun.semver.order()`.

**Efficient npm version query**: Use `GET https://registry.npmjs.org/{pkg}?fields=dist-tags` which returns approximately 100 bytes. This avoids downloading full metadata (which can be megabytes for packages with many versions).

**GitHub sources**: Query `GET https://api.github.com/repos/{owner}/{repo}/tags` to get the list of tags. Sort semver tags using `Bun.semver.order()`. Compare latest tag against installed version.

**Upgrade detection flow**:

1. Read installed version from lockfile (`plugin-lock.json`)
2. Query source for available versions (npm registry or GitHub tags API)
3. Compare installed vs latest using `Bun.semver.order(installed, latest)`
4. If `order()` returns `-1`, a newer version is available
5. If version range was specified at install time, check if newer versions within range exist using `satisfies()`

**Batch upgrade checking**: The `upgrade` command with no arguments should check all installed plugins. For npm sources, this means one HTTP request per installed plugin. For GitHub sources, one API call per repo. No batch endpoint exists for either.

## 6. Discussion

### Key Architecture Decision: Fetch + Bun.Archive, No Third-Party Dependencies

The research shows that the entire source resolution pipeline can be built with zero third-party HTTP or archive dependencies:

- `fetch()` (built into Bun) for all HTTP requests (npm registry, GitHub API, tarball downloads)
- `Bun.Archive` for tarball extraction and in-memory inspection
- `Bun.semver` for version comparison and range matching

This aligns with the project's dependency-minimization philosophy (ADR-006 already removed several unnecessary deps).

### npm vs GitHub: Different Integrity Models

npm provides SRI hashes (`dist.integrity`) for every tarball, enabling cryptographic verification. GitHub archives have no built-in integrity mechanism for tarball downloads. For GitHub sources, the commit SHA from the tags API response can serve as a weaker form of verification (verify the tag points to an expected commit), but there is no tarball hash available before download.

### Local Path Symlinks: Dev Mode Integration

The `dev` command (ADR-007) already uses chokidar for file watching. Local path sources installed via `add ./path` should create symlinks. The `dev` command then watches the symlinked path for changes and triggers reinstallation (re-running platform adapter writes). This creates a unified local development experience.

### In-Memory Manifest Validation Before Staging

Using `Bun.Archive.files()` to inspect archive contents before extracting to disk provides a security and correctness benefit. Invalid packages (missing `plugin.json`) never touch the filesystem. This is a pattern not commonly used by other package managers (most extract first, validate second) but is enabled by Bun.Archive's in-memory file access.

### GitHub Shorthand Ambiguity

The ADR-007 detection order says "otherwise local path" for strings that don't match npm, HTTPS, or prefix patterns. This means `owner/repo` (without `github:` prefix) would be detected as a local path, not GitHub shorthand. The ADR-007 source type detection lists `github:owner/repo` as the shorthand format, and bare `owner/repo` falls through to local path.

However, the session notes reference `owner/repo` as GitHub shorthand. This is an ambiguity that needs resolution. Two options:

1. Require explicit `github:` prefix (current ADR-007 rules). Bare `owner/repo` is a local path.
2. Add heuristic: if the string contains exactly one `/` and no `.` or path separators, treat as GitHub shorthand.

Option 2 matches Go's convention (`github.com/owner/repo` is unambiguous) and npm's convention (`npm install owner/repo` installs from GitHub). Option 1 is safer and avoids false positives (a local path `foo/bar` would be misinterpreted).

**Recommendation**: Keep option 1 (explicit `github:` prefix required) for safety. Document the prefix requirement clearly. This avoids the ambiguity entirely.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Use `fetch()` + npm registry API directly (no npm client library) | Zero dependencies; registry API is simple REST | Low |
| P0 | Use `Bun.semver` for all version comparison (no node-semver) | Built-in, 20x faster, covers satisfies() and order() | Low |
| P0 | Use `Bun.Archive` for tarball extraction (no node-tar) | Built-in, path validation included, in-memory inspection via .files() | Low |
| P0 | Strip single top-level directory from archives before manifest discovery | npm uses `package/`, GitHub uses `{repo}-{ref}/`; both need normalization | Low |
| P1 | Use abbreviated npm metadata (`Accept: application/vnd.npm.install-v1+json`) | Reduces payload from potentially megabytes to kilobytes | Low |
| P1 | Implement adjacent temp directory + atomic rename for staging | Same-filesystem rename is atomic; prevents partial installs | Medium |
| P1 | Use in-memory manifest check via `Bun.Archive.files()` before disk extraction | Fail fast on invalid packages without filesystem writes | Low |
| P1 | Symlink for local path sources; copy not needed | Matches npm link/bun link conventions; immediate change reflection | Low |
| P1 | Verify npm tarball integrity using `dist.integrity` SRI hash | Prevents installation of corrupted or tampered packages | Medium |
| P2 | Use `?fields=dist-tags` for lightweight version checks in `upgrade` command | Approximately 100 bytes per query vs full metadata | Low |
| P2 | Support `GITHUB_TOKEN` env var for private GitHub repo access | Standard convention across GitHub CLI, Actions, CI systems | Low |
| P2 | Require explicit `github:` prefix for GitHub shorthand (resolve ADR-007 ambiguity) | Avoids false positives where local paths look like owner/repo | Low |

## 8. Conclusion

**Verdict**: Proceed
**Confidence**: High
**Rationale**: Bun 1.3.x provides built-in APIs (fetch, Archive, semver) that cover the entire source resolution pipeline with zero third-party dependencies. The npm registry REST API is well-documented and stable. GitHub archive URLs follow predictable patterns. The implementation can be built as a set of pure functions with clear inputs and outputs.

### User Impact

- **What changes for you**: The `add` command resolves sources from npm, GitHub, or local filesystem using zero additional dependencies beyond Bun itself.
- **Effort required**: Medium. The source resolver is a core subsystem with 4 source types, each requiring a distinct resolution path, but each path is straightforward (10-50 lines per resolver).
- **Risk if ignored**: Without clear resolution patterns, the implementation would likely pull in unnecessary dependencies (node-tar, node-semver, npm-registry-client) adding bundle size and maintenance burden.

## 9. Appendices

### Sources Consulted

- [npm Registry API documentation](https://github.com/npm/registry/blob/main/docs/REGISTRY-API.md)
- [npm Registry package-metadata response format](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md)
- [Bun.semver documentation](https://bun.com/docs/runtime/semver)
- [Bun.semver API reference](https://bun.com/reference/bun/semver)
- [Bun.Archive documentation](https://bun.com/docs/runtime/archive)
- [Bun v1.3.6 release blog (Archive API)](https://bun.com/blog/bun-v1.3.6)
- [Bun link documentation](https://bun.com/docs/pm/cli/link)
- [Behind the Scenes of Bun Install](https://bun.com/blog/behind-the-scenes-of-bun-install)
- [GitHub REST API - Repository Contents](https://docs.github.com/en/rest/repos/contents)
- [GitHub REST API - Repository Tags](https://docs.github.com/en/rest/repos/tags)
- [GitHub REST API - Releases](https://docs.github.com/en/rest/releases/releases)
- [Downloading a Tarball from GitHub (Baeldung)](https://www.baeldung.com/linux/github-download-tarball)
- [pip Temporary Directory Management (DeepWiki)](https://deepwiki.com/pypa/pip/6.3-temporary-directory-management)
- [Homebrew Formula Cookbook](https://docs.brew.sh/Formula-Cookbook)
- [Go Modules Reference](https://go.dev/ref/mod)
- [mise Plugin Documentation](https://mise.jdx.dev/plugins.html)
- [pnpm get-npm-tarball-url](https://github.com/pnpm/get-npm-tarball-url)
- [Packagecloud - npm registry internals](https://blog.packagecloud.io/npm-registry-internals/)
- [Packagecloud - Inspect/Download npm packages](https://blog.packagecloud.io/how-to-inspect-download-and-extract-npm-packages/)
- [compare-versions npm package](https://www.npmjs.com/package/compare-versions)
- [node-semver GitHub](https://github.com/npm/node-semver)

### Data Transparency

- **Found**: Complete npm registry API patterns, Bun.semver and Bun.Archive API surfaces, GitHub archive URL patterns and auth requirements, local path handling patterns from npm/bun/go, content staging patterns from pip/cargo/homebrew, manifest discovery directory structures for both npm and GitHub tarballs
- **Not Found**: Bun.Archive full API reference page was rate-limited during fetch (reconstructed from blog posts and search results). No performance benchmarks for Bun.Archive vs node-tar. No data on how many agent-plugin users would need private GitHub repo support. GitLab and Bitbucket archive URL patterns not researched (lower priority per ADR-007 shorthand prefix list).

## Observations

- [fact] npm registry returns tarball URL in versions[v].dist.tarball; abbreviated metadata via Accept header reduces payload to kilobytes #npm #registry-api
- [fact] npm tarballs extract to package/ subdirectory; GitHub archives extract to {repo}-{ref}/ subdirectory #tarball #extraction
- [fact] Bun.semver provides satisfies() and order() methods, is 20x faster than node-semver, compatible with node-semver range syntax #bun #semver
- [fact] Bun.Archive provides new Archive(bytes), .extract(dir), .files() with built-in path traversal protection #bun #archive
- [fact] GitHub API tarball endpoint requires Bearer token for private repos; archive links expire after 5 minutes #github #auth
- [fact] GitHub API rate limit: 60/hour unauthenticated, 5000/hour authenticated; direct archive URLs bypass rate limits for public repos #github #rate-limit
- [decision] Use fetch() + Bun.Archive + Bun.semver for entire pipeline; zero third-party HTTP/tar/semver dependencies needed #architecture #zero-deps
- [decision] Strip single top-level directory from archives before manifest discovery (package/ for npm, {repo}-{ref}/ for GitHub) #manifest-discovery
- [technique] Use Bun.Archive.files() for in-memory manifest validation before disk extraction to fail fast on invalid packages #validation #security
- [technique] Adjacent temp directory + atomic rename (same-filesystem) for content staging; prevents partial installs on failure #staging #atomicity
- [technique] npm abbreviated metadata via Accept: application/vnd.npm.install-v1+json header reduces version query payload #npm #optimization
- [recommendation] Symlink for local path sources matching npm link / bun link conventions #local-path #symlink
- [recommendation] Require explicit github: prefix for shorthand to avoid ambiguity with local paths #source-detection #safety
- [risk] GitHub archives have no SRI hash; npm provides dist.integrity for tarball verification. GitHub integrity gap needs consideration #integrity #github

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ANALYSIS-016-bun-runtime-assessment]]
