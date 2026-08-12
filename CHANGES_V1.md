# Orbifon — v1.0.0 readiness pass

Rust compiler/network access is not available in the environment this
review was done in, so these fixes come from a full manual line-by-line
read of every file, not from `cargo build`. **Run `cargo build` yourself
before deploying** and paste any remaining errors back — this pass should
have caught everything a compiler would flag, but a real build is the
only way to be 100% sure.

## Compile-breaking bugs fixed
- `routes.rs` pointed `/api/hot-town/channels/:id/messages` POST at a
  function `create_message` that didn't exist — the real name is
  `post_message`.
- `AppError` had no `From<serde_json::Error>` / `From<SendError<T>>`
  impls, so every `?` on a `serde_json::to_string(...)` or `tx.send(...)`
  inside a websocket handler would have failed to compile. Added both.
- `src/models/pagination.rs` existed but `models.rs` never declared
  `pub mod pagination;` — the file was never compiled, so `PaginationQuery`
  (used by `profiles.rs`) didn't exist. Declared + re-exported it.
- `src/content_safety.rs` existed but was never listed in `main.rs`'s
  `mod` block — the whole file was dead, uncompiled code. Added
  `mod content_safety;`.
- `audit_log.rs` binds a `serde_json::Value` into a `JSON` column, but
  `Cargo.toml`'s `sqlx` dependency didn't have the `"json"` feature
  enabled — added it.
- `ws/tests.rs` existed but `ws/mod.rs` never declared it — the unit test
  in that file was never actually run. Added `#[cfg(test)] mod tests;`.

## Runtime/logic bugs fixed
- **Biggest one:** every new WebSocket connection built its own fresh
  `WsState::new()` instead of sharing one. That meant two different
  users' sockets lived in completely separate in-memory registries — DMs
  and Hot Town chat would never actually reach anyone. `WsState` now
  lives once in `AppState` and is cloned (it's `Arc`-backed internally)
  into every connection.
- DM socket cleanup on disconnect built the channel key with
  `format!("dm:{}:{}", min(id, 0), max(id, 0))` — since `id` is unsigned,
  that always evaluates to `"dm:0:{id}"`, which never matches the key the
  connection actually registered under. Disconnect cleanup silently did
  nothing. Now the real channel key is captured at `connect` time and
  reused at `close` time.

## Feature gaps closed (things `MODERATION_GUIDE.md` documents as
## working, but the code never actually called)
- Rate limiting wasn't wired into any handler. Added
  `RateLimiter::check_action` to post creation (10/hr), comments (30/hr),
  and voting (100/hr), matching the documented limits.
- Hashtag/mention extraction (`tags::extract_hashtags` /
  `store_post_hashtags`, same for mentions) was never called from
  `create_post` — hashtag search and trending hashtags would always be
  empty. Now wired in.
- `ContentSafety::check_content` was never called anywhere. Now runs on
  new posts/comments; flagged content is still published (not blocked)
  but logged via `tracing::warn!` and recorded in `audit_logs`.
- The reputation system (`+5`/post-upvote, `-20`/actioned report,
  `-50`/suspension) never inserted a single row into
  `reputation_events`, so `users.reputation_score` was always 0. Wired
  events into `vote_post` (post upvotes) and `review_report` (actioned
  reports / suspensions), and the denormalized `reputation_score` column
  is now refreshed after each event.

## Cosmetic
- Version strings were inconsistent (`Cargo.toml` said 0.2.0, the
  startup log said 0.3.0, `/api/stats` said 0.3.0). Unified to `1.0.0`.

## Known, intentionally-not-fixed gap
- `PubSubManager::publish` is called, but nothing ever calls
  `PubSubManager::subscribe` to consume it — so cross-server Redis
  fan-out doesn't actually forward messages between multiple app
  instances yet. For a single-instance deployment (which this
  Gwalior-pilot app is) this doesn't matter, since local `WsState`
  broadcast now works correctly. Flagging it so it's not a surprise if
  you scale to multiple servers later.

## Before you deploy
1. `cargo build` — confirm it compiles clean.
2. Set up `.env` from `.env.example` (`DATABASE_URL`, `JWT_SECRET` ≥32
   chars, etc).
3. Run migrations (`sqlx::migrate!` runs automatically on boot, reading
   `./migrations`).
4. If you want Redis-backed WS caching/rate limiting, set `REDIS_URL`;
   otherwise it falls back to single-server in-memory mode automatically.
