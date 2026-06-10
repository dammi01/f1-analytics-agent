# F1 Analytics Agent — System Design Document (SDD)
**Version:** 1.0 — MVP  
**Author:** Michael Dambock  
**Project:** LLM Zoomcamp Capstone — DataTalksClub  
**Status:** Draft  
**Depends on:** PRD v1.0  

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                               │
│                                                                 │
│  FastF1 API ──► Python Ingestion ──► Parquet Files             │
│                                       data/raw/{year}/{event}/  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    TRANSFORMATION LAYER                         │
│                                                                 │
│  dbt Core                                                       │
│  ├── staging/    (clean, type-cast)                             │
│  ├── marts/      (analytical tables)                            │
│  └── semantic/   (text cards)                                   │
│                          │                                      │
│                    DuckDB (.duckdb file)                        │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                     EMBEDDING LAYER                             │
│                                                                 │
│  mart_text_cards ──► Gemini text-embedding-004 ──► DuckDB VSS  │
│                                                  (HNSW index)  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                       AGENT LAYER                               │
│                                                                 │
│  User Query ──► Intent Router                                   │
│                 ├── RAG Chain ──► VSS search ──► Gemini Flash   │
│                 └── SQL Chain ──► SQL gen ──► DuckDB ──► Gemini │
└─────────────────────────┬───────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    PRESENTATION LAYER                           │
│                                                                 │
│  Streamlit App ──► Text answer + Plotly chart                   │
│  Streamlit Cloud (public URL for peer review)                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Repository Structure (Detailed)

```
f1-analytics-agent/
├── .env                          # API keys (never committed)
├── .gitignore
├── .python-version               # 3.12
├── pyproject.toml                # uv project + dependencies
├── README.md
├── LICENSE                       # MIT
├── Makefile                      # setup, run, eval commands
│
├── docs/
│   ├── PRD.md
│   ├── SDD.md                    ← this document
│   └── ROADMAP.md
│
├── data/
│   ├── raw/                      # Parquet files (committed to git)
│   │   └── {year}/
│   │       └── {event_name}/
│   │           ├── results.parquet
│   │           ├── qualifying.parquet
│   │           ├── laps.parquet
│   │           ├── pit_stops.parquet
│   │           └── weather.parquet
│   └── duckdb/
│       └── f1_analytics.duckdb   # NOT committed (.gitignore)
│
├── src/
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── ingest.py             # main ingestion script
│   │   ├── fetcher.py            # FastF1 wrapper
│   │   └── checker.py            # new race detection logic
│   │
│   ├── dbt/                      # dbt project root
│   │   ├── dbt_project.yml
│   │   ├── profiles.yml
│   │   ├── models/
│   │   │   ├── staging/
│   │   │   │   ├── stg_results.sql
│   │   │   │   ├── stg_qualifying.sql
│   │   │   │   ├── stg_laps.sql
│   │   │   │   ├── stg_pit_stops.sql
│   │   │   │   ├── stg_drivers.sql
│   │   │   │   ├── stg_constructors.sql
│   │   │   │   └── stg_circuits.sql
│   │   │   ├── marts/
│   │   │   │   ├── mart_race_results.sql
│   │   │   │   ├── mart_qualifying.sql
│   │   │   │   ├── mart_pit_stops.sql
│   │   │   │   ├── mart_lap_analysis.sql
│   │   │   │   ├── mart_driver_standings.sql
│   │   │   │   └── mart_constructor_standings.sql
│   │   │   └── semantic/
│   │   │       └── mart_text_cards.sql
│   │   └── tests/
│   │
│   ├── embeddings/
│   │   ├── __init__.py
│   │   ├── embed.py              # generate + store embeddings
│   │   └── vss.py                # DuckDB VSS setup + search
│   │
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── router.py             # intent classification
│   │   ├── rag_chain.py          # RAG pipeline
│   │   ├── sql_chain.py          # Text-to-SQL pipeline
│   │   └── llm.py                # Gemini API wrapper
│   │
│   └── app/
│       ├── app.py                # Streamlit entry point
│       └── components.py         # UI components
│
└── eval/
    ├── ground_truth.json         # 25 Q&A pairs
    ├── evaluate.py               # evaluation runner
    └── results/
        └── report.json           # evaluation output
```

---

## 3. Component Design

### 3.1 Ingestion Layer (`src/ingestion/`)

**Responsibility:** Fetch F1 data from FastF1 and persist as Parquet.

