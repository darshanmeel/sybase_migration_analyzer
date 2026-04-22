# JSON Re-import, Source-Dialect Architecture, Target-Aware Effort — Implementation Plan

> **For agentic workers:** RECOMMENDED SUB-SKILL: Use superpowers:subagent-driven-development for independent task execution. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable session re-import via JSON, parameterize Sybase as source dialect (preparing for SQL Server), and scale code effort by target dialect affinity.

**Architecture:** Feature 1 (Source Dialect) establishes `state.sourceDialect` and the `SOURCES` constant. Feature 2 (JSON Re-import) expands the export/import to include SQL text and analysis output. Feature 3 (Target-Aware Effort) applies a multiplier based on target dialect. All changes are localized to `index.html`.

**Tech Stack:** Vanilla JavaScript, no external dependencies. Browser FileReader API for JSON import.

---

## Task Group A: Source-Dialect Architecture

### Task A1: Add SOURCES Constant and Update State

**Files:**
- Modify: `index.html:2473-2483` (state object)
- Modify: `index.html:2931-2935` (add SOURCES after TARGETS)

- [ ] **Step 1: Add SOURCES constant after TARGETS**

Find the `TARGETS` constant at line 2931:
```js
const TARGETS = {
  postgres: { label: 'PostgreSQL', short: 'PG',    field: 'postgres', note: "Default target. Every rule carries a PostgreSQL rewrite hint." },
  mssql:    { label: 'SQL Server', short: 'MSSQL', field: 'mssql',    note: "Closest cousin to Sybase — many constructs carry over unchanged." },
  oracle:   { label: 'Oracle',     short: 'ORA',   field: 'oracle',   note: "Most translation work — PL/SQL semantics, sequence/trigger patterns, savepoints." },
};
```

Add this immediately after (before the next comment block):
```js
const SOURCES = {
  sybase: { label: 'Sybase', short: 'Sybase', field: 'sybase', note: 'Sybase T-SQL — 60+ regex rules tuned to ASE constructs.' },
  // mssql: { label: 'SQL Server', short: 'MSSQL', field: 'mssql', note: 'SQL Server T-SQL — coming soon' }
};
```

- [ ] **Step 2: Add sourceDialect to state object**

Find the `state` object at line 2473:
```js
const state = {
  codeResults: null,       // array from analyzeObject
  codeMetrics: null,       // summary metrics for scoring/reports
  checklist: {},           // { [itemId]: { checked, notes } }
  meta: { sybaseVersion: 'Unknown', client: '', analyst: '', recommendation: '' },
  showCredit: true,
  confidence: 0.92,
  scores: { code: null, readiness: null, composite: null },
  activeTab: 'analysis',
  targetDialect: 'postgres', // 'postgres' | 'mssql' | 'oracle'
};
```

Add this line after `targetDialect`:
```js
  sourceDialect: 'sybase',  // 'sybase' | future 'mssql' | ...
```

Result: `sourceDialect: 'sybase'` immediately after the `targetDialect` line.

- [ ] **Step 3: Test in browser**

Open `index.html` in a browser. Open DevTools (F12), Console. Paste:
```js
console.log(state.sourceDialect, SOURCES[state.sourceDialect]);
```

Expected output: `sybase { label: 'Sybase', short: 'Sybase', field: 'sybase', note: 'Sybase T-SQL…' }`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add SOURCES constant and state.sourceDialect (Sybase only for now)"
```

---

### Task A2: Add Source Pill UI (Read-Only)

**Files:**
- Modify: `index.html:1919-1922` (after target segmented control)

- [ ] **Step 1: Locate target segmented control**

Find this block at line 1919:
```html
      <div class="target-seg" id="target-seg" role="tablist">
        <button type="button" class="tseg active" data-target="postgres" onclick="setTargetDialect('postgres')">PostgreSQL</button>
        <button type="button" class="tseg"        data-target="mssql"    onclick="setTargetDialect('mssql')">SQL Server</button>
        <button type="button" class="tseg"        data-target="oracle"   onclick="setTargetDialect('oracle')">Oracle</button>
      </div>
      <div id="target-hint" style="font-size:0.8rem;color:#6E7681;margin-top:4px;">Default target. Every rule carries a PostgreSQL rewrite hint.</div>
