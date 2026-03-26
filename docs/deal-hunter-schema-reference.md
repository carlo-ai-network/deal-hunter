# Deal Hunter — Database Schema Reference

**Version:** 1.0 (V1 Spec — Final)
**Database:** SQLite (WAL mode)
**Location:** Configured in `local.yaml` → `database.path`

---

## Conventions

- **Soft deletes:** Tables referenced by polymorphic FKs include `deleted_at` (nullable timestamp). Application queries filter `WHERE deleted_at IS NULL` by default. Consider creating database views that apply this filter automatically.
- **JSON columns:** Used for flexible, type-specific data. Queried in Python, not SQL.
- **Timestamps:** All timestamps are UTC ISO 8601. Browser converts to local time for display. Never convert in Python for API responses.
- **Enums:** Stored as TEXT. Validated by Pydantic models in application code.
- **Configurable defaults:** Per-row threshold values initialized via copy-on-write from `config.yaml` at creation time. Per-row overrides supported.
- **NULL arithmetic:** All SQL involving nullable numeric columns must use `COALESCE` or explicit `IS NULL` checks. All Python arithmetic on nullable fields must guard against `None`. Use `safe_math.py` utilities.
- **Query centralization:** All SQL lives in `src/database/queries.py` (or `queries/` subdirectory). No inline SQL in business logic.
- **Aggregation over N+1:** List endpoints use LEFT JOINs and CTEs, never per-row subqueries in application loops.

---

## Table Overview

| # | Table | Purpose | Growth Pattern |
|---|-------|---------|----------------|
| 1 | `wishlist_items` | What you're hunting for | Low (manual, ~10-50 rows) |
| 2 | `item_errors` | Per-item per-source error tracking | Low (deduplicated) |
| 3 | `raw_listings` | Every listing found by adapters | High (pruned at 30 days) |
| 4 | `scored_deals` | LLM scoring results | Medium (subset of raw_listings) |
| 5 | `price_history` | Time-series price observations | Medium (append-only, kept forever) |
| 6 | `market_summary` | Daily price rollup per item | Low (1 row/item/day) |
| 7 | `auction_watches` | eBay auction state machine | Medium (pruned when resolved) |
| 8 | `alerts` | Notification log with delivery tracking | Medium (kept forever) |
| 9 | `scoring_queue` | Work queue bridging MacBook → PC | Transient (pruned at 7 days) |
| 10 | `source_health` | Per-adapter operational status | Static (1 row per source) |

---

## 1. wishlist_items

The heart of the system. Every other table exists to serve these items.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `name` | TEXT | No | — | Human-readable label (e.g., "DDR5 64GB Kit 6000MHz CL30") |
| `category` | TEXT | No | — | Grouping for scan frequency and alerts (e.g., "pc_hardware", "apple") |
| `status` | TEXT | No | "active" | `active`, `paused`, `fulfilled`, `cancelled` |
| `pause_until` | TIMESTAMP | Yes | NULL | If paused, auto-reactivates when reached. NULL = indefinite pause |
| `cancelled_reason` | TEXT | Yes | NULL | Why cancelled (e.g., "Bought the M4 Pro instead") |
| `target_price` | DECIMAL | No | — | Price at or below which to trigger alerts |
| `max_price` | DECIMAL | No | — | Absolute ceiling. Anything above is filtered out |
| `condition_minimum` | TEXT | No | — | `new`, `open_box`, `refurbished`, `used_like_new`, `used_good`, `any` |
| `keyword_groups` | JSON | No | — | Array of objects, each with `terms` array. OR between groups, AND within |
| `negative_keywords` | JSON | No | — | Array of disqualifying terms |
| `match_criteria` | JSON | No | — | Freeform per-item specs the LLM verifies |
| `urgency_tier` | TEXT | No | — | `critical`, `high`, `normal`, `low` |
| `sources` | JSON | No | — | Array of source adapter names to monitor |
| `notes` | TEXT | Yes | NULL | Freeform context |
| `last_checked_at` | TIMESTAMP | Yes | NULL | When any source last checked for this item |
| `last_alerted_at` | TIMESTAMP | Yes | NULL | Last alert sent (used for cooldown logic) |
| `fulfilled_at` | TIMESTAMP | Yes | NULL | When purchased |
| `fulfilled_price` | DECIMAL | Yes | NULL | What you paid |
| `fulfilled_source` | TEXT | Yes | NULL | Source adapter name of the purchased deal |
| `created_at` | TIMESTAMP | No | — | Row creation time |
| `updated_at` | TIMESTAMP | No | — | Last modification time |

