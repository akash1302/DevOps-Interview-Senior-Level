<div align="center">

# DevOps Interview Q&A — Senior / Principal Level

![Topics](https://img.shields.io/badge/topics-14-blue)
![Level](https://img.shields.io/badge/level-Senior--Principal-orange)
![Format](https://img.shields.io/badge/format-Markdown-informational)
![Stack](https://img.shields.io/badge/stack-AWS%20%7C%20K8s%20%7C%20Terraform%20%7C%20CI%2FCD-success)

Candidate-style interview answers — first person, the way a senior engineer actually talks through these problems in a live interview, not a documentation page.

</div>

---

## Table of Contents

| # | Domain | # | Domain |
|---|--------|---|--------|
| 1 | [Docker — Container Internals](#1-docker--container-internals) | 8 | [AWS Infrastructure & Networking](#8-aws-infrastructure--networking) |
| 2 | [Kubernetes — Orchestration & Control Plane](#2-kubernetes--orchestration--control-plane) | 9 | [ECS Fargate + ALB — Production Failure Modes](#9-ecs-fargate--alb--production-failure-modes) |
| 3 | [CI/CD — Jenkins & Pipeline Engineering](#3-cicd--jenkins--pipeline-engineering) | 10 | [IAM, Secrets & Security Boundaries](#10-iam-secrets--security-boundaries) |
| 4 | [Git — Version Control Internals](#4-git--version-control-internals) | 11 | [Serverless — Lambda Architecture](#11-serverless--lambda-architecture) |
| 5 | [Terraform — State & Fundamentals](#5-terraform--state--fundamentals) | 12 | [Observability — Monitoring, Logging & Auditing](#12-observability--monitoring-logging--auditing) |
| 6 | [Terraform — Multi-Account Architecture](#6-terraform--multi-account-architecture) | 13 | [Linux / Scripting — Systems Operations](#13-linux--scripting--systems-operations) |
| 7 | [Terraform — Drift, Recovery & Advanced Ops](#7-terraform--drift-recovery--advanced-ops) | 14 | [System Design — End-to-End Architecture](#14-system-design--end-to-end-architecture) |

---

## 1. Docker — Container Internals

### Q: What are the volume types in Docker, and what are the performance/security implications of each?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

There are three volume types in Docker, and honestly the interesting part isn't the definitions, it's what each one does to your isolation boundary and your ops story.

**Bind mounts** map a path straight from the host filesystem into the container. The thing I always call out is that they bypass the storage driver's copy-on-write layer entirely — so I/O goes directly to the host filesystem, which is great for raw performance, but it means the container can now see and touch whatever host path you gave it. And ownership resolution is just raw UID and GID matching, there's no container-aware permission model sitting on top of that. **Named volumes**, on the other hand, are fully managed by Docker's `local` driver, they live under `/var/lib/docker/volumes/<name>/_data`, and Docker owns creation, mounting, and cleanup — which is why they're the only volume type that survives a `docker system prune` unless you explicitly pass `-a --volumes`. **Anonymous volumes** behave mechanically like named volumes but get a generated hash instead of a name and have no reference tracking, so they orphan silently the moment the container is removed unless you used `--rm` or ran `docker volume prune`.

Now, where I've actually gotten burned by this — we moved to rootless containers for security hardening, and the first thing that broke was a bind-mounted host path throwing `EACCES` in CI but working fine on every engineer's laptop. What I always watch out for is that in a rootless setup, the container's UID gets remapped through `/etc/subuid` to a totally different UID range on the host. On a laptop it "worked" purely by coincidence — the dev's own UID happened to line up. In CI, it didn't. The fix was reconciling ownership explicitly with `podman unshare chown` against the actual in-container UID, not the UID that happened to work locally. For anything stateful in production — databases, queues — I default to named volumes, because I want Docker managing that lifecycle, not me hoping a bind mount survives a redeploy.

</details>

---

### Q: `CMD` vs `ENTRYPOINT` — what's the actual execution model, and when does mixing them break a container's signal handling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The way I explain it is: `ENTRYPOINT` is what becomes **PID 1** inside the container's namespace, and `CMD` just supplies default arguments to it — or the whole command if there's no `ENTRYPOINT` at all.

Now, the critical catch here is the shell-form-versus-exec-form distinction, and this one bites people constantly. If you write `CMD npm start` instead of `CMD ["npm", "start"]`, Docker wraps that in `/bin/sh -c`, and now the shell is PID 1, not your application. When `docker stop` sends `SIGTERM`, that signal goes to PID 1 — the shell — and by default it is not forwarded down to the actual Node process. So the container just sits there for the full stop timeout, ten seconds by default, and then gets hard-`SIGKILL`ed. What's sneaky about this is it looks completely fine in manual testing — you `docker exec` in, kill it directly, everything's clean. It's only in an orchestrated environment doing real rolling deploys that you notice connections aren't draining and every deploy has a jarring cutoff.

If you look at how I actually fix this in practice — always exec form for both `ENTRYPOINT` and `CMD`, so the application itself is PID 1 and gets the signal directly. If I absolutely need a shell for env var expansion or chaining commands, I make sure the last line does an `exec node server.js` rather than just calling it, because `exec` replaces the shell process instead of forking a child — same PID 1 signal delivery guarantee. And for anything running multiple processes, I'll bring in `tini` as a proper init, either via `docker run --init` or directly as the entrypoint, specifically because PID 1 inside a container namespace doesn't get the default signal dispositions and zombie-reaping behavior a normal host PID 1 gets from the kernel.

</details>

---

### Q: A container restarts and all data is gone. What's actually happening at the storage-driver level, and how do you architect around it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In my experience, when someone reports "we lost data on restart," it's almost never a Docker bug — it's the storage driver doing exactly what it's designed to do, and the team just didn't design around it.

Every container gets a writable layer sitting on top of its read-only image layers, managed by `overlay2`, and that writable layer's lifecycle is tied to the **container**, not the image. So `docker rm` — whether that's explicit, implicit through `--rm`, or a scheduler replacing the container on a redeploy — destroys that layer and everything written to it. That's not a failure mode, that's the contract. The actual defect is architectural: the application wrote durable state to a path that was never mapped to a volume, so nothing outside the container's own lifecycle was ever going to survive.

What I do about it in practice is treat the container filesystem as fully disposable from day one, and I verify volume coverage rather than assume it — I'll run `docker diff` against a live container to see exactly which paths are actually being written to, and cross-check that against what's mounted. In Kubernetes, I back that with a `PersistentVolumeClaim`, and I'm deliberate about the `reclaimPolicy` on the underlying `StorageClass` — `Retain`, not `Delete`, for anything that has to survive pod deletion, because the PVC is what survives rescheduling, not the pod's ephemeral writable layer. If a service assumes local disk persists across restarts without a volume behind it, that's a design defect I'll flag in review, not something I patch after the fact.

</details>

---

### Q: How do you reclaim disk on a Docker host without risking an active build cache or in-use image?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The direct answer is you don't run `docker system prune -a` blind, especially not on a CI runner — I learned that one the expensive way.

`system prune -a` takes out every stopped container, unused network, dangling image, and critically, every image not currently referenced by a running container — which includes tagged images you might actually want for a rollback. The way I actually found out how bad this was: we had it running as a "cleanup step" mid-pipeline on a shared runner, and it evicted the layer cache a completely unrelated concurrent job depended on. That job went from a cache-hit build in under a minute to a full re-pull and re-build. Nobody touched the Dockerfile — the pipeline just quietly got slower, and it took a while to trace it back to that one prune step.

Now what I do instead is scope it deliberately — `docker image prune -f --filter "until=72h"` reclaims space from genuinely stale layers without touching anything from a recent build. On CI runners specifically, I keep the BuildKit cache on its own dedicated volume, mounted via `--mount=type=cache`, and explicitly excluded from whatever prune job runs — so cache eviction is a decision I make on its own schedule, not a side effect of general cleanup. And I don't wait to discover disk pressure when builds start failing — `docker system df` goes into the standard metrics, with an alert on `/var/lib/docker` utilization, so it's a scheduled maintenance action, not a fire drill.

</details>

---

## 2. Kubernetes — Orchestration & Control Plane

### Q: What do taints and tolerations actually enforce at the scheduler level, and what do they *not* protect against?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A taint is a key-value-effect triplet — `NoSchedule`, `PreferNoSchedule`, or `NoExecute` — sitting on a `Node` object, and it's checked by the scheduler's predicate filters before a pod ever gets placed. A toleration on the pod side is just the matching bypass for that filter. And I always make a point of saying this clearly: a toleration doesn't attract a pod to a node, it only permits it to land there if the scheduler happens to consider it.

Now, the critical catch here — and this is usually where the follow-up question goes — is that taints and tolerations only control **scheduling eligibility**. They give you zero enforcement against a pod that's already running and got placed some other way — a manual `kubectl` bypass, a node that mis-registered without the expected labels, or a `DaemonSet` that forgot to declare the toleration for its own node pool's taint. If you look at how the control plane actually handles this, it's a soft, scheduling-time hint honored by kube-scheduler — it is nowhere near a kernel-level isolation boundary the way a namespace or a cgroup is. The one effect that actually reaches into already-running pods is `NoExecute` — that one actively evicts. `NoSchedule` and `PreferNoSchedule` only ever affect future placement decisions.

Where I've actually used this in production — we had a GPU node pool that kept getting non-GPU workloads scheduled onto it because a toleration alone was enough to let them land there. The fix was pairing the taint with a `nodeAffinity` using `requiredDuringSchedulingIgnoredDuringExecution` — toleration says "may run here," affinity says "must run here," and you genuinely need both if you want dedicated placement. And if the actual goal is security isolation rather than just workload dedication, I'll go further — combine it with `NetworkPolicy` for the traffic boundary, and if the threat model calls for real separation, a dedicated node pool with `PodAntiAffinity` so nothing else gets co-scheduled there at all. Taints alone were never meant to stop noisy-neighbor contention or contain a container escape.

</details>

---

### Q: Is pod-to-pod traffic allowed by default, and how do you enforce a default-deny posture without breaking DNS or control-plane traffic?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Yes, by default the pod network is completely flat and permissive — any pod can reach any other pod's IP on any port, cluster-wide, unless something explicit says otherwise. That's upstream Kubernetes behavior, and `NetworkPolicy` is entirely opt-in — it also needs a CNI that actually enforces it, which not all of them do out of the box.

The way I explain the danger here is that the very first time a team flips on a default-deny-all `NetworkPolicy` at the namespace level, they almost always break DNS resolution, because CoreDNS lives in `kube-system`, which is a different namespace, and cross-namespace traffic gets blocked along with everything else unless you've explicitly carved out an exception. Same story for anything the control plane needs to call back into the pod for — admission webhooks, metrics-server scraping. I've seen this exact self-inflicted outage more than once on a team's first rollout of `NetworkPolicy`.

So the way I actually roll this out — baseline is deny-all ingress and egress per namespace, and then I explicitly punch a hole for DNS:

```yaml
egress:
  - to:
      - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}
    ports: [{protocol: UDP, port: 53}, {protocol: TCP, port: 53}]
```

And I never flip this on cluster-wide in one shot. I'll apply it in a non-prod namespace first, verify with an `nslookup` from inside a pod plus actual application health checks, and only then promote it. If the requirement is L7-aware policy — method or path-level enforcement, not just IP and port — that's where I'd bring in Cilium, because native `NetworkPolicy` and Calico are strictly L3/L4, which isn't enough if you're actually trying to run a zero-trust posture.

</details>

---

### Q: `StatefulSet` vs `Deployment` — what specific guarantees does a `StatefulSet` provide that make it non-optional for stateful workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A `Deployment`'s `ReplicaSet` creates pods with randomly generated suffixes and gives you zero ordering guarantee on create, delete, or scale — any replica is interchangeable with any other, which is exactly fine for stateless workloads and exactly wrong for anything that needs stable peer identity, like Kafka broker IDs or an etcd member.

`StatefulSet` gives you four things a `Deployment` structurally cannot: stable, predictable pod names like `pod-0` and `pod-1`; a stable network identity through a headless `Service`, so each pod gets its own resolvable DNS record instead of being load-balanced behind a shared VIP; ordered, sequential rollout — `pod-1` genuinely will not start until `pod-0` is `Running` and `Ready`; and stable per-pod storage through `volumeClaimTemplates`, where each replica gets its own PVC that reattaches to the same ordinal after a reschedule. What I always flag when this comes up is that deploying stateful software on a plain `Deployment` will look completely fine right up until the first rolling update or node eviction — that's the moment pod identity and storage binding stop being guaranteed to match, and that's where I've seen actual data corruption trace back to.

In practice, I default to `podManagementPolicy: OrderedReady` for anything with real peer-aware clustering logic — etcd, ZooKeeper, Kafka in KRaft mode. If the app handles its own peer coordination and doesn't need ordering, I'll switch to `Parallel` specifically to remove that sequential startup bottleneck. And one thing that's bitten a project I worked on — pin the storage class to `WaitForFirstConsumer` binding mode, so the PV actually gets provisioned in the same AZ the pod is scheduled into. Without that, you get cross-AZ EBS attachment failures, and that's a surprisingly common `StatefulSet` incident that has nothing to do with the application at all.

</details>

---

### Q: A pod is stuck in `CrashLoopBackOff`. Walk through the exact diagnostic sequence and what each signal tells you.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First thing I tell people: `CrashLoopBackOff` isn't the error, it's the kubelet's exponential backoff wrapper around a container that keeps exiting — starting at ten seconds and capping out at five minutes. It tells you the kubelet gave up retrying at the current interval. It tells you nothing about *why* the container exited, and that's the part people skip past.

The actual reason lives in one of three places depending on what actually happened. There's the exit code itself — `kubectl describe pod` shows you `Last State: Terminated`, the reason, and the exit code. There's the application's own stdout and stderr, which you have to pull with `kubectl logs --previous`, because the current container instance has already restarted and its logs are fresh. And there's probe failures, which show up distinctly in the pod's events as `Liveness probe failed`, separate from an actual application crash. Now, the one I always check for first — exit code `137` is `SIGKILL`, and in practically every case that's an OOMKill, which you'll see confirmed as `Reason: OOMKilled` — that's the kernel's cgroup memory controller enforcing `limits.memory`, not a Kubernetes bug. Exit code `1` or anything application-specific means the process itself failed, and that's a code problem, not a platform problem.

My actual sequence: `describe pod` for the exit code and events first, then `logs --previous` for the crashed instance's actual output, and if it's `137`, I go cross-reference the configured `limits.memory` against real peak RSS from whatever metrics backend we've got. If it's probe-induced, I compare the `initialDelaySeconds` and `failureThreshold` against how long the app genuinely takes to start — and if it's a slow starter, I'll add a separate `startupProbe` rather than just loosening the liveness probe, so we don't kill it mid-initialization. And if it really is OOMKilled, I treat that as a capacity-planning gap, not a fluke — either raise the limit based on real observed p99 RSS with headroom, or go chase an actual leak if RSS is climbing unbounded instead of plateauing.

</details>

---

### Q: An application deployed to EKS isn't reachable externally. What's the exact layer-by-layer isolation boundary you check, in order?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The way I always frame this is: external reachability crosses five separate failure domains, and the symptom you see at the client — connection refused, timeout, a 502 — doesn't tell you which one broke. So I don't guess, I walk the chain in order, because checking out of order just wastes time chasing something downstream of the actual fault.

I start with pod readiness — `kubectl get endpoints` on the service. If that's empty, the service's selector isn't matching any `Ready` pod, and nothing past this point matters until that's fixed. Next, service type and target port — I'll confirm the `targetPort` in the service spec actually matches what the container is listening on, usually by exec-ing in and checking with `netstat`. A port mismatch here is completely silent, there's no error surfaced anywhere. Third, ingress and load balancer provisioning — `kubectl describe ingress` to check the AWS Load Balancer Controller's reconciliation events, because a missing `IngressClass` or a mismatched `ingressClassName` means the ALB just never gets created, again with nothing obviously screaming at you from `kubectl get` output.

Fourth — and this is the one that trips people up most in EKS specifically — the security group chain. The ALB's SG needs to allow inbound on the listener port, and separately, the node or pod SG needs to allow inbound from the ALB's SG on the target port. If you're using security groups for pods, those are genuinely separate from the node SG, and that split is the most common silent-fail point I've run into on EKS. Last, DNS and Route53 — confirming the ALB's DNS name resolves and that whatever CNAME or alias record points at it is actually current, because stale records after an Ingress got recreated are a real thing, especially post-migration.

</details>

---

### Q: How does cross-pod communication actually traverse the stack inside an EKS cluster, mechanically?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The thing that surprises people coming from self-managed clusters is that EKS pod-to-pod traffic isn't an overlay network at all — the VPC CNI assigns every pod a real, routable IP straight out of the VPC's CIDR, using ENI secondary IPs. So it's native VPC routing, not encapsulated traffic the way Flannel or Calico's overlay mode works. And the practical consequence of that — which is genuinely a capacity-planning trap — is that pod density per node is bound by ENI IP capacity for that instance type, and that's easy to miss until you're staring at pods stuck in `ContainerCreating`.

Service-level discovery is a separate layer on top of that — CoreDNS answers the `ClusterIP` A-record, but that `ClusterIP` is a virtual IP, nothing's actually listening on it. `kube-proxy` programs `iptables` or `IPVS` rules that DNAT the traffic via the kernel's netfilter hooks to one of the real backing pod IPs from the `Endpoints` object. And if you look at how that scales — `IPVS` does hash-table lookup, so it's O(1) regardless of service count, while `iptables` mode walks a sequential rule chain, which is O(n) against the number of services. Past a few hundred services, that `iptables` chain traversal measurably shows up as `kube-proxy` reconciliation lag and per-packet DNAT overhead.

For anything running at real scale, I'll switch `kube-proxy` to IPVS mode specifically to get off that linear scan. And on the CNI side, I keep an eye directly on the `aws-node` DaemonSet's `ipamd` metrics for warm-IP-pool exhaustion — a pod stuck in `ContainerCreating` with a CNI-related event in its description is almost always this, not a scheduler problem. If node density needs to go higher than the default secondary-IP-per-ENI limit allows, that's exactly what prefix delegation mode on the VPC CNI is for.

</details>

---

## 3. CI/CD — Jenkins & Pipeline Engineering

### Q: What are the trade-offs between Jenkins deployment models (static EC2, Docker, Helm-on-EKS), and which failure modes does each eliminate or introduce?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I've run Jenkins all three ways at different points, and honestly the deployment model tells you a lot about what kind of incidents you're signing up for later.

A static EC2 install is the simplest to stand up and the worst to operate at scale — it's a single point of failure with hard-capped build concurrency tied to that one instance's CPU and memory, and if the controller crashes, your entire CI surface is down with a manual recovery path, AMI restore or an EBS snapshot. Running just the controller in Docker fixes environment drift for the controller process itself, but it does nothing for agent isolation — unless the agents are also containerized, builds are still executing on shared, long-lived infrastructure, and one build's leftover state, cached deps, polluted environment variables, can quietly bleed into the next run. What I actually run in production is Helm on EKS with the Kubernetes plugin — the controller comes up as a `StatefulSet` with `JENKINS_HOME` on a PVC, and every single build gets its own ephemeral pod agent that's destroyed the moment the job finishes. That eliminates build-to-build state bleed by construction, not by discipline, and agent capacity scales with the cluster's autoscaler instead of a fixed box.

The way I set this up concretely — controller as a `StatefulSet` on a `StorageClass` that supports snapshots, so a controller reschedule reattaches to the same volume rather than starting fresh. Agent specs get defined declaratively as `podTemplate` blocks right inside the `Jenkinsfile`, which means agent provisioning is version-controlled alongside the pipeline itself instead of living in some global Jenkins UI setting nobody remembers changing. And I always set explicit resource requests and limits on those agent pod templates — an unbounded agent pod is the single most common cause I've seen of node-level starvation on a cluster that's running CI and application workloads side by side.

</details>

---

### Q: A Jenkins pipeline fails intermittently, not on every run. What's the deterministic debugging sequence, and how do you distinguish a flaky test from an infrastructure race condition?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The first thing I do is refuse to call it "flaky" until I've actually proven it's the test and not the infrastructure — because those two failure classes need completely different fixes, and conflating them is how teams end up papering over a real race condition with a retry loop.

Genuine test flakiness is non-determinism inside the test or build itself — timing-dependent assertions, unseeded randomness, a shared mutable fixture. An infrastructure race is something else entirely — an ephemeral agent that wasn't fully ready when the job started, DNS that hadn't propagated yet for a freshly provisioned dependency, credentials rotating mid-build. If you just slap a retry or a sleep into the test code to make an infra race go quiet, you haven't fixed anything — you've just lowered the visible failure rate, and it comes back the moment load goes up or you land on a slower runner.

What I actually do — correlate the failure timestamps against agent provisioning events, not just the build log. If failures cluster right after a scale-up event, that's an infra race, full stop, not test flakiness. I'll also check the console output for credential binding order, because Jenkins injecting credentials into a freshly spun-up ephemeral agent can genuinely race the job's first command if the pod template's init sequence isn't explicitly ordered. And a trick that's saved me a lot of guessing — rerun at fixed concurrency of one. If the failure disappears completely under serialized execution, that's a shared-resource race, test DB contention or a port conflict between parallel agent pods on the same node, not application logic. If after all that it really is just a flaky test, I quarantine it with a ticket and a hard SLA to fix — I don't let blanket retries silently erode what the pipeline is actually telling us.

</details>

---

### Q: How do you manage Jenkins plugin upgrades on a Helm-deployed instance without an untested plugin breaking the production pipeline?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The core problem is that plugin changes made directly through the Jenkins UI on a live controller are completely untracked and unreviewed — a version bump can break pipeline DSL compatibility with zero rollback path besides manually reinstalling the old version. And on a Helm-managed controller specifically, that UI change doesn't even survive — the chart's declared plugin list will just overwrite it on the next `helm upgrade`, which is confusing the first time it happens to you.

So the way I always explain it is: the `values.yaml` file with pinned plugin versions is the single source of truth, full stop. Anything applied through the UI is configuration drift, and it's getting reverted on the next chart sync whether anyone intended that or not. Before I ever bump a plugin version in production, I spin it up on a non-prod Jenkins instance from the exact same Helm chart, with the bumped version, and I run our actual production `Jenkinsfile`s against it — not a synthetic smoke test, the real pipelines. And when I do roll it out, it's `helm upgrade jenkins -f values.yaml jenkins/jenkins --atomic` — that `--atomic` flag is doing real work there, it automatically rolls back the release if the upgrade fails its health checks, so you never end up stuck with a half-applied plugin state on a live controller.

</details>

---

### Q: The Jenkins admin credential is lost and the controller runs on Kubernetes with no external secret backup. What's the recovery path, and how do you prevent this from being a single point of failure again?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

It depends entirely on where in the lifecycle you lost the credential, and that's usually the first clarifying question I'd ask back in an interview, honestly.

If this is still the initial setup wizard state, it's easy — Jenkins writes a one-time password to `/var/jenkins_home/secrets/initialAdminPassword`, and you just `kubectl exec` in and cat that file. But if this is a real admin account, post-setup, that file is long gone, and now you're in shell-access-and-Groovy-surgery territory — you exec into the controller pod, drop an init script under `/var/jenkins_home/init.groovy.d/` that resets the admin user through the Script Console API, or if shell-level access is blocked by policy, you're restoring from the last `JENKINS_HOME` backup. Neither of those is a fun place to be during an incident.

What I actually push for afterward, every time, is fixing this architecturally rather than procedurally. I'll configure Jenkins against the org's actual SSO — SAML or OIDC — instead of relying on local Jenkins accounts at all, which just removes the single-admin-credential dependency entirely going forward. And separately from that, I make sure there are scheduled PVC snapshots of `JENKINS_HOME`, specifically so that credential-store recovery doesn't have to depend on Groovy console surgery while everyone's already under incident pressure.

</details>

---

### Q: Design a rollback strategy set that covers every layer of a deployment — application, infrastructure, and pipeline state.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The mistake I see most often is people treating rollback as one operation — like a single Git revert somehow reverts running infrastructure. It doesn't. Every layer of the stack has its own independent revision history and its own independent rollback mechanism, and conflating them is exactly how a rollback "succeeds" in the repo while production stays completely unchanged.

So the way I walk through this — at the application layer in Kubernetes, `kubectl rollout undo deployment/<name> --to-revision=<n>` is backed by the `ReplicaSet`'s own revision history, it's instant and in-cluster, no rebuild needed. If it's Helm-managed instead, I use `helm rollback` specifically, because that reverts the whole rendered manifest set including anything driven by `values.yaml`, not just the raw `Deployment` object the way `kubectl rollout undo` does. Infrastructure is the odd one out — Terraform has no native rollback command at all. Recovery there is either re-applying a previous Git-tracked revision of the code, or restoring a prior state version from S3 versioning if the state itself got corrupted — which is exactly why reviewing `plan` output before `apply` matters so much, because after-the-fact rollback is never equivalent to having prevented it in the first place. For compute, if we're on an AMI-based pipeline, rollback is swapping the Auto Scaling Group's launch template back to a prior version and triggering an instance refresh — and that rollback path only exists at all because we build immutable AMIs instead of mutating instances in place. And at the source layer, it's `git revert`, not `git reset --hard`, on anything shared — revert is additive and safe, reset rewrites history that other engineers may have already pulled.

</details>

---

## 4. Git — Version Control Internals

### Q: Explain the branching model you enforce, and specifically why `merge` vs `rebase` changes the risk profile of a shared branch.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Merge creates a new commit with two parents and preserves the actual chronological history of both branches — and the reason that's safe on a shared branch is that it never rewrites commit hashes anyone else might already have in their local clone. Rebase does the opposite — it replays your commits onto a new base and generates brand new hashes for every one of them. That's functionally a history rewrite, and if you rebase a branch other people have already pulled, you've just forced every one of those clones into a diverged history that needs a forced sync to reconcile. That's the actual mechanical reason "don't rebase shared branches" is a hard rule and not just a style preference — it's not about aesthetics, it's about what happens to everyone else's local repo.

On the model itself — I've used Gitflow, and it's fine for teams running scheduled release trains with genuinely long-lived release branches, but it comes with real merge ceremony. For teams shipping continuously, I lean toward trunk-based development instead — short-lived feature branches merging straight into `main`, behind feature flags for anything not ready for full exposure yet.

Where I do use rebase is purely on a branch I own exclusively and haven't pushed anywhere shared yet — cleaning up my own commit history with `rebase -i` before the first push is completely safe. Once it's shared, that door closes. And at the platform level, I enforce linear history on `main` through required PR merge strategy — squash or rebase merge enforced by the platform, not developer discipline — because that's what actually makes `git bisect` and rollback tractable once you're at scale. If a team's shipping continuously, I'd rather see trunk-based development with flags than long-lived `release/*` branches, because those release branches just accumulate drift from `main` and reintroduce the exact merge-conflict pain Gitflow was supposed to solve in the first place.

</details>

---

### Q: How do you cleanly squash a range of commits before merging, and what breaks if you do it after pushing to a shared branch?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git rebase -i HEAD~n` is the tool — it rewrites the last `n` commits and lets you mark them `squash` or `fixup` to collapse them down, and every one of those commits gets a new hash in the process. The part people miss is what happens if that branch is already pushed and shared — any collaborator who's already based work on those old commit hashes now has a diverged ancestry, and their next merge is going to throw spurious conflicts or duplicate commits they didn't cause.

So my actual workflow — interactive rebase locally, mark the commits I want collapsed, and then force-push, but specifically with `--force-with-lease`, never a bare `--force`. The difference matters: `--force-with-lease` refuses the push if the remote has commits I haven't fetched yet, which protects against silently clobbering a teammate's concurrent push to the same branch. And if for some reason the branch is already shared and I still need to squash it, I don't just do it quietly — I announce the rewrite first, and afterward everyone re-clones or does a hard reset against the new remote state rather than trying to merge through the divergence.

</details>

---

### Q: The `.git` directory is gone from a working copy. What's actually recoverable, and what isn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`.git` is the entire object database — every commit, every tree, every blob, every ref lives in there. Delete it, and you've lost all of that: local history, any local branches that were never pushed anywhere, stashes, the reflog, all of it. None of that is recoverable from the working tree alone, because the working tree is just a checked-out snapshot — it doesn't carry history metadata of its own.

If there's a remote and everything was actually pushed, this is a non-event — `git init`, add the remote, fetch, and hard reset onto the remote branch, and you're back to exactly where you were, because nothing was actually lost. The part that's genuinely unrecoverable is any local commit that was never pushed — and that's exactly the argument I make for enforcing frequent pushes as policy, or at minimum a scheduled backup of `.git`, rather than treating this as some rare edge case you deal with after it happens. And to be clear, uncommitted working-tree changes are gone too if the deletion touched the working directory at all — that's a filesystem backup problem at that point, completely outside anything Git's recovery model can help with.

</details>

---

### Q: `git fetch` vs `git pull` — what's the actual difference in terms of working-tree risk?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`git fetch` downloads remote refs and objects into your local `.git` database and stops there — it never touches your working tree or your current branch pointer, so there's zero risk of clobbering uncommitted local work. `git pull` is fetch immediately followed by an automatic merge, or a rebase if you've configured `--rebase` — and that part does mutate your working tree right away. If your local uncommitted changes happen to conflict with what's coming in, you can end up in a partially-merged, conflicted state with no warning beforehand.

The way I actually apply this — for interactive day-to-day developer use, `git pull` is completely fine. But I never let it into scripted automation or CI without guardrails. My default there is fetch, then an explicit diff against the remote branch for review, then a deliberate merge or rebase as its own separate step. And if `pull` does end up in an automated context, it's getting `--ff-only` at minimum, so divergence causes a loud failure instead of a silent auto-merge nobody reviewed.

</details>

---

## 5. Terraform — State & Fundamentals

### Q: How do you bring an out-of-band-created AWS resource under Terraform management without recreating it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform import` is the command, but the thing people get wrong is assuming it does more than it actually does — it only writes a state entry mapping a resource address to a real infrastructure ID. It does not generate any HCL for you. So if the config for that resource address doesn't already exist, or doesn't match the resource's real attributes, the very next `plan` sees a diff between "no config" or mismatched config and the actual imported state, and it'll propose something destructive.

So my actual sequence is always: write the HCL resource block first, matching the real configuration as closely as I can get it, then run `terraform import` against that address, and then immediately run `plan` — not apply, plan. A clean plan with no changes is what confirms the HCL genuinely reflects reality; anything short of that has to get reconciled in the config before it's safe to touch. If I'm importing at real bulk — a whole account's worth of resources someone clicked together manually — Terraform 1.5 added `plan -generate-config-out`, which scaffolds HCL directly from existing state. I still treat that generated file as a starting point to hand-review and refactor into proper modules, never as something final I'd merge as-is.

</details>

---

### Q: What specifically breaks when Terraform state is kept local in a multi-engineer team, beyond "it's not backed up"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The backup angle is the obvious one, but honestly it's not even the real problem. The real problem is there's no locking primitive at all with local state — if two engineers run `apply` against the same state file at the same time, they're racing on a read-modify-write, and whichever one finishes second just silently overwrites the first one's state changes. Now Terraform's tracked state is inconsistent with what's actually running in the account — resources the first apply created are effectively orphaned from Terraform's perspective, it doesn't know they exist anymore.

And there's a second failure mode that's just as bad — there's no single source of truth. Each engineer's local state only reflects what *they've* personally applied, so drift between engineers doesn't surface until someone's `plan` unexpectedly proposes destroying resources they didn't even know existed. That's discovered reactively, in the worst possible moment, instead of being prevented structurally from the start.

For any team-based usage, a remote backend with native locking isn't optional in my book. S3 with `use_lockfile` on newer Terraform versions, or the older DynamoDB lock table pattern — either way, that serializes `apply` operations at the state level, so a second `apply` blocks on the lock instead of racing. And I always turn on S3 bucket versioning for the state bucket too, because that's the actual rollback mechanism for corrupted or bad state — Terraform itself has no native `state rollback` command, so restoring a prior object version directly is genuinely the only path back.

</details>

---

### Q: What does a real Terraform testing/validation pipeline enforce before `apply` ever runs against production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform validate` gets overrated in a lot of these conversations — it only checks HCL syntax and internal consistency, referenced variables existing, types lining up. It has zero awareness of what the provider actually cares about. So a config that's perfectly valid syntactically can still violate a real AWS API constraint — an invalid CIDR overlap, an IAM policy that's too big — and that only surfaces at `apply` time against the live API, which is way too late in the loop.

The way I actually layer this is by cost of failure, cheapest checks first. `terraform fmt -check` for style, essentially free. Then `validate` for syntax. Then `tflint`, which is provider-aware and actually catches things like invalid instance types or deprecated arguments before you even get to `plan`. Then `tfsec` or `checkov` for policy-as-code — public S3 buckets, overly permissive security groups, unencrypted EBS, that category of thing. Then `plan`, reviewed as a required check on the PR itself. And `apply` is gated behind manual approval specifically for production. The whole point of stacking it this way is that `apply` should never be the first point anything gets caught — every layer above it exists purely to shift the failure left, away from the moment it would actually touch live infrastructure.

</details>

---

## 6. Terraform — Multi-Account Architecture

### Q: Design the module/state topology for a Terraform codebase spanning dev/staging/prod across separate AWS accounts.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The tension I'm always balancing here is reuse against isolation. If you go with one monolithic root module and just swap variables per environment, you've created a shared blast radius — a bad `apply` can touch every environment's state if the backend isn't isolated separately too. Go the other way, fully duplicated code per environment, and now it drifts out of sync over time with nothing catching the divergence.

The way I actually structure it — reusable modules per component, so `modules/vpc`, `modules/ecs`, `modules/rds`, with zero environment-specific logic baked in anywhere inside them. Environment differences only ever come in through input variables. Then thin root modules per environment — `envs/dev`, `envs/staging`, `envs/prod` — that instantiate those shared modules with environment-specific `.tfvars`. And critically, each of those root modules has its **own** backend configuration pointing at a separate S3 bucket and key per account, so state isolation actually matches account isolation, not just logically but structurally. On the CI side, the pipeline assumes an account-scoped IAM role through STS before it ever runs `plan` or `apply`, and that pipeline identity can only assume roles into accounts it's explicitly trusted for — there are no static per-account credentials sitting in CI anywhere.

</details>

---

### Q: How do you prevent a Terraform state read/write in one AWS account from ever touching another account's state, structurally rather than by convention?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the only thing preventing cross-account state access is "engineers remember to point at the right backend config," that's not actually a control, it's a hope. A copy-pasted backend block or a misconfigured CI variable will silently apply changes against the wrong account, and you won't know until the plan output shows unexpected diffs — and by the time you're looking at that, `apply` may have already run in CI.

So what I actually do is make this structural. One S3 bucket, with its own locking, per AWS account — not a shared bucket with per-environment key prefixes. That matters because it means a misconfigured IAM role literally cannot read or write another account's state, since the bucket policy only grants access to that account's own CI role in the first place — there's no path around it. The bucket policy itself is scoped to the specific CI/CD role ARN, not broad account-level access, so state access is an explicit, auditable grant rather than an implicit side effect of just being in the account. And the backend config per environment is generated or selected by the pipeline logic based on the target environment — nobody's manually editing a backend file per run, which removes the human-error vector entirely.

</details>

---

### Q: How do you keep secrets (DB credentials, API keys) out of both the Terraform codebase and the state file, given that Terraform state stores resource attributes in plaintext by default?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

People usually get the first half of this right — nobody hardcodes a password directly in `.tf` files anymore. What catches teams off guard is that Terraform state itself is a plaintext JSON document by default, so any attribute Terraform manages — including a secret passed as a literal variable value — gets written into that state verbatim. So even if the secret's never in source, passing it as a variable still leaks it into state, and state is a durable artifact that anyone with S3 read access to that bucket can read, not just someone with `apply` permission.

The rule I hold to is: never pass secret literals as Terraform variables, period. Secrets live in Secrets Manager or SSM Parameter Store as SecureStrings, out of band, and get referenced through a data source — what actually lands in state is the ARN reference, not the value. Now, there are exceptions where a resource itself directly consumes the secret as an argument — RDS's `master_password` is the classic one — and for those specific cases I'll use `manage_master_user_password = true` so Terraform never even sees the plaintext value in the first place. On top of all that, I still turn on SSE-KMS on the state bucket and restrict read access to the CI role only, as defense in depth — the plaintext-in-state issue gets mitigated through access control and minimizing what ever touches the resource graph, not eliminated purely by encryption.

</details>

---

### Q: Trace exactly what happens end-to-end when a Terraform change merges to `main` in a CI/CD-driven multi-account pipeline.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Every stage in this pipeline exists to catch a specific class of failure before it hits real infrastructure, and the way I explain it is — if you collapse any of these stages, you've removed the one point where a human could've caught an unintended destroy before it actually executes.

Walking through it — `terraform init` against the environment-scoped remote backend first. Then `plan -out=tfplan`, and that output gets posted as a PR comment or pipeline artifact specifically for human review — this is the real control point in the whole flow, because a plan showing an unexpected destroy-and-recreate on something stateful is your last chance to stop it. From there, a manual approval gate, and I only put that gate in front of production specifically — dev and staging move faster because the cost of being wrong there is much lower. CI/CD then assumes the target account's deployment role via STS, scoped to just that stage's environment. Then `apply` runs against the exact saved plan artifact — not a fresh `plan` re-run at apply time — which guarantees what got reviewed is literally what executes, because state can't have drifted in between review and apply if you're applying the saved plan file. And the plan and apply logs get retained as audit evidence, tied back to the PR and commit that triggered the whole run.

</details>

---

## 7. Terraform — Drift, Recovery & Advanced Ops

### Q: Someone manually changes a resource in the AWS console that Terraform manages. What does `terraform plan` actually show you, and how do you resolve it correctly versus incorrectly?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`terraform plan` does a refresh first — it reads the real current attribute values through the provider API — and then diffs that against whatever's declared in HCL. A manual console change shows up as a diff where Terraform proposes reverting the resource back to what's in the code. And I always make sure people understand this is completely expected behavior — Terraform has no concept of *who* made a change, it only knows whether reality matches the declared config.

Now, the critical catch, and this is the part that actually matters in an interview — the dangerous failure mode isn't the drift itself, it's resolving it wrong. If someone reflexively runs `apply` without actually reading *what* the diff is reverting, and that manual change was, say, an emergency fix during an active incident — someone widened a security group by hand to unblock something — a blind apply silently reverts that fix and can reintroduce the exact incident that was just resolved.

So what I actually do every time is inspect the diff before touching anything. If it's genuinely accidental drift, fine, apply reverts it and restores IaC as the source of truth. But if it was an intentional, undocumented fix, the right move is updating the HCL itself to match the new desired state, and then applying a no-op confirming plan — you're codifying the fix, not reverting it. And structurally, I push for policy that production changes only happen through Terraform, paired with AWS Config rules or CloudTrail-based alerting on manual console changes to Terraform-managed resources — so drift gets triaged the moment it happens instead of getting discovered at the next unrelated plan.

</details>

---

### Q: How do you refactor a Terraform module that's already deployed in production without a destroy/recreate cycle on live resources?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The thing that trips people up here is that Terraform's resource addressing is based on the module path plus the resource name in HCL — it's not tied to a stable resource ID under the hood. So renaming a module, restructuring a `for_each` key, or just changing a resource's local name inside a module all change that address, and Terraform reads an address change as "old resource destroyed, new resource created" — even though the actual cloud resource hasn't changed at all.

My approach is to version the module explicitly, a Git tag or a registry version pin on the `source` argument, and introduce any refactor as a new version rather than mutating a version production is already pinned to. I validate that new version in non-prod first and specifically check that `plan` shows no unexpected destroy-and-recreate. And when a resource address genuinely has to change but the underlying resource absolutely must not be recreated, that's exactly what `moved` blocks are for — Terraform 1.1 and later. A `moved` block tells Terraform to update the state's resource address in place instead of destroying and recreating, and that's the correct mechanism for this exact scenario, not some state-file surgery workaround.

</details>

---

### Q: How do you structure Terraform to safely manage resources across multiple AWS regions from a single codebase?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A single `provider "aws" {}` block is bound to one region, full stop — cross-region deployment needs multiple provider configurations, and here's the part that actually causes bugs: modules don't automatically inherit a specific provider unless you explicitly pass one in. That's exactly how you end up debugging "why did this resource deploy to the wrong region" — implicit provider inheritance across a multi-region module call.

So the way I set this up — aliased provider blocks per region:

```hcl
provider "aws" { alias = "primary"; region = "us-east-1" }
provider "aws" { alias = "secondary"; region = "eu-west-1" }
```

And then every module that needs to be region-aware has to declare a `configuration_aliases` block in its `required_providers`, and receive the provider explicitly through the `providers` argument at the call site. Relying on the default implicit provider inheritance is exactly the bug class this explicit wiring is designed to prevent — I make it a review comment every time I see a multi-region module call without an explicit `providers` map.

</details>

---

### Q: What's the actual access-control model that prevents an engineer from running `terraform apply` against production from their laptop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the answer to this is "engineers are told not to," that's a policy, not a control — and the honest truth is any engineer with valid AWS credentials and read/write access to the state backend can run `apply` locally no matter what the internal docs say. The enforcement has to actually live in IAM.

The way I build this — production deployment roles are only assumable by the CI/CD system's own identity. That's an OIDC federated role trusted specifically for our pipeline's exact context, not by any human IAM principal at all. Engineers simply have no credential path capable of assuming that production apply role — the prevention is structural, not procedural, and it doesn't rely on anyone remembering a rule. State bucket write access for production is locked down the same way, scoped to the CI role ARN only, so even in some scenario where an engineer somehow assumed a broader role, the backend itself would reject the write. What I will allow is a narrowly scoped, read-only role for local `plan` against production state, purely for debugging visibility — but apply-capable credentials never exist outside the CI execution context, full stop.

</details>

---

### Q: How do you compose outputs from one Terraform stack (e.g., a VPC) as inputs to another (e.g., an ECS cluster) without merging them into one monolithic state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Splitting infrastructure into layered stacks — network, then compute, then data — is a deliberate blast-radius decision on my part. I don't want a bug in the ECS stack's apply anywhere near VPC state. But that immediately creates a dependency problem — the ECS stack needs subnet IDs and security group IDs that live in a state file it has no direct reference to.

The mechanism for this is `terraform_remote_state` as a data source — it reads the upstream stack's outputs, read-only, with zero ability to modify the source stack's state at all. The consuming stack just references `data.terraform_remote_state.vpc.outputs.subnet_ids` and moves on. What I'm strict about in review is that this creates an explicit, one-directional dependency graph — network feeds compute feeds data, never the reverse. A downstream stack reading from something that itself depends on it is a circular dependency, and that breaks apply ordering outright. And every output consumed cross-stack has to be explicitly declared in the upstream stack's `outputs.tf` — nothing's implicitly exposed, which keeps the interface between stacks deliberate and something you can actually audit.

</details>

---

### Q: A `terraform apply` fails halfway through, applying some resources and erroring on others. What's the actual recovery process?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The reassuring part of this answer, which I always lead with, is that this is recoverable by design, not a corrupted state situation — Terraform applies resources according to its dependency graph, and successfully created resources get written to state immediately as they're created, not batched at the end. So a mid-apply failure just leaves state accurately reflecting whatever *did* get created, with the graph simply incomplete relative to the full desired configuration.

What I do first is actually diagnose the underlying API-level failure — a quota limit, an IAM permission denial, a naming conflict with something that already exists. Re-running `apply` blind, without understanding why it failed, just reproduces the same error against the same remaining graph nodes. Once the root cause is fixed, re-running `apply` is safe — Terraform recomputes the diff against the current, partially-applied state and only touches what's still outstanding, it leaves the already-created resources alone since they already match desired state. If a resource actually got created on the API side but Terraform's write to state failed before it recorded that — which does happen — I reconcile that with `terraform import` rather than letting `apply` try to create a duplicate. And if I genuinely suspect state itself is inconsistent, I'll restore the last known-good version from S3 versioning before I'd ever reach for manual `state rm` or `state mv` surgery — that's very much a last resort, not a first move.

</details>

---

## 8. AWS Infrastructure & Networking

### Q: An EC2 instance is unreachable. What's the deterministic order of checks, and why does that order matter?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Reachability failure can originate at five genuinely independent layers, and if you check them out of order, you end up chasing symptoms downstream of the real fault — debugging an OS-level firewall rule, for instance, when the security group already dropped the packet before it ever reached the instance.

The order I actually go in: security group first, since it's stateful and instance-level — confirm there's an inbound rule for the source CIDR and port, and remember SGs are stateful, so you only need the inbound allow for a client-initiated connection. Second, NACLs — and this is where people get tripped up, because NACLs are stateless, so you need both inbound *and* outbound rules explicitly allowing the traffic. That's the classic "connection gets accepted then just hangs" symptom that a pure security group review completely misses. Third, route tables — confirming the subnet actually has a route to wherever the traffic's coming from, IGW for public, NAT or a gateway for private — a missing or overridden route just blackholes the packet silently, no error anywhere. Fourth, the instance and OS state itself — EC2 status checks, and I'll go in through a bastion or SSM Session Manager to rule the network path out entirely and isolate whether it's actually the instance. And only last, the OS-level firewall — `iptables` or similar — because that's only relevant once you've confirmed the packet actually reached the instance's ENI in the first place.

</details>

---

### Q: Design a multi-VPC network topology — what's the actual architectural decision between VPC Peering and Transit Gateway, and where does each break down at scale?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

VPC Peering is point-to-point and non-transitive — if you need full mesh connectivity across N VPCs, you need N times N-minus-one over two peering connections, and every one of those route tables has to be managed by hand. And the non-transitivity is the real wall — if A is peered to B and B is peered to C, A still can't reach C, full stop. That's the specific thing that breaks a Peering-based design as VPC count grows past a handful.

Transit Gateway solves that by being a managed, transitive hub — every attached VPC can reach every other attached VPC, subject to how the TGW route tables are associated, all through one hub construct. The trade-off is real though — there's a per-GB data processing charge, and now you've got a centralized failure and blast-radius domain that Peering's fully distributed model never had.

In practice, I'll use Peering for a small, stable set of VPCs with well-known bilateral relationships — a shared-services VPC peered individually to two or three others, where manual route table overhead is genuinely tolerable. Once transitive routing is actually required, or VPC count is past four or five, I move to Transit Gateway — but I'm deliberate about using **separate TGW route tables per VPC association**, not one flat shared route table, because that's the actual mechanism that lets you keep prod VPCs from routing to dev VPCs even though both are attached to the same hub. And CIDR planning has to happen up front, before either topology is chosen — overlapping CIDRs make both approaches fundamentally broken without ugly NAT workarounds, and that's not something you can cheaply fix after VPCs are already provisioned.

</details>

---

### Q: RDS performance degrades under load. What's the exact diagnostic sequence to isolate whether it's compute, connections, or query-level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

"RDS is slow" is really three completely different problems wearing the same symptom, and applying the wrong fix — scaling the instance for what's actually a bad query plan — burns money without touching the real bottleneck.

I check CPU and freeable memory first, and if those are genuinely sustained near the limit under a legitimately compute-bound workload, vertical scaling or offloading reads to a replica is the right call. Then connections — if `DatabaseConnections` is climbing toward `max_connections`, that's almost never a database capacity problem in my experience, it's an application-side connection pooling defect, usually no pooling at all, or a pool sized without any regard for how many app instances and replicas are hitting it concurrently. Scaling the instance there just raises the ceiling the leak eventually hits again. And then query-level — I turn on Performance Insights and look at the top wait events and highest-load SQL by `db.load.avg`. A single unindexed query doing full table scans under concurrent load looks identical to "RDS is slow" in the aggregate metrics, but the fix there is an index, not a bigger box.

The order genuinely matters — I check Performance Insights wait events *before* I'd ever scale compute, because scaling to compensate for a missing index is a recurring, expensive pattern I've seen play out, and it just masks the actual defect instead of fixing it.

</details>

---

### Q: What's the actual blast radius when a NAT Gateway fails, and how do you architect against it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A NAT Gateway is AZ-scoped, and the misconfiguration I see most often — usually cost-driven, since each NAT Gateway bills hourly plus per-GB — is private subnet route tables across multiple AZs all pointing at a single NAT Gateway sitting in one AZ. If that AZ has an issue, or the gateway itself fails, you've just lost outbound internet for every private subnet routing through it, cluster-wide, not just the one AZ.

What I actually build is one NAT Gateway per AZ, with each AZ's private route table pointing at the gateway in its **own** AZ — that bounds the failure to a single AZ instead of the whole region. Yes, that's more NAT Gateways and more cost, but it's a deliberate resilience trade-off, not something to skip by default. If cost pressure is real and explicit, a self-managed NAT instance behind an Auto Scaling Group with health-check-triggered replacement is an alternative, but I'm honest about the trade — that comes with genuine operational overhead, patching, scaling limits, that NAT Gateway exists specifically to remove. My default for production is NAT Gateway per AZ unless there's a hard, stated cost constraint forcing a different call.

</details>

---

### Q: Walk through what actually changed to cut deployment cost by 40% — name the specific mechanisms, not the category.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I try not to give the generic "right-sizing and reserved instances" answer, because that doesn't actually demonstrate I did the work — so let me walk through the real levers.

The biggest one was moving static-capacity EC2 workloads onto EKS with Karpenter — that shifted us from provisioning for peak load around the clock to bin-packed, demand-scaled node pools, and that alone cut a lot of idle compute directly. On top of that, I brought Spot Instances into a mixed-instance Auto Scaling Group for fault-tolerant workloads, with proper `PodDisruptionBudget`s and graceful `SIGTERM` handling so we could absorb the two-minute Spot interruption warning without any real availability hit — Spot pricing on those instance families was routinely 60 to 90 percent below On-Demand. I also went through and optimized our Docker images down to multi-stage, distroless final layers, which cut ECR storage cost, but more importantly shortened cold-start and pull time on scale-out events, so we weren't paying for compute that was provisioned but not yet actually serving traffic. Separately, I ran a Terraform-driven cleanup pass — unattached EBS volumes, idle load balancers nobody had decommissioned, unused Elastic IPs — surfaced through an actual inventory audit and removed as explicit, reviewed deletions rather than someone clicking around the console. And last, S3 lifecycle policies transitioning infrequently accessed data to IA and Glacier on an age-based schedule, once we understood the real access pattern falloff for that data.

</details>

---

## 9. ECS Fargate + ALB — Production Failure Modes

### Q: A Fargate service behind an ALB shows intermittent 502s for 2-3 minutes during every deployment. Diagnose the full failure chain and the fix.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This one I've actually debugged in production, and it comes down to three separate timers that aren't coordinated with each other by default. There's how long the application genuinely takes to be ready to serve traffic, there's the ECS service's own health check grace period before it decides to trust or kill a new task, and there's the ALB's health check interval and threshold before it registers a target as healthy. If the grace period is shorter than real startup time, ECS kills and replaces the task before it was ever given a chance to become ready — and from the outside that looks exactly like an application bug, when it's really just a timing misconfiguration.

There's a second half to this too, on the old task's side going away — the target group's `deregistration_delay` controls how long in-flight requests keep getting routed to a draining target before it's fully pulled. If ECS kills the task's process before the ALB has finished draining its last in-flight connections, those connections get reset, and that's the 502 the client actually sees.

So what I actually configure — `healthCheckGracePeriodSeconds` set with real headroom above measured startup time, not a guess, actually timed. A `/health` endpoint that only returns 200 once every startup dependency is genuinely warm — DB pool connected, cache primed — because a shallow health check that returns 200 immediately defeats the entire point of having a grace period. ALB health check interval and thresholds tuned tight enough to catch readiness promptly without flapping under normal latency noise. `deregistration_delay` set to match or exceed the longest expected in-flight request duration — that's a genuine trade-off, too short drops active connections, too long slows deployment cadence, so I don't treat it as a set-and-forget default. And `minimumHealthyPercent` above 100 on the ECS deployment configuration, so new tasks are fully healthy and registered before old ones start terminating, instead of an overlap that's too tight.

</details>

---

### Q: Precisely distinguish `healthCheckGracePeriodSeconds` from ALB target group `deregistration_delay` — what does each actually control, and what happens if they're confused or misconfigured relative to each other?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

These two live in completely different control planes and govern opposite ends of a task's life, which is exactly why people mix them up. `healthCheckGracePeriodSeconds` is an ECS service setting — it suppresses ECS's own reaction to failing health checks on a task that just started. `deregistration_delay` is an ALB target-group setting — it controls how long a target that's already draining keeps getting traffic before it's fully removed. If you tune the wrong one for a given symptom, nothing improves.

The way I tell them apart in practice — if new tasks are cycling repeatedly through provisioning, running, then stopped without ever reaching steady state, that's a grace-period problem, and the fix is either raising `healthCheckGracePeriodSeconds` or fixing a slow-starting dependency, and it has nothing to do with the ALB at all. If instead I'm seeing 502s or connection resets specifically correlated with task *termination* events — which I'd confirm by lining up ECS service events against ALB access log timestamps — that's a `deregistration_delay` problem on the target group, and touching the ECS service setting wouldn't do anything for it. And honestly, you need both tuned correctly for a genuinely clean rolling deployment — the grace period keeps new tasks from being killed too early, the deregistration delay keeps old tasks from being killed while they're still serving. One without the other still leaves a gap.

</details>

---

### Q: A container's real startup time (90s) exceeds the ALB's health check interval (30s). What's the exact ECS/ALB configuration to prevent 502s here, and why is a shorter interval alone not the fix?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The interval just controls polling frequency, it's not the threshold for declaring health — even at a 30-second interval, if the health endpoint keeps returning non-200 until second 90, the target just keeps failing checks until it finally passes, which is entirely correct behavior on the ALB's part. The actual risk isn't there at all — it's that ECS's `healthCheckGracePeriodSeconds` might be shorter than that real 90-second startup, and if it is, ECS kills the task as unhealthy before it ever gets a shot at its first real passing check.

So the fix is setting `healthCheckGracePeriodSeconds` to something like 120 seconds or more — 90 seconds of real startup plus margin — and that's an ECS service-level setting, it has nothing to do with the ALB's polling interval at all. I also make sure the health endpoint is genuinely gating on readiness and not liveness — it has to keep returning non-200 for that full 90 seconds while dependencies aren't warm yet, otherwise you risk the ALB routing live traffic to a task that's technically up but not actually ready to serve correctly. And one thing I always factor into the math — `healthy_threshold` compounds with the interval. If it takes three consecutive passing checks at a 30-second interval to be marked healthy, that's roughly 90 seconds on top of startup time before the target's actually serving, and the grace period needs to account for that compounding, not just the raw startup number.

</details>

---

### Q: Given a 502 in production, how do you determine — from logs alone, without guessing — whether the ALB or the application generated it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A 502 by definition means the ALB is reporting it got an invalid response, or no response at all, from the target — the application itself may never have actually returned a 502. The ALB synthesizes that status when the backend connection fails, times out, or the target gets deregistered mid-request. So just looking at the client-side status code tells you nothing about which side actually failed.

What I actually do — turn on ALB access logs to S3, and go straight to the `target_status_code` field. A dash there means the ALB never got a response from the target at all — connection refused, target already deregistered, target unreachable — that's an ALB or infrastructure-side failure. A real numeric value, say a 500, means the target *did* respond, and the application itself returned that error — that single field disambiguates which side actually failed, no guessing involved. If I'm seeing a lot of dashes clustered tightly around a deployment or a scale-in event, that confirms it's a deregistration-timing race, not application code. And if `target_status_code` shows a real value, that's my cue to go straight to application logs or APM traces for the actual stack trace — at that point it's genuinely a code bug, not an infrastructure timing issue.

</details>

---

## 10. IAM, Secrets & Security Boundaries

### Q: How do you get secrets to a Lambda function at runtime without ever having them touch source control or unencrypted CI logs?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The failure mode I always watch for is a secret sitting as a plaintext value directly in `serverless.yml`, committed to Git — and the thing people forget is that removing it from a future commit doesn't remove it from history, it's permanently in there. CloudFormation outputs derived from those values can also leak into CI build logs if they're not explicitly masked.

So the way I do it — secrets live in SSM Parameter Store as SecureStrings, or Secrets Manager, never in any `.yml` or `.tfvars` file that's tracked by Git. At deploy time, I use the framework's native resolver syntax to pull them in — something like `${ssm:/path/to/secret~true}` in Serverless Framework — and that `~true` suffix is what specifically triggers SecureString decryption, resolving directly into the Lambda's environment config without ever passing through a CI log line. On the IAM side, the Lambda's execution role only gets `GetParameter` or `GetSecretValue` scoped to the exact ARN it needs, never a wildcard — least privilege at the resource level, not just the action level, so if that function ever got compromised, it couldn't just enumerate every other secret in the account. And I turn on CloudTrail data events for Secrets Manager and SSM access specifically, because access control alone doesn't give you the audit trail you'd actually want during a security review or a postmortem.

</details>

---

### Q: How does a CI/CD pipeline authenticate into multiple AWS accounts without static, long-lived credentials sitting in the CI system?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Static access keys sitting as CI secrets are a standing exposure — they don't expire on their own, they're vulnerable to leaking through build logs or a compromised runner, and revoking one takes manual intervention with no automatic time-bound containment.

What I actually set up is OIDC federation between the CI platform and IAM — GitHub Actions or GitLab CI's OIDC provider trusted directly by an IAM role's trust policy. The CI job requests short-lived STS credentials scoped to a specific role, and that trust policy has real conditions tied to specific claims — the repo, the branch, the environment — so there's no long-lived secret sitting anywhere in the system at all. From a centralized deployment account, the pipeline assumes into each target account's deployment role, and each of those target account's trust policies explicitly lists only that specific CI role ARN as a trusted principal — never one broad cross-account role shared across everything, each scoped to exactly what that pipeline stage needs. And CloudTrail logs every one of those `AssumeRole` calls along with the source identity, which gives a full audit chain from the CI job all the way through to the resource-level API calls — something a shared static key could never give you, because you can't attribute an action to a specific pipeline run with a key everyone's using.

</details>

---

### Q: What's the actual difference in engineering intent between SSM Parameter Store and Secrets Manager — when is using Parameter Store for a secret the wrong call?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Both support SecureString encryption at rest, so that's not actually the differentiator people think it is. The real gap is lifecycle management — Secrets Manager gives you native rotation, Lambda-backed rotation functions on a schedule, plus fine-grained resource-based policies for sharing a secret across accounts. Parameter Store has neither of those built in — if you want rotation there, you're building it yourself from scratch.

So the call I make is — Parameter Store for configuration values and secrets that genuinely don't need rotation, third-party API keys with no rotation mechanism on their end, static feature flags, that kind of thing — it's cheaper and simpler for that use case. Secrets Manager specifically for anything that needs rotation, database credentials being the obvious one, especially paired with RDS's native Secrets Manager integration, which handles rotation without the application even needing to coordinate around it. And if the requirement is cross-account secret sharing, Secrets Manager's resource policies make that materially easier than Parameter Store's more limited IAM-only model — honestly, a multi-account architecture that needs shared secret access is a pretty strong signal on its own to go with Secrets Manager from the start.

</details>

---

### Q: How do you structurally guarantee a feature-branch pipeline run can never deploy to production, rather than relying on pipeline YAML conditionals alone?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A branch-name conditional in pipeline YAML is an application-layer control, and it's only as strong as that pipeline definition — which is editable by anyone with merge access to it. A misconfigured or maliciously edited conditional bypasses the whole thing if that's the only enforcement layer you've got.

So the way I actually enforce this is at the IAM trust-policy layer, not just in YAML. The OIDC trust policy for the production deployment role has a condition restricting the token's claim to exactly the production branch or environment — something like the `sub` claim matching `refs/heads/main`, or a GitHub Actions `environment:production` claim. Even if the pipeline YAML conditional gets bypassed or someone misconfigures it, a feature-branch job's OIDC token simply cannot satisfy that trust policy condition — `AssumeRole` itself gets rejected at the IAM layer, full stop. I'll also use the platform's native environment protection — GitHub Environments with required reviewers, GitLab protected environments — requiring manual approval specifically for production, enforced by the CI platform itself rather than pipeline script logic. And I think about it explicitly as defense in depth — the YAML conditional is the fast-fail UX layer, but the IAM trust policy condition is the actual security boundary, the one that holds even when the first layer gets misconfigured.

</details>

---

## 11. Serverless — Lambda Architecture

### Q: What's the actual mechanism behind a Lambda cold start, and which architectural levers reduce it versus which just mask it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A cold start is the latency of standing up a brand new execution environment — downloading the deployment package, initializing the runtime, and running everything outside the handler, module-level imports, SDK client construction, connection pool setup. And the key mechanical fact that actually determines where optimization pays off is that this happens once per execution environment, not once per invocation — that's the detail that changes how you should be writing the code in the first place.

What actually triggers new cold starts is Lambda needing more concurrent execution environments than it currently has warm — sustained, steady traffic mostly reuses what's already warm, but spiky, high-concurrency bursts are what generate the highest proportion of cold starts, because a lot of new environments are all provisioning at once.

The lever I always check first in code review — is SDK client construction and connection setup sitting at module level, outside the handler? If it's mistakenly inside the handler, you're paying that expensive initialization on every single invocation, cold or warm, instead of once per environment amortized across many invocations. For anything with a real latency SLA and predictable traffic, Provisioned Concurrency pre-initializes a fixed number of environments and genuinely eliminates cold start for that reserved capacity — but I'm careful to frame that as a cost-latency trade, not a general fix, because it does nothing for a burst that exceeds whatever you've provisioned. Minimizing the deployment package and dependency tree is a real, mechanical improvement too, since it shortens the download-and-unpack phase. But Provisioned Concurrency specifically masks the symptom only for the capacity it covers — it doesn't reduce the underlying cost for anything beyond that. For genuinely unpredictable, extreme bursts, the actual architectural answer is either over-provisioning, which is a cost decision, or redesigning so the system tolerates tail latency — buffering through SQS instead of a synchronous API Gateway call, for instance.

</details>

---

### Q: Design a Lambda-based async processing pattern using SNS — what failure modes does this introduce that a synchronous call doesn't have, and how do you close them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The thing I always flag up front is that SNS delivery to a Lambda subscriber is at-least-once, not exactly-once. A transient delivery failure or Lambda throttling triggers SNS's retry policy, which means the same message can genuinely invoke the function more than once. If the handler isn't idempotent — say it's doing a plain insert instead of an upsert — you get duplicate side effects, and that's completely invisible in normal operation, it only shows up when a retry storm actually happens.

There's a second gap too — SNS on its own gives you no visibility into messages that keep failing delivery. Without a dead letter queue configured, anything that exhausts the retry policy just gets dropped, with no record left to investigate afterward.

So the way I design around this — the handler has to be idempotent by construction, checking a message ID, either SNS's own `MessageId` or an application-level idempotency key, against a fast-lookup store like a DynamoDB conditional write, before it processes anything. That turns a duplicate delivery into a no-op instead of a duplicate side effect. I always attach a dead letter queue to the SNS subscription, or use Lambda's own `onFailure` destination — so anything that exhausts retries lands somewhere I can see and reprocess, instead of just vanishing. And if the actual requirement is strict ordering or something closer to exactly-once, I wouldn't force that through SNS at all — I'd use SQS FIFO directly as the trigger, or an SNS-to-SQS fan-out pattern where the queue is what actually provides the ordering guarantee SNS alone can't give you.

</details>

---

### Q: Design a single-codebase Serverless Framework deployment pipeline that deploys the same application across dev/stg/prod AWS accounts with clean environment-specific configuration and no secret leakage between environments.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The failure I'm actually designing against here is configuration bleed — a hardcoded staging VPC ID or a dev-only feature flag that accidentally ships to production because the codebase wasn't cleanly parameterized, or worse, a secret resolved for one environment's deploy leaking into another account's Lambda environment because the config resolution path wasn't properly scoped.

So the way I build it — single codebase, stage-based variable resolution in `serverless.yml`, something like pulling VPC ID from a `custom.${self:provider.stage}` block. Non-sensitive config — VPC IDs, subnet and SG IDs, ARNs, domain names, feature flags — lives in a versioned `config/<stage>.yml` file per environment, reviewed in the same PR as any code change. Secrets never touch those config files at all — they're resolved at deploy time from SSM or Secrets Manager, scoped per account, so each account's SSM parameters are only readable by that account's own deployment role. There's structurally no code path that could resolve a secret cross-environment, even by accident.

The pipeline stage is the single thing driving both which account gets assumed via STS and which config file gets loaded — I never let those be two independently settable variables that could drift apart from each other, because that's exactly how you'd get an account-and-config mismatch. And the artifact itself gets built once — zipped or containerized — stored in S3 or ECR, and that exact same artifact gets promoted through dev, staging, and prod, with only environment config re-injected at deploy time. That's what actually guarantees the code tested in staging is the code that reaches production, instead of a separate build per environment that could quietly drift on dependencies.

</details>

---

## 12. Observability — Monitoring, Logging & Auditing

### Q: Design the logging/auditing layer for an AWS account — what does each service actually give you, and what's the gap if you only use one?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

CloudTrail tells you the who, what, and when of API-level control-plane actions — it'll tell you a security group rule changed and exactly which IAM principal changed it, but it has zero visibility into the actual data-plane traffic that resulted from that change. VPC Flow Logs capture that data-plane traffic — source, destination, port, accept or reject — but they have no concept of *why*, no IAM attribution, no configuration context at all. And AWS Config tracks configuration state over time — what did this resource look like yesterday at 3pm — plus continuous compliance evaluation, but it doesn't tell you who made the change, that's CloudTrail's job, or what traffic resulted from it, that's Flow Logs.

If you only lean on one of these, you've got a real blind spot during an incident. Flow Logs alone might show you a rejected connection, but they can't tell you whether that's expected — an intentional restriction — or a regression, because that causal link only exists once you correlate CloudTrail and Config together.

So I run all three, correlated by timestamp and resource ID, never any one of them in isolation. And I centralize them into something actually queryable — CloudTrail Lake, or shipped out to a SIEM — because during a real incident you need to join across all three by timestamp, not go pulling each one individually from separate consoles while everyone's waiting on you. I'll also turn on CloudTrail data events selectively for sensitive resources — S3 object-level access, Lambda invocations — since data events are opt-in, and that's what actually gives you granular "who read this specific object" auditing that management events alone won't.

</details>

---

### Q: A Redis cluster shows frequent evictions. What's the actual diagnostic path to determine whether this is a capacity problem or an application-design problem?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Evictions happen when `maxmemory` is hit and the configured eviction policy starts clearing keys to make room — that's expected mechanical behavior under memory pressure, not a bug by itself. But frequent evictions specifically mean the working set genuinely exceeds available memory, and there are two structurally different reasons that could be true, and they need different fixes.

What I check first is the trend on used memory over time — a steady climb with no plateau is the tell for unbounded key growth, which is almost always an application-side issue, keys getting written without a TTL and just accumulating forever. I'll audit the write paths specifically for TTL coverage — any cache-write path that doesn't set an expiry is a leak candidate, and I'll sample actual TTLs across keys to confirm. If TTL coverage checks out and evictions are still happening under a legitimately large working set, that's genuine undersizing, and the fix is scaling the node type or shard count based on measured peak working-set size with real headroom — not reactive scaling triggered by eviction counts. And I always double-check the eviction policy matches the actual use case — `noeviction` on a pure cache workload is a much worse failure mode than people realize, because once memory's full, writes start failing outright instead of just degrading cache hit rate. `allkeys-lru` or `allkeys-lfu` is almost always the right call for a pure cache.

</details>

---

### Q: What does a production-grade observability stack actually need to cover, beyond "we have CloudWatch"?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

CloudWatch on its own gives you infrastructure-level metrics and raw logs, but it has no native concept of distributed tracing across service boundaries. In a microservices or Lambda-based architecture, a slow end-user request might cross five services, and CloudWatch metrics alone can't tell you which specific hop actually introduced the latency — without tracing, you're manually correlating log lines by request ID across five separate log groups, which is exactly the kind of thing you don't want to be doing during an active incident.

So the stack I actually build — APM with real distributed tracing, New Relic or X-Ray or equivalent, instrumented across every service in the request path, with a single trace ID propagated through headers so I can see the full request waterfall across Lambda, ECS, and RDS calls in one view instead of stitching it together by hand. CloudWatch stays as the infrastructure layer underneath that — resource utilization, error rates, log aggregation — necessary, but not sufficient on its own for request-level debugging in a distributed system. And I make sure alerting is on the actual user-facing symptom, error rate and p99 latency, not just resource metrics — a CPU-only alerting posture completely misses application-level degradation that doesn't show up as resource pressure, like a slow downstream dependency that's not actually maxing anything out. Last thing I'd add — backup and restore is part of observability, not a separate checkbox. RDS backups and EBS snapshots only count as a real recovery capability if restore is actually tested periodically, otherwise it's an unverified assumption, not a control.

</details>

---

### Q: Define an incident response process that survives beyond "we look at the dashboard and fix it" — what's the actual structured loop?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

An ad hoc detect-and-fix loop with no RCA step guarantees the same class of incident comes back around — without a structured postmortem, whatever fix gets applied under pressure addresses the immediate symptom, but almost never the systemic cause, a missing alert threshold, a capacity assumption nobody ever actually validated, a runbook gap.

The loop I actually run — detect, on symptom-level thresholds, error rate and latency, not just resource metrics, because the closer detection is to the real user-facing symptom, the faster time-to-acknowledge gets. Analyze, correlating across the full observability stack — traces, logs, metrics, and recent deploys or config changes pulled from CloudTrail and Config — to actually find root cause, not just the proximate trigger. Fix — apply the minimal safe remediation to restore service first, a rollback, a scale-out, a traffic shift — and I'm deliberate about treating stabilization and the actual root-cause fix as different actions on different timelines, I don't block restoring service on having the full fix ready. RCA, a blameless postmortem within a defined SLA after the incident, documenting root cause, contributing factors, and specifically calling out any monitoring or alerting gap that delayed detection. And prevent recurrence — the RCA has to produce tracked, owned action items, a new alert threshold, an added runbook step, an actual capacity fix, with follow-up verification that it happened. An RCA that doesn't change anything structurally is just a report, it's not actually prevention.

</details>

---

## 13. Linux / Scripting — Systems Operations

### Q: Write the exact command to purge log files older than 30 days and larger than 50MB, and explain the failure mode of getting the `find` predicate order wrong.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The command is:

```bash
find /var/log -type f -size +50M -mtime +30 -delete
```

The part I always call out is that `-type f` isn't optional here, even though it looks like it might not matter — without it, directories that happen to satisfy the size and age predicates match too, and combined with `-delete`, that can remove entire directory trees instead of just files. It's rare, but it's not impossible, especially with tools that rotate or archive logs into dated subdirectories, and it's the kind of mistake that's very hard to walk back once it's run.

What I always do before running anything destructive like this for the first time on a new environment — swap `-delete` for `-print` and actually review the file list before running it for real, especially since I can't assume the log rotation structure on a new box matches what I'm expecting. One more detail worth knowing — `-mtime` is based on modification time, not creation time, since standard Linux filesystem metadata doesn't track creation time by default. For logs that get written once and never touched again after rotation, that's fine. But for anything actively appended to, `-mtime` reflects the last write, not rotation age — and if the actual intent was "files rotated 30-plus days ago," that distinction matters.

</details>

---

### Q: Write a script that counts ERROR occurrences in a log file, and explain why a naive substring match is a correctness risk at production log volume.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A naive check like `"ERROR" in line` gives you false positives on anything that just happens to contain that substring — a line like "user error_count field updated," or a stack trace referencing a class called `ErrorHandler`, both match even though neither is actually an error event. And the dangerous part is at production log volume, this fails silently — the script runs fine, returns a plausible-looking number, and quietly corrupts the metric with nobody noticing.

What I'd actually write:

```python
import re

pattern = re.compile(r'\bERROR\b')
count = 0
with open("app.log", "r") as f:
    for line in f:
        if pattern.search(line):
            count += 1
print(f"Total ERROR messages: {count}")
```

But honestly, my real answer is that the match should be anchored to the log format's actual severity field, not a free-text search at all — if it's structured JSON logging, parse it and check `record["level"] == "ERROR"` directly, and reserve substring matching, at minimum with a word boundary, for genuinely unstructured plaintext logs. And beyond the script itself — this kind of counting belongs in a log aggregation query, CloudWatch Logs Insights or an ELK query, not a one-off script reading a single host's local file. A local script doesn't scale past that one host and gives you no real cross-instance aggregate.

</details>

---

### Q: Automate Nginx installation across a fleet of 10 servers with Ansible — what's the idempotency guarantee the `apt` module gives you that a raw shell command doesn't?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A raw `apt-get install -y nginx` through the shell module runs unconditionally every single time the playbook executes — it's not idempotent by construction. That's harmless for a plain install, but it becomes a real problem for anything non-idempotent, like appending a config line with `>>` on every run and duplicating it indefinitely. The native `apt` module is different — it checks the current package state first, and only actually takes action if the declared state doesn't already match reality. Run it a second time against a host that's already satisfied, and it reports `ok`, not `changed`.

What I'd write:

```yaml
- hosts: web
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
```

I default to native modules over raw shell or command wherever one exists, specifically for that idempotency guarantee — the `changed` versus `ok` distinction is what actually makes a playbook safely re-runnable, and it doubles as a useful drift signal, because a `changed` result on a run I expected to be a no-op is worth investigating on its own. And I scope `update_cache` to the task itself rather than running a separate unconditional `apt update` every time, so I'm not paying for a full cache refresh on every run when it's really only needed occasionally.

</details>

---

### Q: What's the actual purpose of separating `roles/`, `group_vars/`, and `host_vars/` in an Ansible project, beyond directory tidiness?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This structure exists to enforce variable precedence and real reuse boundaries that a flat playbook just can't express cleanly. Without it, host- or group-specific overrides end up scattered through task-level conditionals, and at that point there's no way to figure out a given host's effective configuration without reading through every single task's logic.

The way I break it down — `roles/` hold reusable, self-contained units of automation, tasks, handlers, templates, defaults — an `nginx` role should be reusable unmodified across every project that needs it, with anything environment-specific injected purely through variables, never hardcoded into the role itself. `group_vars` apply to every host in an inventory group, and that's the right place for anything environment-wide, like every host in the `web` group getting the same worker count. `host_vars` override at the individual host level, and this is the part people miss — Ansible's precedence puts `host_vars` **above** `group_vars`, so a specific host's override always wins without touching the shared group file at all, which keeps exceptions isolated and easy to audit instead of polluting the group-level config for everyone. And this separation is exactly what makes the same role safely deployable across dev, staging, and prod inventories — the role logic itself never changes, only the `group_vars` and `host_vars` values selected by whichever inventory file you're targeting.

</details>

---

## 14. System Design — End-to-End Architecture

### Q: Describe a production multi-account AWS architecture end-to-end — the actual isolation boundaries, the deployment path, and where each control lives.

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

When I design these systems, I start from the fact that multi-account architecture is fundamentally a blast-radius containment strategy — every layer of it, state isolation, IAM trust boundaries, network segmentation, exists so a failure or a compromise in one environment can't propagate into another by construction, not because everyone's being careful.

At the account level, I keep separate AWS accounts per environment — dev, dev2, staging, UAT, production — because that's the outermost boundary, and IAM, service quotas, and billing are all natively scoped per account, which gives you a much harder isolation guarantee than tagging everything inside a single account ever could. On compute, I split by workload shape rather than forcing one model everywhere — event-driven backend services on Lambda through the Serverless Framework, and containerized, longer-running services on ECS Fargate behind an ALB. For data, RDS PostgreSQL for relational and transactional workloads, DynamoDB where the access pattern is genuinely high-throughput key-value — that's a decision made per access pattern, not a default-to-relational habit. S3 for artifacts and static content, EFS reserved specifically for the cases that genuinely need a shared POSIX filesystem across compute.

For decoupling, SQS for durable point-to-point queuing and EventBridge for event-driven fan-out — the whole point is that a downstream outage doesn't cascade synchronously back to the producer. Infrastructure is modular Terraform — reusable component modules, thin per-environment root modules, isolated remote state per account, and CI-driven `AssumeRole` deployment with no local apply path into production at all, the way I described earlier.

Walking the actual deployment flow — a commit triggers the pipeline, it builds an immutable artifact, a Lambda zip or a Docker image, stores it in S3 or ECR, assumes into the target account, applies infrastructure through Terraform and deploys the application through Serverless or ECS blue-green via CodeDeploy. Branch protection maps feature branches to dev only, main and release branches to staging, and production sits behind manual approval enforced at the IAM trust-policy level, not just a pipeline conditional — that's the actual security boundary. Config that isn't sensitive lives versioned in the repo per environment; secrets get resolved from SSM or Secrets Manager at deploy time and never touch source; IAM is scoped tightly per Lambda or task execution role rather than shared broad roles everyone reuses. On observability, APM gives me distributed tracing across the Lambda and ECS request path, CloudWatch covers infrastructure metrics and logs, and alerting is on error rate and p99 latency routed to Slack or PagerDuty, not just resource thresholds. And security ties back to that same account-level isolation as the primary boundary, STS `AssumeRole` with short-lived credentials so there are zero long-term keys anywhere, KMS encryption at rest across RDS, S3, and SSM, and security groups scoped to exactly the ports and sources that are actually required, never broad CIDR ranges out of convenience.

</details>

---

<div align="center">

⭐ *If this helped you prep, consider starring the repo.*

</div>
