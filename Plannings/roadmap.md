# Roadmap — secure-opsdesk (Weeks 5–14) and api-threat-monitor (Weeks 15–20)

Assumptions: solid backend developer, 20+ focused hours/week (≈ 200 h for B, ≈ 120 h for C).
Budget split that works: **60 % build · 20 % tests & security · 20 % docs/ops**. If docs slip to "the end", they never happen.

## 0. Working rules (these are what make it "production-grade", not the feature list)

1. **Vertical slices.** Every week ends with something demo-able through the API and covered by tests — never "models done, views next week".
2. **Authorization tests are written with the endpoint, not after.** Each new endpoint gets a role × tenant matrix test the same day.
3. **One ADR per real decision** (≤ 1 page: context, options, decision, consequences). Interviewers ask "why", not "what".
4. **CI is green on `main` at all times.** Branch → PR → CI → squash-merge, even solo. Your commit history is part of the portfolio.
5. **Friday ritual (1 h):** update README/CHANGELOG, record a 60-second screen capture of the new slice, write down the next week's cut line.
6. **Must / Should / Could** labels on every item below. If you are behind on Wednesday, drop Coulds first, then Shoulds — never Musts.

---

## 1. secure-opsdesk — repository layout (decide once, week 5)

```
secure-opsdesk/
├── backend/
│   ├── config/                 settings/{base,dev,test,prod}.py, urls.py, celery.py, asgi/wsgi
│   ├── apps/
│   │   ├── accounts/           custom User (email login), invitations, API keys
│   │   ├── tenancy/            Organization, Membership, roles, tenant middleware/mixins
│   │   ├── tickets/            models, services/, selectors/, api/, tasks.py, tests/
│   │   ├── assets/
│   │   ├── knowledge/
│   │   ├── notifications/      email + webhook delivery, FailedTask dead-letter
│   │   ├── audit/
│   │   └── ops/                health, metrics, request-id middleware, management commands
│   ├── tests/                  cross-app: permission matrix, tenant isolation, smoke
│   └── pyproject.toml          ruff, mypy (strict on services/), pytest, coverage ≥ 85 % on services
├── frontend/                   Vite + React + TS, generated API client
├── deploy/
│   ├── docker-compose.yml      api, worker, beat, web, postgres:16, redis:7, mailpit, (minio)
│   ├── docker-compose.prod.yml gunicorn, nginx, read-only FS, non-root
│   └── nginx/
├── docs/
│   ├── adr/                    0001-tenancy-model.md …
│   ├── runbooks/               backup-restore.md, stuck-queue.md, rotate-secrets.md, incident-template.md
│   ├── architecture.md         C4-ish diagrams (mermaid)
│   └── api/openapi.yaml        committed + diffed in CI
├── SECURITY_CASE_STUDY.md
├── SECURITY.md                 how to report, supported versions (signals maturity)
├── Makefile                    make up / test / lint / seed / demo / backup / restore
└── .github/workflows/ci.yml
```

Pattern inside each app: `services/` (writes, transactions) · `selectors/` (reads, query optimization) · `api/` (serializers, views, thin) · `tasks.py` (Celery, calls services) · `tests/`.

---

## 2. secure-opsdesk — week by week

### Week 5 — Walking skeleton & delivery pipeline
| | Item |
|---|---|
| Must | Compose stack boots with one command; Postgres 16, Redis 7, Mailpit; settings from env (`django-environ`); custom `User` (email as username) |
| Must | `/health/live`, `/health/ready` (DB, Redis, pending migrations); request-ID middleware; JSON logs |
| Must | CI: ruff + mypy + pytest against a Postgres service container + `docker build`; pre-commit hooks |
| Must | ADR-001 tenancy (shared DB + `org_id`, with RLS as later hardening) · ADR-002 auth for the SPA (HttpOnly session cookie + CSRF vs JWT — recommend session; document why) · ADR-003 app/service layout |
| Should | `make seed` with two tenants and one user per role; factory_boy factories |
| Could | devcontainer / `.env.example` validator |
| **DoD** | fresh clone → `make up && make test` green in < 10 min; README quickstart; CI badge |

