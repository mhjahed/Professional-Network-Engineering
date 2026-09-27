# Learning notes — Project B (secure-opsdesk) & Project C (api-threat-monitor)

Two runnable teaching demos live next to this file. They are deliberately *small* — each one
isolates the ideas that make the real projects hard, so you can read the whole thing in an hour.

| Folder | What it demonstrates | Run |
|---|---|---|
| `secure_opsdesk_demo/` | tenancy, RBAC, service layer, idempotency, state machine, audit log, correlation IDs, health checks, login rate limiting, N+1-safe pagination, idempotent SLA escalation | `python manage.py test -v 2` · `python manage.py seed` · `python manage.py runserver` |
| `api_threat_monitor_demo/` | validate → normalize → pseudonymize pipeline, sliding-window rule engine (4 rules), alert model with severity/confidence/tags/MITRE, synthetic traffic generator with self-scoring, unit tests | `python demo.py` · `python -m pytest -q` |

Demo users (password `correct-horse-battery`): acme → alice (requester), bob (technician), carol (manager), dave (admin);
globex → eve (admin), frank (requester). Browsable API login: `/api-auth/login/`, then open `/api/orgs/acme/tickets/`.

---

## 1. Project B — secure-opsdesk in one paragraph

A **multi-tenant IT service desk + asset inventory** (think a small Jira Service Management / GLPI). Several
organizations share one deployment; inside each org, people have roles (requester → technician → manager →
admin); they raise tickets that move through a workflow with SLA clocks; technicians track assets and who holds
them; a knowledge base deflects tickets; everything security-relevant is audited. The *product* is ordinary on
purpose — the point is that every production concern (isolation, authorization, background jobs, idempotency,
observability, backups, security reasoning) has to be done properly.

### Architecture

```
 React+TS SPA ──HTTPS──▶ nginx ──▶ Django/DRF (gunicorn)  ──▶ PostgreSQL
                                     │  │  ▲                    ▲
                  OpenAPI schema ◀┘  │  │ /health/live,ready │
                                      ▼  │                    │
                                   Redis ◀── Celery workers ─┘   (email, webhooks, SLA scan, retention)
                                        ▲          ▲
                                   Celery beat  Mailpit (dev inbox)
 GitHub Actions: ruff · mypy · pytest (+coverage) · docker build · schema diff · pip-audit/bandit
```

### The engineering requirements — what each one *means* and where the demo shows it

| Requirement | Why it exists | Demo location |
|---|---|---|
| Service layer | business rules must be testable without HTTP and callable from Celery | `core/services.py` |
| DB constraints & transactions | Python checks race; the database doesn't | `core/models.py` (`UniqueConstraint`, `CheckConstraint`), `transaction.atomic` + `select_for_update` in services |
| Idempotency key | clients retry on timeouts; a retry must not create a second ticket | `IdempotencyRecord`, `create_ticket()`, tests in `IdempotencyTests` |
| Pagination & query optimization | list endpoints must be O(1) queries no matter the row count | `get_queryset()` with `select_related`, `assertNumQueries(4)` test |
| Background jobs with retry + dead-letter | email/webhooks fail; re-runs must be safe | `escalate_breached_tickets()` is idempotent via `escalated_at` (Celery `autoretry_for`, `retry_backoff`, `on_failure → FailedJob` table in the real project) |
| Permission tests per role | authz bugs are the #1 web vuln class (BOLA/IDOR) | `RolePermissionMatrixTests`, `TenantIsolationTests` |
| Rate limiting on auth/public endpoints | credential stuffing, scraping | `ThrottledLoginView` (5/min → HTTP 429) |
| File type/size validation | uploads = malware + DoS vector | *(not in demo)* allow-list extensions, sniff MIME with `python-magic`, cap size, random storage names, `Content-Disposition: attachment` |
| Structured logs + correlation IDs | tracing one request across API + worker | `core/middleware.py`, `X-Request-ID` echoed, stored in `AuditLog.request_id` |
| `/health/live` & `/health/ready` | orchestrator restarts vs. load-balancer routing are different questions | `core/views.py` (`ready` = DB + migrations; add Redis in the real stack) |
| Backup & restore runbook | a backup you never restored is a hope, not a backup | *(doc)* `pg_dump -Fc` nightly, restore drill into a scratch DB, record RPO/RTO |
| ADRs in `docs/adr/` | interviewers ask "why", not "what" | *(doc)* e.g. ADR-001 shared-DB tenancy, ADR-002 token vs session auth, ADR-003 attachment storage, ADR-004 Celery vs DB-queue |

### Multi-tenancy — the decision that shapes everything
Three options: **shared DB + `org_id` column** (chosen: simplest, cheapest, easy backups; isolation is code
discipline + optional PostgreSQL Row-Level Security), **schema-per-tenant** (`django-tenants`; stronger isolation,
painful migrations at scale), **DB-per-tenant** (strongest; operationally heavy). Write this up as your first ADR.

