# Senior DevOps Interview Questions: Security

## Q1. How do you implement defense-in-depth security for a production Kubernetes cluster?

### Answer
Defense-in-depth security for Kubernetes requires securing multiple operational layers: API access, workload isolation, container runtime, and networking. **API Level**: Enforce strict RBAC with least privilege, integrate OIDC identity providers, and enable API audit logging. **Workload Isolation**: Enforce Pod Security Standards (`restricted` profile), run containers as non-root, use read-only root filesystems, and apply ResourceQuotas. **Network Level**: Deny default traffic using NetworkPolicies and restrict pod-to-pod communication. **Data Level**: Encrypt secrets at rest in etcd using AWS KMS and inject runtime secrets dynamically.

### Interview Answer
"I implement defense-in-depth across four distinct layers. At the control plane layer, I restrict API access via RBAC, disable public API server endpoints, and turn on audit logging. At the network layer, I implement default-deny NetworkPolicies so pods can only talk to explicitly whitelisted endpoints. At the pod runtime layer, I enforce Pod Security Admission to block root execution and drop capabilities. Finally, I encrypt etcd at rest using KMS and use container vulnerability scanners in CI/CD."

### Practical Example
Default-deny all ingress and egress network policy in a production namespace:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Follow-up Questions
* How does KMS envelope encryption secure Kubernetes secrets stored inside etcd?
* What is the role of Mutating and Validating Admission Controllers in enforcing security policies?
* How does Kyverno or OPA Gatekeeper extend Kubernetes security governance?

### Key Points
* Defense-in-depth applies security controls at control plane, pod runtime, network, and data layers.
* NetworkPolicies enforce microsegmentation using default-deny traffic rules.
* Pod Security Standards block root processes and drop dangerous Linux capabilities.

---

## Q2. How do container image vulnerability scanners integrate into DevSecOps CI/CD pipelines?

### Answer
Container vulnerability scanners (such as Trivy, Clair, or Docker Scout) inspect container image layers for known Common Vulnerabilities and Exposures (CVEs) by cross-referencing OS package indexes and language dependency manifests against security databases (NVD, GitHub Security Advisories). Integrated into DevSecOps pipelines, scanners automatically analyze built images before pushing to registries. Pipelines enforce quality gates—automatically failing builds if vulnerabilities meeting specific severity thresholds (CRITICAL or HIGH) are detected.

### Interview Answer
"In our DevSecOps pipeline, immediately after the `docker build` stage, I run an automated Trivy security scan against the image artifact. I configure Trivy flags to fail the pipeline build (`--exit-code 1`) if any `CRITICAL` or `HIGH` severity CVEs are discovered. I also enable automated image scanning on push in container registries like AWS ECR, and schedule periodic scans against deployed images to catch newly disclosed vulnerabilities."

### Practical Example
CI Pipeline scan step using Trivy CLI:
```bash
trivy image   --severity HIGH,CRITICAL   --exit-code 1   --ignore-unfixed   my-app-image:v1.2.0
```

### Follow-up Questions
* What is the difference between an OS package vulnerability and a application language dependency vulnerability?
* How does generating a Software Bill of Materials (SBOM) improve software supply chain security?
* Why is the `--ignore-unfixed` flag used in automated CI pipeline security checks?

### Key Points
* Image scanners inspect filesystem layers for known CVEs against vulnerability databases.
* Pipelines enforce security gates by failing builds on CRITICAL/HIGH vulnerability discoveries.
* Container registry auto-scanning catches newly published CVEs on existing images.

---

## Q3. How do you securely handle application secret management in cloud and Kubernetes environments?

### Answer
Storing secrets in plain text, committing credentials to version control, or embedding secrets in container images introduces severe security risks. In cloud environments, secrets must be stored encrypted at rest using centralized secret managers like AWS Secrets Manager or HashiCorp Vault. In Kubernetes, default `Secret` manifests are only base64 encoded, not encrypted. Production setups use external secret operators (External Secrets Operator / Secrets Store CSI Driver) to retrieve secrets dynamically from cloud secret managers and inject them into pod memory or environment variables at runtime.

### Interview Answer
"I strictly enforce zero hardcoded secrets. We store production API keys and passwords in AWS Secrets Manager, encrypted with KMS. In Kubernetes, rather than manually creating native secrets—which are merely base64 encoded—we deploy the External Secrets Operator. It syncs secrets directly from AWS Secrets Manager into Kubernetes secret objects at runtime, allowing pods to mount them as in-memory volumes or environment variables without exposing plain text in Git."

### Practical Example
External Secrets Operator `ExternalSecret` manifest sync:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret-sync
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret-k8s
  data:
  - secretKey: password
    remoteRef:
      key: prod/db/credentials
      property: password
```

### Follow-up Questions
* Why is base64 encoding in native Kubernetes secrets insufficient for production security?
* How does the Secrets Store CSI Driver mount secrets as in-memory `tmpfs` volumes inside pods?
* How does IAM Roles for Service Accounts (IRSA) grant EKS pods fine-grained access to Secrets Manager?

### Key Points
* Base64 encoding is not encryption; native Kubernetes secrets require etcd KMS encryption.
* Use centralized secret management services (AWS Secrets Manager / HashiCorp Vault).
* External Secrets Operator syncs cloud secrets into Kubernetes dynamically at runtime.


---
