# Design Spec: JSON Re-import, Source-Dialect Architecture, Target-Aware Effort

**Date:** 2026-04-22  
**Scope:** Three features for the Sybase Migration Evaluator  
**Out of Scope:** Free/Enterprise gating, LLM-based rewrites, SQL Server rule pack

---

## 1. JSON Re-import (Round-Trippable Session Save)

### Goal
Enable users to **export a session and re-import it later**, completing the FAQ promise: "export Raw JSON at the end of each session … Full re-import support is planned."

### Current State
- `downloadRawJSON()` (line 5545) exports: `meta, scores, codeMetrics, checklist, delayFactors, effort`
- No import mechanism exists
- If a user closes the tab and wants to resume, they lose the analyzer output (heat map, per-object detail)

### Design

#### Export Changes
1. **Expand `downloadRawJSON()` payload** to include:
   - `sqlInput` — the original pasted SQL text (from `<textarea id="sql-input">`)
   - `codeResults` — the full per-object analyzer output (from `state.codeResults`)
   - `schemaVersion: 1` — versioning for future-proofing
   - Keep existing fields: `meta, scores, codeMetrics, checklist, delayFactors, effort`

2. **File naming:** Already good (`sybase_evaluation_<timestamp>.json`)

3. **Size impact:** Typical exports grow from ~5–10 KB to ~50–200 KB (includes original SQL). Acceptable for local storage.

4. **Privacy:** No change. File lives on user's disk; no network calls. Users already expect JSON export to be a full state dump.

#### Import UI
1. Add **"Import Raw JSON"** button in the `report-action-grid` (line 2036), placed near the "Raw Data · JSON" export button.
2. On click: trigger a hidden `<input type="file" accept=".json">`.
3. On file selection: read via `FileReader`, parse JSON, validate, restore state.

#### Import Flow
1. **Read & Parse:**
   - Use `FileReader.readAsText()` to load file
   - `JSON.parse()` — show error toast if invalid JSON

2. **Validate:**
   - Must have `tool` field matching `/^Sybase Migration Evaluator/`
   - Must have `schemaVersion` field (exactly `1` for this release)
   - If missing or mismatched: show error toast, abort
   - Non-blocking warning if version mismatch (minor mismatch OK, major mismatch warn but continue)

3. **Restore state:**
   - `state.meta` ← imported `meta`
   - `state.checklist` ← imported `checklist`
   - `state.scores` ← imported `scores`
   - `state.codeMetrics` ← imported `codeMetrics`
   - `state.codeResults` ← imported `codeResults` (if present)
   - Restore `<textarea id="sql-input">` text from imported `sqlInput`
   - Restore Sybase version dropdown from `meta.sybaseVersion`

4. **Re-render:**
   - Header: confidence score, three-score strip
   - Checklist tab: all answers, notes, checked state
   - Heat map & details (if `codeResults` was restored)
   - Reports tab: all cards recompute live
   - Show toast: `"Session imported."`

#### Backwards Compatibility
- Old exports (pre-spec, missing `schemaVersion`, no `codeResults`) → accept them
- Restore what's present, show toast warning: `"Partial import — analyzer output unavailable (legacy export)."`
- User still gets checklist answers and scores back; just not the heat map view

#### Error Handling (Explicit)
Each error gets a distinct, actionable toast:
- `"File not readable (corrupt or locked)."`
- `"Invalid JSON — check file format."`
- `"Not a Sybase Migration Evaluator export (missing 'tool' field)."`
- `"Incompatible export version (this tool expects v1)."`

---

## 2. Source-Dialect Architecture

### Goal
**Parameterize Sybase as the source dialect**, preparing the codebase for SQL Server support without building it yet. Today: Sybase only. Future: add SQL Server, refactor rules, and re-run at that time.

### Current State
- "Sybase" is hardcoded throughout: page title, subtitles, input labels, reports, FAQ prose, rule IDs
- No mechanism to distinguish "source dialect" (what we analyze) from "target dialect" (what we migrate to)

### Design

