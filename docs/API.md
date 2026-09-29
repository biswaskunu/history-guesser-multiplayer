# API — REST + WebSocket

Base: `http://localhost:8000`. JSON only. Errors: `{ "error": "..." }`.

## REST (solo parity)

### `GET /health` → `200 { "status": "ok" }`
### `GET /db-check` → `200 { "db": "ok" }` or 500
### `GET /events/random?difficulty=&exclude=`
- `difficulty` optional `1..5`, `exclude` optional CSV UUIDs (bad entries ignored).
- `200 PublicEvent { id, image_url, title, description, year, difficulty }`
- `400 {error}` bad difficulty · `404 {error}` no events found
### `POST /events/{id}/guess`
- Body `{ latitude: -90..90, longitude: -180..180 }`
- `200 { distance_km (2dp), score, actual_latitude, actual_longitude, year }`
- `400` range · `404` unknown id

## WebSocket `/ws?room=CODE&name=NAME`
Text frames, JSON with `type`. Max frame 64KB.

### Client → Server
| type | fields | notes |
|---|---|---|
| `JOIN` | `room, name` | alternative to query params; name 2–20 chars |
| `CREATE` | `name, totalRounds?, timerSec?, difficulty?` | server returns `CREATED` |
| `START_GAME` | `rounds?, timerSec?, difficulty?` | host only, state must be LOBBY |
| `SUBMIT_GUESS` | `lat, lng` | one per round, GUESSING only, before deadline |
| `NEXT_ROUND` | — | host only, skips intermission |
| `LEAVE` | — | graceful leave |
| `PONG` | — | heartbeat reply |

### Server → Clients
| type | fields |
|---|---|
| `CREATED` | `room` |
| `PLAYER_JOINED / PLAYER_LEFT / HOST_CHANGED` | `players[{name,total,connected}], count` |
| `ROUND_START` | `round, totalRounds, event:PublicEvent, deadlineMs (epoch)` |
| `GUESS_ACK` | `ok:true` (unicast) |
| `PLAYER_GUESSED` | `count, total` (no coords) |
| `ROUND_END` | `round, actual{latitude,longitude,year}, results[{name,distanceKm,score,total,guessed}]` |
| `GAME_OVER` | `standings[{name,total}], winner` |
| `ERROR` | `message` |
| `PING` | — |

### Example
```json
// C→S
{"type":"SUBMIT_GUESS","lat":48.8584,"lng":2.2945}
// S→* (reveal)
{"type":"ROUND_END","round":1,"actual":{"latitude":48.8584,"longitude":2.2945,"year":1889},
 "results":[{"name":"A","distanceKm":0.5,"score":5000,"total":5000,"guessed":true}]}
```

### Rules
- Answer lat/lng appear only in `ROUND_END`. `ROUND_START` must never contain `actual*`.
- Late/duplicate/out-of-state guesses → unicast `ERROR`, no broadcast.
- Join mid-round → gets `PLAYER_JOINED` + waits for next `ROUND_START`.
