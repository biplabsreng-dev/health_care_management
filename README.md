# Healthcare Agentic AI Platform — Phase 1 + Phase 2

This is the foundation of a multi-agent healthcare assistant. Phase 1
implemented the evidence-grounding and safety-gate mechanism fully and
correctly rather than stubbing many agents shallowly. Phase 2 adds three
more agents on top of that same mechanism, real (pluggable) embedding/LLM
wiring, and a Streamlit UI.

**Full enterprise scope note:** the original spec called for FHIR, full
LangGraph orchestration, auth/RBAC, PHI masking, observability
(LangSmith/Prometheus/Grafana), Kubernetes/Helm/Terraform, and a React
frontend. None of that is faked here — it's listed honestly in the
Roadmap table below as not-yet-built, so you know exactly what this
codebase does and doesn't do today.

## Phase 2 additions

- **`app/rag/embeddings.py`** — pluggable embedding provider (OpenAI /
  local BGE-M3 via sentence-transformers / deterministic test-mode with
  no external dependency). Defaults to test mode so the whole stack runs
  without API keys.
- **`app/services/llm_client.py`** + **`app/prompts/guideline_compose_prompt.py`**
  — pluggable LLM composition (Anthropic / OpenAI / test mode) with a
  strict "use only these chunks, no outside knowledge, no diagnosis or
  dosage" system prompt.
- **`app/agents/medication_safety_agent.py`** — drug-interaction /
  contraindication checks; forces human review whenever the query touches
  pregnancy, renal/hepatic function, allergies, or interactions.
- **`app/agents/diagnosis_support_agent.py`** — differential suggestions
  only; `requires_human_review=True` is set unconditionally, at the code
  level, regardless of confidence — this agent can never produce a
  "final" answer.
- **`app/agents/escalation_agent.py`** — rule-based triage short-circuit
  for critical symptoms (chest pain, stroke signs, sepsis, cancer
  suspicion, drug allergy, etc.) that bypasses AI answering entirely.
- **`ui/streamlit_app.py`** — a real, running UI. It calls the FastAPI
  backend over HTTP and holds no safety logic itself: confidence display,
  citation drill-down, and the human-review warning banner all reflect
  what the API actually returned.
- **12 tests total, all passing**, including live verification: the API
  was booted and hit with a real HTTP request during development
  (`POST /api/v1/escalation/check` with a chest-pain query correctly
  returned `risk_level: critical`, `requires_human_review: true`).

## What's real in this drop

- **`app/core/evidence.py`** — the `EvidenceGroundedAnswer` model. It is
  structurally impossible to construct a non-refused clinical answer with
  zero citations (Pydantic validator enforces it, tested in
  `tests/test_safety_gate.py`).
- **`app/core/safety_gate.py`** — the single choke point all agent output
  passes through: confidence-below-threshold → forced refusal;
  critical-symptom keywords → forced human review; insufficient citations
  → forced refusal.
- **`app/rag/retriever.py`** — hybrid retrieval (dense search now, keyword
  search stubbed with an honest `NotImplementedError`-free no-op rather
  than fake results) with reciprocal rank fusion, plus a retrieval-based
  (not LLM-self-reported) confidence score.
- **`app/agents/guideline_retrieval_agent.py`** — reference agent showing
  the retrieve → score → compose → safety-gate pattern every other agent
  should follow.
- **`app/main.py` + `app/routers/guidelines.py`** — a working FastAPI
  endpoint (`POST /api/v1/guidelines/query`) wired end-to-end, though the
  embedding and LLM-composition calls are intentionally left as explicit
  `NotImplementedError` stubs (see below) rather than faked.
- **`app/models/core_models.py`** — Postgres schema for `documents`,
  `embeddings`, `agent_runs` (full audit trail per answer), `audit_logs`.
- **5 passing tests** proving the safety rules can't be bypassed.
- `docker-compose.yml` + `Dockerfile` to run Postgres/Mongo/Redis/Qdrant/API
  locally.

## What's intentionally stubbed (not faked)

Two functions raise `NotImplementedError` with a comment pointing at the
Phase they belong to, instead of pretending to work:
- `app/services/dependencies.py::_embed_fn` — needs a real embedding
  provider (BGE-M3 via sentence-transformers, or OpenAI embeddings).
- `app/services/dependencies.py::_llm_compose_fn` — needs a real LLM call
  with a strict "use only these chunks" system prompt.

Wiring these is the first task of Phase 2 — it's a small amount of code,
but it needs real API keys and a running Qdrant collection to test
against, which is better done together with data ingestion.

## Roadmap

| Phase | Scope |
|---|---|
| 1 (done) | Safety gate, evidence model, hybrid retriever, one reference agent, FastAPI skeleton, core DB schema |
| 2 (done) | Real pluggable embeddings + LLM composition; Escalation, Medication Safety, Diagnosis Support agents; Streamlit UI |
| 3 | Document ingestion pipeline (PDF/DOCX/FHIR → chunk → embed → Qdrant); FHIR REST resources |
| 4 | LangGraph multi-agent orchestration (supervisor, conditional edges, human-approval node, checkpointing) |
| 5 | Auth (JWT/OAuth2/RBAC), PHI/PII masking, prompt-injection detection |
| 6 | Observability (LangSmith, Prometheus/Grafana, OpenTelemetry) |
| 7 | React frontend (production UI, replacing/complementing Streamlit), Kubernetes/Helm/Terraform, CI/CD |

## Running locally

**Everything (API + Postgres/Mongo/Redis/Qdrant + Streamlit UI):**
```bash
cp .env.example .env
docker compose up --build
# API docs:    http://localhost:8000/docs
# Streamlit UI: http://localhost:8501
```

**API + UI only, without Docker** (uses test-mode embeddings/LLM by
default, so no API keys or Qdrant needed for the escalation agent; the
guideline/medication/diagnosis agents still need Qdrant running for
retrieval):
```bash
pip install -r requirements.txt
PYTHONPATH=. uvicorn app.main:app --reload &
streamlit run ui/streamlit_app.py
```

Run tests (no external services required — agents are tested via
dependency-injected fakes, not live Qdrant/LLM calls):
```bash
PYTHONPATH=. pytest tests/ -v
```

### Switching from test-mode to real providers

In `.env`, set:
```
EMBEDDING_PROVIDER=openai   # or bge_m3
LLM_PROVIDER=anthropic      # or openai
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
```
Test mode (`EMBEDDING_PROVIDER=test`, `LLM_PROVIDER=test`) is the
default specifically so this repo is runnable and demoable without any
credentials — it must never be used against real clinical documents,
since its "answers" are literal chunk concatenation, not real synthesis.
