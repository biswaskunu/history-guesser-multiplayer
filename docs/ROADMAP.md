# Roadmap

## Phase 1 — Solo parity (Java replaces Rust)
- [ ] Netty HTTP + Jackson + CORS + `/health`, `/db-check`
- [ ] `Geo` port + `GeoTest` vectors
- [ ] HikariCP + Flyway + `EventRepository` + `/events/random`, `/guess`
- [ ] Docker Compose (app + pg) + solo play via existing frontend

## Phase 2 — Lobby core
- [ ] `netty-handler` WS upgrade at `/ws`, `LobbyHandler`, `RoomManager`, `Room(LOBBY)`
- [ ] `CREATE/JOIN/PLAYER_JOINED`, name uniqueness, 2–8 cap, TTL eviction
- [ ] `RoomTest`: join/leave/host-migrate

## Phase 3 — Timed rounds (authoritative)
- [ ] `ROUND_START/SUBMIT_GUESS/PLAYER_GUESSED/ROUND_END/GAME_OVER`, timer + early-end, idempotent finish
- [ ] Leak test (no `actual*` in `ROUND_START`), late/double reject, disconnect grace
- [ ] Intermission + `NEXT_ROUND`, Play Again

## Phase 4 — Frontend MP
- [ ] Lobby / Room / Standings screens, `useRoomSocket`, multi-pin reveal, countdown from `deadlineMs`

## Phase 5 — Harden & ship
- [ ] Rate limits, frame caps, `PING/PONG`, structured logs, Railway Dockerfile, Netlify `VITE_API_URL`, Worker WS-forward check
- [ ] Soak test: 8 players × 5 rounds

## Later (out of v1)
- Match history tables, persistent leaderboards, auth, public matchmaking, async challenges, admin content pipeline, Redis scale-out, chat.
