# F1 Analytics Agent — Product Requirements Document (PRD)
**Version:** 1.0 — MVP  
**Author:** Michael Dambock  
**Project:** LLM Zoomcamp Capstone — DataTalksClub  
**Status:** Draft for Review  

---

## 1. Problem Statement

Formula 1 generates a rich dataset — lap times, pit stop strategies, qualifying deltas, constructor standings — but analytical insight requires SQL expertise and domain knowledge that most fans and analysts don't have readily available.

Existing tools (FastF1, Ergast API, official F1 dashboards) require either programming skills or provide only pre-canned visualizations. There is no free, conversational interface that lets a user ask a natural language question about F1 data and receive a grounded, data-backed answer with explanation.

This project builds that interface, while simultaneously serving as a rigorous demonstration of LLM engineering concepts: retrieval-augmented generation, vector embeddings, Text-to-SQL, context retrieval, and evaluation.

---

## 2. Target Users

### Primary
- **LLM Zoomcamp peer reviewers** — technically literate, evaluating for correctness, reproducibility, and LLM concept coverage. Must be able to run the project locally or via Streamlit Cloud at zero cost.

### Secondary
- **Portfolio audience** — hiring managers, technical leads evaluating Data Engineer / Analytics Engineer candidates.
- **F1 enthusiasts with analytical curiosity** — no SQL required.

---

## 3. User Stories

| ID | As a... | I want to... | So that... |
|----|---------|--------------|------------|
| US-01 | F1 fan | Ask "Who gained the most positions in the 2024 season?" | I get a ranked answer with data |
| US-02 | F1 analyst | Ask "Compare Ferrari vs McLaren race pace in 2023" | I see a comparative breakdown |
| US-03 | F1 fan | Ask "What was Hamilton's best qualifying lap in Monaco?" | I get a direct factual answer |
| US-04 | F1 analyst | Ask "Which team had the fastest pit stops in 2024?" | I see aggregated pit stop data |
| US-05 | Peer reviewer | Clone the repo and run the app locally | It works with zero paid services |
| US-06 | Peer reviewer | Inspect the evaluation results | I can verify RAG pipeline quality |
| US-07 | F1 fan | Ask a vague question | The agent clarifies or gracefully handles it |

---

## 4. Functional Requirements — MVP

### 4.1 Data Ingestion
- FR-01: Ingest F1 race data for seasons **2024, 2025, 2026** using the FastF1 Python library. Season 2023 may be added in v2 as a full backfill; 2026 races added incrementally as each weekend concludes.
- FR-02: Data entities: race results, qualifying results, lap times, pit stops, drivers, constructors, circuits, championship standings.
- FR-03: Raw data persisted as **Parquet files** locally.
- FR-04: Ingestion triggered by a **Python script** (manual or scheduled). No orchestration tool required for MVP.
- FR-05: New race detection logic: check if a race weekend has concluded before attempting ingestion.

### 4.2 Storage & Transformation
- FR-06: DuckDB as the sole storage engine — no external database required.
- FR-07: dbt Core transforms raw Parquet into **Analytics Marts** (structured, clean, analytical tables).
- FR-08: dbt produces a dedicated **text chunk model** (`mart_text_cards`) — one row per fact, formatted as a human-readable sentence. Example:
  ```
  "In the 2024 Monaco GP, Charles Leclerc qualified P1, finished P1, 
   made 2 pit stops, and recorded an average race pace of 1:14.820."
  ```
- FR-09: Text chunks embedded using **Gemini `text-embedding-004`** (free tier) and stored in DuckDB using the **VSS extension** (vector similarity search).

### 4.3 AI Layer — Hybrid RAG + Text-to-SQL
- FR-10: User query is classified by an **intent router** into one of two paths:
  - **RAG path**: contextual, comparative, or descriptive questions → vector similarity search → retrieved chunks → LLM answer.
  - **SQL path**: precise aggregations, rankings, counts → SQL generation → DuckDB execution → LLM interpretation.
