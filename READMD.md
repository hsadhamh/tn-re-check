# TamilNadu Land Document Pre-Validation Pipeline

> AI-powered pre-validation for Tamil Nadu land and property documents — detects missing fields, format violations, and compliance issues before formal submission to registration authorities.

[![CI](https://img.shields.io/github/actions/workflow/status/your-org/tn-land-validator/ci.yml?label=CI&style=flat-square)](https://github.com/your-org/tn-land-validator/actions)
[![Coverage](https://img.shields.io/codecov/c/github/your-org/tn-land-validator?style=flat-square)](https://codecov.io/gh/your-org/tn-land-validator)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)

---

## Table of contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Technology stack](#technology-stack)
4. [Project structure](#project-structure)
5. [Validation pipeline](#validation-pipeline)
6. [Data model](#data-model)
7. [Local development](#local-development)
8. [Configuration reference](#configuration-reference)
9. [API reference](#api-reference)
10. [Observability](#observability)
11. [CI/CD pipeline](#cicd-pipeline)
12. [Azure deployment](#azure-deployment)
13. [Runbooks](#runbooks)
14. [Roadmap](#roadmap)

---

## Overview

The TN Land Document Pre-Validation Pipeline ingests scanned PDF uploads from a web portal and runs a multi-stage AI validation workflow before a document reaches the Tamil Nadu Registration Department's formal systems.

**What it catches (Phase 1):**
- Missing required fields (applicant name, survey number, taluk, district, village, extent, document type)
- Format violations (date patterns, survey number formats, extent units, PIN code ranges)
- Logical inconsistencies (date of sale in the future, extent = 0, registration date before execution date)
- Administrative hierarchy mismatches (district + taluk + village combinations validated against reference data)

**What it will catch (Phase 2 — wired, not yet active):**
- Semantic anomalies via Pinecone vector similarity against known-good document templates
- Legal clause pattern matching via LangChain + OpenAI embeddings (embeddings only — generation via OpenRouter)
- Cross-document inconsistencies (Patta number referenced in Sale Deed does not exist in records)

**Output:** A structured JSON validation report with per-field status (`PASS` / `FAIL` / `MISSING`), severity (`CRITICAL` / `WARNING`), and human-readable explanations in both English and Tamil, surfaced via a React web portal.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Web portal (React · Azure Static Web Apps)                         │
│  PDF upload → real-time status polling → validation report view     │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS
┌────────────────────────────▼────────────────────────────────────────┐
│  AKS Cluster — South India region                                   │
│                                                                     │
│  ┌─────────────────┐   enqueue   ┌──────────────────────────────┐  │
│  │  FastAPI         ├────────────►  Celery workers               │  │
│  │  /upload         │            │  LangGraph stateful graph     │  │
│  │  /status/{id}    │            │                               │  │
│  │  /report/{id}    │            │  Node 1: OCR extraction       │  │
│  └────────┬─────────┘            │  Node 2: DuckDB field checks  │  │
│           │                      │  Node 3: Cross-field checks   │  │
│           │                      │  Node 4: Semantic (Phase 2)   │  │
│           │                      │  Node 5: Feedback generation  │  │
│           │                      └───────────────┬──────────────┘  │
│           │                                       │                 │
│  ┌────────▼───────────────────────────────────────▼──────────────┐ │
│  │  Data layer                                                    │ │
│  │  PostgreSQL (Azure Flexible)  Redis (Azure Cache)  Blob Store  │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
  Azure Doc Intelligence   OpenRouter          Pinecone
  (OCR + field extract)    (Gemma 3 27B        (semantic index, Ph.2)
                            feedback gen)
```

### Key design decisions

| Decision | Rationale |
|---|---|
| LangGraph for orchestration | Stateful, resumable graph execution. If a node crashes mid-run, Redis checkpoint allows resume from the last completed node rather than restarting the entire pipeline. |
| Celery + Redis task queue | LangGraph `invoke` is synchronous. OCR alone takes 8–15 seconds per document. Celery decouples the HTTP response (200ms) from the pipeline execution. |
| DuckDB in-pod for Phase 1 checks | Field presence and format checks are analytical queries over a small JSON blob. In-process DuckDB runs these 10× faster than round-tripping to PostgreSQL. PostgreSQL only receives the final committed result. |
| PostgreSQL over SQL Server | JSONB columns store semi-structured extraction results without schema migrations per document type. Row-level security enables per-office data isolation. Full SQLAlchemy + Alembic support. |
| Azure Document Intelligence | Best-in-class OCR for mixed Tamil/English handwritten + printed government forms. The prebuilt General Document model handles both without custom training for Phase 1. |
| OpenRouter as LLM gateway | Single OpenAI-compatible endpoint giving access to Gemma 3 27B, Llama 3.3 70B, and others. No GPU infrastructure to manage. Model is swappable via an env var — no code change required. Built-in fallback chain retries with a stronger model if JSON parsing fails. |
| `response_format: json_object` for feedback | Enforces structured JSON output from the feedback node across all OpenRouter-served models. Eliminates regex parsing and prevents hallucinated rule names. |
| OpenAI embeddings for Phase 2 | OpenRouter does not serve embedding models. OpenAI `text-embedding-3-small` is kept solely for Pinecone indexing at ~$0.02/million tokens — negligible cost compared to generation. |

---

## Technology stack

| Layer | Technology | Version |
|---|---|---|
| API | FastAPI | 0.111+ |
| Orchestration | LangGraph | 0.2+ |
| Task queue | Celery + Redis | 5.4+ |
| AI/LLM gateway | OpenRouter (Gemma 3 27B · Llama 3.3 70B fallback) | Latest |
| Embeddings | OpenAI `text-embedding-3-small` (Phase 2 only) | Latest |
| Vector store | Pinecone | v3 SDK |
| OCR | Azure Document Intelligence | 2024-02-29-preview |
| In-pipeline analytics | DuckDB | 0.10+ |
| Primary store | PostgreSQL 16 (Azure Flexible Server) | 16 |
| Session/cache | Redis 7 (Azure Cache for Redis) | 7 |
| Blob storage | Azure Blob Storage | — |
| Container runtime | Docker + AKS | Kubernetes 1.29+ |
| IaC | Terraform | 1.8+ |
| CI/CD | GitHub Actions | — |
| Observability | Azure Monitor + Application Insights + Prometheus + Grafana | — |
| Web portal | React + Azure Static Web Apps | React 18 |

---

## Project structure

```
tn-land-validator/
│
├── api/                            # FastAPI application
│   ├── main.py                     # App factory, middleware, router registration
│   ├── routers/
│   │   ├── upload.py               # POST /v1/documents/upload
│   │   ├── status.py               # GET  /v1/documents/{doc_id}/status
│   │   └── report.py               # GET  /v1/documents/{doc_id}/report
│   ├── dependencies.py             # DB sessions, Redis client, blob client
│   └── middleware/
│       ├── auth.py                 # Azure AD JWT validation
│       └── request_id.py           # X-Request-ID propagation
│
├── pipeline/
│   ├── graph.py                    # LangGraph StateGraph definition
│   ├── state.py                    # DocumentState TypedDict
│   ├── nodes/
│   │   ├── extractor.py            # Node 1: Azure Document Intelligence
│   │   ├── field_checker.py        # Node 2: DuckDB presence + format + logic checks
│   │   ├── cross_field.py          # Node 3: District/taluk hierarchy + date logic
│   │   ├── semantic.py             # Node 4: Pinecone similarity (Phase 2 stub)
│   │   └── feedback.py             # Node 5: OpenRouter feedback generation (Gemma 3 27B)
│   └── rules/
│       ├── required_fields.py      # Field manifests per document type
│       ├── format_rules.py         # Regex + type validators
│       └── reference_data.py       # TN administrative hierarchy loader
│
├── workers/
│   ├── celery_app.py               # Celery app + task definitions
│   └── tasks.py                    # run_validation_pipeline task
│
├── models/                         # SQLAlchemy ORM models
│   ├── document.py
│   └── validation_run.py
│
├── schemas/                        # Pydantic request/response schemas
│   ├── upload.py
│   ├── status.py
│   └── report.py
│
├── db/
│   ├── session.py                  # Async SQLAlchemy engine + session factory
│   └── migrations/                 # Alembic migration scripts
│       └── versions/
│
├── infra/
│   ├── terraform/
│   │   ├── main.tf                 # AKS, PostgreSQL, Redis, Blob, Key Vault
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── helm/
│       ├── api/                    # Helm chart: FastAPI deployment
│       └── worker/                 # Helm chart: Celery worker deployment
│
├── portal/                         # React web portal
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Upload.tsx
│   │   │   ├── Status.tsx
│   │   │   └── Report.tsx
│   │   └── components/
│   └── package.json
│
├── tests/
│   ├── fixtures/
│   │   ├── patta_valid.pdf         # Known-good Patta document
│   │   ├── patta_missing_fields.pdf
│   │   └── sale_deed_valid.pdf
│   ├── unit/
│   │   ├── test_field_checker.py
│   │   ├── test_cross_field.py
│   │   └── test_feedback.py
│   └── integration/
│       ├── test_pipeline_e2e.py
│       └── test_api_upload.py
│
├── .github/
│   └── workflows/
│       ├── ci.yml                  # PR checks: lint, test, build
│       └── cd.yml                  # Main branch: build, push, deploy to AKS
│
├── docker-compose.yml              # Local dev stack
├── Dockerfile.api
├── Dockerfile.worker
├── pyproject.toml
└── README.md
```

---

## Validation pipeline

### DocumentState

The LangGraph state object is the single source of truth throughout the pipeline run.

```python
# pipeline/state.py
from typing import TypedDict, Optional
from enum import Enum

class Severity(str, Enum):
    CRITICAL = "CRITICAL"
    WARNING  = "WARNING"
    INFO     = "INFO"

class FieldStatus(str, Enum):
    PASS    = "PASS"
    FAIL    = "FAIL"
    MISSING = "MISSING"

class CheckResult(TypedDict):
    field:       str
    status:      FieldStatus
    severity:    Severity
    rule:        str           # e.g. "survey_no_format"
    message_en:  str
    message_ta:  str
    value:       Optional[str] # actual extracted value, if any

class DocumentState(TypedDict):
    doc_id:           str
    doc_type:         str           # "patta" | "sale_deed" | "ec"
    blob_url:         str
    fields_extracted: dict          # raw key-value pairs from Doc Intelligence
    check_results:    list[CheckResult]
    critical_count:   int
    warning_count:    int
    current_node:     str
    early_exit:       bool
    feedback_report:  Optional[dict]
    error:            Optional[str]
```

### Graph definition

```python
# pipeline/graph.py
from langgraph.graph import StateGraph, END
from pipeline.state import DocumentState
from pipeline.nodes import extractor, field_checker, cross_field, semantic, feedback

def should_exit_early(state: DocumentState) -> str:
    """Route to early exit if critical failures found in field checks."""
    if state["early_exit"]:
        return "feedback"   # skip cross-field and semantic, go straight to report
    return "cross_field"

graph = StateGraph(DocumentState)

graph.add_node("extractor",    extractor.run)
graph.add_node("field_checker", field_checker.run)
graph.add_node("cross_field",  cross_field.run)
graph.add_node("semantic",     semantic.run)   # Phase 2 stub — returns state unchanged
graph.add_node("feedback",     feedback.run)

graph.set_entry_point("extractor")
graph.add_edge("extractor",    "field_checker")
graph.add_conditional_edges("field_checker", should_exit_early)
graph.add_edge("cross_field",  "semantic")
graph.add_edge("semantic",     "feedback")
graph.add_edge("feedback",     END)

pipeline = graph.compile(checkpointer=redis_checkpointer)
```

### Node: field_checker (Phase 1 core)

```python
# pipeline/nodes/field_checker.py
import duckdb
import json
from pipeline.state import DocumentState, CheckResult, FieldStatus, Severity
from pipeline.rules.required_fields import REQUIRED_FIELDS
from pipeline.rules.format_rules import FORMAT_RULES

def run(state: DocumentState) -> DocumentState:
    fields   = state["fields_extracted"]
    doc_type = state["doc_type"]
    results: list[CheckResult] = []

    conn = duckdb.connect(":memory:")
    conn.execute("CREATE TABLE fields AS SELECT * FROM (VALUES (?)) t(data)", [json.dumps(fields)])

    # 1. Presence checks
    for field_name in REQUIRED_FIELDS[doc_type]:
        value = fields.get(field_name)
        if not value or str(value).strip() == "":
            results.append(CheckResult(
                field=field_name,
                status=FieldStatus.MISSING,
                severity=Severity.CRITICAL,
                rule="required_field_presence",
                message_en=f"Required field '{field_name}' is missing.",
                message_ta=f"தேவையான புலம் '{field_name}' காணப்படவில்லை.",
                value=None,
            ))
            continue

        # 2. Format checks
        if field_name in FORMAT_RULES:
            rule = FORMAT_RULES[field_name]
            if not rule.validate(value):
                results.append(CheckResult(
                    field=field_name,
                    status=FieldStatus.FAIL,
                    severity=rule.severity,
                    rule=rule.name,
                    message_en=rule.message_en(value),
                    message_ta=rule.message_ta(value),
                    value=str(value),
                ))
            else:
                results.append(CheckResult(
                    field=field_name,
                    status=FieldStatus.PASS,
                    severity=Severity.INFO,
                    rule=rule.name,
                    message_en="",
                    message_ta="",
                    value=str(value),
                ))

    conn.close()

    critical_count = sum(1 for r in results if r["severity"] == Severity.CRITICAL)
    return {
        **state,
        "check_results":  results,
        "critical_count": critical_count,
        "current_node":   "field_checker",
        "early_exit":     critical_count >= 3,   # configurable threshold
    }
```

### Phase 1 format rules (sample)

```python
# pipeline/rules/format_rules.py
import re
from dataclasses import dataclass
from pipeline.state import Severity

@dataclass
class FormatRule:
    name:       str
    severity:   Severity
    pattern:    re.Pattern
    message_en: callable
    message_ta: callable

    def validate(self, value: str) -> bool:
        return bool(self.pattern.fullmatch(str(value).strip()))

FORMAT_RULES = {
    "survey_no": FormatRule(
        name      = "survey_no_format",
        severity  = Severity.CRITICAL,
        pattern   = re.compile(r"\d{1,5}/\d{1,3}[A-Z]?"),
        message_en= lambda v: f"Survey number '{v}' must match pattern NNN/N (e.g. 142/3A).",
        message_ta= lambda v: f"கணக்கெடுப்பு எண் '{v}' NNN/N வடிவத்தில் இருக்க வேண்டும்.",
    ),
    "execution_date": FormatRule(
        name      = "date_format_dmy",
        severity  = Severity.CRITICAL,
        pattern   = re.compile(r"(0[1-9]|[12]\d|3[01])/(0[1-9]|1[0-2])/\d{4}"),
        message_en= lambda v: f"Date '{v}' must be in DD/MM/YYYY format.",
        message_ta= lambda v: f"தேதி '{v}' DD/MM/YYYY வடிவத்தில் இருக்க வேண்டும்.",
    ),
    "pin_code": FormatRule(
        name      = "tn_pin_code",
        severity  = Severity.WARNING,
        pattern   = re.compile(r"6[0-9]{5}"),
        message_en= lambda v: f"PIN code '{v}' is not a valid Tamil Nadu PIN (must start with 6).",
        message_ta= lambda v: f"பின் குறியீடு '{v}' தமிழ்நாட்டுக்கு சரியானதாக இல்லை.",
    ),
    "extent_value": FormatRule(
        name      = "extent_positive",
        severity  = Severity.CRITICAL,
        pattern   = re.compile(r"[1-9]\d*(\.\d+)?"),
        message_en= lambda v: f"Extent value '{v}' must be a positive number.",
        message_ta= lambda v: f"பரப்பளவு '{v}' நேர்மறை எண்ணாக இருக்க வேண்டும்.",
    ),
}
```

### Node: feedback (OpenRouter · Gemma 3 27B)

The feedback node is the only node that calls an external LLM. It uses OpenRouter's OpenAI-compatible API with a two-model fallback chain — Gemma 3 27B first, Llama 3.3 70B if JSON parsing fails. The system prompt seeds known-correct Tamil phrases to prevent colloquial or incorrect phrasing in legal contexts.

```python
# pipeline/nodes/feedback.py
from langchain_openai import ChatOpenAI
from pipeline.state import DocumentState
from pipeline.config import settings
import json, structlog
from prometheus_client import Counter

log = structlog.get_logger()

llm_model_used = Counter(
    "feedback_model_used_total",
    "Feedback generation calls by model and outcome",
    ["model", "outcome"],
)

# Ordered fallback chain — first model is tried first; next is used if JSON parse fails
MODELS = [
    "google/gemma-3-27b-it",               # fast, cheap, good Tamil coverage
    "meta-llama/llama-3.3-70b-instruct",   # stronger JSON reliability as fallback
]

SYSTEM_PROMPT = """
You are a document validation assistant for Tamil Nadu land registration offices.
Respond ONLY with valid JSON. No explanation, no markdown, no preamble.

Use these verified Tamil phrases for common errors:
- Missing field:        "தேவையான புலம் காணப்படவில்லை"
- Invalid format:       "தவறான வடிவம் உள்ளது"
- Date error:           "தேதி தவறானது"
- Survey number error:  "கணக்கெடுப்பு எண் தவறானது"
- Hierarchy mismatch:   "மாவட்டம்/தாலுகா பொருந்தவில்லை"

JSON schema (strict — output nothing else):
{
  "overall_status": "PASS" | "FAIL",
  "summary_en": "<one sentence in English>",
  "summary_ta": "<one sentence in Tamil script>",
  "fields": {
    "<field_name>": {
      "status": "PASS" | "FAIL" | "MISSING",
      "severity": "CRITICAL" | "WARNING" | "INFO",
      "message_en": "<explanation in English>",
      "message_ta": "<explanation in Tamil script>"
    }
  }
}
"""

def _build_llm(model: str) -> ChatOpenAI:
    return ChatOpenAI(
        model=model,
        base_url=settings.OPENROUTER_BASE_URL,
        api_key=settings.OPENROUTER_API_KEY,
        model_kwargs={
            "response_format": {"type": "json_object"},
            "extra_headers": {
                "HTTP-Referer": "https://tn-land-validator.yourdomain.in",
                "X-Title": "TN Land Validator",
            },
        },
        temperature=0,
        max_tokens=1024,
    )

def run(state: DocumentState) -> DocumentState:
    violations = [r for r in state["check_results"] if r["status"] != "PASS"]

    user_msg = f"""
Document type: {state["doc_type"]}
Extracted fields: {json.dumps(state["fields_extracted"], ensure_ascii=False)}
Validation violations: {json.dumps(violations, ensure_ascii=False)}

Generate the validation feedback report following the JSON schema exactly.
"""
    for model in MODELS:
        try:
            llm      = _build_llm(model)
            response = llm.invoke([
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user",   "content": user_msg},
            ])
            report = json.loads(response.content)
            llm_model_used.labels(model=model, outcome="success").inc()
            log.info("feedback.generated", doc_id=state["doc_id"],
                     model=model, overall=report.get("overall_status"))
            return {**state, "feedback_report": report, "feedback_model": model}
        except json.JSONDecodeError:
            llm_model_used.labels(model=model, outcome="json_parse_failed").inc()
            log.warning("feedback.json_parse_failed", doc_id=state["doc_id"],
                        model=model, raw=response.content[:200])
            continue
        except Exception as exc:
            llm_model_used.labels(model=model, outcome="error").inc()
            log.error("feedback.error", doc_id=state["doc_id"],
                      model=model, error=str(exc))
            continue

    return {**state, "error": "all_feedback_models_failed"}
```

### Celery task
from workers.celery_app import celery_app
from pipeline.graph import pipeline
from pipeline.state import DocumentState
from db.session import get_session
from models.validation_run import ValidationRun
import structlog

log = structlog.get_logger()

@celery_app.task(bind=True, max_retries=3, default_retry_delay=10)
def run_validation_pipeline(self, doc_id: str, doc_type: str, blob_url: str):
    log.info("pipeline.start", doc_id=doc_id, doc_type=doc_type)
    try:
        initial_state = DocumentState(
            doc_id=doc_id,
            doc_type=doc_type,
            blob_url=blob_url,
            fields_extracted={},
            check_results=[],
            critical_count=0,
            warning_count=0,
            current_node="",
            early_exit=False,
            feedback_report=None,
            error=None,
        )
        final_state = pipeline.invoke(
            initial_state,
            config={"configurable": {"thread_id": doc_id}},
        )
        _persist_result(doc_id, final_state)
        log.info("pipeline.complete", doc_id=doc_id,
                 critical=final_state["critical_count"],
                 warnings=final_state["warning_count"])
    except Exception as exc:
        log.error("pipeline.error", doc_id=doc_id, error=str(exc))
        raise self.retry(exc=exc)

def _persist_result(doc_id: str, state: DocumentState):
    with get_session() as session:
        run = session.query(ValidationRun).filter_by(doc_id=doc_id).first()
        run.fields_extracted = state["fields_extracted"]
        run.check_results    = state["check_results"]
        run.critical_count   = state["critical_count"]
        run.warning_count    = state["warning_count"]
        run.overall_status   = "FAIL" if state["critical_count"] > 0 else "PASS"
        run.completed_at     = "now()"
        session.commit()
```

---

## Data model

```sql
-- documents: one row per uploaded file
CREATE TABLE documents (
    doc_id       UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    uploaded_at  TIMESTAMPTZ NOT NULL    DEFAULT now(),
    uploaded_by  TEXT,                   -- Azure AD object ID
    doc_type     TEXT        NOT NULL,   -- 'patta' | 'sale_deed' | 'ec'
    blob_url     TEXT        NOT NULL,
    status       TEXT        NOT NULL    DEFAULT 'queued',
                                         -- queued | processing | done | failed
    CONSTRAINT   chk_doc_type CHECK (doc_type IN ('patta','sale_deed','ec'))
);

-- validation_runs: one row per pipeline execution
CREATE TABLE validation_runs (
    run_id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    doc_id           UUID        NOT NULL    REFERENCES documents(doc_id),
    started_at       TIMESTAMPTZ NOT NULL    DEFAULT now(),
    completed_at     TIMESTAMPTZ,
    phase            SMALLINT    NOT NULL    DEFAULT 1,
    fields_extracted JSONB,
    check_results    JSONB,                  -- CheckResult[] array
    critical_count   SMALLINT    DEFAULT 0,
    warning_count    SMALLINT    DEFAULT 0,
    overall_status   TEXT,                   -- PASS | FAIL | PARTIAL
    celery_task_id   TEXT
);

CREATE INDEX idx_runs_doc_id   ON validation_runs(doc_id);
CREATE INDEX idx_runs_status   ON validation_runs(overall_status);
CREATE INDEX idx_fields_gin    ON validation_runs USING GIN (fields_extracted);
CREATE INDEX idx_results_gin   ON validation_runs USING GIN (check_results);

-- reference: TN administrative hierarchy (pre-loaded)
CREATE TABLE tn_hierarchy (
    district TEXT NOT NULL,
    taluk    TEXT NOT NULL,
    village  TEXT NOT NULL,
    PRIMARY KEY (district, taluk, village)
);
```

---

## Local development

### Prerequisites

- Docker Desktop
- Python 3.11+
- Node 20+ (for portal)
- `uv` (recommended) or `pip`
- Azure CLI (for Azure service connections in integration tests)

### Start the full local stack

```bash
# Clone
git clone https://github.com/your-org/tn-land-validator.git
cd tn-land-validator

# Copy and populate environment
cp .env.example .env
# Edit .env — see Configuration reference below

# Start PostgreSQL, Redis, and Celery worker
docker-compose up -d

# Install Python dependencies
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"

# Run Alembic migrations
alembic upgrade head

# Seed TN hierarchy reference data
python scripts/seed_hierarchy.py

# Start FastAPI (hot-reload)
uvicorn api.main:app --reload --port 8000

# In a second terminal — start Celery worker
celery -A workers.celery_app worker --loglevel=info --concurrency=4
```

API is available at `http://localhost:8000`. Swagger UI at `http://localhost:8000/docs`.

### docker-compose.yml (local services only)

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       tn_validator
      POSTGRES_USER:     validator
      POSTGRES_PASSWORD: localdev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    environment:
      DATABASE_URL:  postgresql+asyncpg://validator:localdev@postgres/tn_validator
      REDIS_URL:     redis://redis:6379/0
      CELERY_BROKER: redis://redis:6379/1
    depends_on: [postgres, redis]
    volumes: [".:/app"]

volumes:
  pgdata:
```

### Run tests

```bash
# Unit tests only (no external services needed)
pytest tests/unit -v

# Integration tests (requires running docker-compose stack + Azure credentials)
pytest tests/integration -v --timeout=60

# Full suite with coverage
pytest --cov=api --cov=pipeline --cov-report=term-missing
```

---

## Configuration reference

All configuration is injected via environment variables. For local dev, use `.env`. In AKS, these are mounted from Azure Key Vault via the Secrets Store CSI driver.

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL async DSN — `postgresql+asyncpg://user:pass@host/db` |
| `REDIS_URL` | Yes | Redis connection string — `redis://:pass@host:6379/0` |
| `CELERY_BROKER_URL` | Yes | Celery broker — `redis://:pass@host:6379/1` |
| `CELERY_RESULT_BACKEND` | Yes | Celery result backend — `redis://:pass@host:6379/2` |
| `AZURE_BLOB_CONNECTION_STRING` | Yes | Azure Blob Storage connection string |
| `AZURE_BLOB_CONTAINER` | Yes | Container name for PDF uploads |
| `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT` | Yes | Azure Doc Intelligence endpoint URL |
| `AZURE_DOCUMENT_INTELLIGENCE_KEY` | Yes | Azure Doc Intelligence API key (use Key Vault ref in prod) |
| `OPENROUTER_API_KEY` | Yes | OpenRouter API key — `sk-or-...` (use Key Vault ref in prod) |
| `OPENROUTER_BASE_URL` | No | OpenRouter base URL — default `https://openrouter.ai/api/v1` |
| `OPENROUTER_PRIMARY_MODEL` | No | Primary feedback model — default `google/gemma-3-27b-it` |
| `OPENROUTER_FALLBACK_MODEL` | No | Fallback feedback model — default `meta-llama/llama-3.3-70b-instruct` |
| `OPENAI_API_KEY` | No | OpenAI API key — used only for Phase 2 embeddings (`text-embedding-3-small`) |
| `PINECONE_API_KEY` | No | Pinecone API key (Phase 2) |
| `PINECONE_INDEX_NAME` | No | Pinecone index name (Phase 2) |
| `EARLY_EXIT_THRESHOLD` | No | Min critical failures before early exit — default `3` |
| `PIPELINE_TIMEOUT_SECONDS` | No | Max pipeline run duration — default `120` |
| `LOG_LEVEL` | No | `DEBUG` / `INFO` / `WARNING` — default `INFO` |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | No | Azure App Insights connection string |

---

## API reference

### POST `/v1/documents/upload`

Upload a PDF for validation.

**Request:** `multipart/form-data`

| Field | Type | Description |
|---|---|---|
| `file` | `File` | PDF document (max 10 MB) |
| `doc_type` | `string` | `patta` \| `sale_deed` \| `ec` |

**Response `202 Accepted`:**
```json
{
  "doc_id": "3f7a1d2e-...",
  "status": "queued",
  "estimated_seconds": 30
}
```

---

### GET `/v1/documents/{doc_id}/status`

Poll pipeline status.

**Response `200 OK`:**
```json
{
  "doc_id": "3f7a1d2e-...",
  "status": "processing",
  "current_node": "field_checker",
  "elapsed_seconds": 12
}
```

`status` values: `queued` → `processing` → `done` | `failed`

---

### GET `/v1/documents/{doc_id}/report`

Retrieve the full validation report.

**Response `200 OK`:**
```json
{
  "doc_id": "3f7a1d2e-...",
  "doc_type": "patta",
  "overall_status": "FAIL",
  "critical_count": 2,
  "warning_count": 1,
  "completed_at": "2025-01-15T10:32:44Z",
  "fields": {
    "applicant_name":  { "status": "PASS",    "value": "Rajan Murugesan" },
    "survey_no":       { "status": "FAIL",    "value": "14B",   "severity": "CRITICAL",
                         "message_en": "Survey number '14B' must match pattern NNN/N.",
                         "message_ta": "கணக்கெடுப்பு எண் '14B' NNN/N வடிவத்தில் இருக்க வேண்டும்." },
    "taluk":           { "status": "MISSING", "value": null,    "severity": "CRITICAL",
                         "message_en": "Required field 'taluk' is missing.",
                         "message_ta": "தேவையான புலம் 'taluk' காணப்படவில்லை." }
  },
  "summary_en": "2 critical issues found. Document cannot be submitted until resolved.",
  "summary_ta": "2 முக்கிய சிக்கல்கள் கண்டறியப்பட்டன. தீர்க்கப்படும் வரை ஆவணத்தை சமர்ப்பிக்க முடியாது."
}
```

---

## Observability

The pipeline is instrumented at three levels: structured logs, distributed traces, and metrics. All three converge in Azure Monitor.

### Structured logging (structlog)

Every log event is emitted as JSON with consistent fields for log analytics queries.

```python
# api/main.py
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)
```

Log fields emitted at each pipeline stage:

| Event key | Fields |
|---|---|
| `pipeline.start` | `doc_id`, `doc_type`, `celery_task_id` |
| `node.start` | `doc_id`, `node_name` |
| `node.complete` | `doc_id`, `node_name`, `duration_ms` |
| `field_check.result` | `doc_id`, `field`, `status`, `severity` |
| `pipeline.complete` | `doc_id`, `critical_count`, `warning_count`, `overall_status`, `total_duration_ms` |
| `pipeline.error` | `doc_id`, `node_name`, `error`, `traceback` |

### Distributed tracing (OpenTelemetry)

```python
# api/main.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from azure.monitor.opentelemetry.exporter import AzureMonitorTraceExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(
        AzureMonitorTraceExporter(
            connection_string=settings.APPLICATIONINSIGHTS_CONNECTION_STRING
        )
    )
)
trace.set_tracer_provider(provider)
```

Each LangGraph node wraps its execution in an OpenTelemetry span. The entire pipeline run appears as a single trace in Application Insights with child spans per node — making it easy to identify which node is slow.

### Prometheus metrics

A `/metrics` endpoint (Prometheus format) is exposed on port `9090` and scraped by AKS-managed Prometheus.

Key metrics:

| Metric | Type | Description |
|---|---|---|
| `pipeline_runs_total` | Counter | Total pipeline runs, labelled by `doc_type` and `status` |
| `pipeline_duration_seconds` | Histogram | End-to-end pipeline duration |
| `node_duration_seconds` | Histogram | Per-node duration, labelled by `node_name` |
| `ocr_duration_seconds` | Histogram | Azure Doc Intelligence call latency |
| `llm_tokens_used_total` | Counter | Total tokens consumed via OpenRouter, labelled by `model` |
| `feedback_model_used_total` | Counter | Feedback generation calls labelled by `model` and `outcome` (`success` / `json_parse_failed` / `error`) |
| `critical_failures_total` | Counter | Critical validation failures, labelled by `field` and `doc_type` |
| `queue_depth` | Gauge | Celery queue depth (Celery Exporter) |

### Grafana dashboards

Three dashboards are provisioned via Terraform (JSON files in `infra/grafana/`):

1. `Pipeline Overview` — run volume, error rate, p50/p95/p99 durations, queue depth
2. `Validation Quality` — top failing fields, critical vs warning breakdown by doc type, failure trends over time
3. `Infrastructure` — AKS pod CPU/memory, PostgreSQL connections and query latency, Redis memory and hit rate

### Azure Monitor alerts

| Alert | Condition | Severity |
|---|---|---|
| Pipeline error spike | Error rate > 5% over 5 min | Sev 1 |
| Stuck pipeline | Any run in `processing` state > 5 min | Sev 2 |
| Queue depth high | Celery queue depth > 50 for 2 min | Sev 2 |
| OCR latency degraded | p95 OCR duration > 30s | Sev 3 |
| OpenRouter spend | Daily spend > 80% of budget | Sev 3 |
| Fallback model elevated | `feedback_model_used_total{model="llama-3.3-70b"}` > 10% of calls over 30 min | Sev 3 |

---

## CI/CD pipeline

### Overview

```
Developer pushes branch
        │
        ▼
┌───────────────────────────────┐
│  CI (ci.yml) — runs on PR      │
│                               │
│  1. Lint (ruff + mypy)        │
│  2. Unit tests (pytest)       │
│  3. Docker build (no push)    │
│  4. Security scan (Trivy)     │
└───────────────────────────────┘
        │ PR approved + merged to main
        ▼
┌───────────────────────────────────────────────────────┐
│  CD (cd.yml) — runs on push to main                    │
│                                                       │
│  1. Build & push Docker images to ACR                 │
│  2. Run integration tests against staging AKS         │
│  3. Run Alembic migrations against staging DB         │
│  4. Helm upgrade (staging)                            │
│  5. Smoke test staging endpoint                       │
│  6. Manual approval gate ──────────────────────────┐  │
│  7. Helm upgrade (production)                      │  │
│  8. Post-deploy health check                       │  │
└────────────────────────────────────────────────────┘
```

### ci.yml

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB:       tn_validator_test
          POSTGRES_USER:     validator
          POSTGRES_PASSWORD: testpass
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip

      - name: Install dependencies
        run: pip install -e ".[dev]"

      - name: Lint
        run: |
          ruff check .
          mypy api pipeline workers --ignore-missing-imports

      - name: Run migrations
        env:
          DATABASE_URL: postgresql+asyncpg://validator:testpass@localhost/tn_validator_test
        run: alembic upgrade head

      - name: Unit tests
        env:
          DATABASE_URL: postgresql+asyncpg://validator:testpass@localhost/tn_validator_test
          REDIS_URL:    redis://localhost:6379/0
        run: pytest tests/unit -v --cov=api --cov=pipeline --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4

      - name: Build Docker image (validation only)
        run: docker build -f Dockerfile.api -t tn-validator-api:ci .

      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: tn-validator-api:ci
          severity:  CRITICAL,HIGH
          exit-code: 1
```

### cd.yml

```yaml
name: CD

on:
  push:
    branches: [main]

env:
  ACR_REGISTRY:   yourorg.azurecr.io
  AKS_CLUSTER:    tn-validator-aks
  RESOURCE_GROUP: tn-validator-rg

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Login to ACR
        run: az acr login --name yourorg

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.ACR_REGISTRY }}/tn-validator-api
          tags:   type=sha

      - name: Build and push API image
        uses: docker/build-push-action@v5
        with:
          context:    .
          file:       Dockerfile.api
          push:       true
          tags:       ${{ steps.meta.outputs.tags }}

      - name: Build and push Worker image
        uses: docker/build-push-action@v5
        with:
          context:    .
          file:       Dockerfile.worker
          push:       true
          tags:       ${{ env.ACR_REGISTRY }}/tn-validator-worker:${{ github.sha }}

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get AKS credentials
        run: az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER-staging

      - name: Run DB migrations (staging)
        run: |
          kubectl run alembic-migrate \
            --image=${{ env.ACR_REGISTRY }}/tn-validator-api:${{ github.sha }} \
            --restart=Never \
            --env="DATABASE_URL=${{ secrets.STAGING_DATABASE_URL }}" \
            --command -- alembic upgrade head
          kubectl wait --for=condition=complete job/alembic-migrate --timeout=60s

      - name: Helm upgrade (staging)
        run: |
          helm upgrade --install tn-validator-api infra/helm/api \
            --namespace tn-validator \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --values infra/helm/api/values.staging.yaml \
            --wait --timeout 5m

      - name: Smoke test
        run: |
          STAGING_URL=$(kubectl get svc tn-validator-api -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
          curl -f "http://$STAGING_URL/health" || exit 1

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production   # requires manual approval in GitHub Environments
    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get AKS credentials (prod)
        run: az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER

      - name: Run DB migrations (production)
        run: |
          kubectl run alembic-migrate \
            --image=${{ env.ACR_REGISTRY }}/tn-validator-api:${{ github.sha }} \
            --restart=Never \
            --env="DATABASE_URL=${{ secrets.PROD_DATABASE_URL }}" \
            --command -- alembic upgrade head
          kubectl wait --for=condition=complete job/alembic-migrate --timeout=60s

      - name: Helm upgrade (production)
        run: |
          helm upgrade --install tn-validator-api infra/helm/api \
            --namespace tn-validator \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --values infra/helm/api/values.production.yaml \
            --wait --timeout 5m

      - name: Post-deploy health check
        run: |
          sleep 10
          PROD_URL=$(kubectl get svc tn-validator-api -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
          curl -f "http://$PROD_URL/health" || exit 1
```

---

## Azure deployment

### Provision infrastructure

```bash
cd infra/terraform

# Initialise
terraform init \
  -backend-config="resource_group_name=tn-validator-rg" \
  -backend-config="storage_account_name=tnvalidatortfstate" \
  -backend-config="container_name=tfstate"

# Plan
terraform plan -var-file=environments/production.tfvars -out=tfplan

# Apply
terraform apply tfplan
```

Terraform provisions: AKS cluster, Azure Container Registry, Azure Database for PostgreSQL Flexible Server, Azure Cache for Redis, Azure Blob Storage account, Azure Key Vault, Azure Document Intelligence resource, Application Insights, Log Analytics workspace, Managed Identity for pod workload identity.

### Secrets management

All secrets are stored in Azure Key Vault and injected into pods via the Secrets Store CSI driver. No secrets are stored in Kubernetes Secrets objects or environment variable literals in Helm values.

```yaml
# infra/helm/api/templates/secretproviderclass.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: tn-validator-secrets
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID:       "<managed-identity-client-id>"
    keyvaultName:   "tn-validator-kv"
    objects: |
      array:
        - |
          objectName: openrouter-api-key
          objectType: secret
        - |
          objectName: openai-api-key
          objectType: secret
        - |
          objectName: database-url
          objectType: secret
        - |
          objectName: azure-doc-intelligence-key
          objectType: secret
```

### Health checks

```
GET /health       → 200 OK { "status": "healthy", "db": "ok", "redis": "ok" }
GET /health/live  → 200 OK (liveness — process is running)
GET /health/ready → 200 OK (readiness — DB + Redis connections verified)
```

AKS liveness and readiness probes are configured in Helm values to use `/health/live` and `/health/ready` respectively.

---

## Runbooks

### RB-01: Pipeline stuck in `processing`

**Trigger:** Azure Monitor alert "Stuck pipeline" fires — a run has been in `processing` state for > 5 minutes.

**Steps:**
1. Identify the stuck `doc_id` via Log Analytics: `ValidationRuns | where status == 'processing' and started_at < ago(5m)`
2. Check the Celery worker logs for that task: `kubectl logs -l app=tn-validator-worker --since=10m | grep <doc_id>`
3. If the task is dead (no log entries for > 2 min), manually revoke and requeue:
   ```bash
   python scripts/ops/requeue.py --doc-id <doc_id>
   ```
4. If the task is alive but slow (OCR call stuck), the Azure Doc Intelligence endpoint may be degraded. Check https://azure.status.microsoft and increase `PIPELINE_TIMEOUT_SECONDS` temporarily.
5. Mark the run as `failed` in PostgreSQL if unrecoverable:
   ```sql
   UPDATE validation_runs SET overall_status = 'FAILED'
   WHERE doc_id = '<doc_id>' AND completed_at IS NULL;
   UPDATE documents SET status = 'failed' WHERE doc_id = '<doc_id>';
   ```

---

### RB-02: OpenRouter spend spike

**Trigger:** Azure Monitor alert "OpenRouter spend" fires — daily spend > 80% of budget before EOD, or the `feedback_model_used_total{outcome="json_parse_failed"}` rate rises above 10% (Gemma producing malformed JSON, falling back to Llama and doubling cost per document).

**Steps:**
1. Check Grafana `Pipeline Overview` → `feedback_model_used_total` to determine whether it is a volume spike or a fallback-chain issue.
2. If fallback rate is elevated, Gemma 3 27B may be receiving malformed extraction payloads (large garbage OCR text inflating the prompt). Check `fields_extracted` sizes in PostgreSQL:
   ```sql
   SELECT doc_id, pg_column_size(fields_extracted) AS size_bytes
   FROM validation_runs ORDER BY size_bytes DESC LIMIT 10;
   ```
3. If prompts are bloated, truncate the `fields_extracted` payload in `feedback.py` before sending (cap each field value at 500 chars).
4. To switch the primary model without a code deploy, update Key Vault and restart workers:
   ```bash
   az keyvault secret set --vault-name tn-validator-kv \
     --name openrouter-primary-model --value "google/gemma-3-12b-it"
   kubectl rollout restart deployment/tn-validator-worker
   ```
5. If budget is still breached, set `EARLY_EXIT_THRESHOLD=1` to skip feedback generation on severely broken documents (those with 1+ critical failures get a templated report instead of an LLM-generated one).

---

### RB-03: PostgreSQL connection exhaustion

**Trigger:** App Insights alert — DB connection errors appearing in logs; Grafana shows connection count near max.

**Steps:**
1. Check current connections: `SELECT count(*), state FROM pg_stat_activity GROUP BY state;`
2. Confirm PgBouncer is running in the AKS deployment (`kubectl get pods -l app=pgbouncer`).
3. If PgBouncer is down, restart it: `kubectl rollout restart deployment/pgbouncer`
4. If connection count is high despite PgBouncer, a Celery worker is likely leaking connections — identify the worker pod and restart: `kubectl delete pod -l app=tn-validator-worker`
5. Temporarily reduce Celery concurrency: `kubectl set env deployment/tn-validator-worker CELERY_CONCURRENCY=2`

---

## Roadmap

### Phase 1 (current)
- [x] PDF upload + Azure Blob ingestion
- [x] LangGraph pipeline skeleton
- [x] Azure Document Intelligence OCR extraction
- [x] DuckDB field presence + format + logic checks
- [x] OpenRouter feedback generation (Gemma 3 27B · EN + Tamil · fallback chain)
- [x] PostgreSQL persistence
- [x] FastAPI status + report endpoints
- [x] Celery task queue
- [x] AKS deployment + Terraform IaC
- [x] Prometheus metrics + Grafana dashboards
- [x] CI/CD with GitHub Actions

### Phase 2 — semantic validation
- [ ] Pinecone index seeded with 500+ known-good documents
- [ ] Semantic cross-referencing via LangChain + OpenAI embeddings (generation remains via OpenRouter)
- [ ] Legal clause pattern matching
- [ ] Cross-document consistency checks (Patta ↔ Sale Deed number verification)

### Phase 3 — integrations
- [ ] TNREGINET API integration for live Patta number validation
- [ ] EC (Encumbrance Certificate) cross-reference against registration records
- [ ] Webhook callbacks for third-party submission portals
- [ ] Bulk validation API (zip upload with multiple documents)

---

## Contributing

1. Fork the repository and create a feature branch: `git checkout -b feat/your-feature`
2. Write tests first. Unit tests must pass before opening a PR.
3. Ensure `ruff check .` and `mypy` pass locally.
4. Open a PR against `main`. CI runs automatically.
5. One approval required from a code owner before merge.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

*Maintained by the Upliftory Engineering team. For support, open a GitHub issue.*
