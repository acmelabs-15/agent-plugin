---
title: DEBATE-ADR-011 Auto-Generated CLI from MCP Tools
type: critique
permalink: critique/debate-adr-011-auto-generated-cli-from-mcp-tools
tags:
- adr-review
- debate
- cli
- mcp
- auto-generation
- ADR-011
- agent-plugin
---

# DEBATE-ADR-011 Auto-Generated CLI from MCP Tools

## ADR Under Review

ADR-011 Auto-Generated CLI from MCP Tools

## Final Consensus: ACCEPTED (Round 2: 5 Accept, 1 D&C)

Round 1: 0 Accept, 0 D&C, 6 Needs Revision, 0 Block
Round 2: 5 Accept, 1 D&C, 0 Needs Revision, 0 Block

## Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | NEEDS REV | 1 | 3 | 3 |
| Critic | NEEDS REV | 2 | 4 | 3 |
| Independent Thinker | NEEDS REV | 2 | 3 | 2 |
| Security | NEEDS REV | 3 | 2 | 2 |
| Analyst | NEEDS REV | 2 | 3 | 2 |
| Advisor | NEEDS REV | 1 | 3 | 2 |

## P0 Issues (Consolidated)

### P0-1: Grouping Algorithm Contradiction (All 6 agents)

- ADR says "prefix-based" (line 70) but examples show suffix-based noun extraction (line 79)
- ANALYSIS-030 shows a third variant ("memory" not derivable from tool name)
- No pseudocode, no verb definition, no edge cases
- Advisor recommends dropping auto-grouping from v1, shipping flat commands + author-specified cli.groups only
**Decision needed: Drop auto-grouping heuristic for v1, ship flat commands only**

### P0-2: MCP Server Lifecycle Contradictory and Underspecified (Architect, Critic, Analyst, Independent Thinker, Advisor)

- NEG-002 says "subsequent invocations use running server" but IMP-002 says "terminated on CLI exit"
- No crash recovery, orphan detection, concurrent invocation, startup timeout
- Per-invocation vs persistent daemon not specified
- Critic: original DEBATE-ADR-010 P0-4 still unresolved
**Decision needed: Specify per-invocation + daemon lifecycle model**

### P0-3: Binary Name Shadowing (Security)

- No denylist prevents plugin named "git" or "node" from shadowing system binaries in ~/.local/bin
- CWE-427 (Uncontrolled Search Path Element)
**Decision needed: Add reserved name denylist + existing binary detection**

### P0-4: MCP Auto-Start Runs Untrusted Code (Security)

- Auto-starting plugin MCP servers grants arbitrary code execution with full user permissions
- CWE-250 (Execution with Unnecessary Privileges)
- Must document as accepted risk with install-time consent or mitigate
**Decision needed: Document trust model explicitly**

### P0-5: Custom Binary Path Traversal (Security)

- "./path/to/binary" not validated for path containment
- "../../usr/local/bin/something" could escape plugin directory
- CWE-22 (Path Traversal)
**Decision needed: Add realpath containment validation**

### P0-6: No Prior Art / Build-vs-Reuse Analysis (Independent Thinker)

- MCPShim, mcp-cli, FastMCP CLI, mcptools all exist and do similar things
- Alternatives section only lists "no CLI" and "mandatory CLI"
- No justification for building custom vs wrapping existing tools
**Decision needed: Add Prior Art section with justification**

### P0-7: Type Mapping Gap (Analyst)

- MCP inputSchema supports array, object, anyOf/oneOf combinators
- Gunshi supports only 5 primitive types + enum
- No mapping strategy for complex parameter types
**Decision needed: Add parameter type mapping table**

## P1 Issues (Consolidated, Deduplicated)

