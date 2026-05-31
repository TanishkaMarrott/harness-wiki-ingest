# LSP-Based Wiki Ingest System

An automated wiki ingest pipeline that keeps documentation in sync with source code changes — using Language Server Protocol (LSP) for code-structural impact detection instead of semantic graph traversal.

## The Problem

When source code changes, documentation drifts. A function gets renamed, a limit changes, a config rule gets removed — and the wiki still shows the old information. Students and support agents work from stale docs.

The naive solution is semantic similarity: "this file is related to these wiki pages, update them." The problem: **semantic proximity cannot tell you whether code is actually executed.**

**Real example found in testing:**

`s3_limits.py` defined `check_s3_bucket_count` and `check_s3_express_buckets`. A graph-based approach wrote S3 bucket limits into the wiki. Reality: both functions had zero callers — the file was never imported in the dispatch registry. The wiki documented enforcement rules that don't exist.

## The Solution — LSP-Based Impact Detection

LSP understands the entire project structurally — not just what a file contains, but who calls it, what depends on it, and whether it's actually reachable.

```
Graph answers:  which wiki pages are RELATED to this changed file?
LSP answers:    which wiki pages document something ACTUALLY EXECUTED?
```

### 5-Step LSP Path

```
LSP-1  document_symbols   → list all exports from changed file
LSP-2  find_references    → find all callers of each export
                            zero callers = dead code
LSP-3  hover              → get type signatures + return types
LSP-4  reconcile          → dead code → grep wiki → remove stale content
LSP-5  skill decides      → picks grep terms, finds affected pages, recompiles
```

## Architecture

```
Source repo (GitLab)
    │  git push
    ▼
GitLab CI detects changed files + encodes diff
    │  KML_CHANGED_FILES, KML_CHANGED_DIFFS
    ▼
GitHub Actions — ingest-lsp-demo.yml
    │
    ├── Sparse clone source repo (depth 1, platforms/AWS/ only)
    ├── Start cclsp MCP server wrapping pylsp
    ├── Claude Code skill — 5-step LSP path
    ├── Fetch source from API, recompile affected pages
    └── git commit + push wiki
```

### LSP Stack

```
Claude Code Skill
    ↕  MCP tool calls (document_symbols, find_references, hover)
cclsp — single MCP server
    ↕  LSP protocol (stdio)
pylsp — indexes source files on disk
    ↕  reads files
/tmp/source/ — sparse clone (depth 1, no history)
```

## Test Results

| File Changed | Scenario | Graph | LSP |
|---|---|---|---|
| `s3_limits.py` | Dead code | Wrote unenforced limits into wiki | Zero callers — no changes |
| `ec2.py` | Stale function | Would keep stale content | Removed stale, 9 pages recompiled |
| `threshold_config.yaml` | Cosmetic change | Would recompile 31 pages | Wiki accurate — correctly skipped |
| `ec2.py` (E2E) | Real GitLab push | — | 10 pages updated, committed |

**Graph: 0/3 correct. LSP: 4/4 correct.**

LSP also surfaced a source code bug: a dispatch registry still imported a renamed function that no longer exists — would raise `AttributeError` at runtime.

## Self-Healing

When LSP finds dead code (zero callers), the reconciliation step (LSP-4) automatically removes any stale wiki content referencing those symbols. The wiki heals itself.

## Key Design Decisions

- **Sparse clone** — only the directories pylsp needs, depth 1. No history, no irrelevant files.
- **cclsp as single MCP server** — Claude Code talks to one server. cclsp routes by file extension internally.
- **Skill decides grep terms** — no hardcoded service names. Skill reasons from LSP output + diff.
- **No graph artifact** — LSP never needs a pre-built index. Every run is fresh and deterministic.

## Harness Engineering

Per [Addy Osmani's harness engineering principles](https://addyosmani.com/blog/agent-harness-engineering/): Agent = Model + Harness. The improvements in this system are all harness improvements — the model didn't change, the scaffolding did.

The ratchet principle applied: each failure (graph wrote dead code rules, stale function in wiki) became a permanent constraint in `SKILL.md`. The harness evolved through evidence, not anticipation.

## Stack

- Claude Code (skill execution)
- cclsp + pylsp (LSP via MCP)
- GitHub Actions (orchestration)
- GitLab CI (source trigger)
- Python + bash (tooling)
