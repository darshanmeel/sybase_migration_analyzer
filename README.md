# Sybase T-SQL Migration Analyzer

A single-file, fully offline HTML tool for analyzing Sybase T-SQL code and estimating complexity for migration to PostgreSQL.

## What It Does

Open `sybase_analyzer.html` in any modern browser (Chrome, Edge, Firefox, Safari — last 2 major versions). No installation, no server, no internet connection required.

The tool:
- Splits Sybase T-SQL input into individual database objects (stored procedures, views, functions, triggers)
- Runs each object through a rule engine of 40+ Sybase-specific patterns
- Scores each object and assigns a complexity bucket (Low / Medium / High / Very High)
- Produces a heat map table and per-object detail reports with color-coded, annotated source
- Exports the full report as a self-contained HTML file or a clean Markdown file

## How to Use

1. Open `sybase_analyzer.html` directly in your browser (double-click or drag into the browser window).
2. Paste Sybase T-SQL code into the large textarea, or drag-and-drop a `.sql` or `.txt` file onto the upload zone.
3. Click **Analyze**.
4. Review the summary banner, the sortable heat map, and the per-object detail sections.
5. In each detail section, use the **Full Annotation** tab to see color-highlighted source code. Hover over any highlight for a tooltip showing the rule ID, migration note, and PostgreSQL equivalent.
6. Click any row in the heat map to jump directly to that object's detail section.
7. When done, use the **Download Report as HTML** or **Download Report as Markdown** buttons in the footer.

## Input Requirements

**This tool is designed for well-formatted, standard Sybase T-SQL files.**

For accurate results, input should be:
- Clean, properly formatted Sybase T-SQL (stored procedures, views, functions, triggers)
- Standard Sybase ASE / SQL Server T-SQL syntax
- Files with `GO` batch terminators separating objects (standard Sybase script format)

Input that will produce inaccurate or incomplete results:
- SQL embedded inside application code (Java strings, Python variables, etc.)
- Raw export dumps mixed with non-SQL content (binary data, shell commands, etc.)
- Heavily obfuscated or minified SQL
- SQL from other databases (Oracle PL/SQL, MySQL, etc.) — patterns are tuned for Sybase

Messy input will cause the object splitter to misidentify boundaries and the pattern engine to produce false positives or miss real issues.

## Understanding the Output

### Complexity Buckets

| Bucket | Score Range | Estimated Effort |
|---|---|---|
| Low | 0–3 | ~0.5 day |
| Medium | 4–8 | 1–2 days |
| High | 9–15 | 3–5 days |
| Very High | 16+ | 5–10+ days |

Scores combine a rules-based score (each matched pattern contributes points) with a lines-of-code component. These estimates are guidelines, not guarantees.

### Key Categories Detected

| Category | Examples |
|---|---|
| Sybase Functions | ISNULL, GETDATE, CONVERT, DATEPART, DATEADD, DATEDIFF, CHARINDEX |
| Sybase Global Variables | @@identity, @@rowcount, @@error, @@trancount |
| Dynamic SQL | EXEC(), sp_executesql |
| Cross-Database References | dbname..tablename |
| Locking Hints | HOLDLOCK, READPAST, WITH (NOLOCK) |
| Cursors | DECLARE ... CURSOR |
| Control Flow | GOTO |
| Data Types | MONEY, TINYINT, TEXT, IMAGE, TIMESTAMP |

ANSI-standard patterns (COALESCE, CASE WHEN, CAST, standard JOINs) are detected and annotated with zero score — they are portable and need no changes.

## Limitations

1. **Regex-based only.** This tool uses pattern matching, not a real SQL parser. It cannot understand nesting, scope, comments, or string literals. Results are a triage aid, not a substitute for manual review.
2. **Rubric scoring.** Two objects with the same score can differ dramatically in actual migration effort. The score captures syntax complexity, not business logic complexity.
3. **No semantic analysis.** The tool cannot tell whether a CURSOR is actually necessary, whether a GOTO is inside a complex flow, or whether dynamic SQL is high-risk or trivial.
4. **Privacy-safe.** No data leaves your browser. Nothing is uploaded, sent to any server, or stored anywhere. The tool has no network calls of any kind.
5. **Rewrite suggestions are illustrative.** Any suggested PostgreSQL equivalents shown in tooltips or reports are starting points only and are not production-ready rewrites.

## No External Dependencies

`sybase_analyzer.html` is a single self-contained file. All CSS and JavaScript are inline. It works completely offline with no CDN, no npm, no build step, and no server. Copy the file anywhere and open it.
