---
title: ANALYSIS-011 Lockfile Management Patterns
type: note
permalink: analysis/analysis-011-lockfile-management-patterns
tags:
- lockfile
- state-management
- atomic-writes
- analysis
- agent-plugin
---

# ANALYSIS-011 Lockfile Management Patterns

## 1. Objective and Scope

**Objective**: How should @acmelabs-15/agent-plugin implement its JSON lockfile (`plugin-lock.json`) for tracking installed plugins, their components, and file modifications? What packages and patterns make this robust?

**Scope**: Covers 4 research questions: (1) how pnpm, yarn, and npm handle their lockfiles internally, (2) what packages exist for robust lockfile implementation, (3) community best practices for lockfile management, (4) recommended approach for our use case.

**Excluded**: The overlay/recompute pattern for hook merging (covered in ANALYSIS-010). This analysis focuses on the lockfile itself as a state-tracking mechanism.

## 2. Context

@acmelabs-15/agent-plugin needs a JSON lockfile (`plugin-lock.json`) to track:

- Installed plugins and their versions
- Component inventory per plugin (hooks, skills, MCPs, templates, etc.)
- File modifications made during installation (for clean uninstall)
- Hook provenance (which plugin contributed which hooks)

Prior decisions established:

- JSON lockfile format (like package-lock.json)
- `lockfileVersion` field for schema evolution
- Re-derive from disk on corruption (lockfile is cache, not source of truth)
- Always-namespace model (deterministic `plugin:component` naming)
- Overlay/recompute pattern for hook merging (ANALYSIS-010)

The lockfile complements the overlay state directory. Overlays track individual plugin hook contributions. The lockfile tracks the full plugin inventory, component registry, and file modification history.

## 3. Approach

**Methodology**: Web research across 30+ sources covering npm/pnpm/yarn internals, atomic write libraries, lockfile design research papers, and community best practices. Fetched actual source code from pnpm, steno, atomically, conf, and proper-lockfile repositories.

**Tools Used**: WebSearch (16 queries), WebFetch (7 pages including raw GitHub source files), Brain MCP search (3 queries).

**Limitations**: pnpm's exact concurrent access strategy is not documented publicly. Yarn Berry's atomic write mechanism for the lockfile itself is not explicitly documented. npm's internal lockfile write path could not be traced end-to-end from public sources.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| pnpm uses `write-file-atomic` for lockfile writes, wrapping its callback API in a Promise | pnpm source code (lockfile/fs/src/write.ts) | High |
| pnpm maintains two lockfiles: wanted (pnpm-lock.yaml) and current (node_modules/.pnpm-lock.yaml) | DeepWiki pnpm docs, @pnpm/lockfile-file npm | High |
| pnpm handles git merge conflicts via branch-specific lockfiles (.git/pnpm-lock/{branch}.yaml) | pnpm.io/git_branch_lockfiles | High |
| npm uses `write-file-atomic` (temp file + murmurhash naming + rename) with same-file write queuing | write-file-atomic README, GitHub source | High |
| npm lockfileVersion evolved from 1 (npm v6) to 2 (npm v7, backward compatible) to 3 (npm v7, no backward compat) | npm docs, community articles | High |
| Yarn Berry checksums are hashes of cached zip files, not content hashes, causing cross-platform issues | yarnpkg/berry issues #6068, #5575, #6105 | High |
| `write-file-atomic` receives 45.8M weekly downloads; uses temp file + rename + same-file queue | npm, socket.dev | High |
| `atomically` receives 2.1M weekly downloads; retries failed operations, 0 third-party deps, used by `conf` | npm, GitHub source | High |
| `steno` receives 2.0M weekly downloads; smart queue drops intermediate writes, 1000x faster than fs for repeated writes | npm, GitHub source | High |
| `proper-lockfile` receives 3.9M weekly downloads; uses mkdir for atomic lock acquisition with stale detection via mtime | npm, GitHub source | High |
| `conf` (sindresorhus) uses `atomically` for writes, supports semver-based migrations, AJV schema validation | GitHub source (conf/source/index.ts) | High |
| `configstore` receives 13.9M weekly downloads but is predecessor to `conf`; `conf` is the recommended successor | npm, sindresorhus GitHub | High |
| `lowdb` receives 914K weekly downloads; uses `steno` for writes; provides JSONFile adapter | npm, lowdb GitHub | High |
| Flat lockfile layouts merge better in git than nested ones; entries should be independent and sorted | Nesbitt blog post on lockfile format design | High |
| Record `lockfile_version` (format compat) NOT tool version; format stability matters more than elegance | Nesbitt blog post, arxiv paper | High |
| Academic research (arxiv 2505.04834) found no documented research on lockfile corruption, concurrent access, or schema versioning | arxiv paper | High |

