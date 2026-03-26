# Deal Hunter — Architecture Review Sessions

**Version:** 1.1 (Updated post-spec to reflect final V1 design decisions)

---

## Session 1: API & Scraping Cost/Risk Audit

### Source-by-Source Assessment

#### eBay — API (Recommended primary approach)
- **Cost:** Free. The Developer Program is free to join and the Browse API has no per-call charges.
- **Risk:** Medium. Production access requires an application approval process — eBay reviews your use case before granting live access. Sandbox access is immediate. The Browse API returns Buy It Now items by default; auction items require filtering by the `AUCTION` buying option. Rate limits exist but are not publicly documented per-tier — you'll need to monitor response headers.
- **Catch:** You need to apply for production access and describe your application. "Personal price monitoring tool" should be sufficient, but approval isn't instant. Start this application now, before you write any code.
- **Recommendation:** Use the Browse API for structured search + item detail retrieval. Supplement with scraping only for edge cases the API doesn't cover well (e.g., completed/sold listings for price history, if you want that data beyond what the API exposes).

#### Amazon — Product Advertising API (PA-API 5.0)
- **Cost:** Free, but with a significant catch.
- **Risk:** High complexity. You must be an Amazon Associate (affiliate) to access PA-API. Your API rate limit (TPS — transactions per second) starts at 1 TPS and only increases based on how much affiliate revenue you generate. If you don't drive sales through affiliate links, your rate limit stays at the minimum. Amazon can also revoke access if your Associates account is inactive (no qualifying purchases within ~180 days in some regions).
- **Catch:** This API is designed for affiliates building shopping sites, not personal price trackers. You'd need to maintain an active Associates account with periodic qualifying purchases to keep access. For a personal tool, this is friction-heavy.
- **Recommendation:** Skip PA-API for V1. Use Keepa instead for Amazon price tracking (see below), and scraping as a fallback for current price checks. PA-API adds operational overhead that doesn't match your use case.

#### Keepa — API
- **Cost:** Paid. The base Keepa subscription is €19/month (~$21). The API itself requires additional token-based purchases on top of the subscription — the base subscription only gives you 1 token/minute, which is extremely limited. Meaningful API access starts at higher tiers.
- **Risk:** Low technical risk — Keepa has excellent historical price data and the API is well-documented with a solid Python library (`keepa` on PyPI). The risk is purely cost.
- **Recommendation for V1:** Use Keepa's free browser extension and website for manual price history research. Don't pay for API access yet. Instead, build your own price history database by scraping Amazon product pages periodically (price, availability, seller). You'll accumulate your own history quickly. Revisit Keepa API if you need deep historical data you can't build yourself.

#### Reddit (r/buildapcsales, r/hardwareswap, r/appleswap) — API
- **Cost:** Free for personal, non-commercial use. 100 queries per minute (QPM) with OAuth authentication.
- **Risk:** Low for your use case. The free tier at 100 QPM is wildly more than you need. Checking 3 subreddits every few minutes would use maybe 5-10 requests per cycle. You're nowhere near the limits that killed Apollo (which was serving millions of users).
- **Catch:** You need to register a Reddit app (takes minutes), use OAuth 2.0 authentication, and set a unique User-Agent string. Reddit also requires that you delete cached content when the original is deleted. For a personal tool pulling recent posts, this is trivial to comply with.
- **Recommendation:** Use the Reddit API with raw httpx and a custom token manager (not PRAW — see Steering Committee notes). Script app type with password grant flow. The three subreddit adapters share a single rate limiter and auth session via SharedRateLimiterRegistry.