---

## 2. item_errors

Per-item per-source error tracking with deduplication. Same error type increments counter rather than creating duplicates.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `wishlist_item_id` | INTEGER FK | No | — | References `wishlist_items.id` |
| `source` | TEXT | No | — | Adapter name that produced the error |
| `error_type` | TEXT | No | — | `scrape_failed`, `parse_error`, `rate_limited`, `timeout`, `auth_failed` |
| `error_detail` | JSON | Yes | NULL | Structured debug context (HTTP status, selector, URL, response snippet) |
| `status` | TEXT | No | "active" | `active`, `resolved`, `ignored` |
| `first_occurred_at` | TIMESTAMP | No | — | When first seen |
| `last_occurred_at` | TIMESTAMP | No | — | Most recent occurrence |
| `occurrence_count` | INTEGER | No | 1 | Times this error has occurred |
| `resolved_at` | TIMESTAMP | Yes | NULL | When resolved |

**Unique constraint:** `(wishlist_item_id, source, error_type, status)`

---

## 3. raw_listings

Every listing found by any adapter, normalized. Highest-volume table. Soft-deleted at 30 days for non-active listings.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `wishlist_item_id` | INTEGER FK | No | — | Which wishlist item's search produced this |
| `source` | TEXT | No | — | Adapter name |
| `external_id` | TEXT | No | — | Source's own identifier |
| `url` | TEXT | No | — | Direct link (normalized, tracking params stripped) |
| `title` | TEXT | No | — | Listing title |
| `description` | TEXT | Yes | NULL | Full description or post body |
| `price` | DECIMAL | Yes | NULL | Best available price (sticker, extracted, or parsed) |
| `price_confidence` | TEXT | No | "unknown" | `exact`, `parsed`, `extracted`, `unknown` |
| `shipping_cost` | DECIMAL | Yes | NULL | Shipping (NULL if free or unknown) |
| `estimated_tax` | DECIMAL | Yes | NULL | Tax if source provides it |
| `estimated_fees` | DECIMAL | Yes | NULL | Buyer fees if applicable |
| `total_estimated_cost` | DECIMAL | Yes | NULL | Computed: price + shipping + tax + fees |
| `currency` | TEXT | No | "USD" | Currency code |
| `condition` | TEXT | Yes | NULL | `new`, `open_box`, `refurbished`, `used_like_new`, `used_good`, `used_fair`, `unknown` |
| `seller_name` | TEXT | Yes | NULL | Seller display name |
| `seller_rating` | DECIMAL | Yes | NULL | Reputation score |
| `listing_type` | TEXT | No | — | `buy_now`, `auction`, `classified`, `deal_post` |
| `auction_end_time` | TIMESTAMP | Yes | NULL | Only for auctions (UTC) |
| `status` | TEXT | No | "active" | `active`, `ended`, `sold`, `expired`, `deleted` |
| `raw_data` | JSON | No | — | Complete original response (trimmed to relevant sections) |
| `prefilter_passed` | BOOLEAN | Yes | NULL | NULL = not yet filtered |
| `prefilter_reason` | TEXT | Yes | NULL | Why rejected |
| `listing_created_at` | TIMESTAMP | Yes | NULL | When posted on source (UTC) |
| `discovered_at` | TIMESTAMP | No | — | When our system first saw it |
| `updated_at` | TIMESTAMP | No | — | Last data refresh |
| `deleted_at` | TIMESTAMP | Yes | NULL | Soft delete |

