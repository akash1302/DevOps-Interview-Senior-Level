# Senior DevOps Interview Questions: Kubernetes / EKS

### Q: How do Kubernetes Deployments execute zero-downtime rolling updates and rollbacks?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I normally use a RollingUpdate strategy, controlled with `maxSurge` and `maxUnavailable`. For example, with 5 replicas, Kubernetes can start new pods while the old ones are still running. I usually keep `maxUnavailable: 0` when I don't want to lose any capacity during a deploy.

The new pod starts, Kubernetes checks its readiness probe, and only once it's actually ready does an old pod get terminated. I watch the rollout with `kubectl rollout status`, and if something's wrong, `kubectl rollout undo deployment/<name>` rolls straight back to the previous version.

Before I'd call anything zero-downtime, I make sure there are enough replicas, the readiness probes are meaningful, and the app can genuinely handle two versions running side by side for a moment.

</details>

---

### Q: How do Pod Disruption Budgets (PDB) handle voluntary disruptions during cluster maintenance?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A PDB protects against planned actions, like taking a node down for maintenance. It doesn't protect against a random crash.

Without one, draining a node could accidentally kill every copy of an app at once if they're all sitting there. What I normally do is set a rule like "at least 2 copies must stay running," and Kubernetes blocks the drain until enough healthy copies exist elsewhere.

One thing to watch for — if an app only has one replica, this rule can block a drain forever, since there's no backup copy to fall back on.

</details>

---

### Q: How do StatefulSets differ from Deployments when managing stateful workloads requiring persistent storage?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Deployment works fine when any pod can replace any other pod, like a stateless web server.

For something like a database, I use a StatefulSet instead, because it needs a stable identity and its own storage. Each pod gets a fixed name and its own dedicated storage that follows it around, even if it gets rescheduled to a different node.

Also worth knowing — deleting a StatefulSet doesn't delete its storage. That's on purpose, so data doesn't disappear by accident.

</details>

---

### Q: How do you enforce resource isolation and multi-tenancy using Namespaces, ResourceQuotas, and RBAC?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I give each team its own namespace. Then I set a resource quota on it — how much CPU and memory it's allowed to use in total — so one team can't eat up resources meant for another.

I also set default limits for anything that doesn't specify its own. For access, I give each team permission only inside their own namespace, never across the whole cluster, so one team genuinely can't see or touch another team's workloads.

</details>

---

### Q: How do Custom Controllers, CRDs, and the Sidecar pattern extend Kubernetes core functionality?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A CRD lets you register a brand new type of object with Kubernetes. On its own, that's just a definition — it doesn't do anything.

A Custom Controller is what actually makes it useful — it watches for those objects and takes real action to keep the cluster matching what's expected.

The Sidecar pattern is different — it's a helper container running next to your app, inside the same pod. Istio uses this to add a network proxy to every app automatically, without touching the app's own code.

</details>

---

### Q: What's the difference between a Liveness Probe and a Readiness Probe, and what happens if you mix them up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Readiness Probe tells Kubernetes whether a pod is ready to receive traffic right now. If it fails, the pod is just pulled out of rotation, but left running, since it might recover.

A Liveness Probe is different — if it fails, Kubernetes assumes the app is stuck for good and restarts the container.

The mistake I see is using the same check for both. If a slow but recovering app fails the liveness check, Kubernetes keeps restarting it over and over, turning a slow problem into a much bigger outage.

</details>

---

### Q: How does the Horizontal Pod Autoscaler work, and what do you need in place before it will actually work correctly?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The HPA watches a metric, usually CPU, and adds or removes pods to keep it near a target. It only works correctly if every pod already has CPU and memory requests set — without that, it has nothing real to measure against.

I also always set a minimum and maximum replica count, so it can't scale down to zero during a quiet period, and can't scale up forever if something's actually wrong with the app.

</details>

---

### Q: How do you plan and safely execute a Kubernetes cluster version upgrade in production?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I check the current EKS version, node groups, workloads and all important add-ons. Then I check the target Kubernetes version for deprecated or removed APIs and compatibility issues.

I upgrade the non-prod cluster first and test the actual applications there. I check deployments, pods, networking, ingress, storage, autoscaling and application logs.

Once non-prod is stable, I prepare production with backups, monitoring and a recovery plan. Then I upgrade the production control plane, add-ons and node groups in a controlled way during a low-traffic window.

After the upgrade, I closely monitor the applications, nodes, pods, errors and latency. I only close the change after confirming the applications are stable

</details>

---

### Q: Your team wants to migrate a legacy application to EKS. The application isn't containerized and has stateful components. How would you handle this migration?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I wouldn't try to move this in one big step. First, I containerize the app as it is today, no architecture changes yet, and I run that container outside Kubernetes first to confirm it behaves the same way.

For the stateful parts, like a database or local files, I move those out to managed services first — the database to RDS, files to S3 or EFS depending on the access pattern. That makes the actual EKS deployment stateless, which is a much simpler and safer thing to run.

Once it's containerized and stateless, I deploy it to EKS and run it side-by-side with the legacy system for a while, shifting a small percentage of traffic over first, rather than a hard cutover.

</details>

---

### Q: You're running multiple EKS clusters for different environments (dev, staging, production) and need a way to manage them effectively. How would you set this up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I keep environments on fully separate clusters, not namespaces on one shared cluster, so a mistake in dev has no physical path to affecting production.

