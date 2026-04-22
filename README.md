# Sybase Migration Evaluator

A single-file, fully offline HTML tool for **migration evaluation** of Sybase platforms. Combines regex-based T-SQL analysis, a structured migration-readiness checklist, a Sybase discovery-SQL library, and auto-generated technical and executive reports — all in one browser-local file, no install, no server, no network calls.

---

## How to Use

**Recommended: download and run locally.**

The tool is a single self-contained HTML file — no server, no build step, no dependencies.

**Option A — Download the file:**

1. Click **Code → Download ZIP** on this page (or download [`index.html`](index.html) directly)
2. Unzip if needed, then double-click `index.html` — opens in any modern browser
3. Everything works offline. Your SQL never leaves your machine.

**Option B — Clone the repo:**

```bash
git clone https://github.com/darshanmeel/sybase_migration_analyzer.git
cd sybase_migration_analyzer
# open index.html in your browser — that's it
```

**Option C — Live tool (quick tests only):**

> **[https://darshanmeel.github.io/sybase_migration_analyzer/](https://darshanmeel.github.io/sybase_migration_analyzer/)**
>
> Fine for exploring the UI or testing with non-sensitive sample SQL. For real client engagements, use the local file — client SQL should never be pasted into a browser tab that could be screenshotted, shared, or accidentally saved by browser history.

---

## What It Does

### Seven tabs, one file

| Tab | What it does |
|---|---|
| **Analysis** | Paste Sybase T-SQL or upload a `.sql` file. A regex rule engine (60+ patterns) splits input into objects (procs / views / functions / triggers), scores each, and produces a sortable heat map plus per-object collapsible detail with a color-highlighted annotated source view. Every rule carries rewrite hints for three migration targets — **PostgreSQL**, **Microsoft SQL Server**, and **Oracle** — switchable via a segmented control on the Analysis tab. Object scores, bucket labels, and estimated days all update live when the target changes. |
| **Checklist** | 70 phase-based migration-readiness items across six phases (Phase 1–6). Each item has a checkbox, notes/evidence field, risk weight (1–5), and a playbook reference. Readiness score updates live as you tick items. |
| **SQL Library** | 27 ready-to-run Sybase discovery queries across 7 sections — platform & version, object inventory, connections & usage, performance diagnostics, schema quality, Sybase IQ / SQL Anywhere, and export helpers. One-click copy, searchable. |
| **Reports** | Three live scores (Code Complexity, Migration Readiness, Composite Risk), an 8-slide presentation carousel, engagement-context narrative fields, and six export formats (Tech HTML, Tech Markdown, Executive Summary HTML, Executive Summary Markdown, Slide Deck HTML, Raw JSON). Code Migration effort adjusts live with target dialect; Data and Infra effort scale with the readiness gap. |
| **FAQ** | 25 common questions covering capabilities, limitations, scoring model, and how to interpret results. |
| **About** | User-focused overview, pre-engagement Methodology Deck download, methodology, full rule catalog, feedback links, and credit. |
| **Limitations** ⚠ | Dedicated tab (red warning styling) listing exactly what the tool can and cannot do. Read before sharing any report with a stakeholder. |

### Three scores, one composite

- **Code Complexity (0–100)** — derived from rule matches, LOC tiers, and density signals. Scores are **target-aware**: rules that map naturally to SQL Server score lower when that target is selected; Oracle-divergent constructs score higher for Oracle. Tiers: LOW / MEDIUM / HIGH / VERY HIGH. This score measures source code complexity independent of the readiness gap.
- **Migration Readiness (0–100, higher = more gap)** — sum of weights of unchecked checklist items normalized to a percentage. Items like "backup verified by restore test", "original architect available", or "source code in version control" carry higher weights. Tiers: READY / MOSTLY READY / MATERIAL GAPS / SEVERE GAPS.
- **Composite Risk (0–100)** — `0.45 × CodeComplexity + 0.55 × Readiness`. Readiness is weighted slightly heavier because human/process risks are harder to recover from than code.
- **Effort band (person-days)** — computed per object using target-adjusted bucket days, summed across all analyzed objects. The readiness gap inflates **Data and Infra** estimates (process/scheduling risk) but does **not** inflate Code Migration effort — rewrite complexity depends on the source code and target affinity, not on whether the organization has verified their backups. Shown as a ±15% band.
- **Top delay factors** — ranked list on the executive summary: all unchecked red-flag items by weight plus the top 3 rule categories by match count, each with a mitigation hint.

### Three target dialects, one source

The Analysis tab has a segmented control to pick your migration target: **PostgreSQL**, **Microsoft SQL Server**, or **Oracle**. Switching the target updates:

- Heat map score and bucket label for every object row
- Banner Est. Days total
- Per-object detail headers
- Reports Code Migration estimate
- Rewrite hints in rule tooltips and exported reports

The scores and days shown are all derived from the same per-rule, per-target overrides — so the heatmap rows, banner, and Reports tab always agree.

### Session save and restore (JSON round-trip)

Export a full session snapshot via **Raw Data · JSON** in the Reports tab. The export includes the original SQL input, all per-object analyzer results, checklist answers, scores, and a schema version. Re-import it later with **Import Raw JSON** to restore the full session — heat map, per-object detail, checklist, and scores all reload. Old exports (pre-schema-version) import with graceful degradation.

### Two reports, two audiences

- **Technical Report** (HTML or Markdown) — full analyzer findings, heat map, top-10 objects by score, rule-hit summary, phase-grouped checklist answers with notes, delay factors, effort band. Intended for the engineering team.
- **Executive Summary** (HTML or Markdown) — 2–3 page summary: three scores prominently, top-3 macro risks, effort band, approach recommendation (phased vs big-bang, team composition, rollback requirements, key dependencies). Intended for sponsors and decision-makers.

Plus a **Methodology Deck (HTML)** — 8 self-contained slides with keyboard nav to show clients at the start of an engagement, before any analysis is run. The Methodology Deck button is in the **About tab**. There is also a **Slide Deck (HTML)** — the same 8-slide format populated with actual evaluation results — and **Raw Data (JSON)** — full state dump for archival, re-import, or downstream tooling.

### Privacy by default

- **No network calls.** Zero `fetch()`, zero CDN, zero external resources. Completely offline.
- **No storage.** No `localStorage`, no `sessionStorage`. Refreshing the page clears everything.
- **No telemetry.** The tool does not phone home, ever. Your client's SQL never leaves your browser.
- **Export to resume.** Because there is no persistence, export Raw JSON at the end of each session if you need to resume. Re-import the JSON file to restore full state in the next session.

### Confidence is explicit

Every number the tool outputs ships with a visible `~92%` confidence label. The About tab and the header badge both state the confidence band; every exported report includes it at the top. The tool is a triage aid — not an on-site audit and not a substitute for reading the actual code.

---

## What It Does Not Do

**It is not a SQL parser.** The analyzer uses regex pattern matching. It does not understand nesting, scope, string literals, or semantics beyond direct pattern hits. False positives on unusual formatting are possible; false negatives on constructs outside the rule set are certain.

**It does not execute SQL.** The SQL Library is a copy/paste reference. The tool never connects to a database, and the analyzer only reads pasted text.

**It does not save state between sessions.** No localStorage, no file writes, no cloud sync. Close the tab and everything is gone. Export the JSON if you need to resume an engagement across days.

**It does not capture business-logic complexity.** The analyzer measures syntax and pattern counts. It cannot tell whether a cursor is necessary, whether a GOTO guards a complex invariant, or whether dynamic SQL is high-risk or trivial — only that they exist.

**It does not verify checklist answers.** The checklist is a consultant's running record of what the organization has confirmed. The tool treats every answer as self-reported and cannot cross-check against anything.

**It does not generate production-ready rewrites.** The target-dialect equivalents shown in tooltips and reports are illustrative starting points. Every real rewrite requires human judgment about data types, transaction semantics, and error handling.

**It does not support multi-file analyzer upload.** Concatenate your `.sql` files before pasting, or paste each one separately.

**It does not export to PDF.** Use the browser's Print → Save as PDF on any HTML export. The HTML exports are styled to print cleanly.

**Source dialect is Sybase T-SQL only.** The 60+ regex rules are tuned to Sybase T-SQL constructs. SQL Server as a *source* is not yet supported (planned). If you run it against Oracle or MySQL as a source, rule matches will be noisy, but the **checklist, SQL Library, scoring model, reports, and slide decks are fully reusable** regardless of source.

**Read the Limitations tab.** The in-app **Limitations** tab (red warning styling, rightmost in the nav) describes exactly what the tool can and cannot do. Read it before sharing any report with a stakeholder.

---

## Inputs Are Optional — Pick Your Path

You don't need to use every tab. Common paths:

- **Analysis-only:** paste SQL, click Analyze. Code Complexity populates; Migration Readiness stays "Not assessed"; Composite equals Code with a caveat. Export a Technical Report from the analyzer output.
- **Checklist-only:** skip Analysis entirely. Work the checklist during discovery calls. Readiness score updates live. Export an Executive Summary from checklist answers alone.
- **Both (recommended for a complete assessment):** run the analyzer on extracted DDL, work the checklist across the discovery phases, then export both reports from the Reports tab.
- **Methodology Deck first:** export the Methodology Deck before any engagement begins — it requires no data and explains the process to the client. Download it from the **About tab** (Pre-engagement materials section).

---

## Input Requirements (Analyzer Tab)

**The analyzer expects well-formatted, standard Sybase T-SQL.** Good inputs:
- Clean Sybase T-SQL extracted from the source (stored procedures, views, functions, triggers)
- `GO` batch terminators separating objects (standard Sybase script format)
- Optional SQL comments (`--` and `/* */`) — automatically stripped before rule matching

Poor inputs (will produce false positives or missed issues):
- SQL embedded inside application code (Java strings, Python variables)
- Raw script dumps mixed with non-SQL content (binary data, shell commands)
- Heavily obfuscated or minified SQL
- SQL from other dialects

---

## Deploying on GitHub Pages

The repo is ready to serve via GitHub Pages with zero configuration:

1. **Settings → Pages**
2. **Source:** *Deploy from a branch*
3. **Branch:** `main` · folder: `/ (root)`
4. **Save**

GitHub serves `index.html` at `https://<your-username>.github.io/<repo-name>/`. For this repo: **https://darshanmeel.github.io/sybase_migration_analyzer/**

No build step, no GitHub Actions workflow, no `gh-pages` branch needed.

---

## Repository Layout

```
index.html     ← the tool (single self-contained file, ~6,500+ lines)
README.md      ← this file
LICENSE
.gitignore
```

Internal methodology notes and build specs are kept locally in `.internal/` and are not tracked in git.

---

## License

See [LICENSE](LICENSE).

---

## Feedback & Suggestions

The checklist is not exhaustive. Real engagements regularly surface items the tool did not anticipate — document those and send them over so the tool can improve for the next engagement.

**Email:** darshan.meel@gmail.com  
**Suggest a checklist item:** [pre-filled email template](mailto:darshan.meel@gmail.com?subject=Sybase%20Migration%20Evaluator%20%E2%80%94%20New%20checklist%20item&body=Phase%3A%20%0AQuestion%3A%20%0AWhy%20it%20matters%3A%20%0ARisk%20weight%20(1-5)%3A%20%0ARed%20flag%20if%20unchecked%3F%20)

---

## Credit

Built and maintained by **Darshan Singh**. Email: darshan.meel@gmail.com

The checklist items, SQL library, scoring model, and report structure operationalize a Sybase → PostgreSQL migration evaluation playbook covering platform audit, technical-debt inventory, migration risk assessment, integration mapping, and engagement planning.