**Unique constraint:** `(wishlist_item_id, source, external_id)`

---

## 4. scored_deals

LLM scoring results. Full audit trail of model output, parsing, and performance.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `raw_listing_id` | INTEGER FK | No | — | References `raw_listings.id` |
| `wishlist_item_id` | INTEGER FK | No | — | Denormalized for query convenience |
| `spec_score` | INTEGER | Yes | NULL | 0-100: spec match quality |
| `condition_score` | INTEGER | Yes | NULL | 0-100: condition confidence |
| `value_score` | INTEGER | Yes | NULL | 0-100: price relative to target |
| `urgency_multiplier` | DECIMAL | Yes | NULL | 1.0-2.0: time pressure modifier |
| `final_score` | DECIMAL | Yes | NULL | Weighted composite × urgency. Weights in config.yaml |
| `risk_flags` | JSON | Yes | NULL | Array of flags from closed set |
| `llm_reasoning` | TEXT | Yes | NULL | LLM's explanation (max 1000 chars) |
| `llm_model` | TEXT | No | — | Model identifier (e.g., "qwen2.5:14b") |
| `llm_prompt_version` | TEXT | No | — | Prompt version for A/B comparison |
| `llm_raw_response` | TEXT | No | — | Complete raw output before parsing |
| `parse_status` | TEXT | No | — | `success`, `partial`, `failed` |
| `parse_error` | TEXT | Yes | NULL | What went wrong during parsing |
| `prompt_tokens` | INTEGER | Yes | NULL | Input token count |
| `completion_tokens` | INTEGER | Yes | NULL | Output token count |
| `processing_time_ms` | INTEGER | Yes | NULL | Inference duration |
| `alert_triggered` | BOOLEAN | No | false | Whether this score caused a notification |
| `alert_tier` | TEXT | Yes | NULL | `high`, `normal`, `low`, `none` |
| `user_feedback` | TEXT | Yes | NULL | `good_deal`, `bad_deal`, `irrelevant`, `wrong_item`, `purchased`, `missed_it` |
| `scored_at` | TIMESTAMP | No | — | When scoring completed |
| `updated_at` | TIMESTAMP | No | — | Last modification |
| `deleted_at` | TIMESTAMP | Yes | NULL | Soft delete |

**Unique constraint:** `(raw_listing_id)`

---

## 5. price_history

Time-series price observations. Append-only. Kept forever. Dedup: only insert when price changed or 24+ hours since last observation. Sold prices always recorded regardless of dedup.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `wishlist_item_id` | INTEGER FK | No | — | References `wishlist_items.id` |
| `source` | TEXT | No | — | Adapter name |
| `price` | DECIMAL | No | — | Observed price |
| `shipping_cost` | DECIMAL | Yes | NULL | Shipping if known |
| `total_cost` | DECIMAL | Yes | NULL | price + shipping |
| `condition` | TEXT | No | — | Condition tier this price represents |
| `listing_type` | TEXT | No | — | `buy_now`, `auction_sold`, `deal_post` |
| `availability` | TEXT | No | — | `in_stock`, `out_of_stock`, `limited_stock`, `preorder`, `unknown` |
| `external_id` | TEXT | Yes | NULL | Source listing ID for traceability |
| `url` | TEXT | Yes | NULL | Direct link |
| `observed_at` | TIMESTAMP | No | — | When observed |

**No unique constraint.** Multiple observations per day per source are valid.

---

## 6. market_summary

