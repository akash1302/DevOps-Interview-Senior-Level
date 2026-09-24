# Senior DevOps Interview Questions: Security

### Q: How do you implement defense-in-depth security for a production Kubernetes cluster?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I think of this in layers, since no single fix covers everything. At the top, I lock down who can access the Kubernetes API using proper RBAC and real logins, not shared tokens. At the network layer, I block all pod-to-pod traffic by default and only open up the specific paths that are actually needed. At the pod level, nothing runs as root, and every pod gets a resource limit so one workload can't eat up everything else. And underneath all that, secrets are encrypted at rest, and every image gets scanned for known issues before it's ever allowed to run.

</details>

---

### Q: How do container image vulnerability scanners integrate into DevSecOps CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Right after the image is built, I run a scanner like Trivy against it, which checks both the OS packages and the app's own dependencies for known security issues. The important part is making the pipeline actually stop the build if something serious is found, not just log a warning nobody reads. I set it to fail the build on anything critical or high severity. I also turn on automatic scanning inside the registry itself, and re-scan images that are already running, since a new issue can get discovered well after the image was originally built.

</details>

---

### Q: How do you securely handle application secret management in cloud and Kubernetes environments?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Nothing sensitive ever goes into code or gets committed to Git, full stop. Real secrets live in a proper secrets manager, encrypted. The thing I always point out — a plain Kubernetes secret is only encoded, not actually encrypted, so anyone with access to it can read the real value in seconds. Instead, I use a tool that syncs secrets from the secrets manager directly into Kubernetes at run time, so the actual plaintext value never touches a config file or Git at any point.

</details>

---
