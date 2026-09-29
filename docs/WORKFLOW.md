# Workflow

## 1. Game workflow — solo (parity)
1. Client `GET /events/random?difficulty=&exclude=` → renders image/title/desc/year chip.
2. Player clicks Leaflet map → local pin (`handleGuess`).
3. Submit → `POST /events/{id}/guess` → result panel (distance, year, score, verdict).
4. History appends `{round,title,year,distance_km,score}`; totals/streak/best update; Next → step 1 with `exclude += id`.

Verdicts: `>=4500 "touch grass", >=3000 "Close guess" (+streak), >=1000 "In the region", else "missed"`.

## 2. Game workflow — party lobby (v1)
1. **Create:** Host → server generates `CODE`, creates `Room(LOBBY)`, joins as host. Shares code out-of-band.
2. **Join:** Guests send `JOIN {room, name}` → `PLAYER_JOINED` roster to all. Duplicate name → `ERROR`.
3. **Configure:** Host sets `rounds/timerSec/difficulty` (Lobby UI only, not broadcast until start).
4. **Start:** Host `START_GAME` → server validates ≥2 players → `ROUND_START {round, totalRounds, event:PublicEvent, deadlineMs}` to all.
5. **Guess:** Each client sends one `SUBMIT_GUESS {lat,lng}` → server acks (unicast `GUESS_ACK`) + broadcasts `PLAYER_GUESSED {count,total}`. Map locks locally after submit.
6. **Reveal:** All-guessed or `deadlineMs` passes → `ROUND_END {actual{lat,lng,year}, results[{name,distanceKm,score,total,guessed?}]}`. Clients draw all pins + actual + standings snippet. Missing guess = `guessed:false, score:0`.
7. **Next:** 10s auto-countdown (host `NEXT_ROUND` skips) → `ROUND_START` for round+1 with new event (excludes `playedIds`).
8. **End:** After N rounds → `GAME_OVER {standings desc, winner}` → Lobby offers Play Again (new `playedIds=[]`) or Leave.

## 3. Edge cases
- Guest joins mid-round → spectates until next `ROUND_START` (gets roster + round number, no event answer).
- Host leaves → migrate host to oldest remaining player, broadcast `HOST_CHANGED`.
- Disconnect <60s → auto-rejoin same slot; >60s or explicit leave → removed, totals kept for finished rounds.
- Timer fires as last guess arrives → `finishRound` idempotent guard wins, single `ROUND_END`.
- No events for difficulty → `ERROR "no events found"` in lobby, game stays in LOBBY.

## 4. Dev workflow
1. `docker compose up --build` → pg:5432, java:8000, web:5173. Backend reads `.env` (`DATABASE_URL, FRONTEND_ORIGIN, PORT`).
2. Solo check: open web, play 2 rounds, confirm scoring matches Rust vectors.
3. Multi check: open 2 browsers (`?room=CODE` or two tabs), create+join, play full game.
4. Backend dev loop: `mvn compile exec:java` or IDE run `Main`; `mvn test` must stay green (Geo + Room).
5. Migrations: edit `src/main/resources/db/migration/V*.sql` (Flyway), never hand-edit prod DB.

## 5. Message sequence (happy path, 3 players)
```
H -> S: CREATE → S -> H: CREATED {room:ABCDE}
G1,G2 -> S: JOIN → S -> *: PLAYER_JOINED {players:[H,G1,G2]}
H -> S: START_GAME {rounds:5,timer:30} → S -> *: ROUND_START {event, deadline}
G1 -> S: SUBMIT → S -> *: PLAYER_GUESSED {1/3} → ... {3/3}
S -> *: ROUND_END {actual, results} → (10s) → S -> *: ROUND_START {round 2}
... → S -> *: GAME_OVER {standings}
```

## 6. Frontend state mapping
`LOBBY → GUESSING (pin enabled) → SUBMITTED (locked, "waiting for X") → REVEAL (pins+scores) → GAME_OVER (podium)`. Countdown derived from `deadlineMs - Date.now()`. On `onclose`, show Reconnecting + retry with same room/name.
