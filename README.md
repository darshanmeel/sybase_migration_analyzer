# Sybase Migration Evaluator

A single-file, fully offline HTML tool for **evaluating Sybase migration complexity**. Paste your T-SQL, work through a structured readiness checklist, and export polished technical and executive deliverables — all from one file, in your browser, with no network calls and no install.

**Prepared by Darshan Singh** · darshan.meel@gmail.com

---

## Getting the Tool

> **Prefer running locally.** Client SQL should never travel through a shared browser session. Download the file and open it offline for any real engagement.

| Method | Steps |
|---|---|
| **Download (recommended)** | Click **Code → Download ZIP**, unzip, double-click `index.html` |
| **Clone** | `git clone https://github.com/darshanmeel/sybase_migration_analyzer.git` then open `index.html` |
| **Live — quick tests only** | [darshanmeel.github.io/sybase_migration_analyzer](https://darshanmeel.github.io/sybase_migration_analyzer/) |

The live URL is fine for exploring the UI or testing with sample SQL. For client engagements use the local file — it is identical, fully offline, and your SQL never leaves your machine.

---

## Methodology

The tool operationalizes a three-pillar migration evaluation approach:

```
┌─────────────────────────────────────────────────────────────────┐
│  Pillar 1 · Code Complexity   Pillar 2 · Readiness   Pillar 3 · Effort
│  ─────────────────────────   ──────────────────────   ──────────────────
│  Regex analysis of T-SQL      Structured checklist    Per-object days,
│  across 60+ rules, scored     across 6 phases and     by target dialect,
│  per object and per target    70 weighted items        with risk band
└─────────────────────────────────────────────────────────────────┘
                         ↓
              Composite Risk Score (0–100)
              = 0.45 × Code + 0.55 × Readiness

              + Delay Factors + Effort Band
                         ↓
              Technical Report  ·  Executive Summary  ·  Slide Deck
```

**Three scores, one composite:**

| Score | What it measures | Tiers |
|---|---|---|
| **Code Complexity (0–100)** | Rule matches, LOC tiers, dynamic SQL / cursor density. Target-aware: SQL Server-native constructs score lower for MSSQL target; Oracle-divergent constructs score higher for Oracle. | LOW · MEDIUM · HIGH · VERY HIGH |
| **Migration Readiness (0–100)** | Weight of unchecked checklist items, normalized to a gap percentage. Higher = more gap. | READY · MOSTLY READY · MATERIAL GAPS · SEVERE GAPS |
| **Composite Risk (0–100)** | `0.45 × Code + 0.55 × Readiness`. Readiness is weighted heavier because human/process risk is harder to recover from than code. | — |

**Confidence is explicit.** Every number ships with a `~92%` confidence label. The tool is a triage aid — not an on-site audit.

---

## What You Need to Bring

### For code analysis

Extract Sybase T-SQL from the source system and paste it into the Analysis tab. Good inputs:

- Stored procedures, views, functions, and triggers in standard Sybase T-SQL
- `GO` batch terminators separating objects (standard Sybase script format)
- SQL comments (`--` and `/* */`) — stripped automatically before rule matching

Poor inputs that will produce noise:
- SQL embedded inside application code strings (Java, Python, etc.)
- Raw dumps mixed with non-SQL content
- SQL from other dialects (rule matches will be misleading)
- Heavily obfuscated or minified SQL

If you have multiple `.sql` files, concatenate them before pasting, or paste each one separately.

### For the checklist

Work through it during discovery calls with the client. Each item has a notes/evidence field — record what the organization has actually confirmed, not what you expect to be true. The tool cannot cross-check answers; it treats everything as self-reported.

### For the reports

Fill in the engagement-context fields on the Reports tab (client name, scope narrative, approach notes) before exporting. These fields populate the executive summary and slide deck directly.

---

## The Six Phases (Checklist)

The 70-item checklist is organized across six migration phases. Work through them in order or jump to the phase most relevant to where you are in the engagement:

| Phase | Focus |
|---|---|
| **Phase 1 — Platform Audit** | Sybase version, edition, patch level, OS, connectivity, known constraints |
| **Phase 2 — Object Inventory** | Procs, views, functions, triggers, DDL completeness, source control status |
| **Phase 3 — Technical Debt** | Code quality, deprecated constructs, undocumented behaviour, test coverage |
| **Phase 4 — Migration Risk** | Data volume, integration points, downstream dependencies, rollback plan |
| **Phase 5 — Integration Mapping** | APIs, ETL pipelines, apps connecting to Sybase, credential management |
| **Phase 6 — Engagement Planning** | Team composition, timelines, tooling, stakeholder availability, go-live gates |

---

## Reports & Deliverables

All exports are generated from live state — scores, checklist answers, analyzer output, and effort estimates update in real time as you work. Switch the migration target and everything recalculates.

### Before the engagement — Methodology Deck

An 8-slide presentation to share with the client before any analysis begins. Explains the evaluation approach, scoring model, and what to expect. Download it from the **About tab** — it needs no data and is designed to be sent ahead of the first discovery call.

### After analysis — four export formats

| Export | Audience | Contents |
|---|---|---|
| **Technical Report** (HTML or Markdown) | Engineering team | Full analyzer findings, heat map, top-10 objects by score, rule-hit summary, phase-grouped checklist with notes, delay factors, effort band |
| **Executive Summary** (HTML or Markdown) | Sponsors & decision-makers | Three scores prominently, top-3 macro risks, effort band, approach recommendation (phased vs big-bang, team, rollback, key dependencies) |
| **Slide Deck** (HTML) | Presentation | 8-slide version of the executive summary, populated with actual evaluation results. Keyboard navigation. Print to PDF via browser. |
| **Raw Data** (JSON) | Archival / re-import | Full state dump: original SQL, per-object analyzer results, checklist answers, scores, schema version. Re-import later to restore the full session. |

> **Tip:** HTML exports are styled to print cleanly. Use browser Print → Save as PDF for any export — no separate PDF step needed.

### Effort estimates

Code Migration effort is the sum of per-object target-adjusted bucket days — the same values shown in the heat map. Switching the migration target recalculates code effort live across the heat map, banner, and all exports.

Data and Infra estimates scale with the readiness gap (checklist completion). Code Migration does not — rewrite complexity depends on the source code and target affinity, not on whether the organization has verified their backups.

| Component | Driven by |
|---|---|
| Code Migration | Per-rule, per-target adjusted object complexity |
| Data Migration | Object count + readiness gap |
| Infra Migration | LOC tiers + readiness gap |

---

## The Analysis Tab in Detail

Paste T-SQL and click **Analyze**. The rule engine:

1. Splits input into objects separated by `GO` terminators
2. Runs 60+ regex rules across each object (stored proc, view, function, trigger)
3. Scores each object on a 0–100 scale and assigns a bucket (LOW / MEDIUM / HIGH / VERY HIGH)
4. Builds a sortable heat map with per-row score, bucket, estimated days, and rule-hit count
5. Provides collapsible per-object detail with color-highlighted annotated source

**Target selector** — a segmented control on the Analysis tab lets you switch between **PostgreSQL**, **Microsoft SQL Server**, and **Oracle**. Every score, bucket label, estimated days value, and rewrite hint updates immediately. Rules that are native to SQL Server score 0 for the MSSQL target; Oracle-divergent constructs score higher for Oracle.

**SQL Library** — 27 ready-to-run Sybase discovery queries (platform version, object inventory, connection stats, performance diagnostics, schema quality, IQ / SQL Anywhere, export helpers) available on the SQL Library tab. One-click copy.

---

## Session Save & Resume

There is no auto-save. Close the tab and everything is gone.

Before ending a session, export **Raw Data · JSON** from the Reports tab. The file includes the original SQL, all per-object analyzer results, checklist answers, scores, and a schema version. Use **Import Raw JSON** on the next session to restore the full state — heat map, details, checklist, and scores all reload.

Old exports (without a schema version) import with graceful degradation: checklist and scores restore; analyzer output is unavailable.

---

## Privacy

- **No network calls** — zero `fetch()`, zero CDN, zero external resources
- **No storage** — no `localStorage`, no `sessionStorage`, no cookies
- **No telemetry** — the tool never phones home; your client's SQL never leaves the browser
- **Completely offline** — once the file is downloaded, it works with no internet connection

---

## Known Limitations

**Regex, not a parser.** The analyzer matches patterns. It does not understand nesting, scope, or string literals. False positives on unusual formatting are possible; constructs outside the 60+ rule set will be missed.

**No SQL execution.** The SQL Library is a copy/paste reference. The tool never connects to a database.

**No business-logic reasoning.** The tool cannot tell whether a cursor is necessary, whether dynamic SQL is high-risk or trivial, or whether a GOTO guards a complex invariant — only that they exist.

**Self-reported checklist.** The tool treats every answer as correct. It cannot verify what the organization has actually done.

**No production-ready rewrites.** Rewrite hints are illustrative starting points. Every real rewrite requires human judgment about data types, transaction semantics, and error handling.

**Source dialect: Sybase T-SQL only.** SQL Server as a *source* is planned but not yet available. Running it against Oracle or MySQL as a source will produce noisy rule matches, though the checklist, SQL Library, scoring model, and reports are fully usable regardless of source.

**No multi-file upload.** Concatenate `.sql` files before pasting.

**No PDF export.** Use browser Print → Save as PDF.

> The in-app **Limitations tab** (red warning styling) describes all constraints in detail. Read it before sharing any report with a stakeholder.

---

## Common Paths

You do not need to use every tab:

| Goal | Path |
|---|---|
| **Pre-engagement only** | Export Methodology Deck from About tab. No data needed. |
| **Code analysis only** | Paste SQL → Analyze → export Technical Report. Readiness stays "Not assessed". |
| **Checklist only** | Skip Analysis. Work checklist during discovery calls. Export Executive Summary. |
| **Full assessment** | Analyze extracted DDL + complete checklist + fill context fields → export both reports + Slide Deck. |
| **Resume a session** | Import Raw JSON at the start of the next session. |

---

## Deploying on GitHub Pages

```
Settings → Pages → Source: Deploy from a branch → main / (root) → Save
```

No build step, no Actions workflow, no `gh-pages` branch. GitHub serves `index.html` directly.

---

## Repository Layout

```
index.html     ← the entire tool (~6,500+ lines, self-contained)
README.md      ← this file
LICENSE
.gitignore
```

---

## Feedback & Suggestions

The checklist is not exhaustive. Real engagements regularly surface items the tool did not anticipate.

**Email:** darshan.meel@gmail.com  
**Suggest a checklist item:** [pre-filled template](mailto:darshan.meel@gmail.com?subject=Sybase%20Migration%20Evaluator%20%E2%80%94%20New%20checklist%20item&body=Phase%3A%20%0AQuestion%3A%20%0AWhy%20it%20matters%3A%20%0ARisk%20weight%20(1-5)%3A%20%0ARed%20flag%20if%20unchecked%3F%20)

---

## License

See [LICENSE](LICENSE).
