# Senior DevOps Interview Questions: Kubernetes / EKS

### Q: How do Kubernetes Deployments execute zero-downtime rolling updates and rollbacks?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I normally use a RollingUpdate strategy for application deployments. I control the rollout using maxSurge and maxUnavailable.

For example, if I have 5 replicas, Kubernetes can start new pods while the old pods are still running. I usually keep maxUnavailable: 0 for applications where I don't want to lose capacity during deployment.

The new pod starts first, then Kubernetes checks its readiness probe. Once it becomes Ready and is added to the Service endpoints, Kubernetes starts terminating an old pod.

So the flow is:

New pod → readiness check passes → receives traffic → old pod terminates → repeat.

I also monitor the rollout using kubectl rollout status.

If the new version has an issue, I can stop the rollout and rollback to the previous ReplicaSet using:

kubectl rollout undo deployment/<deployment-name>

Before calling it zero-downtime, I also make sure the application has enough replicas, proper readiness probes, and the application can handle multiple versions running at the same time.

</details>

---

### Q: How do Pod Disruption Budgets (PDB) handle voluntary disruptions during cluster maintenance?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A PDB protects against planned actions, like taking a server down for an upgrade. It does not protect against a random crash.

Without a PDB, taking a server down could accidentally kill every copy of an app at once, if they're all sitting on that server.

I set a rule like "at least 2 copies must stay running." Kubernetes will then block the server from being taken down until enough healthy copies exist somewhere else.

One thing to watch for — if an app only has one copy, this rule can block the update forever, since there's no backup copy to rely on.

</details>

---

### Q: How do StatefulSets differ from Deployments when managing stateful workloads requiring persistent storage?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Deployment is for apps where any copy can replace any other copy, no problem. That's fine for something stateless, like a web server with no memory of past requests.

For something like a database, I use a StatefulSet instead, because it needs its own identity and its own storage.

Each copy gets a fixed name and its own dedicated storage that follows it around, even if it moves to a different server.

Also — deleting a StatefulSet does not delete its storage. That storage stays behind on purpose, so data isn't lost by accident.

</details>

---

### Q: How do you enforce resource isolation and multi-tenancy using Namespaces, ResourceQuotas, and RBAC?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I give each team its own namespace, like a separate folder.

Then I set a limit on that namespace — how much CPU and memory it's allowed to use in total. This stops one team from using up resources that other teams need.

I also set default limits for any app that doesn't set its own.

For access control, I give each team permission only inside their own namespace, never across the whole cluster. That way, one team genuinely can't see or touch another team's stuff.

</details>

---

### Q: How do Custom Controllers, CRDs, and the Sidecar pattern extend Kubernetes core functionality?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A CRD lets you add a brand new type of object to Kubernetes, one that isn't built in. On its own, that's just a definition, it doesn't do anything by itself.

A Custom Controller is what actually makes it work — it watches for those objects and takes action to keep things matching what's expected.

**New CRD object gets created → Custom Controller notices it → controller takes real action → cluster state matches what was expected.**

The Sidecar pattern is different. It's a second, helper container that runs next to your app, inside the same pod. Istio uses this to automatically add a network helper to every app, without changing the app's own code at all.

</details>

---

### Q: What's the difference between a Liveness Probe and a Readiness Probe, and what happens if you mix them up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Readiness Probe tells Kubernetes whether a pod is ready to actually receive traffic right now.

If it fails, the pod is just pulled out of the traffic list, but Kubernetes leaves it running, since it might recover on its own.

A Liveness Probe is different — if it fails, Kubernetes assumes the app is stuck for good, and it kills and restarts the container.

The mistake I see a lot is using the same check for both. If a slow but recovering app fails a Liveness Probe, Kubernetes keeps restarting it over and over, which just makes a slow problem into a much bigger outage.

</details>

---

### Q: How does the Horizontal Pod Autoscaler work, and what do you need in place before it will actually work correctly?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The Horizontal Pod Autoscaler watches a metric, usually CPU usage, and adds or removes pod copies to keep that metric near a target you set.

But it only works if every pod already has a CPU or memory request set in its config. Without that, the autoscaler has nothing real to measure against, and it just won't work properly.

I also always set a minimum and a maximum number of copies, so it can't scale down to zero by accident during a quiet period, and it can't scale up forever if something goes wrong.

</details>

---

### Q: How do you plan and safely execute a Kubernetes cluster version upgrade in production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never upgrade production first. I always test the new version on a non-production cluster running the same apps, and check what's changed or removed in that version.

For the real upgrade, I do the control plane first, since it can run a slightly newer version than the worker nodes for a short time.

Then I upgrade the worker nodes in small groups, moving pods off each one safely before touching it, instead of upgrading everything at once.