**`fetcher.py`** — FastF1 wrapper:
```python
import fastf1
import pandas as pd
from pathlib import Path

ENTITIES = ["results", "qualifying", "laps", "pit_stops"]
RAW_PATH = Path("data/raw")

def fetch_event(year: int, round_number: int) -> None:
    session_map = {
        "results": "R",
        "qualifying": "Q",
        "laps": "R",
        "pit_stops": "R",
    }
    fastf1.Cache.enable_cache("data/cache")
    race = fastf1.get_event(year, round_number)
    event_slug = race["EventName"].replace(" ", "_").lower()
    out_path = RAW_PATH / str(year) / event_slug
    out_path.mkdir(parents=True, exist_ok=True)
    # fetch and save each entity as Parquet
```

**`checker.py`** — new race detection:
```python
def get_completed_rounds(year: int) -> list[int]:
    """Returns list of round numbers already concluded this year."""

def get_ingested_rounds(year: int) -> list[int]:
    """Returns rounds already in data/raw/{year}/."""

def get_pending_rounds(year: int) -> list[int]:
    return list(set(get_completed_rounds(year)) - set(get_ingested_rounds(year)))
```

**`ingest.py`** — orchestrator:
```python
SEASONS = [2024, 2025, 2026]

for year in SEASONS:
    for round_num in get_pending_rounds(year):
        fetch_event(year, round_num)
        print(f"Ingested {year} round {round_num}")
```

---

### 3.2 dbt Transformation Layer (`src/dbt/`)

**Responsibility:** Transform Parquet → clean analytics tables → semantic text cards.

**`profiles.yml`** — DuckDB connection:
```yaml
f1_analytics:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: "../../data/duckdb/f1_analytics.duckdb"
      threads: 4
```

**`dbt_project.yml`** — project config:
```yaml
name: f1_analytics
version: 1.0.0
profile: f1_analytics

models:
  f1_analytics:
    staging:
      materialized: view
    marts:
      materialized: table
    semantic:
      materialized: table
```

**Key staging model — `stg_results.sql`:**
```sql
SELECT
    CAST(raceId    AS VARCHAR)      AS race_id,
    CAST(driverId  AS VARCHAR)      AS driver_id,
    CAST(constructorId AS VARCHAR)  AS constructor_id,
    CAST(grid      AS INTEGER)      AS grid_position,
    CAST(position  AS INTEGER)      AS finish_position,
    CAST(points    AS FLOAT)        AS points,
    CAST(laps      AS INTEGER)      AS laps_completed,
    time                            AS race_time,
    year,
    round
FROM read_parquet('data/raw/*/*/results.parquet')
```

**Key semantic model — `mart_text_cards.sql`:**
```sql
-- One human-readable sentence per driver per race
SELECT
    race_id || '_' || driver_id AS card_id,
    'In the ' || year::VARCHAR || ' ' || race_name
    || ', ' || driver_name
    || ' qualified P' || grid_position::VARCHAR
    || ', finished P' || finish_position::VARCHAR
    || ', completed ' || laps_completed::VARCHAR || ' laps'
    || ', scored ' || points::VARCHAR || ' points'
    || CASE WHEN pit_stops IS NOT NULL
       THEN ', made ' || pit_stops::VARCHAR || ' pit stop(s)'
       ELSE '' END
    || '.'                       AS text_chunk,
    year,
    round,
    race_name,
    driver_name,
    constructor_name,
    'race_summary'               AS card_type
FROM mart_race_results
LEFT JOIN mart_pit_stops USING (race_id, driver_id)
```

---

### 3.3 Embedding Layer (`src/embeddings/`)

**Responsibility:** Embed text cards and store vectors in DuckDB VSS.

**`vss.py`** — DuckDB VSS setup:
```python
import duckdb

def setup_vss(conn: duckdb.DuckDBPyConnection) -> None:
    conn.execute("INSTALL vss; LOAD vss;")
    conn.execute("""
        CREATE TABLE IF NOT EXISTS embeddings (
            card_id   VARCHAR PRIMARY KEY,
            text      VARCHAR,
            embedding FLOAT[768],
            metadata  JSON
        )
    """)
    conn.execute("""
        CREATE INDEX IF NOT EXISTS idx_hnsw
        ON embeddings
        USING HNSW (embedding)
        WITH (metric = 'cosine')
    """)

def search(conn, query_embedding: list[float], top_k: int = 5) -> list[dict]:
    results = conn.execute("""
        SELECT card_id, text, metadata,
               array_cosine_similarity(embedding, ?::FLOAT[768]) AS score
        FROM embeddings
        ORDER BY score DESC
        LIMIT ?
    """, [query_embedding, top_k]).fetchall()
    return results
```

