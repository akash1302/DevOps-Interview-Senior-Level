# Senior DevOps Interview Questions: Kubernetes / EKS

### Q: How do Kubernetes Deployments execute zero-downtime rolling updates and rollbacks?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Deployments get zero-downtime rollouts through two knobs — `maxSurge`, which controls how many extra pods can exist above the desired count during the update, and `maxUnavailable`, which controls how many can be offline at once. When I push a new image tag, Kubernetes spins up a new ReplicaSet, and new pods only get added to the Service's endpoints once they actually pass their readiness probe — old pods get scaled down in parallel as new ones become ready.

For something where I really can't afford any capacity dip, I'll set `maxSurge: 25%` and `maxUnavailable: 0`, so at 10 replicas it can burst up to 13 pods during the rollout but never drops below the original 10 healthy ones. If something goes wrong mid-rollout, `kubectl rollout undo deployment/web-app --to-revision=2` shifts traffic straight back to the last stable ReplicaSet, basically instantly, since that old ReplicaSet's pods are just scaled back up. None of this works, though, if the readiness probes aren't actually configured to reflect real app health — that's the piece that makes the whole mechanism trustworthy.

</details>

---

### Q: How do Pod Disruption Budgets (PDB) handle voluntary disruptions during cluster maintenance?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A PDB protects against *voluntary* disruptions specifically — things like draining a node for an OS patch or a cluster upgrade — not against a hardware crash, which is involuntary and a PDB can't do anything about that. Without a PDB in place, draining a node could evict every single running replica of an app at once if they all happen to be scheduled there.

I set something like `minAvailable: 2` on a PDB tied to the app's label selector, and that plugs directly into the Kubernetes Eviction API — when someone tries to drain a node, the API checks the PDB first and simply blocks the eviction until enough replacement pods are healthy elsewhere to keep at least 2 available. It's a real, enforced guarantee, not just a suggestion. One thing I always double check — a single-replica deployment with `minAvailable: 1` will actually block a drain indefinitely, since there's no way to satisfy the budget without a second replica to fall back on, so PDBs only really make sense once you've got redundancy to protect.

</details>

---

### Q: How do StatefulSets differ from Deployments when managing stateful workloads requiring persistent storage?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Deployments are built for stateless apps — pods get random names, any replica is interchangeable with any other, and that's exactly what you want when nothing needs a stable identity. For something like Postgres or Elasticsearch, I use a StatefulSet instead, because those need a stable identity and dedicated storage that follows the pod around.

StatefulSets create pods with deterministic names — `db-0`, `db-1` — in order, each with its own stable DNS hostname through a headless Service, and each gets its own PersistentVolumeClaim auto-provisioned through `volumeClaimTemplates`, requesting something like 50Gi per pod. If `db-1` crashes and gets rescheduled onto a different node, Kubernetes reattaches that exact same PVC, so the data picks up right where it left off — no volume mix-up between replicas. Worth knowing too — deleting or scaling down a StatefulSet doesn't automatically delete the underlying PVCs, they stick around on purpose, which is exactly what you want, but it does mean cleanup is a deliberate, separate step if you actually want that storage gone.

</details>

---

### Q: How do you enforce resource isolation and multi-tenancy using Namespaces, ResourceQuotas, and RBAC?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

To build real multi-tenancy on a shared cluster, I start by isolating each team into its own Namespace, then apply a `ResourceQuota` to hard-cap what that namespace can consume in aggregate — CPU, memory, and pod count — so one noisy team can't starve everyone else on the same hardware. Something like:

```yaml
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

On top of the quota, a `LimitRange` sets sane default requests and limits for individual containers that don't specify their own, so nobody accidentally deploys an unbounded pod that eats the whole quota by itself. And then access is locked down with RBAC — I bind developers to namespace-scoped `Roles`, never `ClusterRoles`, so someone on team-alpha genuinely cannot see or touch workloads sitting in team-beta's namespace, even by accident.

</details>

---

### Q: How do Custom Controllers, CRDs, and the Sidecar pattern extend Kubernetes core functionality?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Custom Resource Definition registers a brand-new API type with the Kubernetes API server — something like a `VirtualService` object that isn't part of core Kubernetes at all. On its own, a CRD is just a schema. What actually makes it do something is a Custom Controller running a reconciliation loop that watches those objects and takes real action to bring the cluster's actual state in line with what's declared.

The Sidecar pattern is a different extension mechanism — it runs a second container inside the same pod as your app, sharing the same network namespace over `localhost` and the same volumes. Istio is the classic example — when a deployment gets submitted, a Mutating Admission Webhook intercepts it and injects an Envoy proxy container right into the pod spec, without the application code ever needing to know a proxy is even there. Both containers share networking through a shared Pause container under the hood, which is what makes `localhost` communication between them actually work.

</details>

---