### Ticket state machine
`new → open → pending ⇄ open → resolved → closed`, with `resolved → open` for re-opens. Every transition writes a
`TicketStatusHistory` row and an `AuditLog` row *inside the same transaction*. Illegal transitions are HTTP 409.

### Security case study — how to approach it
1. **Assets & trust boundaries**: tenant data, credentials/API keys, attachments, audit log; boundaries = browser↔API,
   API↔DB/Redis, API↔email/webhook targets, tenant↔tenant.
2. **Attackers/abuse**: malicious requester in tenant A reading tenant B; disgruntled technician exporting data;
   credential stuffing; stolen API key; malicious attachment; SSRF via webhook URL; SLA job abuse.
3. **Top risks**: BOLA/IDOR across tenants, privilege escalation via role edits, auth brute force, upload abuse,
   webhook SSRF, secrets in logs.
4. **Controls**: everything in the table above + CSP/secure cookies + hashed API keys + webhook URL allow-list/DNS
   re-resolution + secret scanning in CI.
5. **Security tests**: the permission matrix, tenant-isolation tests, throttle tests, upload-rejection tests,
   `bandit`/`pip-audit` in CI, a ZAP baseline scan against *your own* Compose stack.
6. **Remaining risks**: shared-DB isolation relies on code; no MFA; no WAF; single-region; log retention.

---

## 2. Project C — api-threat-monitor in one paragraph

A **mini SIEM for web/API traffic**. Something (a generator, or opsdesk's own logs) posts security events to a
documented ingestion API; events are validated, normalized, pseudonymized and stored; a worker runs them through a
rule engine that keeps per-entity sliding windows; alerts get severity, confidence, status, tags and — only where
honest — MITRE ATT&CK IDs; analysts triage them in a dashboard with notes; a retention job deletes raw events.
You never touch anyone else's system: the "attacks" are synthetic.

### Architecture

```
 generator / opsdesk webhook ──POST /api/v1/events──▶ Ingestion API (validate 422, HMAC-signed source key, idempotent batch id)
                                                          │ enqueue
                                                          ▼
                                                   Worker: normalize → pseudonymize → store → RuleEngine
                                                          │                                   │
                                                   PostgreSQL (events, alerts, notes)      Redis (sliding windows: ZSET per entity)
                                                          ▲
                                   Dashboard/API: alerts list, filters, notes, CSV/JSON export, retention job (beat)
```

### Rule engine — the core idea (see `rules.py`)
* one class per rule, `evaluate(event) -> Alert | None`;
* state = sliding window keyed by entity (`ip:hash`, `user:hash`); in production a Redis sorted set
  (`ZADD`, `ZREMRANGEBYSCORE`, `ZCARD`) gives you the same thing across many workers;
* **cooldown** per (rule, entity) so an attack of 5,000 requests yields one alert, not 5,000;
* tune with the generator: measure detections vs. false positives, then adjust thresholds/windows.

| Rule | Signal | Severity / MITRE |
|---|---|---|
| R1 brute force | ≥N auth failures per IP (spraying if many users) or per account from many IPs | high · T1110.001 / T1110.003 |
| R2 impossible rate | ≥N requests per IP per minute; scanning if many distinct paths + 404s | medium/high · T1595.002 only for the scanning shape |
| R3 authz probing | ≥N 403s for one user across *distinct* resources | medium · no clean ATT&CK technique — say so |
| R4 suspicious UA | scanner signatures, empty UA, UA rotation | high / low / medium · T1595.002 for scanners |

### Privacy-safe data model
Store `HMAC(ip)` and `HMAC(username)` with a rotating key, plus the `/24` network; keep raw events for N days
(retention job), keep aggregates longer; export excludes anything re-identifying. Lesson from the first demo run:
correlated rules can double-alert — R1's account branch now fires only for *distributed* attacks.

---

## 3. How the two projects connect
opsdesk already emits structured JSON request logs and an audit log. Ship them (log shipper or outbound webhook)
to threat-monitor's ingestion API and the monitor detects the brute force that opsdesk's throttle is also blocking —
**defence in depth**, and one coherent portfolio story: *I built the app, I built the controls, I built the detection.*

## 4. Things the demos deliberately skip (real project must-haves)
PostgreSQL + Redis + Celery in Docker Compose · React client · OpenAPI (`drf-spectacular`) · attachments ·
assets + assignment history · knowledge base with Postgres full-text search · email (Mailpit) · API keys/webhooks ·
dashboard aggregates · GitHub Actions · backup runbook · ADRs · SECURITY_CASE_STUDY.md · Prometheus/Grafana.
