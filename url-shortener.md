## Problem 1: URL Shortener

### Initial Answer
Take the long URL, assign a randomly generated string, store those as key/value in a hashmap. Check if the random string already exists in the keys to avoid duplication. Return the shortened URL.

### Feedback
- Correctly identified the collision problem (checking for duplicates)
- Missing: requirements gathering (scale, read:write ratio, expiration, custom aliases, redirect type)
- "Hashmap" isn't a real storage answer at scale — no persistence, no horizontal scaling
- No mention of how the random string is actually generated
- No API surface / system boundaries defined
- No handling of concurrent uniqueness (multiple servers generating codes at once)

### Q: Explain API surface / system boundaries in more depth
**A (learned):** Define explicit endpoints — `POST /api/shorten` and `GET /{shortCode}` — including request/response shapes and status codes. Also sketch the actual components talking to each other: client → load balancer → app servers → cache → database. Redirect type matters: 301 (cacheable, but no analytics after first hit) vs 302 (hits server every time, enables click tracking).

### Q: Explain uniqueness/consistency at scale in more depth
**A (learned):** Check-then-act race condition — two servers can both check "does code X exist?", both get "no," and both try to write it. Three fixes, increasing sophistication:
1. **DB uniqueness constraint + retry** — simple, but retries get expensive as keyspace fills
2. **Pre-generated code pool** — background job generates codes in advance; servers atomically pop one from the pool (e.g., `DELETE ... RETURNING`), no runtime collision possible
3. **Auto-incrementing counter + base62 encoding** — no collision-checking needed at all; counter only ever goes up, and the encoding is a deterministic bijection

### Q: Explain the pre-generated pool fix (#2) with an example
**A (learned):** Two tables — `available_codes` (unused codes) and `urls` (real mappings). A background job tops up `available_codes` on its own schedule. At request time, a server atomically deletes-and-returns one row from `available_codes` — the database guarantees two servers can't claim the same row. Tradeoff: introduces an operational dependency (the pool must never run dry).

### Q: Explain the counter + base62 fix (#3) with an example
**A (learned):** A single atomic counter (`UPDATE counter SET value = value + 1 RETURNING value`) hands out a unique integer per request. That integer is base62-encoded (0-9, a-z, A-Z) into a short string — pure math, no DB lookup needed for the encoding step itself. Since the counter never repeats, there's no collision risk by construction.

**Follow-up — scaling the counter itself:** A single central counter becomes a bottleneck at high throughput (every request queues behind it, plus cross-region latency). Fix: **block allocation** — each server reserves a range (e.g., 1,000 numbers) upfront and hands out IDs locally until it runs out, only then asking the central counter for a new block. Tradeoff: if a server crashes mid-block, the unused numbers in that range are lost forever (acceptable given the size of the keyspace).

### Q: My understanding check — "counter += 1 each call, encode to base62, blocks solve the bottleneck?"
**A confirmed:** Correct on both counts.

### Q: Won't other URL shorteners using the same encoding scheme produce the same short URL?
**A (learned):** No — uniqueness only needs to hold within your own service. The actual short URL is domain + code (e.g., `sho.rt/4C92` vs `tinyurl.com/4C92`), so two different services' codes don't collide. Each service only looks up codes within its own database/namespace.
