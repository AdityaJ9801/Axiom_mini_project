### Complete Architecture, Dual-Mode Configuration & Agent Build Prompts

> **Version:** 1.0  
> **Architecture:** Hierarchical Multi-Agent Microservices  
> **Modes:** ⚡ Free/Local (Ollama) · 🚀 Paid/Cloud (Claude / Groq / OpenAI + AWS)  
> **Deployment:** Docker (both modes) · EC2 (paid mode option)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Core Design Principles](#2-core-design-principles)
3. [Dual-Mode Architecture](#3-dual-mode-architecture)
4. [Agent Hierarchy & Connectivity](#4-agent-hierarchy--connectivity)
5. [Build Phases & Timeline](#5-build-phases--timeline)
6. [Agent 01 — Context Intelligence Agent](#6-agent-01--context-intelligence-agent)
7. [Agent 02 — Orchestrator Agent](#7-agent-02--orchestrator-agent)
8. [Agent 03 — SQL Query Agent](#8-agent-03--sql-query-agent)
9. [Agent 04 — Visualization Agent](#9-agent-04--visualization-agent)
10. [Agent 05 — ML / Prediction Agent](#10-agent-05--ml--prediction-agent)
11. [Agent 06 — NLP / Text Agent](#11-agent-06--nlp--text-agent)
12. [Agent 07 — Report Synthesis Agent](#12-agent-07--report-synthesis-agent)
13. [Docker Compose — Free Mode](#13-docker-compose--free-mode)
14. [Docker Compose — Paid Mode](#14-docker-compose--paid-mode)
15. [Makefile Reference](#15-makefile-reference)
16. [Context Routing Matrix](#16-context-routing-matrix)
17. [Tech Stack Summary](#17-tech-stack-summary)

---

## 1. System Overview

Project is a multi-agent data analysis platform built as 7 independent FastAPI microservices. Each agent specialises in a specific domain (data profiling, SQL, ML, NLP, visualisation, reporting) and communicates through a central Orchestrator. The system runs entirely locally for free using Ollama, or switches to cloud LLM APIs for production-grade performance — the same Docker codebase, different `.env` files.

**The key architectural insight:** LLMs never receive raw dataset rows. Every dataset is first compressed into a ~800–4000 token **Context Object** (schema + statistics + relationships). All agents operate on this context fingerprint, not the data itself, preventing context window overflow on large datasets.

```
UI (React/Next.js)  →  Orchestrator :8000  →  Context Agent  :8001
                                            →  SQL Agent      :8002
                                            →  Viz Agent      :8003
                                            →  ML Agent       :8004
                                            →  NLP Agent      :8005
                                            →  Report Agent   :8006
```

The **Orchestrator is the only public-facing agent**. All others communicate exclusively on the internal Docker network `gemrslize-net`.

---

## 2. Core Design Principles

| Principle | Description |
|-----------|-------------|
| **Context Compression** | Raw data → compact JSON fingerprint (~800 tokens). LLMs never see raw rows. Handles datasets of any size. |
| **Hierarchical Routing** | Orchestrator decomposes user intent into a task graph and routes only the required context slice to each specialist agent. |
| **Parallel Execution** | Tier-1 agents run concurrently via `asyncio.gather`. SQL + ML + NLP work simultaneously on independent context slices. |
| **MCP Isolation** | Each agent owns its MCP servers. No shared tool access. Prevents cross-contamination of capabilities. |
| **Domain Plugins** | Agents load Python plugins (ydata-profiling, sklearn, spaCy) for heavy computation outside the LLM. |
| **Dual-Mode Runtime** | Same codebase. `LLM_PROVIDER` env var switches between Ollama (free) and Claude/Groq/OpenAI (paid). |
| **Feedback Loops** | Agents can request clarification from the Orchestrator, triggering re-routing or context enrichment cycles. |

### Context Compression — Why It Matters

**Without compression:**
```
user_id,revenue,region,date,...
1,5200,North,2024-01-01,...
2,1100,South,2024-01-02,...
... × 1,000,000 rows
→ 50M+ tokens, context overflow
```

**With compression:**
```json
{
  "rows": 1000000,
  "cols": 8,
  "columns": {
    "revenue": {
      "type": "float", "min": 100, "max": 99800,
      "mean": 5420, "nulls": "0.2%",
      "distribution": "right_skewed"
    },
    "region": {
      "type": "categorical", "cardinality": 4,
      "values": ["North","South","East","West"]
    }
  },
  "relationships": [{"cols":["user_id"],"type":"PK"}]
}
→ ~800 tokens ✅
```

---

## 3. Dual-Mode Architecture

### ⚡ FREE / LOCAL Mode

| Property | Value |
|----------|-------|
| **Cost** | $0 / month |
| **LLM** | Ollama (llama3.1:8b / mistral) — runs in Docker container |
| **Storage** | Local filesystem + DuckDB |
| **Database** | SQLite / local Parquet files via DuckDB |
| **Cloud** | None — 100% offline capable |
| **Requirements** | 8GB RAM, 20GB disk for Ollama models |
| **ML Mode** | `FAST_MODE=true` → scikit-learn (trains in seconds) |
| **Reports** | Saved to local `/app/reports` volume |
| **Charts** | Saved to local `/app/charts` volume |

**Strengths:** Zero cost · Privacy (data never leaves machine) · Works offline · Easy setup  
**Limitations:** Slower LLM inference (~3–5s per call) · No cloud storage · Single-machine only · No autoscaling

### 🚀 PAID / CLOUD Mode

| Property | Value |
|----------|-------|
| **Cost** | ~$50–500 / month depending on usage |
| **LLM** | Claude / OpenAI / Groq (API keys) |
| **Storage** | AWS S3 / GCS / Azure Blob |
| **Database** | PostgreSQL / BigQuery / Snowflake |
| **Cloud** | AWS EC2 + Docker + ALB |
| **Requirements** | API keys for services used |
| **ML Mode** | `FAST_MODE=false` → AutoGluon full AutoML |
| **Reports** | S3 bucket + presigned download URLs |
| **Charts** | S3 + CloudFront CDN |

**Strengths:** 10× faster inference · Unlimited scale · Production SLAs · Cloud DBs  
**Limitations:** API costs per request · Requires internet · Vendor dependency

### Per-Agent Mode Comparison

| Agent | Free LLM | Paid LLM | Free Storage | Paid Storage |
|-------|----------|----------|--------------|--------------|
| Context Intelligence | Ollama llama3.1:8b | Claude Haiku | Local disk + DuckDB | AWS S3 |
| Orchestrator | Ollama llama3.1:8b | Claude Sonnet | Redis (local Docker) | Redis Cloud / ElastiCache |
| SQL Query | Ollama llama3.1:8b | Groq llama-70b | DuckDB local files | PostgreSQL / BigQuery / Snowflake |
| Visualization | Ollama llama3.1:8b | Claude Haiku | Local `/app/charts` | S3 + CloudFront |
| ML / Prediction | Ollama llama3.1:8b | Claude Haiku | Local `/app/models` | S3 model bucket |
| NLP / Text | Ollama llama3.1:8b | Groq llama-70b | HuggingFace local cache | S3 + pgvector |
| Report Synthesis | Ollama llama3.1:8b | Claude Sonnet | Local `/app/reports` | S3 + presigned URLs |

> **LLM Selection Rationale (Paid Mode):**  
> Claude Sonnet → Orchestrator planning + Report narrative (best reasoning + writing)  
> Claude Haiku → Context Agent + Viz + ML (fast, cheap, sufficient)  
> Groq llama-70b → SQL Agent + NLP Agent (speed-critical tasks)

---

## 4. Agent Hierarchy & Connectivity

```
TIER 0 — ORCHESTRATION
─────────────────────────────────────────────────────────────────────
  🧠 Orchestrator Agent  :8000  (ONLY public port — UI connects here)
     - Receives user query + context_id
     - LLM decomposes intent into TaskGraph
     - Dispatches to Tier-1 agents in parallel
     - Assembles + returns unified response

              ↓ dispatches tasks ↓

TIER 1 — SPECIALIST AGENTS (run in parallel)
─────────────────────────────────────────────────────────────────────
  🔍 Context Intelligence  :8001   (data ingestion gateway)
  🗄️  SQL Query            :8002   (NL → SQL → result summary)
  📊 Visualization         :8003   (chart specs + PNG render)
  🤖 ML / Prediction       :8004   (AutoML + SHAP explanations)
  📝 NLP / Text            :8005   (sentiment, NER, topics, embeddings)

              ↓ structured insights ↓

TIER 2 — SYNTHESIS
─────────────────────────────────────────────────────────────────────
  📑 Report Synthesis      :8006   (narrative + PDF/DOCX/PPTX export)

SHARED INFRASTRUCTURE
─────────────────────────────────────────────────────────────────────
  Redis  :6379   (context cache + Celery broker + session state)
  Ollama :11434  (free mode only — local LLM inference)
```

### Docker Network: `gemrslize-net`

All agents communicate via Docker bridge network using service names as hostnames. No agent is reachable from outside except the Orchestrator on port 8000.

```
External traffic:  → Orchestrator :8000  (only exposed port)
Internal traffic:  context-agent:8001, sql-agent:8002, viz-agent:8003,
                   ml-agent:8004, nlp-agent:8005, report-agent:8006
Shared services:   redis:6379, ollama:11434 (free mode)
```

---

## 5. Build Phases & Timeline

### Phase 1 — Build Agents Independently (Weeks 1–10)

Each agent is built as a fully standalone microservice with its own `requirements.txt`, `Dockerfile`, `docker-compose.yml` (with Redis sidecar for local testing), `.env.example`, and test suite. No inter-agent dependencies during development.

| Week | Agent | Key deliverable |
|------|-------|-----------------|
| 1–2 | Context Intelligence | Profile pipeline, all connectors, ContextObject model |
| 2–3 | Orchestrator | TaskGraph model, LLM planner, parallel executor |
| 3–4 | SQL Query | NL→SQL, multi-dialect, safety layer, DuckDB |
| 4–5 | Visualization | Chart recommender, Plotly spec generator, Kaleido render |
| 5–7 | ML / Prediction | AutoGluon + sklearn, SHAP, Celery async jobs |
| 7–8 | NLP / Text | Transformers, spaCy, BERTopic, embedding pipeline |
| 8–10 | Report Synthesis | WeasyPrint PDF, python-docx, python-pptx, Jinja2 templates |

### Phase 2 — Integration & Wiring (Weeks 11–12)

- Master `docker-compose.free.yml` and `docker-compose.paid.yml`
- End-to-end flow tests: upload CSV → profile → query → visualize → report
- Context Object contract validation across all agents
- Health check monitoring for all 7 agents

### Phase 3 — UI Development (Weeks 13–16)

- React/Next.js frontend
- Communicates **only** with Orchestrator `:8000`
- File upload → profile → analyze flow
- Real-time SSE streaming for long operations
- Chart rendering (inline Plotly) + report download

### Phase 4 — Production Hardening (Weeks 17–18)

- EC2 deployment scripts per agent
- ALB + Route53 configuration
- Secrets → AWS Parameter Store (no `.env` in production)
- CloudWatch log forwarding
- Load testing each agent independently

---

## 6. Agent 01 — Context Intelligence Agent

**Port:** `8001` | **EC2:** `t3.medium` | **Role:** Schema Profiler & Context Compressor

This is the **entry point for all data**. It connects to any source, samples it intelligently, and produces a compact ContextObject used by every other agent. The LLM never sees raw rows.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/profile` | Profile a dataset from any source — returns ContextObject JSON |
| `POST` | `/profile/stream` | SSE streaming profile for large datasets (S3/GCS) |
| `GET` | `/context/{id}` | Retrieve cached context object by ID |
| `POST` | `/refresh/{id}` | Re-profile and update a cached context |
| `GET` | `/health` | Service health check |

### MCP Tools
- `file-reader-mcp`
- `db-introspection-mcp`
- `s3-connector-mcp`

### Domain Plugins
- `ydata-profiling`
- `Great Expectations`
- `DuckDB sampler`

### Dependencies (`requirements.txt`)
```
fastapi
uvicorn
pandas
polars
pyarrow
sqlalchemy
boto3
google-cloud-storage
paramiko
kafka-python
redis
ydata-profiling
great-expectations
duckdb
httpx
pydantic-settings
```

### Free Mode Config (`.env.free`)
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
STORAGE_TYPE=local
LOCAL_DATA_PATH=/app/data
DB_TYPE=duckdb
USE_DOCKER=true
USE_EC2=false
REDIS_URL=redis://redis:6379
PORT=8001
```
> Mount a `./data` folder — agent reads any file format from it. DuckDB handles all SQL-style profiling in-process. No cloud credentials needed.

### Paid Mode Config (`.env.paid`)
```env
LLM_PROVIDER=claude
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-haiku-4-5-20251001
STORAGE_TYPE=s3
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1
DB_TYPE=postgresql
DATABASE_URL=postgresql+asyncpg://user:pass@host/db
USE_DOCKER=true
USE_EC2=true
EC2_INSTANCE_TYPE=t3.medium
REDIS_URL=redis://redis:6379
PORT=8001
```
> Reads S3 glob patterns (`s3://bucket/path/*.parquet`). Claude Haiku keeps costs low — ~$0.001 per profile job.

### Docker Compose Snippet — Free
```yaml
context-agent:
  build: ./context-agent
  ports: ["8001:8001"]
  volumes:
    - ./data:/app/data        # mount local datasets here
    - context-cache:/app/cache
  environment:
    LLM_PROVIDER: ollama
    OLLAMA_BASE_URL: http://ollama:11434
    STORAGE_TYPE: local
    DB_TYPE: duckdb
    REDIS_URL: redis://redis:6379
  depends_on: [ollama, redis]
  networks: [gemrslize-net]
```

### Docker Compose Snippet — Paid
```yaml
context-agent:
  build: ./context-agent
  ports: ["8001:8001"]
  env_file: ./context-agent/.env.paid
  environment:
    REDIS_URL: redis://redis:6379
  depends_on: [redis]
  networks: [gemrslize-net]
```

### Project Structure
```
context-agent/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── models/
│   │   ├── context.py           # ContextObject Pydantic models
│   │   └── sources.py           # DataSource union models
│   ├── connectors/
│   │   ├── base.py
│   │   ├── csv_connector.py
│   │   ├── parquet_connector.py
│   │   ├── s3_connector.py
│   │   ├── gcs_connector.py
│   │   ├── sftp_connector.py
│   │   ├── postgres_connector.py
│   │   ├── mysql_connector.py
│   │   ├── bigquery_connector.py
│   │   ├── snowflake_connector.py
│   │   ├── kafka_connector.py
│   │   └── api_connector.py
│   ├── profilers/
│   │   ├── base_profiler.py
│   │   ├── schema_profiler.py
│   │   ├── stats_profiler.py
│   │   ├── cardinality_profiler.py
│   │   ├── relationship_profiler.py
│   │   └── anomaly_detector.py
│   ├── llm/
│   │   ├── provider.py
│   │   └── summarizer.py
│   ├── cache/
│   │   └── redis_cache.py
│   ├── routers/
│   │   ├── profile.py
│   │   └── context.py
│   └── utils/
│       ├── sampler.py
│       └── schema_utils.py
├── tests/
│   ├── test_connectors.py
│   ├── test_profilers.py
│   └── test_api.py
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── requirements.txt
└── README.md
```

### Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y gcc libpq-dev curl && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app/
COPY .env.example .env
EXPOSE 8001
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8001/health || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001", "--workers", "2"]
```

### Key Data Models

**DataSource union** (discriminated by `type` field):
```python
DataSource = Union[
  LocalFileSource,    # type="local_file" — CSV/TSV/Excel/Parquet/JSON/JSONL/Avro/ORC
  S3Source,           # type="s3"         — bucket + key (supports glob patterns)
  GCSSource,          # type="gcs"        — GCS bucket + blob path
  SFTPSource,         # type="sftp"       — SFTP host + remote path
  DatabaseSource,     # type="database"   — postgresql/mysql/bigquery/snowflake/sqlite/mssql
  KafkaSource,        # type="kafka"      — topic sampling (N latest messages)
  APISource,          # type="api"        — REST endpoint with pagination support
]
```

**ContextObject** (output — max 4KB):
```python
class ContextObject(BaseModel):
    context_id: str               # UUID
    created_at: datetime
    source_type: str
    source_fingerprint: str       # hash for cache key
    total_rows: int
    total_columns: int
    sampled_rows: int
    sample_pct: float
    columns: list[ColumnProfile]  # per-column stats
    relationships: list[RelationshipHint]
    quality_flags: list[DataQualityFlag]
    suggested_analyses: list[str]
    estimated_token_count: int
    raw_sample_preview: list[dict] | None  # first 3 rows
```

**ColumnProfile fields:** `name`, `dtype`, `semantic_type`, `nullable`, `null_pct`, `unique_count`, `cardinality` (low/medium/high/unique), `sample_values`, `stats` (min/max/mean/std/quartiles for numerics), `top_values` (for categoricals), `date_range`, `avg_length`, `has_pattern` (email/url/phone/uuid/zip detection)

### Smart Sampling Strategy
- Rows < 10,000 → load all rows
- Rows 10K–1M → 5% stratified sample (or random)
- Rows > 1M → max 50,000 rows via reservoir sampling
- Databases → `SELECT ... TABLESAMPLE SYSTEM (5)` or `ORDER BY RANDOM() LIMIT 50000`
- Parquet on S3 → read row group metadata first, sample specific groups
- Kafka → consume latest N messages with `auto.offset.reset='latest'`
- Time-series → ensure sample covers full date range

### LLM Provider Config (same pattern for all agents)
```python
class Settings(BaseSettings):
    LLM_PROVIDER: Literal["ollama","groq","claude","openai"] = "ollama"
    OLLAMA_BASE_URL: str = "http://localhost:11434"
    OLLAMA_MODEL: str = "llama3.1:8b"
    GROQ_API_KEY: str = ""
    GROQ_MODEL: str = "llama-3.1-70b-versatile"
    ANTHROPIC_API_KEY: str = ""
    CLAUDE_MODEL: str = "claude-haiku-4-5-20251001"
    OPENAI_API_KEY: str = ""
    OPENAI_MODEL: str = "gpt-4o-mini"
    USE_DOCKER: bool = True
    USE_EC2: bool = False
    EC2_INSTANCE_TYPE: str = "t3.medium"
    REDIS_URL: str = "redis://localhost:6379"
    CONTEXT_TTL_SECONDS: int = 3600
    PORT: int = 8001
    MAX_SAMPLE_ROWS: int = 50000
    ENABLE_LLM_SEMANTIC_NAMING: bool = True
```

### SSE Streaming Profile Events
```
event: progress
data: {"stage": "connecting", "pct": 10}

event: progress
data: {"stage": "sampling", "pct": 40, "rows_sampled": 12000}

event: progress
data: {"stage": "profiling", "pct": 75}

event: complete
data: { ...ContextObject... }
```

### EC2 Notes
- Instance: `t3.medium` (2 vCPU, 4GB RAM)
- Handles profiling up to 10M row datasets
- ContextObject hard cap: 4096 tokens — post-processing truncates `sample_values` and `top_values` if exceeded

### Coding Agent Prompt

```
# AGENT BUILD PROMPT — Context Intelligence Agent (Microservice 01)

## ROLE
You are an expert Python backend engineer. Build a production-ready FastAPI microservice
called the Context Intelligence Agent for the GEMRSLIZE multi-agent data analysis platform.

## CORE PURPOSE
This agent is the data ingestion gateway. It connects to ANY data source, samples it
intelligently, and produces a compact Context Object (JSON, max 4KB) that all other
agents use. The LLM never sees raw data rows — it only sees this context object.
This prevents context window overflow for large datasets.

## KEY REQUIREMENTS
1. Build all connector classes (LocalFile, S3, GCS, SFTP, Database, Kafka, API)
   as async classes inheriting from BaseConnector with a connect() + sample() method.
2. Implement adaptive sampling: <10K rows=full, 10K-1M=5% stratified, >1M=reservoir 50K.
3. Build ContextObject Pydantic model — must never exceed 4096 tokens. Add post-processing
   step to trim sample_values and top_values if size limit exceeded.
4. Implement LLM provider abstraction: provider.py factory returns unified async client
   for ollama/groq/claude/openai. LLM is used ONLY for semantic type naming + suggested_analyses.
5. All heavy profiling (stats, cardinality, nulls, relationships) done by Python libraries
   (polars, ydata-profiling, duckdb) — NOT by the LLM.
6. POST /profile/stream returns SSE events with progress stages: connecting→sampling→profiling→complete.
7. ContextObjects cached in Redis with CONTEXT_TTL_SECONDS TTL. Cache key = hash(source_config).
8. All errors must return structured error objects — no 500 crashes.
9. For Kafka: read-only, never commit offsets.
10. Support USE_DOCKER=true (service discovery via service names) and USE_EC2=true
    (direct IP-based connectivity with EC2 deployment script).

## BUILD ALL FILES listed in the project structure. Include tests for each connector
using mock/moto libraries, and integration tests using httpx AsyncClient.
```

---

## 7. Agent 02 — Orchestrator Agent

**Port:** `8000` | **EC2:** `t3.large` | **Role:** Master Coordinator & Task Router

The central intelligence hub. Receives user queries + `context_id`, uses an LLM to build a directed task graph, dispatches to specialist agents in parallel, and returns a unified response. Never processes data directly.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/analyze` | Main entry: user query + context_id → full analysis |
| `POST` | `/analyze/stream` | SSE streaming version of /analyze |
| `GET` | `/tasks/{task_id}` | Poll task status and partial results |
| `GET` | `/agents/status` | Check all downstream agent health |
| `POST` | `/plan` | Dry run — show task decomposition plan without executing |
| `GET` | `/health` | Service health |

### MCP Tools
- `orchestration-mcp`
- `memory-mcp`
- `agent-registry-mcp`

### Domain Plugins
- LangGraph planner
- Semantic intent router

### Dependencies (`requirements.txt`)
```
fastapi
uvicorn
httpx
pydantic-settings
redis
celery
langgraph
anthropic
openai
groq
ollama
tenacity
```

### Free Mode Config (`.env.free`)
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
REDIS_URL=redis://redis:6379
CONTEXT_AGENT_URL=http://context-agent:8001
SQL_AGENT_URL=http://sql-agent:8002
VIZ_AGENT_URL=http://viz-agent:8003
ML_AGENT_URL=http://ml-agent:8004
NLP_AGENT_URL=http://nlp-agent:8005
REPORT_AGENT_URL=http://report-agent:8006
USE_DOCKER=true
USE_EC2=false
PORT=8000
```
> Ollama handles task planning well. Expect ~3–5s latency for plan generation with llama3.1:8b vs <1s with Claude/Groq.

### Paid Mode Config (`.env.paid`)
```env
LLM_PROVIDER=claude
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-sonnet-4-20250514
REDIS_URL=redis://redis:6379
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls__...
CONTEXT_AGENT_URL=http://context-agent:8001
SQL_AGENT_URL=http://sql-agent:8002
VIZ_AGENT_URL=http://viz-agent:8003
ML_AGENT_URL=http://ml-agent:8004
NLP_AGENT_URL=http://nlp-agent:8005
REPORT_AGENT_URL=http://report-agent:8006
USE_DOCKER=true
USE_EC2=true
PORT=8000
```
> Claude Sonnet gives dramatically better task decomposition. Worth the ~$0.003/query cost. LangSmith optional but highly recommended for debugging.

### Project Structure
```
orchestrator-agent/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── models/
│   │   ├── request.py         # AnalyzeRequest, PlanRequest
│   │   ├── task.py            # Task, TaskGraph, TaskStatus
│   │   └── response.py        # AnalysisResult, AgentResult
│   ├── llm/
│   │   ├── provider.py
│   │   ├── intent_parser.py
│   │   └── planner.py         # Build task graph from intent + context
│   ├── routing/
│   │   ├── agent_registry.py  # URLs + capabilities of all agents
│   │   └── task_router.py
│   ├── execution/
│   │   ├── task_executor.py   # HTTP calls to downstream agents
│   │   ├── parallel_runner.py # asyncio.gather for parallel tasks
│   │   └── result_merger.py
│   ├── memory/
│   │   └── session_store.py   # Redis session + context cache
│   ├── routers/
│   │   ├── analyze.py
│   │   ├── tasks.py
│   │   └── agents.py
│   └── utils/
│       └── retry.py
├── tests/
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── requirements.txt
```

### Task Graph Model
```python
class Task(BaseModel):
    task_id: str
    task_type: TaskType          # query_data | visualize | train_model | predict |
                                 # extract_entities | summarize_text | generate_report
    agent: AgentType             # sql | viz | ml | nlp | report | context
    priority: int = 1
    depends_on: list[str] = []   # task_ids that must complete first
    payload: dict                # agent-specific request
    context_slice: dict          # minimal context this task needs
    timeout_seconds: int = 30
    status: Literal["pending","running","completed","failed"] = "pending"
    result: dict | None = None
    error: str | None = None
```

### Planning System Prompt
```
You are a data analysis orchestration planner. You receive:
1. A user's natural language analysis request
2. A dataset context object (schema, stats, column profiles)

Your job is to output a JSON task graph with the minimum set of tasks needed
to answer the user's request.

Available agents and their capabilities:
- sql_agent: Execute SQL queries, aggregations, joins, filters on structured data
- viz_agent: Create charts (bar, line, scatter, heatmap, histogram, pie, geo maps)
- ml_agent: Train models (classification, regression, clustering, anomaly detection, forecasting)
- nlp_agent: Sentiment analysis, entity extraction, topic modeling, text summarization
- report_agent: Generate narrative reports, executive summaries, export to PDF/DOCX

Rules:
- Only include tasks that are NECESSARY to answer the request
- Tasks with no dependencies should have empty depends_on (run in parallel)
- Each task's context_slice must contain ONLY the columns relevant to that task
- Never create more than 8 tasks for a single query
- Output ONLY valid JSON matching the TaskGraph schema
```

### Parallel Execution Engine
```python
async def execute_task_graph(graph: TaskGraph) -> list[Task]:
    completed = {}
    pending = graph.tasks.copy()

    while pending:
        ready = [t for t in pending if all(dep in completed for dep in t.depends_on)]
        if not ready:
            raise OrchestratorError("Circular dependency or unresolvable graph")

        results = await asyncio.gather(*[execute_task(t) for t in ready], return_exceptions=True)

        for task, result in zip(ready, results):
            if isinstance(result, Exception):
                task.status = "failed"
                task.error = str(result)
            else:
                task.status = "completed"
                task.result = result
            completed[task.task_id] = task
            pending.remove(task)

    return list(completed.values())
```

### Agent Registry
```python
AGENT_REGISTRY = {
    AgentType.CONTEXT:       {"base_url": "http://context-agent:8001", "timeout": 60},
    AgentType.SQL:           {"base_url": "http://sql-agent:8002",     "timeout": 30},
    AgentType.VISUALIZATION: {"base_url": "http://viz-agent:8003",     "timeout": 20},
    AgentType.ML:            {"base_url": "http://ml-agent:8004",      "timeout": 120},
    AgentType.NLP:           {"base_url": "http://nlp-agent:8005",     "timeout": 45},
    AgentType.REPORT:        {"base_url": "http://report-agent:8006",  "timeout": 60},
}
```

### EC2 Notes
- Instance: `t3.large` (2 vCPU, 8GB RAM)
- Use ALB in front for WebSocket/SSE support
- Set `--workers 1` (task state must stay in single process — use Redis for multi-worker state)

### Coding Agent Prompt

```
# AGENT BUILD PROMPT — Orchestrator Agent (Microservice 02)

## ROLE
You are an expert Python backend engineer. Build a production-ready FastAPI microservice
called the Orchestrator Agent for the GEMRSLIZE multi-agent data analysis platform.

## CORE PURPOSE
The Orchestrator receives user natural language queries + a context_id (pointing to a
pre-computed ContextObject from the Context Agent), uses an LLM to decompose into a
directed task graph, dispatches to specialist agents, collects results, returns unified
response. Never processes data directly.

## KEY REQUIREMENTS
1. Build LLM planning system that outputs valid TaskGraph JSON. Implement retry: if LLM
   returns invalid JSON, retry once with a correction prompt.
2. Implement parallel execution engine using asyncio.gather on tasks with no depends_on.
   Tasks with dependencies execute sequentially within the dependency chain.
3. Agent registry: read agent URLs from env vars (CONTEXT_AGENT_URL, SQL_AGENT_URL etc.)
   with fallback defaults to Docker service names.
4. Each agent call has configurable timeout (default 30s). On timeout: mark task as failed,
   continue with other tasks, include partial results in final response.
5. POST /analyze/stream returns SSE: stream task start/complete events as they happen,
   then stream the final assembled result.
6. POST /plan (dry run): return the task graph JSON without executing any agent calls.
7. GET /agents/status: call /health on each agent in parallel, return aggregated status.
8. Session state stored in Redis with 1h TTL. Include task_id in all responses for polling.
9. Support LLM_PROVIDER switching: Ollama for free mode (planning with llama3.1:8b),
   Claude Sonnet for paid mode (best task decomposition quality).
10. Use_EC2 mode: agent URLs use EC2 private IPs from env vars instead of Docker service names.

## BUILD all files in project structure. Unit test the intent parser with 10 sample queries.
Test parallel execution with mock agents via httpx MockTransport.
```

---

## 8. Agent 03 — SQL Query Agent

**Port:** `8002` | **EC2:** `t3.small` | **Role:** Structured Data Query Specialist

Translates NL + context into safe SQL, executes against target databases, returns compressed result summaries. Never returns raw result sets to the Orchestrator.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/query` | NL → SQL → execute → return summary |
| `POST` | `/query/explain` | Generate SQL + explain plan without executing |
| `POST` | `/query/raw` | Execute raw SQL directly (admin use) |
| `POST` | `/aggregate` | Compute aggregations from context spec |
| `GET` | `/health` | Service health |

### MCP Tools
- `postgres-mcp`
- `bigquery-mcp`
- `snowflake-mcp`
- `duckdb-mcp`

### Domain Plugins
- SQLGlot transpiler
- DuckDB local engine
- Query optimizer

### Dependencies (`requirements.txt`)
```
fastapi
uvicorn
sqlalchemy
asyncpg
aiomysql
duckdb
snowflake-connector-python
google-cloud-bigquery
sqlglot
anthropic
openai
groq
ollama
redis
tenacity
pydantic-settings
```

### Free Mode Config (`.env.free`)
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
DB_DIALECT=duckdb
DUCKDB_DATA_PATH=/app/data
USE_DOCKER=true
USE_EC2=false
REDIS_URL=redis://redis:6379
PORT=8002
```
> DuckDB directly queries local Parquet/CSV files with ANSI SQL. No database server needed. Supports files up to ~10GB on a laptop.

### Paid Mode Config (`.env.paid`)
```env
LLM_PROVIDER=groq
GROQ_API_KEY=gsk_...
GROQ_MODEL=llama-3.1-70b-versatile
DB_DIALECT=postgresql
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/db
# OR for BigQuery:
# BIGQUERY_PROJECT_ID=my-project
# BIGQUERY_DATASET=analytics
USE_DOCKER=true
USE_EC2=true
PORT=8002
```
> Groq is ideal here — SQL generation is fast and Groq's speed advantage is maximum. ~$0.0001 per query.

### SQL Generation Prompt
```
You are a senior SQL analyst. Generate a single SQL SELECT query to answer the user's request.

Database dialect: {dialect}
Available tables and schema: {schema_context}
Column details: {column_context}
User request: {task_description}

Rules:
1. Generate ONLY a SELECT query — no INSERT, UPDATE, DELETE, DROP, CREATE
2. Always include LIMIT {max_rows} unless user explicitly asks for all rows
3. Use column names exactly as shown in schema
4. For aggregations, always include meaningful column aliases
5. Prefer CTEs over nested subqueries for readability
6. Output ONLY the SQL query, no explanation, no markdown fences
```

### Result Compression Model
```python
class QueryResult(BaseModel):
    sql_generated: str
    dialect: str
    rows_returned: int
    rows_total: int | None           # from COUNT(*) if available
    execution_time_ms: int
    columns: list[str]
    data_preview: list[dict]         # first 5 rows
    summary_stats: dict              # per-column min/max/mean for numerics
    natural_language_summary: str    # LLM-generated 2-3 sentence summary
    has_more: bool
    result_token_estimate: int
```

### Safety Layer
```python
FORBIDDEN_SQL_KEYWORDS = [
    "INSERT", "UPDATE", "DELETE", "DROP", "CREATE", "ALTER",
    "TRUNCATE", "REPLACE", "MERGE", "EXEC", "EXECUTE",
    "GRANT", "REVOKE", "SET", "BEGIN", "COMMIT", "ROLLBACK"
]
```
SQL is validated before execution. Any forbidden keyword raises `SQLSafetyError`.

### Multi-Dialect Support (SQLGlot)
Supported: `postgresql`, `mysql`, `bigquery`, `snowflake`, `duckdb`, `sqlite`, `spark`

```python
import sqlglot
transpiled = sqlglot.transpile(sql, read="generic", write="snowflake")[0]
```

### SQL Retry on Error
If SQL execution fails, the error is fed back to the LLM for correction (one retry):
```python
async def generate_sql_with_retry(task, schema, dialect):
    sql = await llm.generate_sql(task, schema, dialect)
    try:
        validate_sql_safety(sql)
        await db.explain(sql)   # dry run
        return sql
    except Exception as e:
        corrected = await llm.correct_sql(sql, str(e), schema, dialect)
        validate_sql_safety(corrected)
        return corrected
```

### Coding Agent Prompt

```
# AGENT BUILD PROMPT — SQL Query Agent (Microservice 03)

## ROLE
You are an expert Python backend engineer. Build a production-ready FastAPI microservice
called the SQL Query Agent for the GEMRSLIZE multi-agent data analysis platform.

## CORE PURPOSE
Receives NL task + context slice from Orchestrator, generates accurate SQL, executes
against target DB, returns compressed result summary — never raw result sets.

## KEY REQUIREMENTS
1. Build connector classes for: PostgreSQL (asyncpg), MySQL (aiomysql), BigQuery,
   Snowflake, DuckDB (reads local Parquet/CSV), SQLite.
2. SQL safety validator: block all DML/DDL keywords. Raise SQLSafetyError on detection.
   Validation runs BEFORE every execution, including retries.
3. SQLGlot integration: when LLM generates generic SQL, transpile to target dialect
   before execution. Support all dialects listed above.
4. SQL retry: on execution error, feed error message back to LLM for correction.
   Max 1 retry. If retry fails, return structured error — do not crash.
5. Result compression: never return more than MAX_RESULT_ROWS (default 1000) rows.
   Always include natural_language_summary generated by LLM from first 20 rows.
6. Free mode: DB_DIALECT=duckdb reads from DUCKDB_DATA_PATH (mounted ./data folder).
   No external DB needed.
7. Paid mode: DB connection string from DATABASE_URL env var. Support connection pooling
   with asyncpg/aiomysql. Max pool size: 10 connections.
8. GET /query/explain: generate SQL + call EXPLAIN on DB, return plan as string.
   Do not execute the actual query.
9. LLM_PROVIDER switching: Groq for paid (fastest SQL gen), Ollama for free.

## BUILD all project files. Include SQL injection tests, dialect transpilation tests,
safety layer tests, and integration tests with SQLite.
```

---

## 9. Agent 04 — Visualization Agent

**Port:** `8003` | **EC2:** `t3.medium` | **Role:** Chart & Dashboard Specialist

Receives small aggregated data + visualization task, recommends chart types via rule-based logic + LLM, generates complete Plotly JSON specs, and optionally renders to PNG/SVG using Kaleido.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/chart` | Generate chart spec from data + intent |
| `POST` | `/chart/render` | Render chart spec to PNG/SVG/HTML |
| `POST` | `/recommend` | Recommend best chart types for given data |
| `POST` | `/dashboard` | Multi-chart dashboard layout spec |
| `GET` | `/health` | Service health |

### MCP Tools
- `plotly-mcp`
- `echarts-render-mcp`

### Domain Plugins
- Plotly renderer
- Apache ECharts
- Observable Plot

### Dependencies (`requirements.txt`)
```
fastapi
uvicorn
plotly
kaleido
pandas
pydantic-settings
anthropic
openai
groq
ollama
redis
httpx
```

### Free Mode Config (`.env.free`)
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
STORAGE_TYPE=local
CHART_OUTPUT_PATH=/app/charts
ENABLE_PNG_RENDER=true
USE_DOCKER=true
USE_EC2=false
PORT=8003
```

### Paid Mode Config (`.env.paid`)
```env
LLM_PROVIDER=claude
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-haiku-4-5-20251001
STORAGE_TYPE=s3
AWS_S3_CHARTS_BUCKET=gemrslize-charts
AWS_REGION=us-east-1
USE_S3_STORAGE=true
USE_DOCKER=true
USE_EC2=true
PORT=8003
```
> Charts stored in S3 and served via presigned URLs. Enables sharing and persistence across sessions.

### Chart Recommendation Logic
Rule-based (no LLM needed for common cases):

| Condition | Chart Type |
|-----------|-----------|
| datetime + numeric columns | `line` (>20 rows) or `bar` |
| "distribution" in task | `histogram` |
| categorical + numeric, ≤15 categories | `bar` |
| categorical + numeric, >15 categories | `table` |
| 2+ numerics + "correlat" in task, >5 cols | `heatmap` |
| 2+ numerics + "correlat" in task, ≤5 cols | `scatter` |
| "proportion/share/percent" + ≤8 categories | `pie` |
| fallback | `bar` |

### Plotly Spec Generation Prompt
```
You are a data visualization expert. Generate a complete Plotly.js JSON figure specification.

Chart type selected: {chart_type}
Task: {task}
Data columns: {columns}
Data sample (first 10 rows): {data_sample}
Color scheme: {color_scheme}

Requirements:
1. Output ONLY valid Plotly JSON with keys: "data" (traces array) and "layout"
2. Include professional layout: axis labels, title, legend, margins
3. For time series: use proper date format on x-axis
4. For bar charts with >10 categories: rotate x-axis labels 45°
5. Include hovertemplate for rich tooltips
6. Enterprise-ready: clean grid, proper font sizes, no cartoon styles
```

### Chart Request Model
```python
class ChartRequest(BaseModel):
    task: str
    data: DataPayload              # max 1000 rows (pre-aggregated)
    chart_type: str | None = None  # auto-select if None
    color_scheme: Literal["corporate","vibrant","pastel","monochrome","dark"] = "corporate"
    output_formats: list[Literal["spec","png","svg","html"]] = ["spec"]
    width: int = 900
    height: int = 500
```

### Coding Agent Prompt

```
# AGENT BUILD PROMPT — Visualization Agent (Microservice 04)

## ROLE
You are an expert Python backend engineer. Build a production-ready FastAPI microservice
called the Visualization Agent for the GEMRSLIZE multi-agent data analysis platform.

## CORE PURPOSE
Receives small aggregated dataset + visualization task from Orchestrator, uses an LLM
to select the best chart type and generate a complete Plotly chart specification,
returns spec JSON plus optional rendered image. Does NOT process raw datasets.

## KEY REQUIREMENTS
1. Build dual recommendation system: rule-based (chart_rules.py, no LLM, fast) and
   LLM-based (chart_selector.py, for complex/ambiguous tasks). Use rule-based first,
   fall back to LLM only when rules return None.
2. LLM generates complete Plotly JSON spec. Validate it is valid JSON with required
   "data" and "layout" keys before returning. Retry once on invalid JSON.
3. Kaleido PNG rendering: install chromium in Dockerfile. Render is async, run in
   executor to avoid blocking FastAPI event loop.
4. Free mode: STORAGE_TYPE=local — charts saved to CHART_OUTPUT_PATH, returned as
   base64 in response.
5. Paid mode: STORAGE_TYPE=s3 — upload PNG to S3, return presigned URL (7-day expiry).
6. Dashboard endpoint: accept list of ChartRequests, create Plotly subplots layout,
   return single HTML with all charts embedded via plotly.subplots.make_subplots.
7. Input validation: MAX_DATA_ROWS=1000. Reject payloads with more rows — return
   400 with message to pre-aggregate first.
8. Support 5 color schemes: corporate (blues/greys), vibrant, pastel, monochrome, dark.
   Include color_palettes.py with hex values for each scheme.

## BUILD all project files. Include tests for rule-based recommender, LLM spec
validation, and PNG render (mock kaleido in tests).
```

---

## 10. Agent 05 — ML / Prediction Agent

**Port:** `8004` | **EC2:** `t3.xlarge` | **Role:** Modeling, Forecasting & Explainability

Auto-selects ML models from feature context, trains via AutoGluon (paid) or scikit-learn (free), generates predictions, and returns structured metrics + SHAP-based feature importance explanations.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/train` | Auto-train model from context + data |
| `POST` | `/predict` | Run inference on new data |
| `POST` | `/cluster` | Unsupervised clustering |
| `POST` | `/forecast` | Time-series forecasting |
| `POST` | `/explain` | SHAP explainability for a trained model |
| `GET` | `/models` | List trained models in registry |
| `GET` | `/health` | Service health |

### MCP Tools
- `sklearn-mcp`
- `autogluon-mcp`
- `model-registry-mcp`

### Domain Plugins
- AutoGluon AutoML
- SHAP explainer
- Prophet forecasting

### Dependencies (`requirements.txt`)
```
fastapi
uvicorn
scikit-learn
autogluon
shap
prophet
pandas
numpy
joblib
redis
pydantic-settings
anthropic
openai
groq
ollama
boto3
```

### Free Mode Config (`.env.free`)
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
FAST_MODE=true
MODEL_STORAGE_TYPE=local
MODEL_STORAGE_PATH=/app/models
USE_S3_MODEL_STORAGE=false
USE_DOCKER=true
USE_EC2=false
REDIS_URL=redis://redis:6379
CELERY_BROKER_URL=redis://redis:6379/1
PORT=8004
```
> `FAST_MODE=true` skips AutoGluon and uses scikit-learn. Trains in seconds, not minutes. Ideal for demo/testing.

### Paid Mode Config (`.env.paid`)
```env
LLM_PROVIDER=claude
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-haiku-4-5-20251001
FAST_MODE=false
MODEL_STORAGE_TYPE=s3
USE_S3_MODEL_STORAGE=true
S3_MODEL_BUCKET=gemrslize-models
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
USE_DOCKER=true
USE_EC2=true
EC2_INSTANCE_TYPE=t3.xlarge
CELERY_BROKER_URL=redis://redis:6379/1
PORT=8004
```
> AutoGluon trains dozens of models and selects the best. Requires `t3.xlarge` minimum. Models saved to S3 for persistence and reuse.

### Docker Compose Snippet — Paid (includes Celery worker)
```yaml
ml-agent:
  build: ./ml-agent
  ports: ["8004:8004"]
  volumes: [ml-models:/app/models]
  env_file: ./ml-agent/.env.paid
  environment:
    REDIS_URL: redis://redis:6379
    CELERY_BROKER_URL: redis://redis:6379/1
  depends_on: [redis]
  networks: [gemrslize-net]

ml-worker:
  build: ./ml-agent
  command: celery -A app.workers.celery_tasks worker --loglevel=info -c 2
  volumes: [ml-models:/app/models]
  env_file: ./ml-agent/.env.paid
  environment:
    REDIS_URL: redis://redis:6379
    CELERY_BROKER_URL: redis://redis:6379/1
  depends_on: [redis]
  networks: [gemrslize-net]
```

### ML Task Classification Prompt
```
Given this analysis task and dataset context, determine the ML problem type.

Task: {task}
Target column (if mentioned): {target}
Available features: {features_summary}

Respond with JSON only:
{
  "problem_type": "binary_classification" | "multiclass_classification" |
                  "regression" | "clustering" | "time_series_forecast" | "anomaly_detection",
  "target_column": "column_name or null",
  "feature_columns": ["list of relevant feature columns"],
  "recommended_algorithms": ["algorithm1", "algorithm2"],
  "preprocessing_needed": ["encoding", "scaling", "imputation"],
  "reasoning": "brief explanation"
}
```

### sklearn Model Map (FAST_MODE=true)
```python
SKLEARN_MODEL_MAP = {
    "binary_classification":    [RandomForestClassifier, GradientBoostingClassifier, LogisticRegression],
    "multiclass_classification": [RandomForestClassifier, GradientBoostingClassifier],
    "regression":               [RandomForestRegressor, GradientBoostingRegressor, Ridge],
    "clustering":               [KMeans, DBSCAN],
    "anomaly_detection":        [IsolationForest, LocalOutlierFactor],
}
```

### ModelResult Structure
```python
class ModelResult(BaseModel):
    task_id: str
    problem_type: str
    best_model_name: str
    metrics: dict                   # accuracy/f1/rmse etc.
    feature_importance: list[dict]  # [{feature, importance_score}]
    shap_summary: dict | None
    predictions_sample: list        # first 10 predictions
    training_time_seconds: float
    model_path: str
    natural_language_summary: str   # LLM-generated summary
    recommendations: list[str]      # LLM next steps
```

### EC2 Notes
- Minimum: `t3.xlarge` (4 vCPU, 16GB RAM) for AutoGluon
- Large datasets (>500K rows): `c5.2xlarge` or `m5.2xlarge`
- Mount EBS at `/app/models` for persistence
- GPU option: `g4dn.xlarge` for deep learning models

### Coding Agent Prompt

```
# AGENT BUILD PROMPT — ML/Prediction Agent (Microservice 05)

## ROLE
You are an expert Python ML engineer. Build a production-ready FastAPI microservice
called the ML/Prediction Agent for the GEMRSLIZE multi-agent data analysis platform.

## CORE PURPOSE
Receives a task description + context slice from Orchestrator, uses LLM to determine
ML problem type, fetches encoded features, trains models, returns structured metrics
+ SHAP explanations. Heavy computation runs in Celery background tasks.

## KEY REQUIREMENTS
1. Build ML task interpreter: LLM classifies task as binary_classification,
   multiclass_classification, regression, clustering, time_series_forecast,
   or anomaly_detection. Output structured JSON with target/feature columns.
2. FAST_MODE=true (free): use scikit-learn SKLEARN_MODEL_MAP, train all models
   in the mapped list, return best by cross-validation score. Trains in <30s.
3. FAST_MODE=false (paid): use AutoGluon TabularPredictor with time_limit=120s,
   presets="medium_quality". Return leaderboard top 5 + feature importance.
4. SHAP explainability: run shap.Explainer on max 100 rows sample. Return
   feature importance sorted by mean absolute SHAP value. LLM narrates top 3 features.
5. Celery async for jobs >30s: POST /train returns task_id immediately.
   Client polls GET /tasks/{task_id}. Store results in Redis.
6. Model persistence: FAST_MODE=false saves to MODEL_STORAGE_PATH (local) or S3.
   GET /models lists all saved models with metadata.
7. Prophet for time_series_forecast: detect datetime index column automatically.
   Return forecast for next 30 periods by default (configurable).
8. Free mode Celery: broker = redis://redis:6379/1 — same Redis container.
9. EC2 mode: mount EBS volume at /app/models. Use S3 for model artifacts.

## BUILD all project files. Include tests using synthetic datasets for each
problem type. Mock AutoGluon in unit tests (it's slow).
```

---

## 11. Agent 06 — NLP / Text Agent

**Port:** `8005` | **EC2:** `t3.large` | **Role:** Unstructured Text & Semantic Specialist

Processes text columns and documents using specialized NLP pipelines. Chunks and batches all text processing — raw text is never bulk-fed to LLMs.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/sentiment` | Batch sentiment analysis on text column |
| `POST` | `/entities` | Named entity recognition (NER) |
| `POST` | `/topics` | Topic modeling (LDA / BERTopic) |
| `POST` | `/classify` | Zero-shot text classification |
| `POST` | `/embed` | Generate embeddings for semantic search |
| `POST` | `/summarize` | Summarize document or text column |
| `GET` | `/health` | Service health |

### MCP Tools
- `embeddings-mcp`
- `spacy-mcp`
- `vector-store-mcp`

### Domain Plugins
- BERTopic
- spaCy NER
- LlamaIndex chunker
- SentenceTransformers

### Dependencies (`requirements.txt`)
```
fastapi
uvicorn
transformers
sentence-transformers
spacy
bertopic
langchain-text-splitters
llama-index
redis
pydantic-settings
anthropic
openai
groq
ollama
numpy
scikit-learn
```

### Free Mode Config (`.env.free`)
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
SPACY_MODEL=en_core_web_sm
SENTIMENT_MODEL=cardiffnlp/twitter-roberta-base-sentiment-latest
EMBEDDING_MODEL=all-MiniLM-L6-v2
HF_CACHE_DIR=/app/hf_cache
USE_DOCKER=true
USE_EC2=false
REDIS_URL=redis://redis:6379
PORT=8005
```
> First startup downloads HuggingFace models (~400MB total). Mount `hf-cache` volume to persist across restarts.

### Paid Mode Config (`.env.paid`)
```env
LLM_PROVIDER=groq
GROQ_API_KEY=gsk_...
GROQ_MODEL=llama-3.1-70b-versatile
SPACY_MODEL=en_core_web_lg
EMBEDDING_MODEL=all-mpnet-base-v2
HF_CACHE_DIR=/app/hf_cache
VECTOR_STORE_TYPE=pgvector
PGVECTOR_URL=postgresql://user:pass@host/db
USE_DOCKER=true
USE_EC2=true
PORT=8005
```
> Groq handles long document summaries efficiently. pgvector enables semantic search across thousands of documents.

### Batch Processing Pattern
```python
async def process_text_column(texts: list[str], pipeline_fn, batch_size: int = 32):
    results = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        batch_results = await asyncio.get_event_loop().run_in_executor(
            None, pipeline_fn, batch
        )
        results.extend(batch_results)
    return results
```
Text is always processed in batches of 32. Never row-by-row.

### NLP Pipeline Models

| Pipeline | Model (Free) | Model (Paid) |
|----------|-------------|--------------|
| Sentiment | `cardiffnlp/twitter-roberta-base-sentiment-latest` | Same (sufficient) |
| NER | `spaCy en_core_web_sm` | `spaCy en_core_web_lg` |
| Embeddings | `all-MiniLM-L6-v2` | `all-mpnet-base-v2` |
| Topic Modeling | BERTopic (local) | BERTopic (local) |
| Summarization | Ollama llama3.1:8b | Groq llama-70b |
| Zero-shot | Ollama llama3.1:8b | Groq llama-70b |

### Document Summarization (Map-Reduce)
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

async def summarize_document(text: str) -> str:
    splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)
    chunks = splitter.split_text(text)

    if len(chunks) == 1:
        return await llm.summarize(chunks[0])

    # Map: summarize each chunk in parallel (max 10 chunks)
    chunk_summaries = await asyncio.gather(*[llm.summarize(c) for c in chunks[:10]])
    # Reduce: summarize the summaries
    return await llm.summarize("\n\n".join(chunk_summaries), mode="final")
```

### Coding Agent Prompt

```
# AGENT BUILD PROMPT — NLP/Text Agent (Microservice 06)

## ROLE
You are an expert Python NLP engineer. Build a production-ready FastAPI microservice
called the NLP/Text Agent for the GEMRSLIZE multi-agent data analysis platform.

## CORE PURPOSE
Handles all unstructured text data — text columns in datasets, documents, free-text
fields. Processes through specialized NLP pipelines (spaCy, Transformers, BERTopic)
and returns structured semantic summaries. Text is chunked and batched — never bulk-fed
to LLMs.

## KEY REQUIREMENTS
1. All NLP pipelines must be initialized once at startup (lazy-loaded singletons).
   Use @app.on_event("startup") to load spaCy, Transformers, SentenceTransformer models.
   Never reload models per-request.
2. All text processing in batches of NLP_BATCH_SIZE=32 using run_in_executor to avoid
   blocking the FastAPI event loop.
3. Sentiment pipeline: use HuggingFace Transformers with cardiffnlp model. Aggregate
   per-row results to column-level distribution (positive/negative/neutral counts + pcts).
4. NER pipeline: use spaCy. Aggregate all entities to type-level summary with top 10
   values per entity type and total counts.
5. BERTopic for topic modeling: nr_topics=10, min_topic_size=5. Return topic keywords,
   sizes, and outlier percentage.
6. Summarization: implement map-reduce chunking using LangChain RecursiveTextSplitter
   (chunk_size=2000, overlap=200). Max 10 chunks → parallel summarize → reduce.
7. SentenceTransformers for /embed endpoint: batch encode, return embeddings as list[list[float]].
   In paid mode: store to pgvector for semantic search.
8. text_cleaner.py: strip HTML tags, normalize whitespace, detect language, truncate at
   MAX_TEXT_LENGTH=10000 chars.
9. Free mode: HF_CACHE_DIR persisted via Docker volume. Include Dockerfile RUN step to
   pre-download spaCy model: python -m spacy download en_core_web_sm.
10. Paid mode: SPACY_MODEL=en_core_web_lg, EMBEDDING_MODEL=all-mpnet-base-v2 (higher quality).

## BUILD all project files. Include tests using sample text datasets for each pipeline.
Mock HuggingFace model calls in unit tests.
```

---

## 12. Agent 07 — Report Synthesis Agent

**Port:** `8006` | **EC2:** `t3.medium` | **Role:** Narrative Generator & Document Exporter

The final output layer. Aggregates structured results from all specialist agents, uses an LLM to write coherent narrative sections, and assembles enterprise-quality documents in PDF, DOCX, or PPTX format.

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/report` | Generate full analysis report from agent outputs |
| `POST` | `/summary` | Generate executive summary (short form) |
| `POST` | `/export/pdf` | Export report to PDF |
| `POST` | `/export/docx` | Export report to DOCX |
| `POST` | `/export/pptx` | Export to PowerPoint slide deck |
| `GET` | `/report/{id}` | Retrieve cached report |
| `GET` | `/health` | Service health |

### MCP Tools
- `docx-mcp`
- `pdf-render-mcp`
- `pptx-mcp`

### Domain Plugins
- WeasyPrint PDF
- python-docx
- python-pptx
- Jinja2 templates

### Dependencies (`requirements.txt`)
```
fastapi
uvicorn
python-docx
reportlab
weasyprint
python-pptx
jinja2
anthropic
openai
groq
ollama
redis
pydantic-settings
boto3
```

### Free Mode Config (`.env.free`)
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
STORAGE_TYPE=local
REPORT_OUTPUT_PATH=/app/reports
USE_S3_STORAGE=false
USE_DOCKER=true
USE_EC2=false
REDIS_URL=redis://redis:6379
PORT=8006
```
> Report quality is slightly lower with Ollama vs Claude for narrative writing. Use `mistral:7b-instruct` as alternative — better writing than llama for this task.

### Paid Mode Config (`.env.paid`)
```env
LLM_PROVIDER=claude
ANTHROPIC_API_KEY=sk-ant-...
CLAUDE_MODEL=claude-sonnet-4-20250514
STORAGE_TYPE=s3
USE_S3_STORAGE=true
S3_REPORTS_BUCKET=gemrslize-reports
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
USE_DOCKER=true
USE_EC2=true
PORT=8006
```
> Claude Sonnet produces noticeably better executive summaries. Reports stored in S3 with 7-day presigned download URLs.

### Report Request Model
```python
class AgentOutputBundle(BaseModel):
    context_summary: dict          # From Context Agent
    sql_results: list[dict] | None # From SQL Agent
    charts: list[dict] | None      # From Viz Agent (Plotly specs + PNG base64)
    ml_results: dict | None        # From ML Agent
    nlp_insights: dict | None      # From NLP Agent
    user_query: str
    analysis_timestamp: datetime

class ReportRequest(BaseModel):
    bundle: AgentOutputBundle
    report_style: Literal["executive","technical","detailed"] = "detailed"
    include_charts: bool = True
    export_format: Literal["json","pdf","docx","pptx","html"] = "json"
    branding: dict | None = None   # {company_name, logo_base64, colors}
    max_pages: int = 20
```

### Narrative Writing Prompts
Each section is written independently by the LLM:

- **data_overview:** 2-paragraph non-technical description of dataset structure and quality
- **sql_findings:** 2-3 paragraphs on query results, patterns, business implications
- **ml_insights:** What the model found + what actions the business should take
- **executive_summary:** 150-word C-suite summary leading with the most impactful finding

### Dockerfile (WeasyPrint requires system libs)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y gcc pango1.0-tools libpango-1.0-0 \
    libpangocairo-1.0-0 libcairo2 libgdk-pixbuf2.0-0 libffi-dev shared-mime-info curl \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app/
COPY templates/ ./templates/
COPY static/ ./static/
EXPOSE 8006
HEALTHCHECK --interval=30s CMD curl -f http://localhost:8006/health || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8006", "--workers", "2"]
```

### EC2 Notes
- Instance: `t3.medium` (WeasyPrint/pango needs ~1GB RAM for complex PDFs)
- For heavy pptx workloads: `t3.large`
- Pango/Cairo system libraries must be installed (included in Dockerfile)

### Coding Agent Prompt

```
# AGENT BUILD PROMPT — Report Synthesis Agent (Microservice 07)

## ROLE
You are an expert Python backend engineer. Build a production-ready FastAPI microservice
called the Report Synthesis Agent for the GEMRSLIZE multi-agent data analysis platform.

## CORE PURPOSE
Final output layer. Receives structured results from all specialist agents, uses LLM
to write coherent narrative sections, assembles enterprise-quality PDF/DOCX/PPTX.

## KEY REQUIREMENTS
1. Build AgentOutputBundle model: accepts optional fields from each agent (sql_results,
   charts, ml_results, nlp_insights). Handle missing agent outputs gracefully — skip
   sections where data is None.
2. LLM narrative writers (separate prompt per section): data_overview, sql_findings,
   ml_insights, nlp_section, executive_summary. Each call is independent. Use Claude
   Sonnet in paid mode for best writing quality.
3. WeasyPrint PDF: use Jinja2 templates in app/templates/. Include base_report.html.j2
   as main template with {% include %} for section partials. Charts embedded as base64
   <img> tags. CSS stylesheet at static/styles/report.css with professional typography.
4. python-docx DOCX: proper heading styles, embedded chart images using doc.add_picture(),
   formatted data tables using add_formatted_table() helper, page breaks between sections.
5. python-pptx PPTX: 16:9 format (13.33 × 7.5 inches). One slide per section.
   Title slide + executive summary + section slides + key takeaways slide.
   Embed chart PNGs on slides.
6. Storage: free mode saves files to REPORT_OUTPUT_PATH, returns base64 in response.
   Paid mode uploads to S3, returns presigned URL (7-day expiry).
7. Redis cache: store report metadata with 1h TTL. GET /report/{id} retrieves from cache
   or returns 404.
8. Branding support: accept {company_name, logo_base64, primary_color} in request.
   Apply to all export formats (header logo, color accents in PDF/PPTX).
9. WeasyPrint requires Pango/Cairo system libs — ensure Dockerfile installs them.

## BUILD all project files including Jinja2 HTML templates and CSS stylesheet.
Include tests using mock LLM responses and verify PDF/DOCX/PPTX file generation.
```

---

## 13. Docker Compose — Free Mode

**File:** `docker-compose.free.yml`  
**Prerequisites:** Docker Desktop (4GB+ RAM allocated), 20GB free disk  
**First run:** `docker-compose -f docker-compose.free.yml up --build`  
**Model pull:** `docker exec -it ollama ollama pull llama3.1:8b`

```yaml
version: '3.9'

networks:
  gemrslize-net:
    driver: bridge

volumes:
  ollama-data:      # persists Ollama models (~5GB)
  redis-data:
  local-data:       # put your CSV/Parquet files here
  hf-cache:         # HuggingFace model cache (~400MB)
  ml-models:
  chart-output:
  report-output:
  context-cache:

services:

  # ── LLM (Ollama — runs models locally) ──────────────────────
  ollama:
    image: ollama/ollama:latest
    ports: ["11434:11434"]
    volumes:
      - ollama-data:/root/.ollama
    networks: [gemrslize-net]
    deploy:
      resources:
        limits:
          memory: 6G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ── Cache / Queue ────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes: [redis-data:/data]
    networks: [gemrslize-net]
    command: redis-server --appendonly yes

  # ── Context Intelligence Agent ───────────────────────────────
  context-agent:
    build: ./context-agent
    ports: ["8001:8001"]
    volumes:
      - ./data:/app/data          # ← DROP YOUR DATA FILES HERE
      - context-cache:/app/cache
    environment:
      LLM_PROVIDER: ollama
      OLLAMA_BASE_URL: http://ollama:11434
      OLLAMA_MODEL: llama3.1:8b
      STORAGE_TYPE: local
      LOCAL_DATA_PATH: /app/data
      DB_TYPE: duckdb
      REDIS_URL: redis://redis:6379
    depends_on:
      ollama: {condition: service_healthy}
      redis: {condition: service_started}
    networks: [gemrslize-net]

  # ── Orchestrator ─────────────────────────────────────────────
  orchestrator:
    build: ./orchestrator-agent
    ports: ["8000:8000"]           # ONLY public-facing port
    environment:
      LLM_PROVIDER: ollama
      OLLAMA_BASE_URL: http://ollama:11434
      OLLAMA_MODEL: llama3.1:8b
      REDIS_URL: redis://redis:6379
      CONTEXT_AGENT_URL: http://context-agent:8001
      SQL_AGENT_URL: http://sql-agent:8002
      VIZ_AGENT_URL: http://viz-agent:8003
      ML_AGENT_URL: http://ml-agent:8004
      NLP_AGENT_URL: http://nlp-agent:8005
      REPORT_AGENT_URL: http://report-agent:8006
    depends_on: [ollama, redis]
    networks: [gemrslize-net]

  # ── SQL Query Agent ──────────────────────────────────────────
  sql-agent:
    build: ./sql-agent
    ports: ["8002:8002"]
    volumes:
      - ./data:/app/data          # same data mount
    environment:
      LLM_PROVIDER: ollama
      OLLAMA_BASE_URL: http://ollama:11434
      DB_DIALECT: duckdb
      DUCKDB_DATA_PATH: /app/data
      REDIS_URL: redis://redis:6379
    depends_on: [ollama, redis]
    networks: [gemrslize-net]

  # ── Visualization Agent ──────────────────────────────────────
  viz-agent:
    build: ./viz-agent
    ports: ["8003:8003"]
    volumes: [chart-output:/app/charts]
    environment:
      LLM_PROVIDER: ollama
      OLLAMA_BASE_URL: http://ollama:11434
      STORAGE_TYPE: local
      CHART_OUTPUT_PATH: /app/charts
    depends_on: [ollama]
    networks: [gemrslize-net]

  # ── ML / Prediction Agent ────────────────────────────────────
  ml-agent:
    build: ./ml-agent
    ports: ["8004:8004"]
    volumes: [ml-models:/app/models]
    environment:
      LLM_PROVIDER: ollama
      OLLAMA_BASE_URL: http://ollama:11434
      FAST_MODE: "true"
      MODEL_STORAGE_TYPE: local
      MODEL_STORAGE_PATH: /app/models
      REDIS_URL: redis://redis:6379
      CELERY_BROKER_URL: redis://redis:6379/1
    depends_on: [ollama, redis]
    networks: [gemrslize-net]

  # ── NLP / Text Agent ─────────────────────────────────────────
  nlp-agent:
    build: ./nlp-agent
    ports: ["8005:8005"]
    volumes: [hf-cache:/app/hf_cache]
    environment:
      LLM_PROVIDER: ollama
      OLLAMA_BASE_URL: http://ollama:11434
      HF_CACHE_DIR: /app/hf_cache
      REDIS_URL: redis://redis:6379
    depends_on: [ollama, redis]
    networks: [gemrslize-net]

  # ── Report Synthesis Agent ───────────────────────────────────
  report-agent:
    build: ./report-agent
    ports: ["8006:8006"]
    volumes: [report-output:/app/reports]
    environment:
      LLM_PROVIDER: ollama
      OLLAMA_BASE_URL: http://ollama:11434
      STORAGE_TYPE: local
      REPORT_OUTPUT_PATH: /app/reports
      REDIS_URL: redis://redis:6379
    depends_on: [ollama, redis]
    networks: [gemrslize-net]
```

---

## 14. Docker Compose — Paid Mode

**File:** `docker-compose.paid.yml`  
**Prerequisites:** API keys in `.env.paid` files, Docker, AWS CLI configured  
**Run:** `docker-compose -f docker-compose.paid.yml up --build`

```yaml
version: '3.9'

networks:
  gemrslize-net:
    driver: bridge

volumes:
  redis-data:
  hf-cache:
  ml-models:
  report-output:

services:

  # ── Cache / Queue (no Ollama in paid mode) ───────────────────
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes: [redis-data:/data]
    networks: [gemrslize-net]

  context-agent:
    build: ./context-agent
    ports: ["8001:8001"]
    env_file: ./context-agent/.env.paid
    environment:
      REDIS_URL: redis://redis:6379
    depends_on: [redis]
    networks: [gemrslize-net]

  orchestrator:
    build: ./orchestrator-agent
    ports: ["8000:8000"]
    env_file: ./orchestrator-agent/.env.paid
    environment:
      REDIS_URL: redis://redis:6379
      CONTEXT_AGENT_URL: http://context-agent:8001
      SQL_AGENT_URL: http://sql-agent:8002
      VIZ_AGENT_URL: http://viz-agent:8003
      ML_AGENT_URL: http://ml-agent:8004
      NLP_AGENT_URL: http://nlp-agent:8005
      REPORT_AGENT_URL: http://report-agent:8006
    depends_on: [redis]
    networks: [gemrslize-net]

  sql-agent:
    build: ./sql-agent
    ports: ["8002:8002"]
    env_file: ./sql-agent/.env.paid
    environment:
      REDIS_URL: redis://redis:6379
    depends_on: [redis]
    networks: [gemrslize-net]

  viz-agent:
    build: ./viz-agent
    ports: ["8003:8003"]
    env_file: ./viz-agent/.env.paid
    networks: [gemrslize-net]

  ml-agent:
    build: ./ml-agent
    ports: ["8004:8004"]
    volumes: [ml-models:/app/models]
    env_file: ./ml-agent/.env.paid
    environment:
      REDIS_URL: redis://redis:6379
      CELERY_BROKER_URL: redis://redis:6379/1
    depends_on: [redis]
    networks: [gemrslize-net]

  # Celery worker for async ML training jobs (paid mode only)
  ml-worker:
    build: ./ml-agent
    command: celery -A app.workers.celery_tasks worker --loglevel=info -c 2
    volumes: [ml-models:/app/models]
    env_file: ./ml-agent/.env.paid
    environment:
      REDIS_URL: redis://redis:6379
      CELERY_BROKER_URL: redis://redis:6379/1
    depends_on: [redis]
    networks: [gemrslize-net]

  nlp-agent:
    build: ./nlp-agent
    ports: ["8005:8005"]
    volumes: [hf-cache:/app/hf_cache]
    env_file: ./nlp-agent/.env.paid
    environment:
      REDIS_URL: redis://redis:6379
    depends_on: [redis]
    networks: [gemrslize-net]

  report-agent:
    build: ./report-agent
    ports: ["8006:8006"]
    volumes: [report-output:/app/reports]
    env_file: ./report-agent/.env.paid
    environment:
      REDIS_URL: redis://redis:6379
    depends_on: [redis]
    networks: [gemrslize-net]
```

---

## 15. Makefile Reference

Place this `Makefile` in the project root directory.

```makefile
.PHONY: free paid setup-free setup-paid stop clean logs pull-model status

# ── Start free/local mode ──────────────────────────────────────
free:
	docker-compose -f docker-compose.free.yml up --build -d
	@echo "\n✅ GEMRSLIZE started in FREE mode"
	@echo "   Orchestrator: http://localhost:8000"
	@echo "   Run 'make pull-model' on first startup"

# Pull Ollama model (run once after first 'make free')
pull-model:
	docker exec -it $$(docker-compose -f docker-compose.free.yml ps -q ollama) \
	  ollama pull llama3.1:8b
	@echo "✅ Model ready"

# ── Start paid/cloud mode ──────────────────────────────────────
paid:
	@test -f context-agent/.env.paid || \
	  (echo "❌ Missing .env.paid files. Run: make setup-paid" && exit 1)
	docker-compose -f docker-compose.paid.yml up --build -d
	@echo "\n✅ GEMRSLIZE started in PAID mode"
	@echo "   Orchestrator: http://localhost:8000"

# ── Setup helpers ──────────────────────────────────────────────
setup-free:
	@mkdir -p data
	@echo "📁 Created ./data — drop your CSV/Parquet files here"
	@echo "✅ Free mode config ready — run: make free"

setup-paid:
	@for agent in context-agent orchestrator-agent sql-agent viz-agent \
	              ml-agent nlp-agent report-agent; do \
	  cp $$agent/.env.example $$agent/.env.paid 2>/dev/null || true; \
	  echo "📝 Edit $$agent/.env.paid with your API keys"; \
	done

# ── Common commands ────────────────────────────────────────────
stop:
	docker-compose -f docker-compose.free.yml down 2>/dev/null || true
	docker-compose -f docker-compose.paid.yml down 2>/dev/null || true

logs:
	docker-compose -f docker-compose.free.yml logs -f

clean:
	docker-compose -f docker-compose.free.yml down -v
	docker-compose -f docker-compose.paid.yml down -v
	@echo "✅ All containers and volumes removed"

status:
	@curl -s http://localhost:8000/agents/status | python3 -m json.tool
```

### Command Reference

| Command | Description |
|---------|-------------|
| `make free` | Start all agents in free/local mode (Ollama + local files) |
| `make paid` | Start all agents in paid/cloud mode (API keys required) |
| `make pull-model` | Pull llama3.1:8b into Ollama (run once after `make free`) |
| `make setup-free` | Create `./data` folder |
| `make setup-paid` | Copy `.env.paid` templates to each agent folder |
| `make stop` | Stop all containers (both modes) |
| `make logs` | Tail logs from free mode containers |
| `make clean` | Remove all containers + volumes (full reset) |
| `make status` | Call `/agents/status` on Orchestrator |

---

## 16. Context Routing Matrix

How data flows through the system — what each agent receives and returns:

| Agent | Context Receives | Data Access | Returns |
|-------|-----------------|-------------|---------|
| Context Agent | Raw file path / DB connection string | Full scan (sampled 2–5%) | Schema JSON + Stats |
| Orchestrator | Schema JSON + user query | None (read-only context) | Task graph + routing plan |
| SQL Agent | Schema JSON + task + dialect | DB via MCP only | Query results summary |
| Viz Agent | Aggregated dataframe (≤1000 rows) | None after aggregation | Chart config JSON |
| ML Agent | Feature context + stats | Encoded arrays via MCP | Model metrics + SHAP |
| NLP Agent | Text column sample + stats | Chunked text via MCP | Entities + sentiments |
| Report Agent | All agent outputs (structured) | None | Final narrative doc |

---

## 17. Tech Stack Summary

| Layer | Technology | Notes |
|-------|-----------|-------|
| **Agent Framework** | FastAPI + Uvicorn | Async, production-ready, SSE support |
| **LLM — Free** | Ollama (llama3.1:8b) | Local, zero cost, Docker container |
| **LLM — Paid** | Claude / Groq / OpenAI | Switchable via `LLM_PROVIDER` env var |
| **Data Engine** | Polars (primary) + DuckDB | Fast columnar processing, in-process SQL |
| **ML — Free** | scikit-learn (FAST_MODE=true) | Trains in seconds |
| **ML — Paid** | AutoGluon | Full AutoML, trains dozens of models |
| **NLP Models** | Transformers + spaCy + BERTopic | HuggingFace ecosystem |
| **Embeddings** | SentenceTransformers | all-MiniLM-L6-v2 (free) / all-mpnet-base-v2 (paid) |
| **PDF Export** | WeasyPrint + Jinja2 | HTML → PDF via Pango/Cairo |
| **DOCX Export** | python-docx | Tables, images, headings |
| **PPTX Export** | python-pptx | 16:9 slides, chart embedding |
| **Cache + Queue** | Redis (redis-py async) | Context TTL, Celery broker |
| **Async Tasks** | Celery | Long-running ML training jobs |
| **SQL Transpiler** | SQLGlot | Multi-dialect support |
| **Chart Renderer** | Plotly + Kaleido | Spec JSON + PNG/SVG render |
| **Storage — Free** | Local Docker volumes | `./data`, charts, models, reports |
| **Storage — Paid** | AWS S3 | Presigned URLs, CloudFront CDN |
| **DB — Free** | DuckDB | Reads Parquet/CSV directly, no server |
| **DB — Paid** | PostgreSQL / BigQuery / Snowflake | Full cloud DB support |
| **Observability** | LangSmith (optional) | Agent trace logging, paid mode |
| **Deployment — Docker** | Docker Compose | Both modes, same codebase |
| **Deployment — EC2** | t3.small → t3.xlarge | Per-agent sizing, see EC2 notes |
| **Networking** | Docker bridge `gemrslize-net` | All agents on internal network |

---

## Quick Reference: Agent Port Map

| Agent | Port | Free Instance | Paid EC2 | Key Flag |
|-------|------|---------------|----------|----------|
| Orchestrator | 8000 | N/A (local) | t3.large | Only public port |
| Context Intelligence | 8001 | N/A (local) | t3.medium | Entry point for all data |
| SQL Query | 8002 | N/A (local) | t3.small | `DB_DIALECT=duckdb` (free) |
| Visualization | 8003 | N/A (local) | t3.medium | `STORAGE_TYPE=local/s3` |
| ML / Prediction | 8004 | N/A (local) | t3.xlarge | `FAST_MODE=true/false` |
| NLP / Text | 8005 | N/A (local) | t3.large | `HF_CACHE_DIR` volume |
| Report Synthesis | 8006 | N/A (local) | t3.medium | `USE_S3_STORAGE=true/false` |
| Redis | 6379 | Docker | ElastiCache | Shared by all agents |
| Ollama | 11434 | Docker | N/A | Free mode only |

---