For the infrastructure itself, I provision all three with the same Terraform module, just different variables, so I'm not maintaining three hand-written configs that slowly drift apart.

For what gets deployed inside each cluster, I use a GitOps approach — each cluster tracks a different branch or folder in the same repo, so promoting a change is a Git operation, not someone running `kubectl apply` by hand against the wrong context. Access also gets tighter the closer you get to production — broad access for dev, restricted for staging, and only a small group plus the pipeline itself for production.

</details>

---

### Q: Your application on EKS is experiencing a high number of failed requests and errors. What steps would you take to troubleshoot this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check the platform layer first before assuming it's an app bug, since that's usually faster to rule out.

First `kubectl get pods` to see if they're healthy or restarting. Then `kubectl get endpoints` for the Service, to confirm it actually has healthy pods behind it. I also check `kubectl top pods` against the configured limits — I've had "random" failures turn out to be CPU throttling, where the app was technically up but too slow to respond in time.

If pods and resources look fine, I check the load balancer's target health next, since a rollout can briefly leave it with no healthy targets.

</details>

---

### Q: Your EKS application requires low-latency access to an RDS database in a different VPC. How would you set this up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I wouldn't route this over the public internet or through a NAT Gateway — that adds cost and latency for something that should stay inside AWS's own network.

For just two VPCs, VPC Peering is the simple option — a direct, private connection. If there are more VPCs involved, or it's likely to grow, I'd use a Transit Gateway instead, since Peering gets hard to manage past a handful of connections.

Either way, the security groups matter just as much — the RDS security group needs to allow traffic from the EKS node security group specifically, not just an IP range, since node IPs change as the cluster scales.

</details>

---

### Q: You want to scale your application on EKS automatically based on CPU and memory usage. What would you do?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

That's the Horizontal Pod Autoscaler, scaling pod count based on CPU or memory against a target I set. It only works properly if every pod already has resource requests set, since the HPA calculates its percentage against that.

I always set a minimum and maximum replica count too, so it doesn't scale to zero in a quiet period or scale up without limit if something's genuinely wrong.

Scaling pods alone isn't enough though — I also need the node capacity to grow to fit them, which is what Cluster Autoscaler or Karpenter handles, adding nodes when pods are stuck waiting for room to run.

</details>

---

### Q: You need to perform a rolling update on your EKS application, but you want to minimize downtime. How would you configure this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The main settings are `maxSurge` and `maxUnavailable`. I set `maxUnavailable: 0`, so Kubernetes never takes an old pod down until a new one is confirmed healthy.

That only works well if the readiness probe is actually meaningful — checking that the app can really serve traffic, not just that the process started. A shallow probe defeats the whole point, since it'll mark a pod ready before it can actually handle requests.

I also give the pod a short grace period before shutdown, so it doesn't get killed while the load balancer is still sending it traffic during the switch.

</details>

---

### Q: You need to restrict access to an EKS cluster to a specific IP range. How would you set this up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This depends on whether I mean the Kubernetes API server itself, or an application running inside the cluster — those are two different settings.

For the API server, EKS lets you restrict the allowed IP ranges on the cluster endpoint, or set it to private entirely, rather than leaving it open to everyone, which is worth checking on any new cluster.

For an application exposed through a load balancer, that's controlled at the load balancer's own security group, not inside the app. If I need something more than a static IP list, like combining it with rate limiting, I'd put a web application firewall in front of it instead.

</details>

---

### Q: Your team needs to track detailed usage and costs for each namespace in the EKS cluster. What's your approach?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Regular AWS cost tools stop at the EC2 or cluster level — they don't know how to split cost by namespace on their own. So I'd bring in a tool built for this, like Kubecost, which breaks down spend by namespace, label, or team.

That breakdown is only as accurate as the resource requests every workload sets, so I make sure every pod actually sets CPU and memory requests, and every namespace has consistent labels for team and environment.

Once that's working, I get it into the same regular cost report the rest of the account already gets, instead of it being a separate tool only the platform team looks at.

</details>

---

### Q: You notice that a Pod in your EKS cluster is in a "CrashLoopBackOff" state. What steps would you take to diagnose and fix the issue?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`CrashLoopBackOff` just means the pod keeps failing and Kubernetes keeps retrying with a longer wait each time. It doesn't tell you why on its own.

First `kubectl describe pod` for the exit code and recent events. Exit code 137 almost always means it ran out of memory, so I'd check real usage against the configured limit. Any other exit code usually points to the app itself, so I check `kubectl logs --previous`, since the current instance already restarted with a clean slate.

If the logs still don't explain it, I'll temporarily change the startup command to just keep the container alive, so I can get inside and check things by hand.

</details>

---

### Q: Your EKS cluster is experiencing CPU throttling issues on certain workloads. How would you resolve this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

CPU throttling means a pod is hitting its CPU limit, and the tricky part is this can happen even when average usage looks fine on a dashboard, since the limit gets enforced over very short windows, not over a full minute.

If Prometheus is set up, I check the throttled-periods metric specifically, since that's what actually shows this — average CPU hides it completely.

The fix is usually raising the CPU limit with real headroom above burst usage, not average usage, or for something genuinely latency-sensitive, removing the limit and relying on the request value for scheduling instead.

</details>

---
