# Whiteboard: The Scaling Decision Ladder

## Overview

The most common follow-up in any system design interview: *"That works for 10K users. What breaks at 1M? What about 100M?"* Staff engineers answer this systematically — not by guessing, but by identifying the bottleneck at each order of magnitude and explaining what architectural change addresses it and why.

This is the framework for answering those questions confidently.

---

## The Core Pattern

Every scaling question follows this structure:

```
1. Identify what bottleneck appears at this scale
2. Name the specific failure mode (not just "it gets slow")
3. State the architectural change that addresses it
4. Explain the trade-off introduced
5. Identify what breaks next
```

The interviewer is not looking for the right answer — they're listening for evidence that you've shipped things and hit these limits in reality.

---

## The Ladder: Frontend Application

### 0 → 10K Users: Works as-is

```
Architecture: SPA + single server + one database
Bottleneck:   None yet — single server handles it
Metrics:      <100ms API response, <2s page load
```

Nothing needs to change here. If you add complexity at this scale, you're over-engineering.

---

### 10K → 100K Users: Static Assets Become the Bottleneck

```
Bottleneck:   Every user downloads JS/CSS/images from your origin server
              100K users × 500KB bundle = 50GB/day of transfer
              Server CPU and bandwidth saturated
Failure mode: Slow page loads globally. Users in Asia wait 8s for US-hosted assets.

Change:       Move static assets to a CDN
Trade-off:    Cache invalidation strategy needed (fingerprinted filenames)
              Stale deploys if cache not busted correctly
              Cost: CDN is cheaper than origin bandwidth at scale

Next limit:   API server becomes bottleneck
```

**How to talk about it:**
> "At 100K users, the first thing I'd add is a CDN for static assets. Every user downloads the same JS bundle — that's pure transfer cost with no compute value. Fingerprinted filenames give us safe aggressive caching: `app.3f9a2c.js` never changes, so `Cache-Control: max-age=31536000` is safe. The trade-off is I need to ensure my deploy pipeline busts caches atomically — old HTML referencing new JS filenames while CDN still serves old JS is worse than no CDN at all."

---

### 100K → 1M Users: API Server Bottleneck

```
Bottleneck:   Single API server handling 10K req/s
              Database connection pool exhausted
Failure mode: API p99 latency spikes. Timeouts during traffic peaks.
              Database becomes the single point of failure.

Change:       Horizontal scaling + load balancer + read replicas
Trade-off:    Session state can't live on one server (use Redis/JWT)
              Read replicas have replication lag (~50–200ms)
              Reads may be stale: user saves, immediately reloads, sees old data

Next limit:   Data volume for single database
```

**The subtle failure that comes up in interviews:**
> "When we add read replicas, we introduce eventual consistency. User saves a record, gets a success response, immediately refreshes — the read hits a replica that hasn't caught up yet. They see the old data. UI shows a stale state. Mitigation: route reads for recently-mutated data to the primary for the first 500ms (read-your-writes consistency), or use optimistic UI so the client doesn't depend on the refetch to show the update."

---

### 1M → 10M Users: Database and Caching Bottleneck

```
Bottleneck:   Single database can't handle write volume
              Popular records hit repeatedly (celebrity problem)
              Full-table scans on large tables
Failure mode: Database CPU at 100%. Write latency 2s+. Cascading timeouts.

Changes:
  - Add caching layer (Redis) for hot reads
  - Database sharding or move to distributed DB
  - Queue heavy writes (async processing)
Trade-offs:
  - Cache: invalidation complexity, stale data windows
  - Sharding: cross-shard queries become very expensive
  - Async writes: eventual consistency, harder error handling

Next limit:   Global latency (single region)
```

**The cache decision criteria:**
> "I'd add a cache when: the data is read much more often than it's written (cache hit rate > 80%), computation or DB query is expensive, and I can tolerate some staleness. For user profile data that changes rarely: perfect cache candidate. For live auction prices changing every second: caching at 60s TTL would show wrong bids — probably not worth it."

---

### 10M → 100M Users: Global Infrastructure

