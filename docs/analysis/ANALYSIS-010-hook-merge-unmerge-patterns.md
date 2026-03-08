---
title: ANALYSIS-010-hook-merge-unmerge-patterns
type: note
permalink: analysis/analysis-010-hook-merge-unmerge-patterns
tags:
- hooks
- merging
- configuration
- analysis
- agent-plugin
---

# ANALYSIS-010 Hook Merge/Unmerge Patterns

## 1. Objective and Scope

**Objective**: How should @acmelabs-15/agent-plugin merge plugin hooks into platform configuration on install, and cleanly unmerge them on uninstall, across 7 target platforms with different hook formats?

**Scope**: Covers 5 research questions: (1) community best practices for configuration hook merging in CLI tools, (2) existing packages/libraries for safe configuration merging with provenance tracking, (3) architectural patterns (marker-based, overlay, diff/patch, manifest tracking, lockfile), (4) robustness concerns (manual edits, force removal, conflicts, atomicity, rollback), (5) real-world examples from 10+ production systems.

## 2. Context

@acmelabs-15/agent-plugin is a cross-platform AI agent plugin manager targeting 7 platforms: Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, and Windsurf. When a plugin is installed, its hooks must be merged with existing hooks. When uninstalled, its hooks must be cleanly removed without disturbing other plugins' hooks or user-owned hooks.

Prior decisions established:

- ADR-003: Hook event collisions are additive (both hooks run), not conflicting. Users can configure execution order.
- ADR-003: Rename tracking via state store maps original-name to installed-name for correct update/removal.
- ANALYSIS-009: Managed section markers (BEGIN/END) adopted for instruction files. Tool-owned templates for content generation. Hybrid strategy: managed sections for shared files, dedicated files for per-file platforms.

The primary challenge is Claude Code's structured JSON hook system. Hooks live in `.claude/settings.json` under the `hooks` key as nested JSON (event -> matcher groups -> handler arrays). Other platforms have simpler or no native hook systems.

## 3. Approach

**Methodology**: Web research across 40+ sources covering package managers, plugin systems, configuration management tools, overlay filesystems, and AI coding platform documentation. Cross-referenced with existing project analyses and real-world bug reports.

**Tools Used**: WebSearch (14 queries), WebFetch (6 pages), Brain MCP search, file reads of existing project analyses (ADR-003, ANALYSIS-005, ANALYSIS-009).