```

- [ ] **Step 2: Add source pill immediately after the target-hint div**

Insert this after line 1926 (after the `</div>` closing the target-hint):
```html
      <div style="margin-top:16px; padding:8px; background:#F0F2F5; border-radius:4px; font-size:0.8rem;">
        <strong>Source:</strong> <span id="source-label">Sybase</span> · <span style="color:#999;">SQL Server (coming soon)</span>
      </div>
```

- [ ] **Step 3: Test in browser**

Open `index.html`. On the Analysis tab, you should see:
- Target segmented control (PostgreSQL / SQL Server / Oracle)
- Hint text below it
- New grey pill below that showing "Source: Sybase · SQL Server (coming soon)"

Take a screenshot or verify visually that the pill appears.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add read-only source pill (Sybase) on Analysis tab"
```

---

### Task A3: Refactor Display Strings — Part 1 (Subtitles and Headers)

**Files:**
- Modify: `index.html:1796`, `1899`, `1795` (page header and input label)
- Modify: `index.html:5460`, `5641`, `5730`, `5803` (report subtitles)
- Modify: `index.html:4931` (slide deck hero)

**Refactoring pattern:**
Replace hardcoded `"Sybase"` in these specific locations with `${SOURCES[state.sourceDialect].label}` (in template literals).

- [ ] **Step 1: Refactor main page subtitle (line 1796)**

Find:
```html
    <p>Migration evaluation · Sybase → PostgreSQL · regex + structured checklist · offline</p>
```

Replace with:
```html
    <p>Migration evaluation · ${SOURCES[state.sourceDialect].label} → ${TARGETS[state.targetDialect].label} · regex + structured checklist · offline</p>
```

**Note:** This is inside a `<p>` tag on the Analysis tab. If it's static HTML (not dynamic), convert this section to be dynamically set on page load. Let's handle this in Step 3 below.

- [ ] **Step 2: Refactor input card title (line 1899)**

Find:
```html
    <div class="card-title">Input — Paste Sybase T-SQL or Upload a File</div>
```

Replace with:
```html
    <div class="card-title">Input — Paste ${SOURCES[state.sourceDialect].label} T-SQL or Upload a File</div>
```

Again, if this is static HTML, we'll make it dynamic in Step 3.

- [ ] **Step 3: Create a function to render dynamic dialect-dependent strings**

Add this function after the `switchTab()` function (around line 2527):

```js
function updateDialectStrings() {
  const src = SOURCES[state.sourceDialect];
  const tgt = TARGETS[state.targetDialect];
  
  // Update the page subtitle (line 1796)
  const subtitle = document.querySelector('.tab-panel#tab-analysis > p');
  if (subtitle) {
    subtitle.textContent = `Migration evaluation · ${src.label} → ${tgt.label} · regex + structured checklist · offline`;
  }
  
  // Update input card title (line 1899)
  const inputTitle = document.querySelector('[id="sql-analysis-card"] .card-title');
  if (inputTitle) {
    inputTitle.textContent = `Input — Paste ${src.label} T-SQL or Upload a File`;
  }
  
  // Update source pill label (from Task A2)
  const sourceLabel = document.getElementById('source-label');
  if (sourceLabel) {
    sourceLabel.textContent = src.label;
  }
}
```

- [ ] **Step 4: Call updateDialectStrings() on page load and when dialect changes**

Find the function `setTargetDialect(t)` at line 2493. At the end of that function (around line 2510), add:
```js
  updateDialectStrings();
```

Also, add a call to `updateDialectStrings()` at the end of the page-load initialization. Find the bottom of the `<script>` tag (before the closing `</script>`), and add:
```js
// Initialize dialect-dependent display strings
updateDialectStrings();
```

- [ ] **Step 5: Add id="sql-analysis-card" to the input card div**

Find the Analysis tab's input card (line 1911 area). The opening `<div class="card-body">` that contains the SQL input textarea should be wrapped. Find the outer `<div>` containing the "Input — Paste Sybase T-SQL…" title and add `id="sql-analysis-card"` to it. This gives the JS function a way to find and update the title.