#### Newegg — No public buyer-facing API
- **Cost:** N/A — Newegg's API is seller/marketplace-focused. There is no public API for browsing products or retrieving prices as a buyer.
- **Risk:** Scraping required. Newegg has moderate anti-bot protections. The community (including PCPartPicker's own founder) has noted that getting pricing data from Newegg officially is very difficult.
- **Recommendation:** Scrape. Use respectful intervals (no more than 1 request per 5-10 seconds), rotate User-Agents, and cache aggressively. Newegg product pages are relatively stable in structure. If they block you, back off — this is a nice-to-have source, not critical.

#### PCPartPicker — No official API, Cloudflare-protected
- **Cost:** Free (scraping), but technically against their unstated preferences.
- **Risk:** Medium-high. PCPartPicker uses Cloudflare protection. Multiple unofficial Python libraries exist (`pypartpicker`, `pcpartpicker`), but they need headless browsers or JS rendering to bypass Cloudflare. This adds complexity and fragility.
- **Recommendation:** Use PCPartPicker as a read-only price comparison reference, not a primary data source. If scraping it proves too fragile, drop it — you'll have direct access to the underlying retailers (Amazon, Newegg, B&H, etc.) anyway. PCPartPicker's value is aggregation; if you're already monitoring the sources, you don't strictly need it.

#### Slickdeals — No official API
- **Cost:** Free (scraping) or use their RSS feeds.
- **Risk:** Low. Slickdeals has RSS feeds for deal categories that are much simpler than scraping. The RSS approach is lightweight, reliable, and requires no authentication.
- **Recommendation:** Start with RSS feeds. Slickdeals RSS gives you title, link, and basic deal info. Parse titles with the LLM for relevance to your wishlist. Only scrape individual deal pages if you need more detail than RSS provides.

#### Best Buy — Limited API
- **Cost:** Best Buy previously had a Products API that was free with an API key. Its current status is uncertain — they've been reducing public API access.
- **Risk:** Medium. May need scraping for Open Box deals specifically, as those are often store-specific and not well-represented in any API.
- **Recommendation:** Scrape the Open Box section for your local Microcenter-adjacent Best Buy. Open Box deals are inherently local and change frequently. Low-frequency checks (every 2-4 hours) are fine here.

#### B&H Photo, Adorama — No public API
- **Cost:** Free (scraping).
- **Risk:** Low-medium. Both sites have relatively stable product page structures. Neither is known for aggressive anti-bot measures at low request volumes.
- **Recommendation:** Scrape. These are supplementary sources — check every 4-6 hours.

#### Microcenter — No public API
- **Cost:** Free (scraping).
- **Risk:** Low. Microcenter's website shows in-store inventory and Open Box availability. Their pages are relatively simple to parse.
- **Recommendation:** Scrape. Since you live near one, this is high-value. Check every 2-4 hours for Open Box and clearance items matching your wishlist.

#### Woot — No public API (Amazon-owned)
- **Cost:** Free (scraping or RSS).
- **Risk:** Low. Woot has RSS feeds and relatively simple pages.
- **Recommendation:** RSS first, scrape if needed.

#### ServerMonkey, RefurbedUS, STH Forums — Niche sources
- **Cost:** Free (scraping).
- **Risk:** Low volume, low detection risk. These sites have minimal anti-bot measures.
- **Recommendation:** Scrape at low frequency (every 6-12 hours). These are long-tail sources where deals persist.

#### Mouser / DigiKey — Official APIs exist
- **Cost:** Free with registration. Both have developer programs with generous free tiers for product search and pricing.
- **Risk:** Very low. Both are developer-friendly.
- **Recommendation:** Use their APIs. DigiKey's API is particularly well-documented.

### Cost Summary

| Source | Method | Monetary Cost | Risk Level |
|--------|--------|---------------|------------|
| eBay | API | Free | Low (after approval) |
| Amazon | Scraping (skip PA-API) | Free | Medium |
| Keepa | Skip API for V1 | Free | N/A |
| Reddit (3 subs) | API | Free | Very Low |
| Newegg | Scraping | Free | Medium |
| PCPartPicker | Scraping (optional) | Free | Medium-High |
| Slickdeals | RSS + scraping | Free | Very Low |
| Best Buy | Scraping | Free | Medium |
| B&H / Adorama | Scraping | Free | Low |
| Microcenter | Scraping | Free | Low |
| Woot | RSS / scraping | Free | Very Low |
| ServerMonkey etc. | Scraping | Free | Very Low |
| Mouser / DigiKey | API | Free | Very Low |

**Total ongoing cost: $0/month for V1.**

The only potential future cost is Keepa API (~€19+/month) if you decide you want deep Amazon historical data you can't build yourself. Everything else is free APIs or lightweight scraping.

### Apollo-Style Risk Assessment

The Apollo scenario (Reddit killing third-party apps via API pricing) is not a realistic concern for your project because:

1. **You're not building a product.** Apollo was a commercial app serving millions. You're a single user running a personal tool over Tailscale. No platform considers you a threat.
2. **Your request volume is negligible.** Even monitoring all sources every 2-4 hours, you'd make maybe 200-500 requests per day total across all sources. That's invisible.
3. **You're diversified.** Apollo depended entirely on Reddit. You have 12+ sources. If any single source locks you out, you lose one data stream, not the whole tool.
4. **The real risk is scraping fragility, not policy changes.** Sites redesign their HTML. Cloudflare rules change. Selectors break. This is the maintenance burden you should plan for — not legal/policy risk.

---

## Session 2: Scope Review

### What's In Scope for V1

1. **Wishlist management** — CRUD for items with target prices, condition preferences, urgency tiers, keyword groups (OR between groups, AND within), and spec matching rules (`match_criteria`).
2. **Source adapters** — Pluggable scrapers/API clients for each data source. V1 ships with: eBay (API, scraper fallback), Reddit (API via raw httpx, 3 subreddits), Amazon (scraping), Slickdeals (RSS via feedparser), Newegg (scraping), Microcenter (scraping), Best Buy (scraping).
3. **Scheduling engine** — Custom scheduler loop using a next-check-at model with urgency tier multipliers. Runs on MacBook. Scheduling state stored in database (`source_health.next_scheduled_check`), not memory. Wakes PC via WOL when LLM scoring is needed.
4. **Worker-to-orchestrator internal API** — The scoring worker on the PC communicates exclusively via HTTP endpoints on the orchestrator (`/api/internal/scoring/*`). The worker does not access the database directly. Secured with a shared secret header.
5. **Price history database** — SQLite (WAL mode). Every price observation stored with timestamp, source, condition, listing type, availability. DatabaseWriter queue serializes all writes through a single async channel to prevent SQLite contention.
6. **LLM scoring pipeline** — Two-stage: fast rule-based pre-filter on MacBook (six checks: negative keywords, keyword match, confidence-aware price ceiling, condition minimum, listing status, duplicate detection) → full Qwen scoring on PC for candidates that pass.
7. **Auction state machine** — eBay-specific. Discovery → Watching → Approaching → Alert Window → Resolved. Dedicated monitor loop separate from main scheduler.
8. **Push notifications** — ntfy.sh on MacBook. Severity-tiered alerts with suppression rules (quiet hours, per-item cooldowns, duplicate detection). Action buttons for auction intent flagging.
9. **Dashboard** — React SPA served by FastAPI (HTM + CDN imports, no build tooling). Price history charts, active watches, deal log, alert history. Server-Sent Events (SSE) for near-real-time updates with polling fallback.

### What's Explicitly Out of Scope for V1

- **Auto-purchasing / auto-bidding.** V1 alerts you. You decide and act. This is a deliberate safety boundary — you don't want bugs buying things.
- **Multi-user support.** This is your personal tool.
- **Mobile app.** iPhone access is via Safari over Tailscale to the dashboard.
- **Keepa API integration.** Build your own price history first.
- **Tier C sources** (ServerMonkey, RefurbedUS, STH Forums, Mouser/DigiKey, Woot, Adorama, B&H). Added incrementally after V1 is stable.
- **PCPartPicker integration.** Only if scraping proves straightforward; otherwise deferred.
- **Machine learning on price predictions.** V1 uses threshold-based alerts, not predictive models.
- **Integration with the Inbox/Self-Knowledge Engine.** These are separate systems for now.
- **Config hot-reload.** Changing config requires orchestrator restart for V1.

### Scope Creep Risk Areas

These are the features that will tempt you during implementation. Flag them now so you recognize them later:

1. **"Let me just add one more source..."** — Each scraper takes 2-4 hours to build, test, and handle edge cases. Stick to the V1 source list.
2. **"The dashboard could show..."** — Resist building an elaborate analytics dashboard. V1 dashboard: active watches, recent deals found, alert history, one price chart per item. That's it.
3. **"What if I add auto-bid for eBay auctions?"** — This is V2 at the earliest. The liability of bugs here is real money.
4. **"I should make the scoring more sophisticated..."** — V1 scoring: does it match the item? Is the price below threshold? Is the condition acceptable? Is the seller reputable? That's four questions. Don't build GPT-4-level analysis for V1.

---

## Session 3: Steering Committee Review

### Architecture Decisions & Rationale

#### Decision 1: MacBook as orchestrator, PC as compute
- **Rationale:** The MacBook is always on. The PC sleeps. Scheduling, lightweight filtering, and notifications belong on the MacBook. Heavy LLM scoring belongs on the PC. This is the same topology as your WOL relay — extend it, don't fight it.
- **Risk:** The MacBook has 8GB RAM. It cannot run even a small LLM for pre-filtering. Pre-filtering must be rule-based (regex, keyword matching, price threshold checks) — not model-based.
- **Worker topology:** The scoring worker on the PC is a pure HTTP client loop — no server process, no open port, no database access. It pulls scoring batches from the orchestrator's internal API, sends them to Ollama, and pushes results back. Worker health is determined from periodic POST reports to the orchestrator (`/api/internal/scoring/worker-health`) and heartbeat freshness in the scoring queue — the orchestrator never pings the worker.
- **Migration path:** When the Mac Mini M4 Pro arrives (3-6 months), move the orchestrator there and potentially run a small model (e.g., Qwen 2.5 3B or 7B) on it for pre-filtering. The PC remains the heavy compute node.

#### Decision 2: SQLite over PostgreSQL
- **Rationale:** Consistency with the Inbox. Your data volume is small (hundreds of listings per day, not millions). SQLite handles concurrent reads fine and WAL mode handles the write patterns you'll have. No additional service to manage.
- **Write serialization:** All writes are serialized through a DatabaseWriter — an async queue that processes operations sequentially through a single connection. This handles the concurrency of multiple adapter completions, the notification worker, the auction monitor, and the maintenance worker all writing simultaneously. A separate DatabaseReader exploits WAL mode for concurrent read access.
- **Risk:** If you eventually want full-text search across listing descriptions, SQLite's FTS5 is decent but not as capable as PostgreSQL's. Acceptable tradeoff for V1.

#### Decision 3: Two-stage scoring (rule filter → LLM)
- **Rationale:** Most listings won't match your wishlist at all. A simple keyword + price filter eliminates 95%+ of results without waking the PC. Only candidates that pass the filter get queued for LLM scoring. This keeps the PC sleeping most of the time.
- **Example flow:** Slickdeals RSS returns 50 new deals. MacBook filter: does any deal title contain keywords from the wishlist? 3 match. Are any below the target price? 2 are. Queue those 2 for LLM scoring. Wake PC. Qwen evaluates condition, spec compatibility, seller reputation. Score > threshold → alert.

#### Decision 4: ntfy.sh for notifications
- **Rationale:** Free, open-source, single binary, no external dependencies, works over Tailscale. The iOS app costs $3.99 (one-time) or you can use the free web interface. Sending a notification is a single HTTP POST. You can also use the public ntfy.sh server initially and self-host later.
- **V1 approach:** Start with the public ntfy.sh server (ntfy.sh) with a long, hard-to-guess topic name — the topic URL is the sole authentication mechanism. Migrate to self-hosted when the Mac Mini arrives.
- **Action buttons:** ntfy.sh supports action buttons on notifications. Auction alerts include an "I'm on it" button that POSTs to `/api/auctions/{id}/flag-intent` on the orchestrator, setting `intent_flagged = true`. This enables auction resolution logic to distinguish "user planned to act" from "user ignored it" without requiring the user to open the dashboard.

#### Decision 5: Adapter pattern for sources
- **Rationale:** Each source is a Python class implementing a common interface: `search(wishlist_item) → AdapterResult`. The orchestrator doesn't know or care whether the adapter uses an API, RSS, or scraping. This lets you add sources without touching the core pipeline.
- **Interface sketch:**
  ```
  class SourceAdapter:
      name: str
      supports_auction_tracking: bool
      supports_detail_fetch: bool
      supports_price_filter: bool
      requires_auth: bool

      async def safe_search(self, query: WishlistQuery) -> AdapterResult
      async def get_listing_detail(self, external_id: str) -> RawListing | None
      async def health_check(self) -> HealthCheckResult
  ```
- **Reddit implementation:** Uses raw httpx with a custom `RedditTokenManager` (OAuth password grant, Script app type) — not PRAW. PRAW is synchronous and manages its own connection state in ways that conflict with the async architecture and shared resource management. Three Reddit adapters (r/buildapcsales, r/hardwareswap, r/appleswap) are separate config entries using the same `RedditSubredditAdapter` class. They share a single rate limiter and auth token via `SharedRateLimiterRegistry` and `SharedAuthRegistry`, keyed by `account_key` in config.

#### Decision 6: Auction state machine as a separate subsystem
- **Rationale:** Auctions have different lifecycle semantics than fixed-price listings. A fixed-price deal is either good or not. An auction goes through stages: discovered → watching → approaching → alert_window → resolved (won/lost/passed/pending_result). This needs its own monitor loop (separate from the main scheduler), its own polling intervals per state, and its own database table.
- **Risk:** This is the most complex piece of V1. If it's blocking progress, ship V1 without auction tracking and add it as V1.1.

### Open Questions for Steering

1. **eBay API approval timeline.** Apply immediately. If approval takes weeks, you may need to start with eBay scraping and migrate to the API later. The adapter is designed with a scraper fallback path.
2. **Qwen model adequacy.** Your 14B model works well for the Inbox's structured extraction. Deal scoring is a different task — it needs to evaluate condition descriptions, parse seller reputation signals, and make judgment calls about compatibility ("is this DDR5 kit compatible with my existing TEAMGROUP kit?"). Test early with real eBay listings to see if Qwen handles this or if you need a different/larger model.
3. **WOL reliability for scoring.** Currently, WOL → 20-second delay → Tailscale reconnects → ready. If the scoring queue has 5 items and the PC wakes, processes them, and goes back to sleep in 3 minutes — does the sleep/wake cycle cause any issues with Ollama cold starts? Test this.

---

## Session 4: Third-Party Audit — Output Alignment, Mechanics & Logic

### Output Alignment Check

**Question: Does V1 actually solve the user's problem?**

Your problem statement: "I want to know when hardware I'm looking for hits a good price, before someone else buys it."

V1 delivers:
- ✅ Monitors the sources where the items appear
- ✅ Filters for relevance using rules + LLM
- ✅ Alerts you via push notification with enough context to decide
- ✅ Tracks price history so you know if "good" is actually good
- ✅ Handles the eBay urgency problem (auctions + Buy Now)
- ✅ Runs autonomously on your existing hardware
- ✅ Scoring worker communicates via orchestrator HTTP API — no shared database access, no port allocation on PC

**Gaps identified:**
- ⚠️ The "is this DDR5 compatible with my existing kit" question requires either hardcoded compatibility rules or a model that understands RAM spec matching. This is a spec-matching problem, not just a price problem. **Recommendation:** For V1, hardcode compatibility requirements directly in the wishlist item's `match_criteria` JSON (e.g., "must contain 'TEAMGROUP T-Create Expert' or 'CTCCD232G6000HC30LDC01' in title or description, must be DDR5-6000 CL30, must be 2x32GB"). Let the LLM verify against `match_criteria` but don't rely on it to infer compatibility.
- ⚠️ The "form factor confusion" problem (OptiPlex Micro vs SFF) is common in eBay listings. Sellers frequently mislabel these. The LLM needs to look at the actual model number, not just the title. **Recommendation:** Include model number patterns in `match_criteria` (e.g., "3060 Micro model numbers: D10U, 3060M" or similar distinguishing identifiers). Research the actual model number patterns for the specific units you want.

### Mechanical Review

**Scheduling mechanics — potential issues:**

1. **Thundering herd on PC wake.** If 3 sources all produce candidates simultaneously, the PC wakes, processes them, and sleeps. Then 2 minutes later another source produces a candidate, waking the PC again. **Fix:** The DatabaseWriter queue serializes all write operations, preventing burst-induced contention. The WOL decision logic batches scoring: the orchestrator only wakes the PC when (a) the scoring queue has ≥5 items, OR (b) any item has waited ≥30 minutes, OR (c) a high-urgency item (auction ending soon) is in the queue. A 5-minute WOL cooldown prevents repeated wake cycles.

2. **Stale auction data.** You check an auction at T-10 minutes and the price is great. By T-5 minutes when you get the alert, the price has changed. **Fix:** For auctions in the alert window (<5 min remaining), the alert should include a direct link and the price *at time of check*, with a clear note that it may have changed. Don't present stale data as current. Follow-up alerts on significant bid changes (>10%) during alert_window, rate-limited to max one per 2 minutes.

3. **Rate limiting across adapters.** If you have 9 sources and each polls independently, you need per-source rate limiting. But adapters sharing the same API account (three Reddit subs) must share a single rate limiter via SharedRateLimiterRegistry. **Fix:** Per-adapter rate limiters configured in `config.yaml`, with `shared_rate_limits` for multi-adapter accounts. The scheduler staggers source checks with jitter. Random delay (0.5-2s) between scraper requests. Adaptive pacing: if rate limit headers show quota running low, slow down dynamically. Rate limit hits do NOT increment consecutive failures.

4. **Deduplication.** The same deal will appear on Slickdeals AND r/buildapcsales AND on the retailer directly. You need dedup at the listing level. **Fix:** Normalize URLs (strip tracking parameters). Unique constraint on `(wishlist_item_id, source, external_id)` in raw_listings. Cross-source dedup (recognizing the same Amazon ASIN from Slickdeals and direct scraping as the same item) deferred to V2.

### Logic Review

**Scoring pipeline logic:**

```
Source Adapter → Raw Listing → Pre-Filter (MacBook) → Scoring Queue → LLM Scoring (PC via internal API) → Scored Deal → Alert Tier Decision → Notification
```

**Pre-filter logic (MacBook, no LLM — six checks in order, cheapest first):**
1. **Negative keywords** — instant disqualification (e.g., "broken," "for parts").
2. **Keyword match** — at least one keyword group must match (OR between groups, AND within each group).
3. **Price ceiling** — confidence-aware buffer: exact prices get no buffer, parsed prices get 15% buffer, extracted prices get 30% buffer, unknown prices pass through for LLM evaluation.
4. **Condition minimum** — listing condition must meet or exceed the wishlist item's minimum (new > open_box > refurbished > used_like_new > used_good > used_fair).
5. **Listing status** — not ended, sold, or expired.
6. **Duplicate detection** — not already in raw_listings for this wishlist item + source + external_id.

If all pass → row in `scoring_queue` with priority based on urgency tier and pre-filter confidence.

**LLM scoring dimensions (PC, Qwen):**
1. **Spec match (0-100):** Does this listing actually match the wishlist item's specifications and `match_criteria`? Catches mislabeled items, wrong configurations, missing components.
2. **Condition assessment (0-100):** Based on description and seller claims — how confident are we in the item's condition?
3. **Value score (0-100):** Given the price relative to target and market rate — how good is this deal?
4. **Urgency multiplier (1.0-2.0x):** Derived from listing type and timing — auctions ending <5min get 2.0×, <30min get 1.5×, <60min get 1.25×, deal posts get 1.15×.
5. **Risk flags:** From a closed set: new_seller, no_returns, price_too_good, vague_description, wrong_item_possible, compatibility_uncertain, condition_mismatch.

**Final score = (Spec × weight_spec + Condition × weight_condition + Value × weight_value) × Urgency**

Weights are configurable in `config.yaml` (default: spec 0.4, condition 0.3, value 0.3).

Alert thresholds (with hard overrides):
- Score ≥ 80: High priority push notification
- Score 60-79: Normal push notification
- Score 40-59: Low priority — logged in dashboard, no push
- Score < 40: None — logged for debugging
- **Override:** spec_score < 30 → always "none" (wrong item, never alert)
- **Override:** condition_mismatch on strict items → cap at "low"
- **Override:** 2+ risk flags → cap at "low"
- **Override:** price_too_good → cap at "normal"

---

## Session 5: PM Checklist — Things We Haven't Discussed

### 1. Data Retention & Cleanup
- How long do you keep raw listing data? Suggestion: 30 days for raw listings, forever for price history observations (they're small — timestamp, item ID, source, price, condition). 7 days for scoring queue entries. Implement a maintenance worker that handles cleanup, market summary generation, and pause_until reactivation.

### 2. Error Handling & Monitoring
- Scrapers break. You need to know when they break, not discover it three weeks later when you realize you haven't gotten Newegg results.
- **Recommendation:** Each adapter reports its last successful run time. The dashboard shows a "source health" panel. Per-adapter `source_health` table tracks consecutive failures, consecutive empty runs (for silent failure detection), response times, rate limit state, and backoff status. If any source hasn't successfully returned data in 2× its poll interval, flag it. Push a low-priority ntfy.sh alert: "Newegg adapter hasn't returned data in 8 hours."
- Transient errors (timeouts, 503s) need more occurrences (10+) before escalating to "down." Non-transient errors (403s, parse failures) escalate faster (3+).

### 3. Configuration Management
- Wishlist items, source configs, polling intervals, alert thresholds — where do these live?
- **Recommendation:** Three-layer config system:
  - `config.yaml` (committed to Git) — all tunable parameters: source settings, poll intervals, rate limits, scoring weights, alert thresholds, notification rules, scraping defaults. References env var *names* for secrets, never values.
  - `local.yaml` (gitignored) — machine-specific overrides: Tailscale IPs, MAC address, database path, port assignments. Deep-merged over config.yaml at load time.
  - `.env` (gitignored, loaded via python-dotenv) — secrets only: API keys, passwords, ntfy.sh topic, internal API shared secret.
- SQLite for wishlist items (because you'll CRUD these from the dashboard).

### 4. Logging
- You'll need structured logging (JSON) from the start. When debugging why a deal wasn't caught, you need to trace: source returned listing → pre-filter result → LLM score → alert decision. Without structured logs, this is hell.
- **Recommendation:** Python `structlog` library. Log every pipeline stage with the listing ID as a correlation key. JSON format for machine-parseable output.
- **Log redaction:** A structlog processor scrubs tokens and secrets from all log output — both pattern-based (bearer tokens, OAuth signatures) and value-based (known secret values loaded from config's `logging.secret_env_vars` list). Tokens never appear in logs, even in error tracebacks.

### 5. Testing Strategy
- You can't easily test scrapers against live sites in CI. But you can save sample HTML responses and test your parsers against those.
- **Recommendation:** For each adapter, save 5-10 real response samples (HTML or API JSON) as test fixtures in `tests/fixtures/{source}/`. Write parser tests against these. When a scraper breaks in production, save the new HTML as a failing test case, then fix. The base class provides `load_fixture()` and `capture_fixture()` helpers. Integration tests marked separately (`pytest -m integration`). Use `freezegun` for time-sensitive tests (auctions, cooldowns, scheduling). Track `expected_field_rates` per adapter to detect gradual field deprecation.

### 6. Backfill Strategy
- When you add a new source adapter, you have no price history from it. Consider building a one-time backfill mode that pulls historical data where available (eBay completed listings API, Keepa free charts).

### 7. The "I Was Asleep" Problem
- A great deal appears at 3 AM. Your system detects it, scores it, and sends an alert. You're asleep. You wake up at 7 AM, the deal is gone. Was the system useful?
- **Options:** (a) Accept this as a known limitation — you're a human, not a bot. (b) For specific high-urgency items, set up escalating alerts (first notification → if unacknowledged in 10 min → louder notification). (c) For V2, consider auto-purchase for very specific, low-risk items (e.g., the M5Stack Atom Echo at $15 — the downside of a mistake is $15).
- **V1 approach:** Quiet hours (23:00-07:00) defer normal/low alerts to morning delivery. Critical alerts bypass quiet hours and ring through.

### 8. Prompt Engineering Lessons from the Inbox
- You learned that Qwen needs concrete examples, not abstract rules. Apply this to deal scoring: include 3 example listings in the scoring prompt with your desired scores and reasoning (strong match ~85, partial match ~55, poor match ~25). This is your most transferable lesson.
- You learned that validation rules belong in Python, not prompts. Apply this: the LLM scores and describes. Python enforces thresholds, formats alerts, and makes go/no-go decisions. The prompt instructs "respond with ONLY this JSON, no other text" — and the brace-depth JSON extractor handles it when the LLM doesn't listen.

### 9. Development Milestones

| Milestone | Description | Est. Time |
|-----------|-------------|-----------|
| M0 | Foundation: schema, config (3-layer), logging (with redaction), DatabaseWriter, utilities | 2-3 days |
| M1 | Reddit adapter end-to-end: adapter base class, Reddit via raw httpx, pre-filter (6 checks), basic dashboard with wishlist CRUD | 4-5 days |
| M2 | Scheduling & source health: scheduler loop with urgency multipliers, source_health tracking, backoff, crash recovery | 3-4 days |
| M3 | LLM scoring pipeline: worker-to-orchestrator internal API, scoring worker, Ollama client, prompt, brace-depth parser, WOL logic | 4-5 days |
| M4 | Notifications: ntfy.sh client, 5 alert formatters, suppression rules (quiet hours, cooldown, dedup), retry logic | 2-3 days |
| M5 | Amazon (scraper), Slickdeals (RSS via feedparser), Newegg (scraper) adapters | 4-5 days |
| M6 | Microcenter (scraper), Best Buy (scraper), eBay (API or scraper fallback) adapters | 4-5 days |
| M7 | Auction state machine: auto-watch, state transitions, alert window with intent flagging, resolution | 4-5 days |
| M8 | Price history, market summaries, dashboard polish, Recharts charts, SSE for real-time updates | 3-4 days |
| M9 | Hardening and dogfooding: memory profiling, service configs (launchd + NSSM), log rotation, bug fixes | 5-7 days |
| **Total** | | **~35-46 days** |

This assumes part-time effort with learning curve. Your actual pace may differ based on how much time you spend per day and how quickly you solve scraping edge cases.

### 10. Technology Stack (Confirmed)

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Language | Python 3.13 | Consistency with Inbox |
| Web framework | FastAPI | Consistency with Inbox |
| Database | SQLite (WAL mode) + aiosqlite | Consistency with Inbox, appropriate for scale |
| Write serialization | DatabaseWriter (async queue) | SQLite single-writer constraint |
| Scheduler | Custom loop (next-check-at model) | No external dependency, state in database |
| LLM | Qwen 2.5 14B via Ollama | Already running, proven |
| Notifications | ntfy.sh (public server initially) | Free, simple, good iOS support, action buttons |
| Dashboard | React SPA (HTM + CDN, no build tooling) | Consistency with Inbox |
| Real-time updates | Server-Sent Events (SSE) | Simpler than WebSockets, auto-reconnect, polling fallback |
| HTTP client | httpx (AsyncClient) | Modern async support, all external + inter-machine HTTP |
| HTML parsing | beautifulsoup4 + lxml | Industry standard for scraping |
| RSS parsing | feedparser | Lightweight, reliable |
| Reddit client | Raw httpx + RedditTokenManager | Async-compatible, shared rate limiting, no PRAW overhead |
| eBay client | Direct REST via httpx | More control than third-party wrappers |
| Validation | Pydantic v2 (AwareDatetime) | Strict timestamp validation, type coercion |
| Service management | NSSM (Windows), launchd (macOS) | Consistency with Inbox |
| Logging | structlog (JSON, secret redaction) | Structured logs with credential scrubbing |
| Config | YAML (config.yaml + local.yaml) + .env (python-dotenv) | Three-layer: committed defaults, machine-specific, secrets |
| Testing | pytest, pytest-asyncio, freezegun | Fixture-based parser tests, time-sensitive test support |

### 11. What Could Save Us Time Later (Things to Get Right Now)

1. **Apply for eBay API production access today.** Approval can take days to weeks. Don't let this block M6.
2. **Register a Reddit app today.** Takes 5 minutes. Get your OAuth credentials ready. Script app type for password grant flow.
3. **Define the data model carefully at M0.** Every adapter produces a `RawListing` that normalizes across sources. If this model is wrong, every adapter needs to change. Spend extra time here. Single `price` with `price_confidence`, not separate columns.
4. **Build the adapter interface before any specific adapter.** Write the abstract base class with `safe_search()` wrapper, write a mock adapter that returns fake data, build the pipeline end-to-end with fake data, THEN start building real adapters. This prevents "works with Reddit but breaks when I add Amazon" surprises.
5. **Set up structured logging from day one.** It's 10 minutes of setup now vs. hours of "why didn't this deal trigger an alert" debugging later. Include the log redaction processor from the start — retrofitting secret scrubbing is error-prone.
6. **Create a `tests/fixtures/` directory from M1.** Every time you successfully scrape a page or get an API response, save a copy. These become your regression tests.
7. **Set up the virtual environment and reference `.venv/bin/python` in launchd/NSSM configs from day one.** This prevents the "works in my shell but the service can't find the module" class of bugs.
8. **Build the DatabaseWriter queue early (M0).** Retrofitting serialized writes after the fact is painful — every module that does a database write needs to be refactored to go through the queue. Get it right at the foundation layer.

---

## Next Steps

This document is your architecture review record. The following documents complete the V1 design package:

- **V1 Technical Specification** — 9 sections with code patterns, data models, and testing approaches for each component.
- **Database Schema Reference (v1 final)** — Authoritative table definitions for all 10 tables.
- **Config Templates** — `config.yaml`, `local.yaml.example`, `.env.example` ready for use.
- **Implementation Handoff Brief** — Concise start-here guide for new implementation chats.
- **Retrospective & Risk Register** — Corrections, risks, V2 parking lot, reusable patterns.

When you're ready to start implementation, hand the relevant spec section + schema doc + handoff brief to a Sonnet coding session, starting with M0.
