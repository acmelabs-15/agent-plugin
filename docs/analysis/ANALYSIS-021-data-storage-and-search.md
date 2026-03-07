---
title: ANALYSIS-021-data-storage-and-search
type: analysis
permalink: analysis/analysis-021-data-storage-and-search
tags:
- analysis
- storage
- sqlite
- search
- orama
- drizzle
- bun
- agent-plugin
---

# ANALYSIS-021 Data Storage and Search

## 1. Objective and Scope

**Objective**: Does `@acmelabs-15/agent-plugin` need SQLite, an ORM, full-text search, or semantic search given the JSON lockfile decision in ADR-003? If so, which libraries should be used?

**Scope**: Evaluates 4 technology decisions from the original design spec: (1) SQLite via `bun:sqlite` or `drizzle-orm`, (2) full-text search via `@orama/orama`, (3) semantic search via `@huggingface/transformers`, (4) lighter alternatives. Assesses necessity before evaluating implementation.

**Excluded**: Lockfile implementation details (covered in ANALYSIS-011). MCP server architecture. Plugin manifest format.

## 2. Context

ADR-003 established a JSON lockfile (`plugin-lock.json`) as the state tracking mechanism for installed plugins. The lockfile stores:

- Plugin inventory (names, versions, sources)
- Component registry per plugin (skills, agents, hooks, MCP servers, commands)
- Hook overlay contributions per plugin per platform
- User-owned hook snapshot
- File modification history
- Integrity hash and schema version

The lockfile is a re-derivable cache. Disk state (installed plugin files) is the source of truth. ANALYSIS-011 selected `atomically` for atomic writes, integer versioning for schema evolution, and layered corruption recovery.

The original design spec proposed `drizzle-orm` + SQLite for data storage, `@orama/orama` for full-text search, and `@huggingface/transformers` for semantic search. This analysis asks: are any of these still needed?

## 3. Approach

**Methodology**: Web research on bun:sqlite capabilities, drizzle-orm Bun compatibility, Orama features, Hugging Face transformers.js model sizes, and lightweight search alternatives. Cross-referenced against the project's actual data requirements derived from ADR-001, ADR-002, and ADR-003.

**Tools Used**: WebSearch (12 queries), Brain MCP search (3 queries), file reading (4 documents: ADR-003, ANALYSIS-011, ANALYSIS-016, ANALYSIS-001).

**Limitations**: Could not run benchmarks for JSON parse performance at various plugin counts. Orama's exact current weekly download count was not directly available from search results. Bundle size numbers for some packages required inference from documentation claims rather than direct measurement.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| bun:sqlite is 3-6x faster than better-sqlite3, synchronous API, WAL mode, prepared statements, transactions | bun.com/docs, bun.com/reference | High |
| better-sqlite3 does NOT work with Bun; requires recompilation and has no plans to support Bun | GitHub issues #16050, #16049 | High |
| drizzle-orm v0.45.1 has 5M weekly downloads and native bun:sqlite support | npm, orm.drizzle.team | High |
| drizzle-orm has documented issues with concurrent statement execution in Bun | GitHub issues, drizzle docs | High |
| @orama/orama v3.1.18 supports full-text, vector, and hybrid search in under 2KB (minified) | npm, orama docs | High |
| Orama is compatible with Bun, Node.js, Deno, browsers, Cloudflare Workers | JSR listing, GitHub README | High |
| @huggingface/transformers requires 25-118 MB model downloads on first run | Hugging Face docs, GitHub discussions | High |
| FlexSearch has 912K weekly downloads, 13.6K GitHub stars; MiniSearch has 687K weekly downloads, 5.8K stars | npm-compare, npmtrends | High |
| JSON files above 100 MB face parsing performance issues (10s parse for 110 MB) | pl-rants.net, community research | High |
| SQLite advantage over JSON is partial reads/writes without full-file parse | SQLite forum, multiple comparisons | High |
| A typical plugin manager user installs 5-20 plugins | Inferred from Claude Code marketplace (36 curated, 9000+ total available) and npm ecosystem patterns | Medium |

