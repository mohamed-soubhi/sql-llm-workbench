# SQL + LLM Workbench

Learn **SQL** and how **LLMs turn plain English into SQL** — using a real, live
database that runs entirely in your browser.

No backend. No install. Open one file and you have a SQLite database, a SQL editor,
and an AI analyst that answers questions about your data.

> **[▶ Open the workbench](sql-workbench.html)** · **[📖 Architecture & help](help.html)**

---

## What's in the box

| File | What it is |
|---|---|
| `sql-workbench.html` | The whole app — database, data, UI, AI, all in one file. |
| `help.html` | Architecture, workflow, and code map. |
| `index.html` | Project page (the deployed tutorial site). |
| `Script_Create_Tables.sql.txt` | The 5-table schema. |
| `*.csv` | The raw HR dataset (headerless). |

The database holds 5 tables and 36 rows:

| Table | Rows | Key columns |
|---|---|---|
| `EMPLOYEES` | 10 | `EMP_ID`, `F_NAME`, `L_NAME`, `SALARY`, `JOB_ID`, `DEP_ID` |
| `JOBS` | 10 | `JOB_IDENT`, `JOB_TITLE`, `MIN_SALARY`, `MAX_SALARY` |
| `JOB_HISTORY` | 10 | `EMPL_ID`, `START_DATE`, `JOBS_ID`, `DEPT_ID` |
| `DEPARTMENTS` | 3 | `DEPT_ID_DEP`, `DEP_NAME`, `MANAGER_ID`, `LOC_ID` |
| `LOCATIONS` | 3 | `LOCT_ID`, `DEP_ID_LOC` |

---

## Quickstart

**Just want SQL?** Open `sql-workbench.html` in a browser. Done — the SQL editor and
database work immediately, offline.

