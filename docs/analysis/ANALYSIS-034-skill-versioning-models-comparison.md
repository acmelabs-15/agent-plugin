---
title: ANALYSIS-034 Skill Versioning Models Comparison
type: analysis
permalink: analysis/analysis-034-skill-versioning-models-comparison-1
tags:
- versioning
- plugin-manager
- tanstack-intent
- vercel-skills
- claude-code-plugins
- installation
- comparison
---

# ANALYSIS-034 Skill Versioning Models Comparison

## 1. Objective and Scope

**Objective**: Compare the installation, versioning, upgrade, and lockfile mechanics of TanStack Intent, Vercel Skills CLI, and Claude Code Plugins. Provide evidence for design decisions in `@acmelabs-15/agent-plugin`.

**Scope**: Installation mechanism, version control, upgrade mechanism, lockfile strategy, and what "controls" the version in each system. Excludes runtime behavior and agent integration details.

## 2. Context

The `@acmelabs-15/agent-plugin` project needs a plugin/skill management system. Three production systems exist as reference implementations. Each takes a fundamentally different approach to distribution, versioning, and state tracking.

## 3. Approach

**Methodology**: Web research of official documentation, blog posts, source code analysis via DeepWiki, and GitHub repository inspection.
**Tools Used**: WebSearch, WebFetch
**Limitations**: TanStack Intent is in alpha. Its internal discovery mechanism (how it detects intent-enabled packages in node_modules) is not fully documented publicly. Some details are inferred from blog posts and X (Twitter) announcements rather than API docs.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| TanStack Intent skills ship inside npm packages | TanStack blog, X announcement | High |
| TanStack Intent scans node_modules for discovery | TanStack blog, X announcement | High |
| TanStack Intent uses standard package manager lockfile | Inferred (no custom lockfile documented) | Medium |
| Vercel Skills uses symlinks by default, copy as fallback | vercel-labs/skills GitHub, DeepWiki | High |
| Vercel Skills tracks global installs in ~/.agents/.skill-lock.json | DeepWiki analysis of vercel-labs/skills source | High |
| Vercel Skills uses GitHub tree SHAs for version detection | DeepWiki analysis | High |
| Claude Code plugins use marketplace.json with multiple source types | Official docs at code.claude.com | High |
| Claude Code plugins support git SHA pinning | Official docs at code.claude.com | High |
| Claude Code plugins are cached at ~/.claude/plugins/cache | Official docs at code.claude.com | High |
| Claude Code plugins support npm source with version ranges | Official docs at code.claude.com | High |

---

## 5. Results

### System 1: TanStack Intent

**Status**: Alpha (as of early 2026)

**Installation Mechanism**: npm dependency model
- Skills are files shipped INSIDE an npm package. A library maintainer runs `npx @tanstack/intent scaffold` to generate skill files from their docs, then publishes them as part of their normal npm package.
- A consumer installs the library via their package manager (`npm install @tanstack/query`, `bun add @tanstack/router`). The skills arrive in node_modules alongside the library code.
- The consumer then runs `npx @tanstack/intent install`, which scans node_modules for all intent-enabled packages, discovers their skills, and wires them into agent configuration files (CLAUDE.md, .cursorrules, etc.).

**Version Control**: Package manager native
- Version is the npm package version. There is no separate skill version.
- When you run `npm update @tanstack/query`, the skills update with the library code.
- No custom manifest for tracking skill versions. The package.json and package-lock.json/bun.lockb ARE the version tracking.

**Upgrade Mechanism**: Standard package manager
- `npm update`, `bun update`, `yarn upgrade` update the dependency.
- Re-run `npx @tanstack/intent install` to rewire updated skills into agent configs.
- No custom upgrade command for skills themselves.

**Lockfile Strategy**: Standard package manager lockfile
- package-lock.json, bun.lockb, or yarn.lock. No separate skill lockfile.
- Skills version is fully coupled to the npm package version.

**What Controls the Version**: The npm package version in package.json. The library maintainer owns the skill content. Skills travel with the library.

**Discovery Mechanism**: Scans node_modules
- The exact detection convention is not publicly documented. Likely looks for a known directory or package.json field within each dependency.
- Each skill declares `metadata.sources` pointing to the source documentation it was generated from.
- `npx @tanstack/intent stale` flags skills whose source docs have changed since generation.

**Key CLI Commands**:
- `npx @tanstack/intent scaffold` - Generate skill drafts from docs
- `npx @tanstack/intent validate` - Check skill well-formedness
- `npx @tanstack/intent install` - Wire discovered skills into agent configs
- `npx @tanstack/intent list` - Show available skills (supports --json)
- `npx @tanstack/intent stale` - Check for version drift vs source docs
- `npx @tanstack/intent feedback` - Structured user issue reporting