### Facts (Verified)

- bun:sqlite ships built into the Bun runtime. Zero additional dependencies. Synchronous API inspired by better-sqlite3 but reimplemented natively against JavaScriptCore. Supports WAL mode via `PRAGMA journal_mode = WAL`, prepared statements, transactions, named/positional parameters, BLOB-to-Uint8Array conversion, bigint support.
- better-sqlite3 does not work with Bun. The native addon requires Node.js ABI compatibility that Bun does not provide. The better-sqlite3 team has no plans to add Bun support because bun:sqlite exists.
- drizzle-orm v0.45.1 has native bun:sqlite driver support with both async and sync APIs. 5M weekly downloads. TypeScript-first. Documented concurrent statement execution issues with Bun.
- @orama/orama v3.1.18 is dependency-free, under 2KB minified, supports full-text + vector + hybrid search, typo tolerance, filters, facets, stemming, geo-search. Compatible with all JavaScript runtimes including Bun.
- @huggingface/transformers requires downloading ML models on first use: 25 MB for smallest quantized models, 118 MB for multilingual models, 1 GB+ for translation models. Models are cached in node_modules.
- FlexSearch performs queries up to 1,000,000x faster than alternatives per its benchmarks. MiniSearch uses less than half the memory of Lunr. Both are pure JavaScript, zero dependencies.
- JSON parse/serialize is a full-file operation. SQLite allows row-level reads and writes. The crossover point depends on data size: JSON is faster for files under ~1 MB, SQLite wins for partial updates on larger datasets.

### Hypotheses (Unverified)

- A typical agent-plugin user will have 5-20 installed plugins, producing a lockfile of 5-50 KB. At this scale, JSON parse time is under 1ms.
- The plugin catalog (remote registry of available plugins) could grow to thousands of entries, but this data would come from a remote API, not local storage.
- AI assistants using the MCP interface already have semantic understanding and do not need client-side semantic search to find relevant plugins.

## 5. Results

### RQ1: Does This Project Need SQLite?

**Short answer: No, not at launch. The JSON lockfile is sufficient for the foreseeable data model.**

The lockfile data model consists of:

| Data | Typical Size | Access Pattern | JSON Sufficient? |
|---|---|---|---|
| Plugin inventory (5-20 plugins) | 2-10 KB | Full read on CLI start, full write on install/uninstall | Yes |
| Component registry (5-50 components) | 3-15 KB | Full read for list/search commands | Yes |
| Hook overlays (0-10 per platform) | 1-5 KB | Full read + merge on install, full write on uninstall | Yes |
| File modification history | 1-5 KB | Read on uninstall | Yes |
| User hook snapshot | 0.5-2 KB | Read on merge/unmerge | Yes |
| **Total lockfile** | **~10-50 KB** | Full parse per operation | **Yes** |

At 10-50 KB, `JSON.parse()` executes in under 1 ms. The lockfile never needs partial reads, relational queries, or concurrent access. Every CLI operation reads the full lockfile, modifies it in memory, and writes it back atomically. This is the exact access pattern that JSON excels at.

**Potential SQLite use cases that do NOT apply to this project:**

| Use Case | Why It Does Not Apply |
|---|---|
| Large dataset (100K+ rows) | Max ~50 plugins, ~200 components |
| Partial reads (query subset) | Full lockfile loaded every time |
| Concurrent writers | Single CLI invocation model |
| Relational queries (JOINs) | Flat plugin-to-components mapping |
| Transaction isolation | Atomic file write handles this |
| Audit log / append-only history | Not in scope for v1 |
| Usage analytics | Not in scope for v1 |

**When SQLite would become necessary (upgrade triggers):**

