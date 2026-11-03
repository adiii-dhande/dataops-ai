# DataOps AI

### Agentic incident investigation for data pipelines: diagnose, verify, recommend, with a human approving every write.

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-Agent_Workflows-1C3C3C)
![FastAPI](https://img.shields.io/badge/FastAPI-Backend-009688?logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/Postgres-pgvector-4169E1?logo=postgresql&logoColor=white)
![MCP](https://img.shields.io/badge/MCP-Tool_Servers-black)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![CI](https://img.shields.io/badge/CI-GitHub_Actions-2088FF?logo=githubactions&logoColor=white)

> **Status: work in progress.** See [Project Status](#2-project-status) for what is built vs planned. Nothing in this README is claimed as working unless it is checked off there.

---

## 1. Problem

Monitoring tells you a pipeline failed. It does not tell you why, what it affects, or what is safe to do next. An engineer still has to find the run, read logs, profile the data, search runbooks, write validation SQL, and decide on a fix.

DataOps AI automates the *investigation*, not the *repair*. It gathers evidence with read-only tools, separates verified findings from hypotheses, and proposes a fix that a human must approve.

**Design stance:** this is a bounded workflow with a few agentic steps, not a free-roaming autonomous agent. Failure classification and approval are deterministic. The LLM decides which diagnostic to run next and writes the explanation.

## 2. Project Status

| Component | Status |
|---|---|
| Log parser + failure classifier | ⬜ Planned |
| RAG over runbooks (pgvector, hybrid + rerank) | ⬜ Planned |
| LangGraph investigation workflow | ⬜ Planned |
| Diagnostic tools (log, profiler, schema, SQL) | ⬜ Planned |
| Human approval (LangGraph `interrupt` + checkpointer) | ⬜ Planned |
| FastAPI service | ⬜ Planned |
| Eval harness + benchmark (100 incidents) | ⬜ Planned |
| Tracing / observability | ⬜ Planned |
| Streamlit dashboard | ⬜ Planned |

Update this table as you build. Replace ⬜ with ✅ only when it is tested.

## 3. Results

Filled in only after running the benchmark. No numbers are claimed before that.

| Metric | Baseline (no tools, no RAG) | DataOps AI |
|---|---|---|
| Root-cause accuracy | TBD | TBD |
| Evidence grounding | TBD | TBD |
| Hallucination rate | TBD | TBD |
| Recall@5 / MRR (retrieval) | TBD | TBD |
| Tool-call success rate | n/a | TBD |
| p50 / p95 latency | TBD | TBD |
| Cost per investigation | TBD | TBD |

The baseline is a single LLM call with the same incident text. If the agent cannot beat it, the agent is not justified.

## 4. Example

Input:

```json
{
  "pipeline_name": "customer_revenue_etl",
  "source": "aws_glue",
  "error_type": "DATA_QUALITY_FAILURE",
  "dataset": "transactions",
  "error_message": "Duplicate transaction IDs detected",
  "execution_id": "glue-run-1042"
}
```

Output (illustrative, schema-validated with Pydantic):

```json
{
  "incident_id": "INC-1042",
  "failure_category": "duplicate_records",
  "root_cause": {
    "statement": "Source delivered duplicate transaction_id values.",
    "status": "verified",
    "confidence": 0.0,
    "evidence": [
      {"tool": "data_profiler", "finding": "N duplicate keys in partition dt=..."},
      {"tool": "runbook_retriever", "source": "runbooks/dedup.md#chunk-4"}
    ]
  },
  "recommended_actions": [],
  "requires_approval": true
}
```

`status` is `verified` (backed by tool output) or `hypothesis` (not yet confirmed). The agent may not emit a root cause without at least one evidence item.

## 5. Architecture

```mermaid
flowchart TD
    A["Incident (API / CloudWatch / Airflow)"] --> B["Context Builder"]
    B --> C["Classify Failure"]
    C --> D["Retrieve Runbooks + Similar Incidents"]
    D --> E["Plan Next Diagnostic"]
    E --> F["Run Read-only Tool"]
    F --> G["Evidence Check"]
    G --> H{"Sufficient or budget hit?"}
    H -->|No| E
    H -->|Yes| I["Root Cause + Remediation Proposal"]
    I --> J["Interrupt: Human Approval"]
    J -->|Approved| K["Dry-run, then Execute"]
    J -->|Rejected| L["Report Only"]
    K --> M["Audit Log + Incident Report"]
    L --> M
```

Key decisions:

| Decision | Reason |
|---|---|
| LangGraph with explicit state, not a bare ReAct loop | Deterministic control flow, resumable runs, testable nodes |
| Postgres checkpointer + `interrupt()` for approval | Approval survives restarts; the run pauses instead of polling |
| Max steps and token budget per investigation | Prevents runaway loops and cost |
| Tools exposed via **MCP servers** | Tools are reusable by other clients and permission-scoped per server |
| Structured outputs (Pydantic / JSON schema) | No free-text parsing of agent decisions |
| Agent never holds write credentials | Writes happen only in the approved execution step, outside the LLM |

## 6. Stack

| Layer | Choice |
|---|---|
| Language | Python 3.11+, `uv` for dependency management |
| Orchestration | LangGraph (state, checkpointing, interrupts) |
| LLM access | Provider-agnostic interface (OpenAI / Anthropic / Gemini), model set by env var |
| Tools | MCP servers (log analyzer, profiler, schema diff, SQL validator) |
| Retrieval | Postgres + pgvector, hybrid (BM25 + vector), cross-encoder reranker |
| SQL safety | `sqlglot` parsing, allow-list of statements, read-only DB role |
| Data checks | Pandas / PySpark, optional Great Expectations or Soda results as input |
| API | FastAPI, Pydantic v2 |
| Cloud | AWS Glue, S3, CloudWatch, Lambda |
| Observability | OpenTelemetry + Langfuse (traces, token cost, latency per step) |
| Evals | Custom benchmark + Ragas (retrieval) + LLM-as-judge with human spot-check |
| Quality | pytest, ruff, mypy, GitHub Actions |
| Packaging | Docker Compose |

## 7. Retrieval (RAG)

1. Load runbooks, Glue docs, schema definitions, and resolved incident reports.
2. Chunk by document structure (headings), with overlap; keep metadata (pipeline, system, doc type, date).
3. Embed and store in pgvector; maintain a Postgres full-text index alongside.
4. Retrieve top-k with **hybrid search**, merge with reciprocal rank fusion, then **rerank**.
5. Filter by metadata (e.g., pipeline, source system) before generation.
6. Every claim must cite a chunk ID. Uncited claims are flagged by the evaluator.

Historical incidents are context, not proof. A past fix is a hypothesis for the current incident.

## 8. Tools (read-only)

| Tool | Input | Output |
|---|---|---|
| `analyze_logs` | run id, time window | error clusters, first-failure timestamp |
| `profile_dataset` | dataset, partition | null %, duplicate keys, distribution drift |
| `diff_schema` | expected vs actual | added / removed / retyped columns |
| `check_freshness` | dataset | last update vs SLA |
| `count_records` | dataset, partitions | volume vs trailing baseline |
| `run_sql` | validated query | capped rows (read-only role, timeout, row limit) |
| `search_runbooks` | query, filters | ranked, cited chunks |

All tools use typed schemas, timeouts, and row/size caps.

## 9. Security

Pipeline logs and source data are **untrusted input**. A log line can contain text that tries to instruct the model (prompt injection). Mitigations:

- Treat all tool output as data; never as instructions. Delimit it in prompts.
- Read-only DB role for all agent queries; SQL parsed and checked with `sqlglot` before execution.
- Allow-list tools per graph node; no tool can write.
- Write actions are separate code, run only after approval, with dry-run first.
- PII redaction before any text leaves your environment for an external LLM; option to use a private/VPC-hosted model.
- Secrets via AWS Secrets Manager or env, never in prompts or traces.
- Full audit log: who approved, what ran, inputs, outputs.
- Per-run step, token, and time budgets.

## 10. Evaluation Plan

Benchmark of 100+ labeled incidents (synthetic plus anonymized), covering duplicates, nulls, schema drift, missing files, freshness breaches, and volume anomalies. Include **adversarial cases**: ambiguous failures, multiple simultaneous causes, injected instructions in logs, and incidents with no sufficient evidence (the correct answer is "insufficient evidence").

Evaluate three levels separately:

1. **Retrieval:** Recall@k, MRR.
2. **Agent trajectory:** correct tool chosen, redundant calls, steps to resolution.
3. **Final answer:** root-cause accuracy, evidence grounding, hallucination rate, safe-action rate.

Run the eval suite in CI on every prompt or graph change to catch regressions. Always report against the no-agent baseline.

## 11. Setup

```bash
git clone https://github.com/adiii-dhande/dataops-ai.git
cd dataops-ai
uv sync
cp .env.example .env        # fill in values
docker compose up -d db     # Postgres + pgvector
python scripts/ingest_documents.py
uvicorn api.main:app --reload
streamlit run dashboard/app.py
```

`.env.example`:

```env
LLM_PROVIDER=openai
LLM_MODEL=<model-name>
EMBEDDING_MODEL=<embedding-model>
DATABASE_URL=postgresql://user:pass@localhost:5432/dataops
READONLY_DATABASE_URL=postgresql://ro_user:pass@localhost:5432/dataops
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
MAX_AGENT_STEPS=8
MAX_TOKENS_PER_RUN=50000
```

## 12. API

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | Health check |
| POST | `/incidents` | Submit incident |
| GET | `/incidents/{id}` | Incident details |
| POST | `/investigations` | Start investigation |
| GET | `/investigations/{id}` | Status, trace link, results |
| POST | `/investigations/{id}/approve` | Approve or reject proposed action |
| GET | `/metrics` | Latency, cost, success rate |

## 13. Project Structure

```text
dataops-ai/
├── agents/          # graph, nodes, state, prompts
├── api/             # FastAPI app, routes, schemas
├── ingestion/       # log parsing, doc loading
├── rag/             # chunking, embeddings, hybrid retriever, reranker
├── tools/           # MCP servers + tool schemas
├── evaluation/      # benchmark, retrieval/agent/answer metrics
├── dashboard/       # Streamlit UI
├── data/            # synthetic logs, datasets, runbooks
├── tests/
├── scripts/
├── .github/workflows/ci.yml
├── docker-compose.yml
├── pyproject.toml
└── README.md
```

## 14. Known Limitations

- Accuracy depends on evidence available and runbook quality.
- Synthetic incidents do not cover all production failure modes.
- LLM output can be wrong; every action needs human approval.
- Single-pipeline scope in v1; cross-pipeline dependency analysis is future work.

## 15. Roadmap

- [ ] Multi-agent split (triage / investigator / remediator) only if evals show it helps
- [ ] Airflow and Databricks integrations
- [ ] Slack / Teams approval flow
- [ ] Lineage-aware impact analysis
- [ ] Cost-aware model routing (small model for triage, larger for RCA)
- [ ] Prompt and graph versioning with regression gates

## 16. License and Author

MIT License.

**Aditya Dhande** · [LinkedIn](https://linkedin.com/in/aditya-dhande-data) · [GitHub](https://github.com/adiii-dhande)