- FR-11: LLM: **Gemini 1.5 Flash** (free tier). System prompt includes DuckDB schema for SQL path.
- FR-12: SQL validation step before execution — catch syntax errors, retry once with error context.
- FR-13: LLM generates a natural language explanation of the result for both paths.
- FR-14: Basic chart rendered by **Plotly** (not LLM-generated) when result is tabular.

### 4.4 Interface
- FR-15: **Streamlit** web application — single-page chat interface.
- FR-16: Deployed on **Streamlit Cloud** (free tier, public URL) for peer review access.
- FR-17: User can type a free-form question in English.
- FR-18: Response shows: answer text + optional chart + data source reference.

### 4.5 Evaluation Framework
- FR-19: Static file `eval/ground_truth.json` containing **25 question-answer pairs** covering both RAG and SQL paths.
- FR-20: Automated evaluation script using **LLM-as-Judge** method (Gemini evaluates retrieved answer vs. ground truth).
- FR-21: Evaluation produces a report with: retrieval relevance score, answer correctness score, SQL execution success rate.

---

## 5. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| Cost | Zero paid services. All APIs on free tier. No paid cloud. |
| Reproducibility | Any peer reviewer can clone and run with a free Gemini API key |
| Response time | Query response < 8 seconds (acceptable for demo context) |
| Data freshness | 2024–2026 seasons; 2026 races ingested incrementally after each race weekend |
| Portability | Runs on local machine (Linux/Mac/Windows) and Streamlit Cloud |
| LLM dependency | Single provider (Gemini) — no multi-provider complexity at MVP |
| Storage | DuckDB single file — no Docker, no external services |

---

## 6. Architecture Overview

```
[FastF1 API]
     │
     ▼
[Python Ingestion Script]
     │
     ▼
[Parquet Files — local /data/raw/]
     │
     ▼
[dbt Core Transformations]
     ├── Analytics Marts (DuckDB)
     └── Text Card Models (mart_text_cards)
                    │
                    ▼
     [Gemini Embeddings — text-embedding-004]
                    │
                    ▼
     [DuckDB VSS — vector store, same file]
                    │
                    ▼
          [User Query — Streamlit UI]
                    │
          [Intent Router — RAG or SQL?]
           ┌────────┴────────┐
       RAG Path          SQL Path
    VSS similarity     LLM generates SQL
      search           DuckDB executes
       └────────┬────────┘
         [Gemini 1.5 Flash]
         Interprets result
                │
         [Streamlit response]
         Text + Plotly chart
```

---

## 7. Tech Stack

| Component | Tool | Justification |
|-----------|------|---------------|
| Data source | FastF1 | Free, Python-native, complete F1 data |
| Storage | DuckDB | Serverless, fast analytics, VSS extension |
| Transformation | dbt Core | Analytics marts + semantic text generation |
| Vector store | DuckDB VSS | Unified storage, zero extra ops |
| Embeddings | Gemini text-embedding-004 | Free tier, high quality |
| LLM | Gemini 1.5 Flash | Free tier, fast, good SQL generation |
| Frontend | Streamlit | Rapid UI, free cloud deployment |
| Charts | Plotly | Deterministic, no LLM involvement |
| Evaluation | LLM-as-Judge (Gemini) | Course-aligned method |
| Language | Python | Consistent with user profile and course |

---

## 8. Success Metrics

| Metric | MVP Target |
|--------|-----------|
| SQL execution success rate | ≥ 80% |
| RAG retrieval relevance score | ≥ 0.75 (LLM-as-Judge, 0–1 scale) |
| Answer correctness (ground truth) | ≥ 70% of 25 eval questions |
| Query response time | < 8 seconds (p90) |
| Seasons covered | 2024–2026 |
| Ground truth Q&A pairs | 25 minimum |
| Zero-cost operation | 100% — no paid service required |
| Peer reviewer can run it | Yes — README + Streamlit Cloud URL |

---

## 9. MVP Boundaries

### In Scope
- Seasons 2024–2026 (2023 as optional v2 backfill)
- Race results, qualifying, lap times, pit stops, drivers, constructors, circuits, standings
- Hybrid RAG + Text-to-SQL agent
- DuckDB VSS vector store
- Streamlit Cloud deployment
- Evaluation framework with ground truth
- English language only

