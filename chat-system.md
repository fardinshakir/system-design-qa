## Problem 3: Chat System

### Initial Answer
Users establish a WebSocket connection, "permanent" until not needed. User 1 sends a message to User 2 via a POST request. Messages are stored in a database for persistence/history, with an assigned message ID.

### Feedback
- Correct instinct on WebSockets for real-time, persistent connections
- Incorrect: "sends a POST request to User 2" — should be delivered over the already-open WebSocket, not a new HTTP request. HTTP/POST is one-shot request/response; the server can never speak first. WebSockets let the server push data to a client unprompted, which is exactly what's needed for real-time delivery.
- Correct: persisting messages with an ID for chat history

### Q: What's the difference between sending via WebSocket vs. POST?
**A (learned):** POST is a one-shot request/response — client asks, server answers, connection effectively ends; the server has no way to reach the client later unprompted (the old workaround was polling). WebSocket is a single connection that stays open indefinitely; either side can send data at any time without asking first. This is why the server can *push* a message to User 2 the instant it arrives, rather than waiting for User 2 to ask "anything new?"

### My restated understanding of the flow
1. Establish user in a `clients` map as connected (the "permanent" WebSocket)
2. On message received: parse data (text, recipient, etc.), check if recipient is online, push directly if so, otherwise consider what to do if offline
3. On disconnect: remove client from the map, close the socket

### Q: Shouldn't the message be stored regardless of the if/else (online/offline)?
**A (self-identified gap):** Yes — this was a correct catch of an oversimplification in the example walkthrough.
**Feedback:** Persistence should happen unconditionally, not just in the offline branch. Reasons: chat history needs to persist beyond the live moment, multi-device consistency requires a shared source of truth, and reliability — "delivered over a socket" isn't the same as "durably saved." Corrected order: **store to database first, then push if online** (storing first avoids a scenario where the recipient sees a message that's never actually saved if the server crashes right after pushing).

### Follow-up Q1: Walk through the exact path a message takes across multiple servers
**A:** With multiple servers, need to know which server holds User 2's WebSocket connection — use Redis to store and fetch that mapping quickly. The connection lives on a specific server (Server B); the sending server needs to know if the recipient is online.
**Feedback:** Correct and fairly advanced insight (this is effectively a "presence" or "connection registry" pattern). Gap: didn't specify *how* Server A actually hands the message to Server B once it knows the mapping.

### Q: Explain Server A → Server B communication in more depth
**A (learned):** Three options:
1. **Redis Pub/Sub** (cleanest, most commonly cited) — Server A publishes to a channel identified by Server B (or by user); Server B, subscribed to that channel, receives it and pushes down its local socket to User 2. Reuses infrastructure already introduced for presence lookup.
2. **Message queue** (Kafka/RabbitMQ) — more durable, slightly more latency/complexity
3. **Direct server-to-server RPC/HTTP call** — works, but requires service discovery/addressability between all servers

### Follow-up Q3: What database/storage for chat messages?
**A:** In-memory for immediate messages, a proper database for long-term storage.
**Feedback:** Right shape, but pure in-memory storage isn't safe even briefly — a crash loses it permanently. Refined: durable database is the source of truth (write always happens there); Redis can additionally cache the most recent N messages per conversation for fast reads, but it's an optimization layered on top, never the only copy. Real systems commonly use a wide-column store (Cassandra/DynamoDB) for chat, since it's write-heavy, partitions naturally by conversation ID, and reads are almost always time-ordered.

### Q: For fast storage, would I just use a Redis cache?
**A (learned):** Yes — Redis holds a bounded, recent slice per conversation (e.g., last 50 messages via `LPUSH` + `LTRIM`), read first on chat open; falls back to the durable DB for anything older or if the cache is cold/empty. Redis data is disposable — if lost, it can be rebuilt from the durable database.
