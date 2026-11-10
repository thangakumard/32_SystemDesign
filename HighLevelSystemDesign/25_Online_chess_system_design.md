# Online Chess (Chess.com / Lichess-style): System Design

> Staff-level interview guide. Scale anchor: **10M DAU, 500K concurrent games (1M+ persistent WebSockets)**.
> Numbers marked *(illustrative)* are reasonable assumptions for interview math, not published figures.

---

## Table of Contents

1. [Functional Requirements](#1-functional-requirements)
2. [Non-functional Requirements](#2-non-functional-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [Core Entities](#4-core-entities)
5. [API / System Interface](#5-api--system-interface)
6. [Data Flow](#6-data-flow)
7. [High-Level Design](#7-high-level-design)
8. [Deep Dives (4 NFRs)](#8-deep-dives-4-nfrs)
9. [Interview Questions: Talk Tracks (90–120 s)](#9-interview-questions-talk-tracks-90120-s)
10. [CAP / PACELC per Component](#10-cap--pacelc-per-component)
11. [Failure Modes & Mitigations](#11-failure-modes--mitigations)
12. [Scalability & Sharding Summary](#12-scalability--sharding-summary)
13. [Senior vs. Staff Differentiators](#13-senior-vs-staff-differentiators)
14. [45-Minute Interview Time Allocation](#14-45-minute-interview-time-allocation)
15. [Quick-Reference Cheatsheet](#15-quick-reference-cheatsheet)
16. [Related Guides](#16-related-guides)

---

## 1. Functional Requirements

**Core (in scope)**
- **FR1 – Real-time play:** Two players play a game with legal-move enforcement, live clocks (bullet/blitz/rapid/classical), check/mate/stalemate/draw detection, resign, draw offer, abort.
- **FR2 – Matchmaking:** Player picks a time control; system pairs them with an opponent of similar rating within seconds.
- **FR3 – Ratings & leaderboard:** Ratings (Glicko-2) update after each rated game; global and per-time-control leaderboards; "my rank".
- **FR4 – Spectate & history:** Watch live games; replay finished games (PGN with clock times).
- **FR5 – Reconnect:** A player who drops can reconnect and resume with exact board + clock state.

**Secondary (mention, then de-scope)**
- Challenge a friend / private link, in-game chat, tournaments, puzzles, anti-cheat (async engine analysis), correspondence chess.

---

## 2. Non-functional Requirements

| NFR | Target | Why it matters |
|---|---|---|
| **Low latency** | Move relay p99 < 100 ms in-region (server processing < 5 ms) | Bullet chess (1+0) is unplayable with jitter |
| **Clock fairness** | Neither player loses time to their own network lag beyond a bounded amount | Core trust of the product |
| **Strong per-game consistency** | Single authoritative game state; no forked boards, no double moves | Correctness beats availability *per game* |
| **Durability of finished games** | Zero lost results/rating updates | Ratings are the user's "currency" |
| **High availability** | 99.95% for play; game survives a server crash with < ~5 s disruption | 500K live games = big blast radius |
| **Scalability** | 500K concurrent games, 1M+ WebSockets, peaks 2–3× | Evening/weekend spikes, streamer events |
| **Fair matchmaking** | Median wait < 5 s for popular pools; bounded rating gap | Retention driver |
| **Eventual consistency OK** | Leaderboard, history, spectator feed (seconds of lag fine) | Lets those tiers be AP and cheap |

---

## 3. Capacity Estimation

*(Decimal/SI units; illustrative assumptions.)*

**Traffic**
- 10M DAU × ~5 games/day ÷ 2 players/game ≈ **25M games/day** → ~290 game starts/s avg, **~1K/s peak**.
- Matchmaking joins: 10M × 5 ÷ 86,400 ≈ 580/s avg, **~2–3K/s peak**.
- Concurrent: **500K games → 1M player WebSockets**, plus spectators (say +0.5M sockets, heavy-tailed: a top streamer game may have 100K viewers).

**Moves**
- Avg game ≈ 80 plies over ~10 min (blitz-heavy mix) → one ply every ~7.5 s per game.
- 500K ÷ 7.5 s ≈ **67K moves/s avg**, **~150–200K moves/s peak** (cluster-wide, client-facing).
- Internal fan-out per move: 1 to opponent + 1 persist event + N spectators → internal events ≈ 3–5× client moves.

**Memory (game servers)**
- Live game state (board, move list, clocks, metadata) ≈ 5–10 KB → 500K × 10 KB ≈ **5 GB total**. Memory is *not* the bottleneck; **connections and per-message CPU/syscalls** are.

**Connections / nodes**
- WebSocket gateway at ~50K active sockets/node (conservative; tuned boxes do 100K+) → 1.5M sockets ÷ 50K ≈ **30 nodes + 50% headroom ≈ 45**.
- Game servers at ~25K games/node → 20 nodes; run **~40** for headroom and failover.

**Storage**
- Finished game (PGN + clock per move + metadata, compressed) ≈ 2 KB → 25M × 2 KB = **50 GB/day ≈ 18 TB/year** (before replication).
- Move event log (Kafka): 25M × 80 × ~100 B ≈ **200 GB/day**, 3–7 day retention.
- Ratings: ~100M registered users × few time controls × ~100 B ≈ tens of GB — fits in Redis/Postgres comfortably.

**Matchmaking queue size (Little's Law)** — the key insight
- L = λ × W = 3K/s × ~3 s ≈ **~10K players waiting globally**. The queue is tiny; the challenge is *correctness (no double pairing) and fairness*, not volume.

---

## 4. Core Entities

| Entity | Key fields | Store |
|---|---|---|
| **User** | user_id, username, country, created_at, flags | Postgres |
| **Rating** | user_id, time_control, rating, rd (deviation), volatility, games_played, last_game_at, version | Postgres (source of truth) + Redis ZSET (rank) |
| **Game** | game_id (Snowflake), white_id, black_id, time_control, rated, status, result, termination, start/end, owner_server, epoch | Live: game server memory + Redis/Kafka log. Final: Cassandra/DynamoDB |
| **Move** | game_id, ply, uci (e.g. `e2e4`), fen_after (optional), client_think_ms, server_ts, white_clock_ms, black_clock_ms | Kafka (live log) → embedded in final PGN |
| **Seek / Queue Ticket** | ticket_id, user_id, time_control, rating, rd, region, joined_at, constraints | Redis / matcher memory |
| **Session / Presence** | user_id, gateway_id, connection_id, last_seen | Redis (TTL) |
| **Game Directory entry** | game_id → (game_server_id, epoch) | Redis / etcd-backed shard map |

---

## 5. API / System Interface

**REST (stateless, via API gateway)**
```
POST   /v1/matchmaking/tickets          {timeControl:"3+2", rated:true}   → 202 {ticketId}
DELETE /v1/matchmaking/tickets/{id}                                       → 204
GET    /v1/games/{gameId}                                                 → game snapshot / PGN
GET    /v1/users/{userId}/games?cursor=…                                  → history (paginated)
GET    /v1/leaderboard?tc=blitz&offset=0&limit=50                         → top-N
GET    /v1/leaderboard/rank?tc=blitz&userId=…                             → {rank, rating, percentile}
POST   /v1/challenges  {opponentId, timeControl}                          → challenge link
```

**WebSocket (stateful) — `wss://play.example.com/ws?token=…`**
```
Client → Server
  {type:"move",   gameId, ply, uci:"e2e4", clientThinkMs:2310, seq:41}
  {type:"resign" | "offerDraw" | "acceptDraw" | "abort", gameId}
  {type:"resume", gameId, lastSeenPly:40}
  {type:"ping",   t:clientMonotonicTs}
  {type:"watch",  gameId}

Server → Client
  {type:"matchFound", gameId, color, opponent, timeControl}
  {type:"move",   gameId, ply, uci, whiteMs, blackMs, serverTs}
  {type:"ack",    gameId, ply, seq}
  {type:"reject", gameId, ply, reason:"illegal|notYourTurn|stalePly"}
  {type:"gameEnd",gameId, result:"1-0", termination:"timeout|mate|resign"}
  {type:"pong",   t}
```
- `ply` + `seq` make moves **idempotent** (retransmits after reconnect don't double-apply).
- Server always sends **authoritative clock values**; client clocks are cosmetic interpolation.

---

## 6. Data Flow

**A. Find a game**
1. Client `POST /tickets` → Matchmaking Service writes ticket into pool `(timeControl, region)`.
2. Matcher (single writer per pool) pairs two tickets atomically.
3. Matcher asks Game Allocator for a game server (least-loaded / consistent hash on `game_id`), creates game, registers `game_id → server, epoch` in the directory.
4. `matchFound` pushed to both players via Notification/Gateway (looked up via presence).

**B. Play a move**
1. Client sends `move` over WebSocket → Gateway → routed by `game_id` to owning Game Server.
2. Game Server: check turn + ply, validate legality (bitboard move-gen, µs), update clock with lag compensation.
3. **Append move to durable log (Kafka/Redis Stream) → ack** (write-ahead before broadcast).
4. Broadcast to opponent's gateway (and to spectator fan-out topic).

**C. Game ends**
1. Game Server detects mate/flag/resign → emits `GameEnded{game_id, result}` (idempotent key = game_id).
2. Rating Service consumes → Glicko-2 update in Postgres transaction (`UNIQUE(game_id)` guard) → updates Redis ZSET.
3. Archiver writes final PGN to Cassandra; Anti-cheat consumes asynchronously.

---

## 7. High-Level Design

### 7.1 Architecture diagram

```
                           ┌──────────────┐
   Players / Spectators    │   CDN / LB   │  (GeoDNS → nearest region; L4 LB for WS)
   (web, iOS, Android) ───▶│ (TLS, L4/L7) │
                           └──────┬───────┘
              REST (stateless)    │      WebSocket (stateful)
          ┌───────────────────────┴───────────────────────┐
          ▼                                               ▼
  ┌───────────────┐                              ┌──────────────────┐
  │  API Gateway  │                              │  WS Gateway Fleet│  holds 1M+ sockets,
  │ (auth, rate   │                              │  (~45 nodes)     │  no game logic
  │  limit)       │                              └───┬──────────┬───┘
  └──┬─────┬────┬─┘                                  │ route by │ presence
     │     │    │                                    │ game_id  ▼
     │     │    │                                    │     ┌───────────┐
     │     │    │                                    │     │Redis      │ session/presence,
     │     │    │                                    │     │(presence, │ game directory
     │     │    │                                    │     │ directory)│
     │     │    │                                    ▼     └───────────┘
     │     │    │       ┌──────────────┐   create  ┌────────────────────────┐
     │     │    └──────▶│ Matchmaking  │──────────▶│  Game Server Cluster   │
     │     │   tickets  │ Service      │  game     │  (stateful, ~40 nodes, │
     │     │            │ (matcher per │           │  sharded by game_id,   │
     │     │            │  pool, leader│           │  in-memory board+clock,│
     │     │            │  elected)    │           │  timer wheel for flags)│
     │     │            └──────┬───────┘           └───┬───────────┬────────┘
     │     │                   │ pools (Redis ZSET)    │ append    │ spectator
     │     │                   ▼                       ▼ move      ▼ fan-out
     │     │            ┌──────────────┐        ┌──────────────┐  ┌──────────────┐
     │     │            │ Redis (pools)│        │ Kafka        │  │ Spectator    │
     │     │            └──────────────┘        │ moves,       │  │ Fan-out tier │
     │     │                                    │ game-events  │  │ (pub/sub,    │
     │     │                                    └──┬────┬────┬─┘  │  edge relays)│
     │     │                       ┌───────────────┘    │    └──┐ └──────────────┘
     │     │                       ▼                    ▼       ▼
     │     │              ┌────────────────┐  ┌──────────────┐ ┌──────────────┐
     │     │              │ Rating Service │  │ Archiver     │ │ Anti-cheat   │
     │     │              │ (Glicko-2,     │  │ (PGN writer) │ │ (async engine│
     │     │              │  idempotent)   │  └──────┬───────┘ │  analysis)   │
     │     │              └───┬─────────┬──┘         ▼         └──────────────┘
     │     │                  ▼         ▼     ┌──────────────┐
     │     └─────────────▶┌────────┐ ┌───────┐│ Cassandra /  │
     │       leaderboard  │Postgres│ │Redis  ││ DynamoDB     │
     │                    │ratings,│ │ZSETs  ││ (game history│
     └──────────────────▶ │users   │ │(rank) ││  by user_id) │
         history          └────────┘ └───────┘└──────────────┘
```

### 7.2 Why each component

- **WS Gateway separate from Game Server:** connections churn (mobile networks) far more than games do; separating lets you deploy game logic without dropping 1M sockets, and scale sockets vs. CPU independently. *Trade-off:* one extra hop (~1 ms in-DC).
- **Stateful Game Server with single owner per game:** a chess game is a tiny, strictly ordered state machine — a single writer gives linearizability for free, no distributed locks per move.
- **Kafka move log:** write-ahead for crash recovery + decouples rating/archive/anti-cheat/spectators from the hot path.
- **Redis (pools, presence, directory, ZSET leaderboard):** sub-ms ops; data is reconstructible.
- **Postgres for ratings:** transactional, idempotent rating updates (`UNIQUE game_id`); ratings are money-like.
- **Cassandra/DynamoDB for history:** write-heavy (50 GB/day), partition by `user_id`, cluster by `ended_at DESC` → "my last 20 games" is a single-partition read.

### 7.3 Four functional requirements in detail (90–120 s each)

#### FR1 — Real-time play
- Both players hold a WebSocket to the **nearest-region gateway**; gateway looks up `game_id → game server` in the directory (cached locally) and forwards over a persistent gRPC stream.
- Game server is the **single authoritative owner** of the game: checks turn, `ply` number, legality via bitboard move generator (~10–50 µs).
- Clock: server stamps receive time, applies **bounded lag compensation**, deducts from mover's clock, starts opponent's clock.
- **Write-ahead:** append move to Kafka (partitioned by `game_id`, `acks=all`, `min.insync.replicas=2`) **before** acking and broadcasting — a crash after broadcast can't lose a move the opponent already saw.
- Broadcast `{move, whiteMs, blackMs}` to opponent; client renders and interpolates clocks locally.
- **Flag detection is server-side** with a hierarchical timer wheel — the server ends the game when time hits zero even if the flagging player's client is offline.
- Idempotency: duplicate `ply` → re-send ack; stale `ply` → reject with current state.
- *Alternatives:* polling/long-poll (simple, but 100s of ms latency — unacceptable for bullet); WebRTC P2P (lowest latency, but no authority → trivially cheatable, NAT pain). **WebSocket + server authority wins.**

#### FR2 — Matchmaking
- `POST /tickets` puts the player into a pool keyed by **(time control, rated, region)** — ~30–50 pools.
- Each pool has **one leader-elected matcher** (lease in etcd) holding the pool in memory, backed by a Redis ZSET (score = rating) for recovery.
- Every ~100–250 ms tick: sort by rating, greedily pair nearest neighbours whose **acceptable windows overlap**; window = `base ± k·RD` and **widens with wait time** (e.g. +25 points/s, capped).
- Pairing is atomic in the single writer → **no double-pairing**; ticket removal + game creation is committed with an idempotency key (`ticket_a:ticket_b`).
- Then allocate game server, write directory entry, push `matchFound` via presence lookup. If a player has disconnected, re-queue the other at the **front** with their original wait time.
- *Alternatives:* Redis Lua atomic `ZRANGEBYSCORE + ZREM` with many stateless workers (no leader, but contention and harder global optimality); lobby/seek list (Lichess's manual seeks — good UX for some, doesn't scale as the only path).

#### FR3 — Ratings & leaderboard
- Game end emits `GameEnded` to Kafka; **Rating Service** consumes, computes Glicko-2 for both players in one Postgres transaction guarded by `UNIQUE(game_id)` → exactly-once effect despite at-least-once delivery.
- Optimistic concurrency (`version` column) handles a player finishing two games concurrently (e.g. simul).
- After commit, `ZADD lb:{tc} rating user_id` in Redis (outbox pattern so Redis can't diverge silently).
- Top-N: `ZREVRANGE 0 49` — cached at CDN/app layer for 5–30 s. "My rank": `ZREVRANK` O(log n).
- Only **active, established** players are ranked (RD < threshold, played in last N days) — keeps the set ~10–20M, not 100M.
- See the Leaderboard guide for sharding the ZSET (score-range shards) vs. approximate ranks via percentile buckets.

#### FR4 — Spectate & history
- Spectators **never connect to the game server** — the game server publishes each move once to a pub/sub topic (Kafka/Redis pub/sub/NATS) keyed by `game_id`.
- **Spectator fan-out tier** subscribes per watched game and pushes to viewers; for a 100K-viewer streamer game, use a **tree of edge relays** (one subscription per relay, relays fan out locally) and a small deliberate delay (anti-cheat: prevents a "helper" relaying positions).
- New spectator gets a snapshot (FEN + clocks) then the delta stream.
- History: Archiver consumes `GameEnded` + move log → writes compressed PGN to Cassandra twice (partition by each player's `user_id`) plus a `game_id`-keyed row; old games tier to object storage (S3) after N months.

---

## 8. Deep Dives (4 NFRs)

### 8.1 NFR — Low latency (p99 move relay < 100 ms)
- **Regional deployment:** GeoDNS/anycast to nearest region; matchmaking prefers same-region pairs; cross-region games place the game server **near the midpoint** or in the region of the higher-latency player's opponent — pick consistently and compensate clocks.
- **Hot path is in-memory only:** validate + update clock + Kafka append + forward. No DB reads per move.
- **Kafka append latency:** batch with `linger.ms` ~1–2 ms; partition by `game_id`; `acks=all` adds ~2–5 ms in-DC — acceptable. *Alternative:* Redis Stream with AOF `everysec` (faster, weaker durability) — fine for live log if final result is durable elsewhere.
- **Compact protocol:** UCI strings or binary frames (~20 B) instead of verbose JSON; permessage-deflate off for tiny frames (CPU > savings).
- **Avoid head-of-line blocking:** one gateway→game-server gRPC stream per gateway pair, multiplexed; per-game ordering preserved by `game_id` affinity.
- **GC & tail latency:** JVM with ZGC/Shenandoah or Go; avoid per-move allocations; pin timer wheel to dedicated threads.
- **Measure:** client RTT pings every 2–5 s feed both the latency dashboard and clock compensation.

### 8.2 NFR — Clock fairness despite network latency
- **Problem:** if the server only uses its own receive time, a player with 300 ms RTT loses ~150 ms per move of thinking time they never had — over 40 moves that's 6 s, decisive in bullet.
- **Option A – Pure server time:** simple, cheat-proof, unfair to high-latency players. ❌ for bullet.
- **Option B – Trust client-reported time:** perfectly fair for honest players, trivially cheatable (client claims 0.1 s every move). ❌
- **Option C – RTT/2 compensation:** subtract measured half-RTT from each move. Better, but jitter and asymmetric paths make it noisy; still exploitable by inflating ping replies.
- **Option D (chosen) – Hybrid, bounded lag compensation** (similar in spirit to Lichess):
  - Server records `T_sent` when opponent's move is sent to player; receives move at `T_recv` → `server_elapsed = T_recv − T_sent`.
  - Client reports `clientThinkMs` using its **monotonic** clock from render-of-opponent-move to submit.
  - `lag = server_elapsed − clientThinkMs` (the claimed network time).
  - **Credit = min(lag, perMoveCap (~0.5–1 s), quotaBucket)** — a per-player **token bucket** that refills slowly; persistent liars quickly drain it.
  - Deduct `server_elapsed − credit` from the mover's clock.
- **Sanity checks:** `clientThinkMs` can't exceed `server_elapsed`; claimed lag compared to measured ping distribution; outliers flagged to anti-cheat.
- **Flag authority stays on the server:** the server ends the game at zero using its own clock (plus any pending credit window, e.g. waits one max-lag grace period before declaring flag if a move may be in flight).
- **Clients show interpolated clocks** synced to server values on every message, so displays don't drift.
- **Increments/delays** (Fischer/Bronstein) applied server-side after the deduction.

### 8.3 NFR — Scalability: 500K games, 1M sockets, stateful servers
- **Split the stateful problems:** sockets on gateways (I/O-bound, epoll/netty, ~50–100K each); game state on game servers (CPU-light, memory-light). Scale each on its own metric.
- **Shard by `game_id`:** both players' messages land on the same owner → no cross-node coordination per move.
- **Placement options:**
  - *Consistent hashing on `game_id`* over the live server set (membership via etcd/ZK) — no directory needed, but membership changes move games.
  - *Directory/allocator (chosen)* — allocator picks least-loaded server at creation, writes `game_id → (server, epoch)` in Redis/etcd; games **never move** unless their server fails or drains. Better load balancing, explicit control.
  - *Actor framework* (Akka Cluster Sharding / Orleans virtual actors) — gives placement, rebalancing, passivation out of the box; trade-off is framework lock-in and opaque failure behaviour.
- **Deploys without killing games:** mark server draining → allocator stops sending new games → existing games finish (≤ ~1 h classical) or are **live-migrated** (snapshot + replay log tail on new owner, bump epoch, gateways re-route).
- **Autoscale** on games-per-node and socket count, with pre-warming before known peaks (evenings, big streams).
- **Spectators are a separate tier** — a celebrity game must not hot-spot its game server (classic hot-key pattern; same class as celebrity fan-out in News Feed).

### 8.4 NFR — Availability & durability (no lost games/results)
- **Move-level durability:** each move appended to Kafka (`acks=all` + `min.insync.replicas=2`, RF=3) before ack.
- **Game server crash:** health checks / lease expiry detect within ~2–3 s → allocator assigns game to another node → new owner **replays the move log** for that `game_id` (≤ ~100 events) → state rebuilt.
- **Split brain / zombie owner:** directory entry carries an **epoch (fencing token)**; new owner increments it; Kafka writes and gateway routing reject the older epoch, so a paused "zombie" server can't accept moves.
- **Clocks during failover:** pause both clocks during the server-side outage window (server's fault, not players') and resume — announce it to clients.
- **Gateway crash:** clients auto-reconnect (exponential backoff + jitter) to any gateway, send `resume{gameId, lastSeenPly}`, get the delta. Player's clock keeps running if *they* disconnect (their fault) with a reconnection grace before abandonment claim.
- **Result durability:** `GameEnded` in Kafka → Rating Service idempotent on `game_id` → Postgres (synchronous replica in-region). Archive retries are safe (upsert by `game_id`).
- **Region failure:** live games in that region are lost-in-progress (adjudicate/abort, no rating change); new games route elsewhere. Full cross-region replication of live games isn't worth the latency cost — say this explicitly.

---

## 9. Interview Questions: Talk Tracks (90–120 s)

### Q1. How would you design the system to support real-time chess gameplay between two players?
- **Transport:** persistent **WebSocket** per player to a regional gateway; polling is too slow, WebRTC P2P has no authority and is cheatable.
- **Authority:** one **game server owns each game** in memory — board, move list, clocks. Single writer = strict ordering without locks.
- **Move path:** client sends `{gameId, ply, uci, clientThinkMs}` → gateway routes by `game_id` → server checks turn & ply, validates legality with bitboards (µs).
- **Durability before broadcast:** append to Kafka partitioned by `game_id` (`acks=all`, `min.insync.replicas=2`), then ack mover and push move + authoritative clocks to opponent.
- **Clocks:** server-authoritative with bounded lag compensation; flags detected server-side by a timer wheel, even if a client vanishes.
- **Idempotency:** `ply` number dedupes retransmits after reconnect; stale ply → reject with current state.
- **Reconnect:** `resume{lastSeenPly}` returns missed moves + current clocks.
- **Failure:** server crash → reassign via directory, replay log, fence old owner with an epoch.
- **Decoupled consumers:** game-end event drives ratings, archive, anti-cheat asynchronously — the hot path stays tiny.

### Q2. How would you design the matchmaking system to pair players for a game?
- Player creates a **ticket**: user, time control, rated flag, rating, rating deviation, region.
- Tickets go into a **pool per (time control, rated, region)** — Redis ZSET scored by rating.
- A **matcher loop** runs every ~100–250 ms: sort by rating, pair nearest neighbours whose acceptance windows overlap.
- **Window widens with wait time** (start ~±50, grow ~25/s, cap) — trades match quality for wait time, tunable per pool.
- **Atomic claim:** remove both tickets and create the game in one step (single matcher per pool, or a Redis Lua script) so nobody gets paired twice.
- **Allocate** a game server, register in directory, push `matchFound` via presence.
- **Edge cases:** cancel races (ticket removed mid-match → re-queue partner at front), disconnected winner of the pairing, color balancing, avoid immediate rematch with same opponent.
- **Alternatives:** seek lobby (manual) for casual users; ELO-only vs Glicko (use RD so new/uncertain players get wider windows).

### Q3. How would you design the leaderboard system to show global player rankings?
- **Source of truth:** ratings in **Postgres**, updated idempotently from `GameEnded` events (`UNIQUE(game_id)`).
- **Serving:** Redis **sorted set per time control** — `ZADD lb:blitz rating user`; top-N via `ZREVRANGE`, my rank via `ZREVRANK` in O(log n).
- **Only rank active, established players** (low RD, recent game) → ~10–20M members, fits in one Redis primary (~1–2 GB).
- **Top-100 cached** 10–30 s at app/CDN — the most-hit query becomes nearly free.
- **Consistency:** outbox pattern Postgres → Redis; leaderboard is **AP / eventually consistent** — seconds of lag is fine.
- **Scaling beyond one node:** shard by **score range** (top-N stays one shard, rank = sum of higher shards' counts) vs. hash sharding (balanced writes but top-N becomes a scatter-gather). The **sharding choice decides which query fans out** — that's the key trade-off.
- **Approximation option:** for "you're in the top 3%", use a rating histogram (fixed buckets) — O(1) and cheap; exact rank only for top 10K.
- **Extras:** country/friends leaderboards as separate small ZSETs or computed on read for friends lists.

### Q4. How would you design matchmaking to fairly pair players at scale (10M DAU, 500K concurrent games)?
- **Size it first:** ~3K joins/s peak × ~3 s wait ≈ **~10K waiting players** (Little's Law). Volume is small; the hard parts are **correctness, fairness, hot pools**.
- **Partition by pool** (time control × rated × region) → ~30–50 independent pools; blitz is the hottest.
- **Single writer per pool:** leader-elected matcher (etcd lease, fencing token) holds the pool in memory, snapshotted to Redis. No double pairing, globally better pairings than racing workers.
- **Hot pool split:** if one pool outgrows a node, sub-partition by **rating band with overlapping edges** so boundary players aren't starved.
- **Fairness policy:** windows based on rating ± k·RD, widening with wait; **bounded maximum wait** fallback (bot or wider/cross-region pool); color alternation; avoid recent opponents; protect new players from strong ones (provisional pool).
- **Integrity:** detect sandbagging (deliberate losses), rating-floor abuse, and queue dodging (penalty timeouts for repeated declines/aborts).
- **Throughput:** matcher work is O(n log n) per tick on a few thousand tickets — trivially fast.
- **Failure:** matcher leader dies → new leader loads ZSET snapshot within seconds; idempotent pairing key prevents duplicate games on retry.
- **Metrics:** wait-time percentiles per rating band, rating-gap distribution, abort rate after match — tune parameters per pool.

### Q5. How would you scale the game servers to handle 500K concurrent games with two persistent WebSockets each?
- **Separate connections from state:** WebSocket **gateways** hold ~1.5M sockets (~50–100K/node, ~45 nodes); **game servers** hold game state (~25K games/node, ~40 nodes). Different bottlenecks, scaled independently.
- Memory is small (~5 GB total) — the real limits are **socket count, per-message CPU, and syscalls**; use event-loop I/O (Netty/epoll), raise FD limits, tune kernel buffers.
- **Placement:** allocator assigns each new game to a least-loaded server and writes `game_id → (server, epoch)` to a directory; gateways cache it. (Alternatives: consistent hashing; Akka/Orleans actors.)
- **Both players → same owner**, so no cross-node coordination per move.
- **Recovery:** moves are in Kafka; on crash the game is reassigned and replayed in < ~3–5 s, old owner fenced by epoch, clocks paused for the outage.
- **Deploys:** drain servers (no new games, let current ones finish or live-migrate via snapshot + log tail); gateways deploy independently with client auto-reconnect + jitter to avoid thundering herd.
- **Spectators off the game servers:** separate pub/sub fan-out tier with edge relays for celebrity games.
- **Autoscale** on games/node and sockets/node; pre-warm before peaks; multi-region with geo-routing.

### Q6. How would you ensure chess clock fairness despite varying network latency?
- **Server is the clock authority** — clients only display interpolated time and resync on every message.
- **The issue:** server-only timing charges a 300 ms-RTT player for network transit every move.
- **Two measurements per move:** server elapsed (sent opponent's move → received reply) and client-reported think time from its **monotonic** clock.
- **Difference = claimed lag.** Credit back `min(lag, per-move cap ~0.5–1 s, token-bucket quota)`.
- **Token bucket** refills slowly, so an honest high-latency player is compensated, but a client lying about think time drains its quota fast.
- **Validation:** think time can't exceed server elapsed; compare claimed lag with measured ping history; anomalies → anti-cheat.
- **Flagging** decided server-side, with a short grace equal to max allowed lag before declaring timeout, so an in-flight move isn't unfairly lost.
- **Rejected options:** pure server time (unfair), pure client time (cheatable), raw RTT/2 (noisy, spoofable).
- **Structural help:** pair players in the same region and host the game near both, minimizing the lag that needs compensating.

---

## 10. CAP / PACELC per Component

| Component | Choice | Rationale |
|---|---|---|
| Live game state | **CP / EC** (single owner, fenced) | Two divergent boards is catastrophic; brief unavailability on failover is acceptable |
| Matchmaking pairing | **CP** (single writer per pool) | Double pairing is worse than a 2 s delay |
| Rating updates | **CP** (Postgres txn, idempotent) | Money-like; exactly-once effect |
| Move log (Kafka) | CP within ISR | Durability contract = `acks=all` + `min.insync.replicas` |
| Leaderboard | **AP / EL** | Seconds-stale rank is fine |
| Game history | AP / EL | Replay availability > immediate consistency |
| Presence / spectator feed | AP / EL | Best effort; small staleness OK |

Most of the system is AP; the **three CP islands are live game state, pairing, and ratings**.

---

## 11. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Game server crash | Its ~25K games freeze | Lease expiry → reassign → replay Kafka log; pause clocks; epoch fencing |
| Zombie owner (GC pause) | Two owners accept moves | Epoch/fencing token checked by gateways and on log append |
| Gateway crash | ~50K sockets drop | Client reconnect with jitter; `resume{lastSeenPly}`; stateless gateways |
| Reconnect thundering herd | Load spike after outage | Exponential backoff + jitter; connection admission rate limit |
| Kafka partition leader loss | Move append stalls | RF=3, `min.insync.replicas=2`; short retry; clocks paused if stall > threshold |
| Matcher leader failover | Pairing paused seconds | etcd lease, snapshot in Redis, idempotent pairing key |
| Duplicate `GameEnded` | Double rating change | `UNIQUE(game_id)` in rating txn |
| Player disconnects mid-game | Opponent waits | Mover's clock runs; reconnect grace; opponent can claim win after timeout |
| Celebrity game (100K viewers) | Hot spot | Spectator fan-out tier + relay tree + broadcast delay |
| Clock manipulation | Unfair advantage | Capped token-bucket lag credit; anomaly → anti-cheat |
| Engine cheating | Integrity | Async analysis of move log (centipawn loss, move-time patterns); out of hot path |
| Region outage | Live games lost | Abort/adjudicate without rating change; geo-failover for new games |

---

## 12. Scalability & Sharding Summary

| Data / Service | Shard key | Notes |
|---|---|---|
| Game servers | `game_id` (via allocator directory) | Games don't move except on failure/drain |
| Kafka `moves` | `game_id` | Per-game ordering |
| Kafka `game-events` | `game_id` | Rating service idempotent on it |
| Matchmaking | pool = (tc, rated, region), sub-split by rating band | Single writer per partition |
| Ratings (Postgres) | `user_id` (if ever sharded) | Small; one primary + replicas likely enough |
| Leaderboard ZSET | per time control; score-range shards if needed | Score-range keeps top-N single-shard |
| History (Cassandra) | `user_id`, clustered by `ended_at DESC` | Plus `game_id` lookup table |
| Gateways | none (any gateway) | L4 LB; presence maps user → gateway |

---

## 13. Senior vs. Staff Differentiators

| Topic | Senior answer | Staff answer |
|---|---|---|
| Transport | "Use WebSockets" | Splits gateway vs. game server and explains why (churn, deploys, independent scaling) |
| Consistency | "Store game in Redis" | Single-owner per game for linearizability, epoch fencing for zombie owners, write-ahead before broadcast |
| Clocks | "Server keeps the clock" | Explains lag unfairness quantitatively and proposes capped token-bucket compensation with anti-abuse |
| Matchmaking | "Sorted set by rating" | Little's Law shows the queue is ~10K; focuses on double-pairing, fairness windows with RD, hot-pool splits |
| Leaderboard | "Redis ZSET" | Ranks only active players; score-range vs hash sharding decides which query fans out |
| Failure | Mentions retries | Names failover time, paused clocks, reconnect herds, region-loss policy unprompted |
| CAP | System-wide claim | Per component: three CP islands, rest AP |
| Spectators | Same servers | Treats celebrity games as a hot-key problem; relay tree + delay |

---

## 14. 45-Minute Interview Time Allocation

| Minutes | Phase |
|---|---|
| 0–5 | Clarify FRs/NFRs, time controls, scale (10M DAU, 500K games) |
| 5–9 | Capacity: moves/s, sockets, memory, Little's Law for queue |
| 9–13 | Entities + API (REST + WS message types, `ply` idempotency) |
| 13–25 | HLD diagram; walk FR1 (play) and FR2 (matchmaking), then FR3/FR4 briefly |
| 25–40 | Deep dives: clock fairness, game-server scaling & failover, matchmaking fairness at scale, leaderboard sharding |
| 40–45 | CAP per component, failure modes, what you'd build next (anti-cheat, tournaments) |

---

## 15. Quick-Reference Cheatsheet

- **Scale:** 25M games/day · ~200K moves/s peak · 1.5M sockets · ~5 GB live state · ~10K waiting in queue.
- **Play:** WS → gateway → game server (single owner by `game_id`) → validate → Kafka append → ack + broadcast.
- **Clocks:** server-authoritative; credit = min(lag, cap, token bucket); server-side flagging with grace.
- **Matchmaking:** pool per (tc, rated, region); single leader-elected matcher; window = rating ± k·RD, widens with wait.
- **Ratings:** Glicko-2; Kafka `GameEnded` → Postgres txn `UNIQUE(game_id)` → outbox → Redis ZSET.
- **Leaderboard:** ZSET per tc, active players only, cache top-N; score-range sharding.
- **Failover:** lease expiry → reassign → replay log → bump epoch → pause clocks.
- **Spectators:** separate pub/sub tier, relay tree, broadcast delay.
- **CP islands:** live game, pairing, ratings. Everything else AP.

---

## 16. Related Guides

- **Leaderboard System** — ZSET sharding, score-range vs hash, approximate ranks.
- **Messaging Application (WhatsApp/MS Teams)** — WebSocket gateway fleets, presence, reconnect handling.
- **Apache Kafka** — `acks` + `min.insync.replicas` durability contract, partitioning for ordering.
- **Leader Election in Distributed Systems** — matcher leader per pool, leases, fencing tokens.
- **Distributed Unique ID Generation** — Snowflake `game_id`.
- **Sharding Strategies** — directory vs consistent hashing for stateful placement.
- **CAP / PACELC Theorem** — per-component positioning.
- **News Feed System / Twitter Fan-out** — hot-key pattern behind celebrity spectator games.
- **Notification System** — `matchFound` / challenge delivery when the app is backgrounded.
- **Circuit Breaker Pattern** — protecting game servers from slow downstreams (archive, rating).