#### Constants
1. **Add `SOURCES` map** (next to `TARGETS` at line 2931):
   ```js
   const SOURCES = {
     sybase: { 
       label: 'Sybase',
       short: 'Sybase',
       field: 'sybase',
       note: 'Sybase T-SQL — 60+ regex rules tuned to ASE constructs.'
     },
     // mssql: { label: 'SQL Server', short: 'MSSQL', field: 'mssql', note: '...' }  // future
   };
   ```

2. **Update `state`** (line 2473): add `sourceDialect: 'sybase'`

#### UI: Read-Only Source Pill
- Add a **source selector** on the Analysis tab, visually next to the target segmented control (after line 1922)
- **For now:** a read-only pill or disabled segmented button: `"Source: Sybase · SQL Server (coming soon)"`
- Purpose: signal to users (and developers) that source and target are distinct concepts
- No user action possible in this release

#### String Refactoring Targets (Display-only)
Refactor **only** strings that describe the source-dialect concept. Replace hardcoded "Sybase" with `SOURCES[state.sourceDialect].label`:

1. **Page title & subtitles:**
   - Line 1796: `"Migration evaluation · Sybase → PostgreSQL · ..."` → `"Migration evaluation · ${SOURCES[...].label} → ${TARGETS[...].label} · ..."`
   - Similar updates in slide deck (line 4931), reports (lines 5460, 5641, 5730, 5803)

2. **Input card title:**
   - Line 1899: `"Paste Sybase T-SQL or Upload a File"` → `"Paste ${SOURCES[...].label} T-SQL or Upload a File"`

3. **Raw JSON payload:**
   - Add `source: state.sourceDialect` to the export

4. **Reports text:**
   - E.g., `"Sybase → PostgreSQL migration evaluation"` → `"${SOURCES[...].label} → ${TARGETS[...].label} migration evaluation"`

#### String Refactoring Exclusions (Intentional)
**Do NOT refactor** these — they are intrinsically Sybase or describe the tool itself:

- `RULES` array and rule IDs (`SYB-001`, etc.) — rule pack is Sybase-specific; will be branched when SQL Server rules are authored
- SQL Discovery Library (queries are Sybase-specific)
- FAQ prose that refers to "Sybase" by name — answering conceptual questions about the platform, not the source dialect
- Confidence calc's "Sybase-specific patterns" check — this is a dialect sanity check; refactoring is premature
- Page `<title>` and main `<h1>` — these describe the **tool**, not the source; less critical to parameterize
- README, license, footer version string
- `setSybaseVersion()` function and the Sybase version dropdown — scoped to the Sybase source

**Rationale:** Parameterize *only* the strings that conceptually belong to "which source are we analyzing." Leave strings that describe rules, the SQL library, or the tool's identity alone; those will shift when SQL Server support lands and the tool is extended.

#### No Functional Changes Yet
- Analyzer still uses `RULES` (Sybase-only)
- No SQL Server rule pack
- No source-dialect switching logic in the analyzer

---

## 3. Target-Aware Code Effort Adjustment

### Goal
**Scale code migration effort based on target dialect**. Sybase → SQL Server should estimate lower effort than Sybase → PostgreSQL, reflecting syntactic affinity.

### Current Model
`computeEffortBand()` (line 5265) calculates:
- **Code:** sum of per-object bucket days + bumps for dynamic SQL / cursors, scaled by readiness gap
- **Data:** object count + readiness gap
- **Infra:** fixed base + LOC bonus + readiness gap
- No target consideration

### Design

#### Constants
Add `TARGET_CODE_MULTIPLIER` constant (next to `SOURCES` and `TARGETS`):
```js
const TARGET_CODE_MULTIPLIER = {
  mssql:    0.75,  // Sybase cousin; ~25% less rewrite effort
  postgres: 1.0,   // default baseline
  oracle:   1.25,  // more translation; ~25% more effort
};
```

**Justification:**
- **MSSQL (0.75):** Sybase and MSSQL are T-SQL cousins. Many constructs carry over unchanged. ~25% rewrite savings.
- **PostgreSQL (1.0):** Baseline. Moderate rewrite effort (no affinity, no extra distance).
- **Oracle (1.25):** PL/SQL semantics diverge more. Savepoints, sequences, trigger syntax, cursor behavior. ~25% extra effort.

