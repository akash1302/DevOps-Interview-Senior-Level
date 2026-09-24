# 🚀 DAY 2 — PRINCIPAL-LEVEL DEEP DIVE
## Kubernetes • Docker • CI/CD • AWS • SRE — Advanced Scenario Q&A

> **Target:** Senior → Principal DevOps / Platform / SRE Engineer
>
> **Focus:** Outage troubleshooting, failure modes, and architectural trade-offs (not definitions)
>
> **Format:** Simple, plain-English candidate answers, the way you'd actually explain it out loud in an interview.

---

## 📚 TABLE OF CONTENTS

1. [Kubernetes — PDBs, Node Drains & CFS Throttling](#1-kubernetes--pdbs-node-drains--cfs-throttling)
2. [Docker — Multi-Stage Builds & Rootless UID/GID Mismatches](#2-docker--multi-stage-builds--rootless-uidgid-mismatches)
3. [CI/CD — ArgoCD Sync Storms & Multi-Account Artifact Promotion](#3-cicd--argocd-sync-storms--multi-account-artifact-promotion)
4. [AWS — Multi-Region TGW DNS Split & VPC DNS Throttling](#4-aws--multi-region-tgw-dns-split--vpc-dns-throttling)
5. [SRE & Observability — Prometheus High-Cardinality Explosion](#5-sre--observability--prometheus-high-cardinality-explosion)

---

### Q: A rolling node drain during a cluster upgrade keeps stalling, and a subset of pods behind a PDB never get rescheduled cleanly — some end up double-evicted with brief 502s at the LB. Diagnose and fix.

**Scenario:** You're draining nodes in batches during a cluster upgrade. A `Deployment` with 3 replicas and a `PodDisruptionBudget` of `minAvailable: 2` is causing the drain to hang on some nodes for 10+ minutes, and on other nodes pods are evicted in a way that still causes a brief 502 spike at the load balancer. Separately, the same pods show CPU throttling even though average CPU usage looks well under the configured limit.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is actually three separate things happening at once, so I'd break it down one at a time.

The drain hang isn't a bug, it's the PDB working correctly. With `minAvailable: 2` on 3 replicas, only one pod is allowed to go down at a time. If drains are running in parallel across nodes without checking where the pods actually sit, the second eviction attempt gets blocked until the first one finishes.

The 502s happen for a different reason. Even when the PDB allows the eviction, the pod can get killed before the load balancer has actually finished removing it from rotation — the pod removal and the load balancer's deregistration aren't perfectly in sync.

The CPU throttling shows up because the Linux scheduler checks CPU usage in very short windows, like every 100 milliseconds, not over a full minute. A pod can look like it's using 40% CPU on average, but still get throttled if it bursts to 100% for just one of those short windows.

**Drain runs node by node → PDB blocks a second eviction until the first is healthy → pod gets a short delay before shutdown so the load balancer finishes deregistering it first → CPU limits set with real headroom above burst usage, not just average.**

The fix is to drain nodes one at a time instead of in parallel, add a short delay before the pod actually shuts down so the load balancer has time to catch up, and set the CPU limit based on real burst usage instead of the average shown in basic monitoring.

</details>

---

### Q: Your production image builds are correct but bloated (1.4GB), and a rootless container mounting a host-bind path fails with `Permission denied` only in CI, not on developer laptops. Explain both failure modes and fix them.

**Scenario:** A multi-stage Dockerfile builds a Go/Node service. The final image is 1.4GB despite using multi-stage builds, because the final copy step pulls in the whole build folder instead of just the finished file. Separately, the team moved to rootless containers for better security, and now a host-mounted folder throws a permission error in CI, while the exact same command works fine on an engineer's laptop.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

These are two separate problems that just happened to show up around the same time.

The image bloat happens because the final stage is copying the entire build folder, not just the compiled file. That drags along test files, dev dependencies, and build caches that were never meant to reach production.

The permission error is a user ID mismatch, not a real permissions bug. In rootless mode, the container's internal user ID gets mapped to a different, real user ID on the host. On a laptop, that mapped ID happens to match the developer's own user by coincidence, so it looks like it "just works." In CI, the mapping is completely different, so the same command fails.

**Copy only the compiled file in the final stage → image shrinks from 1.4GB to a few MB. Match host folder ownership to the container's actual mapped user ID → permission error goes away in CI too.**

The fix for the image is only copying the final compiled file into a clean, minimal final stage. The fix for the permission error is setting the host folder's ownership to match the real, mapped user ID the rootless container actually uses, not whatever ID happens to work on someone's laptop.

</details>

---

### Q: A single bad Helm values change to a shared ArgoCD `ApplicationSet` triggers hundreds of Applications syncing simultaneously, saturating the Kubernetes API server and causing unrelated deploys across isolated AWS accounts to fail mid-promotion. Design the fix.

**Scenario:** ArgoCD generates one Application per service per environment across three separate AWS accounts. A shared Helm library chart change triggers every single Application to sync at the same time, overloading the shared ArgoCD control plane, and an unrelated production deploy fails halfway because its sync got starved out.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The real issue is that one small change fanned out into hundreds of syncs happening at once, with nothing controlling the pace.

The accounts themselves were isolated for safety, but the ArgoCD control plane managing all of them was shared and became the actual bottleneck. Isolating the AWS accounts didn't isolate the thing actually doing the syncing.

There was also no locking around promotions between environments, so it assumed there'd always be enough capacity, which broke the moment a real storm hit.

**One shared chart change → syncs are rolled out in waves instead of all at once → each wave has to be healthy before the next one starts → production stays gated behind a manual approval, unaffected by the storm.**

The fix is rolling changes out in controlled waves instead of all at once, giving the ArgoCD control plane enough dedicated capacity per environment so one storm can't starve another, and requiring an explicit approval step before anything reaches production instead of letting it happen automatically from a shared chart bump.

</details>

---

### Q: A multi-region Transit Gateway setup with private Route53 hosted zones works fine most of the time, but intermittently resolves the *wrong* region's backend, and separately a burst of Lambda cold starts silently fails DNS lookups under load. Diagnose both.

**Scenario:** A hub-and-spoke network setup spans two regions, with a shared private DNS zone for service discovery. Clients in one region sometimes get routed to the other region's backend, adding latency. Separately, during a traffic spike, a burst of Lambda invocations sees sporadic DNS failures that fix themselves within seconds, with nothing showing up in the logs.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

These are two different problems, and neither one is actually random, even though they look that way at first.

The wrong-region resolution happens because there's a single DNS record shared across both regions, with no rule telling it to prefer the nearest or healthiest region. Every region's resolver gets the exact same answer, so it's not actually random, it's just not smart about location.

The Lambda DNS failures come from a hard limit on how many DNS lookups a single network interface can handle per second. When a lot of Lambda functions cold-start at once, they can briefly go over that limit, and the extra lookups just get dropped silently, with nothing logged, because the limit is enforced at the network layer, not the DNS service itself.

**Add a routing policy so each region's traffic prefers the nearest healthy region → wrong-region resolution stops. Reduce DNS lookups per Lambda cold start by reusing connections outside the handler → fewer lookups per burst → limit stops getting hit.**

The fix for the DNS routing is adding a real routing policy, like latency-based or failover, so each region's traffic actually prefers a healthy, nearby target instead of a flat shared record. The fix for the Lambda issue is cutting down how many DNS lookups happen per cold start, by reusing SDK clients and connections outside the function handler instead of setting them up on every single call.

</details>

---

### Q: A Prometheus deployment that was stable for months suddenly starts OOM-killing and query latency spikes into the minutes, right after a service team shipped a change. Diagnose the cardinality blowup and design the fix.

**Scenario:** A central Prometheus instance has run stably for 8 months. After a routine deploy, its memory usage climbs fast and it starts getting killed for running out of memory, and dashboards start timing out. One metric is found to now account for the overwhelming majority of tracked data, because a team added a new label using a raw request path and a user ID, to help with debugging.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is a classic case of adding the wrong kind of label to a metric, and it grows a lot faster than people expect.

Prometheus creates a completely separate tracked series for every unique combination of label values. Adding something like a raw URL or a user ID means every different user and every different path creates a brand new series, and that multiplies fast, not just adds up a little.

It gets worse specifically because the metric involved is a histogram, which already creates several series per request on its own, so a bad label on a histogram is much more damaging than the same label on a simple counter.

**New high-cardinality label added → series count explodes → memory climbs → Prometheus gets OOM-killed → drop the bad label at the scrape config immediately → fix the actual label in the app code → add a limit so it can't happen silently again.**

The immediate fix is dropping that specific label at the scrape configuration level, so it stops flowing in right away. The real fix is in the application code — using a general route pattern instead of the raw URL, and removing the user ID from metrics entirely, since that kind of per-user detail belongs in logs, not in a metric label. Going forward, I'd also add a hard limit on how many data points one target can send, so a mistake like this fails loudly and immediately instead of quietly filling up memory over a few hours.

</details>

---
