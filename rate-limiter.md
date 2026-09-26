## Problem 2: Rate Limiter

### Initial Answer
A counter assigned per user, set to some number of allowed requests (x), refilling by some rate (x per minute / y per second). Each request removes a token; no tokens means no more requests. Bulk requests would have to wait or go slower.

### Feedback
- Correctly described the token bucket algorithm (capacity, refill rate, decrement, reject-when-empty, implicit burst handling)
- Missing: where the counter physically lives (in-memory breaks across multiple servers)
- Missing: alternative algorithms (fixed window, sliding window log/counter, leaky bucket)
- Missing: what the client experiences when rate-limited (status code, retry info)
- Missing: limit granularity (per user/IP/API key/endpoint)
- Missing: where in the architecture the check happens (app server vs. middleware vs. gateway)

### Q1: How do you enforce the limit consistently across multiple servers?
**A:** The tracker needs to be separate from the individual servers — an external, shared counter all servers reference, making it a global rule.
**Feedback:** Correct — this is the standard solution (pull state out of individual servers into shared storage).

### Q2: What actual data store would you use, and why?
**A:** A key-value dictionary — user mapped to a value.
**Feedback:** Right shape, but "dictionary" names the data structure, not the deployed system. Real answer: **Redis** — fast (in-memory, sub-millisecond), supports atomic `INCR`/`DECR` (no race conditions between servers), and native key expiration (`TTL`) for refill windows.

### Q3: What should the API return to a rate-limited client?
**A:** HTTP 429 (Too Many Requests).
**Feedback:** Correct. One layer deeper: pair it with headers like `Retry-After` or `X-RateLimit-Remaining`/`X-RateLimit-Reset` so clients can back off intelligently.

### Follow-up Q1: How would you implement the refill with Redis TTL specifically?
**A:** Set a TTL of 60s; after 60s the counter hard-resets back to full capacity.
**Feedback:** This is actually the **fixed window** algorithm, not true token bucket. Known flaw: a user can burst at the very end of one window and immediately burst again at the start of the next, getting up to 2x requests across the boundary. True token bucket refills continuously instead of resetting all at once. Both are valid answers if the tradeoff is named explicitly.

### Follow-up Q2: What happens if the single Redis instance goes down — fail open or fail closed?
**A:** Fail closed (reject requests) — don't want to blindly accept potentially malicious traffic; would also add a health check for Redis.
**Feedback:** Reasonable and well-justified. Worth knowing: many real systems actually fail *open* for rate limiters specifically, reasoning that blocking all legitimate traffic during an outage is worse than a temporary burst of unmetered requests. No universally "right" answer — the key skill is naming it as a conscious tradeoff.

### Follow-up Q3: What do you key the limiter on for unauthenticated requests?
**A:** IP address, since there's no user ID to reference. Could split traffic into authenticated vs. unauthenticated buckets.
**Feedback:** Correct instinct. Caveat: IP-based limiting can punish legitimate users sharing an IP (corporate NAT, university networks, CGNAT) — a known, unsolved limitation worth naming.