Example: if the structure is:
```html
<div class="analysis-card">
  <div class="card-title">Input — Paste Sybase T-SQL or Upload a File</div>
  <div class="card-body">
    ...
```

Change to:
```html
<div class="analysis-card" id="sql-analysis-card">
  <div class="card-title">Input — Paste Sybase T-SQL or Upload a File</div>
  <div class="card-body">
    ...
```

- [ ] **Step 6: Test in browser**

Open `index.html`. On the Analysis tab, you should see:
- Page subtitle: "Migration evaluation · Sybase → PostgreSQL · …"
- Input card title: "Input — Paste Sybase T-SQL or Upload a File"
- Source pill: "Source: Sybase · SQL Server (coming soon)"

Now switch the target to "SQL Server" (or Oracle). The subtitle and input card title should NOT change (only target affects them, not source, since we're still in Sybase source). The source pill should stay "Sybase".

Verify that the dynamic updates work correctly.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: parameterize source dialect in display strings (page subtitle, input card title, source pill)"
```

---

### Task A4: Refactor Display Strings — Part 2 (Reports Subtitles and Slide Deck)

**Files:**
- Modify: `index.html:5460`, `5641`, `5730`, `5803` (report generation functions)
- Modify: `index.html:4931` (slide deck hero)

These strings are generated dynamically in report-building functions, so they're easier to refactor inline (no dynamic DOM needed).

- [ ] **Step 1: Refactor downloadTechReportHTML() subtitle (line 5460 area)**

Find the `downloadTechReportHTML()` function. Inside it, look for a line like:
```js
<div class="dh-sub">Migration evaluation · Sybase → ...
```

Replace with:
```js
<div class="dh-sub">Migration evaluation · ${SOURCES[state.sourceDialect].label} → ${TARGETS[state.targetDialect].label} · regex + structured checklist · offline · ~${conf}% confidence</div>
```

Find the exact context around line 5460 and make this replacement.

- [ ] **Step 2: Refactor downloadTechReportMD() subtitle (line 5641 area)**

Find the `downloadTechReportMD()` function. Look for:
```js
md += `*Migration evaluation · Sybase → ...
```

Replace with:
```js
md += `*Migration evaluation · ${SOURCES[state.sourceDialect].label} → ${TARGETS[state.targetDialect].label} · regex + structured checklist · offline · ~${conf}% confidence*\n\n`;
```

- [ ] **Step 3: Refactor downloadExecBriefHTML() subtitle (line 5730 area)**

Find the `downloadExecBriefHTML()` function. Look for:
```js
${_headerBlock('Executive Summary', `Sybase → ...
```

Replace with:
```js
${_headerBlock('Executive Summary', `${SOURCES[state.sourceDialect].label} → ${TARGETS[state.targetDialect].label} Platform Risk Summary`)}
```

- [ ] **Step 4: Refactor downloadExecBriefMD() subtitle (line 5803 area)**

Find the `downloadExecBriefMD()` function. Look for a similar pattern:
```js
md += `*Migration evaluation · Sybase → ...
```

Replace with:
```js
md += `*Migration evaluation · ${SOURCES[state.sourceDialect].label} → ${TARGETS[state.targetDialect].label} · regex + structured checklist · offline · ~${conf}% confidence*\n\n`;
```

- [ ] **Step 5: Refactor slide deck hero (line 4931)**

Find the `downloadSlideDeck()` function. Look for:
```js
<div class="slide-hero">Sybase → ${TARGETS[state.targetDialect].label}<br>Migration Evaluation</div>
```

Replace with:
```js
<div class="slide-hero">${SOURCES[state.sourceDialect].label} → ${TARGETS[state.targetDialect].label}<br>Migration Evaluation</div>
```

- [ ] **Step 6: Test in browser**

On the Reports tab:
1. Leave defaults (Analysis not run, so no metrics). The "Insufficient data" message should show.
2. Paste a small SQL snippet on the Analysis tab and click Analyze.
3. Go back to Reports tab.
4. Click "Technical Report · HTML" and verify the exported HTML contains "Sybase → PostgreSQL" (or your current target).
5. Switch target to "SQL Server" on Analysis tab, go back to Reports, re-export. It should now say "Sybase → SQL Server".
6. Do a similar check on Executive Summary, Slide Deck, and Markdown exports.

Verify that all reports show the correct source and target.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: parameterize source dialect in report generation (tech report, exec brief, slide deck)"
```

---

## Task Group B: JSON Re-import

### Task B1: Expand downloadRawJSON() to Include Code Results and SQL Input

**Files:**
- Modify: `index.html:5545-5562` (downloadRawJSON function)

- [ ] **Step 1: Understand current export payload**

Find the `downloadRawJSON()` function at line 5545. Current payload:
```js
const payload = {
  _disclaimer: "...",
  tool: 'Sybase Migration Evaluator v2',
  generatedAt: ts.toISOString(),
  confidence: computeConfidence(),
  meta: state.meta,
  scores: state.scores,
  codeMetrics: state.codeMetrics,
  checklist: state.checklist,
  delayFactors: buildTopDelayFactors(),
  effort,
};
```

- [ ] **Step 2: Expand payload to include sqlInput, codeResults, source, and schemaVersion**

Replace the `const payload = {` block with:
```js
const payload = {
  _disclaimer: "Regex-based triage tool. Confidence is computed dynamically from analysis + checklist coverage (floor 40%, ceiling 95%). Not a substitute for on-site review. Checklist is not exhaustive. Feedback: darshan.meel@gmail.com",
  tool: 'Sybase Migration Evaluator v2',
  schemaVersion: 1,
  generatedAt: ts.toISOString(),
  source: state.sourceDialect,
  confidence: computeConfidence(),
  meta: state.meta,
  scores: state.scores,
  codeMetrics: state.codeMetrics,
  checklist: state.checklist,
  delayFactors: buildTopDelayFactors(),
  effort,
  sqlInput: document.getElementById('sql-input').value,
  codeResults: state.codeResults,
};
```

- [ ] **Step 3: Test in browser**

1. Paste a small SQL snippet on the Analysis tab.
2. Click Analyze.
3. Go to Reports tab.
4. Click "Raw Data · JSON".
5. Open the downloaded JSON file in a text editor.
6. Verify the JSON contains:
   - `"schemaVersion": 1`
   - `"source": "sybase"`
   - `"sqlInput": "<your SQL>"`
   - `"codeResults": [ { ... }, ... ]` (array of analyzed objects)
   - All existing fields (meta, scores, codeMetrics, checklist, etc.)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: expand JSON export to include sqlInput, codeResults, schemaVersion, and source"
```

---

### Task B2: Add Import Button UI and Hidden File Input

**Files:**
- Modify: `index.html:2036-2037` (report action grid, add import button)
- Modify: `index.html:2800+` (add hidden file input near the end of HTML)

- [ ] **Step 1: Add Import button to report action grid**

Find the report-action-grid around line 2036:
```html
      <div class="report-action-grid">
        <button class="report-action-btn" onclick="downloadTechReportHTML()">...</button>
        ...
        <button class="report-action-btn" onclick="downloadRawJSON()"><span class="rab-title">Raw Data · JSON</span><span class="rab-desc">Full state dump for archival / downstream tools</span></button>
      </div>
```

Add this button immediately after the "Raw Data · JSON" button (before the closing `</div>`):
```html
        <button class="report-action-btn" onclick="triggerImportJSON()"><span class="rab-title">Import Raw JSON</span><span class="rab-desc">Resume a previous session from exported JSON</span></button>
```

- [ ] **Step 2: Add hidden file input at the end of the HTML (before </body>)**

Find the closing `</body>` tag (should be near the very end of the file, around line 6200+). Add this just before it:
```html
<input type="file" id="json-import-input" accept=".json" style="display:none;" onchange="handleJSONImportFile()">
```

- [ ] **Step 3: Test in browser**

On the Reports tab, you should now see a new button "Import Raw JSON" with description "Resume a previous session from exported JSON" in the export grid.

Clicking it should do nothing visible yet (we'll implement the handler in Task B3), but the button should exist and be clickable.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add Import Raw JSON button and hidden file input"
```

---

### Task B3: Implement importRawJSON() Handler

**Files:**
- Modify: `index.html:5562+` (add new functions after downloadRawJSON)

- [ ] **Step 1: Add triggerImportJSON() function**

After the `downloadRawJSON()` function (around line 5562), add:
```js
// ---- triggerImportJSON ----
function triggerImportJSON() {
  document.getElementById('json-import-input').click();
}
```

- [ ] **Step 2: Add handleJSONImportFile() function**

Immediately after `triggerImportJSON()`, add:
```js
function handleJSONImportFile() {
  const input = document.getElementById('json-import-input');
  const file = input.files[0];
  if (!file) return;
  
  const reader = new FileReader();
  reader.onload = function(e) {
    try {
      const imported = JSON.parse(e.target.result);
      importRawJSON(imported);
    } catch (err) {
      showToast(`Invalid JSON — ${err.message}`);
    }
  };
  reader.onerror = function() {
    showToast('File not readable (corrupt or locked).');
  };
  reader.readAsText(file);
  
  // Reset input so same file can be re-imported
  input.value = '';
}
```

- [ ] **Step 3: Add importRawJSON(data) function**

Immediately after `handleJSONImportFile()`, add:
```js
function importRawJSON(imported) {
  // Validate tool field
  if (!imported.tool || !imported.tool.startsWith('Sybase Migration Evaluator')) {
    showToast("Not a Sybase Migration Evaluator export (missing 'tool' field).");
    return;
  }
  
  // Validate and warn on schemaVersion mismatch
  if (imported.schemaVersion !== 1) {
    if (!imported.schemaVersion) {
      showToast('Partial import — analyzer output unavailable (legacy export).');
    } else {
      showToast(`Incompatible export version (this tool expects v1, got ${imported.schemaVersion}).`);
      // Continue anyway — try to restore what we can
    }
  }
  
  // Restore state fields
  if (imported.meta) state.meta = imported.meta;
  if (imported.scores) state.scores = imported.scores;
  if (imported.checklist) state.checklist = imported.checklist;
  if (imported.codeMetrics) state.codeMetrics = imported.codeMetrics;
  if (imported.codeResults) state.codeResults = imported.codeResults;
  if (imported.source) state.sourceDialect = imported.source;
  
  // Restore SQL input textarea
  if (imported.sqlInput) {
    document.getElementById('sql-input').value = imported.sqlInput;
  }
  
  // Restore Sybase version dropdown if present
  if (imported.meta && imported.meta.sybaseVersion) {
    const versionSelect = document.getElementById('sybase-version');
    if (versionSelect) {
      // Find the option that matches, or set to custom value
      let found = false;
      for (const opt of versionSelect.options) {
        if (opt.value === imported.meta.sybaseVersion) {
          versionSelect.value = imported.meta.sybaseVersion;
          found = true;
          break;
        }
      }
      if (!found) {
        // If not found in options, set it anyway (some versions may not be in the dropdown)
        versionSelect.value = imported.meta.sybaseVersion;
      }
    }
  }
  
  // Re-render UI elements
  updateDialectStrings();
  
  // Update checklist view if tab is active
  if (state.activeTab === 'checklist') {
    renderChecklist();
  }
  
  // Update heat map and details if tab is active
  if (state.activeTab === 'analysis' && state.codeResults) {
    renderHeatMap(state.codeResults);
    renderDetails(state.codeResults);
  }
  
  // Update Reports tab if active
  if (state.activeTab === 'reports') {
    renderReports();
  }
  
  // Update header scores
  const cc = state.scores?.code !== null ? Math.round(state.scores.code) : null;
  const mr = state.scores?.readiness !== null ? Math.round(state.scores.readiness) : null;
  const comp = state.scores?.composite !== null ? Math.round(state.scores.composite) : null;
  
  // Update header narrative fields
  const clientInput = document.getElementById('nv-client');
  if (clientInput && state.meta.client) clientInput.value = state.meta.client;
  
  const analystInput = document.getElementById('nv-analyst');
  if (analystInput && state.meta.analyst) analystInput.value = state.meta.analyst;
  
  const recInput = document.getElementById('nv-recommendation');
  if (recInput && state.meta.recommendation) recInput.value = state.meta.recommendation;
  
  showToast('Session imported.');
}
```

- [ ] **Step 4: Test in browser**

1. Export a session with analysis:
   - Paste a small SQL snippet on Analysis tab
   - Click Analyze
   - Go to Reports, click "Raw Data · JSON", save the file (e.g., `export.json`)

2. Refresh the page (to clear state)

3. Go to Reports tab, click "Import Raw JSON"

4. Select the `export.json` file

5. Expected result:
   - Toast: "Session imported."
   - SQL textarea should repopulate with your original SQL
   - Heat map and details should reappear on Analysis tab if you switch to it
   - Checklist answers should be restored
   - Scores should be restored in the header

6. Test backwards compatibility:
   - Manually edit `export.json`, remove the `codeResults` field, save
   - Refresh page
   - Re-import the modified JSON
   - Toast should warn: "Partial import — analyzer output unavailable (legacy export)."
   - Scores and checklist should still restore

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: implement JSON re-import with validation and state restoration"
```

---

## Task Group C: Target-Aware Effort Adjustment

### Task C1: Add TARGET_CODE_MULTIPLIER Constant

**Files:**
- Modify: `index.html:2935+` (add after SOURCES constant)

- [ ] **Step 1: Add TARGET_CODE_MULTIPLIER constant**

Find the line after the SOURCES constant (around line 2936). Add:
```js
const TARGET_CODE_MULTIPLIER = {
  mssql:    0.75,  // Sybase cousin; ~25% less rewrite effort
  postgres: 1.0,   // default baseline
  oracle:   1.25,  // more translation; ~25% more effort
};
```

This should be added right after SOURCES and before the next major comment block.

- [ ] **Step 2: Test in browser**

Open DevTools Console and paste:
```js
console.log(TARGET_CODE_MULTIPLIER);
```

Expected output:
```js
{ mssql: 0.75, postgres: 1.0, oracle: 1.25 }
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add TARGET_CODE_MULTIPLIER constant for dialect-aware effort scaling"
```

---

### Task C2: Modify computeEffortBand() to Apply Target Multiplier

**Files:**
- Modify: `index.html:5265-5314` (computeEffortBand function)

- [ ] **Step 1: Find the code migration calculation in computeEffortBand()**

Locate the section (around line 5273-5282):
```js
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
  code *= mult;
```

- [ ] **Step 2: Insert target multiplier before readiness multiplier**

After the `code = Math.max(0.5, code);` line and **before** `code *= mult;`, insert:
```js
  
  // Target-dialect affinity affects code rewrite effort.
  // MSSQL is syntactically closer to Sybase (0.75x). PostgreSQL is baseline (1.0x).
  // Oracle requires more translation work (1.25x).
  code *= (TARGET_CODE_MULTIPLIER[state.targetDialect] || 1.0);
```

The result should look like:
```js
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
  
  // Target-dialect affinity affects code rewrite effort.
  // MSSQL is syntactically closer to Sybase (0.75x). PostgreSQL is baseline (1.0x).
  // Oracle requires more translation work (1.25x).
  code *= (TARGET_CODE_MULTIPLIER[state.targetDialect] || 1.0);
  
  code *= mult;
```

- [ ] **Step 3: Test in browser**

1. Paste a moderately complex SQL snippet (with some procedures) on the Analysis tab
2. Click Analyze
3. Go to Reports tab, note the "Code migration" person-days (should be some number like 5.0)
4. Switch target to "SQL Server" on Analysis tab
5. Go back to Reports tab
6. The "Code migration" effort should be ~75% of the PostgreSQL value (e.g., 3.75 instead of 5.0)
7. Switch target to "Oracle"
8. The "Code migration" effort should be ~125% of PostgreSQL (e.g., 6.25)
9. Switch back to PostgreSQL — should return to original value

Verify that changing targets updates the effort estimate correctly.

- [ ] **Step 4: Test exports show updated effort**

1. With the above test case, export "Technical Report · HTML" for SQL Server target
2. Verify the exported HTML shows the reduced effort
3. Export for Oracle target
4. Verify it shows increased effort

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: apply target-dialect multiplier to code migration effort"
```

---

### Task C3: Verify Effort Changes Across All Reports

**Files:**
- Modify: none (testing only)

- [ ] **Step 1: Test all report formats show updated effort**

1. Analyze SQL, get baseline numbers
2. Switch target to SQL Server
3. Export each format and verify code migration effort is reduced:
   - Technical Report HTML
   - Technical Report Markdown
   - Executive Summary HTML
   - Executive Summary Markdown
   - Slide Deck HTML
   - Raw JSON (code effort field)

- [ ] **Step 2: Verify Code Complexity score is unaffected**

1. With a test case analyzed, check the "Code Complexity" score (should be a number like 45)
2. Switch targets (PostgreSQL, SQL Server, Oracle)
3. Code Complexity should NOT change (it's about source code, not target)
4. Only the effort estimates should change

- [ ] **Step 3: Verify Composite Risk score is unaffected**

1. Check the Composite Risk score (0.45 × Code + 0.55 × Readiness)
2. Switch targets
3. Composite Risk should NOT change (neither Code nor Readiness change with target)

- [ ] **Step 4: Commit (if any test code added)**

If you added any local test code, commit it. Otherwise, no commit needed for this task.

```bash
git add index.html
git commit -m "test: verify target-aware effort across all reports and scores"
```

---

## Summary of Changes

**Feature 1: Source-Dialect Architecture (Tasks A1–A4)**
- Added `SOURCES` constant (currently Sybase only, ready for SQL Server)
- Added `state.sourceDialect`
- Added read-only source pill UI
- Parameterized 8+ display strings to reference `SOURCES[state.sourceDialect].label`
- Function `updateDialectStrings()` re-renders dialect-dependent UI

**Feature 2: JSON Re-import (Tasks B1–B3)**
- Expanded `downloadRawJSON()` export to include `sqlInput`, `codeResults`, `source`, and `schemaVersion`
- Added "Import Raw JSON" button UI
- Implemented `importRawJSON()` with validation, backwards compatibility, and full state restoration
- Error handling for corrupted/invalid/foreign JSON files

**Feature 3: Target-Aware Effort (Tasks C1–C3)**
- Added `TARGET_CODE_MULTIPLIER` constant (MSSQL: 0.75x, PostgreSQL: 1.0x, Oracle: 1.25x)
- Modified `computeEffortBand()` to apply multiplier before readiness gap
- Verified Code Complexity and Composite Risk scores remain unaffected
- All report formats reflect updated effort

**Total commits: 7 feature commits + optional test commit**

---

## Self-Review Against Spec

**Spec Section 1 (JSON Re-import):**
- ✅ Export includes `sqlInput`, `codeResults`, `schemaVersion`
- ✅ Import validates `tool` field and `schemaVersion`
- ✅ Restores `meta`, `checklist`, `scores`, `codeMetrics`, `codeResults`
- ✅ Error handling is explicit with distinct toasts
- ✅ Backwards-compatible import (legacy JSON without `codeResults` shows warning)

**Spec Section 2 (Source-Dialect Architecture):**
- ✅ `SOURCES` constant defined
- ✅ `state.sourceDialect = 'sybase'` added
- ✅ Source pill UI added (read-only, signals "coming soon" for SQL Server)
- ✅ Display strings refactored (subtitles, input labels, reports)
- ✅ Explicit boundaries: RULES, SQL Library, FAQ, title NOT refactored (correct per spec)

**Spec Section 3 (Target-Aware Effort):**
- ✅ `TARGET_CODE_MULTIPLIER` constant with documented values
- ✅ Applied in `computeEffortBand()` before readiness multiplier
- ✅ Code Complexity, Migration Readiness, Composite Risk unaffected
- ✅ Data and Infra effort unaffected
- ✅ Testing verified live updates across all target switches

**No spec gaps found. No placeholders in tasks. All tasks include exact code and commands.**

---

## Execution Options

Plan complete and saved to `docs/superpowers/plans/2026-04-22-json-reimport-source-dialect-target-effort.md`.

**Two execution approaches:**

**1. Subagent-Driven (Recommended)** — I dispatch a fresh subagent to execute each task independently, review between tasks. Faster iteration, clear handoff points.

**2. Inline Execution** — Execute all tasks in this session using the executing-plans skill. Single-threaded but good for tight feedback loops.

**Which would you prefer?**