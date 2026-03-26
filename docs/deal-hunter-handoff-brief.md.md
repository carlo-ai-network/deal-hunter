# Deal Hunter — Implementation Handoff Brief

**Purpose:** Start-here guide for new implementation chats. Read this first, then pull in relevant spec sections as needed.

---

## 1. Project Summary

Deal Hunter is a personal deal-monitoring tool that watches hardware marketplaces, deal sites, and community forums for items on a configurable wishlist. It scores listings with a local LLM (Qwen 2.5 14B), applies rule-based pre-filtering to reduce noise, and sends push notifications via ntfy.sh when deals meet alert thresholds. The system also tracks eBay auctions through a dedicated state machine and maintains price history for trend analysis. It's built for one user — me — running across a MacBook Air and a Windows gaming PC connected over Tailscale.

---

## 2. System Architecture at a Glance

The system is split across two machines with a strict ownership boundary: the MacBook exclusively owns the SQLite database; the PC never touches it. The MacBook runs a single FastAPI orchestrator process that handles source polling, pre-filtering, scheduling, notifications, auction monitoring, maintenance, and the dashboard. The PC runs a scoring worker — a pure HTTP client loop with no server and no open port — that pulls scoring batches from the orchestrator's internal API over Tailscale, sends them to Qwen via Ollama, and pushes results back. The PC sleeps when idle and wakes via Wake-on-LAN when scoring work accumulates. All inter-machine communication is HTTP over Tailscale: MacBook at 100.120.55.116 (orchestrator on port 9825, WOL relay on 9824), PC at 100.75.153.116 (Ollama on 11434). The internal API is secured with a shared secret in the `X-Internal-Secret` header.

---

## 3. Technology Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.13 |
| Web framework | FastAPI |
| Database | SQLite (WAL mode) |
| HTTP client | httpx (AsyncClient) |
| LLM | Qwen 2.5 14B via Ollama |
| Notifications | ntfy.sh |
| Dashboard | React SPA (HTM + CDN imports, no build tooling) |
| HTML parsing | beautifulsoup4 + lxml |
| RSS parsing | feedparser |
| Logging | structlog (JSON, secret redaction) |
| Config | YAML + .env (python-dotenv) |
| Validation | Pydantic v2 (AwareDatetime) |
| Service management | launchd (macOS), NSSM (Windows) |
| Testing | pytest, pytest-asyncio, freezegun |
| Networking | Tailscale mesh VPN |

---

## 4. Key Design Decisions

These are non-negotiable architectural choices. Every implementation chat needs to respect them.

1. **All database writes go through DatabaseWriter** — a single async queue processing writes sequentially. SQLite can't handle concurrent writers. Never bypass it.

2. **Scoring worker is a pure HTTP client** — no database access, no server, no open port. It pulls work from the orchestrator's internal API and pushes results back. The orchestrator owns all state.

3. **AwareDatetime enforced on all Pydantic timestamp fields** — naive datetimes are rejected at validation. Adapters convert to UTC at the boundary using `to_utc()`. The API returns UTC; the browser converts for display.

4. **Adapters implement only the happy path** — the `safe_search()` wrapper on the base class catches all exceptions. Adapters never crash the orchestrator. Errors are captured inside `AdapterResult`, never raised.

5. **Single `price` field with `price_confidence`** — not separate columns for sticker/extracted/parsed prices. The pre-filter applies confidence-aware buffers (exact: 1.0×, parsed: 1.15×, extracted: 1.3×, unknown: pass-through).

6. **`keyword_groups` uses AND/OR logic** — OR between groups, AND within each group's `terms` list. This is how the wishlist query model works; adapters must construct queries accordingly.

7. **All SQL centralized in `database/queries.py`** — no inline SQL anywhere in business logic. Every query is a named function or constant imported from one place.

8. **Copy-on-write for per-row thresholds** — auction watch thresholds are copied from config at creation time, then independent. Config changes don't retroactively update existing rows.

9. **Schedule state lives in the database** — `source_health.next_scheduled_check` survives restarts. The scheduler uses a next-check-at model, not cron.

10. **Shared registries for multi-consumer resources** — three Reddit adapters share one rate limiter and one auth token via `SharedRateLimiterRegistry` and `SharedAuthRegistry`, keyed by `account_key` in config.

---

## 5. What's Been Produced

