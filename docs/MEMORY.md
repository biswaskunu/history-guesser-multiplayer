# Memory — decisions & context

This file is the project's long-term memory. Update it when a decision changes so future work stays consistent.

## 1. Source of truth
- Original game: https://github.com/biswaskunu/history-guesser — Rust/Axum backend (`main.rs`, `routes/events.rs`, `models/events.rs`), React 19 + Leaflet frontend (`App.jsx`), Postgres, Railway + Netlify + Cloudflare Worker proxy.
- Scoring port must stay identical: `R=6371.0`, `5000 pts <=1km`, `5000*exp(-d/2000)` above. Don't "improve" curve without playtest note.

## 2. Locked decisions (2026-09-15)
- Full Java port (replace Rust, not sidecar).
- First multiplayer = private party lobbies (room code, 2–8), not matchmaking or async.
- Client = reuse React+Leaflet, add lobby/room screens.
- Server = raw Netty on Java 21 (not Spring Boot / Javalin / Quarkus). Accept manual routing/WS cost for control + learning.
- Rooms in-memory v1; only `events` table persisted.
- Defaults: code 5 chars, 8 max, 5 rounds, 30s timer, 10s reveal, difficulty per-game.

## 3. Key behaviors copied from Rust (do not regress)
- `exclude` CSV silently drops bad UUIDs; fallback ignores exclude when exhausted.
- `difficulty` validated 1..5 → 400 otherwise.
- Guess lat/lng range-checked → 400; unknown event → 404.
- `distance_km` rounded to 2dp.
- CORS origins from `FRONTEND_ORIGIN` CSV, trailing slashes trimmed.
- Migrations embedded/run at startup (Flyway replaces `sqlx::migrate!`).

## 4. Gotchas learned
- Cloudflare Worker proxy in original repo breaks WS unless forwarding enabled — must verify for `/ws`.
- Netty `pom.xml` currently lacks `netty-handler`/`codec` (needed for WS) and all JSON/DB deps.
- `src/main/java/com/game.java` + `test.java` are empty placeholders — replace with `com.historyguesser` packages.
- Frontend streak rule (`>=3000`) matches "Close guess" tier — keep linked.

## 5. Conventions
- Authoritative server: never trust client score/distance; recompute.
- `PublicEvent` type must not contain lat/lng fields (leak-proof by construction).
- Per-room `synchronized` + idempotent `finishRound`; JDBC off EventLoop.
- Error shape: `{ "error": "..." }` (REST) / `{ "type":"ERROR", "message":"..." }` (WS).

## 6. TODO when resuming
- Check `docs/ROADMAP.md` for next phase; update this file when defaults change.