| Trigger | Threshold | Action |
|---|---|---|
| Plugin count exceeds 100 | Lockfile > 500 KB | Consider SQLite for component search index |
| Remote plugin catalog stored locally | 1000+ entries to search | SQLite for catalog index (separate from lockfile) |
| Usage analytics or audit log | Append-only writes | SQLite for analytics DB (separate from lockfile) |
| Plugin content indexing (SKILL.md bodies, agent definitions) | 500+ documents | SQLite FTS5 for content search |

None of these triggers are expected at launch or in the near-term roadmap.

### RQ2: If SQLite Is Needed Later, bun:sqlite vs drizzle-orm?

**Use bun:sqlite directly. Skip drizzle-orm.**

| Factor | bun:sqlite Direct | drizzle-orm + bun:sqlite |
|---|---|---|
| Dependencies added | 0 (built into Bun) | 1 (drizzle-orm, 5M downloads) |
| API complexity | Simple: `db.query()`, `db.run()`, `db.prepare()` | Schema definitions, query builders, migrations tooling |
| TypeScript support | Manual types for query results | Auto-inferred types from schema |
| Learning curve | Minimal (SQL + simple API) | Medium (ORM concepts, schema DSL) |
| Data model complexity | Low (3-5 tables max for this project) | ORMs shine with 10+ tables, relations |
| Concurrent statement bug | N/A (synchronous API) | Documented issues in Bun |
| Performance | 3-6x faster than better-sqlite3 | Adds ORM overhead on top of bun:sqlite |
| Bundle size impact | 0 KB | ~150 KB (drizzle-orm package) |

**Rationale**: An ORM adds value when the data model has complex relations, frequent schema changes, or many tables. This project's potential SQLite usage would be 2-3 simple tables (plugin catalog, search index, analytics). Direct SQL via bun:sqlite's `query()` method is simpler, faster, and zero-dependency.

### RQ3: Full-Text Search Assessment

**Start with simple string matching. Defer Orama until a clear scale trigger is hit.**

| Approach | Setup Effort | Dependencies | Search Quality | Scale Limit |
|---|---|---|---|---|
| `Array.filter()` + `String.includes()` | None | 0 | Exact substring match | ~100 items |
| `Array.filter()` + regex | None | 0 | Pattern matching | ~100 items |
| MiniSearch | Low | 1 (36 KB gzipped) | Fuzzy, prefix, boosting | ~10K items |
| FlexSearch | Low | 1 (6 KB gzipped) | Fastest, phonetic, partial | ~1M items |
| @orama/orama | Medium | 1 (under 2 KB gzipped) | Full-text + vector + hybrid | ~1M items |
| SQLite FTS5 | Medium | 0 (built into bun:sqlite) | Full-text with ranking | ~10M items |

**What a user actually searches for in a plugin manager:**

1. "List my installed plugins" -- `Array.filter()` on lockfile data
2. "Find a plugin named X" -- `String.includes()` or exact match on plugin names
3. "Find plugins that do X" -- Search plugin descriptions and tags
4. "Find a skill for code review" -- Search component names and descriptions

For use cases 1-2, simple filtering is sufficient. For use cases 3-4, the question is: how many plugins does a user have installed?

| Plugin Count | Search Method | Justification |
|---|---|---|
| 1-20 | `Array.filter()` + `String.includes()` | Sub-millisecond on any hardware |
| 20-100 | MiniSearch or FlexSearch in-memory index | Typo tolerance and prefix matching improve UX |
| 100-1000 | @orama/orama or SQLite FTS5 | Ranked results and faceted search become valuable |
| 1000+ | SQLite FTS5 | Persistent index avoids rebuild on every CLI invocation |

**Expected user behavior**: Most users will have 5-20 plugins installed. Power users might reach 50. Simple filtering handles this range with zero dependencies and zero complexity.

**Search library comparison (if search is needed later):**

