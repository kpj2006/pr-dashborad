# AOSSIE PR Dashboard

> A local AI-powered PR analysis tool for open-source maintainers. No cloud. No pricing. No data leaves your machine.

> **⚠️ Testing Mode Active:** `github.py` is currently configured to fetch the last 10 **closed** PRs for testing. Before using in production, switch to open PRs — see [Configuration](#configuration).

### Motivation
Built to solve a real problem in AOSSIE's contributor workflow — when multiple contributors submit PRs solving the same problem differently, the maintainer had to read through long Discord threads to understand what conflicted, what duplicated, and what order to merge things. This tool automates that entire process locally.

---

## The Problem It Solves

Open-source projects with active contributors frequently face this:

- **PR #56** — contributor A adds persistence via JSON files
- **PR #61** — contributor B fixes the same persistence but with different wiring
- **PR #52** and **PR #51** — two contributors both implement dynamic difficulty, different approaches

The maintainer has to manually read every PR, compare approaches, decide what conflicts, and figure out merge order. For 10+ open PRs this becomes a significant cognitive load.

**This tool does that automatically.**

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                        PR Dashboard Pipeline                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. GitHub CLI (gh)                                              │
│     └── Fetch all open PRs                                      │
│     └── Extract CodeRabbit Walkthrough + Changes only           │
│         (ignores sequence diagrams, poems, tips — noise)        │
│                                                                  │
│  2. sentence-transformers (all-MiniLM-L6-v2, ~90MB, local CPU) │
│     └── Embed each PR's walkthrough into a semantic vector      │
│     └── community_detection() clusters PRs by meaning           │
│         - PRs solving same problem → same cluster               │
│         - Unrelated PRs → isolated                              │
│                                                                  │
│  3. Ollama (qwen2.5:7b, fully local)                            │
│     └── Loads repo context.md for giving context of what
          alredy done in repo.     
│         │
│     └── Deep analysis per conflict group:                       │
│         - Why do these approaches conflict?                      │
│         - What happens if A merged first? B first? Both?        │
│         - Recommended merge sequence                             │
│         - Which PR to prefer and why                            │
│     └── Single analysis per isolated PR                         │
│                                                                  │
│  4. Renders two HTML files, opens in browser                    │
│     └── conflicts_tree.html  — DAG merge graph                  │
│     └── isolated_prs.html    — safe-to-merge PRs                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## The ML Stack (All Local, All Free)

| Component | What It Does | Model/Tool | Size |
|---|---|---|---|
| **Semantic Embedding** | Converts PR text to vectors | `all-MiniLM-L6-v2` | ~90MB |
| **Clustering** | Groups PRs by meaning | `util.community_detection` | built-in |
| **LLM Analysis** | Explains conflicts, recommends order | `qwen2.5:7b` via Ollama | ~4.5GB |
| **PR Fetching** | GitHub API with auth | GitHub CLI (`gh`) | installed |
| **NLI** *(next)* | Labels pairs: duplicate / conflict / isolated | `cross-encoder/nli-deberta-v3-small` | ~180MB |

Zero API calls. Zero per-seat pricing. Zero data sent to external servers.

---

## The Dashboard Output

### `conflicts_tree.html` — Merge DAG

Visual tree showing every conflict group:

**DUPLICATE group** (same problem, same approach):
```
main ──┬── PR#A ──────────────╮
       │                       → [ SAME OUTCOME ] ← purple convergence node
       └── PR#B ──────────────╯
```

**CONFLICT group** (same problem, different approach):
```
main ──┬── PR#A ──► [outcome A] ──────╮
       │                               → [if both merged] ← dashed convergence
       └── PR#B ──► [outcome B] ──────╯
```

Each PR card shows: problem being solved, approach taken, strengths, weaknesses, outcome if merged, open questions for the maintainer.

### `isolated_prs.html` — Safe to Merge

PRs with no semantic overlap with anything else. Each shows full CodeRabbit walkthrough + Ollama analysis. These can be reviewed and merged independently.

---

## Setup

**Requirements:**
- Python 3.11+
- [GitHub CLI](https://cli.github.com/) installed and authenticated
- [Ollama](https://ollama.ai/) running with `qwen2.5:7b`

```bash
# 1. Clone / copy this folder
cd pr-dashboard

# 2. Create virtual environment
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # Mac/Linux

# 3. Install dependencies
pip install -r requirements.txt

# 4. Pull the Ollama model (one time)
ollama pull qwen2.5:7b

# 5. Authenticate GitHub CLI (one time)
gh auth login

# 6. Add repo context (optional but recommended)
# Create context.md with description of what's already built in the repo
# The more specific, the better the conflict analysis

# 7. Run
python main.py
```

---

## Configuration

All config is at the top of each file — no `.env` needed.

**`github.py`** — change the repo:
```python
REPO = "StabilityNexus/MiniChain"  # ← change this
```

**`grouping.py`** — tune clustering sensitivity:
```python
THRESHOLD = 0.55   # lower = more groups, higher = fewer but tighter groups
MIN_SIZE  = 2      # minimum PRs to form a conflict group
```

**`ollama.py`** — change the model:
```python
OLLAMA_MODEL = "qwen2.5:7b"  # any Ollama model works
```

**Switch to open PRs** (currently testing on closed PRs) in `github.py`:
```python
# Comment out test line, uncomment production line
return gh_paginate(f"repos/{REPO}/pulls?state=open&per_page=100")
```

---

## context.md

Drop a `context.md` file in the same folder as `main.py`. It gets prepended to every Ollama analysis call. Write it like a briefing for a new reviewer:

```markdown
# MiniChain Context

MiniChain is a minimal educational Python blockchain implementing the Marabu protocol.
Philosophy: conceptual minimality above all. Every PR is evaluated against this.

## Already Merged
- Ed25519 transaction signing and verification
- P2P networking with peer discovery
- Basic mempool as sorted queue (nonce-ordered)
- JSON file persistence (save/load on shutdown/startup)
- Dynamic difficulty with PID controller

## Current Priorities (Step 2)
- SQLite persistence migration
- Validation consolidation
- ...
```

The more context you give, the more accurate the conflict reasoning becomes.

---

## File Structure

```
pr-dashboard/
  main.py        ← entry point, orchestration
  github.py      ← GitHub CLI fetching + CodeRabbit extraction
  ollama.py      ← Ollama calls: group analysis, single PR analysis
  grouping.py    ← sentence-transformers clustering + group resolution
  render.py      ← HTML + SVG DAG tree generation
  context.md     ← your repo briefing (create this)
  requirements.txt
  .gitignore
```

---

## Roadmap
- [ ] fetch all unread messages from discord also just like i did in https://github.com/kpj2006/skill-bot-ask-ai- (for better context).
- [ ] **NLI layer** — `cross-encoder/nli-deberta-v3-small` to label each pair as `duplicate / conflict / isolated` with higher precision than cosine threshold
- [ ] **Merge order export** — output a `merge_order.md` Bruno can follow directly
- [ ] **GitHub comment integration** — post analysis as a PR comment (optional, keeps everything local by default)