### Week 6 — Tenancy, roles, tickets core
| | Item |
|---|---|
| Must | Organization, Membership(role), invitation by email (token, expiry, single-use) |
| Must | Ticket: per-tenant number, category, priority, state machine + `TicketStatusHistory`, comments (public/internal), assignee; all writes via services in `transaction.atomic` |
| Must | DB constraints: unique(org, number), check(status/priority), FK `PROTECT` where deletion would orphan history |
| Must | Cursor pagination, `select_related/prefetch_related`, `assertNumQueries` tests on every list endpoint |
| Must | OpenAPI via `drf-spectacular`, schema committed and diffed in CI |
| Must | **Permission matrix tests: 4 roles × every endpoint × own-tenant/other-tenant** (parametrized) |
| Should | Ticket filtering/sorting (`django-filter`), search on title |
| **DoD** | requester/technician/manager/admin flows work end-to-end via curl; matrix tests ≥ 60 cases |

### Week 7 — Attachments, audit log, idempotency, rate limiting
| | Item |
|---|---|
| Must | Attachments: size cap, extension allow-list, MIME sniff (`python-magic`), random storage name, served with `Content-Disposition: attachment` + `X-Content-Type-Options: nosniff`, stored on a volume (Should: MinIO S3-compatible → cloud story) |
| Must | `AuditLog` (append-only; DB trigger or model guard) + `audit(actor, action, target, meta)` helper; audited: login success/failure, role change, invitation, API-key create/revoke, attachment download, ticket delete, export |
| Must | `Idempotency-Key` on POST creates (tickets, webhooks): (org, key) unique, payload hash, replay with header, 422 on mismatch, 24 h retention job |
| Must | Throttles: login 5/min/IP + per-account lockout with backoff (`django-axes` or custom), password reset, public endpoints; anon vs user defaults |
| Must | ADR-004 attachment storage · ADR-005 idempotency design |
| Should | Upload-bypass tests (double extension, polyglot, oversized, wrong MIME) |
| **DoD** | security tests for every control above; audit log visible to admins (read-only endpoint) |

### Week 8 — Background jobs, notifications, SLA
| | Item |
|---|---|
| Must | Celery worker + beat; base task with `autoretry_for`, `retry_backoff=True`, `retry_jitter`, `max_retries`, `acks_late`; enqueue via `transaction.on_commit` |
| Must | **Dead-letter:** `on_failure` writes `FailedTask(task, args, exc, traceback, attempts)`; admin list + `requeue_failed` management command |
| Must | Email notifications (assign, status change, comment, SLA warning) to Mailpit; templates; unsubscribe per event type |
| Must | SLA policy per org (response/resolution hours by priority); `sla_due_at` computed in service; beat job every minute finds breaches → escalates **idempotently** (`escalated_at`), notifies manager, audit entry |
| Must | ADR-006 job queue & failure handling |
| Should | SLA pause while `pending` (clock stop) — great interview topic; business-hours calendar |
| Could | Correlation ID propagated in Celery headers so worker logs show the originating request |
| **DoD** | kill Redis mid-run → jobs retry and land in dead-letter; requeue works; tests cover retries with eager mode + unit tests on services |

### Week 9 — Assets & knowledge base
| | Item |
|---|---|
| Must | Asset (type, serial, status, purchase/warranty dates, location); `AssetAssignment(asset, user, from, to, by)` with a Postgres **exclusion constraint** preventing overlapping assignments; link assets ↔ tickets |
| Must | KB articles (draft/published, org-scoped), Postgres full-text search (`SearchVector` + GIN index, ranking, headline snippets) |
| Should | "Suggested articles" while typing a ticket title; asset import via CSV with validation report |
| Could | Asset QR label export |
| **DoD** | search tests, assignment-history integrity tests, CSV import rejects bad rows with line numbers |