```
Bottleneck:   Single region means high latency for users far from the server
              Single point of failure — one region outage = full downtime
Failure mode: Users in Europe wait 300ms RTT to US servers.
              Regional AWS outage = global downtime.

Changes:
  - Multi-region deployment
  - Edge computing for latency-sensitive operations
  - Global CDN with regional origins
Trade-offs:
  - Multi-region: data synchronization complexity, eventual consistency globally
  - Active-active: conflict resolution for concurrent writes to different regions
  - Cost: 2x infrastructure
```

---

## The Scale Ladder: Specific Frontend Systems

### Chat Application

| Scale | Bottleneck | Change |
|---|---|---|
| 1K users | Nothing | Single WebSocket server |
| 100K users | WebSocket connections per server (~50K/server) | Multiple WS servers + pub/sub (Redis) |
| 1M users | Message fanout (1 message → 1M connections) | Message queue, fan-out workers |
| 10M users | Presence system (who's online) at scale | Approximate presence, heartbeat sampling |
| 100M users | Hot channels (100K users in one room) | Room sharding, tiered delivery |

### Infinite Feed (Twitter/LinkedIn)

| Scale | Bottleneck | Change |
|---|---|---|
| 10K users | Nothing | Query posts ordered by time |
| 100K users | Feed generation per request | Pre-compute feeds (fan-out on write) |
| 1M users | Pre-computing feeds for 1M users | Hybrid: pre-compute for low-follower, compute on read for high-follower |
| 10M users | Storage for 10M pre-computed feeds | Tiered storage, TTL on feed cache |
| 100M users | Following graph traversal | Celebrity accounts use pull model |

---

## How to Answer "What Breaks at 10x?"

Use this template out loud in the interview:

```
"At [current scale], the bottleneck would be [specific resource].
 What breaks is [specific failure mode] — you'd see [observable symptom].
 The change I'd make is [specific architectural change].
 The trade-off introduced is [what gets harder].
 After that change, the next thing to break is [next bottleneck]."
```

**Example answer to "your design handles 10K users, what about 1M?":**

> "At 1M, the bottleneck shifts from static asset delivery — which we've already solved with CDN — to the API layer. Specifically, I'd expect the database connection pool to become the limiting factor. The failure mode is cascading timeouts: the pool fills up, requests queue, queue fills up, requests fail, users retry, which makes it worse. The change is horizontal API scaling behind a load balancer with session state externalized to Redis. The trade-off is read-your-writes consistency — we need to be careful that a user who just saved data and immediately refreshes doesn't hit a read replica that's 200ms behind. After solving that, the next limit is the write capacity of the primary database."

---

## When to Say "I'd Measure First"

Interviewers at staff level respect this answer — it shows you don't over-engineer. Use it when:

- You're asked to choose between two architectures with unclear scale requirements
- The trade-off depends on actual traffic patterns you haven't seen
- Adding the optimization would add significant complexity

**What to say:**
> "I'd want to measure before making that call. The change adds [concrete complexity]. I'd only add it if [specific metric] exceeded [threshold] — for example, I'd add Redis caching when DB query latency p99 exceeded 100ms, not before. That threshold is where the complexity pays for itself."

This is not hedging — it's the answer of someone who has shipped things and learned that premature optimization is real.

---

## Interview Questions

**Q: "Your design works for 10K users. Walk me through what breaks at each order of magnitude."**
A: [Use the ladder above — bottleneck → failure mode → change → trade-off → next limit]

**Q: "When would you add a caching layer?"**
A: When: (1) read:write ratio is high (cache hit rate >80%); (2) the underlying query or computation is expensive; (3) I can tolerate a staleness window. The TTL depends on how stale is acceptable — product prices might tolerate 5 minutes, live auction bids tolerate 0 seconds.

**Q: "What's the hardest problem that appears when you scale to multiple regions?"**
A: Data consistency. In a single region, writes are synchronous — a user writes, reads back the same data. In multi-region, replication lag means users in different regions can see different states of the same data. The hardest case is concurrent writes to the same record from different regions — you need a conflict resolution strategy: last-write-wins (simple, can lose data), vector clocks (complex, preserves intent), or CRDT (automatic merge for specific data types).
