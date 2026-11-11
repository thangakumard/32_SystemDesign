# Real-time Updates Cheat Sheet

A comprehensive guide for System Design Interviews based on [Hello Interview: Real-time Updates](https://www.hellointerview.com/learn/courses/system-design/lesson/realtime-updates/realtime-updates).

---

## Table of Contents
1. [Overview & Core Challenge](#1-overview--core-challenge)
2. [Networking Foundations & OSI Layers](#2-networking-foundations--osi-layers)
3. [Hop 1: Server-to-Client Communication (Protocols & Tradeoffs)](#3-hop-1-server-to-client-communication-protocols--tradeoffs)
   - [Short Polling](#short-polling)
   - [Long Polling](#long-polling)
   - [Server-Sent Events (SSE)](#server-sent-events-sse)
   - [WebSockets](#websockets)
   - [WebRTC](#webrtc)
   - [Protocol Comparison Matrix](#protocol-comparison-matrix)
4. [Hop 2: Source-to-Server / Intra-Backend Communication](#4-hop-2-source-to-server--intra-backend-communication)
   - [Stateful vs Stateless Gateway Architecture](#stateful-vs-stateless-gateway-architecture)
   - [Pub/Sub & Message Brokers (Redis Pub/Sub, Kafka)](#pubsub--message-brokers-redis-pubsub-kafka)
   - [Connection & State Management](#connection--state-management)
5. [Scaling & Architectural Challenges](#5-scaling--architectural-challenges)
   - [Load Balancing Persistent Connections](#load-balancing-persistent-connections)
   - [Connection Memory & Resource Exhaustion](#connection-memory--resource-exhaustion)
   - [Reconnection & Backoff Strategies](#reconnection--backoff-strategies)
   - [Reconnection Storms & Thundering Herd](#reconnection-storms--thundering-herd)
6. [Interview Strategy & Common Pitfalls](#6-interview-strategy--common-pitfalls)

---

## 1. Overview & Core Challenge

Real-time updates address the challenge of delivering immediate notifications, state changes, or event streams from servers to clients with minimal latency.

### The Two-Hop Problem
In real-time system design, updates are split into two distinct hops:
1. **Hop 1 (Server to Client):** Transport protocol used to push updates from the edge/gateway server to the end user's device.
2. **Hop 2 (Source to Server / Intra-Backend):** Internal messaging layer used to deliver events from internal microservices to the specific connection server handling the target client.

---

## 2. Networking Foundations & OSI Layers

Understanding key networking layers is critical for making informed architecture choices:

* **Layer 3 (Network Layer - IP):** Handles packet routing and addressing between hosts. *Best-effort delivery* (packets can be lost, duplicated, or reordered).
* **Layer 4 (Transport Layer - TCP/UDP):**
  * **TCP (Transmission Control Protocol):** Connection-oriented, guaranteed, in-order delivery via handshakes (`SYN`, `SYN-ACK`, `ACK`). Higher overhead and latency.
  * **UDP (User Datagram Protocol):** Connectionless, fast ("fire and forget"), no order or delivery guarantees. Used for media streams, gaming, telemetry.
* **Layer 7 (Application Layer - HTTP, WebSockets, SSE, WebRTC):** High-level protocols built on top of Layer 4.

---

## 3. Hop 1: Server-to-Client Communication (Protocols & Tradeoffs)

### Short Polling
Client periodically sends HTTP requests to the server on a fixed time interval (e.g., every 5 seconds).

* **How it Works:** Client sends standard `GET /updates` request $\rightarrow$ Server responds immediately with data (or empty) $\rightarrow$ Client waits interval and repeats.
* **Pros:** Extremely simple to implement, stateless, works natively with existing HTTP infrastructure/load balancers.
* **Cons:** High latency (delay up to polling interval), massive network overhead, inefficient server resource utilization (empty responses).
* **When to Use:** Low frequency requirements, legacy infrastructure, or minimal update velocity.

---

### Long Polling
Client opens an HTTP request, and the server holds the request open until new data is available or a timeout occurs.

* **How it Works:** Client requests `GET /updates` $\rightarrow$ Server holds connection open $\rightarrow$ Server responds when event occurs (or times out) $\rightarrow$ Client immediately opens a new request.
* **Pros:** Lower latency than short polling, works over HTTP/1.1 without special protocol upgrades, fallback for firewalls blocking WebSockets.
* **Cons:** High HTTP header overhead per message, connection resets after every message, complex state management on server.
* **When to Use:** Low-frequency updates where WebSockets/SSE are restricted or unsupported.

---

### Server-Sent Events (SSE)
A persistent, unidirectional HTTP connection where the server streams text-based events (`text/event-stream`) to the client over standard HTTP.

* **How it Works:** Client opens an HTTP request with `Accept: text/event-stream` $\rightarrow$ Server keeps response open indefinitely $\rightarrow$ Server pushes JSON/text chunks over the single connection.
* **Pros:** Native browser auto-reconnection (`EventSource` API), lightweight, works over HTTP/2 without connection limits, multiplexing support, built-in event IDs.
* **Cons:** Unidirectional only (server to client), HTTP/1.1 browsers limit max concurrent connections to 6 per domain, text/UTF-8 streaming only (binary payload requires base64 encoding).
* **When to Use:** Single-directional streams (news feeds, live scoreboards, price tickers, LLM streaming output, notifications).

---

### WebSockets
Full-duplex, bidirectional, persistent TCP connection initiated via an HTTP handshake.

* **How it Works:** Client sends HTTP GET request with headers `Upgrade: websocket` and `Connection: Upgrade` $\rightarrow$ Server responds with HTTP `101 Switching Protocols` $\rightarrow$ Protocol switches from HTTP to WebSocket (`ws://` or `wss://`) over the same underlying TCP socket.
* **Pros:** Low overhead (minimal framing), bidirectional low-latency messaging, supports binary and text data streams.
* **Cons:** Stateful connections complicate load balancing and auto-scaling, requires L4 load balancers or proxy upgrades, no native reconnect support (must handle manually).
* **When to Use:** Interactive bidirectional systems (chat apps like WhatsApp/Slack, collaborative editors like Google Docs, multiplayer games).

---

### WebRTC
Peer-to-peer (P2P) protocol designed for ultra-low latency real-time audio, video, and data transmission.

* **How it Works:** Requires a signaling server to exchange network info (ICE candidates/SDP) $\rightarrow$ Direct P2P UDP connection established between clients (uses STUN/TURN servers for NAT traversal).
* **Pros:** Extremely low latency, direct client-to-client transmission bypasses server bandwidth bottlenecks.
* **Cons:** Highly complex setup, media streaming focus, doesn't scale natively to large broadcasting groups without SFUs (Selective Forwarding Units).
* **When to Use:** Video calling, audio chat, peer-to-peer file sharing.

---

### Protocol Comparison Matrix

| Protocol | Direction | Transport | Overhead | Multiplexing / Scalability | Primary Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Short Polling** | Client $\rightarrow$ Server | HTTP/1.1 or HTTP/2 | High (HTTP Headers) | High | Low-frequency status checks |
| **Long Polling** | Client $\rightarrow$ Server | HTTP/1.1 or HTTP/2 | Medium-High | Medium | Legacy fallback real-time |
| **SSE** | Server $\rightarrow$ Client | HTTP/1.1 or HTTP/2 | Low | High (H2 multiplexing) | Dashboards, Tickers, LLM Streaming |
| **WebSockets** | Bidirectional | TCP (upgraded HTTP) | Very Low | Needs custom routing | Chat apps, Collaborative tools |
| **WebRTC** | Peer-to-Peer | UDP (typically) | Lowest | Low (complex topologies) | Audio/Video calling |

---

## 4. Hop 2: Source-to-Server / Intra-Backend Communication

Once an update arrives at the backend, it must be routed to the specific server holding the open TCP connection to the destination client.

### Stateful vs Stateless Gateway Architecture

```
[ Clients ]  <--- (WebSockets / SSE) --->  [ Real-time Gateways / Connection Servers ]
                                                        |
                                              [ Pub/Sub / Message Broker ]
                                                        |
                                                [ Application Services ]
```

* **Stateless Gateway:** Gateway servers do not track connection state locally; they query a centralized store (e.g., Redis) or hash client IDs to find destination nodes.
* **Stateful Gateway:** Gateway holds persistent connection instances. A global registry mapping `Client_ID -> Connection_Server_IP` is maintained in a fast, centralized cache (e.g., Redis, DynamoDB).

---

### Pub/Sub & Message Brokers

1. **Redis Pub/Sub:**
   * Direct push-based delivery to active subscribers with minimal memory footprint.
   * *Tradeoff:* At-most-once delivery (fire and forget); no message persistence. If a connection server drops, missed messages are lost unless backed by a persistent store.
2. **Apache Kafka / RabbitMQ:**
   * Persistent message logs and topic partitions.
   * Enables durable delivery guarantees (at-least-once) and message playback upon client reconnection.

---

### Connection & State Management
* **Presence Tracking:** Heartbeats / Ping-Pongs sent across connections to update client presence (e.g., online/offline) in Redis.
* **Routing Keys:** Messages published to topics scoped by user ID (`user:123`), channel ID (`channel:marketing`), or broadcast streams.

---

## 5. Scaling & Architectural Challenges

### Load Balancing Persistent Connections
* **Layer 4 vs Layer 7 Load Balancing:**
  * **L4 (TCP):** Distributes connections based on IP and Port. Fast, lightweight, content-agnostic. Preferred for long-lived WebSocket connections.
  * **L7 (Application):** Inspects HTTP headers/URLs. Required for SSE routing and WebSocket handshakes, but maintains double connection overhead (Client-to-LB and LB-to-Server).

### Connection Memory & Resource Exhaustion
* Standard OS file descriptors and socket buffers consume memory ($\sim 10\text{ KB} - 50\text{ KB}$ per open connection).
* **Formula:** $1,000,000\text{ connections} \times 20\text{ KB/connection} \approx 20\text{ GB RAM}$ just for connection overhead.
* **Mitigations:** Tune OS `sysctl` settings (e.g., `max user processes`, `file-max`), use asynchronous I/O frameworks (e.g., Netty, Tokio, Node.js).

### Reconnection & Backoff Strategies
* When network drops occur, thousands/millions of clients attempt immediate reconnection.
* **Exponential Backoff with Jitter:** Prevent server overload by adding randomized delays:
  $$\text{Backoff Delay} = \min(\text{MaxDelay}, \text{Base} \times 2^{\text{retry\_count}}) + \text{RandomJitter}$$

### Reconnection Storms & Thundering Herd
* Avoid synchronized reconnect attempts during deployments or backend outages.
* Cache missing messages using sequence IDs or timestamps so clients can catch up without querying primary relational databases (`GET /updates?since_id=12345`).

---

## 6. Interview Strategy & Common Pitfalls

### Strategy Checklist
1. **Default to REST/HTTP First:** Don't jump to WebSockets unless bidirectional or sub-second latency is explicitly needed.
2. **Choose SSE for One-Way Streams:** If the requirement is server-to-client notifications (e.g., sports scores, notification icons, news updates), prefer SSE over WebSockets due to simplicity and native HTTP infrastructure compatibility.
3. **Address Both Hops:** Explain how updates traverse from client $\rightarrow$ backend service (Hop 1), and how backend services propagate updates across persistent connection servers using Pub/Sub (Hop 2).
4. **Size Your Connection Fleet:** Demonstrate capacity awareness by calculating memory required for open TCP sockets at peak concurrent connections.

### Common Pitfalls to Avoid
* ❌ **Proposing WebSockets everywhere:** WebSockets introduce stateful infrastructure complexity, sticky routing problems, and scaling challenges.
* ❌ **Ignoring Hop 2 routing:** Forgetting how the backend knows *which* specific WebSocket server holds the socket for a given user.
* ❌ **Omitting retry mechanics:** Failing to mention sequence numbers, reconnect backoffs, or message recovery when connections break.