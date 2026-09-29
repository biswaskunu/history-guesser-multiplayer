# History Guesser Java Multiplayer — PRD

## 1. Overview
Port https://github.com/biswaskunu/history-guesser (Rust + Axum + React + Leaflet) to Java with authoritative multiplayer support. Same core fantasy: "shown a historical photo, drop a pin, scored on closeness" — now playable together in private party lobbies.

Live reference: https://history-guesser-game.netlify.app

## 2. Goals
1. Full Java port of solo game (no Rust runtime dependency).
2. Party-lobby multiplayer for 2–8 players sharing the same event per round.
3. Reuse React + Leaflet frontend (add lobby / room screens only).
4. Stay on raw Netty (Java 21), no Spring Boot.
5. Authoritative server: clients render, server decides.

## 3. Non-goals (v1)
- Public matchmaking queue, ELO, leaderboards across games.
- Async challenge links / play-by-mail.
- New JavaFX/Swing client.
- Content pipeline / admin UI for adding events.
- Auth / JWT (anonymous names in rooms).
- Horizontal scaling (single JVM is fine).

## 4. Users
- Players: history buffs playing solo or with friends.
- Hosts: create room, share code, configure rounds/timer, start/next.
- Future: admins adding events (out of scope).

## 5. Functional requirements

### 5.1 Solo parity (must match Rust behavior)
- `GET /events/random?difficulty=1..5&exclude=id1,id2` → `PublicEvent{id,image_url,title,description,year,difficulty}`.
  - Invalid difficulty → 400. No match → fallback ignoring `exclude`, else 404.
- `POST /events/{id}/guess {latitude,longitude}` → `{distance_km (2dp), score, actual_latitude, actual_longitude, year}`.
  - Out-of-range lat/lng → 400. Unknown id → 404.
- Scoring: `haversine_km` (R=6371.0). `score = 5000` if `dist <= 1km`, else `round(5000 * exp(-dist/2000))`.
- Client-side (unchanged): total, streak (consecutive rounds ≥3000), best (localStorage), verdict tiers (4500/3000/1000).

### 5.2 Multiplayer — party lobbies (v1 scope)
- Create room → 4–6 char code (e.g. `KQ7P`). Join with display name (unique per room).
- Capacity 2–8. Empty rooms expire after TTL (e.g. 30 min).
- Host config (locked after start): `totalRounds` default 5 (1–10), `timerSec` default 30 (10–120), `difficulty` optional 1–5.
- Same `PublicEvent` broadcast to all players per round. Answer lat/lng never sent before reveal.
- One guess per player per round. Late / duplicate guesses rejected.
- Round ends on all-guessed or timer expiry → reveal with per-player distance/score + actual location.
- Game ends after N rounds → standings sorted by total, winner highlighted.
- Live presence: join/leave feed, `PLAYER_GUESSED count/total` (no coords leaked).
- Reconnect: grace window (~60s) to rejoin with same name; else marked left.

### 5.3 Frontend (React reuse)
- New screens only: Home (+ Create/Join), Lobby (code, roster, host settings), Room (reuse `GameMap` + `RoundHistory`), GameOver podium.
- Room map: own pin + on reveal show all pins color-coded + actual marker.
- WS hook with reconnect + countdown timer from `deadlineMs`.

## 6. Non-functional requirements
- Java 21, Maven, raw Netty 4.2.x.
- Latency: guess ack < 200ms LAN, reveal fan-out < 500ms for 8 players.
- Correctness: no answer leak (tested), timer race-free, no EventLoop blocking on JDBC.
- Observability: `/health`, `/db-check`, structured logs.
- Config via env: `PORT`, `DATABASE_URL`, `FRONTEND_ORIGIN` (CSV, same semantics as Rust).
- Deploy: Docker Compose (app + Postgres) locally; Railway-style (Dockerfile + `/health`) + Netlify frontend via `VITE_API_URL`. Note: Cloudflare Worker proxy needs WS-forwarding for `/ws`.

## 7. Data
Reuse Rust schema: `events(id UUID PK, image_url TEXT, title TEXT, description TEXT, latitude DOUBLE, longitude DOUBLE, year INT, difficulty SMALLINT 1–5)`. Flyway reuses existing migration SQL. No new tables required for v1 rooms (in-memory); persist match history later.

## 8. Success criteria
- Solo flow playable against Java backend with zero frontend scoring changes.
- 8-player room completes 5 rounds without desync or leak.
- Timer expiry and all-guessed early-end both work.
- `GeoTest` (haversine + scoring vectors) + `RoomTest` (state machine) green.

## 9. Open decisions (locked defaults in parens)
- Room code length (5), max players (8), rounds (5), timer (30s), intermission (10s, host-skippable).
- Difficulty per-game vs per-round (per-game v1).
- Chat in rooms (no v1, only presence feed).