**`embed.py`** — embedding pipeline:
```python
import google.generativeai as genai
import duckdb, json, time
from pathlib import Path

BATCH_SIZE = 100  # Gemini free tier rate limit buffer
EMBED_MODEL = "models/text-embedding-004"

def embed_text_cards(conn: duckdb.DuckDBPyConnection) -> None:
    cards = conn.execute(
        "SELECT card_id, text_chunk, year, round, driver_name FROM mart_text_cards"
    ).fetchall()

    for i in range(0, len(cards), BATCH_SIZE):
        batch = cards[i:i + BATCH_SIZE]
        for card_id, text, year, round_, driver in batch:
            result = genai.embed_content(
                model=EMBED_MODEL,
                content=text,
                task_type="retrieval_document"
            )
            conn.execute(
                "INSERT OR REPLACE INTO embeddings VALUES (?, ?, ?, ?)",
                [card_id, text, result["embedding"],
                 json.dumps({"year": year, "round": round_, "driver": driver})]
            )
        time.sleep(1)  # respect free tier rate limit
        print(f"Embedded batch {i // BATCH_SIZE + 1}")
```

---

### 3.4 Agent Layer (`src/agent/`)

**Responsibility:** Route queries, execute RAG or SQL pipeline, return grounded answers.

#### Intent Router (`router.py`)

Think of the router as a **train switchman**: same track in, different destination out depending on the signal.

```python
SQL_KEYWORDS = [
    "most", "least", "fastest", "slowest", "average", "total",
    "how many", "count", "rank", "top", "bottom", "best", "worst",
    "sum", "percentage", "ratio", "compare points", "standings"
]

RAG_KEYWORDS = [
    "tell me about", "describe", "what happened", "explain",
    "what was it like", "history", "context", "strategy", "narrative"
]

def classify_intent(query: str) -> str:
    """Returns 'sql' or 'rag'."""
    q = query.lower()
    sql_score = sum(1 for kw in SQL_KEYWORDS if kw in q)
    rag_score  = sum(1 for kw in RAG_KEYWORDS  if kw in q)
    return "sql" if sql_score >= rag_score else "rag"
```

#### RAG Chain (`rag_chain.py`)

```
Query
  │
  ▼
Embed query (Gemini text-embedding-004)
  │
  ▼
DuckDB VSS cosine search → top 5 text chunks
  │
  ▼
Build prompt:
  "Answer the question using only the context below.
   Context: {chunks}
   Question: {query}"
  │
  ▼
Gemini 1.5 Flash → answer text
```

#### SQL Chain (`sql_chain.py`)

```
Query
  │
  ▼
Build prompt:
  "Generate a DuckDB SQL query for:
   Schema: {mart_schema}
   Question: {query}
   Rules: SELECT only, no DDL, use mart_ tables only"
  │
  ▼
Gemini 1.5 Flash → SQL string
  │
  ▼
Validate SQL (syntax check via DuckDB EXPLAIN)
  │
  ├── Error → retry once with error context → fail gracefully
  │
  ▼
DuckDB execute → DataFrame result
  │
  ▼
Gemini 1.5 Flash → interpret result in natural language
  │
  ▼
Plotly → chart (if result is tabular with > 2 rows)
```

#### LLM Wrapper (`llm.py`)

```python
import google.generativeai as genai
import os

genai.configure(api_key=os.getenv("GEMINI_API_KEY"))
model = genai.GenerativeModel("gemini-1.5-flash")

def generate(prompt: str, system: str = "") -> str:
    response = model.generate_content(
        f"{system}\n\n{prompt}" if system else prompt
    )
    return response.text

def embed(text: str) -> list[float]:
    result = genai.embed_content(
        model="models/text-embedding-004",
        content=text,
        task_type="retrieval_query"
    )
    return result["embedding"]
```

---

### 3.5 Streamlit Application (`src/app/app.py`)

**Single-page chat interface. No authentication. No session persistence.**

```
┌─────────────────────────────────────┐
│  🏎️  F1 Analytics Agent             │
├─────────────────────────────────────┤
│                                     │
│  [Chat history area]                │
│                                     │
│  User: Who won the most races       │
│        in 2024?                     │
│                                     │
│  Agent: Max Verstappen won 9        │
│         races in 2024...            │
│         [Bar chart]                 │
│                                     │
├─────────────────────────────────────┤
│  [Text input]          [Send ▶]     │
└─────────────────────────────────────┘
```

