# Sybase Migration Evaluator (v2)

A single-file, fully offline HTML tool for **pre-acquisition technical diligence** of Sybase platforms. Combines regex-based T-SQL analysis, a structured migration-readiness checklist, a discovery-SQL library, and auto-generated technical and executive reports — all in one browser-local file, no install, no server, no network calls.

**Prepared by Darshan Singh.** The generated reports include this credit by default; it can be turned off per session via the gear icon in the header.

---

## What It Does

Open `sybase_analyzer.html` in any modern browser (Chrome, Edge, Firefox, Safari — last 2 major versions). The tool provides five tabs:

| Tab | What it's for |
|---|---|
| **Analysis** | Paste Sybase T-SQL or upload a `.sql` file. Rule engine (40 patterns) splits input into objects, scores each, and produces a heat map + annotated source. |
| **Checklist** | ~70 phase-based due-diligence items mapped to the engagement playbook (Pre-Call → Call 1-4 → Ops Maturity). Every item has a checkbox, notes field, risk weight, and playbook reference. |
| **SQL Library** | Copy-paste-ready Sybase discovery queries: version & config, object inventory, who's-connected, performance/locking snapshots, schema-quality audits, IQ-specific queries, and export helpers. |
| **Reports** | Three live scores (Code Complexity, Migration Readiness, Composite Risk), an 8-slide presentation carousel, and four export formats for downstream use. |
| **About** | Confidence disclosure (~90–95%), methodology, the full rule catalog, and known limitations. |

---

## How to Use

**You don't need to do all five tabs — use whichever match your current engagement phase.**

1. **Analysis-only path.** Paste SQL, click Analyze. The Code Complexity score populates immediately; the Composite score shows "needs checklist" until you fill one in.
2. **Checklist-only path.** Skip Analysis entirely. Work through the Checklist tab as interviews progress. Migration Readiness score updates live as you tick items. Reports tab generates an executive brief from checklist answers alone.
3. **Both (recommended for a full assessment).** Run the analyzer on extracted DDL, work the checklist across your discovery calls, then export the full Technical Report and Executive Brief from the Reports tab.

### Inputs

- **SQL:** paste into the textarea, or drag-and-drop a `.sql` / `.txt` file onto the upload zone.
- **Checklist:** tick items as you confirm them. Add notes/evidence per item. Export progress as JSON if you need to resume across sessions (no localStorage — refresh clears the tool).
- **Narrative:** On the Reports tab, add target company name, acquirer name, and a one-sentence "bottom line" — these appear on the cover of every export.
- **Sybase version** (optional): set via the gear icon; appears on report headers.

### Outputs

From the Reports tab:
- **Technical Report (HTML / Markdown)** — full analyzer findings, heat map, top-10 objects, rule hit summary, all checklist answers with notes, delay factors, effort band.
- **Executive Brief (HTML / Markdown)** — 2–3 page summary for acquirer co-founders: three scores, top 3 macro risks, effort band, commercial framing (price adjustment / TSA / escrow / walk-away).
- **Slide Deck (HTML)** — 8 self-contained slides with keyboard nav. Present live on a call.
- **Raw Data (JSON)** — full state dump for archival or downstream tooling.

All exports are self-contained HTML/MD — no external dependencies, no CDN, open anywhere.

---

## Input Requirements (Analyzer Tab)

**The analyzer is designed for well-formatted, standard Sybase T-SQL.**

Good input:
- Clean Sybase T-SQL extracted from the source (stored procedures, views, functions, triggers)
- `GO` batch terminators separating objects (standard Sybase script format)
- Optional SQL comments (`--` and `/* */`) — automatically stripped before rule matching

Poor input (will produce false positives or missed issues):
- SQL embedded inside application code (Java strings, Python variables)
- Raw script dumps mixed with non-SQL content (binary data, shell commands)
- Heavily obfuscated or minified SQL
- SQL from other dialects (Oracle PL/SQL, MySQL) — rules are tuned for Sybase

---

## Scoring Model

### Code Complexity (0–100)
Derived from the analyzer: per-object rule matches, LOC tiers, and density signals (Sybase-specific issues per 100 LOC, dynamic SQL, cross-DB refs, cursors). Tiers: **LOW 0–30 · MEDIUM 31–55 · HIGH 56–80 · VERY HIGH 81–100**.

### Migration Readiness (0–100, higher = more gap)
Sum of weights of unchecked checklist items, normalized to a percentage. Each of the ~70 items has a weight 1–5 and a red-flag marker for dealbreaker-adjacent risks (key-person loss, no backup, no version control, etc.). Tiers: **READY 0–25 · MOSTLY READY 26–50 · MATERIAL GAPS 51–75 · SEVERE GAPS 76–100**.

### Composite Risk (0–100)
`0.45 × CodeComplexity + 0.55 × Readiness`. Readiness is weighted slightly heavier because human and process risks are harder to recover from than code. If only one input exists, the composite equals that input (with a caveat).

### Effort Band (person-months)
Calibrated to the playbook reference (48k LOC / medium complexity ≈ 12 PM) and inflated by the readiness gap. Range shown as ±15% around the central estimate.

### Top Delay Factors
Ranked list on the executive brief: all unchecked red-flag items ordered by weight, plus the top 3 rule categories by match count. Each factor includes the mitigation hint pulled from the playbook.

---

## Confidence

Every number the tool outputs ships with a visible `~0.92` / `conf ~92%` label. For a well-formed input and a substantially-filled checklist, scores should land within about one tier of the reality that a human architect would report after on-site review. Confidence drops sharply when:

- SQL is malformed or mixed with non-SQL content
- The checklist is sparsely filled
- The platform uses features outside the rule set (custom extended stored procedures, IQ columnar analytics, Replication Server topology, undocumented thick-client dependencies)

**This tool is a triage aid, not a substitute for on-site review, DBA interviews, or reading the actual code.** See the About tab in the tool for the full methodology and limitations.

---

## Limitations

1. **Regex-based only.** No real SQL parser. Cannot understand nesting, scope, or semantics beyond pattern matches.
2. **Rubric scoring.** Two objects with the same score can differ dramatically in actual migration effort.
3. **No business-logic analysis.** The tool only reads syntax, not semantics. Complex business rules are invisible to it.
4. **Checklist is self-reported.** Answers are not independently verified — they're a consultant's running record of what the target company has confirmed.
5. **Scoring weights are opinionated.** Other engagements may warrant different weights; the weights here are based on the accompanying due-diligence playbook.
6. **Privacy-safe.** No data leaves your browser. No network calls, no storage, no logging. Refresh clears everything.
7. **Rewrite suggestions are illustrative.** Any PostgreSQL equivalents shown in tooltips or reports are starting points only.

---

## No External Dependencies

`sybase_analyzer.html` is a single self-contained file. All CSS and JavaScript are inline. It works completely offline with no CDN, no npm, no build step, and no server. Copy the file anywhere and open it.

---

## Playbook

The checklist items, SQL library, and report structure operationalize the methodology in `Sybase_MA_Playbook (1).md` — a pre-acquisition due-diligence playbook covering platform audit, technical-debt inventory, migration risk, integration mapping, and commercial framing.
