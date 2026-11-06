# Server-Sent Events (SSE) vs WebSocket
### Senior & Staff Engineer Level

---

## Table of Contents

1. [What Is SSE?](#1-what-is-sse)
2. [How It Works](#2-how-it-works)
3. [Where and When It's Used](#3-where-and-when-its-used)
4. [Pros and Cons](#4-pros-and-cons)
5. [SSE vs WebSocket](#5-sse-vs-websocket)
6. [Decision Framework](#6-decision-framework)
7. [Other Options Worth Naming](#7-other-options-worth-naming)
8. [Staff-Level Concerns: Operating Push at Scale](#8-staff-level-concerns-operating-push-at-scale)
9. [Interview One-Liner](#9-interview-one-liner)

---

## 1. What Is SSE?

**Server-Sent Events (SSE)** is a standard way for a server to push a continuous stream of updates to a client over a single, long-lived HTTP response. The client asks once, and the server keeps the response open and writes events into it whenever it has something new. Data flows only from server to client.

---

## 2. How It Works

```
Client                                   Server
  │  GET /events                           │
  │  Accept: text/event-stream             │
  │ ─────────────────────────────────────▶ │
  │                                        │
  │  HTTP/1.1 200 OK                       │
  │  Content-Type: text/event-stream       │
  │  Cache-Control: no-cache               │
  │ ◀───────────────────────────────────── │
  │  id: 41\ndata: {"price": 101.2}\n\n    │  ← pushed whenever ready
  │ ◀───────────────────────────────────── │
  │  id: 42\ndata: {"price": 101.5}\n\n    │
  │ ◀───────────────────────────────────── │
  │        ... connection stays open ...   │
```

### The wire format

The format is plain UTF-8 text. Each event is a set of `field: value` lines, and a blank line ends the event:

```
event: price-update          ← optional event type (default "message")
id: 42                       ← event ID, used for resume after reconnect
retry: 5000                  ← tells the client how long to wait before reconnecting (ms)
data: {"symbol":"NOW","price":101.5}
data: second line of the same event    ← several data lines are joined with \n

: this is a comment, often used as a heartbeat
```

### The client (browser `EventSource` API)

```javascript
const es = new EventSource("/events");

es.onmessage = (e) => console.log(e.data);            // default events
es.addEventListener("price-update", (e) => {          // named events
  render(JSON.parse(e.data));
});
es.onerror = () => { /* browser reconnects automatically */ };
```

### The server (Spring WebFlux)

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<Price>> stream(
        @RequestHeader(value = "Last-Event-ID", required = false) String lastId) {

    Flux<ServerSentEvent<Price>> replay = replayStore.since(lastId);   // missed events
    Flux<ServerSentEvent<Price>> live = priceBus.subscribe()
        .map(p -> ServerSentEvent.<Price>builder()
            .id(p.seq()).event("price-update").data(p).build());
    Flux<ServerSentEvent<Price>> heartbeat = Flux.interval(Duration.ofSeconds(15))
        .map(i -> ServerSentEvent.<Price>builder().comment("ping").build());

    return Flux.merge(replay.concatWith(live), heartbeat);
}
```

### The feature that makes SSE useful: automatic reconnect and resume

When the connection drops, the browser reconnects on its own after the `retry` interval. It sends a **`Last-Event-ID: 42`** header with the reconnect request. If the server keeps a replay buffer, it can send everything after event 42, so the client misses nothing. With WebSocket, you have to build all of this yourself.

---

## 3. Where and When It's Used

| Use case | Why SSE fits |
|---|---|
| **LLM token streaming** (OpenAI and Anthropic APIs, ChatGPT/Claude UIs) | The request goes up once and tokens stream back. This is the most common SSE use today. |
| Notifications and activity feeds | The server pushes and the client only reads. |
| Live dashboards and metrics | The data flows one way, and the client can recover after a disconnect. |
| Stock and sports tickers, live scores | High fan-out, server-driven updates. |
| Progress of long-running jobs (CI logs, file processing, query execution) | Stream status updates as the job proceeds instead of having the client poll. |
| Collaborative "someone changed this" signals | Pushes go over SSE, and client edits go up via normal POST requests. |

**Rule of thumb:** use SSE when the server talks a lot and the client talks rarely. Any client-to-server messages can go over ordinary HTTP requests.

---

## 4. Pros and Cons

### Pros

- **It's just HTTP.** It works with existing auth (cookies), load balancers, proxies, CDNs, observability tools and firewalls. There is no protocol upgrade.
- **Reconnect and resume are built in** through `retry` and `Last-Event-ID`.
- **It's simple.** There's no framing library, no handshake, and no ping/pong protocol. You can debug it with `curl -N`.
- **HTTP/2 and HTTP/3 multiplex it.** Many SSE streams share one TCP/QUIC connection with other requests.
- **Server code is stateless-friendly.** Each stream is one response, so it fits reactive stacks (WebFlux, Node, Go) naturally.
- **Compression works** because it's a normal HTTP response.

### Cons

- **It's unidirectional.** Client-to-server messages need separate HTTP requests.
- **It carries text only (UTF-8).** Binary data has to be Base64-encoded, which adds about 33% overhead.
- **The browser `EventSource` API can't set custom headers.** You can't send `Authorization: Bearer ...`, so you have to use cookies, a short-lived token in the query string (which ends up in logs), or a `fetch()`-based client. LLM SDKs use the `fetch()` approach.
- **HTTP/1.1 limits each browser to about 6 connections per domain.** Opening several tabs can starve the app. HTTP/2 removes this limit.
- **Proxy buffering can break streaming.** Nginx and some CDNs buffer responses and hold events back. You need `X-Accel-Buffering: no`, `proxy_buffering off`, and to avoid gzip buffering in the middle of the path.
- **Idle timeouts at load balancers** (for example the ALB default of 60s) will cut silent streams, so you need heartbeat comments.
- **Every open stream holds a connection on the server,** just like WebSocket. This is cheap with non-blocking I/O and expensive with thread-per-request servlets.

---

## 5. SSE vs WebSocket

**WebSocket** starts as an HTTP request, then upgrades (`Upgrade: websocket`) into a separate, **full-duplex, framed protocol** over the same TCP connection. After the upgrade, it no longer behaves like HTTP.

| Dimension | SSE | WebSocket |
|---|---|---|
| Direction | Server → client only | Full duplex (both directions) |
| Protocol | Plain HTTP (1.1/2/3) | Its own protocol after the HTTP upgrade (`ws://`, `wss://`) |
| Data types | UTF-8 text | Text and binary frames |
| Reconnect | Automatic, built into the browser | Manual: you write backoff, jitter and resume logic |
| Message resume | `Last-Event-ID` is built in | You build sequence numbers and replay yourself |
| Auth | Cookies; no custom headers in `EventSource` | Cookies or a token in the first message; no custom headers in the browser API either |
| HTTP/2 multiplexing | Yes, natively | Rarely (RFC 8441 exists but support is spotty), so usually one TCP connection per socket |
| Proxy/LB/firewall friendliness | High; it's ordinary HTTP | Medium; some corporate proxies break upgrades, and you need L7 LB support |
| Per-message overhead | Small text framing | 2–14 bytes of frame header; very efficient |
| Latency for client → server | One HTTP request per message | Very low, because the connection is already open |
| Debuggability | `curl`, browser network tab, normal access logs | Needs WebSocket-aware tooling |
| Implementation complexity | Low | Medium to high (heartbeats, reconnect, backpressure, protocol on top) |
| Typical uses | LLM streaming, feeds, notifications, dashboards, job progress | Chat, multiplayer games, collaborative editors, trading terminals, live cursors |

---

## 6. Decision Framework

```
Does the client need to send frequent, low-latency messages?
├── No  → SSE (the client's occasional writes go via POST)
└── Yes
    ├── Binary data or very high message rates (games, audio, trading)? → WebSocket
    ├── Conversational, but the client writes rarely (chat at human speed)? → either works;
    │      SSE + POST is simpler to operate, WebSocket gives lower write latency
    └── Need unreliable/unordered delivery or many independent streams? → look at WebTransport
```

---

## 7. Other Options Worth Naming

- **Long polling:** the client sends a request, the server holds it until data exists, then the client sends a new one. It works everywhere but wastes a round trip per message. Still a common fallback.
- **Short polling:** the simplest option and the most wasteful. It's fine for updates measured in minutes.
- **gRPC server streaming:** like SSE for service-to-service traffic, with binary Protobuf and HTTP/2. Browsers need gRPC-Web.
- **WebTransport (HTTP/3):** the emerging successor to WebSocket, with multiplexed streams and datagrams. Browser and infrastructure support is still maturing.

---

## 8. Staff-Level Concerns: Operating Push at Scale

These issues apply to both SSE and WebSocket. Raising them without being asked is what separates a Staff answer from a Senior one.

1. **Fan-out needs a backplane.** User A's stream is on node 3, but the event was produced on node 7. Put a pub/sub layer between them (Redis Pub/Sub, Redis Streams, Kafka or NATS). Each gateway node subscribes and routes events to its local connections.
2. **Separate the connection gateway from business logic.** A dedicated stream-gateway tier holds the connections. Backend services stay stateless and publish to the bus. Each tier then scales on its own metric: connections for the gateway, CPU for the services.
3. **The replay buffer decides delivery guarantees.** `Last-Event-ID` only helps if you can serve the missed events. Redis Streams or a Kafka offset per user or topic, with a bounded retention window, gives at-least-once delivery. Clients must dedupe by ID.
4. **Reconnect storms (thundering herd).** A deploy or node failure drops 100K connections, and they all reconnect at the same moment. Fixes:
   - Set a randomized `retry:` value per connection.
   - Drain nodes gradually during deploys.
   - Rate-limit at the gateway.
5. **Heartbeats.** Send a `: ping` comment every 15–30s to stay under load balancer and NAT idle timeouts. Heartbeats also let you detect dead clients.
6. **Connection density.** On an event-loop server (Netty, WebFlux, Node, Go), one node can hold 100K+ idle streams. The real limits are file descriptors, memory per connection, and TLS state. Thread-per-connection servlets fall over orders of magnitude earlier.
7. **Backpressure for slow consumers.** A client on a slow network makes the server's buffer for that connection grow. Cap it, then drop or collapse old events (send only the latest price), or disconnect the client. Never buffer without a bound.
8. **Load balancing.** Long-lived connections make round-robin imbalanced over time, because new nodes get no existing connections. Use least-connections balancing and periodically rebalance by asking clients to reconnect.

---

## 9. Interview One-Liner

> "I'd use SSE here because the flow is server-to-client, it rides plain HTTP/2 through our existing infrastructure, and reconnect-with-resume is built into the protocol. Client actions go over normal POSTs. I'd switch to WebSocket only if we needed high-frequency bidirectional or binary messaging. Either way, the hard part is the same: a gateway tier, a pub/sub backplane for fan-out, a replay buffer for resume, and reconnect-storm protection."

---

*Guide compiled for Senior & Staff-level system design interview preparation.*
*Topics: Server-Sent Events, WebSocket, Real-Time Push, Fan-Out, Connection Gateways.*