**Limitations**: Claude Code's hook merge implementation is closed source. Only external behavior and one deduplication bug report provide insight into internals. No cross-platform AI agent plugin manager exists as a reference implementation.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| Claude Code merges plugin hooks with user/project hooks at runtime; all matching hooks run in parallel | Claude Code hooks docs | High |
| Claude Code deduplicates hooks by raw command template string BEFORE expanding ${CLAUDE_PLUGIN_ROOT}, causing multi-plugin bugs (issue #29724) | GitHub issue #29724 | High |
| Gemini CLI hooks merge from .gemini/settings.json, user-level, system-level configs, and gemini-extension.json; project settings override user/system | Gemini CLI docs | High |
| systemd drop-in overrides distinguish accumulating parameters (additive) from replacing parameters (override), with empty-value reset mechanism | DEV Community article | High |
| ESLint flat config merges configuration objects in order, later objects override previous for conflicts; arrays are NOT merged, objects ARE merged recursively | ESLint docs | High |
| Docker Compose merges files in order: single-value options replace, maps merge recursively, lists replace entirely | Docker docs | High |
| Kubernetes Kustomize uses strategic merge patches: replaces scalars, merges maps, uses merge keys for lists | Kubernetes docs | High |
| webpack-merge provides per-field strategies: append, prepend, replace, merge, plus unique strategy for singleton enforcement | webpack-merge npm/GitHub | High |
| json-merger provides $import, $remove, $replace, $merge, $concat operations as inline JSON directives | json-merger npm/GitHub | High |
| NixOS module system uses numeric priority markers for merge ordering; last overlay wins for composition | NixOS docs | High |
| Git config uses layered include files with conditional includeIf; lower-level configs override higher-level | Git docs | High |
| Terraform override files merge required_providers element-by-element; each override replaces the corresponding element entirely | Terraform docs | High |
| JSON Merge Patch (RFC 7396) uses partial JSON for changes; null means delete; cannot set a value to null | IETF RFC 7396 | High |
| JSON Patch (RFC 6902) uses explicit operation arrays (add, remove, replace, move, copy) for atomic changes | IETF RFC 6902 | High |
| deepmerge npm package creates new objects (immutable merge), handles nested objects, allows custom merge functions | npm | High |

### Facts (Verified)

- [fact] Claude Code's hook system supports 17 event types. Hooks from multiple sources (user, project, plugin, skill, agent frontmatter) merge and run in parallel for the same event.
- [fact] Claude Code has a known deduplication bug (#29724) where hooks from different plugins with the same relative command path are collapsed because dedup occurs before ${CLAUDE_PLUGIN_ROOT} expansion. Fixed in v2.1.53+.
- [fact] No npm package exists that combines JSON configuration merging WITH provenance tracking (knowing which plugin contributed which entry). This is a custom requirement.
- [fact] The managed section marker pattern (ANALYSIS-009) applies to instruction files but NOT to structured JSON/YAML configuration files where hooks live.
- [fact] Gemini CLI supports `gemini hooks install/uninstall <package>` commands with npm integration and automatic extension bundling, making it the closest reference implementation for hook lifecycle management.

### Hypotheses (Unverified)

- [hypothesis] A manifest/lockfile approach (tracking what each plugin contributed in a separate state file) is more robust than in-place JSON modification for structured configuration files.
- [hypothesis] Overlay/layer systems that compute merged results at runtime avoid the merge/unmerge problem entirely but require runtime control not available on all platforms.
- [hypothesis] For non-Claude-Code platforms without native hook systems, hooks can be shimmed through instruction file directives rather than configuration file merging.

## 5. Results

### RQ1: Community Best Practices for Configuration Hook Merging

Three dominant patterns exist across the ecosystem:

**Pattern 1: Additive Array Merging (Claude Code, Gemini CLI)**
Multiple hook definitions for the same event are concatenated into an array. All hooks run (in parallel or sequence). No conflict resolution needed because hooks are additive. This is the cleanest model for hook-type configuration.

Key properties:

- Event-level arrays are concatenated, not replaced
- Each hook handler retains identity through its command string or unique identifier
- Deduplication prevents exact duplicate hooks from running twice
- The "strictest wins" policy means if ANY hook blocks, the action is blocked

**Pattern 2: Layered Override (systemd, Docker Compose, ESLint, Git config)**
Configuration from multiple sources is merged with a defined precedence. Later/more-specific sources override earlier/more-general ones. Some fields accumulate (arrays, lists) while others replace (scalars).

Key properties:

- Clear precedence hierarchy (system < user < project < override)
- Per-field merge semantics (some fields accumulate, some replace)
- systemd's "empty value reset" pattern: assign empty to clear inherited values before setting new ones
- Docker Compose: single-value replaces, maps merge, lists replace entirely

**Pattern 3: Overlay/Layer Systems (Kustomize, NixOS, Terraform)**
Each contributor provides a partial configuration (patch/overlay). The system computes the final merged result from all layers. Layers are ordered; later layers can override or extend earlier ones.

Key properties:

- Base + overlay model: clean separation of concerns
- Strategic merge (Kustomize): intelligent field-level merge using schema knowledge
- Priority-based merge (NixOS): numeric priorities determine which value wins
- Override files (Terraform): each override replaces corresponding elements entirely

**Verdict for our use case**: Pattern 1 (Additive Array Merging) matches our requirements best. ADR-003 already decided that hook event collisions are additive. Hooks should concatenate, not override.

### RQ2: Existing Packages and Libraries

#### Deep Merge Libraries

| Package | Weekly Downloads | Approach | Provenance Tracking | Unmerge Support |
|---|---|---|---|---|
| deepmerge | 30M+ | Creates new merged object, custom merge functions per field | No | No |
| lodash.merge | 15M+ | Recursive deep merge, mutates destination | No | No |
| webpack-merge | 8M+ | Per-field strategies (append/prepend/replace/merge/unique) | No | No |
| json-merger | ~5K | Inline directives ($import, $remove, $replace, $merge) | No | Yes ($remove) |

#### JSON Patch Standards

| Standard | Approach | Reversible | Provenance |
|---|---|---|---|
| RFC 6902 (JSON Patch) | Explicit operations array (add, remove, replace, move, copy) | Yes (operations are invertible) | No (but operations can be tagged) |
| RFC 7396 (JSON Merge Patch) | Partial JSON document, null = delete | No (destructive, lossy for null values) | No |

#### Configuration Management

| Tool | Approach | Provenance | Unmerge |
|---|---|---|---|
| Ansible blockinfile | Marker-delimited sections with BEGIN/END | Yes (marker identifies owner) | Yes (state=absent removes block) |
| WordPress insert_with_markers | Paired BEGIN/END markers with file locking | Yes (marker parameter) | Yes (remove block between markers) |
| Kustomize | Strategic merge patches from overlay directories | Yes (overlay directory identifies source) | Yes (remove overlay) |
| systemd drop-ins | Directory-based overlay files with naming-based precedence | Yes (filename identifies source) | Yes (delete drop-in file) |

**Key finding**: No existing library combines JSON deep merging with provenance tracking and clean unmerge. This is a gap we must fill with a custom solution.

### RQ3: Architectural Patterns Analysis

#### Pattern A: Marker-Based Sections

**How it works**: Delimit plugin-contributed content with markers. For structured JSON, this means embedding provenance metadata within the JSON structure itself.

**Applicability**: Works well for text files (ANALYSIS-009 already adopted this for instruction files). Does NOT work for JSON configuration files because JSON does not support comments. Would require adding metadata fields to the JSON structure itself.

**For Claude Code hooks, a marker-equivalent approach would be:**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "path/to/script.sh",
            "_plugin": "my-plugin",
            "_version": "1.0.0"
          }
        ]
      }
    ]
  }
}
```

**Problem**: Claude Code will reject or ignore unknown fields like `_plugin`. We cannot add provenance metadata to the target platform's configuration format.

**Verdict**: [FAIL] for JSON config files. [PASS] for instruction files (already adopted in ANALYSIS-009).

#### Pattern B: Overlay/Layer System (Recommended)

**How it works**: Each plugin's hooks are stored in a separate file. The system computes the merged result when needed.

**Two sub-variants:**

**B1: Runtime Overlay (Claude Code native model)**
Claude Code already does this. Plugin hooks live in `hooks/hooks.json` inside each plugin. The runtime loads and merges them at session start. We do not need to write hooks into settings.json for Claude Code plugins installed through its native plugin system.

**B2: Static Overlay with Computed Merge**
For platforms where we must write merged configuration to a single file:

1. Each plugin's hook contribution is stored in our state directory as a separate JSON file: `~/.agent-plugin/state/hooks/{platform}/{plugin-name}.json`
2. On install/update/uninstall, we recompute the merged hooks from ALL overlay files
3. We write the merged result to the target platform's configuration file
4. The state directory serves as both provenance tracking AND unmerge mechanism

**Properties**:

- Provenance is inherent (each file = one plugin's contribution)
- Unmerge is clean: delete the plugin's overlay file, recompute merged result
- Manual edits to the merged output are overwritten on next recompute (with backup)
- Order can be controlled by plugin installation order or explicit priority
- Atomic: write the entire merged result, not incremental patches

**Verdict**: [PASS]. This is the recommended approach for structured configuration files.

#### Pattern C: Diff/Patch Approach

**How it works**: Record the JSON Patch (RFC 6902) that each plugin applies. On uninstall, apply the inverse patch.

**Properties**:

- Clean conceptual model
- RFC 6902 patches are well-standardized
- Inverse patches can be computed for reversibility
- FRAGILE: if the user manually edits the configuration, the inverse patch may fail or produce incorrect results
- FRAGILE: if another plugin modifies the same JSON paths, patch ordering matters and inverse patches may conflict

**Verdict**: [FAIL]. Too fragile for real-world use where users modify configuration files.

#### Pattern D: Manifest/Lockfile Tracking

**How it works**: Maintain a lockfile that records what each plugin contributed to each configuration file, with enough detail to reconstruct or remove it.

```json
{
  "plugins": {
    "my-plugin": {
      "version": "1.0.0",
      "installed": "2026-03-07T10:00:00Z",
      "hooks": {
        "claude-code": {
          "PreToolUse": [
            {
              "matcher": "Bash",
              "hooks": [{ "type": "command", "command": "..." }]
            }
          ]
        }
      }
    }
  }
}
```

**Properties**:

- Clear provenance tracking
- Can reconstruct what each plugin contributed
- Removal is a lookup + delete from both lockfile and target config
- More complex than overlay approach (lockfile duplicates information)
- Lockfile can become stale if target config is modified outside the tool

**Verdict**: [WARNING]. Viable but inferior to Pattern B2 (overlay) because it duplicates data and requires sync between lockfile and target config.

#### Pattern E: Database/State Store

**How it works**: Use a structured state store (SQLite or JSON database) to track all plugin contributions. Similar to Pattern D but with richer querying.

**Verdict**: [WARNING]. Overkill for this use case. The overlay approach (Pattern B2) provides the same benefits with simpler implementation.

### RQ4: Robustness Concerns

#### Manual Edit Detection

**Problem**: User manually edits the merged configuration file. On next install/update/uninstall, the tool must decide what to do.

**Solutions by pattern**:

- Overlay (B2): User edits are overwritten on recompute. Mitigate with backup + warning. Users should edit their OWN settings file, not the plugin-managed sections. Clear documentation is required.
- Lockfile (D): Can detect drift by comparing current file state to lockfile expectations. Can warn but not resolve automatically.

**Recommended approach**:

1. Compute hash of the merged section before writing
2. On next operation, compare current file content's hook section against stored hash
3. If mismatch, warn user: "Hook configuration was modified outside agent-plugin. Your changes will be preserved in a backup."
4. Create backup, then recompute from overlays + user's own hooks

#### Force Removal Without Proper Uninstall

**Problem**: User deletes a plugin's files without running `agent-plugin uninstall`.

**Solution**: The `agent-plugin doctor` command should detect orphaned hooks (hooks referencing scripts that no longer exist) and offer to clean them up. The overlay directory makes this straightforward: if a plugin's overlay file exists but the plugin is not in the registry, the hooks are orphaned.

#### Merge Conflicts Between Plugins

**Problem**: Two plugins register hooks for the same event with the same matcher pattern.

**Solution per ADR-003**: Hooks are additive. Both hooks run. No conflict. For the "strictest wins" policy on blocking decisions: if ANY hook returns "deny" for a PreToolUse event, the tool call is blocked. This is already how Claude Code handles it natively.

**Ordering**: Default to installation order (first-installed runs first). Allow explicit priority override in plugin manifest.

#### Atomicity

**Problem**: A crash during merge could leave configuration in a partially written state.

**Solution**:

1. Read current configuration
2. Compute new merged configuration in memory
3. Write to a temporary file
4. Atomic rename (rename temp file to target)
5. If rename fails, the original file is untouched

On platforms where atomic rename is not possible (some Windows scenarios), use the backup-write-verify pattern: backup original, write new, verify new file is valid JSON/YAML, if verification fails, restore from backup.

#### Rollback

**Problem**: A merge produces an invalid or undesirable configuration.

**Solution**: The overlay approach inherently supports rollback:

1. Delete the plugin's overlay file
2. Recompute merged result from remaining overlays
3. The pre-merge backup provides an additional safety net

### RQ5: Real-World Examples Analysis

| System | Merge Strategy | Unmerge Strategy | Key Insight |
|---|---|---|---|
| Claude Code plugins | Runtime merge from separate plugin hooks.json files | Plugin disable/uninstall removes from runtime | Best model: keep contributions separate, merge at runtime |
| Gemini CLI extensions | Merge from settings.json layers + extension bundled hooks | `gemini hooks uninstall <pkg>` removes npm package | npm-based lifecycle management |
| systemd drop-ins | Directory-based overlays, filename-based precedence | Delete the drop-in file | Directory = provenance, filename = ordering |
| Docker Compose | File-order merge, single-values replace, maps merge | Remove the override file | File ordering = precedence |
| Kustomize | Base + overlay directories with strategic merge patches | Remove overlay | Directory structure = provenance |
| ESLint flat config | Array of config objects, later overrides earlier | Remove the config object from the array | Explicit ordering in code |
| webpack-merge | Per-field strategy customization, unique for singletons | N/A (build-time only) | Per-field strategy is powerful |
| NixOS modules | Priority-based merge with mkBefore/mkAfter ordering | Remove the module import | Functional composition |
| Git config | Hierarchical include with conditional overrides | Remove the include directive | Clean layering model |
| Terraform overrides | Override files replace corresponding elements | Delete the override file | File = provenance |
| Ansible blockinfile | Marker-delimited sections in text files | state=absent removes block | Markers = provenance in text |
| WordPress | Marker-delimited sections with file locking | Remove block between markers | Proven for 20+ years |

**Cross-cutting insight**: The most robust systems keep each contributor's configuration in a separate file/location and compute the merged result. Systems that modify a single shared file (Ansible, WordPress) work but require marker-based provenance tracking. For structured configuration (JSON/YAML), the overlay/directory approach is universally preferred.

## 6. Discussion

### The Core Design Choice: Overlay vs In-Place Modification

For Claude Code's JSON hooks in settings.json, we face a fundamental choice:

**Option A: Modify settings.json directly.** Insert plugin hook entries into the user's settings.json, adding provenance metadata or relying on a lockfile to track contributions. On uninstall, find and remove the specific entries.

**Option B: Use the overlay/recompute approach.** Store each plugin's hook contributions in our state directory. Compute the merged hooks section and write it to a designated area of settings.json. On uninstall, remove the overlay file and recompute.

**Option C: Leverage Claude Code's native plugin system.** For Claude Code specifically, install plugins through `claude plugin install`, which handles hook merging at runtime without modifying settings.json at all.

Option C is the cleanest for Claude Code but does not generalize. Our tool must support platforms where we do not control the runtime.

Option A is fragile. JSON does not support comments or markers. Provenance metadata would require non-standard fields that the target platform might reject.

Option B is the recommended approach. It mirrors how every robust configuration management system works (systemd, Kustomize, Docker Compose, NixOS). It provides inherent provenance tracking, clean unmerge, and deterministic merge.

### Platform-Specific Hook Strategies

| Platform | Native Hook System | Our Strategy |
|---|---|---|
| Claude Code | JSON hooks in settings.json + plugin hooks.json | Prefer native plugin install. Fallback: overlay + recompute into settings.json |
| Gemini CLI | JSON hooks in settings.json + extension hooks | Overlay + recompute into settings.json |
| Cursor | No native hooks | Instruction-file-based behavioral hooks (ANALYSIS-009 pattern) |
| Kiro | Hooks in .kiro/ directory | Overlay + recompute into .kiro/ config |
| OpenCode | Configuration-based | Overlay + recompute into config file |
| Amp | Configuration-based | Overlay + recompute into config file |
| Windsurf | No native hooks | Instruction-file-based behavioral hooks (ANALYSIS-009 pattern) |

### The "Strictest Wins" Policy

For platforms with blocking hook semantics (Claude Code PreToolUse can deny tool calls), the merge must respect a "strictest wins" policy:

- If Plugin A's hook allows an action and Plugin B's hook blocks it, the action is blocked
- This is inherent in Claude Code's design: all hooks run in parallel, and if any returns "deny", the action is denied
- Our merge strategy does not need to implement this logic; the platform runtime handles it
- We only need to ensure ALL hook entries from ALL installed plugins are present in the merged configuration

### Recommended State Directory Structure

```
~/.agent-plugin/
  state/
    hooks/
      claude-code/
        plugin-a.json      # Plugin A's hook contributions for Claude Code
        plugin-b.json      # Plugin B's hook contributions for Claude Code
      gemini-cli/
        plugin-a.json
      kiro/
        plugin-a.json
    merged/
      claude-code/
        hooks-merged.json  # Last computed merge result (for drift detection)
      gemini-cli/
        hooks-merged.json
    registry.json           # Plugin registry with installation metadata