Daily rollup per wishlist item. Generated by maintenance worker from price_history. Powers dashboard trend charts.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `wishlist_item_id` | INTEGER FK | No | — | References `wishlist_items.id` |
| `summary_date` | DATE | No | — | The date covered |
| `lowest_price` | DECIMAL | Yes | NULL | Cheapest listing |
| `lowest_total_cost` | DECIMAL | Yes | NULL | Cheapest including shipping |
| `highest_price` | DECIMAL | Yes | NULL | Most expensive |
| `median_price` | DECIMAL | Yes | NULL | Median across observations |
| `average_price` | DECIMAL | Yes | NULL | Mean across observations |
| `total_listings_seen` | INTEGER | No | 0 | Active listings observed |
| `sources_checked` | INTEGER | No | 0 | Sources that returned data |
| `sources_with_stock` | INTEGER | No | 0 | Sources with item in stock |
| `best_source` | TEXT | Yes | NULL | Source with lowest total cost |
| `best_condition_at_lowest` | TEXT | Yes | NULL | Condition of cheapest listing |
| `price_change_pct` | DECIMAL | Yes | NULL | % change from previous day's lowest |
| `generated_at` | TIMESTAMP | No | — | When computed |

**Unique constraint:** `(wishlist_item_id, summary_date)`

---

## 7. auction_watches

eBay auction state machine. Tracks through discovery → watching → alert → resolution. Per-row configurable thresholds (copy-on-write from urgency tier defaults).

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `raw_listing_id` | INTEGER FK | No | — | References `raw_listings.id` |
| `wishlist_item_id` | INTEGER FK | No | — | References `wishlist_items.id` |
| `ebay_item_id` | TEXT | No | — | eBay item ID (denormalized for API lookups) |
| `url` | TEXT | No | — | Direct link |
| `title` | TEXT | No | — | Auction title (denormalized for alerts) |
| `state` | TEXT | No | — | `discovered`, `watching`, `approaching`, `alert_window`, `resolved`, `outpriced` |
| `resolution` | TEXT | Yes | NULL | `won`, `lost`, `passed`, `pending_result`, `expired_no_bids`, `cancelled_by_seller`, `bin_purchased` |
| `current_bid` | DECIMAL | Yes | NULL | Latest observed bid |
| `bid_count` | INTEGER | Yes | NULL | Number of bids |
| `starting_price` | DECIMAL | Yes | NULL | Auction starting price |
| `buy_it_now_price` | DECIMAL | Yes | NULL | BIN price (disappears after first bid) |
| `shipping_cost` | DECIMAL | Yes | NULL | Shipping |
| `seller_name` | TEXT | Yes | NULL | Seller |
| `seller_rating` | DECIMAL | Yes | NULL | Feedback score |
| `condition` | TEXT | Yes | NULL | Item condition |
| `auction_end_time` | TIMESTAMP | No | — | Mutable — updated from source every poll (supports auto-extend) |
| `approaching_threshold_min` | INTEGER | No | 30 | Minutes before end to enter `approaching` |
| `alert_threshold_min` | INTEGER | No | 5 | Minutes before end to enter `alert_window` |
| `last_poll_at` | TIMESTAMP | Yes | NULL | When last checked |
| `next_poll_at` | TIMESTAMP | Yes | NULL | When scheduler should next check |
| `poll_count` | INTEGER | No | 0 | Total polls |
| `scored_deal_id` | INTEGER FK | Yes | NULL | References `scored_deals.id` |
| `final_price` | DECIMAL | Yes | NULL | Actual closing price |
| `user_bid_placed` | BOOLEAN | No | false | User marked they placed a bid |
| `intent_flagged` | BOOLEAN | No | false | User tapped "I'm on it" from notification |
| `intent_flagged_at` | TIMESTAMP | Yes | NULL | When intent was flagged |
| `alert_sent_at` | TIMESTAMP | Yes | NULL | When alert window notification sent |
| `outpriced_at` | TIMESTAMP | Yes | NULL | When bid exceeded max_price |
| `outpriced_note` | TEXT | Yes | NULL | Context (bid amount, max price, bidders, time remaining) |
| `discovered_at` | TIMESTAMP | No | — | When first found |
| `updated_at` | TIMESTAMP | No | — | Last modification |
| `resolved_at` | TIMESTAMP | Yes | NULL | When reached terminal state |
| `deleted_at` | TIMESTAMP | Yes | NULL | Soft delete |

