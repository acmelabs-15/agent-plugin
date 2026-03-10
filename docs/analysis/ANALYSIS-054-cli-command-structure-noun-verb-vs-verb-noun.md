---
title: ANALYSIS-054 CLI Command Structure Research Noun-Verb vs Verb-Noun
type: analysis
permalink: analysis/analysis-054-cli-command-structure-noun-verb-vs-verb-noun
tags:
- cli
- command-structure
- ux
- research
- tab-completion
- discoverability
---

# ANALYSIS-054 CLI Command Structure Research: Noun-Verb vs Verb-Noun

## 1. Objective and Scope

**Objective**: Determine whether agent-kit should use resource-first (noun-verb: `agent-kit skills add`) or verb-first (verb-noun: `agent-kit add skill`) command structure for managing 8 content types with 4 operations.

**Scope**: Research covers authoritative CLI design guides, 15+ production CLI tools, tab completion behavior, discoverability patterns, and AI/developer tooling precedent.

## 2. Evidence Catalog

### Authoritative Sources

| Source | Recommendation | Evidence |
|---|---|---|
| clig.dev | Noun-verb | "Either noun verb or verb noun ordering works, but noun verb seems to be more common." Example: `docker container create` |
| Heroku CLI Style Guide | Noun-verb | "Topics are plural nouns and commands are verbs." Example: `heroku apps:create` |
| PowerShell (Microsoft) | Verb-noun | Cmdlets use `Get-Process`, `Set-Item`. Designed for discoverability via verb reuse. |
| POSIX/GNU | Verb-first (flat) | Traditional Unix: `ls`, `rm`, `cp`. Single verbs, no subcommand hierarchy. |

### Production Tool Survey (15 tools)

**Resource-first (noun-verb) tools:**

| Tool | Pattern | Example | Resource Count |
|---|---|---|---|
| Docker | noun-verb | `docker container create`, `docker image ls` | 10+ resource types |
| GitHub CLI (gh) | noun-verb | `gh issue create`, `gh pr list`, `gh repo clone` | 6+ resource types |
| AWS CLI | noun-verb | `aws s3 ls`, `aws ec2 describe-instances` | 200+ services |
| Fly.io (flyctl) | noun-verb | `fly apps list`, `fly volumes create`, `fly secrets set` | 15+ resource types |
| Heroku | noun-verb | `heroku apps:create`, `heroku config:set` | 10+ resource types |
| Wrangler (Cloudflare) | noun-verb | `wrangler kv namespace create`, `wrangler r2 bucket list` | 5+ resource types |
| Vercel CLI | noun-verb | `vercel dns ls`, `vercel domains add`, `vercel env pull` | 8+ resource types |

**Verb-first tools:**

| Tool | Pattern | Example | Resource Count |
|---|---|---|---|
| kubectl | verb-noun | `kubectl get pods`, `kubectl create deployment` | 50+ resource types |
| PowerShell | verb-noun | `Get-Process`, `Set-Item` | 5000+ cmdlets |

**Flat/mixed (no nested subcommands):**

| Tool | Pattern | Example | Resource Count |
|---|---|---|---|
| git | flat verbs | `git add`, `git commit`, `git push` | 1 (repo) |
| npm | flat verbs | `npm install`, `npm publish`, `npm test` | 1 (package) |
| cargo | flat verbs | `cargo build`, `cargo add`, `cargo test` | 1 (crate) |
| brew | flat verbs | `brew install`, `brew tap`, `brew services` | 1 (formula) |
| terraform | flat verbs | `terraform plan`, `terraform apply`, `terraform destroy` | 1 (infra state) |
| Railway | mixed | `railway up`, `railway variable set` | 5+ resource types |

### Critical Pattern: Resource Count Determines Structure

Tools with 1 primary resource type (git, npm, cargo, brew, terraform) use flat verb-first because there is no ambiguity about the target. Tools managing multiple resource types (Docker, gh, AWS, Fly.io, Wrangler, Vercel) use noun-verb because the resource type is the primary disambiguation axis.

agent-kit manages 8 content types. This places it firmly in the multi-resource category alongside Docker, gh, AWS CLI, and Fly.io.

### Tab Completion Analysis

**Noun-verb advantage for multi-resource tools:**

With 8 resource types and 4 operations, the completion tree differs:

Noun-verb (`agent-kit skills [TAB]`):
- First TAB after `agent-kit`: 8 resource groups + a few top-level commands (~12 options)
- Second TAB after resource: 3-5 operations specific to that resource type
- Narrowing: immediate context reduction to relevant operations

Verb-first (`agent-kit add [TAB]`):
- First TAB after `agent-kit`: 4 verbs + a few other commands (~8 options)
- Second TAB after verb: all 8 content types (~8 options)
- No narrowing: every verb presents the same 8 content types

The noun-verb pattern produces more useful completions at the second level. After typing `agent-kit skills`, TAB shows only the operations valid for skills (create, remove, list, analyze). After typing `agent-kit add`, TAB shows all 8 content types regardless of context.