### Week 10 — API keys, webhooks, dashboard
| | Item |
|---|---|
| Must | API keys: `prefix.secret`, store only hash, scopes (read:tickets, write:tickets…), expiry, last-used, revoke; DRF auth class; audited |
| Must | Outbound webhooks: per-org subscriptions, HMAC-SHA256 signature + timestamp header, delivery table (attempts, response code), retry via Celery, **SSRF guard** (https only, block private/link-local ranges, resolve DNS and pin) |
| Must | Inbound webhook: create ticket from monitoring with `Idempotency-Key` |
| Must | Dashboard endpoints: backlog by status/priority, SLA breached/at-risk, created vs resolved per day (30 d), technician load — aggregate queries with tests on SQL count |
| Must | ADR-007 API-key format · ADR-008 webhook security |
| **DoD** | replay a signed webhook to a local receiver; SSRF tests (169.254.169.254, 10.0.0.0/8, DNS rebinding case) |

### Week 11 — React + TypeScript client
| | Item |
|---|---|
| Must | Vite + React + TS; API client generated from OpenAPI (`openapi-typescript` + fetch wrapper or `orval`); auth (session + CSRF); ticket list/detail/create, comments, transitions, attachments; role-aware UI |
| Must | Dashboard page (recharts), KB search, assets list; error toasts that show the `X-Request-ID` ("quote this to support") |
| Should | Vitest + Testing Library for 3–4 critical components; Playwright smoke (login → create ticket) in CI |
| Could | Dark mode / polish — explicitly out of scope |
| **DoD** | a non-developer can complete the requester journey without curl |

### Week 12 — Observability, hardening, ops
| | Item |
|---|---|
| Must | `django-prometheus` (or OpenTelemetry) + Grafana dashboard JSON committed (RPS, p95, 5xx, queue depth, SLA breaches) |
| Must | Security headers (CSP, HSTS, Referrer-Policy, frame-ancestors), secure cookies, CORS allow-list, `manage.py check --deploy` clean |
| Must | Supply chain in CI: `pip-audit`, `bandit`, `gitleaks`, Dockerfile lint; images non-root, multi-stage, pinned digests |
| Must | **Backup & restore runbook**: nightly `pg_dump -Fc` + attachments volume; *perform* a restore drill into a scratch stack, record RPO/RTO and timings |
| Should | k6/locust load test against your own stack; fix the top 3 slow queries; document before/after |
| Should | Retention jobs: idempotency records, old failed tasks, invitation tokens |
| **DoD** | Grafana shows a live load test; runbook has real timestamps from a real restore |

### Week 13 — Security case study & security testing
| | Item |
|---|---|
| Must | Threat model: data-flow diagram, trust boundaries, STRIDE per boundary; abuse cases per role |
| Must | `SECURITY_CASE_STUDY.md` (6 required sections) written from the threat model, with links to the tests that prove each control |
| Must | Security test suite: IDOR across tenants for every object type, mass-assignment (role/org fields), upload bypasses, throttle/lockout, CSRF, SSRF, API-key scope violations |
| Must | OWASP ZAP baseline scan against **your own Compose stack**; triage → fix → document remaining risk |
| Should | Postgres Row-Level Security policies as defence-in-depth (ADR-009) — big differentiator |
| **DoD** | every "control implemented" line in the case study points to a passing test or a config line |

### Week 14 — Polish, documentation, demo
| | Item |
|---|---|
| Must | README: problem, architecture diagram, quickstart, screenshots, 2-minute video, "how I tested security", "what I'd do next" |
| Must | ADR index, runbooks index, CHANGELOG, tag `v1.0.0`, GitHub release |
| Must | Written STAR stories (see §5) — one per job track |
| Should | Blog post / LinkedIn article: "Building a multi-tenant service desk: 5 decisions and what they cost" |
| **DoD** | a stranger can run it, understand it, and break nothing in 15 minutes |