**Unique constraint:** `(ebay_item_id, wishlist_item_id)`

**State transitions:**
```
discovered → watching              (after initial assessment)
watching → approaching             (end_time - now < approaching_threshold)
watching → outpriced               (current_bid + shipping > max_price)
watching → resolved                (BIN purchased or early end)
approaching → alert_window         (end_time - now < alert_threshold)
approaching → outpriced            (current_bid + shipping > max_price)
approaching → watching             (end_time extended via auto-extend)
approaching → resolved             (early end)
alert_window → approaching         (end_time extended via auto-extend)
alert_window → resolved            (auction ended)
```

---

## 8. alerts

Notification log for all alert types. Uses polymorphic FK to link back to trigger source. Referenced tables use soft deletes to prevent orphaned FKs.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `wishlist_item_id` | INTEGER FK | Yes | NULL | NULL for system alerts |
| `alert_type` | TEXT | No | — | `deal`, `auction`, `price_drop`, `restock`, `system` |
| `priority` | TEXT | No | — | `critical`, `high`, `normal`, `low` |
| `title` | TEXT | No | — | Notification title |
| `body` | TEXT | No | — | Notification body |
| `url` | TEXT | Yes | NULL | Link attached to notification |
| `source` | TEXT | Yes | NULL | Adapter that produced the trigger |
| `trigger_type` | TEXT | No | — | `score_threshold`, `auction_state_change`, `price_delta`, `availability_change`, `error_threshold`, `health_check` |
| `trigger_reference_id` | INTEGER | Yes | NULL | Polymorphic FK to trigger row |
| `trigger_reference_table` | TEXT | Yes | NULL | Target table name |
| `context` | JSON | No | — | Type-specific payload (scores, auction state, etc.) |
| `delivery_status` | TEXT | No | "pending" | `pending`, `sent`, `failed`, `suppressed` |
| `delivery_attempts` | INTEGER | No | 0 | Send attempts |
| `max_delivery_attempts` | INTEGER | No | 3 | Retry ceiling (5 for auction alerts) |
| `last_delivery_attempt_at` | TIMESTAMP | Yes | NULL | Most recent attempt |
| `delivery_error` | TEXT | Yes | NULL | Why delivery failed |
| `suppressed_reason` | TEXT | Yes | NULL | Why suppressed |
| `send_after` | TIMESTAMP | Yes | NULL | Deferred delivery (quiet hours) |
| `ntfy_topic` | TEXT | Yes | NULL | Which topic was used |
| `user_action` | TEXT | Yes | NULL | `acknowledged`, `acted_on`, `dismissed`, `expired` |
| `acted_on_at` | TIMESTAMP | Yes | NULL | When user responded |
| `created_at` | TIMESTAMP | No | — | Alert creation |
| `sent_at` | TIMESTAMP | Yes | NULL | Successful delivery |
| `expires_at` | TIMESTAMP | Yes | NULL | Auto-expire if no user action |

---

## 9. scoring_queue