### Facts (Verified)

- [fact] pnpm wraps `write-file-atomic` in a Promise for lockfile writes. The dual-lockfile system (wanted vs current) lets pnpm detect when node_modules is out of sync with desired state.
- [fact] npm's `write-file-atomic` uses temp file naming via murmurhash of filename + PID + invocation count, then atomic rename. Concurrent writes to the same file are serialized via a Promise queue.
- [fact] Yarn Berry's `@yarnpkg/fslib` provides filesystem abstractions but does not document an explicit atomic write strategy for yarn.lock. Checksums are hashes of cached zip files, causing known cross-platform inconsistencies.
- [fact] `atomically` (used by `conf`) retries failed operations for up to 7500ms async / 1000ms sync, handles ENAMETOOLONG via path truncation, and uses a Scheduler for file-level write serialization.
- [fact] `steno` achieves 1000x performance over raw `fs.writeFile` for repeated writes by smart queue batching: only the most recent write payload is kept in the queue, eliminating redundant writes.
- [fact] `proper-lockfile` uses `mkdir` as an atomic lock acquisition primitive (EEXIST = already locked). Stale locks are detected by comparing mtime against a configurable threshold (default 10s). Lock freshness maintained via periodic `utimes()` calls at stale/2 intervals.
- [fact] npm lockfileVersion uses integer versioning with backward compatibility affordances: v2 includes both `packages` and legacy `dependencies` fields; v3 drops `dependencies` to reduce size.
- [fact] The 2025 arxiv paper studying 7 package managers found Go has 99.7% lockfile adoption (automatic generation) vs Gradle at 0.9% (opt-in). Automatic generation is the single biggest predictor of adoption.

### Hypotheses (Unverified)

- [hypothesis] `write-file-atomic` is sufficient for our use case without `proper-lockfile`, because agent-plugin is a CLI tool with single-writer guarantee (only one CLI invocation modifies the lockfile at a time).
- [hypothesis] Schema migration via integer lockfileVersion + migration functions (conf pattern) is simpler and more appropriate for our use case than semver-based versioning.
- [hypothesis] Sorted-key deterministic JSON serialization provides meaningful git diff improvement for plugin-lock.json, even though the lockfile is primarily machine-managed.

## 5. Results

### RQ1: How Do Major Package Managers Handle Their Lockfiles?

#### pnpm

**Internal Packages**: `@pnpm/lockfile.fs` (I/O), `@pnpm/lockfile.types` (TypeScript types), `@pnpm/lockfile.utils` (manipulation), `@pnpm/lockfile.verification` (validation), `@pnpm/lockfile.settings-checker` (settings comparison).

**Atomic Writes**: Uses `write-file-atomic` directly. The `writeLockfiles()` function in `@pnpm/lockfile.fs` writes both wanted and current lockfiles. When both are identical, YAML is stringified once for efficiency.

**Two-Lockfile System**:

- Wanted lockfile (`pnpm-lock.yaml`): desired dependency tree. Committed to git.
- Current lockfile (`node_modules/.pnpm-lock.yaml`): actual installed tree. Gitignored.
- Comparison detects when `node_modules` is out of sync with wanted state.

