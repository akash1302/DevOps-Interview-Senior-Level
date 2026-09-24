# Senior DevOps Interview Questions: Kubernetes / EKS

## Q1. How do Kubernetes Deployments execute zero-downtime rolling updates and rollbacks?

### Answer
Kubernetes Deployments manage application updates declaratively using ReplicaSets. During a rolling update, the Deployment controller creates a new ReplicaSet running the updated container image and gradually scales up its pod replica count while simultaneously scaling down the old ReplicaSet. The rate of pod replacement is controlled by `maxSurge` (how many pods can exist above the desired count) and `maxUnavailable` (how many pods can be offline during the update). If deployment health checks fail, `kubectl rollout undo deployment/<name>` instantly rolls back traffic to the previous ReplicaSet.

### Interview Answer
"Deployments achieve zero downtime by using `maxSurge` and `maxUnavailable` strategy parameters. When I update an image tag, Kubernetes provisions a new ReplicaSet and spins up new pods. Kubernetes waits for new pods to pass readiness probes before adding them to Service endpoint slices and terminating old pods in parallel. If errors occur mid-rollout, I run `kubectl rollout undo` to immediately shift traffic back to the stable old ReplicaSet."

### Practical Example
Deployment strategy definition:
```yaml
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # Up to 13 pods during update
      maxUnavailable: 0    # Ensures 100% current capacity remains active
```
Rollback execution command:
`kubectl rollout undo deployment/web-app --to-revision=2`

### Follow-up Questions
* Why must readiness probes be configured properly for zero-downtime rolling updates to succeed?
* What is the difference between `maxSurge` expressed as a percentage versus an absolute integer?
* How does `kubectl rollout status` help automate deployment validation in CI/CD pipelines?

### Key Points
* Rolling updates scale new ReplicaSets up while scaling old ReplicaSets down incrementally.
* `maxUnavailable: 0` ensures existing application capacity is never reduced during updates.
* Rollbacks revert traffic instantaneously by re-scaling previous ReplicaSets.

---

## Q2. How do Pod Disruption Budgets (PDB) handle voluntary disruptions during cluster maintenance?

### Answer
A Pod Disruption Budget (PDB) limits the number of pods of a replicated application that can be down simultaneously during voluntary disruptions—such as node draining, cluster upgrades, or cluster autoscaler node scale-downs. Unlike involuntary disruptions (hardware crashes or network partitions), voluntary disruptions interact with the Kubernetes Eviction API. The Eviction API checks the PDB rules (`minAvailable` or `maxUnavailable`) before allowing a node drain operation to evict pods, preventing accidental application outages.

### Interview Answer
"Voluntary disruptions occur when admins drain nodes for OS patching or EKS cluster upgrades. Without a PDB, draining a node might evict all running instances of an application at once. By defining a PDB with `minAvailable: 80%` or `minAvailable: 2`, the Kubernetes Eviction API blocks node draining until replacement pods are running on other nodes, guaranteeing that application availability SLAs remain intact."

