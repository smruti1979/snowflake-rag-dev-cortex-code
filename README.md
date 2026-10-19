# Building AI Applications with Snowflake Cortex

**RAG, Text-to-SQL & Cortex Code (CoCo) — Hands-on Workshop Notebook**

A guided Snowflake Notebook that walks through building an AI application on Snowflake end to end: calling LLMs inside your account, grounding them with your own data using Retrieval-Augmented Generation (RAG), and replacing fragile "naive" text-to-SQL with a governed **Semantic View** and **Cortex Analyst**.

Everything runs inside Snowflake. No external API keys are required and no data leaves your security perimeter.

---

## Table of Contents

- [Overview](#overview)
- [Workshop Modules](#workshop-modules)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Snowflake Objects Created](#snowflake-objects-created)
- [Key Results](#key-results)
- [Using the RAG_App Class](#using-the-rag_app-class)
- [Cleanup](#cleanup)
- [Next Steps](#next-steps)
- [Resources](#resources)

---

## Overview

Most business questions fall into one of two categories, and each needs a different technique:

| Question | Data Type | Technique |
|----------|-----------|-----------|
| "What does our policy say about refunds?" | Text, documents | **RAG**: retrieve relevant chunks, ground the LLM |
| "What were total sales last quarter?" | Tables | **Text-to-SQL**: translate to SQL, execute, return the result |

This notebook builds both paths and unifies them in a single Python class, `RAG_App`.

**Snowflake features covered**

- `SNOWFLAKE.CORTEX.COMPLETE`: serverless LLM inference
- **Cortex Search**: managed hybrid (keyword + vector) search service
- **Cortex Analyst**: managed text-to-SQL driven by a semantic model
- **Semantic Views**: governed business definitions (metrics, dimensions, joins, synonyms)
- **Snowflake Cortex Code (CoCo)**: AI pair programmer for generating SQL and Python
- **Snowpark** and the `snowflake.core` Python API

## Workshop Modules

| Module | Topic | Time |
|--------|-------|------|
| 1 | Your AI Foundation: Cortex Complete and RAG concepts | 30 min |
| 2 | Production RAG with Cortex Search | 35 min |
| 3 | Text-to-SQL: from naive to production | 15 min |

### Module 1: Your AI Foundation
- Make a first LLM call with `CORTEX.COMPLETE` and compare `llama3.1-8b` against `llama3.1-70b`.
- Demonstrate **hallucination** by asking about private or recent facts the model cannot know.
- Apply **RAG grounding** using live data from `SNOWFLAKE_SAMPLE_DATA` (TPC-H).
- Create a 15-document text corpus of Snowflake feature descriptions (`FEATURE_DOCS`).

### Module 2: Production RAG with Cortex Search
- Chunk documents with overlap (`chunk_size=1500`, `overlap=200`) into `CHUNKED_DOCS`.
- Create a Cortex Search Service (`FEATURE_SEARCH_SERVICE`) using the `snowflake-arctic-embed-l-v2.0` embedding model.
- Build the `RAG_App` class: retrieve the top-k chunks, then generate a grounded answer.
- Test the app and **evaluate quality with an LLM judge** on three metrics: groundedness, context relevance, and answer relevance.

### Module 3: Text-to-SQL, from Naive to Production
- Establish gold-standard **Revenue** and **COGS** figures on TPC-H.
- Show that a schema-only LLM prompt produces SQL that **runs successfully but returns wrong numbers**.
- Create a **Semantic View** (`TPCH_ORDER_ANALYTICS`) that encodes metric definitions, join paths, and synonyms.
- Query it through **Cortex Analyst** with `app.query_data()` and verify the results match the gold standard.

## Architecture

**Unstructured path (RAG)**

```
FEATURE_DOCS → chunk text → CHUNKED_DOCS → Cortex Search Service → RAG_App.query() → Cortex Complete
```

**Structured path (Text-to-SQL)**

```
Natural-language question → Cortex Analyst → Semantic View → SQL → Results (pandas DataFrame)
```

## Prerequisites

- A Snowflake account (a trial account works) with access to **Cortex** features in your region
- Access to the `SNOWFLAKE_SAMPLE_DATA` shared database (pre-loaded in trial accounts)
- A role with privileges to create databases, schemas, warehouses, stages, Cortex Search Services, and Semantic Views
- **Snowflake Notebooks** in Snowsight, with the `snowflake` package (provides Snowpark and `snowflake.core`) plus `pandas` and `requests`
- Notebook environment setup from the lab guide: `AI_WORKSHOP_DB`, the `RAG_DATA` and `ANALYTICS` schemas, and the `WORKSHOP_WH` warehouse

If you are not following a lab guide, the following SQL provides the minimum setup the notebook expects (adjust the warehouse size as needed):

```sql
CREATE DATABASE IF NOT EXISTS AI_WORKSHOP_DB;
CREATE SCHEMA   IF NOT EXISTS AI_WORKSHOP_DB.RAG_DATA;
CREATE SCHEMA   IF NOT EXISTS AI_WORKSHOP_DB.ANALYTICS;
CREATE WAREHOUSE IF NOT EXISTS WORKSHOP_WH
  WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
```

## Getting Started

1. **Complete the setup** described under [Prerequisites](#prerequisites).
2. **Import the notebook** into Snowsight: *Projects → Notebooks → ⋮ → Import .ipynb file*, then select `accelerate-app-dev-cortex-code.ipynb`.
3. **Set the notebook's database, schema, and warehouse** to `AI_WORKSHOP_DB`, `RAG_DATA`, and `WORKSHOP_WH`.
4. **Run the cells in order, top to bottom.** Each step builds on the previous one.
5. *(Optional)* Try the **CoCo** demos in a SQL Worksheet using the sparkle icon (✦). The notebook includes a safety-net cell that creates the stage if you skip them.

> **Note:** The Cortex Search Service takes up to ~3 minutes to index. The notebook polls its status every 10 seconds. Wait for `Service is ACTIVE` before running queries. The naive COGS cell in Module 3 may take 1–3 minutes because it joins a 6M-row table to an 800K-row table.

## Snowflake Objects Created

| Object | Type | Location |
|--------|------|----------|
| `AI_WORKSHOP_DB.RAG_DATA.DOCS_STAGE` | Internal stage (directory table enabled, SSE encryption) | `RAG_DATA` |
| `FEATURE_DOCS` | Table: 15 product feature descriptions | `RAG_DATA` |
| `CHUNKED_DOCS` | Table: chunked text for indexing | `RAG_DATA` |
| `FEATURE_SEARCH_SERVICE` | Cortex Search Service (`TARGET_LAG = '1 minute'`) | `RAG_DATA` |
| `TPCH_ORDER_ANALYTICS` | Semantic View over TPC-H tables | `ANALYTICS` |

## Key Results

Module 3 demonstrates that the failure mode of naive text-to-SQL is not a query error. It is a **query that succeeds with wrong numbers**.

| | Naive LLM (schema only) | Cortex Analyst + Semantic View |
|---|---|---|
| **Revenue** | Used `O_TOTALPRICE`, about **4% inflated** (includes tax) | Correct formula every time |
| **COGS** | Computed revenue and labeled it "cost", about **2.8x too high** | Correct formula every time |
| SQL errors? | None | None |
| Deterministic? | No, varies between runs | Yes, same SQL every time |

**Metric definitions used as the gold standard**

```text
Revenue = SUM(L_EXTENDEDPRICE * (1 - L_DISCOUNT))
COGS    = SUM(PS_SUPPLYCOST   * L_QUANTITY)
```

Benchmark values for the `BUILDING` segment: revenue ≈ **$44.1B**, COGS ≈ **$15.5B**.

## Using the RAG_App Class

By the end of the notebook, a single object handles both data types:

```python
app = RAG_App(
    session,
    semantic_view='AI_WORKSHOP_DB.ANALYTICS.TPCH_ORDER_ANALYTICS'
)

# Unstructured: Cortex Search + LLM
app.query("What is the difference between Cortex Search and Cortex Analyst?")

# Structured: Cortex Analyst + Semantic View (returns a pandas DataFrame)
app.query_data("What is the total revenue by market segment?")
app.query_data("Which region has the highest profit margin?")
```

| Method | Backend | Returns |
|--------|---------|---------|
| `query(question)` | Cortex Search → `CORTEX.COMPLETE` (`llama3.1-70b`) | `str` |
| `retrieve_context(question)` | Cortex Search | `list[str]` of chunks |
| `generate_answer(question, context)` | `CORTEX.COMPLETE` | `str` |
| `query_data(question)` | Cortex Analyst REST API | `pandas.DataFrame` |

## Cleanup

To remove everything created by this workshop and stop incurring compute and search-service costs:

```sql
DROP CORTEX SEARCH SERVICE IF EXISTS AI_WORKSHOP_DB.RAG_DATA.FEATURE_SEARCH_SERVICE;
DROP DATABASE  IF EXISTS AI_WORKSHOP_DB;
DROP WAREHOUSE IF EXISTS WORKSHOP_WH;
```

## Next Steps

| Idea | Resource |
|------|----------|
| Upload real PDFs with `PARSE_DOCUMENT` | [Cortex Search docs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview) |
| Add a chat UI with Streamlit in Snowflake | [Streamlit docs](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit) |
| Build a unified agent that routes between Search and Analyst | [Getting started with Cortex Agents](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_agents) |

## Resources

- [Snowflake Cortex documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/overview)
- [Cortex Analyst](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst)
- [Semantic Views](https://docs.snowflake.com/en/user-guide/views-semantic/overview)

---

## Repository Contents

```text
.
├── snoflake-rag-dev-cortex-code.ipynb   # Workshop notebook
└── README.md                              # This file
```

## Disclaimer

This notebook is intended for **learning and demonstration**. The sample documents and TPC-H data are for illustration only. Model availability, function names, and Cortex features vary by region and change over time, so consult the Snowflake documentation for current details. Running the notebook consumes Snowflake credits (warehouse compute, Cortex LLM tokens, and the Cortex Search Service).