**Key Streamlit patterns:**
```python
import streamlit as st
from agent.router import classify_intent
from agent.rag_chain import rag_answer
from agent.sql_chain import sql_answer

st.title("🏎️ F1 Analytics Agent")

if "messages" not in st.session_state:
    st.session_state.messages = []

for msg in st.session_state.messages:
    st.chat_message(msg["role"]).write(msg["content"])

if query := st.chat_input("Ask anything about F1..."):
    st.session_state.messages.append({"role": "user", "content": query})
    
    with st.spinner("Analysing..."):
        intent = classify_intent(query)
        if intent == "sql":
            answer, df = sql_answer(query)
            st.chat_message("assistant").write(answer)
            if df is not None and len(df) > 2:
                st.plotly_chart(build_chart(df))
        else:
            answer = rag_answer(query)
            st.chat_message("assistant").write(answer)
    
    st.session_state.messages.append({"role": "assistant", "content": answer})
```

---

### 3.6 Evaluation Pipeline (`eval/`)

**`ground_truth.json` structure:**
```json
[
  {
    "id": "Q001",
    "question": "Who won the most races in the 2024 season?",
    "expected": "Max Verstappen",
    "path": "sql",
    "category": "standings"
  },
  {
    "id": "Q002",
    "question": "What was the fastest pit stop in the 2024 Bahrain GP?",
    "expected": "Red Bull Racing, under 2.5 seconds",
    "path": "rag",
    "category": "pit_stops"
  }
]
```

**`evaluate.py`** — LLM-as-Judge:
```python
JUDGE_PROMPT = """
You are evaluating an AI answer against a ground truth.
Score the answer from 1 to 5:
  5 = correct, complete, well-explained
  3 = partially correct
  1 = wrong or irrelevant

Question: {question}
Expected: {expected}
Generated: {generated}

Return JSON only: {{"score": <int>, "reason": "<str>"}}
"""

def evaluate_all() -> dict:
    results = []
    for item in load_ground_truth():
        generated = run_agent(item["question"])
        score = judge(item["question"], item["expected"], generated)
        results.append({**item, "generated": generated, **score})
    
    avg_score = sum(r["score"] for r in results) / len(results)
    sql_success = sum(1 for r in results if r["path"] == "sql" 
                      and r["score"] >= 3) / sql_count
    
    return {"avg_score": avg_score, "sql_success_rate": sql_success, 
            "details": results}
```

---

## 4. DuckDB Schema Reference

### Staging Views
| Model | Key Columns |
|-------|-------------|
| `stg_results` | race_id, driver_id, constructor_id, grid_position, finish_position, points, laps_completed, year, round |
| `stg_qualifying` | race_id, driver_id, constructor_id, position, q1_time, q2_time, q3_time |
| `stg_laps` | race_id, driver_id, lap_number, lap_time_ms, sector1_ms, sector2_ms, sector3_ms |
| `stg_pit_stops` | race_id, driver_id, stop_number, lap, duration_ms |
| `stg_drivers` | driver_id, code, forename, surname, nationality |
| `stg_constructors` | constructor_id, name, nationality |
| `stg_circuits` | circuit_id, name, location, country |

### Analytics Marts (Tables)
| Model | Purpose |
|-------|---------|
| `mart_race_results` | Full race result with driver/constructor/circuit joined |
| `mart_qualifying` | Qualifying positions + time deltas to pole |
| `mart_pit_stops` | Pit stop durations + strategy analysis |
| `mart_lap_analysis` | Avg/best/std lap times per driver per race |
| `mart_driver_standings` | Championship points after each round |
| `mart_constructor_standings` | Constructor championship after each round |

### Semantic (Table)
| Model | Purpose |
|-------|---------|
| `mart_text_cards` | card_id, text_chunk, year, round, race_name, driver_name, constructor_name, card_type |

### Embeddings (DuckDB VSS)
| Column | Type | Notes |
|--------|------|-------|
| card_id | VARCHAR PK | FK to mart_text_cards |
| text | VARCHAR | original text chunk |
| embedding | FLOAT[768] | Gemini text-embedding-004 |
| metadata | JSON | year, round, driver, constructor |

---

## 5. Setup & Run

### One-time setup
```bash
# Install dependencies
uv sync

# Run ingestion (downloads F1 data → Parquet)
uv run python src/ingestion/ingest.py

# Build dbt models (Parquet → DuckDB marts + text cards)
cd src/dbt && uv run dbt run && cd ../..

# Generate embeddings (text cards → DuckDB VSS)
uv run python src/embeddings/embed.py
```

### Run the app
```bash
uv run streamlit run src/app/app.py
```

### Run evaluation
```bash
uv run python eval/evaluate.py
# outputs: eval/results/report.json
```