### Practical Example
PDB specification ensuring at least 2 replicas remain active:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-service
```

### Follow-up Questions
* What is the critical difference between voluntary and involuntary disruptions?
* What happens if a node drain command is executed on a single-replica deployment with `minAvailable: 1`?
* How does Cluster Autoscaler interact with Pod Disruption Budgets when scaling down nodes?

### Key Points
* PDBs protect applications specifically against voluntary cluster maintenance disruptions.
* The Eviction API enforces PDB rules (`minAvailable` / `maxUnavailable`) before terminating pods.
* PDBs do not prevent outages caused by involuntary hardware failures or kernel crashes.

---

## Q3. How do StatefulSets differ from Deployments when managing stateful workloads requiring persistent storage?

### Answer
Deployments manage stateless pods with interchangeable identities and random pod names (`app-75bdf48447-x9z2l`). In contrast, StatefulSets manage stateful workloads (like databases, Zookeeper, or Kafka) requiring unique network identities, ordered deployment and scaling, and sticky persistent storage. Each pod in a StatefulSet receives a deterministic ordinal index (`app-0`, `app-1`), a stable Headless Service DNS hostname, and a dedicated PersistentVolumeClaim (PVC) auto-provisioned via `volumeClaimTemplates` that persists across pod reschedules.

### Interview Answer
"Stateless apps use Deployments because any pod can replace any other. For databases like PostgreSQL or Elasticsearch, I use StatefulSets because they require stable identities and dedicated storage. StatefulSets create pods sequentially (`db-0`, `db-1`) with deterministic DNS names, and attach dedicated PVCs using `volumeClaimTemplates`. If `db-1` crashes and reschedules onto another node, Kubernetes reattaches the exact same persistent storage volume to maintain data continuity."

### Practical Example
StatefulSet `volumeClaimTemplates` snippet:
```yaml
spec:
  serviceName: "postgres"
  replicas: 3
  template:
    ...
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 50Gi
```

### Follow-up Questions
* Why do StatefulSets require a Headless Service (`clusterIP: None`) for network identity?
* What happens to PersistentVolumeClaims when a StatefulSet is scaled down or deleted?
* How does the `OrderedReady` pod management policy differ from `Parallel` in StatefulSets?

### Key Points
* StatefulSets provide deterministic pod naming (`app-0`), stable DNS, and sequential rollout.
* `volumeClaimTemplates` auto-provision dedicated PVCs bound to specific pod ordinals.
* Deleting or scaling down a StatefulSet does not automatically delete underlying PVCs/PVs.

---

## Q4. How do you enforce resource isolation and multi-tenancy using Namespaces, ResourceQuotas, and RBAC?

### Answer
Kubernetes multi-tenancy partitions a single physical cluster into isolated virtual environments using Namespaces. Resource isolation is enforced by defining `ResourceQuota` objects per namespace to cap total CPU, Memory, Storage, and Pod counts, preventing a single tenant from monopolizing cluster hardware. `LimitRange` objects set default/max CPU and memory requests and limits for individual containers. Access control is enforced via Role-Based Access Control (RBAC), binding Roles/ClusterRoles to users or ServiceAccounts to restrict API operations.

### Interview Answer
"To build secure multi-tenancy, I isolate teams into dedicated Namespaces. I apply a `ResourceQuota` to each namespace to hard-cap aggregate CPU and RAM consumption, and a `LimitRange` to enforce default container requests and limits. Finally, I write fine-grained RBAC policies—binding developers to namespace-scoped `Roles` rather than `ClusterRoles`—ensuring they can only view and manage workloads within their designated team namespace."

### Practical Example
Namespace `ResourceQuota` specification:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

### Follow-up Questions
* What is the difference between a `RoleBinding` and a `ClusterRoleBinding` in Kubernetes RBAC?
* How do NetworkPolicies complement Namespaces to enforce network-level tenant isolation?
* What happens if a developer tries to deploy a pod without specifying resource requests in a namespace with a ResourceQuota?

### Key Points
* Namespaces create logical virtual cluster boundaries for resource isolation.
* `ResourceQuota` caps aggregate namespace resource usage; `LimitRange` enforces container-level defaults.
* RBAC `Roles` restrict tenant actions strictly to their assigned namespace boundaries.

---

## Q5. How do Custom Controllers, CRDs, and the Sidecar pattern extend Kubernetes core functionality?

### Answer
Kubernetes extensibility rests on Custom Resource Definitions (CRDs) and the Operator/Custom Controller pattern. A CRD registers new custom API object types (e.g., `VirtualService` or `CertManager`) with the API server. A Custom Controller continuously runs a reconciliation loop watching the custom resources, taking operational actions to align current state with desired state. The Sidecar pattern runs a secondary container inside the same pod (sharing localhost network and storage) to enhance the main app container—e.g., Envoy proxies in Service Meshes (Istio) or log shippers (Fluentbit).

### Interview Answer
"CRDs allow us to define custom declarative APIs beyond built-in objects like Pods or Services. A Custom Controller watches these CRDs in a control loop, executing custom logic to handle operations—like auto-provisioning database instances. The Sidecar pattern injects a helper container alongside the main app in the same pod. For instance, Istio uses mutating admission webhooks to inject an Envoy sidecar proxy into app pods, enabling mTLS and traffic management transparently without modifying app code."

### Practical Example
In Istio, when a deployment manifest is submitted, a Mutating Admission Webhook intercepts the request and injects an Envoy proxy sidecar container into the pod spec alongside the application container. Both containers share the pod's network namespace (`localhost`) via a Pause container.

### Follow-up Questions
* What role does the Pause container play in sharing networking across containers within a single pod?
* What is the difference between a Mutating Admission Webhook and a Validating Admission Webhook?
* How does the reconciliation loop (`Reconcile()`) in Custom Controllers maintain state alignment?

### Key Points
* CRDs register custom schema definitions with the Kubernetes API server.
* Custom Controllers execute reconciliation loops to manage custom resource states.
* Sidecars run alongside main application containers, sharing `localhost` networking and volumes.


---
