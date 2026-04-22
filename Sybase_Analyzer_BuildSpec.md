# Sybase Analyzer — Build Specification

## What to Build

A **single-file HTML tool** (`sybase_analyzer.html`) that runs locally in any modern browser (no server, no install, no network calls). The user pastes Sybase T-SQL code or uploads a `.sql` file, and the tool analyzes it and produces a structured complexity and migration report targeting PostgreSQL.

## Hard Constraints

1. **One single `.html` file.** All CSS and JavaScript inline — no external files, no CDN dependencies, no `<script src="...">` to anywhere.
2. **Fully offline.** No `fetch`, no network calls, nothing that pings the internet. Everything runs client-side.
3. **No `localStorage`, `sessionStorage`, or any browser storage.** Fresh session every time. This is also privacy-clean — nothing the user pastes is ever persisted.
4. **Modern browser only** (Chrome, Edge, Firefox, Safari — last 2 major versions). Use modern JS (ES2020+).
5. **Pure regex-based parsing.** No real SQL parser. Be explicit about this limitation in the UI.

## Core Features

### 1. Input (both paste and file upload)

1. A large `<textarea>` for pasting code.
2. A file input / drag-drop zone accepting `.sql`, `.txt` files (single file per session).
3. When a file is uploaded, its contents populate the textarea (so the user can edit before analyzing).
4. A big "Analyze" button triggers the analysis.
5. A "Clear" button resets everything.

### 2. Object Splitter

Split the input into individual database objects. Detect these object types:

1. Stored procedures — `CREATE PROCEDURE`, `CREATE PROC`
2. Views — `CREATE VIEW`
3. Functions — `CREATE FUNCTION`
4. Triggers — `CREATE TRIGGER`

**Splitting rules:**
1. Case-insensitive matching on the `CREATE ...` keywords.
2. Handle `CREATE OR REPLACE` and Sybase's `IF EXISTS ... DROP ... CREATE` patterns.
3. Respect the `go` batch terminator (case-insensitive, on its own line) as a boundary.
4. Also respect `/` on its own line as a terminator (Oracle-style, sometimes seen in Sybase scripts).
5. If the splitter finds no recognizable objects, treat the whole input as **one anonymous block** and analyze it anyway.

### 3. Rule Engine

A single JavaScript array of rule objects. Each rule has this shape:

```javascript
{
  id: 'SYB-001',
  name: 'ISNULL function',
  category: 'sybase_function',       // see categories below
  severity: 'low' | 'medium' | 'high',
  score: 1,                           // points added to complexity score
  pattern: /\bisnull\s*\(/i,          // regex
  postgres: 'coalesce(...)',          // PostgreSQL equivalent
  note: 'ISNULL is Sybase-specific. Use ANSI COALESCE for portability.',
  rewrite: (match) => match.replace(/isnull/i, 'coalesce')  // optional simple rewrite
}
```

**Categories (used for grouping in the report):**

1. `sybase_function` — Sybase-specific built-ins (isnull, convert, datepart, etc.)
2. `sybase_global` — `@@identity`, `@@rowcount`, `@@error`, `@@trancount`, `@@servername`, `@@version`
3. `dynamic_sql` — `exec(`, `execute(`, `sp_executesql` with string args
4. `cross_database` — `dbname..tablename` pattern
5. `temp_table` — `#tablename` pattern (local temp)
6. `locking_hint` — `holdlock`, `noholdlock`, `readpast`, `shared`, `exclusive`, `with (nolock)`
7. `cursor` — `declare ... cursor`, `open`, `fetch`, `close`, `deallocate`
8. `control_flow` — `goto`, labels, `if ... begin ... end`, `while`
9. `error_handling` — `raiserror`, `print`
10. `data_type` — Sybase-specific types: `money`, `smallmoney`, `text`, `image`, `unitext`, `timestamp`, `tinyint`
11. `syntax` — `top n`, `+` for concat, `select into`, `insert exec`
12. `ansi_good` — things that ARE portable (coalesce, cast, standard joins) — reported for context, zero score
13. `identity` — `identity` column, `set identity_insert`, `@@identity`

