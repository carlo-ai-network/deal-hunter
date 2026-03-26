# Deal Hunter — V1 Technical Specification

**Version:** 1.0
**Status:** Final Design — Ready for Implementation
**Stack:** Python 3.13 · FastAPI · SQLite (WAL) · Qwen 2.5 14B via Ollama · ntfy.sh

---

## Table of Contents

1. [Database Schema](#1-database-schema)
2. [Source Adapter Contracts](#2-source-adapter-contracts)
3. [Scheduling & Orchestration](#3-scheduling--orchestration)
4. [Scoring Pipeline](#4-scoring-pipeline)
5. [Auction State Machine](#5-auction-state-machine)
6. [Notification System](#6-notification-system)
7. [Dashboard & API](#7-dashboard--api)
8. [Project Structure](#8-project-structure)
9. [Development Milestones](#9-development-milestones)

**Companion documents:**
- `deal-hunter-database-schema-v1-final.md` — Full table definitions, indexes, constraints, state transitions
- `config.yaml` — Committed configuration with all tunable parameters
- `local.yaml.example` — Machine-specific overrides template
- `.env.example` — Secrets template

---

## Infrastructure

### Hardware

| Machine | Role | Specs | Power State |
|---------|------|-------|-------------|
| MacBook Air M1 | Orchestrator, adapters, pre-filter, scheduler, notifications, auction monitor, dashboard | 8GB unified memory | Always on |
| Windows Gaming PC | Scoring worker (Ollama + Qwen 2.5 14B) | Ryzen 9800X3D, RX 7900 XTX 24GB VRAM, 32GB RAM | Sleeps when idle, wakes via WOL |
| iPhone | iOS Shortcuts for capture, Safari dashboards | — | — |

### Network

All inter-machine communication runs over Tailscale mesh VPN.

| Service | Host | Tailscale IP | Port |
|---------|------|--------------|------|
| Headless Inbox API | PC | 100.75.153.116 | 9823 |
| WOL Relay | MacBook | 100.120.55.116 | 9824 |
| Deal Hunter Orchestrator | MacBook | 100.120.55.116 | 9825 |
| Ollama | PC | 100.75.153.116 | 11434 |

### Technology Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| Language | Python 3.13 | |
| Web framework | FastAPI | Orchestrator + dashboard + internal API |
| Database | SQLite (WAL mode) | Single writer, concurrent readers |
| HTTP client | httpx (AsyncClient) | All external + inter-machine HTTP |
| LLM | Qwen 2.5 14B via Ollama | Local inference on PC GPU |
| Notifications | ntfy.sh | HTTP POST, priority mapping, action buttons |
| Dashboard | React SPA (HTM + CDN imports) | No build tooling |
| HTML parsing | beautifulsoup4 + lxml | Scraper adapters |
| RSS parsing | feedparser | Slickdeals adapter |
| Logging | structlog (JSON) | Secret redaction processor |
| Config | YAML + .env (python-dotenv) | Layered: config.yaml → local.yaml → .env |
| Service management | launchd (macOS), NSSM (Windows) | Auto-start, restart on crash |
| Testing | pytest, pytest-asyncio, freezegun | |
| Validation | Pydantic v2 | `AwareDatetime` for all timestamp fields |

### Architecture Overview

The system is split across two machines with a strict ownership boundary: the MacBook exclusively owns the SQLite database. The PC never touches the database directly.

**MacBook (always on):** A single FastAPI process (the orchestrator) runs four concurrent background tasks:

1. **Scheduler** — Polls sources on configurable intervals adjusted by urgency tier multipliers.
2. **Notification worker** — Delivers alerts via ntfy.sh with suppression rules (quiet hours, cooldowns, dedup).
3. **Auction monitor** — Dedicated loop for the eBay auction state machine, separate from the main scheduler.
4. **Maintenance worker** — Cleanup (raw_listings retention, scoring_queue pruning), daily market summary generation, pause_until reactivation checks.

**PC (sleeps, wakes via WOL):** A scoring worker process — a pure HTTP client loop that:

1. Calls the orchestrator's internal API to pick up scoring batches.
2. Sends listings to Qwen via Ollama for LLM scoring.
3. Submits results back to the orchestrator API.
4. Reports health periodically.

The worker has no server, no open port, no database access. It is a client that pulls work and pushes results.

**Internal API endpoints** (secured with shared secret in `X-Internal-Secret` header):

| Endpoint | Purpose |
|----------|---------|
| `POST /api/internal/scoring/pick-batch` | Atomically claim pending scoring queue items |
| `POST /api/internal/scoring/heartbeat` | Keep-alive during batch processing |
| `POST /api/internal/scoring/submit` | Submit one scoring result |
| `POST /api/internal/scoring/report-failure` | Report a scoring failure |
| `POST /api/internal/scoring/worker-health` | Periodic health report |

---

## 1. Database Schema

See **Database Schema Reference (v1 final)** for complete table definitions, column types, constraints, indexes, state transitions, and entity relationships.

### Design Patterns

**Single-writer architecture.** All database writes are serialized through a `DatabaseWriter` — an async queue that processes write operations one at a time. This eliminates SQLite contention without requiring connection pooling or WAL checkpointing workarounds. A separate `DatabaseReader` allows concurrent read access from any task.

```python
class DatabaseWriter:
    """Serializes all writes through a single async channel."""

    def __init__(self, db_path: str):
        self._queue: asyncio.Queue[WriteOperation] = asyncio.Queue()
        self._db_path = db_path
        self._running = False

    async def start(self):
        """Start the write loop. Called once at orchestrator startup."""
        self._running = True
        asyncio.create_task(self._process_loop())

    async def execute(self, query: str, params: tuple = ()) -> int:
        """Queue a write and await its completion. Returns lastrowid."""
        op = WriteOperation(query=query, params=params)
        await self._queue.put(op)
        return await op.result

    async def _process_loop(self):
        """Single connection, single loop, processes writes sequentially."""
        async with aiosqlite.connect(self._db_path) as db:
            await db.execute("PRAGMA journal_mode=WAL")
            await db.execute("PRAGMA foreign_keys=ON")
            while self._running or not self._queue.empty():
                op = await self._queue.get()
                try:
                    cursor = await db.execute(op.query, op.params)
                    await db.commit()
                    op.result.set_result(cursor.lastrowid)
                except Exception as e:
                    op.result.set_exception(e)
```

**Soft deletes.** Tables referenced by polymorphic FKs (`raw_listings`, `scored_deals`, `auction_watches`) use `deleted_at` rather than hard deletes. Application queries always filter `WHERE deleted_at IS NULL`. Consider creating database views that apply this filter automatically.

**JSON columns** store flexible, type-specific data: `keyword_groups`, `match_criteria`, `raw_data`, `context`, `error_detail`, `risk_flags`. Queried in Python, not SQL.

**Copy-on-write thresholds.** Per-row values (like auction `approaching_threshold_min`) are initialized from `config.yaml` tier defaults at creation time. Once copied, the row's values are independent — config changes don't retroactively update existing rows.

**Centralized SQL.** All queries live in `src/database/queries.py` (or a `queries/` subdirectory). No inline SQL in business logic. Aggregation queries use LEFT JOINs and CTEs to avoid N+1 patterns.

**NULL arithmetic safety.** All SQL involving nullable numeric columns uses `COALESCE` or explicit `IS NULL` checks. Python arithmetic on nullable fields uses `safe_math.py` utilities that guard against `None`.

### Key Tables Summary

| Table | Purpose | Volume |
|-------|---------|--------|
| `wishlist_items` | What you're hunting for | Low (~10-50 rows) |
| `raw_listings` | Every listing found by adapters | High (pruned at 30 days) |
| `scored_deals` | LLM scoring results and audit trail | Medium |
| `price_history` | Time-series price observations | Medium (append-only, kept forever) |
| `market_summary` | Daily price rollup per item | Low (1 row/item/day) |
| `auction_watches` | eBay auction state machine | Medium |
| `alerts` | Notification log with delivery tracking | Medium (kept forever) |
| `scoring_queue` | Work queue bridging MacBook → PC | Transient (pruned at 7 days) |
| `source_health` | Per-adapter operational status | Static (1 row per source) |
| `item_errors` | Per-item per-source error tracking | Low (deduplicated) |

---

## 2. Source Adapter Contracts

### Abstract Base Class

Every source adapter inherits from `SourceAdapter`. The base class provides the safety wrapper, rate limiting, URL normalization, and error handling. Adapters implement only the happy path.

```python
class SourceAdapter(ABC):
    """Base class for all source adapters."""

    # Capability flags — override in subclasses
    supports_auction_tracking: bool = False
    supports_detail_fetch: bool = False
    supports_price_filter: bool = False
    requires_auth: bool = False
    expect_results: bool = True  # False for sources that legitimately return empty

    def __init__(self, config: dict, shared_resources: SharedResources):
        """Synchronous setup only. No I/O here."""
        self.config = config
        self.source_name = config["source_name"]
        self.rate_limiter = shared_resources.get_rate_limiter(self.source_name)
        self.log = structlog.get_logger().bind(source=self.source_name)

    async def initialize(self) -> None:
        """Async setup: OAuth token refresh, session creation, etc."""
        pass

    async def close(self) -> None:
        """Teardown: close HTTP sessions, release resources."""
        pass

    # --- Required methods ---
    @abstractmethod
    async def _do_search(self, query: WishlistQuery) -> AdapterResult:
        """Implement the actual search. Only the happy path."""
        ...

    @abstractmethod
    async def health_check(self) -> HealthCheckResult:
        """Verify adapter can reach its source."""
        ...

    # --- Optional capabilities ---
    async def get_listing_detail(self, external_id: str) -> RawListing | None:
        raise NotImplementedError

    async def check_auction_status(self, external_id: str) -> AuctionStatus | None:
        raise NotImplementedError

    async def check_availability(self, external_id: str) -> AvailabilityStatus | None:
        raise NotImplementedError

    async def refresh_auth(self) -> None:
        raise NotImplementedError
```

### safe_search Wrapper

The base class wraps `_do_search` with `safe_search()`, which catches all exceptions so adapters never crash the orchestrator:

```python
async def safe_search(self, query: WishlistQuery) -> AdapterResult:
    """Call _do_search with full error handling. Adapters never override this."""
    start = time.monotonic()
    try:
        await self.rate_limiter.acquire()
        result = await self._do_search(query)
        result.response_time_ms = int((time.monotonic() - start) * 1000)
        self._validate_results(result)
        self._check_deprecation_headers(result)
        return result
    except Exception as e:
        return AdapterResult(
            source=self.source_name,
            listings=[],
            error=AdapterError(
                message=str(e),
                error_type=classify_error(e),
                is_transient=is_transient_error(e),
            ),
            response_time_ms=int((time.monotonic() - start) * 1000),
        )
```

### Input Contract: WishlistQuery

Built from `wishlist_items` row but only includes what adapters need. Does NOT include `match_criteria` or `urgency_tier` — those are for the scoring pipeline, not the adapter.

```python
class KeywordGroup(BaseModel):
    """One group of terms. All terms within a group must match (AND).
    Multiple groups are combined with OR."""
    terms: list[str]

class WishlistQuery(BaseModel):
    wishlist_item_id: int
    keyword_groups: list[KeywordGroup]
    negative_keywords: list[str] = []
    target_price: Decimal | None = None
    max_price: Decimal | None = None
    condition_minimum: str | None = None
    include_auctions: bool = True
    include_buy_now: bool = True
    search_since: AwareDatetime | None = None  # With 2-minute overlap buffer applied
```

The `search_since` value is computed by the scheduler as `last_checked_at - 2 minutes`. The 2-minute buffer prevents missed listings at poll boundaries. Adapters that support time-based filtering use this value; others ignore it.

### Output Contract: RawListing

A single normalized listing from any source — eBay JSON, Reddit post, scraped HTML, or RSS entry.

```python
class RawListing(BaseModel):
    source: str
    external_id: str
    url: str                          # Normalized, tracking params stripped
    title: str
    description: str | None = None
    price: Decimal | None = None      # Best available price
    price_confidence: Literal["exact", "parsed", "extracted", "unknown"] = "unknown"
    shipping_cost: Decimal | None = None
    estimated_tax: Decimal | None = None
    estimated_fees: Decimal | None = None
    total_estimated_cost: Decimal | None = None
    currency: str = "USD"
    condition: str | None = None      # Normalized to enum values
    seller_name: str | None = None
    seller_rating: Decimal | None = None
    listing_type: Literal["buy_now", "auction", "classified", "deal_post"]
    auction_end_time: AwareDatetime | None = None
    listing_created_at: AwareDatetime | None = None
    raw_data: dict                    # Preserved for debugging (trimmed, not full pages)
```

Key design decisions:

- **Single `price` field** with `price_confidence` rather than separate columns for sticker/extracted/parsed prices. The pre-filter uses `price_confidence` to apply appropriate buffers.
- **URL normalization** is built into the base class. A `normalize_url()` utility strips UTM parameters, affiliate tags, and other tracking noise. Every adapter calls it before returning listings.
- **`AwareDatetime`** from Pydantic v2 on all timestamp fields. Naive datetimes are rejected at validation time. Adapters call `to_utc()` (base class helper) at the boundary to convert source-local times.
- **`raw_data`** preserves the original response trimmed to relevant sections — not entire HTML pages, but enough to debug parsing issues.

### Output Contract: AdapterResult

Wraps listings with metadata about the search operation itself.

```python
class AdapterError(BaseModel):
    message: str
    error_type: str  # scrape_failed, parse_error, rate_limited, timeout, auth_failed
    is_transient: bool  # Timeouts, 503s = transient. 403s, parse errors = not transient
    http_status: int | None = None

class AdapterResult(BaseModel):
    source: str
    listings: list[RawListing]
    error: AdapterError | None = None
    total_from_source: int = 0       # Pre-filter count (how many the source had)
    failed_parse_count: int = 0      # Items that failed individual parsing
    response_time_ms: int = 0
    rate_limit_remaining: int | None = None
    rate_limit_resets_at: AwareDatetime | None = None
    search_query_used: str | None = None  # For debugging
```

Errors are captured inside the result object, never raised as exceptions. The `is_transient` flag determines backoff behavior — transient errors (timeouts, 503s) need more occurrences (10+) before escalating to "down" status, while non-transient errors (403s, parse failures) escalate faster (3+).

### Error Handling: Three Levels

1. **Total failure** — the entire search crashes. Caught by `safe_search()`, returned as `AdapterResult` with error.
2. **Individual parse failures** — one listing in a batch can't be parsed. Caught per-item inside `_do_search`, counted in `failed_parse_count`. Other listings still returned.
3. **Suspicious data** — results look wrong but didn't error. Caught by `_validate_results()`.

The `_validate_results()` method checks for anomalies:
- All prices are zero or identical
- All titles are identical (suggests scraping a template, not results)
- Expected fields missing at higher rates than normal (`expected_field_rates` per adapter)
- Field health monitoring: if an adapter historically returns `seller_rating` 95% of the time and suddenly returns it 0% of the time, that's a silent failure signal

The `_check_deprecation_headers()` method watches for API sunset warnings (e.g., eBay's `Deprecation` header) and logs them as warnings.

### Rate Limiting

```python
class RateLimiter:
    """Per-adapter rate limiter with asyncio.Lock for concurrent safety."""

    def __init__(self, requests_per_second: float, requests_per_minute: int):
        self._lock = asyncio.Lock()
        self._rps = requests_per_second
        self._rpm = requests_per_minute
        self._request_times: deque[float] = deque()

    async def acquire(self):
        """Block until a request slot is available."""
        async with self._lock:
            now = time.monotonic()
            # Enforce per-second and per-minute limits
            self._cleanup_old(now)
            if len(self._request_times) >= self._rpm:
                wait = self._request_times[0] + 60 - now
                if wait > 0:
                    await asyncio.sleep(wait)
            # Per-second check
            recent = [t for t in self._request_times if now - t < 1.0]
            if len(recent) >= self._rps:
                await asyncio.sleep(1.0 / self._rps)
            self._request_times.append(time.monotonic())
```

**SharedRateLimiterRegistry:** Multiple adapters sharing the same API account (three Reddit adapters) share a single `RateLimiter` instance. The registry maps `account_key` → `RateLimiter`. Config drives this via the `shared_rate_limits` section in `config.yaml`.

**Adaptive pacing:** If rate limit response headers (`X-RateLimit-Remaining`, etc.) show quota running low, the adapter dynamically reduces its request rate without waiting for a 429. Rate limit hits update `source_health` but do NOT increment `consecutive_failures`.

**Scraper jitter:** Scraper adapters add random delay (0.5–2s) between requests to avoid predictable request patterns.

### Scraper Base Class

Scraper adapters (Amazon, Newegg, Microcenter, Best Buy) extend `ScraperAdapter`, which adds:

```python
class ScraperAdapter(SourceAdapter):
    """Base class for HTML-scraping adapters."""

    def __init__(self, config: dict, shared_resources: SharedResources):
        super().__init__(config, shared_resources)
        self._user_agent = self._pick_session_user_agent()  # One UA per session, not per request
        self._min_delay = config.get("min_delay_seconds", 3.0)

    def _pick_session_user_agent(self) -> str:
        """Select one user agent for the entire session from config pool."""
        agents = self.config.get("user_agents", settings.scraping.user_agents)
        return random.choice(agents)

    def _build_headers(self) -> dict:
        """Browser-like headers. Same UA for entire session."""
        return {
            "User-Agent": self._user_agent,
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "Accept-Language": "en-US,en;q=0.9",
            "Accept-Encoding": "gzip, deflate, br",
            "Connection": "keep-alive",
        }
```

User-Agent rotation picks one agent per session (adapter initialization), not per request. The pool of user agents is configured in `config.yaml` under `scraping.user_agents`. Configurable `robots.txt` compliance defaults to false for scrapers — see `scraping.respect_robots_txt_default` in config.

### Authentication

**Secrets management:** `.env` holds actual credentials. `config.yaml` references env var *names* (e.g., `client_id_env: "EBAY_CLIENT_ID"`), never values.

**TokenManager:** Handles OAuth token lifecycle with `asyncio.Lock` for concurrent safety.

```python
class TokenManager:
    """Thread-safe OAuth token manager with auto-refresh."""

    def __init__(self, token_url: str, client_id: str, client_secret: str, scopes: list[str]):
        self._lock = asyncio.Lock()
        self._token: str | None = None
        self._expires_at: float = 0

    async def get_token(self) -> str:
        """Return a valid token, refreshing if expired."""
        async with self._lock:
            if self._token and time.monotonic() < self._expires_at - 60:
                return self._token
            self._token = await self._refresh()
            return self._token
```

**SharedAuthRegistry:** Like rate limiters, multiple adapters sharing an API account share one `TokenManager`. Maps `account_key` → `TokenManager`.

**`_authenticated_get()`:** Base method that retries once on 401 with a freshly refreshed token before giving up.

**Reddit authentication:** Uses raw httpx with `RedditTokenManager` — NOT PRAW. Password grant flow with Script app type. Token refresh on 401 with the same retry-once pattern.

**Log redaction:** A structlog processor scrubs tokens from all log output using both pattern-based matching (bearer tokens, OAuth signatures) and value-based matching (known secret values from config).

### Adapter Registration

Adapters are registered via an explicit dict, not auto-discovery:

```python
ADAPTER_CLASSES: dict[str, type[SourceAdapter]] = {
    "ebay": EbayAdapter,
    "reddit": RedditSubredditAdapter,
    "amazon": AmazonAdapter,
    "slickdeals": SlickdealsAdapter,
    "newegg": NeweggAdapter,
    "microcenter": MicrocenterAdapter,
    "bestbuy": BestBuyAdapter,
}
```

The `create_adapters()` factory reads config, instantiates adapters, and calls `initialize()` concurrently:

```python
async def create_adapters(config: dict) -> dict[str, SourceAdapter]:
    adapters = {}
    for source_name, source_config in config["sources"].items():
        if not source_config.get("enabled", True):
            continue
        cls = ADAPTER_CLASSES[source_config["adapter_class"]]
        adapter = cls(
            config={**source_config, "source_name": source_name},
            shared_resources=shared,
        )
        adapters[source_name] = adapter
    await asyncio.gather(*[a.initialize() for a in adapters.values()])
    return adapters
```

Reusable classes: one `RedditSubredditAdapter` class serves three subreddits (buildapcsales, hardwareswap, appleswap) via different config entries. The subreddit name comes from config, not the class.

A `validate_source_config()` function runs at startup to catch config mistakes (missing env vars, unknown adapter classes, conflicting rate limits) before any network calls.

### V1 Sources

| Source | Adapter Class | Type | Auth | Poll Interval |
|--------|--------------|------|------|---------------|
| eBay | `ebay` | API (or scraper fallback) | OAuth client credentials | 1 hour |
| r/buildapcsales | `reddit` | API | OAuth password grant | 5 min |
| r/hardwareswap | `reddit` | API | OAuth password grant | 10 min |
| r/appleswap | `reddit` | API | OAuth password grant | 10 min |
| Slickdeals | `slickdeals` | RSS | None | 30 min |
| Amazon | `amazon` | Scraper | None | 2 hours |
| Newegg | `newegg` | Scraper | None | 4 hours |
| Microcenter | `microcenter` | Scraper | None | 2 hours |
| Best Buy | `bestbuy` | Scraper | None | 4 hours |

### Testing Strategy

**Fixture-based parser tests.** Save real responses from each source in `tests/fixtures/{source}/`. The base class provides `load_fixture()` to read them and `capture_fixture()` to save new ones during development.

```python
# tests/test_reddit_adapter.py
async def test_parse_buildapcsales_post():
    adapter = create_test_adapter("reddit", subreddit="buildapcsales")
    fixture = load_fixture("reddit/buildapcsales_gpu_listing.json")
    listings = adapter._parse_response(fixture)
    assert len(listings) == 1
    assert listings[0].price == Decimal("549.99")
    assert listings[0].price_confidence == "extracted"
```

**`create_test_adapter()`** helper instantiates an adapter with stub registries (no real auth, no real rate limiter).

**Integration tests** marked with `pytest -m integration`, skipped in CI, run against live APIs with real credentials.

**Pipeline tests** using `make_fake_listing()` and `make_fake_result()` factories to test the flow from raw listing through pre-filter to scoring queue.

**`freezegun`** for time-sensitive tests: auction state transitions, cooldown logic, schedule calculations. Each fixture documents its reference time.

**`expected_field_rates`** per adapter: defines what percentage of listings should have each field populated. Used both in `_validate_results()` at runtime and in tests to verify parser completeness.

---

## 3. Scheduling & Orchestration

### Orchestrator Lifecycle

The orchestrator is a single long-running FastAPI process on the MacBook. On startup:

```python
async def startup():
    # 1. Load config (config.yaml + local.yaml + .env)
    config = load_config()

    # 2. Initialize database (create tables, run migrations)
    db_writer = DatabaseWriter(config.database.path)
    await db_writer.start()
    db_reader = DatabaseReader(config.database.path)

    # 3. Crash recovery
    await recover_stuck_items(db_writer)       # Re-queue stuck 'processing' scoring items
    await recalculate_schedules(db_writer)     # Rebuild next_scheduled_check for all sources
    await reactivate_paused_items(db_writer)   # Check pause_until timestamps

    # 4. Create adapters
    adapters = await create_adapters(config)

    # 5. Start background tasks
    tasks = [
        asyncio.create_task(scheduler_loop(adapters, db_writer, db_reader)),
        asyncio.create_task(notification_worker_loop(db_writer, db_reader)),
        asyncio.create_task(auction_monitor_loop(adapters["ebay"], db_writer, db_reader)),
        asyncio.create_task(maintenance_worker_loop(db_writer, db_reader)),
    ]
```

### Poll Scheduling

Scheduling uses a **next-check-at model** (not cron). Each source has a base `poll_interval_seconds` in config. The effective interval is adjusted by the highest-urgency active wishlist item's urgency multiplier:

```
effective_interval = source.poll_interval_seconds × urgency_multiplier[tier]
```

Urgency multipliers from `config.yaml`:
- `critical: 0.5` — check twice as often
- `high: 0.75`
- `normal: 1.0` — use source interval as-is
- `low: 2.0` — check half as often

The `next_scheduled_check` timestamp is stored in `source_health` in the database, not in memory. This survives process restarts. After each poll, the scheduler calculates and writes the next check time.

Floor and ceiling: `min_poll_interval_seconds: 60` and `max_poll_interval_seconds: 86400` from config.

### Scheduler Loop

```python
async def scheduler_loop(adapters, db_writer, db_reader):
    while not shutting_down:
        # 1. Query sources due for checking
        due_sources = await db_reader.get_due_sources(now=utc_now())

        # 2. For each due source, gather active wishlist items targeting it
        for source_health in due_sources:
            source_name = source_health.source
            adapter = adapters[source_name]
            items = await db_reader.get_active_items_for_source(source_name)

            # 3. Run searches concurrently across sources, sequentially per source
            for item in items:
                query = build_wishlist_query(item)
                result = await adapter.safe_search(query)
                await process_adapter_result(result, item, db_writer)

            # 4. Update source_health and schedule next check
            await update_source_health(source_name, result, db_writer)

        await asyncio.sleep(5)  # Check for due sources every 5 seconds
```

Sources run concurrently via `asyncio.gather` at the source level. Items within a single source run sequentially (they share a rate limiter). This means multiple sources poll in parallel, but an adapter's individual requests are paced.

### WOL (Wake-on-LAN) Decision Logic

The orchestrator decides when to wake the PC for scoring:

```python
async def check_wol_needed(db_reader) -> bool:
    """Determine if the PC needs to be woken for scoring."""
    pending = await db_reader.get_pending_scoring_count()
    oldest_pending = await db_reader.get_oldest_pending_age_seconds()

    if pending >= config.orchestrator.wol.batch_threshold:  # 5+ items queued
        return True
    if oldest_pending and oldest_pending >= config.orchestrator.wol.max_wait_seconds:  # 30 min
        return True
    # Check for urgent items (auctions ending soon)
    if await db_reader.has_urgent_pending_items():
        return True
    return False
```

WOL sequence:
1. Check cooldown (don't send WOL more than once per 5 minutes).
2. Send magic packet via WOL relay on MacBook (port 9824).
3. Wait `pc_wake_delay_seconds` (20s) for initial boot.
4. Poll PC readiness: Tailscale ping → Ollama health endpoint → worker health (from periodic POST reports, NOT by pinging the worker).
5. Timeout after `pc_ready_timeout_seconds` (120s). If PC doesn't come up, create a system alert.

The PC status check chain: the orchestrator never tries to connect to the worker directly. It checks Tailscale reachability, then Ollama's health endpoint, then looks for recent `worker-health` POST reports in its own state. The worker is the one that initiates all connections.

### Concurrency Model

All async operations use standard `asyncio`. No threading except where forced by synchronous libraries.

**Database access:**
- All writes go through `DatabaseWriter` (single async queue → single connection).
- Reads use `DatabaseReader` (can open multiple read connections; SQLite WAL allows concurrent reads).

**HTTP connections:** Conservative pool limits for MacBook's 8GB memory constraint. Per-adapter connection pool caps at 10 max connections. Total memory budget: 200–300MB.

**Memory monitoring:** The orchestrator tracks its own memory usage. At `memory_warning_mb` threshold (400MB from config), it logs a warning and can shed load (skip low-priority polls, delay non-urgent scoring).

### Graceful Shutdown

```python
async def shutdown():
    # 1. Signal all loops to stop
    shutting_down.set()

    # 2. Wait for in-flight operations (30s timeout from config)
    try:
        await asyncio.wait_for(
            asyncio.gather(*background_tasks, return_exceptions=True),
            timeout=config.orchestrator.shutdown_timeout_seconds,
        )
    except asyncio.TimeoutError:
        # 3. Force-cancel remaining tasks
        for task in background_tasks:
            task.cancel()

    # 4. Drain the DatabaseWriter queue (process remaining writes)
    await db_writer.drain()

    # 5. Close all adapter sessions
    await asyncio.gather(*[a.close() for a in adapters.values()])
```

### Crash Recovery

On startup, the orchestrator handles interrupted state:

1. **Stuck scoring items:** Any `scoring_queue` entries with `status = 'processing'` and stale `heartbeat_at` (older than `stale_timeout_min`) get reset to `pending` with incremented `attempts`.
2. **Schedule recalculation:** All `source_health.next_scheduled_check` values are recalculated based on current time and configured intervals. This handles both the "was offline for hours" case and the "config changed while offline" case.
3. **Paused item reactivation:** Any `wishlist_items` with `status = 'paused'` and `pause_until < now` get set back to `active`.

### Testing

- **Schedule calculation tests:** Verify urgency multipliers produce correct intervals. Use `freezegun` to control time.
- **WOL decision tests:** Mock database queries to test threshold logic (batch size, age, urgency).
- **Crash recovery tests:** Set up dirty state (stuck items, stale schedules), run recovery, verify cleanup.
- **Shutdown tests:** Verify write queue drains, connections close, no data loss.

---

## 4. Scoring Pipeline

The scoring pipeline has two phases: a rule-based pre-filter running on the MacBook (cheap, fast) and LLM scoring running on the PC (expensive, slow).

### Phase 1: Pre-Filter (MacBook)

The pre-filter runs immediately after a listing is inserted into `raw_listings`. It applies six checks in order, cheapest first — any failure short-circuits:

```python
class PreFilterResult(BaseModel):
    passed: bool
    reason: str | None = None       # Why rejected (None if passed)
    checks_run: int                  # How many checks ran before decision
    confidence: Literal["high", "low"]  # "low" if borderline price

PREFILTER_CHECKS = [
    check_negative_keywords,     # 1. Instant disqualification terms
    check_keyword_match,         # 2. At least one keyword group must match
    check_price_ceiling,         # 3. Confidence-aware price buffer
    check_condition_minimum,     # 4. Meets minimum condition requirement
    check_listing_status,        # 5. Not ended/sold/expired
    check_duplicate,             # 6. Not already seen (source + external_id + wishlist_item)
]
```

**Confidence-aware price buffer:** The pre-filter applies different price ceilings based on `price_confidence`:

| Confidence | Buffer | Effect |
|------------|--------|--------|
| `exact` | 1.0× | Trust it — reject if `price > max_price` |
| `parsed` | 1.15× | 15% buffer — reject if `price > max_price × 1.15` |
| `extracted` | 1.3× | 30% buffer — reject if `price > max_price × 1.3` |
| `unknown` | 999999 | No price — always let through for LLM to evaluate |

Borderline prices (between `target_price` and `max_price`) pass with `confidence: "low"`, which gives them lower priority in the scoring queue.

**Condition ranking** for the condition minimum check:

```python
CONDITION_RANK = {
    "new": 6, "open_box": 5, "refurbished": 4,
    "used_like_new": 3, "used_good": 2, "used_fair": 1,
}
```

A listing passes if its condition rank is ≥ the wishlist item's minimum condition rank. `condition_minimum: "any"` skips this check.

After pre-filter, passing listings get a row in `scoring_queue` with `status: "pending"`. The `priority` field is set based on urgency tier and pre-filter confidence:

```python
def calculate_queue_priority(item: WishlistItem, prefilter: PreFilterResult) -> int:
    """Lower number = higher priority."""
    base = {"critical": 0, "high": 100, "normal": 200, "low": 300}[item.urgency_tier]
    if prefilter.confidence == "low":
        base += 50  # Borderline prices get deprioritized
    # Auction urgency boost
    if listing.listing_type == "auction" and listing.auction_end_time:
        minutes_left = (listing.auction_end_time - utc_now()).total_seconds() / 60
        if minutes_left < 60:
            base -= 100  # Urgent auctions jump the queue
    return base
```

### Phase 2: LLM Scoring (PC)

The scoring worker is a loop that pulls work from the orchestrator and sends it to Ollama:

```python
async def scoring_worker_loop():
    while running:
        # 1. Pick up a batch
        batch = await pick_batch(
            url=f"http://{orchestrator_ip}:9825/api/internal/scoring/pick-batch",
            batch_size=config.scoring.batch_size,  # 10
        )
        if not batch:
            await asyncio.sleep(10)
            continue

        # 2. Process each item
        for item in batch:
            await send_heartbeat(item.queue_id)
            result = await score_with_ollama(item)
            if result.success:
                await submit_result(item.queue_id, result)
            else:
                await report_failure(item.queue_id, result.error)

        # 3. Report health
        await report_worker_health()
```

**Prompt structure:** The scoring prompt is versioned in `prompts/scoring_v1.0.txt` and tracked in `scored_deals.llm_prompt_version`.

The prompt includes:
1. System context — what Deal Hunter is, what the scoring task is.
2. The wishlist item's `name`, `match_criteria`, `target_price`, `max_price`, `condition_minimum`.
3. The listing data — title, description, price, condition, seller info.
4. Three concrete examples: a strong match (score ~85), a partial match (score ~55), and a poor match (score ~25).
5. Response format instruction: "Respond with ONLY this JSON, no other text. Do NOT nest objects."

Expected response format:

```json
{
    "spec_score": 82,
    "condition_score": 90,
    "value_score": 75,
    "risk_flags": ["new_seller"],
    "reasoning": "DDR5 6000MHz CL30 64GB kit matches spec exactly. New seller with limited feedback history."
}
```

**Risk flags** are drawn from a closed set:
- `new_seller` — seller has little/no feedback history
- `no_returns` — no return policy stated
- `price_too_good` — price suspiciously below market
- `vague_description` — insufficient detail to verify specs
- `wrong_item_possible` — listing could be a different item
- `compatibility_uncertain` — unclear if compatible with specified system
- `condition_mismatch` — stated condition doesn't match description

**Model configuration:** `qwen2.5:14b` at temperature 0 for deterministic output. Ollama endpoint on PC at port 11434.

### Score Parsing

The LLM response is parsed with a brace-depth JSON extractor — not regex. This handles cases where the LLM wraps its response in markdown code fences or adds preamble text:

```python
def extract_json_from_response(raw: str) -> dict | None:
    """Find the first complete JSON object using brace-depth tracking."""
    depth = 0
    start = None
    for i, char in enumerate(raw):
        if char == '{':
            if depth == 0:
                start = i
            depth += 1
        elif char == '}':
            depth -= 1
            if depth == 0 and start is not None:
                try:
                    return json.loads(raw[start:i+1])
                except json.JSONDecodeError:
                    continue
    return None
```

**Pydantic validation** with `mode="before"` validators handles LLM output quirks:

```python
class ScoringResponse(BaseModel):
    spec_score: int = Field(ge=0, le=100)
    condition_score: int = Field(ge=0, le=100)
    value_score: int = Field(ge=0, le=100)
    risk_flags: list[str] = []
    reasoning: str = ""

    @field_validator("spec_score", "condition_score", "value_score", mode="before")
    @classmethod
    def coerce_score(cls, v):
        """Handle floats, strings, decimals, out-of-range values."""
        if isinstance(v, str):
            v = float(v)
        v = int(round(float(v)))
        return max(0, min(100, v))

    @field_validator("risk_flags", mode="before")
    @classmethod
    def validate_flags(cls, v):
        """Filter to known flags only."""
        known = {"new_seller", "no_returns", "price_too_good", "vague_description",
                 "wrong_item_possible", "compatibility_uncertain", "condition_mismatch"}
        if isinstance(v, list):
            return [f for f in v if f in known]
        return []
```

**Partial parse:** If some fields are valid but others aren't, use the valid ones with defaults for missing. The `parse_status` field in `scored_deals` records `success`, `partial`, or `failed`. The `llm_raw_response` is always preserved for debugging.

### Final Score Calculation

```python
def calculate_final_score(
    spec: int, condition: int, value: int,
    weights: dict, listing: RawListing, item: WishlistItem
) -> Decimal:
    """Weighted composite × urgency multiplier."""
    base = (
        spec * weights["spec"] +          # 0.4
        condition * weights["condition"] + # 0.3
        value * weights["value"]           # 0.3
    )

    # Urgency multiplier for time-sensitive listings
    multiplier = 1.0
    if listing.listing_type == "auction" and listing.auction_end_time:
        minutes_left = (listing.auction_end_time - utc_now()).total_seconds() / 60
        if minutes_left < 5:
            multiplier = 2.0
        elif minutes_left < 30:
            multiplier = 1.5
        elif minutes_left < 60:
            multiplier = 1.25
    elif listing.listing_type == "deal_post":
        multiplier = 1.15

    return Decimal(str(round(base * multiplier, 2)))
```

Weights are configurable in `config.yaml` under `scoring.weights`.

### Alert Tier Determination

After scoring, the alert tier is determined with hard overrides:

```python
def determine_alert_tier(score: Decimal, scoring: ScoringResponse, item: WishlistItem) -> str:
    """Map score to alert tier with safety overrides."""
    # Hard overrides (checked first)
    if scoring.spec_score < 30:
        return "none"  # Wrong item — never alert
    if "condition_mismatch" in scoring.risk_flags and item.condition_minimum in ("new", "open_box"):
        return min_tier("low", score_to_tier(score))  # Cap at low
    if len(scoring.risk_flags) >= 2:
        return min_tier("low", score_to_tier(score))  # Multiple risks — cap at low
    if "price_too_good" in scoring.risk_flags:
        return min_tier("normal", score_to_tier(score))  # Suspicious price — cap at normal

    # Standard thresholds (from config.yaml scoring.alert_thresholds)
    if score >= 80:
        return "high"
    if score >= 60:
        return "normal"
    if score >= 40:
        return "low"
    return "none"
```

### Feedback Loop

The `user_feedback` field on `scored_deals` accepts: `good_deal`, `bad_deal`, `irrelevant`, `wrong_item`, `purchased`, `missed_it`. A dashboard endpoint provides scoring accuracy analysis: hit rate by source, false positive rate by category, score distribution charts.

### Testing

- **Pre-filter tests:** Test each check individually with edge cases (zero price, null condition, exact boundary prices). Test short-circuit ordering.
- **Score parsing tests:** Test the brace-depth extractor with valid JSON, wrapped JSON, malformed JSON, partial responses. Test Pydantic coercion with floats, strings, out-of-range values.
- **Alert tier tests:** Test all override paths. Use `make_fake_listing()` with various risk flag combinations.
- **Integration test:** Full pipeline from raw listing → pre-filter → queue → score → alert tier. Requires Ollama running locally.

---

## 5. Auction State Machine

The auction monitor is a dedicated background loop, separate from the main scheduler, that tracks eBay auctions through their lifecycle.

### State Diagram

```
discovered ──→ watching ──→ approaching ──→ alert_window ──→ resolved
                  │              │    ↑           │    ↑
                  │              │    └───────────┘    │
                  │              │   (auto-extend)     │
                  ├──→ outpriced ←──┘                  │
                  └──→ resolved                        └──→ resolved
```

See **Database Schema Reference** for the full state transition table and `auction_watches` column definitions.

### Auto-Watch Creation

When the scoring pipeline processes an eBay auction listing and `final_score >= auctions.min_watch_score` (40), an `auction_watches` row is automatically created:

```python
async def maybe_create_auction_watch(scored_deal: ScoredDeal, listing: RawListing, item: WishlistItem):
    if listing.listing_type != "auction":
        return
    if scored_deal.final_score < config.auctions.min_watch_score:
        return

    # Copy-on-write thresholds from urgency tier
    tier_config = config.auctions.tier_defaults[item.urgency_tier]

    await db_writer.execute(INSERT_AUCTION_WATCH, {
        "raw_listing_id": listing.id,
        "wishlist_item_id": item.id,
        "ebay_item_id": listing.external_id,
        "state": "discovered",
        "approaching_threshold_min": tier_config["approaching_threshold_min"],
        "alert_threshold_min": tier_config["alert_threshold_min"],
        # ... other fields from listing
    })
```

### Monitor Loop

The auction monitor runs independently from the scheduler with its own tick rate:

```python
async def auction_monitor_loop(ebay_adapter, db_writer, db_reader):
    while not shutting_down:
        # Priority-ordered: alert_window first, then approaching, then watching, then discovered
        watches = await db_reader.get_active_auction_watches()

        for watch in watches:
            if watch.next_poll_at and watch.next_poll_at > utc_now():
                continue  # Not due yet

            await process_auction_watch(watch, ebay_adapter, db_writer)

        await asyncio.sleep(5)
```

All auction watches share the eBay rate limiter, so they're processed **sequentially** (not concurrently). Priority ordering ensures that auctions in `alert_window` state are checked before those in `watching`.

### Poll Processing

Each poll follows this logic:

```python
async def process_auction_watch(watch, adapter, db_writer):
    # 1. Fetch current auction status
    status = await adapter.check_auction_status(watch.ebay_item_id)
    if not status:
        return  # Transient failure — try again next poll

    # 2. Capture old values BEFORE any database write
    old_bid = watch.current_bid
    old_end_time = watch.auction_end_time

    # 3. Check terminal conditions FIRST
    if status.ended:
        await resolve_auction(watch, status, db_writer)
        return

    # 4. Check outpriced
    total_cost = (status.current_bid or 0) + (watch.shipping_cost or 0)
    max_price = await get_max_price_for_item(watch.wishlist_item_id)
    if total_cost > max_price:
        await mark_outpriced(watch, status, max_price, db_writer)
        return

    # 5. Time-based state transitions
    minutes_remaining = (status.auction_end_time - utc_now()).total_seconds() / 60

    new_state = watch.state
    if minutes_remaining <= watch.alert_threshold_min:
        new_state = "alert_window"
    elif minutes_remaining <= watch.approaching_threshold_min:
        new_state = "approaching"
    elif watch.state in ("approaching", "alert_window") and minutes_remaining > watch.approaching_threshold_min:
        # Auto-extend detected — auction end time moved forward
        new_state = "watching"

    # 6. Update database with new status
    await update_auction_watch(watch, status, new_state, db_writer)

    # 7. Handle state entry events
    if new_state == "alert_window" and watch.state != "alert_window":
        await send_auction_alert(watch, status)
    elif new_state == "alert_window" and watch.state == "alert_window":
        # Check for significant bid changes during alert window
        if old_bid and status.current_bid:
            change_pct = abs(status.current_bid - old_bid) / old_bid
            if change_pct > 0.10:  # >10% change
                await send_bid_change_alert(watch, old_bid, status.current_bid)

    # 8. Set next poll time based on state
    interval = get_poll_interval(new_state)
    await set_next_poll(watch.id, utc_now() + timedelta(minutes=interval))
```

### Polling Intervals by State

From `config.yaml` under `auctions.polling`:

| State | Interval |
|-------|----------|
| `discovered` | 30 min |
| `watching` | 15 min |
| `approaching` | 5 min |
| `alert_window` | 1 min |

### Alert Window Behavior

When an auction enters `alert_window`:

1. **Initial alert** sent via ntfy.sh with an "I'm on it" action button. The button triggers a POST to `/api/auctions/{id}/flag-intent`, which sets `intent_flagged = true` and `intent_flagged_at`.
2. **Follow-up alerts** on significant bid changes (>10%) during `alert_window`, rate-limited to max one per 2 minutes.
3. **1-minute polling** to capture final moments.

### Resolution

When an auction ends:

```python
async def resolve_auction(watch, status, db_writer):
    resolution = determine_resolution(watch, status)
    # Always record sold price to price_history
    if status.final_price:
        await record_price_history(watch, status.final_price, "auction_sold")

    await update_auction_watch(watch.id, {
        "state": "resolved",
        "resolution": resolution,
        "final_price": status.final_price,
        "resolved_at": utc_now(),
    })
```

Resolution values:
- `won` — user won (confirmed via dashboard)
- `lost` — user bid but was outbid (confirmed via dashboard)
- `passed` — user chose not to bid
- `pending_result` — user flagged intent OR placed bid; awaiting manual confirmation
- `expired_no_bids` — auction ended with zero bids
- `cancelled_by_seller` — seller cancelled the auction
- `bin_purchased` — Buy It Now purchase detected

The system never assumes "lost" automatically. If `intent_flagged` or `user_bid_placed` is true, the resolution is `pending_result` until the user confirms the outcome through the dashboard.

### Outpriced Short-Circuit

When `current_bid + shipping > max_price`, the auction transitions to `outpriced` and stops polling. This saves API quota on auctions the user can't win. The `outpriced_at` timestamp and `outpriced_note` (containing bid amount, max price, number of bidders, time remaining) are recorded.

### Testing

- **State transition tests:** For every valid transition, verify it happens at the right time. For every invalid transition, verify it doesn't. Use `freezegun` to control time precisely.
- **Auto-extend tests:** Simulate auction end time moving forward, verify backward transition from `approaching` → `watching`.
- **Outpriced tests:** Test threshold boundary (exactly at max, one cent over, with/without shipping).
- **Alert tests:** Verify initial alert sent on `alert_window` entry. Verify bid change alerts respect 10% threshold and 2-minute rate limit.
- **Resolution tests:** Test each resolution path. Verify `pending_result` used when `intent_flagged` is true.

---

## 6. Notification System

### ntfy.sh Client

All notifications go through ntfy.sh — a simple HTTP POST to a topic URL. The ntfy topic acts as the sole authentication mechanism (use a long random string).

```python
class NtfyClient:
    """Sends notifications via ntfy.sh HTTP API."""

    def __init__(self, base_url: str, topic: str):
        self.url = f"{base_url}/{topic}"
        self._client = httpx.AsyncClient()

    async def send(self, notification: PreparedNotification) -> bool:
        headers = {
            "Title": notification.title,
            "Priority": self._map_priority(notification.priority),
            "Tags": ",".join(notification.tags),
        }
        if notification.url:
            headers["Click"] = notification.url
        if notification.actions:
            headers["Actions"] = notification.format_actions()

        response = await self._client.post(
            self.url, content=notification.body, headers=headers
        )
        return response.status_code == 200
```

### Alert Types and Formatters

Five alert types, each with its own formatting function:

| Type | Trigger | Content |
|------|---------|---------|
| `deal` | Score threshold met | Item name, price, score, source, direct link |
| `auction` | State change (entering alert_window, bid changes) | Title, current bid, time remaining, action button |
| `price_drop` | Price decreased significantly | Item, old price, new price, source, % drop |
| `restock` | Previously out-of-stock item returns | Item, price, source |
| `system` | Errors, health issues | Source name, error type, consecutive failures |

Each formatter produces a `PreparedNotification` with title, body, priority, tags (emoji), URL, and optional action buttons.

### Suppression Rules

Evaluated in order. First matching rule determines the outcome:

**1. Quiet hours:**
- Configured in `config.yaml`: 23:00–07:00 America/New_York by default.
- Defers (does not discard) — sets `send_after` timestamp to quiet hours end.
- `critical` priority bypasses quiet hours (configurable `bypass_priorities` list).

**2. Cooldown per item:**
- Prevents notification spam for the same wishlist item.
- Cooldown duration varies by priority:
  - `critical`: 0 minutes (no cooldown)
  - `high`: 30 minutes
  - `normal`/`low`: 120 minutes (2 hours)
- Auction alerts bypass cooldown entirely (their own rate limiting handles frequency).
- Higher priority alert overrides a lower-priority cooldown. If a `normal` alert was sent 10 minutes ago and a `high` alert triggers, the `high` alert goes through.

**3. Duplicate detection:**
- Same `trigger_reference_id` + `trigger_type` within 30 minutes → suppressed.
- Prevents re-alerting on the same scored deal if it gets re-processed.

Suppressed notifications still get an `alerts` row with `delivery_status: "suppressed"` and `suppressed_reason` for audit trail.

### Notification Worker

A background task that polls the `alerts` table and delivers pending notifications:

```python
async def notification_worker_loop(db_writer, db_reader):
    while not shutting_down:
        # 1. Get deliverable alerts (pending, send_after <= now, priority-ordered)
        alerts = await db_reader.get_pending_alerts(limit=20)

        for alert in alerts:
            # 2. Check suppression rules
            suppression = check_suppression(alert)
            if suppression.suppressed:
                await mark_suppressed(alert, suppression.reason, db_writer)
                continue

            # 3. Attempt delivery
            success = await ntfy_client.send(format_alert(alert))
            if success:
                await mark_sent(alert, db_writer)
            else:
                await mark_delivery_attempt(alert, db_writer)

        await asyncio.sleep(5)  # 5-second poll cycle
```

### Retry Logic

- Default max attempts: 3 (configurable). Auction alerts get 5 attempts.
- Exponential backoff between retries: 30s, 60s, 120s, etc.
- Before each retry, check `expires_at` — don't retry expired alerts (e.g., auction already ended).
- On permanent failure (max attempts exceeded), create a system alert about the delivery failure.

**Recursion guard:** System alerts about delivery failures have a flag that prevents them from spawning their own failure alerts if delivery also fails. Without this, a dead ntfy.sh server would generate infinite cascading system alerts.

### Testing

- **Quiet hours tests:** Test deferral with `freezegun`. Test critical bypass. Test edge cases at boundary times.
- **Cooldown tests:** Test per-priority cooldown windows. Test higher-priority override.
- **Duplicate tests:** Test same trigger within/outside 30-minute window.
- **Retry tests:** Test exponential backoff timing. Test expiry check. Test permanent failure → system alert. Test recursion guard.
- **Formatter tests:** Each alert type formatter tested with representative data, verify output structure.

---

## 7. Dashboard & API

### Architecture

The dashboard is a React SPA served as static files from the orchestrator's FastAPI process. No build tooling — uses HTM for JSX-like syntax and CDN imports for React and Recharts.

```
dashboard/
  static/
    index.html          # Shell + CDN imports
    app.js              # Main React app with HTM
    components/         # React components
    styles/             # CSS
```

All timestamps are returned as UTC from the API. The browser converts to local time using JavaScript `Intl.DateTimeFormat` or similar. No timezone conversion happens in Python.

### Real-Time Updates

**Server-Sent Events (SSE)** for near-real-time dashboard updates instead of polling:

```python
@app.get("/api/events")
async def event_stream():
    async def generate():
        async for event in event_bus.subscribe():
            yield f"data: {event.json()}\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```

Events pushed on: new scored deal, auction state change, alert sent, source health change. The dashboard falls back to periodic polling if the SSE connection drops.

### Dashboard Views

**1. Overview** — At-a-glance system health and recent activity.
- Active wishlist items with best current deal per item
- Recent alerts (last 24h)
- Source health summary (healthy/degraded/down counts)
- Scoring pipeline status (queue depth, worker health)
- Uses a single aggregated query with LEFT JOINs — no N+1

**2. Wishlist Management** — CRUD for wishlist items.
- Create/edit items with keyword groups, price targets, condition minimums
- Pause/resume/cancel items
- Set urgency tiers and source selection
- View per-item error history

**3. Deals Feed** — Scored deals feed with filters.
- Filter by item, source, alert tier, score range, date range
- Sort by score, price, recency
- User feedback buttons (good_deal, bad_deal, irrelevant, wrong_item)
- Link to original listing

**4. Price History** — Trend charts powered by Recharts.
- Per-item price over time from `price_history`
- Market summary trends from `market_summary`
- Source comparison (which source has the best prices)
- Condition breakdown

**5. Auction Center** — Active auction monitoring.
- List of active watches by state (alert_window first)
- Per-auction detail: bid history, time remaining, state timeline
- "I'm on it" / "Bid placed" manual toggles
- Resolution confirmation for `pending_result` auctions

**6. System Health** — Operational monitoring.
- Per-source status with last success/failure, response times, error counts
- Source enable/disable toggles
- Force-run button per source
- Scoring worker status and health history
- Notification delivery stats

### Public API Endpoints

```
# Wishlist
GET    /api/wishlist                      # List active items
POST   /api/wishlist                      # Create item
GET    /api/wishlist/{id}                 # Get item detail
PUT    /api/wishlist/{id}                 # Update item
DELETE /api/wishlist/{id}                 # Cancel item (soft)
POST   /api/wishlist/{id}/pause           # Pause (with optional pause_until)
POST   /api/wishlist/{id}/resume          # Resume
POST   /api/wishlist/{id}/fulfill         # Mark purchased

# Deals
GET    /api/deals                         # Scored deals feed (filterable)
GET    /api/deals/{id}                    # Deal detail with full scoring info
POST   /api/deals/{id}/feedback           # Submit user feedback

# Prices
GET    /api/prices/{item_id}              # Price history for item
GET    /api/prices/{item_id}/summary      # Market summary trends

# Auctions
GET    /api/auctions                      # Active auction watches
GET    /api/auctions/{id}                 # Auction detail
POST   /api/auctions/{id}/flag-intent     # "I'm on it" from notification
POST   /api/auctions/{id}/mark-bid        # User placed a bid
POST   /api/auctions/{id}/resolve         # Confirm outcome

# System
GET    /api/health                        # Overall system health
GET    /api/sources                       # Source health list
POST   /api/sources/{name}/toggle         # Enable/disable source
POST   /api/sources/{name}/force-run      # Trigger immediate poll
GET    /api/scoring/stats                 # Scoring accuracy analysis

# Real-time
GET    /api/events                        # SSE stream
```

### Internal API Endpoints

Under `/api/internal/` — secured with `X-Internal-Secret` header. These are how the PC scoring worker communicates with the orchestrator.

```python
@app.post("/api/internal/scoring/pick-batch")
async def pick_batch(request: PickBatchRequest):
    """Atomically claim pending scoring queue items."""
    verify_internal_secret(request)
    items = await db_writer.execute(
        PICK_BATCH_QUERY,
        {"batch_size": request.batch_size, "worker_id": request.worker_id}
    )
    return items

@app.post("/api/internal/scoring/heartbeat")
async def heartbeat(request: HeartbeatRequest):
    """Update heartbeat timestamp for items being processed."""
    verify_internal_secret(request)
    await db_writer.execute(UPDATE_HEARTBEAT, {"queue_ids": request.queue_ids})

@app.post("/api/internal/scoring/submit")
async def submit_result(request: SubmitResultRequest):
    """Submit one scoring result. Creates scored_deal, updates queue, triggers alert check."""
    verify_internal_secret(request)
    # 1. Insert scored_deal
    # 2. Update scoring_queue status to completed
    # 3. Determine alert tier
    # 4. Create alert row if threshold met
    # 5. Maybe create auction_watch
    # 6. Record to price_history

@app.post("/api/internal/scoring/report-failure")
async def report_failure(request: FailureRequest):
    """Report a scoring failure. Increments attempts, may requeue."""
    verify_internal_secret(request)

@app.post("/api/internal/scoring/worker-health")
async def worker_health(request: WorkerHealthRequest):
    """Periodic health report from scoring worker."""
    verify_internal_secret(request)
```

The `verify_internal_secret()` function compares the `X-Internal-Secret` header against the `INTERNAL_API_SECRET` env var. Mismatch returns 403.

### Testing

- **API endpoint tests:** FastAPI TestClient with test database. Test CRUD operations, filter combinations, pagination.
- **SSE tests:** Test event delivery, reconnection behavior.
- **Internal API tests:** Test batch pickup atomicity, heartbeat updates, result submission flow.
- **Dashboard tests:** Manual testing in browser. Verify timezone display, chart rendering, real-time updates.

---

## 8. Project Structure

```
deal-hunter/
├── src/
│   ├── __init__.py
│   ├── orchestrator/
│   │   ├── __init__.py
│   │   ├── main.py                    # Entry point: python -m src.orchestrator.main
│   │   ├── app.py                     # FastAPI app setup, routes, middleware
│   │   ├── scheduler.py               # Poll scheduling loop
│   │   ├── auction_monitor.py         # Auction state machine loop
│   │   ├── notification_worker.py     # Alert delivery loop
│   │   ├── maintenance.py             # Cleanup, market summaries, pause reactivation
│   │   └── startup.py                 # Initialization, crash recovery
│   ├── scoring/
│   │   ├── __init__.py
│   │   ├── worker.py                  # Entry point: python -m src.scoring.worker
│   │   ├── prefilter.py               # Rule-based pre-filter (runs on MacBook)
│   │   ├── prompt.py                  # Prompt construction
│   │   ├── parser.py                  # JSON extraction + Pydantic validation
│   │   ├── calculator.py              # Score weighting + alert tier determination
│   │   └── ollama_client.py           # Ollama HTTP client
│   ├── adapters/
│   │   ├── __init__.py
│   │   ├── base.py                    # SourceAdapter ABC
│   │   ├── scraper_base.py            # ScraperAdapter base class
│   │   ├── registry.py                # ADAPTER_CLASSES dict + create_adapters()
│   │   ├── ebay.py
│   │   ├── reddit.py                  # RedditSubredditAdapter (serves 3 subs)
│   │   ├── amazon.py
│   │   ├── slickdeals.py
│   │   ├── newegg.py
│   │   ├── microcenter.py
│   │   └── bestbuy.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── wishlist.py                # WishlistItem, WishlistQuery, KeywordGroup
│   │   ├── listings.py                # RawListing, AdapterResult, AdapterError
│   │   ├── scoring.py                 # ScoringResponse, PreFilterResult, ScoredDeal
│   │   ├── auctions.py                # AuctionWatch, AuctionStatus
│   │   ├── alerts.py                  # Alert, PreparedNotification
│   │   └── health.py                  # SourceHealth, WorkerHealth, HealthCheckResult
│   ├── database/
│   │   ├── __init__.py
│   │   ├── writer.py                  # DatabaseWriter (async queue)
│   │   ├── reader.py                  # DatabaseReader (concurrent reads)
│   │   ├── schema.py                  # CREATE TABLE statements + migrations
│   │   └── queries.py                 # ALL SQL centralized here (or queries/ subdir)
│   ├── notifications/
│   │   ├── __init__.py
│   │   ├── ntfy_client.py             # ntfy.sh HTTP client
│   │   ├── formatters.py              # Per-alert-type formatters
│   │   └── suppression.py             # Quiet hours, cooldown, dedup logic
│   ├── shared/
│   │   ├── __init__.py
│   │   ├── config.py                  # YAML + .env loading, layered merge
│   │   ├── auth.py                    # TokenManager, SharedAuthRegistry
│   │   ├── rate_limiter.py            # RateLimiter, SharedRateLimiterRegistry
│   │   ├── url_utils.py               # normalize_url(), strip_tracking_params()
│   │   ├── time_utils.py              # to_utc(), utc_now(), duration helpers
│   │   ├── safe_math.py               # None-safe arithmetic for nullable decimals
│   │   ├── logging.py                 # structlog setup, secret redaction processor
│   │   └── exceptions.py              # Custom exception hierarchy
│   └── api/
│       ├── __init__.py
│       ├── public.py                  # Public REST endpoints
│       ├── internal.py                # Worker-facing internal endpoints
│       ├── events.py                  # SSE endpoint
│       └── middleware.py              # Auth middleware, error handlers
├── dashboard/
│   └── static/
│       ├── index.html
│       ├── app.js
│       ├── components/
│       └── styles/
├── prompts/
│   └── scoring_v1.0.txt               # Versioned scoring prompt
├── tests/
│   ├── conftest.py                    # Shared fixtures, test database setup
│   ├── fixtures/                      # Saved real responses per source
│   │   ├── reddit/
│   │   ├── ebay/
│   │   ├── amazon/
│   │   └── ...
│   ├── test_prefilter.py
│   ├── test_scoring_parser.py
│   ├── test_adapters/
│   │   ├── test_reddit.py
│   │   ├── test_ebay.py
│   │   └── ...
│   ├── test_scheduler.py
│   ├── test_auction_monitor.py
│   ├── test_notifications.py
│   ├── test_api.py
│   └── test_database.py
├── config.yaml                        # Committed config (see companion doc)
├── local.yaml.example                 # Machine-specific template
├── .env.example                       # Secrets template
├── requirements.txt
├── pyproject.toml
└── data/                              # SQLite database lives here (gitignored)
    └── .gitkeep
```

### Key Design Rules

**`models/`** contains pure Pydantic definitions only. No business logic, no database access, no imports from other `src/` modules (except `shared/`).

**`database/queries.py`** centralizes ALL SQL. Business logic modules import query functions, never write raw SQL.

**`shared/`** is the cross-cutting utilities layer. Every other module can import from `shared/`. Nothing in `shared/` imports from business-logic modules.

### Configuration Loading

Config is layered: `config.yaml` (committed defaults) → `local.yaml` (machine-specific overrides, gitignored) → `.env` (secrets, gitignored). The merge is deep — `local.yaml` can override individual nested keys without replacing entire sections.

```python
def load_config() -> Config:
    """Load and merge configuration from all layers."""
    base = yaml.safe_load(open("config.yaml"))

    local_path = Path("local.yaml")
    if local_path.exists():
        local = yaml.safe_load(open(local_path))
        deep_merge(base, local)  # local overrides base

    load_dotenv()  # .env into os.environ

    return Config(**base)
```

### Entry Points

- **MacBook:** `python -m src.orchestrator.main` — starts the FastAPI orchestrator with all background tasks.
- **PC:** `python -m src.scoring.worker` — starts the scoring worker loop.

Both reference `.venv/bin/python` (or `.venv\Scripts\python.exe` on Windows) in their service configurations.

### Service Management

**macOS (launchd):** A plist in `~/Library/LaunchAgents/` configured with `KeepAlive: true` for auto-restart on crash. References the project's virtualenv Python.

**Windows (NSSM):** NSSM wraps the scoring worker as a Windows service. Configured with auto-restart, logging to file, and the project's virtualenv Python.

---

## 9. Development Milestones

Each milestone produces a working, testable increment. Dependencies flow downward — each milestone builds on the previous ones.

### M0: Foundation (2–3 days)

**Goal:** Runnable project skeleton with database, config, and logging.

**Deliverables:**
- Project structure created per Section 8
- `config.yaml`, `local.yaml.example`, `.env.example` in place
- Config loading with deep merge (`shared/config.py`)
- structlog setup with JSON formatting and secret redaction (`shared/logging.py`)
- SQLite database initialization with all 10 tables (`database/schema.py`)
- `DatabaseWriter` and `DatabaseReader` classes (`database/writer.py`, `database/reader.py`)
- `safe_math.py`, `time_utils.py`, `url_utils.py` utilities
- `pyproject.toml` and `requirements.txt`
- Basic test infrastructure (`conftest.py` with test database fixture)

**Test:** `pytest` passes. Database creates all tables. Config loads and merges correctly.

### M1: First Adapter End-to-End (4–5 days)

**Goal:** Reddit adapter → pre-filter → dashboard shows results.

**Deliverables:**
- `SourceAdapter` ABC and `safe_search` wrapper
- `RateLimiter` and `SharedRateLimiterRegistry`
- `RedditSubredditAdapter` with `RedditTokenManager` (raw httpx, password grant)
- `SharedAuthRegistry`
- `WishlistQuery` and `RawListing` models
- Pre-filter with all 6 checks
- Basic FastAPI app with wishlist CRUD endpoints
- Minimal dashboard: wishlist list, raw listings table
- One Reddit subreddit (r/buildapcsales) connected

**Test:** Create a wishlist item via API. Reddit adapter finds listings. Pre-filter passes/rejects correctly. Dashboard shows results.

### M2: Scheduling & Source Health (3–4 days)

**Goal:** Automated polling with source health tracking and crash recovery.

**Deliverables:**
- Scheduler loop with urgency multipliers and next-check-at model
- `source_health` table updates on every poll (success, failure, rate limit)
- Backoff logic for failed sources (transient vs. non-transient)
- Silent failure detection (consecutive empty runs)
- Crash recovery on startup (stuck items, schedule recalculation, pause reactivation)
- Source health dashboard view
- Force-run and enable/disable from dashboard

**Test:** Run scheduler, verify polls happen at correct intervals. Simulate failures, verify backoff. Kill and restart, verify recovery.

### M3: LLM Scoring Pipeline (4–5 days)

**Goal:** Listings scored by Qwen via Ollama on PC, communicated through internal API.

**Deliverables:**
- `scoring_queue` table operations
- Internal API endpoints (pick-batch, heartbeat, submit, report-failure, worker-health)
- Scoring worker loop (`src/scoring/worker.py`)
- Ollama client
- Scoring prompt (`prompts/scoring_v1.0.txt`)
- Brace-depth JSON extractor and Pydantic score parser
- Final score calculation with urgency multipliers
- Alert tier determination with hard overrides
- WOL decision logic and PC status checking
- Scored deals visible in dashboard

**Test:** Queue a listing. Worker picks it up, scores it, submits result. Verify score parsing with various LLM output formats. Test WOL decision thresholds.

### M4: Notifications (2–3 days)

**Goal:** Push notifications via ntfy.sh with full suppression rules.

**Deliverables:**
- ntfy.sh client
- All 5 alert type formatters
- Suppression logic (quiet hours, cooldown, dedup)
- Notification worker loop
- Retry logic with exponential backoff
- Recursion guard on system alerts
- Notification health in dashboard

**Test:** Score a deal above threshold → notification appears on phone. Test quiet hours deferral. Test cooldown. Test retry on failure.

### M5: Amazon, Slickdeals, Newegg Adapters (4–5 days)

**Goal:** Three more sources producing scored results.

**Deliverables:**
- `SlickdealsAdapter` (RSS via feedparser)
- `AmazonAdapter` (scraper with `ScraperAdapter` base)
- `NeweggAdapter` (scraper)
- Saved fixtures for each source
- URL normalization for each source's tracking parameters

**Test:** Each adapter returns parsed listings from saved fixtures. Integration test against live sites.

### M6: Microcenter, Best Buy, eBay Adapters (4–5 days)

**Goal:** Remaining adapters, including eBay which enables auction tracking.

**Deliverables:**
- `MicrocenterAdapter` (scraper)
- `BestBuyAdapter` (scraper)
- `EbayAdapter` (API if approved, scraper fallback) with `check_auction_status()` capability
- `TokenManager` for eBay OAuth
- Saved fixtures for each source

**Test:** Each adapter returns parsed listings. eBay adapter can fetch auction status.

### M7: Auction State Machine (4–5 days)

**Goal:** Full auction monitoring lifecycle.

**Deliverables:**
- Auction monitor loop
- Auto-watch creation from scored deals
- All state transitions (time-based, outpriced, terminal, auto-extend backward)
- Alert window notifications with "I'm on it" action button
- Bid change alerts during alert window
- Resolution logic (won, lost, passed, pending_result, etc.)
- Auction center dashboard view
- `/api/auctions/{id}/flag-intent` endpoint

**Test:** Create an auction watch. Simulate time progression through states. Test outpriced detection. Test alert window behavior. Test resolution paths.

### M8: Price History & Dashboard Polish (3–4 days)

**Goal:** Price tracking, trend charts, and dashboard refinements.

**Deliverables:**
- Price history recording from all adapter results
- Dedup logic (only insert on price change or 24+ hours since last observation)
- Sold prices always recorded from auction resolutions
- Market summary generation in maintenance worker
- Price history charts with Recharts
- SSE for real-time dashboard updates
- Dashboard polish: loading states, error handling, responsive layout

**Test:** Verify price history dedup. Verify market summary generation. Verify SSE events arrive in browser.

### M9: Hardening & Dogfooding (5–7 days)

**Goal:** Production-ready stability for daily use.

**Deliverables:**
- Memory profiling and optimization for MacBook's 8GB constraint
- Connection pool tuning
- Graceful shutdown testing
- launchd plist for MacBook auto-start
- NSSM configuration for PC auto-start
- Log rotation setup
- Maintenance worker: raw_listings 30-day retention, scoring_queue 7-day retention
- End-to-end testing with real wishlist items
- Bug fixes from dogfooding
- Documentation: README with setup instructions

**Test:** Run the full system for several days. Monitor memory usage. Verify no data loss on restart. Verify all notification paths work on real phone.

### Timeline

Total estimated: **35–46 days part-time**.

This assumes working with Claude Sonnet for implementation, using the milestone structure to keep each implementation chat focused on a well-defined scope with clear inputs and outputs.

---

## Appendix: Configuration Reference

See the companion configuration files for full details:

- **`config.yaml`** — All tunable parameters with comments: orchestrator settings, scoring weights and thresholds, auction tier defaults, notification rules, scraping defaults, per-source configuration, rate limits, database path, logging, and maintenance schedules.
- **`local.yaml.example`** — Machine-specific overrides: hardware IPs, MAC address for WOL, database path, port assignments.
- **`.env.example`** — Secret credentials: eBay API keys, Reddit API credentials, ntfy.sh topic, internal API shared secret.

### Config Layering

```
config.yaml (committed)     ← Defaults and tunable parameters
    ↓ deep merge
local.yaml (gitignored)     ← Machine-specific: IPs, paths, ports
    ↓ env var resolution
.env (gitignored)            ← Secrets: API keys, passwords, tokens
```

Config references env var *names* (e.g., `client_id_env: "EBAY_CLIENT_ID"`). The application resolves these at runtime via `os.environ`. Secrets never appear in YAML files.
