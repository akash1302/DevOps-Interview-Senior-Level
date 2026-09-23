# 🚀 DAY 2 — PRINCIPAL-LEVEL DEEP DIVE
## Kubernetes • Docker • CI/CD • AWS • SRE — Advanced Scenario Q&A

> **Target:** Senior → Principal DevOps / Platform / SRE Engineer
>
> **Focus:** Outage troubleshooting, failure modes, and architectural trade-offs (not definitions)
>
> **Format:** Click **View Answer** to reveal each explanation (flashcard style)

---

## 📚 TABLE OF CONTENTS

1. [Kubernetes — PDBs, Node Drains & CFS Throttling](#1-kubernetes--pdbs-node-drains--cfs-throttling)
2. [Docker — Multi-Stage Builds & Rootless UID/GID Mismatches](#2-docker--multi-stage-builds--rootless-uidgid-mismatches)
3. [CI/CD — ArgoCD Sync Storms & Multi-Account Artifact Promotion](#3-cicd--argocd-sync-storms--multi-account-artifact-promotion)
4. [AWS — Multi-Region TGW DNS Split & VPC DNS Throttling](#4-aws--multi-region-tgw-dns-split--vpc-dns-throttling)
5. [SRE & Observability — Prometheus High-Cardinality Explosion](#5-sre--observability--prometheus-high-cardinality-explosion)

---

### Q: A rolling node drain during a cluster upgrade keeps stalling, and a subset of pods behind a PDB never get rescheduled cleanly — some end up double-evicted with brief 502s at the LB. Diagnose and fix.

**Scenario:** You're draining nodes in batches during an EKS/kubeadm minor-version upgrade using `kubectl drain --ignore-daemonsets`. A `Deployment` with 3 replicas and a `PodDisruptionBudget` of `minAvailable: 2` is causing the drain to hang on some nodes for 10+ minutes, and on other nodes pods are evicted in a way that still causes a brief 502 spike at the ALB, even though the PDB should have prevented it. Separately, the same pods show `CPU throttled` in `kubectl top` despite average CPU usage sitting well under the configured `limits.cpu`.

<details>
<summary><b>🔍 View Answer & Explanation</b></summary>

#### 💥 Core Root Cause:
* **PDB math vs replica placement:** `minAvailable: 2` with 3 replicas only tolerates **one** voluntary disruption at a time. If node drains are run in parallel across batches without checking pod-to-node distribution, the eviction API will correctly *block* the second concurrent eviction — this is the 10-minute hang, not a bug. The drain isn't "stuck," the Eviction API is doing its job and returning `429 Too Many Requests` on retry loop.
* **Readiness gate race causes the 502s:** Even when the PDB permits an eviction, `kubectl drain` sends a `SIGTERM` and the pod is removed from the `Endpoints`/`EndpointSlice` **asynchronously** relative to `kube-proxy`/ALB Target Group deregistration. If there's no `preStop` hook or `terminationGracePeriodSeconds` buffer, the pod can be killed **before** the LB has finished deregistering the target — the PDB protects pod *count*, not *traffic drain timing*.
* **CFS throttling despite low average CPU:** The Completely Fair Scheduler enforces `limits.cpu` using 100ms accounting periods (`cfs_period_us` / `cfs_quota_us`). A pod can average 40% CPU over a minute but still get throttled if it bursts to 100%+ of its quota *within a single 100ms window* (e.g., GC pauses, JIT warmup, request bursts). `kubectl top` reports averages over a scrape interval and completely hides this micro-throttling.

#### 🛠️ Senior-Level Remediation & Architecture Strategy:
* **Fix the drain hang (expected, not broken):** Treat PDB-induced stalls as correct behavior — orchestrate drains node-by-node (`--max-unavailable=1` at the node-pool/Karpenter/CA level) rather than parallel batch draining, and set realistic `drain timeout` values instead of treating retries as a failure.
* **Fix the 502 with a `preStop` sleep + graceful shutdown:** Add `lifecycle.preStop.exec.command: ["sleep", "10"]` (or use `terminationGracePeriodSeconds: 30+`) so the pod stays alive and *serving* long enough for the LB controller to deregister the target via health-check failure before `SIGTERM` actually stops the process. This decouples "pod removed from Endpoints" from "pod killed."
* **Use `readinessProbe` failure as the actual signal**, not pod deletion — have the app fail readiness immediately on `SIGTERM` receipt so kube-proxy/ALB pulls it from rotation fast, while the `preStop` sleep buys time for propagation.
* **Fix CFS throttling at scale — decouple limits from requests:** Set `requests.cpu` accurately (drives scheduling/bin-packing) but either **remove `limits.cpu` entirely** for latency-sensitive services (accepting the noisy-neighbor risk, mitigated by `requests` + node-level `--cpu-manager-policy=static` for critical workloads) or size `limits.cpu` generously above p99 burst, not average.
* **Instrument the real signal:** Alert on `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` (cAdvisor/Prometheus), not on `top`-style averages — this is the only metric that actually exposes per-period throttling.
* **At scale, prefer Guaranteed QoS for latency-critical pods** (`requests == limits` for both CPU and memory) combined with the static CPU manager policy to get exclusive core pinning and avoid CFS quota math altogether for the hottest paths.

</details>

---

### Q: Your production image builds are correct but bloated (1.4GB), and a rootless container mounting a host-bind path fails with `Permission denied` only in CI, not on developer laptops. Explain both failure modes and fix them.

**Scenario:** A multi-stage `Dockerfile` builds a Go/Node service. The final image is 1.4GB despite a multi-stage build being in place, because a single `COPY --from=builder /app /app` layer is dragging in build caches and `node_modules` dev dependencies. Separately, the team moved to rootless containers (Podman or Docker with `userns-remap` / a non-root `USER` in the Dockerfile) for security hardening, and now a host-mounted volume (`-v /data:/app/data`) throws `EACCES` in the CI runner, while the exact same command works fine on an engineer's laptop.

<details>
<summary><b>🔍 View Answer & Explanation</b></summary>

#### 💥 Core Root Cause:
* **Heavy final layer despite multi-stage:** Multi-stage builds only help if the **final `COPY --from=builder`** is scoped to the *artifact*, not the *build directory*. Copying `/app` wholesale drags along `.git`, dev `node_modules`, test fixtures, and compiler caches that were left in the builder stage's working directory — the stage boundary doesn't auto-prune anything, it only avoids carrying forward *previous Dockerfile instructions' layers*.
* **Layer cache invalidation compounding image size over time:** If `COPY package.json package-lock.json` isn't separated from `COPY . .`, every source change invalidates the `npm ci`/`go mod download` layer, so CI ends up re-downloading and re-caching dependencies into a fresh layer each time — and old layers linger in the registry even if not in the final image, but locally `docker history` shows the real bloat sits in one giant `COPY` layer.
* **UID/GID mismatch is a rootless *namespace* problem, not a permissions typo:** In rootless mode, the container's UID 1000 is remapped via `/etc/subuid`/`/etc/subgid` to a **different** UID range on the host (e.g., host UID 100999). A host-mounted directory owned by `1000:1000` on a developer's laptop (where the dev's own UID happens to be 1000, so it "just works" by coincidence) will **not** map to the same effective UID inside the rootless container in CI, where the CI runner's user and subuid range differ — the volume is mounted with the *host's* real UID visible inside the container's user namespace, which the container's non-root user doesn't own.

#### 🛠️ Senior-Level Remediation & Architecture Strategy:
* **Fix image bloat — copy only build artifacts, not directories:**
  ```dockerfile
  FROM golang:1.22 AS builder
  WORKDIR /src
  COPY go.mod go.sum ./
  RUN go mod download
  COPY . .
  RUN CGO_ENABLED=0 go build -o /out/app .

  FROM gcr.io/distroless/static-debian12
  COPY --from=builder /out/app /app
  USER 65532:65532
  ENTRYPOINT ["/app"]
  ```
  This drops the final image to single-digit MBs by copying **only the compiled binary**, and switching the runtime base to `distroless`/`scratch` eliminates the shell, package manager, and CVEs entirely.
* **For interpreted languages (Node/Python), prune before copying:** run `npm ci --omit=dev` (or `pip install --no-cache-dir`) *in the builder stage* and `COPY --from=builder /app/node_modules ./node_modules` rather than copying dev dependencies forward — never `COPY . .` into the final stage.
* **Order layers for cache efficiency:** dependency manifests (`package.json`, `go.mod`) copied and installed **before** source code, so source-only changes don't bust the dependency layer cache.
* **Fix the UID/GID mismatch — align ownership at the boundary, don't fight the user namespace:**
  * Use `podman unshare chown -R 1000:1000 /data` (or the Docker equivalent inside the user namespace) to set host directory ownership to match the **in-container** UID *as seen from the host's remapped range*, not the laptop-coincidental UID.
  * Prefer `:U` / `--userns=keep-id` (Podman) or explicit `chown` in an init container/entrypoint script that runs once as root before dropping privileges, so the mount is normalized regardless of which host's subuid range is in play.
  * Standardize the container's non-root UID/GID as a build-arg (`ARG APP_UID=10001`) and document/enforce the matching host-side `chown` as part of the CI runner's volume provisioning step, so the fix isn't laptop-specific tribal knowledge.
  * As a durable fix, avoid host bind-mounts for CI entirely where possible — use named volumes or tmpfs for ephemeral CI data, since bind-mount UID mapping is inherently host-environment-dependent and this class of bug will recur on every new runner image.

</details>

---

### Q: A single bad Helm values change to a shared ArgoCD `ApplicationSet` triggers hundreds of Applications syncing simultaneously, saturating the Kubernetes API server and causing unrelated deploys across isolated AWS accounts to fail mid-promotion. Design the fix.

**Scenario:** You run ArgoCD with an `ApplicationSet` generating one `Application` per service per environment across dev/stg/prod in three separate AWS accounts (isolated for blast-radius reasons). A shared Helm library chart bump touches a common `_helpers.tpl`, and ArgoCD's auto-sync fires **all** downstream Applications simultaneously — a "sync storm" — hammering the hub cluster's API server, causing controller reconciliation to time out, and worse, a promotion pipeline that was mid-flight for an unrelated prod deploy fails partway because its sync got starved and left resources in a half-applied state.

<details>
<summary><b>🔍 View Answer & Explanation</b></summary>

#### 💥 Core Root Cause:
* **Fan-out without concurrency control:** `ApplicationSet` generators (especially Git-generator with a shared chart dependency) create a topology where **one Git commit fans out to N Applications**, and ArgoCD's default `--app-resync`/auto-sync has no built-in per-change blast-radius limiting — every matching Application re-renders and syncs on the same reconciliation tick.
* **Shared control-plane, isolated data-plane mismatch:** Multi-account isolation was designed for blast-radius on the *target* (AWS account/cluster) side, but the ArgoCD control plane itself (API server, repo-server, redis) is a **single shared bottleneck** — isolating accounts doesn't isolate the sync engine driving them, so a storm in one account's Applications starves API server capacity needed by an unrelated account's in-flight sync.
* **No concurrency/locking around artifact promotion:** Promotion (dev → stg → prod) wasn't treated as a serialized, lockable operation — it assumed the hub cluster always had spare reconciliation capacity, which broke under contention, leaving a prod `Application` `OutOfSync`/partially-applied with no automatic rollback.

#### 🛠️ Senior-Level Remediation & Architecture Strategy:
* **Bound the fan-out with `ApplicationSet` strategies:** Use the `RollingSync` strategy (`spec.strategy.type: RollingSync`) with explicit `matchExpressions` steps (e.g., dev → stg → prod, or canary %-based) so a shared-chart change propagates in **waves**, not all at once — each wave must reach `Healthy` before the next unlocks.
* **Cap repo-server and controller concurrency explicitly:** tune `ARGOCD_CONTROLLER_REPLICAS`, `--app-resync`, `--status-processors`/`--operation-processors`, and shard Applications across controller replicas via `argocd-application-controller` sharding so no single storm exhausts the whole control plane.
* **Isolate blast radius on the control plane too, not just the data plane:** run per-environment (or per-account) ArgoCD instances/ApplicationSets fronted by a shared "hub" only for visibility (e.g., ArgoCD notifications/ApplicationSet in "hub-spoke" mode), so a dev storm architecturally *cannot* starve prod's controller capacity — true isolation means the control plane blast radius matches the account blast radius.
* **Add sync windows and manual promotion gates for prod:** `spec.syncPolicy.syncOptions` + ArgoCD `SyncWindows` to block auto-sync into prod outside change windows, requiring an explicit promotion event (e.g., a Git tag/PR merge to a `prod` overlay) rather than implicit propagation from a shared chart bump.
* **Decouple shared library changes from consumer auto-sync:** pin consumer charts to specific library chart **versions** (semver-pinned dependency in `Chart.yaml`) instead of tracking a moving branch/tag, so a library bump requires an explicit, reviewable version-bump PR per consumer — turning an implicit fan-out storm into deliberate, staggered promotion PRs.
* **Add a promotion lock/queue for cross-account artifact promotion:** use a pipeline-level mutex (e.g., GitLab CI `resource_group`, or an external lock via DynamoDB/etcd) so concurrent promotions into the same environment tier are serialized, and make partial-apply failures automatically trigger `argocd app rollback` to the last known-good revision rather than leaving `OutOfSync` state.

</details>

---

### Q: A multi-region Transit Gateway setup with private Route53 hosted zones works fine most of the time, but intermittently resolves the *wrong* region's backend, and separately a burst of Lambda cold starts silently fails DNS lookups under 1024 PPS. Diagnose both.

**Scenario:** You operate a hub-and-spoke Transit Gateway (TGW) topology across `us-east-1` and `eu-west-1`, with a shared private hosted zone (`internal.company.com`) associated with VPCs in both regions for service discovery. Under normal load, clients in `eu-west-1` intermittently resolve `api.internal.company.com` to an `us-east-1` IP, causing extra latency and occasional TGW inter-region routing failures during a regional degradation. Separately, during a traffic spike a burst of Lambda invocations in one VPC starts seeing sporadic `DNS resolution failed` errors that self-resolve within seconds, with no corresponding Route53 or VPC Resolver errors logged.

<details>
<summary><b>🔍 View Answer & Explanation</b></summary>

#### 💥 Core Root Cause:
* **Single global record, no regional routing policy:** A private hosted zone associated with multiple regions' VPCs is just a **flat namespace** — if `api.internal.company.com` is a single `A`/`CNAME` record (or an `ALIAS` to one region's ALB) without a Route53 **routing policy** (latency-based, geoproximity, or failover), *every* VPC's resolver returns the *same* answer regardless of which region asked. What looks like "intermittent wrong-region resolution" is actually **correct, deterministic behavior** for a policy-less record — the randomness people perceive is really just which record set order/weighted split was configured, or DNS caching (VPC Resolver TTL behavior + client-side stub resolver caching) serving stale answers across an app restart.
* **TGW makes the wrong-region answer "work," masking the real bug:** Because TGW inter-region peering routes the traffic anyway, cross-region resolution doesn't hard-fail — it just adds ~70-100ms RTT and, during a regional degradation event, routes traffic into the *degraded* region instead of failing over, which is the worst-case outcome of not having failover routing.
* **The 1024 PPS silent throttle is a documented but easy-to-miss VPC limit:** Each ENI's built-in VPC DNS Resolver (`.2` address) is limited to **1024 packets per second per ENI**. This is a **hard, silent** limit — exceeded requests are simply dropped, not queued or error-logged in Route53/CloudTrail, because the throttle happens at the VPC network layer, not the DNS service layer. Lambda functions scaling out under burst concurrency each get their own ENI (in VPC-attached mode) or share ENIs in SnapStart/Hyperplane-backed execution environments, and a burst of concurrent cold starts doing simultaneous DNS lookups (SDK client init, secrets manager, etc.) can transiently exceed 1024 PPS on a shared ENI, causing the resolution failures to "self-heal" once the burst subsides and retries succeed — with **nothing** in Route53 Resolver query logs because the packets never reached the resolver.

#### 🛠️ Senior-Level Remediation & Architecture Strategy:
* **Fix the region-split routing with actual Route53 routing policies:** Convert the flat record to **latency-based routing** (one record per region, same name, `SetIdentifier` per region) so each region's VPC Resolver gets the geographically nearest healthy endpoint by default — layer a **failover** routing policy on top (primary/secondary with Route53 health checks against a `/health` endpoint) so a regional degradation actively fails traffic to the healthy region instead of TGW silently routing into the degraded one.
* **Consider geoproximity/geolocation routing if strict data-residency or regional pinning is required**, rather than latency-based, and always pair it with **Route53 health checks** so "nearest" never means "nearest but down."
* **Reduce blind cross-region traffic reliance on TGW as a DNS-correctness safety net:** TGW should be the *fallback path* for legitimate cross-region calls (e.g., active-active writes), not a silent correctness patch for a missing routing policy — audit for services that "accidentally" work today only because TGW happens to route the wrong-region answer successfully.
* **Fix the 1024 PPS DNS throttle at the architecture level:**
  * Deploy **Route53 Resolver endpoints are not the fix here** (that's for hybrid on-prem DNS) — for VPC-internal ENI-level throttling, the actual mitigations are: reduce **per-ENI** DNS query volume by enabling **NAT Gateway or VPC endpoint DNS caching**, and for Lambda specifically, minimize cold-start DNS lookups by **reusing SDK clients outside the handler** (so DNS lookups happen once per execution environment, not once per invocation) and enabling **connection/keep-alive reuse**.
  * For high-fan-out Lambda in VPC, use **Provisioned Concurrency** to reduce simultaneous cold-start bursts, and/or move DNS-heavy dependency resolution (Secrets Manager, Parameter Store) to a **Lambda Extension with local caching** to cut redundant lookups per invocation.
  * At the platform level, monitor `EC2 - VPC Networking / DNS query throttling` is not a native CloudWatch metric — instead proxy-detect it via elevated **Resolver query latency**, application-level DNS lookup timeouts, and correlate spikes with **concurrent execution count** (Lambda) or ENI count/instance density on the affected subnet — and treat "self-resolving DNS errors during traffic bursts" as a standing signature of this limit even without a directly-named CloudWatch metric.
  * Where sustained high query volume is unavoidable (e.g., a dense EKS node with many pods doing DNS-heavy service discovery), deploy **NodeLocal DNSCache** so pod DNS queries resolve from a local cache on the node instead of each pod independently hitting the VPC `.2` resolver, collapsing N pods' PPS into one node-level cached path.

</details>

---

### Q: A Prometheus deployment that was stable for months suddenly starts OOM-killing and query latency spikes into the minutes, right after a service team shipped a change. Diagnose the cardinality blowup and design the fix.

**Scenario:** Your central Prometheus (or Thanos/Mimir-backed) instance has run stably for 8 months. After a routine deploy, `prometheus` pod memory climbs from a steady 12GB to OOM-killing at 32GB within hours, `/metrics` scrape duration for one job balloons, and Grafana dashboards start timing out on queries that used to return in under a second. `promtool tsdb analyze` shows one metric, `http_request_duration_seconds`, now accounts for the overwhelming majority of active series. Investigation shows the service team added a new label to that metric to "help debug" — using the raw request path and, in some handlers, the authenticated user ID.

<details>
<summary><b>🔍 View Answer & Explanation</b></summary>

#### 💥 Core Root Cause:
* **Unbounded label cardinality is combinatorial, not additive:** Prometheus creates one **unique time series** per distinct combination of label values. Adding `path` (raw, un-parameterized URL like `/users/8492/orders/1123`) and `user_id` to a histogram means every unique request path × every unique user × every histogram `le` bucket becomes its own series. A metric that had ~200 series (per route template × status code × bucket) can explode into **millions** of series as soon as high-cardinality dimensions like IDs or raw paths are added — this is combinatorial explosion, not a linear increase.
* **Histograms multiply the damage:** Each `http_request_duration_seconds` sample under a histogram type generates one series **per bucket** (`le="0.1"`, `le="0.5"`, ... `le="+Inf"`) plus `_sum` and `_count` — so a naive high-cardinality label on a histogram metric is typically **10-15x worse** than the same label on a plain counter/gauge, because the label combinatorics apply to every bucket independently.
* **Memory blowup is structural, not just data volume:** Every active series holds an in-memory `head chunk` in the TSDB, plus label-set metadata in Prometheus's series index — so cardinality growth directly drives heap growth (`prometheus_tsdb_head_series` metric), and query latency degrades because PromQL queries that touch this metric now have to scan orders of magnitude more series to compute an aggregate, even a simple `rate()`.
* **This is a common, well-known failure mode ("cardinality explosion")** typically introduced exactly the way described here — a well-intentioned engineer adding a raw identifier or unparameterized path as a label to "help debug," without understanding that Prometheus labels are meant for **bounded, low-cardinality dimensions** (route template, method, status class), not free-form or unique-per-request values.

#### 🛠️ Senior-Level Remediation & Architecture Strategy:
* **Immediate triage — stop the bleeding:** Identify the offending metric/label via `promtool tsdb analyze` or `topk(10, count by (__name__)({__name__=~".+"}))`, then apply a **`metric_relabel_configs`** drop rule at the scrape-config level to drop the high-cardinality label (or the entire metric series matching it) *before* it enters the TSDB:
  ```yaml
  metric_relabel_configs:
    - source_labels: [__name__]
      regex: 'http_request_duration_seconds.*'
      action: drop   # or target specific label removal below
    - action: labeldrop
      regex: 'user_id|path'
  ```
  Note `metric_relabel_configs` (scrape-time, post-exposition) is what's needed here, not `relabel_configs` (which only affects target discovery, pre-scrape).
* **Fix at the source — parameterize the label properly:** The correct fix is in the application/instrumentation code, not just Prometheus config — replace the raw `path` label with the **route template** (`/users/{id}/orders/{id}`, using the framework's route pattern, not `request.url`), and **remove `user_id` from metrics entirely** — user-level identifiers belong in logs/traces (high-cardinality-friendly systems), never in Prometheus label sets.
* **Guardrails to prevent recurrence:**
  * Enforce **`sample_limit`** per scrape target in the scrape config, so a single misbehaving target can't silently balloon global series count — it fails the scrape loudly instead, surfacing the regression at deploy time via `up{job=...} == 0` alerts.
  * Add a standing alert on `count(count by (__name__)({__name__=~".+"})) ` deltas or, more directly, on `prometheus_tsdb_head_series` growth rate — alert on **rate of change**, not just absolute value, so a cardinality regression pages on the deploy that caused it, not hours later at OOM.
  * Bake cardinality review into code review / CI for metrics instrumentation — a linter or convention check that flags label names matching common high-cardinality patterns (`.*_id$`, `path`, `url`, `email`) on new/changed metrics before merge.
* **Architectural mitigation for legitimately high-cardinality needs:** If per-user or per-path granularity is a genuine requirement, that's a signal to route that data to a system designed for high cardinality — **exemplars** (trace IDs attached to histogram buckets, sampled not indexed), or a dedicated high-cardinality backend (e.g., logs/traces in Loki/Tempo, or a purpose-built high-cardinality metrics store) instead of forcing it through Prometheus's label-indexed model.
* **Capacity/isolation as defense in depth:** In a shared multi-tenant Prometheus, use **per-team scrape jobs with individual `sample_limit`s** or federate/shard via Thanos/Mimir so one team's instrumentation mistake degrades only their shard's query performance, not the entire platform's.

</details>