| Library | Weekly Downloads | Bundle Size (gzipped) | Dependencies | Typo Tolerance | Bun Compatible | Best For |
|---|---|---|---|---|---|---|
| MiniSearch | 687K | ~36 KB | 0 | Yes | Yes (pure JS) | Small-medium datasets, simple API |
| FlexSearch | 912K | ~6 KB | 0 | Yes (phonetic) | Yes (pure JS) | Performance-critical, large datasets |
| @orama/orama | ~50K (estimated) | under 2 KB | 0 | Yes | Yes (confirmed) | Full-text + vector + hybrid search |
| Lunr | ~1M | ~8 KB | 0 | No | Yes (pure JS) | Simple, established, read-only index |
| Fuse.js | ~2.5M | ~5 KB | 0 | Yes (fuzzy) | Yes (pure JS) | Fuzzy search on small lists |
| SQLite FTS5 | N/A (built-in) | 0 KB | 0 (bun:sqlite) | No | Yes | Persistent index, large datasets |

**Recommendation**: If search is needed, start with MiniSearch (simple API, small, good TypeScript support) or FlexSearch (faster for larger datasets). Orama is the right choice only if vector/hybrid search becomes necessary. SQLite FTS5 is the right choice for persistent indexes across CLI invocations.

### RQ4: Semantic Search Assessment

**Verdict: Over-engineering. Remove from scope entirely.**

| Concern | Assessment |
|---|---|
| Model download size | 25-118 MB on first run. Unacceptable for a CLI tool where `npm install` should be fast. |
| Startup overhead | Model loading adds seconds to CLI startup. CLI tools must start in under 100ms. |
| Disk footprint | Models cached in node_modules. 25-118 MB per model. |
| Use case | "Find a plugin that does X" -- the AI assistant asking via MCP already understands semantics. |
| Alternative | Plugin descriptions and tags provide sufficient signal for keyword search. |
| Maintenance burden | Model version pinning, ONNX runtime compatibility, GPU/CPU detection. |
| User expectation | No CLI plugin manager (npm, pip, brew, cargo) offers semantic search. Users do not expect it. |

**The MCP server eliminates the need for semantic search.** The primary "search" user is an AI assistant operating via the MCP interface. That assistant already has semantic understanding built into its language model. It can interpret a user's intent ("find me a plugin for code review") and translate it into a keyword search against plugin names, descriptions, and tags. Client-side embedding generation duplicates capability that already exists in the AI assistant.

**Comparison with other plugin managers:**

| Tool | Search Capability | Semantic Search? |
|---|---|---|
| npm | Keyword search on name, description, keywords | No |
| pip | Keyword search on name, description | No |
| brew | Keyword search on name, description | No |
| cargo | Keyword search on name, description, keywords | No |
| Vercel skills | `find` command with keyword matching | No |
| Claude Code plugins | Marketplace search (keyword) | No |

Zero precedent for client-side semantic search in a CLI package/plugin manager.

### RQ5: better-sqlite3 Assessment

**Not viable. Eliminated.**