**Rules to include (minimum set — add more if obvious):**

| ID | Pattern | Category | Severity | Score | PostgreSQL equivalent |
|---|---|---|---|---|---|
| SYB-001 | `\bisnull\s*\(` | sybase_function | low | 1 | `coalesce(...)` |
| SYB-002 | `\bgetdate\s*\(\s*\)` | sybase_function | low | 1 | `now()` or `current_timestamp` |
| SYB-003 | `\bconvert\s*\(` | sybase_function | medium | 2 | `cast(... as ...)` or `to_char(...)` with format |
| SYB-004 | `\bdatepart\s*\(` | sybase_function | medium | 2 | `extract(... from ...)` |
| SYB-005 | `\bdateadd\s*\(` | sybase_function | medium | 2 | `+ interval '...'` |
| SYB-006 | `\bdatediff\s*\(` | sybase_function | medium | 2 | `age(...)` or subtract timestamps |
| SYB-007 | `\bcharindex\s*\(` | sybase_function | low | 1 | `position(... in ...)` or `strpos` |
| SYB-008 | `\bpatindex\s*\(` | sybase_function | medium | 2 | Regex functions (`~`, `substring`) |
| SYB-009 | `\bstuff\s*\(` | sybase_function | medium | 2 | `overlay(...)` |
| SYB-010 | `\bstr\s*\(` | sybase_function | low | 1 | `to_char(...)` |
| SYB-020 | `@@identity\b` | sybase_global | high | 3 | `currval(seq)` or `RETURNING` clause |
| SYB-021 | `@@rowcount\b` | sybase_global | medium | 2 | `GET DIAGNOSTICS n = ROW_COUNT;` |
| SYB-022 | `@@error\b` | sybase_global | high | 3 | `EXCEPTION` blocks in PL/pgSQL |
| SYB-023 | `@@trancount\b` | sybase_global | high | 3 | No direct equivalent — redesign transaction logic |
| SYB-024 | `@@servername\b` | sybase_global | low | 1 | `inet_server_addr()` / config |
| SYB-030 | `\bexec(?:ute)?\s*\(` | dynamic_sql | high | 5 | `EXECUTE ...` in PL/pgSQL |
| SYB-031 | `\bsp_executesql\b` | dynamic_sql | high | 5 | `EXECUTE ... USING` in PL/pgSQL |
| SYB-040 | `\b\w+\s*\.\.\s*\w+\b` | cross_database | high | 3 | FDW (`postgres_fdw`) or schema consolidation |
| SYB-050 | `#\w+\b` | temp_table | medium | 1 | `CREATE TEMP TABLE ...` (semantics differ) |
| SYB-060 | `\bholdlock\b` | locking_hint | medium | 2 | `SELECT ... FOR UPDATE` or isolation level |
| SYB-061 | `\bnoholdlock\b` | locking_hint | medium | 2 | Default PostgreSQL MVCC |
| SYB-062 | `\breadpast\b` | locking_hint | high | 2 | `SKIP LOCKED` (PostgreSQL 9.5+) |
| SYB-063 | `\bwith\s*\(\s*nolock\s*\)` | locking_hint | high | 2 | No equivalent; use `READ COMMITTED` isolation |
| SYB-070 | `\bdeclare\s+\w+\s+cursor\b` | cursor | high | 3 | PL/pgSQL cursors — syntax differs |
| SYB-080 | `\bgoto\s+\w+` | control_flow | high | 3 | PL/pgSQL has no GOTO — redesign |
| SYB-081 | `\braiserror\b` | error_handling | medium | 1 | `RAISE EXCEPTION` |
| SYB-082 | `\bprint\b` | error_handling | low | 1 | `RAISE NOTICE` |
| SYB-090 | `\bmoney\b|\bsmallmoney\b` | data_type | medium | 1 | `numeric(19,4)` |
| SYB-091 | `\btext\b(?!\s*=)|\bimage\b` | data_type | medium | 2 | `text` / `bytea` |
| SYB-092 | `\btinyint\b` | data_type | low | 1 | `smallint` |
| SYB-093 | `\bunitext\b` | data_type | medium | 1 | `text` with UTF-8 |
| SYB-094 | `\btimestamp\b` | data_type | medium | 2 | Redesign (Sybase timestamp ≠ PG timestamp) |
| SYB-100 | `\btop\s+\d+` | syntax | low | 1 | `LIMIT n` |
| SYB-101 | `\bselect\s+.+\binto\s+#\w+` | syntax | medium | 2 | `CREATE TEMP TABLE ... AS SELECT ...` |
| SYB-102 | `\binsert\s+.+\bexec\b` | syntax | high | 3 | Rewrite using CTE or function returning table |
| SYB-110 | `\bidentity\b` | identity | medium | 2 | `GENERATED ... AS IDENTITY` |
| SYB-111 | `\bset\s+identity_insert\b` | identity | high | 3 | `OVERRIDING SYSTEM VALUE` clause |

