# Architecture

## 1. System overview
Single Java monolith (raw Netty) serving both REST (solo parity) and WebSocket (rooms). React frontend is a dumb renderer. Postgres stores events. Rooms are in-memory.

```
Browser (React + Leaflet)
  |-- REST :8000 /health /events/random /events/{id}/guess
  |-- WS   :8000 /ws?room=CODE&name=NAME
Netty (boss + worker EventLoopGroups)
  |-- HttpServerCodec + HttpObjectAggregator (64KB) + CORS
  |-- RouterHandler -> HealthHandler / EventsHandler
  |-- WebSocketServerProtocolHandler (/ws) -> LobbyHandler
RoomManager (ConcurrentHashMap<code, Room>)
EventRepository (HikariCP) <-> Postgres (events)
```

## 2. Tech choices (locked)
- Java 21, Maven, Netty 4.2.17 (`common, buffer, transport, codec-http` present; must add `handler, codec` for WS).
- Jackson `databind` for JSON, HikariCP + `postgresql` driver, Flyway for migrations, `slf4j + logback`, JUnit 5 + Mockito + Awaitility.
- Virtual threads / bounded executor for JDBC so EventLoop never blocks.

## 3. Package layout (target)
```
com.historyguesser/
  Main.java            // env load, Flyway migrate, start Server
  Server.java          // bootstrap pipeline, bind PORT
  config/AppConfig.java// PORT, DATABASE_URL, FRONTEND_ORIGIN CSV
  model/Event.java, PublicEvent.java, GuessRequest.java, GuessResponse.java
  util/Geo.java        // haversine_km, score_from_distance (port of models/events.rs)
  repo/EventRepository.java // randomEvent(difficulty, exclude UUIDs), findById
  service/ScoringService.java, RoomManager.java, Room.java
  http/RouterHandler.java, EventsHandler.java, HealthHandler.java, CorsUtil.java, JsonUtil.java
  ws/LobbyHandler.java, RoomMessages.java (DTOs), WsUtil.java
```

## 4. HTTP pipeline
`LoggingHandler? -> HttpServerCodec -> HttpObjectAggregator -> CorsUtil -> RouterHandler`
- Router matches method+path, delegates; unknown → 404 JSON.
- `EventsHandler`: validates `difficulty 1..5`, parses `exclude` CSV silently dropping bad UUIDs (Rust parity), rounds `distance_km` to 2dp.
- Errors always `{ "error": "..." }` with 400/404/500.

## 5. WebSocket design
- Upgrade at `/ws`, then `LobbyHandler extends SimpleChannelInboundHandler<TextWebSocketFrame>`.
- Auth: query params `room` + `name` (2–20 chars, unique per room). No JWT v1.
- Per-room `ChannelGroup (DefaultChannelGroup + GlobalEventExecutor)` for fan-out.
- Inbound JSON has `type` field; Jackson polymorphic DTOs in `RoomMessages`.
- Outbound broadcast helpers: `broadcast(room, msg)` excluding/including sender as needed.
- Heartbeats: server `PING` every 25s, client `PONG`; also WS `PingWebSocketFrame` from Netty idle handler.

## 6. Room state machine
```
LOBBY --(host START_GAME)--> GUESSING --(all guessed | timer)--> REVEAL --(host NEXT / auto)--> GUESSING ... --(round==total)--> FINISHED
```
- `Room` fields: `code, hostId, players{id,name,channelId,total,connected}, state, round, totalRounds, timerSec, difficulty, currentEvent(full), guesses{playerId->Guess}, playedIds[], timerFuture, deadlineMs`.
- All mutations via `synchronized` per-room methods; timers via `ctx.executor().schedule()`.
- `ROUND_START`: repo fetch (off EventLoop), set deadline, broadcast `PublicEvent` only.
- `SUBMIT_GUESS`: validate range + state==GUESSING + not-duplicate + before deadline; store; broadcast `PLAYER_GUESSED` count.
- `finishRound()`: cancel timer (idempotent), compute each `Geo.score`, broadcast `ROUND_END` with actual + results.
- Disconnect: mark `connected=false`, keep 60s grace; `channelInactive` triggers leave broadcast; empty room → `RoomManager.remove` + cancel timer.

## 7. Data
- `events` table reused verbatim from Rust migrations. Sample row needed for dev seed.
- v1 rooms not persisted. Future: `matches, match_players, round_guesses` tables.
- `exclude` list = room `playedIds` so no repeats per game; fallback to ignore-exclude when exhausted (Rust parity).

## 8. Concurrency & safety
- Never call JDBC inside EventLoop; wrap in `CompletableFuture.supplyAsync(dbExecutor)` then hop back to event loop for broadcast.
- Timer vs last-guess race: `finishRound` guarded by `if (state != GUESSING) return`.
- Answer leak prevention: `PublicEvent` DTO has no lat/lng fields at type level; test asserts serialized `ROUND_START` contains no `actual_*`.
- Backpressure: 8 players/room trivial; cap rooms e.g. 1000 + LRU eviction; max frame 64KB.

## 9. Config & deploy
- Env: `PORT (8000), DATABASE_URL, FRONTEND_ORIGIN (CSV, trim trailing /, default https://history-guesser.netlify.app)`.
- Local: `docker compose up` (postgres:5432, java:8000, frontend:5173). Prod: Railway Dockerfile + `/health` check; Netlify `VITE_API_URL` → Worker proxy URL with WS forwarding enabled.

## 10. Testing strategy
- `GeoTest`: haversine vectors (same city ~0, antipodes ~20015km), scoring (0km→5000, 1km→5000, 2000km→~1839, huge→~0).
- `RoomTest`: join/start/guess/reveal/standings, late-guess reject, timer path (Awaitility), leak assert.
- Manual: 2 browsers same code, host/guest flow, timer expiry, refresh-reconnect.