### Makefile targets
```makefile
setup:   uv sync
ingest:  uv run python src/ingestion/ingest.py
dbt:     cd src/dbt && uv run dbt run
embed:   uv run python src/embeddings/embed.py
app:     uv run streamlit run src/app/app.py
eval:    uv run python eval/evaluate.py
all:     ingest dbt embed
```

---

## 6. Dependencies (`pyproject.toml`)

```toml
[project]
name = "f1-analytics-agent"
version = "0.1.0"
requires-python = ">=3.12"

dependencies = [
    "fastf1>=3.4",
    "duckdb>=1.1",
    "dbt-core>=1.8",
    "dbt-duckdb>=1.8",
    "google-generativeai>=0.8",
    "streamlit>=1.40",
    "plotly>=5.24",
    "pandas>=2.2",
    "python-dotenv>=1.0",
    "pyarrow>=17.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "ruff>=0.7",
]
```

---

## 7. Sequence Diagrams

### 7.1 RAG Query Flow
```
User          Streamlit        Router        RAG Chain       DuckDB VSS      Gemini
 │                │               │               │               │             │
 │──"Tell me      │               │               │               │             │
 │  about         │               │               │               │             │
 │  Verstappen    │               │               │               │             │
 │  2024"────────►│               │               │               │             │
 │                │──classify()──►│               │               │             │
 │                │◄──"rag"───────│               │               │             │
 │                │───────────────────────────────►               │             │
 │                │               │          embed(query)─────────────────────►│
 │                │               │          ◄────────────────────────────────[vec]
 │                │               │          search(vec, k=5)────►│             │
 │                │               │          ◄──────[5 chunks]────│             │
 │                │               │          build_prompt()        │             │
 │                │               │          generate(prompt)──────────────────►│
 │                │               │          ◄─────────────────────────────[answer]
 │◄───────[answer]│               │               │               │             │
```

### 7.2 SQL Query Flow
```
User          Streamlit        Router        SQL Chain       DuckDB        Gemini
 │                │               │               │               │             │
 │──"Top 5        │               │               │               │             │
 │  drivers       │               │               │               │             │
 │  2024"────────►│               │               │               │             │
 │                │──classify()──►│               │               │             │
 │                │◄──"sql"───────│               │               │             │
 │                │───────────────────────────────►               │             │
 │                │               │          gen_sql(query)────────────────────►│
 │                │               │          ◄──────────────────────────────[sql]
 │                │               │          validate(sql)─────────►             │
 │                │               │          ◄──────[ok / error]───│             │
 │                │               │          execute(sql)──────────►│             │
 │                │               │          ◄──────[DataFrame]─────│             │
 │                │               │          interpret(df)──────────────────────►│
 │                │               │          ◄────────────────────────────[answer]
 │◄──[answer+chart]│              │               │               │             │
```

---

## 8. Environment Variables (`.env`)

```bash
GEMINI_API_KEY=your_key_here        # from aistudio.google.com
FASTF1_CACHE_PATH=data/cache        # local FastF1 cache
DUCKDB_PATH=data/duckdb/f1_analytics.duckdb
DBT_PROFILES_DIR=src/dbt
```

---

## 9. Deployment — Streamlit Cloud

1. Push code + Parquet files to `github.com/dammi01/f1-analytics-agent` (public)
2. Connect repo at `share.streamlit.io`
3. Set entry point: `src/app/app.py`
4. Add `GEMINI_API_KEY` in Streamlit Cloud secrets
5. Add `requirements.txt` (generated from `pyproject.toml`) for Streamlit Cloud compatibility

**Note:** Streamlit Cloud does not run `make all`. The DuckDB file must be either:
- Pre-built locally and committed (if under 100MB), **or**
- Built at startup via `@st.cache_resource` that calls the setup pipeline once

Recommended: pre-build locally, commit `data/duckdb/f1_analytics.duckdb` for Streamlit Cloud only via Git LFS.

---

## 10. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Gemini free tier rate limit during bulk embedding | Medium | Medium | Batch with sleep(1), embed once and cache in DuckDB |
| FastF1 slow data download for 3 seasons | High | Low | Cache enabled, one-time operation |
| DuckDB VSS cosine search slow at query time | Low | Medium | HNSW index, k=5 is fast |
| SQL generation hallucination | Medium | Medium | Schema injection + EXPLAIN validation + retry |
| Streamlit Cloud DuckDB file too large | Medium | High | Git LFS or pre-filter to marts only |

---

*SDD v1.0 — Next step: implement `src/ingestion/` layer*