**Test new version in non-prod → upgrade control plane → upgrade worker nodes in small batches → move pods off each node first → verify → move to the next batch.**

If something breaks partway through, only part of the cluster is affected, not all of it at once.

</details>

---

### Q: Your team wants to migrate a legacy application to EKS. The application isn't containerized and has stateful components. How would you handle this migration?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I wouldn't try to move this in one shot. Containerizing and moving to Kubernetes at the same time as dealing with the stateful parts is exactly how these migrations blow their timeline and end up with a broken rollback plan.

First step is just containerizing the app as-is, no architecture changes yet — a **Dockerfile** that replicates its current runtime environment, and I run that container in the existing on-prem or EC2 setup first, to confirm it behaves identically to the non-containerized version before Kubernetes even enters the picture.

For the stateful components — usually a database or a local file store the app was writing to directly — I don't try to run those inside EKS as StatefulSets unless there's a real reason to. I'd move the database to **RDS/Aurora** and point the containerized app at it instead, and for file storage, either **S3** if the access pattern allows it, or an **EFS**-backed PersistentVolume if the app genuinely needs a shared POSIX filesystem. Pulling state out of the app onto managed services first makes the actual EKS deployment stateless, which is a much simpler and safer thing to run.

Once the app is containerized and its state is externalized, I deploy it to EKS behind a **Deployment** with proper readiness probes, and I run it side-by-side with the legacy system for a while, routing a small percentage of traffic over first rather than a hard cutover.

**Simple flow:** Containerize app as-is → validate outside Kubernetes → externalize state to RDS/S3/EFS → deploy stateless container to EKS → shift traffic gradually → decommission legacy.

**Key point:** I separate "containerize" from "make stateless" from "move to Kubernetes" — three distinct steps, not one big migration, so each one can be validated and rolled back independently.

</details>

---

### Q: You're running multiple EKS clusters for different environments (dev, staging, production) and need a way to manage them effectively. How would you set this up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I keep environments on fully separate clusters, not namespaces on one shared cluster — for production specifically, I don't want a dev team's mistake, like a runaway resource request or a bad CRD, to have any physical path to affecting production at all.

For managing the actual cluster infrastructure consistently across all three, I provision them with **Terraform**, using the same module with environment-specific variables — node group sizes, instance types, and add-on versions differ, but the underlying structure is identical, so I'm not maintaining three different hand-written cluster configs that slowly drift apart.

For what's actually deployed inside each cluster, I use a **GitOps** approach with **ArgoCD** — each cluster points at a different branch or directory in the same Git repo, so promoting a change from dev to staging to production is a Git operation, a PR merging one environment's manifests forward, not someone running `kubectl apply` by hand against three different clusters and hoping they typed the right context.

Access is scoped per cluster through IAM — engineers get broad access to dev, more restricted access to staging, and production access is limited to a small group plus the CI/CD pipeline's own role, enforced via **EKS access entries** or `aws-auth`, not shared kubeconfig files passed around.

**Key point:** Same Terraform module for consistent infrastructure, GitOps for consistent and auditable deployments, and IAM-based access that gets tighter the closer you get to production.

</details>

---

### Q: Your application on EKS is experiencing a high number of failed requests and errors. What steps would you take to troubleshoot this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't start reading application code — I check the platform layer first, since that's faster to rule in or out and often is the actual cause.

First, `kubectl get pods` to see if pods are actually healthy, or cycling through restarts. If pods look fine, I check `kubectl get endpoints` for the Service, to confirm it's actually got healthy pods behind it — an empty or partial endpoint list means some requests are being routed to nothing.

Then I check resource pressure — `kubectl top pods` against the configured requests/limits. I've had "random" failed requests turn out to be CPU throttling under load, where the app was technically running but too slow to respond within the client's timeout, which shows up as failures on the client side with nothing obviously wrong on the pod side.

If pods and resources both look fine, I check the **ALB/Ingress** layer next — target group health in the AWS console, and whether the failures are actually 5xx from the app or something like 503s from the load balancer having no healthy targets during a rollout.

**Simple flow:** Check pod health → check Service endpoints have healthy targets → check CPU/memory against limits → check ALB/Ingress target health → check app logs for the specific failing requests → correlate failure timestamps against any recent deploy.

**Key point:** I rule out platform-level causes — pod health, resource limits, load balancer routing — before assuming it's an application bug, since in my experience it's the platform layer more often than not.

</details>

---

### Q: Your EKS application requires low-latency access to an RDS database in a different VPC. How would you set this up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I wouldn't route this over the public internet or through a NAT Gateway — that adds latency and cost for something that should stay entirely inside AWS's network.

