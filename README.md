# NexusChat — Agentic Natural-Language-to-SQL Engine

> **Type a question in plain English → get a validated SQL query, the live result set, and a human-friendly answer.**
> Built on top of the Microsoft **AdventureWorks 2019** sample database, with a self-correcting LLM pipeline that drafts, executes, audits, and repairs its own queries before responding.

---

## Table of Contents

1. [What Is NexusChat?](#1-what-is-nexuschat)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Repository Layout](#3-repository-layout)
4. [The Two-Phase Engine (Pre-Plan + Post-Plan)](#4-the-two-phase-engine-pre-plan--post-plan)
5. [Pre-Plan — Building the Vector Knowledge Base](#5-pre-plan--building-the-vector-knowledge-base)
6. [Post-Plan — The Runtime 6-Step Agentic Pipeline](#6-post-plan--the-runtime-6-step-agentic-pipeline)
7. [The FastAPI Agent (HTTP Bridge)](#7-the-fastapi-agent-http-bridge)
8. [The Next.js UI](#8-the-nextjs-ui)
9. [Setup & Run](#9-setup--run)
10. [Connecting from the UI](#10-connecting-from-the-ui)
11. [Validation Benchmark — AdventureWorks 12-Question Test Bench](#11-validation-benchmark--adventureworks-12-question-test-bench)
12. [Troubleshooting](#12-troubleshooting)
13. [Tech Stack](#13-tech-stack)

---

## 1. What Is NexusChat?

NexusChat is a self-hosted **Natural-Language-to-SQL (NL2SQL)** engine built around three cooperating processes:

| Layer            | Runs On   | Purpose                                                                     |
| :--------------- | :-------- | :-------------------------------------------------------------------------- |
| **Next.js UI**   | Port 3000 | Chat interface, settings panel, execution-blueprint inspector, history page |
| **Python Agent** | Port 8000 | FastAPI bridge — runs the AI pipeline and calls SQL Server via Docker       |
| **SQL Server**   | Port 1433 | Docker container (`mcr.microsoft.com/mssql/server:2022-latest`) with AdventureWorks loaded |

The system is **zero-static** — nothing about the schema is hardcoded. The AI engine discovers tables, joins, and business semantics from the live database at pre-plan time, embeds everything into a vector store, and then consults that store at runtime.

---

## 2. High-Level Architecture

```
┌──────────────────────┐
│  Browser (Next.js)   │  http://localhost:3000
│  - Chat              │
│  - Settings          │
│  - History           │
└──────────┬───────────┘
           │  fetch() → JSON
           ▼
┌──────────────────────┐
│  FastAPI Agent       │  http://localhost:8000
│  agent/main.py       │
│  - /health           │
│  - /connect          │
│  - /query            │
│  - /services/*       │
└──────────┬───────────┘
           │  (subprocess) docker exec sqlcmd
           ▼
┌──────────────────────┐
│  SQL Server (Docker) │  localhost:1433
│  AdventureWorks DB   │
│  sqlserver_data vol  │  ← persistent named volume
└──────────────────────┘

   OpenAI GPT-4o / text-embedding-3-small
   is called from the Python process for:
   - refinement
   - embedding
   - SQL generation
   - critique / audit
   - final NL synthesis
```

---

## 3. Repository Layout

```
NexusChat/
├── README.md                      # ← You are here
├── HOW_TO_RUN.md                  # Step-by-step setup guide (first-time + daily)
├── Dockerfile                     # Python agent image (msodbcsql18 + deps)
├── docker-compose.yml             # Defines sqlserver + nexus-agent services
├── requirements.txt               # fastapi, uvicorn, openai, pyodbc, numpy, pandas
├── .env                           # OPENAI_API_KEY, DB_SERVER, DB_USER, DB_PASSWORD
│
├── app/                           # Core AI engine (imported by the agent)
│   ├── main.py                    # CLI entry (interactive / --query / --setup)
│   ├── preplanned_main.py         # Entry to run the offline pipeline
│   ├── postplanned_main.py        # Entry to run the runtime pipeline
│   │
│   ├── preplan/                   # OFFLINE — builds the vector knowledge base
│   │   ├── schema_pipeline.py     # Extract → Analyze → Embed (3 idempotent steps)
│   │   ├── doc/README.md          # Deep-dive docs for pre-plan
│   │   └── output/                # Auto-generated artifacts
│   │       ├── database_schema.txt     # Tables, columns, FKs, 2-row samples
│   │       ├── database_analysis.md    # GPT-4o narrative per table
│   │       └── database_vectors.json   # 1536-dim embeddings + metadata (~3.8 MB)
│   │
│   └── postplan/                  # RUNTIME — turns questions into validated SQL
│       ├── query_processor.py     # Master orchestrator (6-step pipeline)
│       ├── blueprint/
│       │   ├── anchor_selector.py # Picks the structural anchor (FROM table)
│       │   ├── graph_pruner.py    # Prunes FK graph to a valid subgraph
│       │   ├── sql_generator.py   # LLM T-SQL writer (with business rules)
│       │   └── sql_executor.py    # Runs SQL via `docker exec sqlcmd`
│       └── docs/README.md         # Deep-dive docs for post-plan
│
├── agent/                         # FastAPI HTTP bridge
│   ├── main.py                    # All endpoints (/connect, /query, /services/*)
│   ├── connection_cache.json      # Persisted active connection across restarts
│   └── README.md                  # Agent-specific docs
│
└── ui/                            # Next.js 16 / React 19 frontend
    ├── src/app/
    │   ├── page.tsx               # Main chat page
    │   ├── layout.tsx             # Root layout + theme
    │   ├── history/page.tsx       # Past queries
    │   └── settings/page.tsx      # DB connection + preferences
    ├── src/components/
    │   ├── chat/                  # chat-input, header, message-item, sidebar
    │   ├── settings/              # database-selector, settings-group
    │   ├── history/               # history-card, terminal-logs
    │   └── layout/                # main-layout
    ├── src/types/index.ts
    └── README.md                  # UI-specific docs
```

> Every subdirectory (`agent/`, `ui/`, `app/preplan/doc/`, `app/postplan/docs/`) has its **own** deep-dive README. This top-level document is the index that ties them all together.

---

## 4. The Two-Phase Engine (Pre-Plan + Post-Plan)

NexusChat separates **"learn the schema once"** from **"answer questions many times"**:

| Phase         | When It Runs                                   | What It Produces                                              |
| :------------ | :--------------------------------------------- | :------------------------------------------------------------ |
| **Pre-Plan**  | Once, offline (first-time setup)               | A searchable vector knowledge base of every table in the DB   |
| **Post-Plan** | Every time a user sends a question to the chat | A validated T-SQL query, a live result set, a friendly answer |

The Pre-Plan knowledge base lives as three files in `app/preplan/output/` — they are loaded by the Post-Plan engine at startup and never regenerated at runtime.

---

## 5. Pre-Plan — Building the Vector Knowledge Base

**File:** `app/preplan/schema_pipeline.py` · **Entry:** `python3 -m app.preplanned_main`

The `SchemaPipeline` class runs **three idempotent steps** (if the output file exists, that step is skipped):

### Step 1 — Schema Extraction

Uses `pyodbc` + `INFORMATION_SCHEMA` + `sys.foreign_keys` to dump:

- Column metadata (name / type / nullability) for every `BASE TABLE` and `VIEW`
- `SELECT TOP 2` rows from each table (real values — no truncation)
- All foreign-key relationships

→ written to `output/database_schema.txt` in a per-entity block format:

```
ENTITY: Sales.SalesOrderHeader
COLUMNS:   - SalesOrderID | int | NO
           - TotalDue     | money | NO
           ...
SAMPLES:   Row 1: {'SalesOrderID': '43659', 'TotalDue': '23153.23', ...}
RELATIONSHIPS:
   - [FK_SalesOrderHeader_SalesPerson] 'SalesPersonID' -> Sales.SalesPerson(BusinessEntityID)
   ...
```

### Step 2 — LLM Schema Analysis (GPT-4o, temp 0.2)

For every entity block, GPT-4o is asked to produce a structured markdown document with:

- `## [Schema.TableName]` header
- **BUSINESS DESCRIPTION** (150–200 words — the "why")
- **COLUMNS** (business role of each significant column)
- **RELATIONSHIP ANALYSIS** (narrative describing the FK graph)
- **KEYWORDS** (6–8 RAG-tuned search terms)

The result is appended incrementally to `output/database_analysis.md` so a crash mid-way preserves prior work.

### Step 3 — Vector Embeddings (`text-embedding-3-small`)

For each analyzed block:

- `description` = the BUSINESS DESCRIPTION text
- `keywords` and `relationships` are stored as metadata (not embedded)
- Embedding input = `"{TableName}\n{description}"` (the table name acts as an anchor signal)

Output → `output/database_vectors.json`:

```json
{
  "table_name": "Sales.SalesOrderHeader",
  "description": "This table is essential for managing...",
  "keywords": ["sales orders", "revenue", "gross", "freight", ...],
  "relationships": "The SalesOrderHeader table links directly to Sales.Customer via...",
  "vector": [0.0023, -0.0141, ..., 0.0087]   // 1536 floats
}
```

> **Why only embed TableName + description?** Embedding relationships would leak join topology into the semantic space and make unrelated fact tables look similar. Keeping relationships as metadata lets Step 3 of Post-Plan expand the graph deterministically.

Full details in [`app/preplan/doc/README.md`](app/preplan/doc/README.md).

---

## 6. Post-Plan — The Runtime 6-Step Agentic Pipeline

**File:** `app/postplan/query_processor.py` · **Entry:** called by `agent/main.py` on every `/query`

```
User Question
     │
     ▼
┌─────────────────────────┐
│ 1. refine_query()       │   GPT-4o rewrites the question into a
│                         │   technical, keyword-dense search phrase
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 2. search_relevant_     │   Embed the refined phrase →
│    tables()             │   cosine similarity vs database_vectors.json
│                         │   → top-5 table matches
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 3. get_exact_table_     │   Read database_schema.txt directly.
│    details()            │   2-hop FK traversal to find bridge tables
│                         │   + harvest 2 data samples per table
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 4. advanced_sql_        │   Deterministic blueprint synthesis:
│    synthesis()          │   • anchor_selector → pick FROM table
│                         │   • graph_pruner → keep only reachable joins
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 5. Draft → Execute →    │   Agentic loop (max 10 iterations):
│    Audit → Repair       │   generate_sql → SELECT TOP 5 test run
│                         │   → evaluate_execution_sample (Critic)
│                         │   → feedback → retry until PASS
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 6. Final Execution +    │   Run the final SQL without TOP 5 limit.
│    NL Synthesis         │   GPT-4o-mini formats results as a
│                         │   friendly, currency-aware answer
└───────────┬─────────────┘
            ▼
  { friendly_answer, final_sql, execution_results, iteration_history, logs, metrics }
```

### Step 4a — Anchor Selection (`anchor_selector.py`)

Picks the primary `FROM` table using a **weighted scoring model**:

| Signal                    | Weight      |
| :------------------------ | :---------- |
| Semantic keyword match    | × 1.5       |
| Graph connectivity        | × 0.5       |
| Structural stability      | +2.0 / -1.0 |
| Vector similarity score   | × 3.0       |

For _"top selling sales people"_ → **`Sales.SalesPerson`** wins because of the triple signal (keyword hit, real table, high vector score).

### Step 4b — Graph Pruning (`graph_pruner.py`)

Builds a BFS (up to 3 hops) from the anchor, classifying every candidate as `Entity` / `MetricSource` / `Dimension` / `Noise` based on column count and in/out-degree. Only reachable, non-Noise tables survive. A second pass collects every join edge where **both** endpoints are in the connected set — preventing the classic "missing bridge" bug.

### Step 5 — Draft-Critic Loop (the heart of NexusChat)

- **Generator** (`sql_generator.py`, GPT-4o temp 0.0) writes T-SQL from the blueprint. The system prompt enforces strict AdventureWorks business rules:
  - `LineTotal` for **Net / Product Revenue**, `TotalDue` for **Gross / Cash Flow**
  - `Sales.SalesTerritory.[Group]` for regional queries
  - `OnlineOrderFlag` for channel queries
  - `TOP 50`/`TOP 100` safety cap unless user explicitly asks otherwise
- **Executor** (`sql_executor.py`) wraps the query in `SELECT TOP 5 …` and runs it via `docker exec sqlserver sqlcmd` with `-C` (trust cert) and `-W` (strip whitespace).
- **Critic** (`evaluate_execution_sample`, GPT-4o temp 0.0) inspects the 5-row sample and rejects common traps:
  - Over-counting `TotalDue` after joining `SalesOrderDetail` (massive inflation)
  - Using `COUNT(...)` instead of `COUNT(DISTINCT SalesOrderID)` for orders
  - Proxy columns instead of `OnlineOrderFlag` for channel questions

On **FAIL**, the feedback string is threaded back into the Generator's next attempt. The loop is capped at **10 iterations**.

### Step 6 — Natural-Language Synthesis

`generate_natural_language_answer` uses `gpt-4o-mini` to convert the raw result set into a clean answer, formatting currency intelligently (e.g. `4319749.73` → `$4.32M`) and listing every row the user explicitly asked for ("Top 5", "Top 10", etc.).

Full details in [`app/postplan/docs/README.md`](app/postplan/docs/README.md).

### The Over-Counting Trap (the single most important rule)

```sql
--  WRONG
SELECT SUM(soh.TotalDue), SUM(sod.LineTotal)
FROM Sales.SalesOrderHeader soh
JOIN Sales.SalesOrderDetail sod ON soh.SalesOrderID = sod.SalesOrderID;
-- TotalDue is repeated once per line item → inflated 3-10×

-- CORRECT — keep them in separate subqueries
SELECT s.SalesPersonID, h.GrossRevenue, l.NetRevenue
FROM Sales.SalesPerson s
JOIN (SELECT SalesPersonID, SUM(TotalDue) AS GrossRevenue
      FROM Sales.SalesOrderHeader GROUP BY SalesPersonID) h
  ON s.BusinessEntityID = h.SalesPersonID
JOIN (SELECT soh.SalesPersonID, SUM(sod.LineTotal) AS NetRevenue
      FROM Sales.SalesOrderDetail sod
      JOIN Sales.SalesOrderHeader soh ON sod.SalesOrderID = soh.SalesOrderID
      GROUP BY soh.SalesPersonID) l
  ON s.BusinessEntityID = l.SalesPersonID;
```

Both the Generator prompt and the Critic actively reject the wrong pattern.

---

## 7. The FastAPI Agent (HTTP Bridge)

**File:** `agent/main.py` · **Port:** 8000

The agent is a thin FastAPI server that the browser can call over HTTP. It doesn't know how to do AI itself — it imports `QueryProcessor` from `app/postplan/` and delegates.

### Endpoints

| Method | Path                   | Purpose                                                                         |
| :----- | :--------------------- | :------------------------------------------------------------------------------ |
| GET    | `/health`              | Heartbeat                                                                       |
| GET    | `/connection-status`   | Is a DB connection currently active?                                            |
| POST   | `/connect`             | Test DB credentials via `docker exec sqlcmd`; on success, cache them in memory |
| POST   | `/query`               | Run the full 6-step pipeline; blocked if no active connection                   |
| GET    | `/services/status`     | Is the SQL Server Docker container running?                                     |
| POST   | `/services/start`      | `docker-compose up sqlserver -d`                                                |
| POST   | `/services/stop`       | `docker rm -f sqlserver`                                                        |

### Connection State

- Credentials are **never** hardcoded — they come from the Settings UI.
- Stored in the `active_connection` dict (and persisted to `agent/connection_cache.json` so agent restarts pick up the last session).
- Dynamically discovers the SQL container name (exact match → image match → name-contains match).
- Queries are **blocked** if `active_connection` is empty — the UI receives `"⚠️ Please connect to a database first..."`.

### Log Capture for the UI

```python
with redirect_stdout(log_buffer):
    result = processor.process_for_vector_search(request.query)
logs = log_buffer.getvalue().splitlines()
```

Every `print()` emitted during the 6-step pipeline is captured and returned as the `logs` array — the UI renders it as the **Engine Execution Blueprint** panel, giving the user full visibility into refinement, matches, joins, iterations, audits, and the final SQL.

Full details in [`agent/README.md`](agent/README.md).

---

## 8. The Next.js UI

**Stack:** Next.js 16.2.3 · React 19.2.4 · Tailwind 4 · Framer Motion · Lucide · next-themes · Geist

### Pages

| Route        | Component              | Purpose                                                                        |
| :----------- | :--------------------- | :----------------------------------------------------------------------------- |
| `/`          | `app/page.tsx`         | Main chat — message list + input + blueprint inspector modal                   |
| `/history`   | `app/history/page.tsx` | Past-session queries with collapsible execution logs                           |
| `/settings`  | `app/settings/page.tsx`| 4 provider cards (Docker MSSQL, PostgreSQL, Supabase, Snowflake) — real verify |

### The Execution Blueprint Inspector

Every AI message exposes a 6-tab inspector:

1. **Original** — raw user intent
2. **Refined** — GPT-4o's technical re-write
3. **Agentic Logs** — colour-coded terminal output from Steps 1–6
4. **Final T-SQL** — the validated query
5. **Results Table** — the live sample data
6. **Metrics** — total time, API cost, iteration count

Full details in [`ui/README.md`](ui/README.md).

---

## 9. Setup & Run

> There is a detailed walkthrough in [`HOW_TO_RUN.md`](HOW_TO_RUN.md). What follows is the condensed version.

### Prerequisites

- Docker Desktop running
- Python 3.10+
- Node.js 18+
- A valid `OPENAI_API_KEY` in `.env`

### First-Time Setup

```bash
# 1. Start SQL Server (persistent volume keeps DB data across restarts)
docker-compose up sqlserver -d

# 2. Download the AdventureWorks backup and restore it
curl -L 'https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorks2019.bak' \
     -o /tmp/AdventureWorks2019.bak
docker cp /tmp/AdventureWorks2019.bak sqlserver:/var/opt/mssql/AdventureWorks2019.bak

docker exec sqlserver /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P 'your password' -C \
  -Q "RESTORE DATABASE AdventureWorks FROM DISK='/var/opt/mssql/AdventureWorks2019.bak' \
      WITH MOVE 'AdventureWorks2019'     TO '/var/opt/mssql/data/AdventureWorks.mdf', \
           MOVE 'AdventureWorks2019_log' TO '/var/opt/mssql/data/AdventureWorks_log.ldf', REPLACE"

# 3. Build the vector knowledge base (one-time; ~5-10 mins)
pip3 install -r requirements.txt
python3 -m app.preplanned_main
```

> **Why a persistent volume?** Before the fix, the SQL container stored data inside the container layer. Any `docker rm` or Mac restart wiped AdventureWorks. The named volume `sqlserver_data` in `docker-compose.yml` now stores DB files on the host — data survives restarts permanently.

### Daily Startup (3 terminals)

```bash
# Terminal 1 — Database
docker-compose up sqlserver -d

# Terminal 2 — Python Agent
python3 agent/main.py           # serves on 0.0.0.0:8000

# Terminal 3 — UI
cd ui && npm run dev            # serves on http://localhost:3000
```

### Shutdown

```bash
docker-compose stop sqlserver   # stop DB
# Ctrl+C in the agent and UI terminals
```

---

## 10. Connecting from the UI

1. Open **http://localhost:3000** → ⚙️ **Settings**
2. Click the **Docker MSSQL** card
3. Enter:
   - **DB Name:** `AdventureWorks`
   - **Username:** `sa`
   - **Password:** the value of `MSSQL_SA_PASSWORD` in `docker-compose.yml`
   - Host / Port are pre-filled
4. Click **Verify & Connect** → wait for the green banner
5. Start asking questions on the Chat page

### Connection Lifecycle

| Event                     | `active_connection` | UI Badge       |
| :------------------------ | :------------------ | :------------- |
| Agent starts (no cache)   | `{}`                | No ACTIVE tag  |
| Successful `/connect`     | `{user, pwd, ...}`  | ACTIVE      |
| Agent restart (with cache)| Restored from disk  | ACTIVE      |
| Failed `/connect`         | Unchanged           | or none     |

---

## 11. Validation Benchmark — AdventureWorks 12-Question Test Bench

NexusChat is benchmarked against the canonical AdventureWorks analysis from
**_Exploring AdventureWorks2019 Database using SQL_** by Iyanu Olatunji
([Medium article](https://medium.com/@iyanuolatunji21/exploring-adventureworks2019-database-using-sql-fdffac5d8b25)).

The article defines **12 business questions** that together exercise every major part of the sales schema. NexusChat is expected to autonomously produce SQL that returns **the same results** as the article's hand-written queries — without any schema hardcoding.

Execution logs, timings, SQL variants per iteration, Critic verdicts and final answers are tracked in the
[**NexusChat Validation Sheet**](https://docs.google.com/spreadsheets/d/1baeedstrM9wZqRiRsaW7h1oVU3f0RoY24nAsww6WnEs/edit?gid=527362238#gid=527362238).

### Question Bench

| # | User Query (plain English)                                                   | AdventureWorks Reference (expected logic)                                                              |
| :-| :----------------------------------------------------------------------------| :------------------------------------------------------------------------------------------------------|
| 1 | How does sales performance vary across product categories?                   | `SUM(LineTotal)` joined through `SalesOrderDetail → Product → ProductSubcategory → ProductCategory`    |
| 2 | Are there seasonal patterns or trends in sales?                              | `SUM(LineTotal)` grouped by `YEAR(OrderDate), MONTH(OrderDate)`                                        |
| 3 | Which sales representatives have the highest sales performance?              | `SalesPerson.SalesYTD / SalesQuota` ratio                                                              |
| 4 | Are there opportunities for cross-selling and up-selling?                    | Self-join `SalesOrderDetail` → product-pair frequency with `RANK() OVER(...)`                          |
| 5 | Which products have the highest units sold?                                  | `SUM(OrderQty)` grouped by `Product.Name`                                                              |
| 6 | Which region generated the highest revenue?                                  | `SUM(TotalDue)` grouped by `SalesTerritory.[Group]`                                                    |
| 7 | Are there differences in purchasing behaviour between territories?           | Territory-level unique customers / unique products / avg order value / total sales                     |
| 8 | Product popularity across regions                                            | `RANK() OVER (PARTITION BY Region ORDER BY COUNT(ProductID) DESC)`                                     |
| 9 | Who are the top-spending customers?                                          | `Person.Person ⟕ Customer ⟕ SalesOrderHeader` with `SUM(TotalDue)`                                     |
| 10| What are the most common sales channels?                                     | `CASE OnlineOrderFlag WHEN 0 THEN 'In Person' ELSE 'Online' END`                                       |
| 11| Most profitable products?                                                    | `LineTotal - (OrderQty * StandardCost)` → profit margin                                                |
| 12| Overall sales performance across the years?                                  | `SUM(TotalDue)` grouped by `YEAR(OrderDate)`                                                           |

### Example Match — Q6 "Which region generated the highest revenue?"

**Reference SQL (Medium article):**

```sql
SELECT ST.[Group] AS Region, SUM(SOH.Totaldue) AS Total_sales
FROM Sales.SalesTerritory ST
INNER JOIN Sales.SalesOrderHeader SOH ON ST.TerritoryID = SOH.TerritoryID
GROUP BY ST.[Group]
ORDER BY total_sales DESC;
```

**Expected insight (from the article):**

> North America recorded the highest revenue, amounting to **$89,228,792**. Europe followed with **$22,173,617**, while the Pacific region generated **$11,814,376**.

**NexusChat delivered the same result set.** The Generator correctly selected `TotalDue` (not `LineTotal`) because the question asks for "revenue" at the **region / gross** level, and the Critic signed off on `PASS` without retries. Full trace — refinement phrase, cosine match scores, iteration history, and timing — is logged in the validation sheet linked above.

The same validation has been run end-to-end across all 12 questions with matching results.

---

## 12. Troubleshooting

| Problem                                             | Fix                                                                        |
| :-------------------------------------------------- | :------------------------------------------------------------------------- |
| `No active database connection` in chat             | Settings → Docker MSSQL → **Verify & Connect**                             |
| `No SQL Server container found`                     | `docker-compose up sqlserver -d`                                           |
| `AdventureWorks` not found after `/connect` passes  | Re-run the restore step (see [Setup](#9-setup--run))                       |
| Agent not responding                                | Is `python3 agent/main.py` running in a terminal?                          |
| UI not loading                                      | Is `npm run dev` running in `ui/`?                                         |
| Container data wiped after Mac restart              | **Fixed** by the `sqlserver_data` named volume in `docker-compose.yml`     |
| `text-embedding-3-small` / GPT-4o errors            | Is `OPENAI_API_KEY` set in `.env` and loaded by the agent?                 |
| "Over-counting" detected by Critic                  | Expected — the loop will auto-retry with the Fix feedback (up to 10×)      |

---

## 13. Tech Stack

| Concern        | Tool                                                                  |
| :------------- | :-------------------------------------------------------------------- |
| UI framework   | Next.js 16 + React 19                                                 |
| UI styling     | Tailwind CSS 4 + Framer Motion + Geist font                           |
| UI icons/theme | Lucide + `next-themes`                                                |
| HTTP bridge    | FastAPI + Uvicorn                                                     |
| DB driver      | `pyodbc` (pre-plan) + `docker exec sqlcmd` (post-plan runtime)        |
| LLMs           | GPT-4o (refine, generate, critique) · GPT-4o-mini (final NL synth)    |
| Embeddings     | `text-embedding-3-small` (1536-dim)                                   |
| Database       | `mcr.microsoft.com/mssql/server:2022-latest` with AdventureWorks 2019 |
| Data safety    | Named Docker volume `sqlserver_data` (persisted on host disk)         |

---

### Sub-documents (deep dives)

- [`HOW_TO_RUN.md`](HOW_TO_RUN.md) — detailed setup walkthrough
- [`app/preplan/doc/README.md`](app/preplan/doc/README.md) — Pre-Plan engine
- [`app/postplan/docs/README.md`](app/postplan/docs/README.md) — Post-Plan 6-step pipeline
- [`agent/README.md`](agent/README.md) — FastAPI agent internals
- [`ui/README.md`](ui/README.md) — Next.js UI architecture

---

_NexusChat v3.2 — UI-Integrated Agentic NL2SQL Engine · Zero-Static Schema Understanding_