1. **~/.local/bin not on PATH by default** (Analyst, Critic) -- macOS and Ubuntu do not include ~/.local/bin in PATH by default. Need ensurepath mechanism or post-install warning.
2. **MCP schema type gaps for CLI parameter mapping** (Critic, Analyst) -- No specification of how array, object, enum map to CLI flags.
3. **Binary name collision / shadowing risk** (Critic, Security, Architect) -- Overlaps with P0-3, also covers CI mode behavior.
4. **Windows CLI distribution remains unscoped** (Critic, Advisor, Analyst) -- "Deferred to implementation" is ambiguous. Should be "out of scope for v1."
5. **Symlink attack on ~/.local/bin directory** (Security) -- Before creating symlink, verify ~/.local/bin is real directory not itself a symlink. CWE-59.
6. **No schema validation on CLI input forwarding** (Security) -- CLI inputs should be validated against MCP tool schemas before forwarding. CWE-20.
7. **Generation mechanism unspecified** (Analyst) -- Gunshi requires static command definitions. What artifact is produced at install time?
8. **Install-time command selection is premature for v1** (Advisor) -- Drop per-group multiselect, ship all-or-nothing CLI installation.
9. **MADR 4.0 frontmatter incomplete** (Architect, Critic) -- Missing status and date fields in YAML frontmatter.
10. **cli.groups schema validation not specified** (Architect, Critic) -- What happens with duplicate tool assignments, non-existent tool names, empty groups?
11. **MCP server auto-start security unaddressed** (Independent Thinker, Security) -- Overlaps with P0-4.
12. **Server lifecycle contradiction between NEG-002 and IMP-002** (Independent Thinker, Analyst) -- Overlaps with P0-2.

## D&C Reservations

None. All 6 agents voted NEEDS REVISION.

## Observations

- [decision] ADR-011 needs revision: 0 Accept, 0 D&C, 6 Needs Revision #adr-review
- [fact] 7 P0 issues identified across 6 reviewers, unanimous needs-revision verdict #revision-needed
- [fact] Grouping algorithm contradiction flagged by all 6 agents as highest-priority issue #grouping
- [fact] 3 security P0s: binary shadowing CWE-427, auto-start CWE-250, path traversal CWE-22 #security
- [risk] MCP server lifecycle has no specification for crash recovery, orphan detection, or concurrent invocation #lifecycle
- [insight] Advisor and Independent Thinker both recommend reducing v1 scope: flat commands only, no auto-grouping heuristic #scope
- [fact] MCPShim, mcp-cli, FastMCP CLI, mcptools identified as prior art not acknowledged in ADR #prior-art

## Round 2 Results

### Consensus: ACCEPTED (5 Accept, 1 D&C)

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | ACCEPT | 0 | 0 | 1 |
| Critic | ACCEPT | 0 | 3 | 2 |
| Independent Thinker | D&C | 0 | 3 | 3 |
| Security | ACCEPT | 0 | 0 | 4 |
| Analyst | ACCEPT | 0 | 2 | 1 |
| Advisor | ACCEPT | 0 | 0 | 2 |

### P0 Resolutions Verified

All 7 P0 issues from Round 1 resolved:

- P0-1: Grouping algorithm replaced with flat subcommands, no heuristic
- P0-2: MCP server lifecycle fully specified in Decision 2 (daemon/stdio)
- P0-3: Binary name denylist (31 entries) + which-based detection added
- P0-4: Trust Model section documents accepted risk with install-time consent
- P0-5: realpathSync() boundary check for custom binary paths
- P0-6: Prior Art section comparing MCPShim, mcp-cli, FastMCP CLI, mcptools
- P0-7: Parameter type mapping table covering 7 MCP schema types

### D&C Reservations (Independent Thinker)

- Daemon transport protocol must be specified before daemon feature ships
- `cli` field schema polymorphism (string vs object) needs clearer documentation
- Symlink directory attack vector (CWE-59) on ~/.local/bin not explicitly addressed

### Common P1 Themes

1. Daemon transport protocol unspecified (4 agents flagged)
2. `cli` field schema polymorphism — string vs object form (2 agents)
3. `mcp` as reserved group name to prevent author collision (1 agent)
4. Generation artifact format unspecified (1 agent)
5. ANALYSIS-030 contradicts ADR-011 on grouping and install-time selection (1 agent)

### Status Change

- ADR-011 status changed from Proposed to **Accepted**

- [outcome] ADR-011 Round 2 ACCEPTED: All 7 P0 issues resolved, 5 Accept + 1 D&C consensus achieved #round-2
- [outcome] Independent Thinker D&C: daemon transport protocol must be specified before daemon feature ships #reservation
- [fact] 4 of 6 agents flagged daemon transport protocol as the top remaining P1 issue #daemon-transport

## Relations

- reviews [[ADR-011 Auto-Generated CLI from MCP Tools]]
- relates_to [[ANALYSIS-030-auto-generated-cli-from-mcp-tools]]
- relates_to [[DEBATE-ADR-010 Installation Lifecycle and CLI Generation]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ADR-006 Core Dependency Stack]]