**Concurrent Access**: Not explicitly documented. pnpm relies on the single-CLI-invocation model. The store uses per-package locks for download concurrency (issue #941).

**Git Merge Conflicts**: Branch-specific lockfiles (`pnpm-lock.{branch}.yaml`) with `--merge-git-branch-lockfiles` to reunify. Deep merge strategy selects the higher version number on conflict.

**Schema Versioning**: `LOCKFILE_VERSION = 9.0`. Major version checked for compatibility. Outdated major version triggers full re-resolution with `needsFullResolution = true`.

**Empty Handling**: Empty lockfiles are deleted via `rimraf` rather than written as empty files.

#### npm

**Atomic Writes**: Uses `write-file-atomic` throughout. The package generates temp file names via murmurhash(filename, PID, invocationCount), writes to temp, then atomic rename.

**Concurrent Write Safety**: `write-file-atomic` serializes concurrent writes to the same file path via a Promise queue. Writes to different files remain parallel.

**Schema Versioning**: Integer `lockfileVersion` field.

- v1 (npm v5-v6): nested `dependencies` structure
- v2 (npm v7+): `packages` field + `dependencies` for backward compatibility
- v3 (npm v7+): `packages` only, no backward compat (used for hidden lockfile at `node_modules/.package-lock.json`)

**Corruption Handling**: If package-lock.json is corrupted or missing, npm regenerates it from package.json + registry resolution. The lockfile is treated as a cache that can be rebuilt.

#### Yarn Berry

**Internal Packages**: `@yarnpkg/parsers` (lockfile parsing), `@yarnpkg/fslib` (filesystem abstraction layer).

**Lockfile Format**: YAML with special formatting (extra empty line delimiters, extra quotes). Top-level keys split by commas for deduplication.

**Checksums**: Hashes of cached zip files, NOT content hashes. Tied to compression level setting. Known cross-platform inconsistencies when compression level differs between machines. This is a design weakness.

**Atomic Writes**: Uses `@yarnpkg/fslib` abstractions including `xfs.writeFilePromise`. The exact atomicity guarantee is not documented.

**Update Modes**: `--immutable` prevents lockfile changes (CI enforcement). `update-lockfile` mode skips linking and only updates missing entries.

### RQ2: Packages for Robust Lockfile Implementation

#### Atomic Write Libraries

| Package | Weekly Downloads | Mechanism | Concurrency | Retry | Dependencies | License |
|---|---|---|---|---|---|---|
| write-file-atomic | 45.8M | Temp file (murmurhash name) + rename | Promise queue per file path | No | signal-exit, imurmurhash | ISC |
| atomically | 2.1M | Temp file + rename + Scheduler | Scheduler-based file locking | Yes (7.5s async, 1s sync) | stubborn-fs | MIT |
| steno | 2.0M | Temp file (.{name}.tmp) + rename with retry | Smart queue (latest-write-wins) | Yes (10 attempts, 100ms delay) | None (0 deps) | MIT |

**write-file-atomic**: The industry standard. Used by npm, pnpm, and thousands of packages. Battle-tested at massive scale. Callback-based API (requires Promise wrapper). No retry on failure. No read support.

**atomically**: Rewrite of write-file-atomic by fabiospampinato. Adds retry with configurable timeout, handles more error types (ENAMETOOLONG), 0 third-party deps (stubborn-fs is same author). Used by `conf`. Slightly faster. TypeScript native.

**steno**: Specialized for repeated writes to the same file. Smart queue drops intermediate writes, keeping only the latest payload. 1000x faster than raw fs for repeated same-file writes. 0 dependencies. Best for high-frequency write scenarios. Used by `lowdb`.

**Verdict**: For our use case (infrequent writes, single CLI invocation), `write-file-atomic` or `atomically` are appropriate. `steno` is optimized for a write pattern we do not have (high-frequency repeated writes). `atomically` provides the best fit: retry on failure, 0 third-party deps, TypeScript native, actively maintained.

#### File Locking Libraries

| Package | Weekly Downloads | Mechanism | Stale Detection | Cross-Machine |
|---|---|---|---|---|
| proper-lockfile | 3.9M | mkdir (atomic on all FS) | mtime threshold (10s default) | Yes (network FS) |
| lockfile (npm) | 5.8M | File creation + polling | Stale option with wait | No |

**proper-lockfile**: Uses `mkdir` as atomic lock primitive (EEXIST = already locked). Stale lock detection via mtime comparison. Periodic mtime update (stale/2 interval) maintains lock freshness. Compromised lock detection triggers `onCompromised` callback. Works on network filesystems.

**Verdict**: File locking is unnecessary for our use case. agent-plugin is a CLI tool with a single-writer model. Only one CLI invocation modifies the lockfile at a time. If we later need multi-process safety (e.g., concurrent plugin installs from different terminals), `proper-lockfile` is the right choice.

#### JSON Config/Database Libraries

| Package | Weekly Downloads | Atomic Writes | Schema Validation | Migration | Dot-Notation | Suitable |
|---|---|---|---|---|---|---|
| conf | 1.3M | Yes (atomically) | AJV JSON Schema | Semver-based | Yes (dot-prop) | Partial |
| configstore | 13.9M | Yes (write-file-atomic) | No | No | No | No |
| lowdb | 914K | Yes (steno) | No | No | No | Partial |

**conf**: The most feature-rich option. Atomic writes via `atomically`. JSON Schema validation via AJV. Semver-based migration system. Dot-notation access. File watching for external changes. Corruption recovery (returns empty object on invalid JSON). However: designed for app configuration, not lockfile state. Opinionated about file location (system config dir). The migration system expects semver project versions, not integer lockfile versions.

**configstore**: Predecessor to `conf`. Higher downloads due to inertia. No schema validation or migration. Stores in `~/.config`. Not recommended for new projects.

**lowdb**: JSON database with adapters. Treats JSON file as a database. Uses `steno` for fast writes. No schema validation. No migration. Good for CRUD operations on JSON data. More database-like than lockfile-like.

**Verdict**: None of these are a direct fit for a lockfile. `conf` is closest but imposes opinions about file location, config-dir conventions, and migration semantics that do not match lockfile requirements. Better to compose from lower-level primitives: `atomically` (or `write-file-atomic`) for writes + custom schema validation + custom migration.

#### Deterministic Serialization Libraries

| Package | Weekly Downloads | Purpose |
|---|---|---|
| json-stable-stringify | 17M+ | Deterministic JSON with sorted keys |
| safe-stable-stringify | 12M+ | Fast, safe deterministic JSON (handles circular refs) |
| json-stringify-deterministic | 180K | Deterministic with custom comparators |

**Verdict**: Use `JSON.stringify()` with a key-sorting replacer function, or `safe-stable-stringify` for production robustness. Sorted keys ensure minimal git diffs when the lockfile changes.

### RQ3: Community Best Practices for Lockfile Management

#### Atomic Write Pattern (Temp File + Rename)

This is the universal standard. Every production lockfile implementation uses it:

1. Write data to a temporary file (same directory as target)
2. Call `fsync` to flush to disk
3. Rename temp file to target path (atomic on POSIX)
4. On failure, unlink temp file

**Why same-directory temp files**: Rename is atomic only within the same filesystem mount. Cross-device rename falls back to copy+delete (non-atomic).

**Why fsync matters**: Without fsync, data may be in the OS page cache but not on disk. A power failure after rename but before flush could produce a zero-length file.

#### Concurrent Access Patterns

Three strategies exist in the ecosystem:

1. **Single-writer guarantee** (npm, pnpm): Assume only one process writes the lockfile. No locking needed. Simplest and most common.

2. **File-level locking** (proper-lockfile): Acquire an exclusive lock before read-modify-write. Handles multi-process scenarios. Adds complexity and failure modes (stale locks, deadlocks).

3. **Write serialization** (write-file-atomic, atomically): Queue concurrent writes to the same file path. Does NOT prevent read-modify-write races (two processes can read the same state, compute different updates, and the second write overwrites the first).

**Best practice for CLI tools**: Single-writer guarantee. Document that concurrent CLI invocations are unsupported. Add a PID-based lock check as a safety net (warn, do not block).

#### Corruption Detection and Recovery

**Detection approaches**:

- JSON.parse failure (syntax error = corruption)
- Checksum mismatch (store hash of last-written content, compare on read)
- Schema validation failure (valid JSON but wrong structure = version mismatch or corruption)

**Recovery approaches**:

- **Re-derive from disk** (our chosen approach): Scan installed plugins and reconstruct lockfile. Lockfile is cache, not source of truth.
- **Backup restore**: Keep `plugin-lock.json.bak` from last successful write. Restore on corruption. Used by `conf`.
- **Empty state**: Return empty/default state on corruption. User must reinstall. Last resort.

**Best practice**: Layer all three. Try JSON.parse. On failure, try backup. On failure, re-derive from disk. Log all recovery actions.

#### Schema Versioning and Migration

**npm model (integer versioning)**: `lockfileVersion: 1, 2, 3`. Simple. Major version check for compatibility. Backward compatibility via dual-field approach (v2 includes both old and new formats).

**conf model (semver migration)**: Semver version ranges with migration callbacks. More powerful but more complex. Designed for user-facing config evolution.

**pnpm model (float versioning)**: `LOCKFILE_VERSION = 9.0`. Major version for compatibility checks. Minor version for non-breaking additions.

**Best practice for our use case**: Integer versioning with migration functions. Start at `lockfileVersion: 1`. On read, check version. If version < current, run migration pipeline. Each migration transforms v(N) to v(N+1). After migration, write updated lockfile.

#### Git-Friendliness

**Deterministic serialization**: Sort object keys alphabetically. Use consistent indentation (2 spaces). Produce identical output for identical input.

**Flat structure**: Each plugin entry should be independent. Adding/removing a plugin should change only that entry's lines, not cascade through the structure.

**Entry independence**: Avoid cross-references between entries where possible. Each plugin entry should be self-contained.

**Trailing commas**: JSON does not support trailing commas. Accept that the last entry's closing line will change when a new entry is appended. This is unavoidable in JSON.

### RQ4: Recommended Approach for Our Use Case

#### Architecture: Lockfile as State Cache

```
plugin-lock.json (single file, project root or ~/.agent-plugin/)
  |
  +-- lockfileVersion: integer (schema evolution)
  +-- generatedAt: ISO timestamp (informational)
  +-- plugins: { [name]: PluginEntry }
  |     +-- version, source, installedAt
  |     +-- components: { hooks, skills, mcps, templates, ... }
  |     +-- fileModifications: [ { path, action, hash, backup } ]
  +-- _integrity: SHA-256 of content (corruption detection)
```

The lockfile is a **cache** that can be re-derived from disk. It is NOT the source of truth. The installed plugin files on disk ARE the source of truth. This matches our prior decision and aligns with how npm treats package-lock.json.

#### Package Selection

| Layer | Package | Rationale |
|---|---|---|
| Atomic writes | `atomically` | Retry on failure, 0 third-party deps, TypeScript native, used by `conf`. Alternative: `write-file-atomic` (more battle-tested but callback API, no retry). |
| File locking | None (single-writer) | CLI tool with single-writer guarantee. Add `proper-lockfile` later if multi-process support needed. |
| JSON serialization | Built-in `JSON.stringify` with sorted keys | Use a `replacer` function or `safe-stable-stringify` for deterministic output. |
| Schema validation | `ajv` (if complex) or manual | Start with manual validation. Add AJV if schema grows complex. |
| Migration | Custom integer-based pipeline | Each migration transforms v(N) to v(N+1). Simpler than semver for lockfile evolution. |

#### Write Flow

```
1. Read current plugin-lock.json
2. Parse JSON. On SyntaxError:
   a. Try plugin-lock.json.bak
   b. On failure, re-derive from disk scan
   c. Log recovery action
3. Validate lockfileVersion
   a. If version < CURRENT_VERSION, run migration pipeline
   b. If version > CURRENT_VERSION, warn (newer tool version created this)
4. Modify in-memory state
5. Compute SHA-256 of new content (minus _integrity field)
6. Set _integrity field
7. Serialize with sorted keys + 2-space indent
8. Write to plugin-lock.json.bak (backup of previous)
9. Atomic write to plugin-lock.json via atomically
```

#### Read Flow

```
1. Read plugin-lock.json
2. Parse JSON. On SyntaxError:
   a. Try plugin-lock.json.bak
   b. On failure, re-derive from disk scan
3. Verify _integrity hash
   a. On mismatch, warn (file modified externally)
   b. Re-derive from disk if critical
4. Validate lockfileVersion, migrate if needed
5. Return parsed state
```

#### Migration Pattern

```typescript
const CURRENT_LOCKFILE_VERSION = 1;

const migrations: Record<number, (state: any) => any> = {
  // Example: v1 -> v2 migration
  // 2: (state) => {
  //   // Transform state from v1 schema to v2 schema
  //   for (const plugin of Object.values(state.plugins)) {
  //     plugin.components = plugin.components ?? {};
  //   }
  //   state.lockfileVersion = 2;
  //   return state;
  // }
};

function migrate(state: any): any {
  let version = state.lockfileVersion ?? 0;
  while (version < CURRENT_LOCKFILE_VERSION) {
    const nextVersion = version + 1;
    const migrator = migrations[nextVersion];
    if (!migrator) throw new Error(`No migration for v${version} -> v${nextVersion}`);
    state = migrator(state);
    version = nextVersion;
  }
  return state;
}
```

## 6. Discussion

### Why Not Use conf or lowdb Directly?

Both `conf` and `lowdb` are designed for different use cases:

- `conf` is for application configuration (user preferences, settings). It stores files in the system config directory, supports dot-notation access patterns, and uses semver-based migrations tied to project version. Our lockfile is project-scoped state tracking, not user configuration.

- `lowdb` is a JSON database. It provides CRUD operations and treats the JSON file as a data store. Our lockfile is a flat state snapshot, not a database with query needs.

Using either would impose abstractions that do not match our requirements and would hide the write-flow details we need to control (integrity hashing, backup rotation, migration pipeline, re-derivation from disk).

### Why atomically Over write-file-atomic?

Both are viable. `atomically` has three advantages for our use case:

1. **Retry on failure**: Default 7.5s async retry window. `write-file-atomic` fails on first error.
2. **Zero third-party deps**: Easier to audit. `write-file-atomic` depends on `signal-exit` and `imurmurhash`.
3. **TypeScript native**: No @types package needed.

`write-file-atomic` has one advantage: 20x more weekly downloads (45.8M vs 2.1M), indicating broader battle-testing. Both are maintained by reputable authors (npm org vs fabiospampinato).

For a CLI tool that writes infrequently, either works. `atomically` is the slight edge due to retry behavior.

### The Integrity Hash Question

Including `_integrity` in the lockfile enables detecting external modification. But it creates a chicken-and-egg: the hash must exclude itself. Two approaches:

1. **Exclude _integrity from hash computation**: Serialize without `_integrity`, compute hash, then add `_integrity` to the serialized output. On read, strip `_integrity`, reserialize, compare hash.

2. **Separate integrity file**: Store hash in `plugin-lock.json.sha256` alongside the lockfile. Simpler but adds a second file.

Approach 1 is cleaner (single file) and matches how npm stores `integrity` hashes inline. Approach 2 mirrors Go's `go.sum` pattern. Recommend approach 1 for simplicity.

### Relationship to Overlay State Directory

ANALYSIS-010 recommended an overlay/recompute pattern for hook merging. The lockfile and overlay directory serve complementary roles:

- **Overlay directory** (`~/.agent-plugin/state/hooks/{platform}/{plugin}.json`): Source of truth for individual plugin hook contributions. Used for merge/unmerge operations.
- **Lockfile** (`plugin-lock.json`): Cache of full plugin inventory, component registry, and file modifications. Used for status queries, clean uninstall, doctor diagnostics.

The lockfile tracks WHAT is installed. The overlays track HOW hooks are merged. Both are needed.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Use `atomically` for atomic writes of plugin-lock.json. Temp file + rename with retry. | Battle-tested pattern used by pnpm, npm (via write-file-atomic), and conf (via atomically). Prevents corruption from crashes. Retry handles transient FS errors. | Low |
| P0 | Implement integer `lockfileVersion` field with sequential migration pipeline. Start at version 1. | Matches npm's proven model. Simpler than semver for format evolution. Each migration transforms v(N) to v(N+1). Allows schema evolution without breaking existing lockfiles. | Low |
| P0 | Treat lockfile as cache with re-derivation from disk on corruption. Disk is source of truth. | Matches npm model. Eliminates single point of failure. If lockfile is missing or corrupt, scan installed plugins and reconstruct. | Medium |
| P1 | Implement layered corruption recovery: try parse, try backup, re-derive from disk. Log all recovery actions. | Defense in depth. Multiple recovery paths prevent data loss. Users see clear diagnostics on recovery. | Low |
| P1 | Use deterministic JSON serialization (sorted keys, 2-space indent) for git-friendly diffs. | Sorted keys ensure identical input produces identical output. Minimizes diff noise when plugins are added/removed. | Low |
| P1 | Include `_integrity` SHA-256 hash in lockfile for external modification detection. Exclude hash field from computation. | Detects manual edits, disk corruption, or tool version mismatches. Warn-only (do not block operations). | Low |
| P1 | Maintain `plugin-lock.json.bak` as backup of previous state before each write. | Fast corruption recovery. If atomic write fails AND temp file is lost, backup provides previous known-good state. | Low |
| P2 | Add PID-based lock check (warn, do not block) as safety net against concurrent CLI invocations. | Prevents accidental corruption from two terminal sessions running install simultaneously. Use `proper-lockfile` if full locking needed later. | Low |
| P2 | Implement `agent-plugin doctor` lockfile validation: schema check, integrity verify, orphan detection, overlay-lockfile consistency. | Diagnostic command that catches drift between lockfile state and actual disk state. Pairs with re-derivation for repair. | Medium |

## 8. Conclusion

**Verdict**: Proceed with implementation

**Confidence**: High

**Rationale**: The lockfile implementation pattern is well-established across the Node.js ecosystem. pnpm and npm both use `write-file-atomic` (or equivalent) for atomic writes, integer-based schema versioning for evolution, and treat the lockfile as a re-derivable cache. Our use case (plugin state tracking for a CLI tool) is simpler than package-lock.json, requiring fewer features. The recommended stack (atomically + integer versioning + layered recovery + sorted-key serialization) composes proven primitives without taking on unnecessary abstraction from conf or lowdb.

### User Impact

- **What changes for you**: Plugin state is tracked in a single `plugin-lock.json` file with corruption detection, atomic writes, and automatic recovery. Schema can evolve across tool versions without breaking existing installs.
- **Effort required**: Low-to-medium. Core write/read/migrate functions are straightforward. The re-derivation-from-disk scanner is the largest component.
- **Risk if ignored**: Without atomic writes, a crash during lockfile update could corrupt the file and orphan installed plugins (no record of what is installed). Without schema versioning, lockfile format changes require users to manually delete and recreate the lockfile.

## 9. Appendices

### Package Comparison Summary

| Package | Weekly Downloads | Use Case | Atomic Writes | Retry | Dependencies | Our Verdict |
|---|---|---|---|---|---|---|
| write-file-atomic | 45.8M | Atomic file writes | Yes (temp+rename) | No | 2 | Viable, battle-tested |
| atomically | 2.1M | Atomic file writes | Yes (temp+rename) | Yes (7.5s) | 0 third-party | Recommended |
| steno | 2.0M | High-frequency writes | Yes (temp+rename) | Yes (10 attempts) | 0 | Overkill for our write frequency |
| proper-lockfile | 3.9M | Inter-process locking | N/A (locking, not writing) | N/A | 2 | Not needed now; add if concurrent access required |
| conf | 1.3M | App config management | Yes (via atomically) | Yes | 6+ | Too opinionated for lockfile use |
| configstore | 13.9M | Config storage | Yes (via write-file-atomic) | No | 3 | Deprecated in favor of conf |
| lowdb | 914K | JSON database | Yes (via steno) | Yes | 1 (steno) | Database abstractions unnecessary |
| safe-stable-stringify | 12M+ | Deterministic JSON | N/A | N/A | 0 | Consider for serialization layer |

### Package Manager Lockfile Comparison

| Feature | npm | pnpm | Yarn Berry |
|---|---|---|---|
| Format | JSON | YAML | YAML |
| Atomic writes | write-file-atomic | write-file-atomic | @yarnpkg/fslib |
| Schema versioning | lockfileVersion integer (1,2,3) | LOCKFILE_VERSION float (9.0) | Not documented |
| Backward compat | v2 includes legacy fields | Major version check | N/A |
| Checksums | SHA-512 inline | SHA-512 inline | Cache zip hash (problematic) |
| Git conflict strategy | Regenerate on conflict | Branch-specific lockfiles | Manual resolve |
| Dual lockfile | No (hidden lockfile in node_modules) | Yes (wanted + current) | No |
| Corruption recovery | Regenerate from package.json | Regenerate from package.json | Regenerate from package.json |
| Concurrent access | Single-writer assumed | Single-writer assumed | Single-writer assumed |

### Academic Research Reference

"The Design Space of Lockfiles Across Package Managers" (Gamage et al., 2025, arxiv 2505.04834, published in Empirical Software Engineering). Key recommendations: (1) commit lockfiles universally, (2) prioritize developer experience with sensible defaults, (3) include only essential content, (4) generate lockfiles by default. The paper found no existing research on lockfile corruption handling, concurrent access, or schema versioning.

### Sources Consulted

- pnpm lockfile write source: <https://raw.githubusercontent.com/pnpm/pnpm/main/lockfile/fs/src/write.ts>
- pnpm lockfile system: <https://deepwiki.com/pnpm/pnpm/3.1-lockfile-system>
- pnpm git branch lockfiles: <https://pnpm.io/git_branch_lockfiles>
- npm package-lock.json docs: <https://docs.npmjs.com/cli/v11/configuring-npm/package-lock-json/>
- write-file-atomic: <https://github.com/npm/write-file-atomic>
- atomically: <https://github.com/fabiospampinato/atomically>
- steno: <https://github.com/typicode/steno>
- proper-lockfile: <https://github.com/moxystudio/node-proper-lockfile>
- conf: <https://github.com/sindresorhus/conf>
- lowdb: <https://github.com/typicode/lowdb>
- Lockfile Format Design and Tradeoffs: <https://nesbitt.io/2026/01/17/lockfile-format-design-and-tradeoffs.html>
- Understanding Lockfiles: <https://blog.shalvah.me/posts/understanding-lockfiles>
- The Design Space of Lockfiles (arxiv): <https://arxiv.org/html/2505.04834v1>
- Yarn Berry checksums: <https://github.com/yarnpkg/berry/issues/6068>
- npm lockfileVersion discussion: <https://www.abrahamberg.com/blog/npm-package-json-lock-version-1-or-2/>
- Node.js file locking: <https://blog.logrocket.com/understanding-node-js-file-locking/>
- safe-stable-stringify: <https://github.com/BridgeAR/safe-stable-stringify>

### Data Transparency

- **Found**: pnpm source code confirming write-file-atomic usage. Full implementation details for atomically, steno, proper-lockfile, and conf. npm lockfileVersion evolution history. Academic research on lockfile design across 7 package managers. Lockfile format design tradeoffs from practitioner blog.
- **Not Found**: Yarn Berry's exact atomic write mechanism for yarn.lock (uses fslib abstraction, internals undocumented). pnpm's concurrent access strategy (not documented, assumed single-writer). npm's exact code path from install command to lockfile write. Performance benchmarks comparing atomically vs write-file-atomic for single-write scenarios.

## Observations

- [decision] atomically selected as atomic write library for plugin-lock.json: retry on failure, 0 third-party deps, TypeScript native #architecture #lockfile
- [decision] Integer lockfileVersion with sequential migration pipeline selected over semver: matches npm model, simpler for format evolution #schema #lockfile
- [decision] Lockfile treated as re-derivable cache: disk state is source of truth, lockfile reconstructable via disk scan on corruption #reliability #lockfile
- [fact] pnpm and npm both use write-file-atomic (temp file + rename) for lockfile writes. This is the universal pattern across the Node.js ecosystem. #prior-art
- [fact] No npm package combines atomic JSON writes + schema validation + integer migration + integrity hashing + re-derivation. This must be composed from primitives. #gap-analysis
- [technique] Layered corruption recovery: try JSON.parse, try .bak backup, re-derive from disk scan. Each layer catches a different failure mode. #reliability #recovery
- [technique] Integrity hash (_integrity SHA-256) excludes itself from computation: serialize without field, hash, then inject hash into output. Detects external modification. #integrity
- [insight] Academic research (2025) found automatic lockfile generation is the single biggest predictor of adoption (Go 99.7% vs Gradle 0.9%). Our lockfile should be generated automatically on first install. #ux
- [risk] Single-writer assumption means concurrent CLI invocations could corrupt the lockfile. PID-based lock check provides a safety net without full locking complexity. #concurrency

## Relations

- extends [[ANALYSIS-010-hook-merge-unmerge-patterns]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