**Positive (ANSI) rules for context — zero score, shown as "good patterns":**

| ID | Pattern | Category | Note |
|---|---|---|---|
| ANSI-001 | `\bcoalesce\s*\(` | ansi_good | ANSI-standard; portable to PostgreSQL as-is |
| ANSI-002 | `\bcase\s+when\b` | ansi_good | Portable |
| ANSI-003 | `\bcast\s*\(.+\bas\b` | ansi_good | Portable |
| ANSI-004 | `\b(inner|left|right|full)\s+(outer\s+)?join\b` | ansi_good | Standard joins — portable |

### 4. Scoring Rubric

For each object, sum the scores from matched rules plus LOC-based points:

**LOC points:**
1. `< 50 lines` → 0
2. `50–200 lines` → 2
3. `200–500 lines` → 4
4. `500–1500 lines` → 7
5. `> 1500 lines` → 12

**Bucketing:**

| Total score | Bucket | Estimated rewrite effort | Color |
|---|---|---|---|
| 0–3 | Low | 0.5 day | green |
| 4–8 | Medium | 1–2 days | yellow |
| 9–15 | High | 3–5 days | orange |
| 16+ | Very High | 5–10+ days | red |

### 5. Analysis Output — Interactive View

Render the report directly in the page, styled cleanly. Structure:

#### Top summary banner
1. Filename or "Pasted input"
2. Timestamp of analysis
3. Total objects found (broken down by type: procs, views, functions, triggers)
4. Bucket breakdown counts: Low / Medium / High / Very High
5. Total estimated effort (sum of days)
6. Total lines of code analyzed

#### Heat map table
Sortable by any column. Columns:
1. Object name
2. Object type (proc / view / function / trigger)
3. LOC
4. Total issues found
5. Sybase-specific count
6. Dynamic SQL (yes/no)
7. Cross-DB refs (yes/no)
8. Cursors (yes/no)
9. Score
10. Bucket (color-coded badge)
11. Estimated days

Click a row to jump to that object's detail section.

#### Per-object detail sections (collapsible)

For each object, render:

1. **Header:** object name, type, bucket badge, score, estimated days
2. **Summary tab:** bullet list of issues by category, counts, and key notes
3. **Full annotation tab:** source code displayed in a `<pre>` block with color-coded highlights on matched patterns. Hovering a highlight shows a tooltip with: category, note, PostgreSQL equivalent, (optional) rewrite suggestion
4. **Toggle** between Summary view and Full annotation view (user-chosen per object)