better-sqlite3 does not work with Bun and has no plans to add Bun support (GitHub issue #16050). Bun's built-in bun:sqlite replaces it entirely with a faster, zero-dependency implementation. No further analysis needed.

## 6. Discussion

### The "Start Simple" Path

The project should follow a layered architecture that starts with JSON and adds complexity only when specific triggers are hit:

```
Phase 1 (Launch): JSON lockfile only
  - plugin-lock.json for all state
  - Array.filter() for search
  - Zero additional dependencies

Phase 2 (Scale trigger: 50+ plugins or remote catalog):
  - Add MiniSearch or FlexSearch for in-memory search
  - 1 small dependency added
  - No architectural change

Phase 3 (Scale trigger: 1000+ searchable items or persistent index needed):
  - Add bun:sqlite for search index (separate from lockfile)
  - SQLite FTS5 for full-text search
  - Lockfile remains JSON (source of truth is still disk)
  - Zero additional dependencies (bun:sqlite is built-in)

Phase 4 (Scale trigger: vector/hybrid search demand):
  - Add Orama for vector search
  - Only if keyword search proves insufficient
  - 1 small dependency added
```

**Semantic search (@huggingface/transformers) is not on this path at any phase.** The MCP interface makes it unnecessary.

### Why the Design Spec Over-Specified Storage

The original design spec proposed SQLite + drizzle-orm + Orama + Hugging Face transformers before the lockfile design was finalized. ADR-003 then decided that a JSON lockfile handles all state tracking needs. The spec's storage layer was designed for a more complex data model that did not materialize.

The spec also assumed search would be a core differentiator. ANALYSIS-001 found that no competitor offers advanced search. This is likely because plugin managers deal with small datasets where simple search suffices. The absence of a feature in all competitors is signal, not an opportunity.

### Lockfile is the Right Abstraction

The lockfile pattern is proven across npm, pnpm, Yarn, Cargo, and Go modules. These tools manage orders of magnitude more packages than agent-plugin ever will, and they all use flat files (JSON or YAML), not databases, for their primary state. npm's `package-lock.json` can contain thousands of entries and still parses in milliseconds.

The lockfile also has a key advantage for this project: it is human-readable, git-diffable, and debuggable. A SQLite database is a binary file. When a user reports a bug with their plugin installation, asking them to share `plugin-lock.json` is trivial. Asking them to export a SQLite database is not.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Do NOT add SQLite at launch. Use JSON lockfile only. | Data model fits JSON. 10-50 KB lockfile parses in under 1ms. Zero additional dependencies. | None (remove from spec) |
| P0 | Do NOT add drizzle-orm. | ORM adds complexity for a 2-3 table data model. bun:sqlite direct is sufficient if SQLite is ever needed. | None (remove from spec) |
| P0 | Do NOT add @huggingface/transformers. | 25-118 MB model downloads, seconds of startup overhead, duplicates AI assistant's built-in semantic understanding. Zero precedent in CLI plugin managers. | None (remove from spec) |
| P0 | Use `Array.filter()` + `String.includes()` for search at launch. | 5-20 installed plugins. Sub-millisecond search. Zero dependencies. | Low |
| P1 | Design search interface as an abstraction layer. | Allows swapping search implementations (Array.filter -> MiniSearch -> SQLite FTS5) without changing the CLI or MCP API contract. | Low |
| P2 | Add MiniSearch or FlexSearch when plugin count exceeds 50 or search quality complaints emerge. | Typo tolerance and ranked results improve UX at moderate scale. Under 36 KB gzipped. | Low |
| P2 | Add bun:sqlite (direct, no ORM) when persistent search index or analytics are needed. | Built into Bun. Zero dependencies. 3-6x faster than better-sqlite3. | Medium |
| P3 | Add @orama/orama only if vector/hybrid search demand emerges. | Under 2 KB. Bun compatible. But no evidence this will be needed. | Low |

## 8. Conclusion

**Verdict**: Remove SQLite, drizzle-orm, and @huggingface/transformers from the design spec. Start with JSON lockfile + simple array filtering.

**Confidence**: High

**Rationale**: The JSON lockfile decided in ADR-003 handles all current state tracking needs. A typical user will have 5-20 plugins. At this scale, JSON parse is under 1ms and simple string filtering provides adequate search. Every comparable tool (npm, pip, brew, cargo, Vercel skills) uses flat files and keyword search. Client-side semantic search is over-engineering: the MCP interface delegates search to AI assistants that already have semantic understanding. The "start simple" path preserves the option to add SQLite (via bun:sqlite, built into the runtime) and advanced search (MiniSearch, Orama) at clearly defined scale triggers without architectural rework.

### User Impact

- **What changes for you**: 3 dependencies removed from the design spec (drizzle-orm, @orama/orama, @huggingface/transformers). Simpler architecture. Faster initial development. Lower maintenance burden. Search works out of the box with zero configuration.
- **Effort required**: None for removal. Low effort to implement search abstraction layer that allows future upgrades.
- **Risk if ignored**: Adding SQLite + ORM + full-text + semantic search at launch adds 4 dependencies, 200+ KB of bundle size, model download latency, and architectural complexity for a data model that fits in a 50 KB JSON file. This complexity slows development and creates maintenance burden with zero user benefit at launch scale.

## 9. Appendices

### Dependency Impact Summary

| Dependency | Spec Proposed | Recommendation | Weekly Downloads | Bundle Impact | Justification |
|---|---|---|---|---|---|
| drizzle-orm | Include | Remove | 5M | ~150 KB | ORM unnecessary for 2-3 table model. Use bun:sqlite direct if needed. |
| @orama/orama | Include | Defer (Phase 4) | ~50K est. | under 2 KB | Full-text + vector search unnecessary at launch scale. |
| @huggingface/transformers | Include | Remove permanently | 1.5M+ | 25-118 MB models | Duplicates AI assistant capability. Unacceptable CLI overhead. |
| better-sqlite3 | Not proposed | Not viable | 3M | N/A | Does not work with Bun. Use bun:sqlite. |
| MiniSearch | Not proposed | Defer (Phase 2) | 687K | ~36 KB | Add when search quality matters (50+ plugins). |
| FlexSearch | Not proposed | Defer (Phase 2) | 912K | ~6 KB | Alternative to MiniSearch for larger datasets. |
| bun:sqlite | Built-in | Defer (Phase 3) | N/A (built-in) | 0 KB | Available when persistent index or analytics needed. |

### Scale Threshold Decision Matrix

| Plugin Count | Lockfile Size | Parse Time | Search Method | Storage |
|---|---|---|---|---|
| 1-20 | 5-25 KB | under 1 ms | Array.filter() | JSON |
| 20-50 | 25-75 KB | under 1 ms | Array.filter() | JSON |
| 50-100 | 75-200 KB | 1-2 ms | MiniSearch/FlexSearch | JSON |
| 100-500 | 200 KB - 1 MB | 2-10 ms | SQLite FTS5 | JSON + SQLite index |
| 500+ | 1 MB+ | 10+ ms | SQLite FTS5 | Consider SQLite primary |

### bun:sqlite Quick Reference (for future use)

```typescript
import { Database } from "bun:sqlite";

// Open database (creates if not exists)
const db = new Database("search-index.sqlite");

// Enable WAL mode for performance
db.run("PRAGMA journal_mode = WAL");

// Create table
db.run(`CREATE VIRTUAL TABLE IF NOT EXISTS plugins_fts
  USING fts5(name, description, tags)`);

// Insert
const insert = db.prepare(
  "INSERT INTO plugins_fts VALUES ($name, $description, $tags)"
);
insert.run({ $name: "my-plugin", $description: "...", $tags: "..." });

// Full-text search
const results = db.query(
  "SELECT * FROM plugins_fts WHERE plugins_fts MATCH $query"
).all({ $query: "code review" });

// Transaction
const tx = db.transaction(() => {
  // multiple operations
});
tx();
```

### Sources Consulted

- Bun SQLite docs: https://bun.com/docs/runtime/sqlite
- Bun SQLite API reference: https://bun.com/reference/bun/sqlite
- Bun SQLite guide (2026): https://oneuptime.com/blog/post/2026-01-31-bun-sqlite/view
- better-sqlite3 Bun compatibility: https://github.com/oven-sh/bun/issues/16050
- better-sqlite3 Bun discussion: https://github.com/oven-sh/bun/discussions/16049
- Drizzle ORM Bun SQLite: https://orm.drizzle.team/docs/connect-bun-sqlite
- Drizzle ORM Bun guide: https://bun.com/docs/guides/ecosystem/drizzle
- Drizzle ORM npm: https://www.npmjs.com/package/drizzle-orm
- Drizzle vs Prisma 2026: https://www.bytebase.com/blog/drizzle-vs-prisma/
- Orama GitHub: https://github.com/oramasearch/orama
- Orama npm: https://www.npmjs.com/package/@orama/orama
- Orama docs: https://docs.orama.com/docs/orama-js
- Orama JSR (Bun compat): https://jsr.io/@orama/orama
- @huggingface/transformers npm: https://www.npmjs.com/package/@huggingface/transformers
- Transformers.js semantic search: https://deepwiki.com/huggingface/transformers.js-examples/3.3-semantic-search-and-embeddings
- Transformers.js v4 preview: https://huggingface.co/blog/transformersjs-v4
- SemanticFinder (transformers.js demo): https://github.com/do-me/SemanticFinder
- FlexSearch GitHub: https://github.com/nextapps-de/flexsearch
- MiniSearch npm: https://www.npmjs.com/package/minisearch
- Search library comparison: https://npm-compare.com/elasticlunr,flexsearch,fuse.js,minisearch
- npm trends (search libraries): https://npmtrends.com/flexsearch-vs-fuzzysearch-vs-minisearch
- JSON vs SQLite performance: https://pl-rants.net/posts/when-not-json/
- SQLite flat files forum: https://sqlite.org/forum/forumpost/3d7be1ad3d

### Data Transparency

- **Found**: bun:sqlite full API docs and performance benchmarks. better-sqlite3 Bun incompatibility confirmed. drizzle-orm version, downloads, and Bun driver status. Orama feature set, bundle size claim (under 2KB), and runtime compatibility. Hugging Face model download sizes. FlexSearch and MiniSearch download counts and feature comparison. JSON vs SQLite crossover thresholds from multiple sources.
- **Not Found**: Orama exact weekly download count (npm page was not fetched directly). Exact bundle sizes for FlexSearch and MiniSearch via bundlephobia (inferred from documentation). Real-world benchmarks of JSON.parse at various plugin counts on Bun specifically (inferred from general JavaScript benchmarks). Whether any AI coding assistant's MCP implementation uses semantic search internally for plugin discovery.

## Observations

- [decision] SQLite removed from launch scope: JSON lockfile at 10-50 KB handles all state tracking needs, parsing in under 1ms #storage #simplification
- [decision] drizzle-orm removed from scope: ORM adds unnecessary complexity for a 2-3 table data model; bun:sqlite direct queries suffice if SQLite is ever needed #orm #simplification
- [decision] @huggingface/transformers removed permanently: 25-118 MB model downloads and seconds of startup overhead are unacceptable for CLI tools; MCP interface delegates semantic understanding to AI assistants #search #over-engineering
- [decision] Simple Array.filter() + String.includes() adopted for search at launch: sufficient for 5-20 installed plugins with sub-millisecond performance #search #simplification
- [fact] bun:sqlite is built into Bun runtime, 3-6x faster than better-sqlite3, supports WAL mode, prepared statements, and transactions with zero dependencies #bun #sqlite
- [fact] better-sqlite3 does not work with Bun and has no plans to add support; bun:sqlite replaces it entirely #bun #compatibility
- [fact] No CLI plugin manager (npm, pip, brew, cargo, Vercel skills, Claude Code) offers semantic search or SQLite-based storage for plugin state #prior-art
- [risk] Premature addition of SQLite + ORM + search libraries adds 4 dependencies, 200+ KB bundle size, and architectural complexity for zero user benefit at launch scale #yagni
- [insight] MCP interface eliminates need for client-side semantic search: AI assistants already have built-in semantic understanding and can translate user intent into keyword queries #architecture #mcp
- [technique] Layered upgrade path preserves options: JSON -> MiniSearch (50+ plugins) -> bun:sqlite FTS5 (1000+ items) -> Orama (vector search demand) without architectural rework #architecture #scalability

## Relations

- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ANALYSIS-011-lockfile-management-patterns]]
- relates_to [[ANALYSIS-016-bun-runtime-assessment]]
- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
