---
parent: More info
nav_order: 310
description: Implementation-level guide to where Aider builds, ranks and token-optimizes repository maps.
---

# Repo map internals: where each part actually happens

This document maps high-level claims about Aider's repo map to concrete code locations.

## 1) "Builds a directed dependency graph using tree-sitter"

Core implementation is in `aider/repomap.py`:

- **Tree-sitter extraction of definitions/references** happens in `get_tags_raw()`:
  - chooses language/parser (`filename_to_lang`, `get_language`, `get_parser`)
  - loads `*-tags.scm` query files
  - parses file source and runs captures
  - emits `Tag(..., kind="def"|"ref")` records.
- **Directed graph construction** happens in `get_ranked_tags()`:
  - aggregates symbol definers/referencers
  - builds `nx.MultiDiGraph()`
  - adds weighted edges from referencer file -> definer file (dependency direction).

## 2) "Ranks files with PageRank"

Also in `get_ranked_tags()`:

- Computes optional personalization weights (chat files, mentioned files, path/identifier matches).
- Runs `networkx.pagerank(...)` over the graph and handles edge cases.
- Distributes rank onto definition-level `(file, ident)` targets and sorts output tags by rank.

## 3) "Produces token-optimized repo maps"

Primary flow:

- `RepoMap.get_repo_map()` sets effective token budget (`--map-tokens`, plus dynamic expansion when no files are in chat).
- `get_ranked_tags_map()` applies caching + refresh policy and calls uncached generation when needed.
- `get_ranked_tags_map_uncached()` does the **token optimization loop**:
  - ranks tags first,
  - then binary-searches how many ranked entries can fit,
  - repeatedly renders candidate trees and measures token count,
  - keeps the best map under budget (or near-budget tolerance).
- `to_tree()` and `render_tree()` produce the compact textual map containing only high-value lines of interest around definitions.

## 4) "Where this gets injected into prompts"

- `Coder` constructs `RepoMap(...)` during setup when repo-map usage is enabled.
- `Coder.get_repo_map()` gathers chat files, other repo files, mentions, and obtains repo content (with fallbacks).
- `Coder.get_repo_messages()` places the generated repo map into chat messages sent to the model.

## 5) What is **not** directly established by this code alone

The following claims are not computed in the core repo-map code path itself and should be sourced from benchmark/analysis docs if cited:

- "least tokens of any major coding agent"
- "8.5-13K tokens" and "4-6% context utilization"
- "competitive on edit accuracy"

The code here provides the mechanism (ranking + token budgeting), but those comparative metrics come from external evaluation methodology.

## 6) About the "~600 lines" claim

The repo-map core has historically been compact, but current `aider/repomap.py` is larger than 600 lines and includes caching, compatibility, rendering, and helper utilities in one file.

## 7) Related docs

- High-level user-facing explanation: `aider/website/docs/repomap.md`.
- Historical design narrative: `aider/website/_posts/2023-10-22-repomap.md`.

## 8) Additional claim check

Claim:

> "Tree-sitter repo maps (what Aider uses): builds in minutes even for the Linux kernel (28 million lines), minimal memory, supports 130+ languages, sub-millisecond incremental updates. About 600 lines of code."

How this lines up with this repo:

- **Tree-sitter repo maps**: yes, implemented in `aider/repomap.py` with tree-sitter queries and parsing.
- **Supports 130+ languages**: partially supported by project history, but phrased as **linter support** in the changelog (`130 new languages with linter support`), not an explicit hard guarantee about repo-map coverage in this file.
- **Builds in minutes for Linux kernel (28M LOC)**: not documented or tested in this repo as a reproducible benchmark artifact.
- **Minimal memory**: not claimed with concrete measurements in core implementation docs/code.
- **Sub-millisecond incremental updates**: not how this implementation is described; refresh behavior is based on map caching/refresh policy, and there is no explicit sub-millisecond incremental-update benchmark claim in tree/map docs.
- **About 600 lines of code**: outdated shorthand for current codebase size; current `aider/repomap.py` is larger.

If you want, we can add a companion benchmark note with measured numbers, hardware, repo size, and command lines so these claims become auditable.