**VPC Peering** between the EKS cluster's VPC and the RDS VPC is the straightforward option if it's just these two VPCs involved — it's a direct, private connection with no additional hop, and once the peering connection and route tables are set up, traffic between the pods and RDS stays on AWS's internal network the whole way.

If there are more than a couple of VPCs involved, or this is likely to grow — more clusters, more shared services — I'd use a **Transit Gateway** instead, since Peering doesn't scale cleanly past a handful of VPCs and Transit Gateway gives a proper hub instead of a growing mesh of point-to-point connections.

Either way, the security groups matter as much as the network path — the RDS security group needs an inbound rule allowing traffic from the EKS node/pod security group on the database port, referencing the security group directly rather than a CIDR range, since node IPs can change as the cluster scales.

**Key point:** Peering or Transit Gateway for the private network path — VPC Peering for a simple two-VPC case, Transit Gateway once it's more than that — plus security group rules that reference the actual security group, not an IP range that'll drift.

</details>

---

### Q: You want to scale your application on EKS automatically based on CPU and memory usage. What would you do?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For scaling the number of pod copies, that's the **Horizontal Pod Autoscaler**, watching CPU and memory against a target I set — but it only works correctly if every pod already has `resources.requests` set in its spec, since the HPA calculates its percentage against the request value, not against the node's total capacity.

I always set explicit `minReplicas` and `maxReplicas` too, so it can't scale down to zero during a quiet period and can't scale up without bound if something's actually wrong, like a bug causing runaway CPU usage — without a max, the HPA would just keep adding pods trying to bring CPU back to target, potentially exhausting the cluster's node capacity in the process.

For metrics beyond basic CPU/memory — like scaling based on queue depth or request rate — I'd use the **Kubernetes Metrics Server** for the basics, or **KEDA** if I need to scale off something like an SQS queue length, which the standard HPA can't do on its own.

On top of pod-level scaling, I also need the actual node capacity to grow to fit those new pods — that's **Cluster Autoscaler** or **Karpenter** watching for pods stuck in `Pending` because there's no room, and provisioning new nodes to fit them.

**Simple flow:** Set resource requests on every pod → HPA scales pod count based on CPU/memory against those requests → Karpenter/Cluster Autoscaler adds nodes when pods can't be scheduled → min/max bounds on both layers.

**Key point:** HPA scales pods, Karpenter/Cluster Autoscaler scales nodes — you need both working together, not just one, or pods end up stuck `Pending` with nowhere to actually run.

</details>

---

### Q: You need to perform a rolling update on your EKS application, but you want to minimize downtime. How would you configure this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The core settings are `maxSurge` and `maxUnavailable` on the Deployment's `RollingUpdate` strategy. I set `maxUnavailable: 0`, so Kubernetes never takes an old pod down until a new one is confirmed healthy — capacity never actually drops below 100% during the rollout, only surges above it temporarily.

That alone isn't enough without a real readiness probe — one that actually checks the app can serve traffic, not just that the process started, since Kubernetes only adds a pod to the Service's endpoints once its readiness probe passes. A shallow probe defeats the whole point of `maxUnavailable: 0`, since it'll mark a pod ready before it's actually able to handle requests.

I also set `terminationGracePeriodSeconds` with real headroom, and add a short `preStop` hook if the app needs time to finish in-flight requests before shutting down — otherwise a pod can get killed while the load balancer hasn't finished deregistering it yet, which shows up as brief errors during every deploy even with the rollout strategy configured correctly.

**Simple flow:** maxUnavailable: 0, maxSurge set for capacity headroom → new pod starts → real readiness probe passes → added to Service → old pod gets a preStop grace period before shutdown → repeat per pod.

**Key point:** The rollout strategy settings only work as well as the readiness probe backing them — I check the probe is meaningful before trusting `maxUnavailable: 0` to actually deliver zero downtime.

</details>

---

### Q: You need to restrict access to an EKS cluster to a specific IP range. How would you set this up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This depends on whether I mean access to the Kubernetes API server itself, or access to an application running inside the cluster — they're two different controls.

For the API server, EKS lets you configure the **cluster endpoint access** — I'd set it to private, or if public access is still needed for some tooling, restrict the public endpoint's allowed CIDR blocks to the specific IP ranges that should be able to reach it, like the office VPN's egress IP or a bastion host, rather than leaving it open to `0.0.0.0/0`, which is the default and something I always check and tighten on a new cluster.

For an application exposed through an Ingress or a Service of type LoadBalancer, that's controlled at the **ALB/NLB security group** level, or through an annotation on the Ingress that restricts the load balancer's allowed source CIDR, rather than trying to do IP filtering inside the app itself.

