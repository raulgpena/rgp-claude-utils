---
name: Logging improvements (May 2026)
description: Structured logging added to all route handlers and services for troubleshooting; Sentry PII fixed
type: project
originSessionId: fcb30f6e-cbeb-43cf-961c-51d670544060
---
Added structured logging throughout the FastAPI backend for observability.

**Why:** Zero operational logging in routes made troubleshooting impossible — no visibility into which pipelines ran, for which patients, or how long they took.

**Changes made:**
- `app/core/logging_config.py`: Added `_SensitiveDataFilter` — redacts fields named password, first_name, last_name, genotype, token, etc. from all log records as defense-in-depth.
- `app/main.py`: Fixed `send_default_pii=True` → `False` in Sentry init (was leaking PHI). Added `_sanitize_path()` that replaces numeric IDs in URL paths with `{id}` before logging them in exception handlers.
- `app/routes/ai_router.py`: Added correlation_id + structured start/complete/fail logs with duration_ms to every endpoint. Fixed `detail=str(e)` → `"Internal server error"` on 500s to stop leaking exception internals to clients. Added error handling + logging inside streaming event_stream generators.
- `app/routes/data_router.py`: Same pattern — logs patient_id, page/per_page, has_search (bool, not the search value itself).
- `app/data/data_service.py`: Added WARNING logs when patient/results not found.

**How to apply:** All log entries include `correlation_id` from `get_correlation_id()` in the middleware, so a single request can be traced end-to-end across middleware → route → service layers.
