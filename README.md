# Sybase Migration Evaluator

A single-file, fully offline HTML tool for **pre-acquisition technical diligence** of Sybase platforms. Combines regex-based T-SQL analysis, a structured migration-readiness checklist, a Sybase discovery-SQL library, and auto-generated technical and executive reports — all in one browser-local file, no install, no server, no network calls.

**Live tool (GitHub Pages):** https://darshanmeel.github.io/sybase_migration_analyzer/

**Offline use:** download [`index.html`](index.html) and open it in any modern browser. That's the entire app.

**Prepared by Darshan Singh.** The generated reports include this credit by default; it can be turned off per session via the gear icon in the header.

---

## What It Does

### Five tabs, one file

| Tab | What it does |
|---|---|
| **Analysis** | Paste Sybase T-SQL or upload a `.sql` file. A regex rule engine (40 patterns) splits input into objects (procs / views / functions / triggers), scores each, and produces a sortable heat map plus per-object collapsible detail with a color-highlighted annotated source view. |
| **Checklist** | 70 phase-based due-diligence items across six phases (Pre-Call Prep, Call 1–4, Ops Maturity). Each item has a checkbox, notes/evidence field, risk weight (1–5), and a playbook reference. Readiness score updates live as you tick items. |
| **SQL Library** | 27 ready-to-run Sybase discovery queries across 7 sections — platform & version, object inventory, connections & usage, performance diagnostics, schema quality, Sybase IQ / SQL Anywhere, and export helpers. One-click copy, searchable. |
| **Reports** | Three live scores (Code Complexity, Migration Readiness, Composite Risk), an 8-slide presentation carousel, engagement-context narrative fields, and six export formats (Tech HTML, Tech Markdown, Executive Brief HTML, Executive Brief Markdown, Slide Deck HTML, Raw JSON). |
| **About** | Prominent confidence disclosure (~90–95%), methodology, full rule catalog, and limitations. |

### Three scores, one composite

- **Code Complexity (0–100)** — derived from rule matches, LOC tiers, and density signals (Sybase-specific issues per 100 LOC, dynamic SQL, cross-DB refs, cursors). Tiers: LOW / MEDIUM / HIGH / VERY HIGH.
- **Migration Readiness (0–100, higher = more gap)** — sum of weights of unchecked checklist items normalized to a percentage. Items like "backup verified by restore test", "original architect available", or "source code in version control" carry higher weights. Tiers: READY / MOSTLY READY / MATERIAL GAPS / SEVERE GAPS.
- **Composite Risk (0–100)** — `0.45 × CodeComplexity + 0.55 × Readiness`. Readiness is weighted slightly heavier because human/process risks are harder to recover from than code.
- **Effort band (person-months)** — calibrated to the playbook reference (48k LOC / medium complexity ≈ 12 PM) and inflated by the readiness gap. Shown as a range.
- **Top delay factors** — ranked list on the executive brief: all unchecked red-flag items by weight plus the top 3 rule categories by match count, each with a mitigation hint.

### Two reports, two audiences

- **Technical Report** (HTML or Markdown) — full analyzer findings, heat map, top-10 objects by score, rule-hit summary, phase-grouped checklist answers with notes, delay factors, effort band. Intended for the engineering team.
- **Executive Brief** (HTML or Markdown) — 2–3 page summary: three scores prominently, top-3 macro risks, effort band, commercial framing recommendation (price adjustment / TSA / escrow / walk-away), recommended acquisition conditions. Intended for acquirer co-founders and decision-makers.

Plus a **Slide Deck (HTML)** — 8 self-contained slides with keyboard nav to present live on a call — and **Raw Data (JSON)** — full state dump for archival or downstream tooling.

### Privacy by default

- **No network calls.** Zero `fetch()`, zero CDN, zero external resources. Completely offline.
- **No storage.** No `localStorage`, no `sessionStorage`. Refreshing the page clears everything.
- **No telemetry.** The tool does not phone home, ever. Your client's SQL never leaves your browser.

### Confidence is explicit

Every number the tool outputs ships with a visible `~92%` confidence label. The About tab and the header badge both state the confidence band; every exported report includes it at the top. The tool is a triage aid — not an on-site audit and not a substitute for reading the actual code.

---

## What It Does Not Do

**It is not a SQL parser.** The analyzer uses regex pattern matching. It does not understand nesting, scope, string literals, or semantics beyond direct pattern hits. False positives on unusual formatting are possible; false negatives on constructs outside the 40-rule set are certain.

**It does not execute SQL.** The SQL Library is a copy/paste reference. The tool never connects to a database, and the analyzer only reads pasted text.

**It does not save state between sessions.** No localStorage, no file writes, no cloud sync. Close the tab and everything is gone. Export the JSON if you need to resume an engagement across days.

**It does not capture business-logic complexity.** The analyzer measures syntax and pattern counts. It cannot tell whether a cursor is necessary, whether a GOTO guards a complex invariant, or whether dynamic SQL is high-risk or trivial — only that they exist.

**It does not verify checklist answers.** The checklist is a consultant's running record of what the target company has confirmed. The tool treats every answer as self-reported and cannot cross-check against anything.

**It does not generate production-ready PostgreSQL rewrites.** The PG equivalents shown in tooltips and reports are illustrative starting points. Every real rewrite requires human judgment about data types, transaction semantics, and error handling.

**It does not support multi-file analyzer upload.** Concatenate your `.sql` files before pasting, or paste each one separately.

**It does not export to PDF.** Use the browser's Print → Save as PDF on any HTML export. The HTML exports are styled to print cleanly.

**It is not tuned for Oracle, SQL Server, or MySQL.** Rules are Sybase ASE-focused. Running it on other dialects will produce mostly meaningless results.

---

## Inputs Are Optional — Pick Your Path

You don't need to use every tab. Common paths:

- **Analysis-only:** paste SQL, click Analyze. Code Complexity populates; Migration Readiness stays "Not assessed"; Composite equals Code with a caveat. Export a Technical Report from the analyzer output.
- **Checklist-only:** skip Analysis entirely. Work the checklist during discovery calls. Readiness score updates live. Export an Executive Brief from checklist answers alone.
- **Both (recommended for a complete assessment):** run the analyzer on extracted DDL, work the checklist across the discovery calls, then export both reports from the Reports tab.

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

## Using Locally

```
git clone https://github.com/darshanmeel/sybase_migration_analyzer.git
cd sybase_migration_analyzer
# open index.html in your browser — that's it
```

Or just download `index.html` directly from the repo and double-click it. It runs the same offline as it does on GitHub Pages.

---

## Repository Layout

```
index.html     ← the tool (single self-contained file, ~4,600 lines)
README.md      ← this file
LICENSE
.gitignore
```

Internal methodology notes and build specs are kept locally in `.internal/` and are not tracked in git.

---

## License

See [LICENSE](LICENSE).

---

## Credit

Built and maintained by **Darshan Singh**. The checklist items, SQL library, scoring model, and report structure operationalize a pre-acquisition Sybase → PostgreSQL due-diligence playbook covering platform audit, technical-debt inventory, migration risk, integration mapping, and commercial framing.
