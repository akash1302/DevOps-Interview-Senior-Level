# Senior DevOps Interview Questions: Database & Caching

### Q: Your Redis cache hit rate has dropped significantly and database load has spiked at the same time. How do you investigate?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A falling hit rate with rising database load usually means the cache stopped serving requests, not that traffic actually increased, so I check Redis itself first.

`INFO stats` gives me hits versus misses directly, and `INFO memory` tells me if it's near its memory limit and evicting keys. If evictions are climbing, that's the answer — the working set doesn't fit in memory anymore, and keys get evicted before they can be reused.

If memory looks fine, I check for a recent TTL change. I've seen a well-meaning "keep data fresher" tweak drop a TTL from an hour to a couple minutes, which tanks the hit rate immediately since keys expire before they're reused. If it's genuinely undersized, I scale or shard the cache. If it's a TTL mistake, that's a one-line revert.

</details>

---

### Q: How do you handle a database schema migration on a large table with zero downtime?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The mistake I've seen is a migration that's instant in dev but locks the whole table for minutes in production — on tens of millions of rows, that's real downtime for every write.

My approach is splitting it into safe, backward-compatible steps. I add a nullable column with no default first, since that's fast with no table rewrite. If a default or a not-null constraint is actually required, I backfill in small batches instead of one giant update.

For a rename or type change, I do it across multiple deploys — add the new column, have the app write to both, backfill in batches, switch reads over, then drop the old column once nothing references it. I always test the migration against a production-sized copy first, since something instant on 10,000 rows can behave very differently on 50 million.

</details>

---

### Q: The application is throwing "too many connections" errors against the database. What's your approach?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is almost never a database capacity problem the first time I see it, it's the app's connection pooling, so I check that before touching the database's limit.

I do the math — app instances times pool size per instance. I've seen a pool size that was fine at 5 instances turn into way more connections than the database allows the moment autoscaling took it to 25, because nobody recalculated the total.

If the math checks out and it's still maxing out, I check for a connection leak — a code path that opens a connection but doesn't release it on an error path shows up as the count climbing steadily and never coming back down. Raising the database's connection limit directly is a last resort — I'd rather add a connection pooler in front of it than just delay the problem.

</details>

---

### Q: How do you design a caching strategy to prevent a cache stampede when a popular key expires?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A stampede is when a hot key expires and a flood of requests all miss at the same instant and hit the database for the same value at once — I've seen this take down an otherwise healthy database in seconds.

The first fix is jittering the TTL, so keys set around the same time don't all expire at the exact same second. For genuinely hot keys, I use a short-lived lock on cache miss — whoever gets the lock repopulates the cache, everyone else waits briefly or serves stale data instead of all hitting the database independently.

For data that doesn't need to be perfectly fresh, serving the old value while quietly refreshing it in the background works well too, so there's never a hard miss at all.

</details>

---

### Q: A read replica is lagging behind the primary database by several minutes. How do you troubleshoot?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Lag climbing means the replica can't apply changes as fast as the primary generates them, so I check the replica's own resources first, not the primary.

I check CPU and disk I/O, and whether something else is competing for it, like a heavy reporting query running directly against the replica. I also check for a long-running query on the replica itself, since that can actually block replay depending on the database.

If the replica looks fine, I check the primary's recent write volume — a bulk load or a big batch update can generate enough replication traffic on its own to make a normally healthy replica fall behind temporarily.

</details>

---

### Q: How do you decide between adding read replicas, adding a cache, or sharding, when a database can't handle the load anymore?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't jump to sharding, it's the most expensive option operationally, and I've seen teams reach for it when a missing index would've bought another year.

First I profile what's actually slow — read-heavy, write-heavy, or just inefficient queries that would be slow regardless of architecture. If it's repeated read load, caching is the cheapest fix. If reads don't need the absolute latest write, read replicas come next.

Sharding is only for write-bound load or a dataset too large for one instance, and even then, I pick the shard key based on real query patterns first, since a bad shard key guarantees painful cross-shard queries later. Sharding is a one-way door, much harder to undo than to add, so it's genuinely the last option, not the first.

</details>

---

### Q: How do you handle stale cache data after the underlying database record gets updated?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I default to invalidating on write — the same code path that writes to the database also updates or deletes the corresponding cache key, instead of waiting for a TTL to expire it naturally.

When one write affects many cache entries, like a user profile embedded in several cached views, I use a versioned cache key instead of hunting down every single entry — bumping the version on update makes every old entry naturally unreachable.

I always keep a short TTL underneath explicit invalidation too, since that assumes every write path remembers to invalidate correctly, and in a real codebase that eventually breaks somewhere. A short TTL means a missed invalidation self-corrects in minutes instead of staying wrong indefinitely.

</details>

---