**Want the AI too?** The English→SQL mode talks to [Ollama](https://ollama.com) on
your machine. Browsers block cross-origin calls by default, so pick one:

```bash
# Option A — let the browser talk to Ollama, then open the file directly
OLLAMA_ORIGINS='*' ollama serve

# Option B — serve the page, then browse to it
python3 -m http.server 8000
# → http://localhost:8000/sql-workbench.html

# Check the model name matches one you have:
ollama list
```

Set the model in the top bar (default `deepseek-v4.1-flash:cloud`).

---

# Part 1 — SQL, from zero

Every example below runs in the **Write SQL** tab against the real data. Type it,
press `Ctrl`+`Enter`, read the table that comes back.

### 1. Read a table

```sql
SELECT * FROM EMPLOYEES;
```

`SELECT` picks columns, `*` means "all of them", `FROM` names the table. You get 10 rows.

### 2. Pick columns and rename them

```sql
SELECT F_NAME, L_NAME, SALARY FROM EMPLOYEES;
```

### 3. Filter rows with WHERE

```sql
SELECT F_NAME, L_NAME, SALARY
FROM EMPLOYEES
WHERE SALARY > 70000;
```

Comparisons: `=`, `>`, `<`, `>=`, `<=`, `<>` (not equal). Combine with `AND` / `OR`:

```sql
SELECT F_NAME, L_NAME, SEX, SALARY
FROM EMPLOYEES
WHERE SEX = 'F' AND SALARY >= 70000;
```

### 4. Sort and limit

```sql
SELECT F_NAME || ' ' || L_NAME AS name, SALARY
FROM EMPLOYEES
ORDER BY SALARY DESC
LIMIT 5;
```

`||` joins text in SQLite. `AS` names the result column. `ORDER BY ... DESC` sorts
high→low; `LIMIT` caps the rows.

**Result**

| name | SALARY |
|---|---|
| John Thomas | 100000 |
| Nancy Allen | 90000 |
| Alice James | 80000 |
| Ahmed Hussain | 70000 |
| Andrea Jones | 70000 |

### 5. Aggregate — count, average, min, max

```sql
SELECT COUNT(*) AS employees,
       ROUND(AVG(SALARY), 0) AS avg_salary,
       MIN(SALARY) AS lowest,
       MAX(SALARY) AS highest
FROM EMPLOYEES;
```

An aggregate collapses many rows into one number.

### 6. Group — one row per category

```sql
SELECT SEX, COUNT(*) AS n
FROM EMPLOYEES
GROUP BY SEX;
```

`GROUP BY` splits rows into buckets, then the aggregate runs per bucket. `5 F`, `5 M`.

### 7. Join — combine two tables

This is the heart of relational SQL. `EMPLOYEES.DEP_ID` points at
`DEPARTMENTS.DEPT_ID_DEP`, so we can bring in the department name:

```sql
SELECT d.DEP_NAME,
       COUNT(*) AS people,
       ROUND(AVG(e.SALARY), 0) AS avg_salary
FROM EMPLOYEES e
JOIN DEPARTMENTS d ON e.DEP_ID = d.DEPT_ID_DEP
GROUP BY d.DEP_NAME
ORDER BY avg_salary DESC;
```

**Result**

| DEP_NAME | people | avg_salary |
|---|---|---|
| Architect Group | 3 | 86667.0 |
| Design Team | 3 | 66667.0 |
| Software Group | 4 | 65000.0 |

The `e` and `d` are **aliases** — shorthand for the table names. `JOIN ... ON`
states the matching rule.

### 8. Join three tables

```sql
SELECT e.F_NAME || ' ' || e.L_NAME AS name,
       j.JOB_TITLE,
       e.SALARY
FROM EMPLOYEES e
JOIN JOBS j ON e.JOB_ID = j.JOB_IDENT
WHERE e.SALARY > 70000
ORDER BY e.SALARY DESC;
```

**Result**

| name | JOB_TITLE | SALARY |
|---|---|---|
| John Thomas | Sr. Architect | 100000 |
| Nancy Allen | Lead Architect | 90000 |
| Alice James | Sr. Software Developer | 80000 |

### The join map

```
EMPLOYEES.DEP_ID   ─┐
                    ├─► DEPARTMENTS.DEPT_ID_DEP
JOB_HISTORY.DEPT_ID ┘

EMPLOYEES.JOB_ID     ──► JOBS.JOB_IDENT
JOB_HISTORY.JOBS_ID  ──► JOBS.JOB_IDENT
JOB_HISTORY.EMPL_ID  ──► EMPLOYEES.EMP_ID
DEPARTMENTS.LOC_ID   ──► LOCATIONS.LOCT_ID
```

---

# Part 2 — How an LLM writes SQL

The **Ask** tab takes a question like *"who earns the most, with their department?"*
and turns it into the query you'd have written by hand. Here's the whole mechanism —
it's simpler than it looks.

### The pipeline

```
your question
      │
      ▼
  build a prompt  ──  schema + glossary + rules + examples + question
      │
      ▼
  call the model   ──  Ollama, temperature 0
      │
      ▼
  extract the SQL  ──  strip prose/fences, keep the SELECT
      │
      ▼
  read-only guard  ──  refuse anything that isn't SELECT/WITH/EXPLAIN
      │
      ▼
  run it in SQLite ──  show the SQL + the result table
```

### The four ingredients of a good NL→SQL prompt

1. **Schema** — the actual `CREATE TABLE` statements. The model must see real table
   and column names; guessing produces wrong identifiers.
2. **Glossary** — the semantics the schema can't express. *"Employee full name is
   `F_NAME || ' ' || L_NAME`"*, *"to attach a department, join `DEP_ID` to
   `DEPT_ID_DEP`"*. This is what makes joins work.
3. **Rules** — constrain the output: *"Output ONLY the SQL. One SELECT. No DDL/DML."*
4. **Examples** — two or three question→SQL pairs. Few-shot examples teach style and
   shape far better than instructions alone.

### Why grounding beats cleverness

A model knows SQL. What it doesn't know is *your* schema. Nearly every bad generated
query is a naming failure — it invents a column, or joins on the wrong key. Feed it
the exact DDL and the join keys and the error rate collapses. That's the single
highest-leverage thing you can do.

### Guardrails

Generated SQL is untrusted input. The app only accepts statements starting with
`SELECT` / `WITH` / `EXPLAIN` and containing no `INSERT`, `UPDATE`, `DELETE`, `DROP`,
`ALTER`, `CREATE`, `REPLACE`, `TRUNCATE`, or `ATTACH`. Anything else is shown but
**not run**. On top of that, the database is in-memory — reloading resets it — so even
a mistake can't touch your files.

### Failure modes and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `no such column` | Model guessed a name | Tighten the schema block; list columns explicitly |
| Wrong/missing join | Join keys not stated | Add the join to the glossary |
| Prose instead of SQL | Prompt not strict enough | "Output ONLY the SQL" + few-shot |
| Two statements | Ambiguous question | Ask narrower; keep one SELECT |
| Slow / odd result | Model hallucinated rows | Set `temperature: 0` (already done) |

### Try these questions in the Ask tab

- *How many employees are there?*
- *List the top 3 salaries with full names.*
- *Which department has the highest average salary?*
- *Show female employees earning more than 70,000.*
- *What is the pay range for a Sr. Architect?*

Watch the generated SQL appear above each answer — that's the model showing its work.

---

# Part 3 — Exercises

Try each in the **Write SQL** tab. Answers are one obvious variation of Part 1.

1. List every employee's full name and salary, cheapest first.
2. How many people work in the Software Group?
3. Show the highest-paid person in each department.
4. Which job titles pay more than 90,000 at the top?
5. Find employees hired before 2001 (hint: `JOB_HISTORY.START_DATE`, stored as
   `YYYY-MM-DD` text — string comparison sorts correctly).
6. Count how many distinct departments appear in `JOB_HISTORY`.

---

# Architecture

```
┌─ sql-workbench.html (browser) ──────────────────────┐
│                                                      │
│   UI layer      tabs · chat · editor · result grid   │
│        │                                             │
│        ▼                                             │
│   sql.js        SQLite compiled to WebAssembly       │
│        ▲                                             │
│        │                                             │
│   data          DDL + 5 CSVs embedded as strings     │
│                                                      │
└──────────────────────────────────────────────────────┘
        │                            │
        ▼                            ▼
  CDN (sql.js)              Ollama :11434  (Ask mode only)
```

**Two paths, one engine.** Direct SQL reaches SQLite straight. Ask goes through the
model first, then the same engine. If Ollama is down, Direct mode still works.

Full details, flow diagrams, and a line-by-line code map are in
**[help.html](help.html)**.

---

# Deploy (GitHub Pages)

This repo is a static site — Pages serves it with no build step.

1. Push the repo to GitHub.
2. **Settings → Pages → Source: Deploy from a branch → `main` / `/ (root)`**.
3. Wait a minute. The site is at `https://<user>.github.io/sql-llm-workbench/`.

`index.html` is the landing/tutorial page; `sql-workbench.html` and `help.html` are
served alongside it.

> The AI mode cannot run on the *deployed* site — a browser page on `github.io`
> can't reach your local Ollama. Pages hosts the **tutorial and the SQL editor**;
> open the file locally for the full English→SQL experience.

---

## License

MIT — use it, fork it, teach with it.