---

## 3. api-threat-monitor — stack decision and week by week

**Stack:** FastAPI + Pydantic v2 + SQLAlchemy 2 + Alembic + PostgreSQL + Redis, worker with `arq` (or Celery for consistency), pytest + httpx. Using a second framework shows range and FastAPI's Pydantic-first validation fits an ingestion API perfectly. If week 15 goes badly, fall back to Django/DRF — you already have the muscle memory. Record this as ADR-001 either way.

### Week 15 — Event schema & ingestion
| | Item |
|---|---|
| Must | Versioned event schema (`schema_version`, `event_type`, `occurred_at`, `source`, `http{method,path,status}`, `client{ip,user_agent}`, `actor{user_id}`, `auth{outcome}`); Pydantic models; `POST /api/v1/events:batch` (≤ 1 000 events, 1 MB) returning per-item 202/422 |
| Must | Source authentication (per-source API key, hashed), idempotent `batch_id`, size limits, 429 throttling |
| Must | **Pseudonymization at the door**: HMAC(ip), HMAC(user) with key id; raw values never persisted; `/24` retained; ADR-002 privacy model |
| Must | Integration tests (httpx): valid batch, mixed batch, oversized, replayed batch_id, bad key |
| **DoD** | OpenAPI docs published; 10k events/min ingested locally without errors |

### Week 16 — Async processing & rule-engine core
| | Item |
|---|---|
| Must | Queue + worker; `RuleEngine` with `Rule.evaluate(event) -> Alert | None`; sliding windows as **Redis ZSETs per (rule, entity)** with TTL; per-(rule, entity) cooldown |
| Must | R1 brute force (single-source, spraying, distributed) · R2 impossible rate (flood vs content-discovery shape) |
| Must | Alert model: severity, confidence, status, tags, `mitre[]`, evidence JSON, first/last seen, dedupe key; alert *update* (count++) instead of duplicate while open |
| Must | Unit tests with crafted sequences (threshold−1, threshold, window expiry, cooldown, classification) — port the demo tests |
| Should | `replay` CLI: re-run rules over stored events (for tuning and regression) |
| **DoD** | worker crash mid-batch → no lost events (ack after processing), no duplicate alerts |

### Week 17 — More rules, generator, evaluation
| | Item |
|---|---|
| Must | R3 repeated authz failures (distinct-resource ratio) · R4 suspicious UA (signatures, empty, rotation) |
| Must | Sample generator: benign personas + attack scenarios with hidden labels; deterministic seeds; CLI to post to the API |
| Must | **Evaluation harness**: precision/recall per rule against labels; `docs/detection-tuning.md` with the numbers and the thresholds you chose |
| Should | R5 error-rate spike per endpoint (5xx burst) · R6 impossible travel using synthetic geo field · R7 new-UA-for-known-user |
| Could | Simple per-IP baseline (EWMA) instead of fixed thresholds for R2 |
| **DoD** | ≥ 0.9 recall on scenarios, ≤ 2 false positives per 10k benign events, documented |