Work queue bridging MacBook (pre-filter) and PC (LLM scoring). Worker communicates via orchestrator HTTP API, not direct database access.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `raw_listing_id` | INTEGER FK | No | — | References `raw_listings.id` |
| `wishlist_item_id` | INTEGER FK | No | — | Denormalized for priority sorting |
| `status` | TEXT | No | "pending" | `pending`, `processing`, `completed`, `failed`, `cancelled` |
| `priority` | INTEGER | No | — | Lower = higher priority |
| `priority_reason` | TEXT | Yes | NULL | Human-readable (e.g., "auction_ending_soon: 18min") |
| `batch_id` | TEXT | Yes | NULL | UUID shared by items pulled together |
| `attempts` | INTEGER | No | 0 | Processing attempts |
| `max_attempts` | INTEGER | No | 3 | Retry ceiling |
| `last_attempt_at` | TIMESTAMP | Yes | NULL | Most recent attempt |
| `heartbeat_at` | TIMESTAMP | Yes | NULL | Worker updates periodically. Stale = abandoned |
| `stale_timeout_min` | INTEGER | No | 10 | Minutes without heartbeat before considered abandoned |
| `error_message` | TEXT | Yes | NULL | Why processing failed |
| `scored_deal_id` | INTEGER FK | Yes | NULL | Populated on completion |
| `worker_id` | TEXT | Yes | NULL | Which worker picked up the job |
| `queued_at` | TIMESTAMP | No | — | When entered queue |
| `picked_up_at` | TIMESTAMP | Yes | NULL | When worker started |
| `completed_at` | TIMESTAMP | Yes | NULL | When finished |
| `expires_at` | TIMESTAMP | Yes | NULL | Queue entry stale after this (e.g., auction ended) |

**Retention:** Hard delete after 7 days.

---

## 10. source_health

One row per source adapter. Living status board updated every poll cycle.

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER PK | No | Auto | Auto-increment |
| `source` | TEXT | No | — | Adapter name (unique) |
| `status` | TEXT | No | "healthy" | `healthy`, `degraded`, `down`, `silent_failure`, `disabled` |
| `status_reason` | TEXT | Yes | NULL | Human-readable explanation |
| `enabled` | BOOLEAN | No | true | Manual kill switch |
| `force_run_requested` | BOOLEAN | No | false | UI-triggered immediate run (self-clears after run) |
| `last_success_at` | TIMESTAMP | Yes | NULL | Last error-free run |
| `last_failure_at` | TIMESTAMP | Yes | NULL | Last failed run |
| `last_attempt_at` | TIMESTAMP | Yes | NULL | Most recent run |
| `last_rate_limited_at` | TIMESTAMP | Yes | NULL | Last rate limit hit |
| `consecutive_failures` | INTEGER | No | 0 | Resets on success |
| `consecutive_successes` | INTEGER | No | 0 | Resets on failure |
| `consecutive_empty_runs` | INTEGER | No | 0 | Runs returning 0 from source. Resets on non-empty |
| `empty_run_threshold` | INTEGER | No | — | Consecutive empty runs before `silent_failure` |
| `total_successes` | INTEGER | No | 0 | Lifetime count |
| `total_failures` | INTEGER | No | 0 | Lifetime count |
| `avg_response_time_ms` | INTEGER | Yes | NULL | Rolling average |
| `last_response_time_ms` | INTEGER | Yes | NULL | Most recent |
| `listings_found_last_run` | INTEGER | Yes | NULL | Post-filter listings from last run |
| `total_from_source_last_run` | INTEGER | Yes | NULL | Pre-filter results from source |
| `listings_found_total` | INTEGER | No | 0 | Lifetime listings |
| `last_error_is_transient` | BOOLEAN | Yes | NULL | Was the most recent error transient? |
| `rate_limit_remaining` | INTEGER | Yes | NULL | API quota remaining |
| `rate_limit_resets_at` | TIMESTAMP | Yes | NULL | When rate limit resets |
| `next_scheduled_check` | TIMESTAMP | Yes | NULL | When orchestrator will next run this |
| `poll_interval_seconds` | INTEGER | No | — | Configured polling interval |
| `backoff_until` | TIMESTAMP | Yes | NULL | Don't attempt before this time |
| `backoff_multiplier` | DECIMAL | No | 1.0 | Exponential backoff level (max 32.0) |
| `config_snapshot` | JSON | Yes | NULL | Adapter config for dashboard display |
| `created_at` | TIMESTAMP | No | — | Row creation |
| `updated_at` | TIMESTAMP | No | — | Last modification |

**Unique constraint:** `(source)`