### Out of Scope (MVP)
- Telemetry data (too heavy, optional later)
- Kestra orchestration (Python script sufficient for MVP)
- Qdrant / Chroma (DuckDB VSS covers the need)
- Multi-language support
- User authentication
- RAG over external documents (race reports, FIA documents, news)
- Strategy advisor / race engineer simulator
- Real-time race data

---

## 10. Data Model — Key dbt Layers

```
staging/           ← cleaned raw FastF1 data
  stg_results
  stg_qualifying
  stg_laps
  stg_pitstops
  stg_drivers
  stg_constructors

marts/             ← analytical tables
  mart_race_results
  mart_qualifying
  mart_pit_stops
  mart_driver_standings
  mart_constructor_standings

semantic/          ← LLM-ready
  mart_text_cards  ← one sentence per fact, used for embedding
```

---

## 11. Evaluation Framework Detail

**File:** `eval/ground_truth.json`

```json
[
  {
    "id": "Q001",
    "question": "Who won the most races in the 2023 season?",
    "expected_answer": "Max Verstappen, with 19 wins.",
    "path": "sql"
  },
  {
    "id": "Q002",
    "question": "What was the fastest pit stop recorded in 2024?",
    "expected_answer": "Red Bull Racing, 2.1 seconds at the Bahrain GP.",
    "path": "rag"
  }
]
```

**Scoring:** LLM-as-Judge prompt evaluates generated answer against expected on a 1–5 scale. Aggregated into a final report at `eval/results/report.json`.

---

## 12. Repository Structure

```
f1-analytics-agent/
├── README.md
├── docs/
│   ├── PRD.md          ← this document
│   ├── SDD.md          ← next to be created
│   └── ROADMAP.md
├── data/
│   ├── raw/            ← Parquet files from FastF1
│   └── duckdb/         ← f1_analytics.duckdb
├── src/
│   ├── ingestion/      ← FastF1 ingestion scripts
│   ├── dbt/            ← dbt project
│   │   ├── models/
│   │   │   ├── staging/
│   │   │   ├── marts/
│   │   │   └── semantic/
│   │   └── dbt_project.yml
│   ├── embeddings/     ← embedding generation + VSS loader
│   ├── agent/          ← intent router, RAG, SQL, LLM calls
│   └── app/            ← Streamlit application
└── eval/
    ├── ground_truth.json
    ├── evaluate.py
    └── results/
```

---

## 13. Future Roadmap (Post-MVP)

| Phase | Feature |
|-------|---------|
| v2 | Kestra orchestration for automated 2026 race ingestion |
| v2 | Backfill 2023 season data |
| v2 | RAG over FIA race reports and stewards decisions |
| v2 | Strategy advisor / what-if analysis |
| v3 | Telemetry analysis (sector times, speed traces) |
| v3 | Race engineer simulator persona |
| v3 | DBT Documentation Copilot (separate project) |

---

## 14. Open Questions

| # | Question | Owner | Status | Resolution |
|---|----------|-------|--------|------------|
| OQ-01 | Does Gemini free tier sustain embedding generation for 4 seasons of text cards without hitting rate limits? | Michael | ✅ Closed | Reduced to 2024+2025+2026 only (~5K–15K vectors). Free tier is sufficient. 2023 added in v2 if needed. |
| OQ-02 | Does DuckDB VSS handle ~50K text vectors within acceptable query latency? | Michael | ✅ Closed | MVP vector count is ~5K–15K — well within DuckDB VSS capability. Benchmark before v2 backfill of 2023. |
| OQ-03 | Is FastF1 2026 mid-season data available? | Michael | ✅ Closed | FastF1 supports 2026 season. Standard data (results, laps, pit stops, qualifying) available. ERS/active aero/DRS restricted by F1 for 2026 — no impact since telemetry is out of MVP scope. |

---

*PRD v1.0 — Next step: System Design Document (SDD)*