---

### System 2: Vercel Skills CLI (`npx skills`)

**Status**: Production (as of early 2026)

**Installation Mechanism**: Git clone + symlink/copy
- Skills are NOT npm packages. They are directories in git repositories containing a SKILL.md file.
- `npx skills add owner/repo` clones the repo to a temp directory, then copies skill directories to a canonical location.
- Supports GitHub shorthand, full GitHub URLs, GitLab URLs, any git URL, and local paths.
- Default: creates symlinks from agent-specific directories to a canonical copy in `.agents/skills/`.
- Fallback: `--copy` flag creates independent copies for environments without symlink support.

**Version Control**: GitHub tree SHA hashing
- No semver enforcement. The SKILL.md frontmatter has an optional `metadata` field where you CAN put a version, but it is not used for update detection.
- Version tracking uses GitHub tree SHAs (content-addressable hashing of the skill folder).
- The lockfile stores these SHAs and compares against remote on `npx skills check`.

**Upgrade Mechanism**: Custom CLI command
- `npx skills check` - Detects available updates by comparing local SHAs against remote.
- `npx skills update` - Re-runs `npx skills add` for each outdated skill via `spawnSync`.
- Update tracking only works for globally-installed skills. Project-scoped skills are version-controlled via git (committed to repo).

**Lockfile Strategy**: Custom lockfile (`skill-lock.json`)
- Global: `~/.agents/.skill-lock.json` tracks globally installed skills.
- Project: `skills-lock.json` (checked into version control) for project-scoped installs.
- Each entry contains: skill name, source repo, GitHub tree SHA, installation metadata.
- Project-scoped skills rely on git version control rather than the lockfile for tracking.

**What Controls the Version**: The content of the skill directory in the source git repository. There is no declarative version number that gates updates. Any change to the skill folder (detected via tree SHA) triggers an available update.

**Storage Layout**:
- Project scope canonical: `./.agents/skills/<skill-name>/`
- Global scope canonical: `~/.agents/skills/<skill-name>/` or `~/.config/agents/skills/`
- Agent-specific symlinks: `./.claude/skills/`, `./.cursor/skills/`, `~/.claude/skills/`, etc.

**SKILL.md Format**:
- YAML frontmatter: `name` (required, 1-64 chars, lowercase with hyphens), `description` (required, 1-1024 chars)
- Optional: `metadata` (arbitrary key/value), `allowed-tools`, `license`, `compatibility`
- Markdown body: Agent instructions

---

### System 3: Claude Code Plugins

**Status**: Production (since October 2025, v1.0.33+)

**Installation Mechanism**: Marketplace-mediated with multiple source types
- Plugins are directories with a defined structure: `.claude-plugin/plugin.json` manifest, plus `skills/`, `commands/`, `agents/`, `hooks/`, `.mcp.json`, `.lsp.json`.
- Distribution through "marketplaces" - catalogs defined in `.claude-plugin/marketplace.json`.
- Plugin sources in marketplace entries support 6 types:
  1. Relative path (`"./plugins/my-plugin"`) - local within marketplace repo
  2. GitHub (`{ source: "github", repo: "owner/repo", ref?, sha? }`)
  3. Git URL (`{ source: "url", url: "https://...git", ref?, sha? }`)
  4. Git subdirectory (`{ source: "git-subdir", url, path, ref?, sha? }`) - sparse clone for monorepos
  5. npm (`{ source: "npm", package: "@scope/name", version?, registry? }`)
  6. pip (`{ source: "pip", package, version?, registry? }`)
- On install, plugins are copied to `~/.claude/plugins/cache` (never used in-place for security).

**Version Control**: Semver in plugin.json + marketplace.json
- `plugin.json` has a `version` field following semver (MAJOR.MINOR.PATCH).
- `marketplace.json` plugin entries can also declare a `version`.
- When both exist, `plugin.json` takes priority (silently overrides marketplace version).
- Version determines cache paths and update detection.
- Git sources support optional SHA pinning (full 40-character commit SHA).
- npm sources support version ranges (`^2.0.0`, `~1.5.0`, exact `2.1.0`).

**Upgrade Mechanism**: Auto-update + manual CLI
- Official Anthropic marketplaces auto-update by default (at startup).
- Third-party marketplaces: auto-update disabled by default, togglable per marketplace.
- `claude plugin update <plugin>` for manual updates.
- `/plugin marketplace update <name>` refreshes marketplace catalog.
- `/reload-plugins` applies changes without restart.
- `DISABLE_AUTOUPDATER=1` disables all auto-updates; `FORCE_AUTOUPDATE_PLUGINS=true` overrides to keep plugin updates.

