# Senior DevOps Interview Questions: Kubernetes / EKS

### Q: How do Kubernetes Deployments execute zero-downtime rolling updates and rollbacks?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A rolling update uses two simple settings. One controls how many extra pods can be added while updating. The other controls how many old pods can go offline at once. When I push a new version, Kubernetes starts new pods, waits for each one to pass its health check, and only then shuts down an old one. If I set the offline number to zero, the app never loses capacity during the update. If something breaks halfway through, I can run one command to switch traffic straight back to the last working version.

</details>

---

### Q: How do Pod Disruption Budgets (PDB) handle voluntary disruptions during cluster maintenance?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A PDB protects against planned actions, like taking a server down for an upgrade. It does not protect against a random crash. Without a PDB, taking a server down could accidentally kill every copy of an app at once, if they're all sitting on that server. I set a rule like "at least 2 copies must stay running." Kubernetes will then block the server from being taken down until enough healthy copies exist somewhere else. One thing to watch for — if an app only has one copy, this rule can block the update forever, since there's no backup copy to rely on.

</details>

---

### Q: How do StatefulSets differ from Deployments when managing stateful workloads requiring persistent storage?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A Deployment is for apps where any copy can replace any other copy, no problem. That's fine for something stateless, like a web server with no memory of past requests. For something like a database, I use a StatefulSet instead, because it needs its own identity and its own storage. Each copy gets a fixed name and its own dedicated storage that follows it around, even if it moves to a different server. Also — deleting a StatefulSet does not delete its storage. That storage stays behind on purpose, so data isn't lost by accident.

</details>

---

### Q: How do you enforce resource isolation and multi-tenancy using Namespaces, ResourceQuotas, and RBAC?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I give each team its own namespace, like a separate folder. Then I set a limit on that namespace — how much CPU and memory it's allowed to use in total. This stops one team from using up resources that other teams need. I also set default limits for any app that doesn't set its own. For access control, I give each team permission only inside their own namespace, never across the whole cluster. That way, one team genuinely can't see or touch another team's stuff.

</details>

---

### Q: How do Custom Controllers, CRDs, and the Sidecar pattern extend Kubernetes core functionality?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A CRD lets you add a brand new type of object to Kubernetes, one that isn't built in. On its own, that's just a definition, it doesn't do anything by itself. A Custom Controller is what actually makes it work — it watches for those objects and takes action to keep things matching what's expected. The Sidecar pattern is different. It's a second, helper container that runs next to your app, inside the same pod. Istio uses this to automatically add a network helper to every app, without changing the app's own code at all.

</details>

---