PowerShell's own documentation acknowledges this weakness: "typing 'Get-' and hitting Tab results in being overwhelmed with possibilities." The same problem applies to verb-first with many resource types.

**Noun-verb enables resource-specific operations:**

agent-kit has operations that only apply to certain content types:
- `analyze` / `analyze --fix` applies to skills, agents, mcp (not hooks, commands, rules)
- `create-tool` / `remove-tool` applies only to mcp

With noun-verb, these naturally nest under the resource group. With verb-first, they require awkward disambiguation or cannot exist.

### Discoverability Analysis

**New user question flow:**

Noun-verb: "I want to work with skills" -> `agent-kit skills --help` -> sees all skill operations
Verb-first: "I want to add something" -> `agent-kit add --help` -> sees all content types

Both flows are viable. The question is which mental model users arrive with.

For a tool managing AI agent content, users think in terms of content types first: "I need to manage my skills" or "I need to configure my hooks." They rarely think "I need to add something, but what?" The noun-verb pattern matches this mental model.

**Help output comparison:**

Noun-verb groups operations by resource, producing self-contained help per resource type. Verb-first groups resources by operation, requiring users to understand all content types before choosing.

### AI/Developer Tooling Precedent

| Tool | Pattern | Notes |
|---|---|---|
| Claude Code | Flat slash commands | `/review`, `/compact`, `/memory`. Single resource (conversation). |
| Gemini CLI | Flat commands | `show`, `add`, `refresh`. Single resource (memory). |
| VS Code Extensions CLI | Flat | `code --install-extension`, `code --list-extensions`. Single resource (extensions). |
| Vercel Skills CLI | Flat | `npx skills` (single command, interactive wizard). |
| ADR-014 current design | Noun-verb | `agent-kit skills create`, `agent-kit hooks list` |

AI coding tools use flat structures because they manage 1 resource type. agent-kit is closer to Docker/gh in managing multiple distinct resource types.

## 3. Recommendation

**Use resource-first (noun-verb): `agent-kit skills add`, `agent-kit plugins remove`**

### Justification by weight of evidence

**1. Ecosystem precedent is decisive.** 7 of 9 multi-resource CLI tools surveyed use noun-verb (Docker, gh, AWS, Fly.io, Heroku, Wrangler, Vercel). Only kubectl and PowerShell use verb-noun among multi-resource tools, and kubectl's design was driven by REST API mirroring rather than CLI UX principles.

**2. Tab completion produces better results.** Noun-verb narrows the completion space at the second level to 3-5 resource-specific operations. Verb-first shows the same 8 content types regardless of the verb chosen.

**3. Resource-specific operations require noun-verb.** `analyze` only applies to skills, agents, and mcp. `create-tool` only applies to mcp. Noun-verb accommodates these naturally. Verb-first cannot express them without awkward workarounds.

**4. ADR-014 already uses noun-verb.** The accepted architecture decision specifies `skills create`, `agents list`, `mcp create-tool`. Changing to verb-first would require an ADR amendment.

**5. clig.dev and Heroku both recommend noun-verb.** The two most cited CLI design guides both favor noun-verb for multi-resource tools.

**6. Users think in content types first.** "I want to manage my skills" is a more natural starting point than "I want to add something" for a content management CLI.

### One caveat worth noting

The top-level consumer commands (add, remove, update, install, list) in ADR-014 ARE verb-first: `agent-kit add <source>`, `agent-kit remove [plugin]`. This is correct because these commands target plugins (a single resource type), making them analogous to git/npm flat verbs. The noun-verb pattern applies to the content-type authoring commands where 8 resource types create disambiguation needs.

This hybrid is the same pattern Docker uses: `docker pull` (verb-first for the primary resource) alongside `docker container ls` (noun-verb for sub-resources).

## Observations

- [decision] Noun-verb (resource-first) recommended for content-type commands based on 7/9 multi-resource tool precedent #cli #architecture
- [fact] 7 of 9 surveyed multi-resource CLI tools use noun-verb: Docker, gh, AWS, Fly.io, Heroku, Wrangler, Vercel #evidence
- [fact] clig.dev states "noun verb seems to be more common" for two-level subcommand structures #authority
- [fact] Heroku CLI Style Guide mandates "topics are plural nouns and commands are verbs" #authority
- [insight] Resource count determines structure: 1 resource = flat verbs (git, npm), multiple resources = noun-verb (Docker, gh) #pattern
- [fact] Tab completion narrows to 3-5 operations per resource in noun-verb vs showing all 8 types per verb in verb-first #ux
- [insight] ADR-014 already specifies noun-verb for content-type groups with hybrid verb-first for top-level consumer commands #alignment
- [constraint] Resource-specific operations (analyze, create-tool) require noun-verb grouping to avoid disambiguation problems #architecture

## Relations

- relates_to [[ADR-014 Explicit Installation Model and Content Features]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ADR-012 Scaffolding and Content Management]]
- relates_to [[ANALYSIS-033 Consumer and Author Commands]]
- relates_to [[ANALYSIS-017 CLI Framework Comparison]]