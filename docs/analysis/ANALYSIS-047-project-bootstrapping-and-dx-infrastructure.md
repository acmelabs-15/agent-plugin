---
title: ANALYSIS-047 Project Bootstrapping and DX Infrastructure
type: note
permalink: analysis/analysis-047-project-bootstrapping-and-dx-infrastructure-1
tags:
- toolchain
- dx
- infrastructure
- pending
---

# ANALYSIS-047 Project Bootstrapping and DX Infrastructure

## Status

PENDING -- captured as future workstream, not yet researched.

## Scope

Engineering infrastructure decisions for the `@acmelabs-15/agent-plugin` CLI codebase itself. Covers toolchain selection, CI/CD, GitHub config, and release management.

## Open Questions

| Area | Questions |
|---|---|
| Linting/Formatting | Biome vs ESLint+Prettier? Biome config settings? |
| Runtime | Pure Bun (confirmed). Bun test runner vs Vitest? |
| Monorepo | Turbo repo? Or single package (one CLI)? |
| Build | TanStack Config? `bun build`? tsup? |
| Releases | Changesets? semantic-release? Bun publish? |
| Versioning | SemVer strategy, pre-release channels? |
| CI/CD | GitHub Actions workflows, triggers |
| GitHub Config | `main` not `master`, branch protection, required reviews, status checks |
| Testing | Vitest vs Bun test, coverage thresholds |

## Observations

- [requirement] Pure Bun runtime confirmed from ADR-005 #toolchain
- [requirement] Must determine if monorepo structure needed or single package sufficient #architecture
- [requirement] Release automation and versioning strategy needed before first publish #releases

## Relations

- depends_on [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[SESSION-2026-03-07_01 Agent Plugin Spec Ideation]]