```

### Merge Algorithm (Pseudocode)

```
function mergeHooks(platform, overlayDir, targetConfigPath):
    overlayFiles = listFiles(overlayDir)
    merged = {}

    for file in overlayFiles (sorted by install order):
        pluginHooks = readJSON(file)
        for event in pluginHooks:
            if event not in merged:
                merged[event] = []
            for matcherGroup in pluginHooks[event]:
                merged[event].append(matcherGroup)

    existingConfig = readJSON(targetConfigPath)
    userHooks = getUserOwnedHooks(existingConfig)  // hooks not from any plugin
    finalHooks = deepMergeAdditive(userHooks, merged)

    backup(targetConfigPath)
    existingConfig.hooks = finalHooks
    atomicWrite(targetConfigPath, existingConfig)
    saveHash(platform, hash(finalHooks))
```

### Identifying User-Owned Hooks

A critical detail: when recomputing merged hooks, we must preserve hooks the user added manually. The overlay approach requires distinguishing user hooks from plugin hooks.

**Solution**: On first run (or when no overlays exist), snapshot the existing hooks as "user-owned". Store this snapshot in state. On recompute, start with the user snapshot, then layer plugin overlays on top. If the user adds new hooks manually, detect them on the next operation (entries in the target config that are not in any overlay and not in the user snapshot) and add them to the user snapshot.

This is analogous to how package managers handle user-modified configuration files (dpkg's conffile handling).

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Adopt the overlay/recompute pattern for structured configuration hooks. Each plugin's hook contributions stored as separate JSON files in the state directory. | Proven pattern across systemd, Kustomize, Docker Compose, NixOS. Provides inherent provenance, clean unmerge, deterministic merge. | Medium |
| P0 | For Claude Code, prefer native plugin installation (`claude plugin install`) when possible. Use overlay/recompute as fallback for non-plugin-compatible hooks. | Claude Code's runtime handles hook merging, deduplication, and lifecycle natively. Avoids reimplementing complex merge logic. | Low |
| P0 | Preserve user-owned hooks during merge/unmerge. Snapshot existing hooks before first plugin install. Track user hook changes across operations. | Users must not lose their manually configured hooks when plugins are installed or removed. | Medium |
| P1 | Implement atomic write for configuration file updates (write to temp file, atomic rename). Create timestamped backups before every modification. | Prevents corruption from crashes. Provides rollback path. | Low |
| P1 | Implement drift detection: hash the merged hooks section after writing, compare on next operation, warn if modified externally. | Prevents silent data loss when users edit plugin-managed configuration. | Low |
| P1 | Store merge hash and installation metadata in the state directory for `agent-plugin doctor` diagnostics. | Enables detection of orphaned hooks, stale overlays, and configuration drift. | Low |
| P1 | Use additive array concatenation for hook event arrays (append, never replace). Apply "strictest wins" semantics through the platform runtime, not our merge logic. | Matches ADR-003 decision. Platform runtimes already implement strictest-wins for blocking hooks. | Low |
| P2 | Implement an `agent-plugin doctor` command that validates hook state: detects orphaned hooks, missing overlays, configuration drift, and invalid JSON. | Safety net for force-removal, manual edits, and state corruption scenarios. | Medium |
| P2 | Support explicit hook priority in plugin manifest for controlling execution order within the merged result. Default to installation order. | Allows plugin authors to express ordering requirements. | Low |
| P2 | For platforms without native hook systems (Cursor, Windsurf), use instruction-file behavioral directives rather than attempting configuration-level hook injection. | These platforms lack structured hook configuration. Instruction files are the only interface. Already covered by ANALYSIS-009. | Low |

## 8. Conclusion

**Verdict**: Proceed with overlay/recompute pattern

**Confidence**: High

**Rationale**: The overlay/recompute pattern is the dominant approach across production configuration management systems (systemd drop-ins, Kustomize overlays, Docker Compose overrides, NixOS modules). It provides inherent provenance tracking (each file = one plugin), clean unmerge (delete file + recompute), deterministic merge (no state synchronization problems), and robustness against manual edits (backup + recompute from overlays). No existing npm package solves the combined problem of JSON configuration merging + provenance tracking + clean unmerge, confirming this requires a custom implementation.

### User Impact

- **What changes for you**: When you install a plugin with hooks, the tool stores the plugin's hook contribution separately and computes the merged configuration. On uninstall, the plugin's hooks are cleanly removed without affecting other plugins or your own hooks. A backup is created before every modification.
- **Effort required**: Medium. The overlay state directory, merge algorithm, user-hook preservation, and drift detection are the significant implementation tasks. The merge algorithm itself is straightforward array concatenation. Platform-specific adapters determine where to read/write configuration.
- **Risk if ignored**: Without provenance-aware merging, uninstall cannot reliably remove only the hooks from a specific plugin. Naive approaches (in-place JSON modification without tracking) risk data loss, orphaned hooks, and corruption of user-owned configuration.

## 9. Appendices

### Pattern Comparison Matrix

| Pattern | Provenance | Clean Unmerge | Manual Edit Handling | Atomicity | Complexity |
|---|---|---|---|---|---|
| Overlay/Recompute (B2) | Inherent (file=owner) | Delete + recompute | Backup + recompute; user hooks preserved | Atomic rename | Medium |
| Marker-Based (A) | In-file markers | Delete between markers | Markers can be edited/broken | File-level | Low (text) / N/A (JSON) |
| Diff/Patch (C) | Patch identifies changes | Inverse patch | Fragile (inverse may fail) | Patch-level | Medium |
| Manifest/Lockfile (D) | Lockfile tracks contributions | Lockfile lookup + delete | Lockfile drift detection | Lockfile + file | High |
| Database (E) | Full query support | Query + delete | Drift detection | Transaction | High |

### Library Reference

| Library | Use Case in Our System | Notes |
|---|---|---|
| deepmerge | Merging overlay JSON objects | Immutable merge, custom merge functions for arrays |
| json-merger | Alternative: inline merge directives | Overkill for our use case; designed for file-include workflows |
| webpack-merge | Reference for per-field strategy design | Not directly applicable but strategy pattern is instructive |
| RFC 6902 (JSON Patch) | Could be used for lockfile approach | Good for recording changes, fragile for unmerge |
| RFC 7396 (JSON Merge Patch) | Not recommended | Cannot represent null values; lossy |

### Claude Code Hook Deduplication Bug (Reference)

Issue #29724 in anthropics/claude-code: When multiple plugins register hooks with the same relative command path (e.g., `bash ${CLAUDE_PLUGIN_ROOT}/hook.sh`), deduplication occurs on the unexpanded template string, collapsing different plugins' hooks into one. Workaround: use unique script filenames per plugin. This bug is relevant to our merge implementation because we must ensure unique identification of hooks from different plugins.

### Sources Consulted

- Claude Code hooks reference: <https://code.claude.com/docs/en/hooks>
- Claude Code plugins reference: <https://code.claude.com/docs/en/plugins-reference>
- Claude Code hook deduplication bug: <https://github.com/anthropics/claude-code/issues/29724>
- Gemini CLI hooks docs: <https://geminicli.com/docs/hooks/>
- Gemini CLI hook install/uninstall issue: <https://github.com/google-gemini/gemini-cli/issues/9135>
- ESLint flat config: <https://eslint.org/docs/latest/use/configure/combine-configs>
- webpack-merge: <https://github.com/survivejs/webpack-merge>
- json-merger: <https://github.com/boschni/json-merger>
- deepmerge: <https://www.npmjs.com/package/deepmerge>
- Docker Compose merge: <https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/>
- Kubernetes Kustomize: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/>
- systemd drop-in overrides: <https://dev.to/redrum_yot/understanding-drop-in-overrides-in-systemd-when-parameters-accumulate-vs-override-3noi>
- NixOS modules: <https://wiki.nixos.org/wiki/NixOS_modules>
- Git config include: <https://git-scm.com/docs/git-config>
- Terraform override files: <https://developer.hashicorp.com/terraform/language/files/override>
- JSON Merge Patch RFC 7396: <https://datatracker.ietf.org/doc/html/rfc7396>
- JSON Patch vs Merge Patch comparison: <https://erosb.github.io/json-patch-vs-merge-patch/>

### Data Transparency

- **Found**: Complete hook configuration schemas for Claude Code and Gemini CLI. Merge/override patterns from 12 production systems. Package comparison for 5 npm merge libraries. Two RFC standards for JSON patching. Real bug report demonstrating multi-plugin hook deduplication failure.
- **Not Found**: Claude Code's internal hook merge implementation (closed source). How Gemini CLI internally handles hook deregistration on extension uninstall. Whether Kiro, OpenCode, or Amp have structured hook configuration systems (documentation insufficient). Performance characteristics of overlay-recompute vs in-place modification at scale (untested).

## Observations

- [decision] Overlay/recompute pattern selected for structured configuration hook merging: each plugin's hooks stored separately, merged result computed on install/update/uninstall #architecture #hooks
- [decision] Additive array concatenation for hook event arrays: never replace, always append. Platform runtime handles strictest-wins semantics. #conflict-resolution #hooks
- [fact] No npm package combines JSON configuration merging with provenance tracking and clean unmerge. This is a custom implementation requirement. #gap-analysis #npm
- [fact] Claude Code deduplicates hooks by unexpanded command template string, causing multi-plugin collisions (issue #29724). Our merge must use plugin-qualified identifiers. #claude-code #bug
- [fact] 12 production systems surveyed: overlay/directory-based approaches dominate for structured configuration (systemd, Kustomize, Docker Compose, NixOS, Terraform, Git config) #prior-art
- [technique] User-hook preservation via snapshot: capture existing hooks before first plugin install, track user changes across operations, layer plugin overlays on top of user snapshot #robustness
- [technique] Drift detection via hash comparison: store hash of merged hooks section, compare on next operation, warn if modified externally #robustness
- [risk] Platforms without native hook systems (Cursor, Windsurf) cannot use configuration-level hook injection. Must fall back to instruction-file behavioral directives. #cross-platform #limitation
- [insight] Claude Code's native plugin system handles hook merging at runtime without modifying settings.json. This is the ideal model but requires runtime control unavailable on most platforms. #claude-code #architecture
- [requirement] Atomic write operations (temp file + rename) required for configuration file updates to prevent corruption from crashes #reliability

## Relations

- extends [[ADR-003 Conflict Resolution and Namespacing]]
- extends [[ANALYSIS-009-instruction-file-update-patterns]]
- relates_to [[ANALYSIS-005 Claude Code Plugin Format]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
