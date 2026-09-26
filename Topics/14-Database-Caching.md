# Senior DevOps Interview Questions: Database & Caching

### Q: Your Redis cache hit rate has dropped significantly and database load has spiked at the same time. How do you investigate?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A falling hit rate with rising DB load usually means the cache stopped serving requests, not that traffic increased, so I check Redis itself first.

`INFO stats` gives me `keyspace_hits` versus `keyspace_misses` directly, and `INFO memory` tells me if it's near `maxmemory` and evicting. If `evicted_keys` is climbing, that's the answer — the working set doesn't fit in memory anymore, and keys are getting evicted before they can be reused.

If memory's fine, I check for a recent TTL change — I've seen a "keep data fresher" tweak drop a TTL from an hour to a couple minutes, which tanks the hit rate immediately since keys expire before they're reused.

*Check hits vs misses and eviction count → check memory against maxmemory → check for a recent TTL or deploy change → check if Redis itself restarted recently (cold cache looks identical to this).*

If it's genuinely undersized, I scale or shard the cache. If it's a TTL mistake, that's a one-line revert — the `INFO` output tells me which one within a minute.

</details>

---

### Q: How do you handle a database schema migration on a large table with zero downtime?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The mistake I've seen is a migration that's instant in dev locking the whole table for minutes in production — an `ADD COLUMN` with a default used to lock the entire table on older Postgres versions, and on tens of millions of rows that's real downtime for every write.

My approach is splitting it into safe, backward-compatible steps. Add a nullable column with no default first — that's fast, no rewrite needed. If a default or `NOT NULL` is actually required, I add it separately or backfill in small batches instead of one giant `UPDATE`.

For a rename or type change, I do it across multiple deploys — add the new column, dual-write to both from the app, backfill in batches, switch reads over, then drop the old column once nothing references it.

*Check what lock level the migration actually takes → nullable column first, no default → backfill in batches, not one UPDATE → dual-write during the transition for renames/type changes → drop the old column last.*

I always test the migration against a production-sized copy first — instant on 10,000 rows can behave completely differently on 50 million.

</details>

---

### Q: The application is throwing "too many connections" errors against the database. What's your approach?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is almost never a database capacity problem the first time I see it — it's the app's connection pooling, so I check that before touching the database's limit.

I do the math: app instances × pool size per instance. I've seen a pool size of 20 that was fine at 5 instances turn into 500 connections the moment autoscaling took it to 25 — well past a 200 connection limit nobody had recalculated.

If the math checks out and it's still maxing out, I check for a leak — a code path that opens a connection but doesn't release it on an error path shows up as the count climbing steadily and never coming back down, even at low traffic.

*Calculate total possible connections across instances → compare to max_connections → check for a leak if the math doesn't explain it → add a pooler like PgBouncer instead of just raising the DB limit.*

Raising `max_connections` directly is a last resort — each connection reserves real memory on the database, and it doesn't fix a leak, just delays it.

</details>

---

### Q: How do you design a caching strategy to prevent a cache stampede when a popular key expires?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A stampede is when a hot key expires and a flood of requests all miss at the same instant and hit the database for the same value simultaneously — I've seen this take down an otherwise healthy database in seconds.

First fix is jittering the TTL, so keys set around the same time don't all expire at the exact same second. For genuinely hot keys, I use a short-lived lock in Redis on cache miss — whoever gets the lock repopulates the cache, everyone else waits briefly or serves stale data instead of all hitting the database independently.

For data that doesn't need to be perfectly fresh, stale-while-revalidate works well — serve the old value immediately while one background request quietly refreshes it, so there's never a hard miss at all.

*Jitter TTLs so keys don't all expire together → lock on miss so only one request repopulates a hot key → everyone else waits or serves stale → non-critical data gets stale-while-revalidate instead of a hard miss.*

</details>

---

### Q: A read replica is lagging behind the primary database by several minutes. How do you troubleshoot?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Lag climbing means the replica can't apply changes as fast as the primary generates them, so I check the replica's own resources first, not the primary.

I check CPU, disk I/O, and whether something else is competing for it — a heavy reporting query running directly against the replica competes with replication for the same disk I/O. I also check for a single long-running query on the replica itself, since on Postgres a long read can actually block replay depending on configuration.

If the replica looks fine, I check the primary's recent write volume — a bulk load or a big batch `UPDATE` generates a lot of replication traffic at once, and a normally fine replica can fall behind from a genuine spike, not a replica problem at all.

*Check replica CPU/disk I/O → check for a query blocking replay → check primary's write volume for a recent spike → resource-bound, scale it or move competing reads off → one-time batch job, let it catch up and monitor.*

</details>

---

### Q: How do you decide between adding read replicas, adding a cache, or sharding, when a database can't handle the load anymore?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't jump to sharding — it's the most expensive option operationally, and I've seen teams reach for it when a missing index would've bought another year.

I profile first: is it read-heavy, write-heavy, or just inefficient queries that'd be slow regardless of architecture? If it's read load on data that repeats, caching is the cheapest fix. If reads don't need the absolute latest write, read replicas come next.

Sharding is only for write-bound load or a dataset too large for one instance — and even then, I pick the shard key based on real query patterns first, since a bad shard key guarantees painful cross-shard queries later.

*Profile the real bottleneck → cacheable reads, add caching → non-cacheable but not latest-critical reads, add replicas → write-bound or dataset too large, consider sharding with a shard key based on real access patterns.*

Sharding is a one-way door — much harder to undo than to add — so it's the last option, not the first.

</details>

---

### Q: How do you handle stale cache data after the underlying database record gets updated?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I default to invalidate-on-write — the same code path that writes to the database also deletes or updates the corresponding cache key, instead of waiting for a TTL to expire it naturally.

When one write affects many cache entries — like a user profile embedded in a dozen cached list views — I use a versioned cache key instead of hunting down every entry. The key includes a version, like `user:123:v4`, and updating the user just bumps the version, so every old entry naturally becomes unreachable.

I always keep a short TTL underneath explicit invalidation too, since that assumes every write path remembers to invalidate correctly, and in a real codebase that assumption eventually breaks somewhere. A short TTL means a missed invalidation self-heals in minutes instead of staying wrong indefinitely.

</details>

---