**Lockfile Strategy**: Settings-based tracking (no separate lockfile)
- Installed plugins tracked in Claude Code settings files by scope:
  - User: `~/.claude/settings.json` (enabledPlugins)
  - Project: `.claude/settings.json` (shared via version control)
  - Local: `.claude/settings.local.json` (gitignored)
  - Managed: admin-controlled read-only settings
- No dedicated lockfile. The settings.json files and the plugin cache serve as the state record.
- Plugin cache at `~/.claude/plugins/cache` holds the actual plugin files.

**What Controls the Version**: The `version` field in `plugin.json` (or `marketplace.json` if not in plugin.json). For git sources, optional SHA pinning provides exact reproducibility. For npm sources, the npm version/range controls resolution.

**Marketplace Architecture**:
- Marketplace = catalog of plugins (`marketplace.json`)
- Marketplace sources: GitHub repos, git URLs, local paths, remote URLs
- Plugin sources: relative paths, GitHub, git URL, git-subdir, npm, pip
- Scope system: user, project, local, managed
- `strict` mode controls whether plugin.json or marketplace.json is authoritative for component definitions
- Managed settings can restrict allowed marketplaces via `strictKnownMarketplaces`

---

## 6. Discussion

### Comparison Matrix

| Dimension | TanStack Intent | Vercel Skills CLI | Claude Code Plugins |
|---|---|---|---|
| **Distribution** | Inside npm packages | Git repositories | Git repos, npm, pip, local |
| **Installation** | npm install + intent install | npx skills add (clone+symlink) | Marketplace-mediated (copy to cache) |
| **Version identifier** | npm package semver | GitHub tree SHA (content hash) | Semver in plugin.json |
| **Version granularity** | Coupled to library version | Any git commit | Semver + optional SHA pin |
| **Lockfile** | Standard package manager | Custom skill-lock.json | Settings.json (no lockfile) |
| **Upgrade command** | npm update + intent install | npx skills update | claude plugin update / auto-update |
| **Auto-update** | No (manual npm update) | No (manual check/update) | Yes (configurable per marketplace) |
| **Reproducibility** | Package manager lockfile | Git tree SHA in lockfile | SHA pinning (optional) |
| **Scope model** | Project only (node_modules) | Project + global | User + project + local + managed |
| **Registry/catalog** | npmjs.com | skills.sh (install telemetry) | Marketplace repos |
| **Who controls content** | Library maintainer | Skill author | Plugin author or marketplace operator |

### Three Distinct Philosophies

**TanStack Intent: "Skills are a feature of the library"**
Skills version with the library. No separate lifecycle. The library author owns both the code and the skill content. This eliminates version drift between library and skill but couples them tightly. You cannot update a skill without updating the library.

**Vercel Skills: "Skills are documents in git repos"**
Content-addressable versioning via tree SHAs. No semver enforcement. Any change to the skill folder is an update. This is simple and transparent but provides no semantic signal about breaking vs non-breaking changes. There is no version negotiation or range matching.

**Claude Code Plugins: "Plugins are packages distributed through catalogs"**
Full semver support with multiple source types. The most feature-rich system with marketplace hierarchies, scope management, auto-update, SHA pinning, npm/pip integration, and managed enterprise controls. Also the most complex.

### Key Tradeoffs for @acmelabs-15/agent-plugin

1. **Coupled vs decoupled versioning**: TanStack couples skill version to library version. Vercel and Claude Code decouple them. Decoupled versioning allows skill fixes without library updates but requires separate version tracking.

2. **Content-hash vs semver**: Vercel uses content hashing (any change = update). Claude Code uses semver (meaningful version bumps). Semver communicates intent (breaking vs patch) but requires author discipline. Content hashing is automatic but noisy.

3. **Standard vs custom lockfile**: TanStack reuses package-lock.json. Vercel has skill-lock.json. Claude Code uses settings.json. A custom lockfile adds complexity but enables skill-specific tracking without polluting the package manager.

4. **Symlink vs copy**: Vercel symlinks for efficiency and single source of truth. Claude Code copies for security isolation. Symlinks save disk but create external dependencies. Copies are self-contained but duplicate storage.