If the requirement is more nuanced than a static IP range — like needing to combine IP restriction with authentication — I'd look at putting something like **AWS WAF** in front of the ALB, which lets me combine IP allow-listing with rate limiting and other rules in one place, rather than stacking multiple separate mechanisms.

**Key point:** API server access is an EKS cluster-level setting; application access is a load balancer/security group setting — restricting the wrong one leaves the other side still wide open.

</details>

---

### Q: Your team needs to track detailed usage and costs for each namespace in the EKS cluster. What's your approach?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Native AWS cost tools stop at the EC2/EKS cluster level — they don't know how to split cost by Kubernetes namespace on their own, since from AWS's perspective it's all just compute on shared nodes. So I bring in something purpose-built for this, like **Kubecost** or **AWS's own Split Cost Allocation Data** feature for EKS, which specifically breaks down cluster spend by namespace, label, or team.

The accuracy of that breakdown depends entirely on every workload having proper `resources.requests` set, since cost gets allocated proportionally based on requested (or actual) resource usage per namespace — a namespace with no requests set throws off the whole allocation, so I enforce that through admission policy, not just convention.

I also make sure every namespace carries a consistent labeling scheme — team, environment, cost-center — so the breakdown can actually be grouped meaningfully, not just shown as a flat list of thirty namespace names nobody outside the platform team recognizes.

Once that's flowing, I get it into the same weekly report I'd already be sending for regular AWS cost allocation, broken down by namespace/team alongside the rest of the account's spend, rather than as a separate tool only the platform team ever opens.

**Key point:** Namespace-level cost visibility needs a purpose-built tool like Kubecost, and it's only as accurate as the resource requests and labels every workload is actually setting.

</details>

---

### Q: You notice that a Pod in your EKS cluster is in a "CrashLoopBackOff" state. What steps would you take to diagnose and fix the issue?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`CrashLoopBackOff` just means the pod keeps failing and Kubernetes keeps retrying with a longer backoff each time — it's the kubelet's retry wrapper, not the actual error, so I don't stop there.

`kubectl describe pod` first, for the exit code, the reason, and recent events. Exit code `137` means it was `SIGKILL`ed, almost always an OOM kill — I'd check that against `kubectl top pod` history or the metrics stack, comparing real usage to the configured `limits.memory`. Exit code `1` or another app-specific code usually means the application itself failed on startup, which points me at the app, not the platform.

Then `kubectl logs <pod> --previous`, since the current container instance already restarted with a clean slate — the previous instance's logs are what actually show why it died, and people often check `logs` without `--previous` and just see an empty or freshly-started log with nothing useful in it.

If the logs are genuinely empty or unhelpful, I'll temporarily override the container's command to something like `sleep 3600` in a throwaway copy of the manifest, exec into it while it's held open, and check environment variables, config, and connectivity by hand — this is my last resort, not the first thing I reach for.

**Simple flow:** describe pod for exit code/events → 137 means OOM, check real memory usage vs limit → other exit code, check `logs --previous` for the app's actual error → still unclear, override command to keep it alive and exec in to debug interactively.

**Key point:** The exit code from `describe pod` tells me which direction to actually investigate — I don't guess between "it's a memory problem" and "it's an app bug," the exit code answers that directly.

</details>

---

### Q: Your EKS cluster is experiencing CPU throttling issues on certain workloads. How would you resolve this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

CPU throttling specifically means a pod is hitting its configured `limits.cpu` and the kernel's CFS scheduler is holding it back — and the tricky part is this can happen even when `kubectl top` shows average CPU usage well under the limit, since the scheduler enforces the limit over very short windows, like every 100 milliseconds, not over a full minute.

I check `container_cpu_cfs_throttled_periods_total` against `container_cpu_cfs_periods_total` in Prometheus if it's set up, since that's the metric that actually shows per-period throttling — `top`-style averages hide this completely, so a workload can look fine on a dashboard while genuinely getting throttled in short bursts that hurt latency.

The fix is either raising `limits.cpu` with real headroom above observed burst usage rather than average usage, or for genuinely latency-sensitive workloads, removing the CPU limit entirely and relying on `requests.cpu` for scheduling, accepting the noisy-neighbor trade-off, or pairing it with the static CPU manager policy for guaranteed, pinned cores on the node.

I also check if this is actually a capacity problem in disguise — if every pod on a node is set to burst simultaneously, like at the top of every minute for a scheduled batch job, the node itself doesn't have enough real CPU to give everyone their burst at once, no matter how the limits are configured, and that needs either spreading the workloads out or adding more node capacity.

**Key point:** Average CPU from `kubectl top` hides real throttling — the CFS throttled-periods metric is what actually shows it, and the fix is sizing limits against burst usage, not average usage.

</details>

---
