# Index Setup Guide

> Instructions for building and maintaining the Dhee LMS codebase indexes.
> These indexes enable token-efficient code exploration by avoiding raw file reads.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Graphify Knowledge Graph](#graphify-knowledge-graph)
   - [What It Is](#what-it-is)
   - [Build the Full Workspace Graph](#build-the-full-workspace-graph)
   - [What a Successful Build Produces](#what-a-successful-build-produces)
   - [Incremental Update](#incremental-update)
   - [Full Rebuild](#full-rebuild)
   - [Query the Graph](#query-the-graph)
4. [JCodeMunch Symbol Indexes](#jcodemunch-symbol-indexes)
   - [What They Are](#what-they-are)
   - [Check Which Repos Are Indexed](#check-which-repos-are-indexed)
   - [Index a Repository](#index-a-repository)
   - [Index All Repositories](#index-all-repositories)
   - [Re-index After Code Changes](#re-index-after-code-changes)
5. [When to Rebuild Each Index](#when-to-rebuild-each-index)
6. [Decision Flow: Which Tool to Use](#decision-flow-which-tool-to-use)

---

## Overview

Two indexing systems work together to minimize token consumption when exploring
the Dhee LMS codebase:

| Index | What it stores | Best for | Token cost |
|-------|---------------|----------|------------|
| **Graphify** | Knowledge graph (23K+ nodes, 51K+ edges) | Architecture, cross-service relationships, design | Cheap — traverses subgraph, not raw files |
| **JCodeMunch** | Per-repo symbol tables | Functions, classes, references, imports, impact analysis | Extremely cheap — 99.6% avg reduction vs raw reads |

Both must be kept in sync with the actual codebase. Outdated indexes return
misleading results.

---

## Prerequisites

### Graphify

```bash
# Install via uv (recommended)
uv tool install graphifyy

# Or via pip
pip install graphifyy
```

Verify installation:

```bash
graphify --help
```

### JCodeMunch

The jcodemunch MCP tools are available via the `jcodemunch_order` function.
No separate installation is needed — the tools are provided by the MCP server.

---

## Graphify Knowledge Graph

### What It Is

A pre-built knowledge graph of the **entire workspace** (all repos combined).
It captures:
- AST-extracted code entities (functions, classes, imports, calls)
- Semantic relationships from docs, papers, and images
- Community clusters (logical groupings of related code)
- Cross-service connections

The graph lives at `graphify-out/` in the workspace root.

### Build the Full Workspace Graph

Run this from the **workspace root** (`/home/sourav/projects/dhee-lms`):

```bash
cd /home/sourav/projects/dhee-lms
graphify .
```

This runs the full pipeline:

1. **Detect** — scans all files, categorizes them (code/docs/papers/images)
2. **Extract** — AST extraction for code files + semantic extraction for docs/papers/images
3. **Build** — constructs the NetworkX graph from extracted entities and edges
4. **Cluster** — runs community detection to group related nodes
5. **Analyze** — identifies god nodes, surprising connections, generates questions
6. **Export** — writes `graph.json`, `GRAPH_REPORT.md`, `graph.html`

> **Timing:** The full build on ~2,700 files takes several minutes. Most of the
> time is spent on semantic extraction (docs/papers/images), which requires
> the LLM to process each chunk.

### What a Successful Build Produces

```
graphify-out/
├── graph.json              # Raw graph data (33MB, 23K+ nodes, 51K+ edges)
├── GRAPH_REPORT.md         # Human-readable report with communities, god nodes
├── graph.html              # Interactive HTML visualization
├── cost.json               # Token usage tracker
└── .graphify_python        # Python interpreter path (used by --update)
```

### Incremental Update

When only some files have changed (e.g., you edited a few files or pulled new
code), use `--update` instead of a full rebuild:

```bash
cd /home/sourav/projects/dhee-lms
graphify --update
```

This re-extracts only new or modified files and merges results into the
existing graph. Much faster than a full build.

### Full Rebuild

If the graph is corrupted or you want a completely fresh index:

```bash
cd /home/sourav/projects/dhee-lms
rm -rf graphify-out
graphify .
```

### Query the Graph

Once built, answer questions without touching raw files:

```bash
# Broad architecture question
graphify query "What are the main microservices and how do they connect?" --budget 3000

# Path between two concepts
graphify path "Authentication" "Gateway Service"

# Explain a specific node
graphify explain "proxy_request"

# Focused question
graphify query "Which services depend on TOAuth?"
```

---

## JCodeMunch Symbol Indexes

### What They Are

Per-repository databases containing symbol-level information:
- Functions, classes, methods, variables
- Imports and dependencies
- References and usages
- Type information

Each repo is stored as:
- A database file: `TechOptionsBD-<service-name>.db`
- An extracted directory: `TechOptionsBD-<service-name>/`

Both live under `/home/sourav/.code-index/`.

### Check Which Repos Are Indexed

```bash
jcodemunch_order(action="list_repos")
```

This returns all repositories currently indexed. Current indexed repos include:

| Repo | Service |
|------|---------|
| `TechOptionsBD-TOAuth` | Authentication & RBAC |
| `TechOptionsBD-lms-course-management-service` | Course Management |
| `TechOptionsBD-lms-content-management-service` | Content Management |
| `TechOptionsBD-lms-content-delivery-service` | Content Delivery |
| `TechOptionsBD-exam-service` | Exam |
| `TechOptionsBD-question-bank-service` | Question Bank |
| `TechOptionsBD-lms-assessment-service` | Assessment |
| `TechOptionsBD-lms-finance-service` | Finance |
| `TechOptionsBD-lms-reporting-service` | Reporting |
| `TechOptionsBD-lms-personalization-service` | Personalization |
| `TechOptionsBD-certificate-generation-service` | Certificate Generation |
| `TechOptionsBD-tutor-management-service` | Tutor Management |
| `TechOptionsBD-dhee-gateway-service` | Gateway |
| `TechOptionsBD-dhee-app` | Main Frontend |
| `TechOptionsBD-tenant-management-service` | Tenant Management |
| `TechOptionsBD-tenant-management-app` | Tenant Admin App |
| `TechOptionsBD-cirrs-notification-service` | Notification |
| `TechOptionsBD-profile-service` | Profile |
| `TechOptionsBD-payment-wallet-service` | Payment Wallet |
| `TechOptionsBD-ocr-service` | OCR |
| `TechOptionsBD-dhee-deployment` | Deployment |
| `TechOptionsBD-deployment-toolkit` | Deployment Toolkit |

### Index a Repository

```bash
jcodemunch_order(
    action="index_repo",
    args={"repo": "TechOptionsBD/<service-name>"},
    allow_state_change=true
)
```

Example:

```bash
jcodemunch_order(
    action="index_repo",
    args={"repo": "TechOptionsBD-lms-course-management-service"},
    allow_state_change=true
)
```

### Index All Repositories

To index every repository in the workspace, run the above command for each
repo. The index is incremental by default — subsequent runs are fast.

### Re-index After Code Changes

After editing code, switching branches, or pulling new code:

```bash
jcodemunch_order(
    action="index_repo",
    args={"repo": "TechOptionsBD/<affected-service>"},
    allow_state_change=true
)
```

This is an **incremental update** — only changed files are re-indexed.

---

## When to Rebuild Each Index

### Rebuild Graphify when...

| Trigger | Command |
|---------|---------|
| New service/repo added to workspace | `graphify --update` |
| Cross-service API changed (new routes, gateway proxies) | `graphify --update` |
| Architecture docs updated (ARCHITECTURE.md, README.md) | `graphify --update` |
| Wrong/incomplete answers from graph queries | `graphify --update` (or full rebuild) |
| User explicitly asks to refresh | `graphify --update` |

### Re-index JCodeMunch when...

| Trigger | Command |
|---------|---------|
| Code files added/modified/deleted | `index_repo` for affected repo(s) |
| Branches switched via `pull-code.sh` | `index_repo` for affected repo(s) |
| Rename refactoring completed | `index_repo` for affected repo(s) |
| New repo added to workspace | `index_repo` for the new repo |

### Rule of Thumb

Before any architecture-level investigation, run:

```bash
graphify --update
```

After any code change, re-index the affected repo(s):

```bash
jcodemunch_order(action="index_repo", args={"repo": "TechOptionsBD/<service>"}, allow_state_change=true)
```

When in doubt, rebuild — an accurate index saves far more tokens than a
rebuild costs.

---

## Decision Flow: Which Tool to Use

```
Question about codebase?
  │
  ├─ Architecture / design / high-level?
  │   └─ Use Graphify:  graphify query "..." --budget 3000
  │                      graphify path "X" "Y"
  │                      graphify explain "Symbol"
  │
  ├─ Specific symbol, function, or class?
  │   └─ Use JCodeMunch:  search_symbols("query")
  │                        get_context_bundle("Symbol")
  │                        find_references("Symbol")
  │
  ├─ Impact analysis / refactoring safety?
  │   └─ Use JCodeMunch:  get_blast_radius("Symbol")
  │                        check_rename_safe("Symbol")
  │                        get_dependency_graph("Symbol")
  │
  ├─ Exact file content, known path?
  │   └─ Use Read (direct file read)
  │
  └─ Fallback (nothing else works)
      └─ Use Grep / Glob (most expensive)
```