### Week 18 — Alerts dashboard & analyst workflow
| | Item |
|---|---|
| Must | Alerts API: filter (severity, status, rule, tag, time), sort, cursor pagination; status transitions (new → triaged → investigating → resolved / false_positive) with audit; assign to analyst |
| Must | Analyst notes (append-only); bulk status update; MITRE technique display with link |
| Must | CSV/JSON export (streaming), export excludes any re-identifying field; export is audited |
| Must | UI: small React (reuse B's setup) or HTMX pages — alerts table, detail with evidence timeline, notes |
| **DoD** | an analyst can triage a scenario end-to-end and export the report |

### Week 19 — Retention, privacy, operations
| | Item |
|---|---|
| Must | Retention job: raw events > N days deleted, hourly aggregates kept; alerts kept longer; documented data map (what, why, how long) |
| Must | Pseudonymization key rotation (key id on rows; correlation naturally expires across rotations) |
| Must | Health endpoints, metrics (events/s, queue lag, alerts by rule, rule latency), Compose, CI (same standard as B), load test of ingestion |
| Should | Postgres partitioning by day for events (drop partition = instant retention) |
| **DoD** | retention verified by test; Grafana panel for queue lag |

### Week 20 — Integration with opsdesk & write-up
| | Item |
|---|---|
| Must | opsdesk → monitor: ship access/auth events via opsdesk's outbound webhook (or a log shipper sidecar) |
| Must | monitor → opsdesk: high-severity alert creates a ticket through opsdesk's inbound webhook with `Idempotency-Key = alert.id` — **the loop closes** |
| Must | `docs/boundary.md`: what you test, what you never test, why; README, video, ADR index, `v1.0.0` |
| Should | Write-up: "Detection engineering on synthetic traffic: how I measured my own false positives" |
| **DoD** | one demo video: attack generator → monitor alert → opsdesk ticket → technician resolves |

---

## 4. Global Definition of Done (every milestone)

- [ ] CI green: ruff, mypy, tests (coverage ≥ 85 % on `services/` and rules), schema diff, security scanners
- [ ] New endpoints have role × tenant permission tests **and** an N+1 query-count test
- [ ] Migrations are reversible and run from zero in CI
- [ ] OpenAPI updated; generated client regenerated (B)
- [ ] ADR written for any decision you'd have to explain in an interview
- [ ] README / runbooks updated; `make demo` still works
- [ ] No High findings from `pip-audit` / `bandit`; secrets only via env

## 5. Portfolio and interview framing

**Repository first impression (30 seconds):** architecture diagram, CI badge, 2-minute video, "Security" section, ADR index. Pin both repos.

**STAR stories to prepare (write them in `docs/interview-notes.md`, private if you like):**

| Track | Story anchored in the projects |
|---|---|
| Backend | idempotency under concurrent retries (unique constraint race → savepoint); SLA clock with pause; per-tenant sequence with row locks; N+1 hunt with query-count tests |
| Junior platform / DevOps | Compose → CI → image hardening; dead-letter + requeue; restore drill with measured RTO; health-check semantics (live vs ready) |
| Technical support engineering | correlation IDs shown to users and searchable in logs; audit log for "who changed this ticket"; runbooks; the SLA/escalation model from the technician's seat |
| Junior AppSec / DevSecOps | threat model → controls → tests traceability; SSRF guard on webhooks; upload validation bypass tests; ZAP scan triage; detection rules with measured precision/recall and honest MITRE mapping |

**Talking-point rule:** every claim on your CV maps to a file path. "Implemented rate limiting" → `apps/accounts/throttles.py` + `tests/test_throttles.py` + ADR.

## 6. Time traps and cut lines

- **React polish** — cap at week 11; functional beats pretty.
- **Observability stack** — Prometheus + one dashboard is enough; skip tracing unless ahead.
- **KB suggestions, CSV import, business-hours SLA, RLS, partitioning** — all Should/Could; they are differentiators only if the Musts are flawless.
- **Do not start C before B's SECURITY_CASE_STUDY.md is written.** The case study is worth more than any single feature.
- If a week overruns by > 2 days, cut scope — never cut tests or the Friday ritual.

## 7. Immediate next steps (Week 5, day 1–2)

1. Create the `secure-opsdesk` repo with the layout in §1; copy the demo's `services.py`, `permissions.py`, `middleware.py`, and tests as the seed of `apps/tickets` and `apps/ops` (they are already correct for Postgres).
2. Write ADR-001/002/003 before any feature code — 30 minutes each.
3. Get `docker compose up` → `/health/ready` green with real Postgres + Redis, then CI green on an empty test.