| Document | Description |
|----------|-------------|
| **V1 Technical Specification** | Master reference: 9 sections covering database, adapters, scheduling, scoring, auctions, notifications, dashboard, project structure, milestones. Includes code patterns, data models, and testing approach per section. |
| **Database Schema Reference (v1 final)** | Authoritative table definitions: 10 tables with columns, types, constraints, indexes, state transitions, entity relationships. The spec references this — it doesn't duplicate it. |
| **config.yaml** | Committed configuration with all tunable parameters: orchestrator settings, scoring weights, auction thresholds, notification rules, scraping defaults, per-source config, rate limits. |
| **local.yaml.example** | Machine-specific overrides template: Tailscale IPs, MAC address, database path, port assignments. |
| **.env.example** | Secrets template: eBay API keys, Reddit credentials, ntfy.sh topic, internal API secret. |
| **Implementation Handoff Brief** | This document. |
| **Retrospective & Risk Register** | Corrections made during design, risk assessment, V2 parking lot, reusable patterns. |

---

## 6. Build Order

| # | Milestone | Delivers | Est. Days |
|---|-----------|----------|-----------|
| M0 | Foundation | Schema, config, logging, DatabaseWriter, utilities, test infrastructure | 2–3 |
| M1 | First Adapter E2E | Reddit adapter → pre-filter → basic dashboard with wishlist CRUD | 4–5 |
| M2 | Scheduling & Health | Scheduler loop, source_health tracking, backoff, crash recovery | 3–4 |
| M3 | LLM Scoring | Internal API, scoring worker, Ollama client, prompt, parser, WOL logic | 4–5 |
| M4 | Notifications | ntfy.sh client, 5 alert formatters, suppression rules, retry logic | 2–3 |
| M5 | Tier A Adapters | Amazon (scraper), Slickdeals (RSS), Newegg (scraper) | 4–5 |
| M6 | Tier B Adapters | Microcenter, Best Buy (scrapers), eBay (API or scraper) | 4–5 |
| M7 | Auction Machine | Full auction lifecycle: auto-watch, state transitions, alerts, resolution | 4–5 |
| M8 | Price History | Price tracking, dedup, market summaries, Recharts charts, SSE | 3–4 |
| M9 | Hardening | Memory profiling, service configs, log rotation, dogfooding, bug fixes | 5–7 |

**Total: 35–46 days part-time.** Each milestone is a working, testable increment. Dependencies flow strictly downward.

---

## 7. How to Use These Documents

**Architecture questions** (why is X designed this way? how do components interact?) → V1 Technical Specification, relevant section.

**Table or column questions** (what type is this field? what's the constraint? what are the valid states?) → Database Schema Reference. This is the authoritative source. If the spec and schema disagree, the schema wins.

**Config questions** (what's the default poll interval? what are the urgency multipliers?) → `config.yaml` for committed defaults, `local.yaml.example` for machine-specific values, `.env.example` for secrets.

**Starting implementation of a milestone** → Hand the relevant spec section(s) + the database schema doc + this handoff brief to a Sonnet chat. For example, M3 (LLM Scoring) needs Spec Sections 3 and 4 plus the schema doc.

**Design changes during implementation** → Bring the question to an Opus chat with the spec section for context. Update the spec before implementing the change, so the spec stays authoritative.

---

## 8. First Steps (M0)

These are the literal first things to do before any business logic:

1. **Create the repo** — `deal-hunter/` with the directory structure from Spec Section 8.

2. **Set up the virtualenv** — `python3.13 -m venv .venv && source .venv/bin/activate`.

3. **Install core dependencies** — `pip install fastapi uvicorn httpx aiosqlite pydantic python-dotenv pyyaml structlog beautifulsoup4 lxml feedparser pytest pytest-asyncio freezegun`.

4. **Copy config templates** — `config.yaml` into repo root (committed), `local.yaml.example` → `local.yaml` (fill in real values, gitignored), `.env.example` → `.env` (fill in secrets, gitignored).

5. **Implement config loading** — `src/shared/config.py`: load `config.yaml`, deep-merge `local.yaml` if present, load `.env` into `os.environ`.

6. **Implement logging** — `src/shared/logging.py`: structlog JSON output with secret redaction processor.

7. **Implement database initialization** — `src/database/schema.py`: all 10 `CREATE TABLE IF NOT EXISTS` statements, `PRAGMA journal_mode=WAL`, `PRAGMA foreign_keys=ON`.

8. **Implement DatabaseWriter and DatabaseReader** — `src/database/writer.py` and `reader.py` per the patterns in Spec Section 1.

9. **Implement shared utilities** — `safe_math.py` (None-safe arithmetic), `time_utils.py` (`to_utc()`, `utc_now()`), `url_utils.py` (`normalize_url()`).

10. **Write tests** — `conftest.py` with in-memory test database fixture. Test that all tables create, config loads, logging works, DatabaseWriter processes writes sequentially.

11. **Verify it runs** — `python -m src.orchestrator.main` starts FastAPI on port 9825 and responds to `GET /api/health`.

Once M0 passes, you're ready for M1: the Reddit adapter end-to-end.