**Status transitions:**
```
On success:
  consecutive_failures = 0, backoff_multiplier = 1.0, backoff_until = NULL
  If was degraded/down and consecutive_successes >= 3 → healthy
  If total_from_source == 0 → consecutive_empty_runs += 1
  If consecutive_empty_runs >= empty_run_threshold → silent_failure

On failure (not rate limit):
  consecutive_successes = 0, consecutive_failures += 1
  If consecutive_failures >= 5 → down (trigger system alert)
  If consecutive_failures >= 2 → degraded
  Transient errors need more occurrences (10+) before escalating to down
  Non-transient errors escalate faster (3+ → down)
  backoff_until = now + (base_interval × min(multiplier × 2, 32))

On rate limit:
  If rate_limit_resets_at known → backoff_until = resets_at + 60s
  Else → generic backoff
  Do NOT increment consecutive_failures
```

---

## Indexes

```sql
-- Raw listings: overview aggregation and dedup
CREATE INDEX idx_raw_listings_item_status
    ON raw_listings(wishlist_item_id, status, discovered_at)
    WHERE deleted_at IS NULL;

-- Scored deals: overview aggregation and feed queries
CREATE INDEX idx_scored_deals_item_date
    ON scored_deals(wishlist_item_id, scored_at)
    WHERE deleted_at IS NULL;

-- Auction watches: overview and monitor queries
CREATE INDEX idx_auction_watches_item_state
    ON auction_watches(wishlist_item_id, state)
    WHERE deleted_at IS NULL;

-- Auction watches: monitor loop polling
CREATE INDEX idx_auction_watches_next_poll
    ON auction_watches(next_poll_at, state)
    WHERE deleted_at IS NULL;

-- Market summary: price chart queries
CREATE INDEX idx_market_summary_item_date
    ON market_summary(wishlist_item_id, summary_date);

-- Price history: chart queries and dedup checks
CREATE INDEX idx_price_history_item_source_date
    ON price_history(wishlist_item_id, source, observed_at);

-- Scoring queue: worker batch pickup
CREATE INDEX idx_scoring_queue_pending
    ON scoring_queue(status, priority, queued_at)
    WHERE status = 'pending';

-- Scoring queue: stale heartbeat detection
CREATE INDEX idx_scoring_queue_processing
    ON scoring_queue(status, heartbeat_at)
    WHERE status = 'processing';

-- Alerts: notification worker queries
CREATE INDEX idx_alerts_pending
    ON alerts(delivery_status, send_after, priority)
    WHERE delivery_status = 'pending';

-- Alerts: cooldown checks
CREATE INDEX idx_alerts_item_sent
    ON alerts(wishlist_item_id, delivery_status, sent_at);

-- Source health: scheduling queries
CREATE INDEX idx_source_health_scheduling
    ON source_health(enabled, status, next_scheduled_check);
```

---

## Entity Relationship Summary

```
wishlist_items (1) ──→ (many) raw_listings
wishlist_items (1) ──→ (many) scored_deals
wishlist_items (1) ──→ (many) price_history
wishlist_items (1) ──→ (many) market_summary
wishlist_items (1) ──→ (many) auction_watches
wishlist_items (1) ──→ (many) alerts
wishlist_items (1) ──→ (many) item_errors
wishlist_items (1) ──→ (many) scoring_queue

raw_listings (1) ──→ (0..1) scored_deals
raw_listings (1) ──→ (0..1) auction_watches
raw_listings (1) ──→ (many) scoring_queue

scored_deals (1) ←── (0..1) auction_watches.scored_deal_id
scored_deals (1) ←── (0..1) scoring_queue.scored_deal_id

alerts.trigger_reference_id ──→ scored_deals | auction_watches | item_errors (polymorphic)

Worker → Orchestrator API (HTTP over Tailscale):
  POST /api/internal/scoring/pick-batch
  POST /api/internal/scoring/heartbeat
  POST /api/internal/scoring/submit
  POST /api/internal/scoring/report-failure
  POST /api/internal/scoring/worker-health
```