#### Code Modification
In `computeEffortBand()` (line 5265), after computing raw `code` effort but before applying readiness multiplier:

```js
function computeEffortBand() {
  const m = state.codeMetrics;
  const gap = state.scores.readiness || 0;
  const mult = 1 + 0.004 * gap;  // readiness gap inflates up to ~40%
  const round1 = (v) => Math.round(v * 10) / 10;

  let code = 0.5;
  if (m && m.buckets) {
    code = (m.buckets.veryHigh || 0) * 3
         + (m.buckets.high     || 0) * 1.5
         + (m.buckets.medium   || 0) * 0.5
         + (m.buckets.low      || 0) * 0.15;
    if (m.dynamicSqlCount > 0) code += Math.min(3,   0.5  * m.dynamicSqlCount);
    if (m.cursorCount     > 0) code += Math.min(2,   0.25 * m.cursorCount);
    code = Math.max(0.5, code);
  }
  
  // NEW: Apply target-dialect multiplier
  code *= (TARGET_CODE_MULTIPLIER[state.targetDialect] || 1.0);
  
  // Apply readiness gap multiplier
  code *= mult;

  // ... rest of function (data, infra, return)
}
```

#### What This Changes
- **Live:** When user switches target (via segmented control), `code` effort recomputes immediately.
- **Reports:** All exports (HTML, Markdown, JSON) show the adjusted effort.
- **Slide deck:** effort total updates to match target.

**Example:** 45 person-days for PostgreSQL → 34 person-days for SQL Server (same source, same code complexity, lower target-specific effort).

#### What This Does NOT Change
- **Code Complexity score** (0–100) — unchanged. Measure of source code, independent of target.
- **Composite Risk score** (0–100) — unchanged. Formula: `0.45 × Code + 0.55 × Readiness`.
- **Migration Readiness score** — unchanged. Independent of target.
- **Data & Infra effort** — unchanged. Target-agnostic.

#### Docstring
Add a brief comment in the code explaining the multipliers:
```js
// Target-dialect affinity affects code rewrite effort.
// MSSQL is syntactically closer to Sybase (0.75x). PostgreSQL is baseline (1.0x).
// Oracle requires more translation work (1.25x).
```

---

## Testing Checklist

- [ ] JSON export includes `sqlInput`, `codeResults`, `schemaVersion`
- [ ] JSON import validates `tool` field and `schemaVersion`
- [ ] JSON import restores all state fields and re-renders correctly
- [ ] Backwards-compatible import (old JSON without `codeResults` shows warning)
- [ ] Error toasts are distinct and actionable
- [ ] Source pill displays "Sybase" (read-only for now)
- [ ] All display strings reading `SOURCES[state.sourceDialect].label` render correctly
- [ ] Target multiplier applies correctly: switching targets changes code effort
- [ ] Code Complexity score is unaffected by target change
- [ ] Reports and slide deck reflect adjusted effort
- [ ] Existing functionality (analysis, checklist, all three targets) unaffected

---

## Files to Modify

1. `index.html` (only file in this project)
   - Add `SOURCES` constant
   - Add `TARGET_CODE_MULTIPLIER` constant
   - Update `state` initialization
   - Add import UI button and file input
   - Implement `importRawJSON()` function
   - Expand `downloadRawJSON()` export
   - Update `computeEffortBand()` with target multiplier
   - Refactor display strings (14 locations)
   - Update re-render logic where needed

---

## Out of Scope (Deferred)

- Free/Enterprise gating (top-5 issues, hidden phases, pre-migration trimming)
- LLM-based code rewrites (Enterprise feature)
- SQL Server rule pack (will be added in a separate project when SQL Server source support is built)

---

## Notes

- **Single-file simplicity:** No new files, no build step. Everything in `index.html`.
- **Backwards compatibility:** Old exports still importable with graceful degradation.
- **Preparation for the future:** Once SQL Server rule pack is ready, only `RULES` and `SOURCES.mssql` need to be added/updated; the architecture is already in place.
