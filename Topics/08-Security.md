# Senior DevOps Interview Questions: Security

### Q: How do you implement defense-in-depth security for a production Kubernetes cluster?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I think about this in layers, because no single control covers everything. At the API level, I lock down access with least-privilege RBAC, integrate a real OIDC identity provider instead of static tokens, and turn on audit logging so every API action is traceable. At the network level, I default-deny all pod traffic and only open specific, whitelisted paths between services.

A baseline default-deny policy looks like this — a `NetworkPolicy` with an empty `podSelector` and both `Ingress` and `Egress` in `policyTypes`, applied cluster-wide in the namespace, blocking everything until I explicitly punch holes for what's actually needed. On top of the network layer, Pod Security Admission enforces the `restricted` profile so nothing runs as root or with dangerous capabilities, and `ResourceQuotas` stop any one workload from starving the rest. Underneath all of that, I encrypt secrets at rest in etcd using KMS, and I run vulnerability scanning in CI before anything even reaches the cluster — that's the data layer, the last line of defense if everything above it somehow gets bypassed.

</details>

---

### Q: How do container image vulnerability scanners integrate into DevSecOps CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Right after the `docker build` step, I run an automated scan against the image using Trivy, which checks both OS packages and language dependencies against known CVE databases. The key part is making the pipeline actually enforce something with that result, not just log it and move on — I configure it to fail the build outright if it finds anything CRITICAL or HIGH severity.

That's a command like `trivy image --severity HIGH,CRITICAL --exit-code 1 --ignore-unfixed my-app-image:v1.2.0`. The `--ignore-unfixed` flag matters in practice — without it, you'd be failing builds over CVEs that don't even have a patch available yet, which just creates noise nobody can act on. Beyond the CI gate, I also turn on automatic scanning in the registry itself, like ECR's built-in scan-on-push, and run periodic re-scans against images that are already deployed, since a CVE can get disclosed well after an image was originally built and passed clean.

</details>

---

### Q: How do you securely handle application secret management in cloud and Kubernetes environments?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I enforce zero hardcoded secrets, full stop — nothing goes into a Dockerfile, a config file, or gets committed to Git. Production credentials live in AWS Secrets Manager, encrypted with KMS. The thing I always flag when this comes up in Kubernetes specifically — a native `Secret` object is only base64 encoded, that's not encryption, anyone with read access to it can trivially decode the value.

So instead of hand-creating native secrets, I deploy the External Secrets Operator, which syncs secrets directly from Secrets Manager into Kubernetes secret objects at runtime — something like an `ExternalSecret` manifest pointing at `prod/db/credentials` with a `refreshInterval` of an hour, so it stays in sync automatically. Pods then mount that as an in-memory volume or environment variable, and the plaintext value never once touches Git or a config file. On EKS specifically, I pair that with IAM Roles for Service Accounts, so each pod's access to Secrets Manager is scoped tightly to exactly the secrets it actually needs, not a broad shared credential.

</details>

---
