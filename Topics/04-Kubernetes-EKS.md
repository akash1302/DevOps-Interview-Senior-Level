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
