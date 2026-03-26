# PR Dashboard MVP overview

Summary of the current PR Dashboard MVP in this repository.

This is written as a system-module view (not as an isolated app):

- PR Dashboard analyzes pull requests and produces merge guidance.
- It consumes repository context now (`context.md`), and is designed to consume Skills Core context in the larger system.
- It outputs maintainer-facing conflict and isolated PR dashboards.

Reading guide:

- Each box is an action plus expected output.
- Diagrams are intentionally compact for proposal readability.
- Description blocks explain exactly what each diagram is showing.

---

## 1) End-to-End MVP (System Fit)

```mermaid
flowchart LR
  GH[GitHub PRs plus CodeRabbit comments] --> PD[PR Dashboard runtime]
  PD --> EMB[Embedding plus semantic clustering]
  EMB --> LLM[Ollama conflict reasoning]
  LLM --> OUT[conflicts_tree.html and isolated_prs.html]
  OUT --> MAINT[Maintainer merge decision]
  MAINT --> SYS[Feedback to Discord and future skill updates]
```

Description:

- Shows where this module sits in the full contributor-to-maintainer loop.
- Emphasizes MVP value: transform noisy PR streams into decision-ready outputs.
- Keeps the final actor clear: maintainer decides merge order.

---

## 2) Internal Pipeline (Code-Level Flow)

```mermaid
flowchart TD
  A[Check gh auth and Ollama availability] --> B[Load context.md if present]
  B --> C[Fetch PR list]
  C --> D[Fetch changed files and CodeRabbit sections per PR]
  D --> E[Create PR text for semantic matching]
  E --> F[Embed and cluster via community_detection]
  F --> G{Cluster size >= 2?}
  G -- Yes --> H[Run group analysis with Ollama]
  G -- No --> I[Run isolated PR analysis with Ollama]
  H --> J[Render conflict HTML]
  I --> K[Render isolated HTML]
```

Description:

- Matches the actual execution path in `main.py`, `grouping.py`, `ollama.py`, and `render.py`.
- Highlights the hard branch where PRs become either conflict groups or isolated cards.
- Distinguishes deterministic stages from LLM reasoning stages.

---

## 3) Data Flow (Inputs to Outputs)

```mermaid
flowchart LR
  IN1[PR metadata title body author files] --> NORM[Normalize PR records]
  IN2[CodeRabbit Walkthrough plus Changes] --> NORM
  IN3[context.md repo briefing] --> PROMPTCTX[Prompt context block]
  NORM --> TXT[Title plus walkthrough plus changes text]
  TXT --> VEC[Embedding vectors]
  VEC --> CLUS[Semantic clusters and isolated set]
  CLUS --> LLMIN[LLM input payloads]
  PROMPTCTX --> LLMIN
  LLMIN --> ANALYSIS[Structured JSON analyses]
  ANALYSIS --> HTML[Two HTML artifacts]
```

Description:

- Focuses on information movement, not function call order.
- Shows where context is injected and where structure is recovered from LLM output.
- Clarifies why output is operationally useful: structured analysis rendered as dashboards.

---

## 4) Runtime Sequence (Interaction by Module)

```mermaid
sequenceDiagram
  actor U as Maintainer
  participant M as main.py
  participant G as github.py
  participant R as grouping.py
  participant O as ollama.py
  participant H as render.py

  U->>M: run python main.py
  M->>G: fetch_prs and per-PR details
  G-->>M: PR records plus CodeRabbit sections
  M->>R: resolve_groups(pr_data, context)
  R->>O: analyse_group or analyse_single_pr
  O-->>R: structured analysis JSON
  R-->>M: conflict_groups plus isolated
  M->>H: build_conflict_html and build_isolated_html
  H-->>M: html strings
  M-->>U: open dashboard files in browser
```

Description:

- Shows call ownership and boundaries between modules.
- Makes external dependency points explicit: GitHub CLI and Ollama API paths.
- Useful for debugging and onboarding.

---

## 5) Decision Logic (Conflict vs Isolated)

```mermaid
flowchart TD
  S[Start with all PR embeddings] --> C[community_detection clusters]
  C --> Q{PR in cluster?}
  Q -- No --> ISO[Mark isolated and run single analysis]
  Q -- Yes --> G2{Cluster size >= 2}
  G2 -- No --> ISO
  G2 -- Yes --> CG[Conflict group candidate]
  CG --> T{Avg similarity > 0.75}
  T -- Yes --> DUP[Tag duplicate-leaning]
  T -- No --> CON[Tag conflict-leaning]
  DUP --> GA[Run group reasoning]
  CON --> GA
```

Description:

- Captures classification rules currently used in MVP.
- Makes clear that similarity labels are heuristic in v1 and refined by LLM reasoning.
- Useful as a bridge to roadmap Phase 3 (NLI precision layer).

---

## 6) Component Architecture (Current Boundaries)

```mermaid
flowchart LR
  subgraph Core[PR Dashboard Core]
    MAIN[main.py orchestration]
    GH[github.py ingestion and extraction]
    GR[grouping.py embeddings and clustering]
    OL[ollama.py reasoning client]
    RE[render.py HTML builder]
  end

  GHA[GitHub API via gh CLI] --> GH
  CTX[context.md local repo context] --> MAIN
  MAIN --> GH
  MAIN --> GR
  GR --> OL
  MAIN --> RE
  OL --> RE
  RE --> ART[conflicts_tree.html and isolated_prs.html]
```

Description:

- Shows module responsibilities and data handoff boundaries.
- Highlights that MVP is local-first and file-output oriented.
- Useful for identifying where to add cache, validation, and scheduling.

---

## 7) Integration View (Org-Level Interaction)

![alt text](public/image-6.png)

Description:

- Positions PR Dashboard within the full multi-module system.
- Shows current and target integration paths clearly in one compact view.
- Captures the feedback loop that keeps system knowledge fresh.

---

## 8) Key MVP Notes

1. Current context source is local `context.md`; roadmap target is Skills Core retrieval.
2. Semantic grouping is embedding-based (`all-MiniLM-L6-v2`) and local.
3. Conflict explanation and merge-order guidance are produced by local Ollama models.
4. Output is intentionally maintainer-facing HTML, not automatic merge actions.
5. MVP is local-first: no cloud LLM dependency and no mandatory data exfiltration.
6. Best next precision upgrade is NLI pair classification before DAG labeling.