5. **Auto-update**: Only Claude Code supports auto-update out of the box. This is valuable for enterprise but adds complexity and trust requirements.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Support semver in plugin manifest | Communicates breaking vs non-breaking changes. Content-hash alone (Vercel model) provides no semantic signal. | Low |
| P0 | Custom lockfile (not settings.json) | Explicit state tracking. Easier to debug, diff, and version control than embedding in settings. TanStack's approach of reusing package-lock.json only works if plugins ARE npm packages. | Medium |
| P1 | Support multiple source types | Claude Code's model (git, npm, local) provides the most flexibility. At minimum support git repos and local paths. | High |
| P1 | Copy-to-cache over symlink | Security isolation and portability. Vercel's symlink model is elegant but fragile. Claude Code's copy model is safer. | Medium |
| P2 | Optional SHA pinning for reproducibility | Claude Code demonstrates this well. Pin to exact git commit when reproducibility matters. | Medium |
| P2 | Auto-update as opt-in feature | Follow Claude Code's pattern: enabled for official sources, disabled for third-party. | High |

## 8. Conclusion

**Verdict**: Proceed with hybrid model
**Confidence**: High
**Rationale**: Each system optimizes for a different use case. TanStack targets library authors (npm-native). Vercel targets the cross-agent skill ecosystem (git-native). Claude Code targets enterprise plugin management (marketplace-native). The `@acmelabs-15/agent-plugin` project should take Claude Code's multi-source + semver approach as the primary reference, with Vercel's lockfile simplicity, and skip TanStack's tightly-coupled model.

### User Impact

- **What changes for you**: Clear versioning model, reproducible installs, multiple source support
- **Effort required**: Medium (custom lockfile + multi-source resolution)
- **Risk if ignored**: Ad-hoc versioning that breaks reproducibility and makes debugging difficult

## 9. Appendices

### Sources Consulted

- [TanStack Intent Blog: From Docs to Agents](https://tanstack.com/blog/from-docs-to-agents)
- [TanStack Intent Landing Page](https://tanstack.com/intent/latest)
- [TanStack Intent GitHub Repository](https://github.com/TanStack/intent)
- [TanStack CLI GitHub Repository](https://github.com/TanStack/cli)
- [TanStack X/Twitter Announcement](https://x.com/tan_stack/status/2029973163455766769)
- [Vercel Skills GitHub Repository](https://github.com/vercel-labs/skills)
- [Vercel Agent Skills Documentation](https://vercel.com/docs/agent-resources/skills)
- [Vercel Skills FAQ](https://vercel.com/blog/agent-skills-explained-an-faq)
- [Vercel Skills Changelog Announcement](https://vercel.com/changelog/introducing-skills-the-open-agent-skills-ecosystem)
- [Vercel Agent Skills KB Guide](https://vercel.com/kb/guide/agent-skills-creating-installing-and-sharing-reusable-agent-context)
- [DeepWiki: vercel-labs/skills Analysis](https://deepwiki.com/vercel-labs/skills)
- [Claude Code Plugins Reference](https://code.claude.com/docs/en/plugins-reference)
- [Claude Code Discover Plugins](https://code.claude.com/docs/en/discover-plugins)
- [Claude Code Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- [Claude Code Plugins README on GitHub](https://github.com/anthropics/claude-code/blob/main/plugins/README.md)
- [skills.sh Directory](https://skills.sh)

### Data Transparency

- **Found**: Complete installation, versioning, lockfile, and upgrade mechanics for Vercel Skills and Claude Code Plugins. High-level model for TanStack Intent.
- **Not Found**: TanStack Intent's exact package.json field or directory convention for marking a package as "intent-enabled." The internal discovery algorithm is not publicly documented (product is in alpha). Also not found: Vercel Skills' exact SkillLockEntry TypeScript interface (inferred from DeepWiki analysis).

## Observations

- [fact] TanStack Intent ships skills inside npm packages; version = npm package version #versioning #tanstack
- [fact] Vercel Skills CLI uses GitHub tree SHAs for update detection, stored in skill-lock.json #versioning #vercel
- [fact] Claude Code plugins support 6 source types: relative path, GitHub, git URL, git-subdir, npm, pip #distribution #claude-code
- [fact] Claude Code plugin.json version field takes priority over marketplace.json version #versioning #claude-code
- [decision] Recommend semver in manifest + custom lockfile + multi-source support for agent-plugin #architecture
- [insight] Three philosophies: skills-as-library-feature (TanStack), skills-as-documents (Vercel), plugins-as-packages (Claude Code) #patterns
- [risk] TanStack's tightly-coupled model prevents skill updates without library updates #tradeoff
- [insight] Content-hash versioning (Vercel) is automatic but provides no semantic signal about breaking changes #tradeoff

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[TASK-002 Placement Strategies]]
- relates_to [[REQ-002 Placement Strategies]]
- relates_to [[ANALYSIS-021 Data Storage and Search]]