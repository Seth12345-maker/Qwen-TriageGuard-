# TriageGuard Backend

AI Incident & Compliance Triage Agent — backend for the Qoder AI Hackathon.

Pipeline: **Input (text/photo/voice)** → **LLM extraction (structured JSON)** →
**LLM classification (category + severity + confidence)** → **RAG policy retrieval
(deterministic keyword search with citations)** → **LLM draft (report + recommended
actions)** → **human review queue** → **dashboard + audit log**.

Built with FastAPI + pydantic v2. Data persists to local JSON files — no database
setup needed for the demo.

## Quick start

Requires Python 3.10+.

```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # demo mode is ON by default — no API key needed
uvicorn app.main:app --reload --port 8000
```

Open http://localhost:8000/docs for the interactive API docs.

## Try it (demo mode, no LLM required)

```bash
# Health check — shows demo_mode, policies loaded, incidents stored
curl http://localhost:8000/api/health

# Submit an incident (pre-written pipeline output for the 3 demo scenarios)
curl -X POST http://localhost:8000/api/triage \
  -H "Content-Type: application/json" \
  -d '{"description": "A contractor left a cable across the walkway near Warehouse 3; someone nearly tripped."}'

# Dashboard list (filter: ?status=&severity=&category=)
curl http://localhost:8000/api/incidents

# Human review — approve / escalate (reason required) / edit
curl -X POST http://localhost:8000/api/incidents/<id>/review \
  -H "Content-Type: application/json" \
  -d '{"action": "escalate", "actor": "hse-officer", "reason": "Area still unsafe"}'

# Audit trail for an incident
curl http://localhost:8000/api/incidents/<id>/audit
```

Seed the 3 documented demo scenarios and check them against expected outputs:

```bash
python -m scripts.seed
```

End-to-end smoke test against a running server (stdlib only, no extra deps):

```bash
python -m scripts.smoke_test
```

## Demo mode vs. live LLM

`DEMO_MODE=true` (default) returns pre-written pipeline outputs for the three
documented demo scenarios — the demo survives Wi-Fi/API failure. Any other
description falls back to keyword-signature matching, then the first sample.

Set `DEMO_MODE=false` and provide an OpenAI-compatible endpoint to run the real
pipeline:

| Variable          | Default                       | Notes                                  |
| ----------------- | ----------------------------- | -------------------------------------- |
| `DEMO_MODE`       | `true`                        | Offline fallback for the demo          |
| `LLM_API_KEY`     | (empty)                       | Never commit; `.env` is gitignored     |
| `LLM_BASE_URL`    | `https://api.openai.com/v1`   | Any OpenAI-compatible endpoint         |
| `LLM_MODEL`       | `gpt-4o-mini`                 |                                        |
| `LLM_TIMEOUT`     | `30`                          | Seconds                                |
| `CONFIDENCE_THRESHOLD` | `0.6`                    | Below this, incident requires human approval |
| `DATA_DIR`        | `backend/data`                | Policies, incidents, audit log, demos  |
| `HOST` / `PORT`   | `127.0.0.1` / `8000`          |                                        |

## Guardrails (enforced in code)

- All AI steps return structured JSON (validated with pydantic; one repair retry).
- Critical severity **or** confidence below threshold ⇒ status `awaiting_approval`,
  `requires_human_approval=true`. There is no auto-execution path.
- An edit that upgrades severity to `critical` flips the incident back to
  `awaiting_approval` — even mid-review.
- Escalation requires a reason (422 otherwise).
- Every action is written to the audit trail: timestamp, actor, action, previous
  value, new value, reason.
- Policy citations attach to every recommendation (document + section + excerpt).
- Sample/anonymized data only — do not submit real personal data.

## Layout

```
app/
  main.py        FastAPI app, CORS, startup init
  config.py      env config with tiny .env loader
  models.py      pydantic models + enums
  llm.py         OpenAI-compatible JSON chat client
  rag.py         keyword policy index (policies/*.md, split by ## sections)
  demo.py        pre-written demo outputs + scenario picker
  pipeline.py    extraction → classification → RAG → draft → storage + audit
  storage.py     JSON-file persistence (incidents, audit log)
  routes/        triage, incidents, review, audit_log, health
data/
  policies/          8 markdown SOPs (the RAG knowledge base)
  demo_responses/    pre-written pipeline outputs (demo mode)
  samples/           sample inputs + expected outputs
scripts/
  seed.py        run 3 samples through the pipeline, PASS/FAIL vs expected
  smoke_test.py  end-to-end API test (run server first)
```

## Swapping in the backup idea (SME Invoice Reconciler)

The pipeline is input-agnostic. For the invoice reconciler: replace the extraction
prompt (invoice fields instead of incident fields), swap the policy docs
(`approval_policy.md`, `vat_policy.md`, vendor master), and change classification
to match/fraud-flag. The review queue, audit log, and dashboard endpoints stay
as-is. Do this only if TriageGuard is disqualified — the primary idea is stronger.
