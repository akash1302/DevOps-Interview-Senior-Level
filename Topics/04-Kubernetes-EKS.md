# Senior DevOps Interview Questions: Kubernetes / EKS

### Q: How do Kubernetes Deployments execute zero-downtime rolling updates and rollbacks?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A rolling update works through two simple settings — `maxSurge`, which controls how many extra pods can be added on top of the normal count, and `maxUnavailable`, which controls how many pods can be offline at once. When I push a new version, Kubernetes brings up new pods, waits for each one to actually pass its readiness check, and only then starts shutting down the old ones. If I set `maxUnavailable: 0`, the app never drops below full capacity during the whole update. If something breaks mid-rollout, `kubectl rollout undo` switches traffic straight back to the last working version almost instantly.

</details>

---

### Q: How do Pod Disruption Budgets (PDB) handle voluntary disruptions during cluster maintenance?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A PDB protects against planned disruptions, like draining a node for an upgrade — not against a random crash, which it can't do anything about. Without one, draining a node could accidentally take down every copy of an app at once if they're all sitting there. I set something like `minAvailable: 2`, and Kubernetes will actually block the drain until enough healthy pods exist somewhere else to keep that minimum met. One thing to watch for — a single-replica app with `minAvailable: 1` will block a drain forever, since there's no second copy to fall back on.

</details>

---

### Q: How do StatefulSets differ from Deployments when managing stateful workloads requiring persistent storage?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Deployments are for apps where any pod can replace any other pod, no problem — that's fine for something stateless. For something like a database, I use a StatefulSet instead, because it needs a stable identity and its own storage. Each pod gets a fixed name, like `db-0` and `db-1`, and its own dedicated storage that follows it around. If `db-1` crashes and moves to a different node, Kubernetes reconnects it to the exact same storage, so no data gets mixed up between pods. Also worth knowing — deleting a StatefulSet does not delete its storage automatically, that stays behind on purpose.

</details>

---

### Q: How do you enforce resource isolation and multi-tenancy using Namespaces, ResourceQuotas, and RBAC?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I give each team its own namespace, and then set a `ResourceQuota` on it to cap the total CPU, memory, and pod count that team can use — so one team can't accidentally eat up resources meant for everyone else. On top of that, a `LimitRange` sets sane defaults for any container that doesn't specify its own limits. For access control, I use RBAC and bind each team to a `Role` scoped to just their own namespace, never a cluster-wide role, so one team genuinely can't see or touch another team's stuff.

</details>

---

### Q: How do Custom Controllers, CRDs, and the Sidecar pattern extend Kubernetes core functionality?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A CRD lets you register a brand new type of object with Kubernetes, something that isn't built in by default. On its own, that's just a schema — it doesn't actually do anything. A Custom Controller is what makes it useful, it watches those objects and takes real action to keep the cluster matching what's declared. The Sidecar pattern is different — it's a second container running in the same pod as your app, sharing the same network. Istio uses this to automatically inject a proxy container into every pod, so traffic gets managed and secured without the app itself even knowing it's there.

</details>

---