Color coding:
1. `sybase_function` — light orange background
2. `sybase_global` — red background
3. `dynamic_sql` — red background, bold
4. `cross_database` — purple background
5. `temp_table` — yellow background
6. `locking_hint` — red background
7. `cursor` — orange background
8. `control_flow` — yellow background
9. `error_handling` — blue background
10. `data_type` — light purple background
11. `syntax` — light yellow background
12. `identity` — pink background
13. `ansi_good` — light green background

### 6. Export / Download

Two download buttons:

1. **Download as HTML** — a self-contained `.html` file with the full report baked in (same styling, no interactivity needed — it's a static snapshot).
2. **Download as Markdown** — a clean `.md` file with the heat map as a markdown table and per-object details as sections. No source code in the Markdown version (too noisy) — just the findings.

Both should use a filename like `sybase_analysis_<YYYYMMDD_HHMM>.html` / `.md`.

### 7. "About / Limitations" Panel

Always visible (footer or collapsible section). Must contain:

1. "This tool uses regex-based pattern matching, not a real SQL parser. Results are a triage aid, not a substitute for manual review."
2. "Scoring is rubric-based. Two objects with identical scores may take very different times to rewrite in practice."
3. "Business logic complexity is not captured — the tool only analyzes syntax, not semantics."
4. "No data leaves your browser. Nothing is uploaded, logged, or stored."
5. "Rewrite suggestions (if shown) are illustrative only and are NOT production-ready."

## UI & Layout

Keep it clean and professional — this may be shown to an acquirer.

1. **Header:** tool name, short tagline, a small "?" help button
2. **Input panel (top):** textarea + file upload + Analyze/Clear buttons
3. **Summary banner (below input once analyzed):** the stats block
4. **Heat map table (middle):** sortable, color-coded
5. **Object detail sections (bottom):** collapsible, one per object
6. **Footer:** limitations panel, export buttons, version

Use a neutral, professional palette (dark navy header, white background, subtle borders). No emojis in the UI, no gimmicks. This is a tool for a consulting engagement.

Use a monospace font for code display (`ui-monospace`, `SFMono-Regular`, `Menlo`, `Consolas`, `monospace`).

## Implementation Notes for the Builder

1. Keep the rule array as a single, clean data structure at the top of the JS — easy to extend.
2. Escape HTML when rendering source code (don't let pasted SQL break the DOM).
3. When highlighting, walk the source once and emit `<span>`s with classes per category — don't do multiple passes with overlapping regexes or you'll get nested/broken spans. A reasonable approach: find all match ranges, sort by position, then render non-overlapping spans.
4. For the heat map sort, plain JS sort on table rows is fine — no need for a library.
5. For file upload, use `FileReader.readAsText()`.
6. For downloads, use a `Blob` + anchor click trick — no dependencies.
7. The whole file should be under ~1500 lines of code. Don't over-engineer.

## Acceptance Tests

The tool should correctly handle each of these test inputs:

1. A single stored proc with no Sybase-specific constructs → Low bucket, score 0–3.
2. A single stored proc with dynamic SQL and cross-database refs → High bucket.
3. A file with 5 procs separated by `go` → 5 objects in the heat map.
4. A file mixing procs, views, and functions → all types appear in the heat map with correct type badges.
5. An input that doesn't start with `CREATE` → analyzed as one anonymous block.
6. Input with SQL comments (`--` and `/* */`) → comments should NOT trigger rule matches (bonus if comments are stripped before matching; acceptable for v1 to leave them in).
7. Uploading a 500 KB `.sql` file → completes analysis in under 3 seconds.
8. Clicking "Download Markdown" → produces a valid `.md` file that renders cleanly in any markdown viewer.

## Out of Scope for v1

1. Real SQL parsing.
2. Actual automated rewriting (beyond the trivial per-rule suggestions in tooltips).
3. Diff view between Sybase source and suggested PostgreSQL.
4. Export to PDF.
5. Multi-file upload.
6. Saving state between sessions.
7. Oracle or SQL Server targets.

## Suggested Deliverable Format from Haiku

Just one file: `sybase_analyzer.html`. No README, no build step, no other files.
