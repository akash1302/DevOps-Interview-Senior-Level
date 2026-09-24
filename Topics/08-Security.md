# Senior DevOps Interview Questions: Security

### Q: How do you implement defense-in-depth security for a production Kubernetes cluster?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I think of this as layers, since no single fix covers everything. At the top, I control who can access Kubernetes, using real logins, not shared passwords. At the network layer, I block all traffic between pods by default, and only open up the specific paths that are actually needed. At the pod level, nothing runs as root, and every pod has a limit on how much it can use. Under all of that, secrets are encrypted, and every image is scanned for known problems before it's allowed to run.

</details>

---

### Q: How do container image vulnerability scanners integrate into DevSecOps CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Right after the image is built, I run a scanner against it, which checks for known security problems in both the operating system and the app's own dependencies. The important part is making the pipeline actually stop the build if something serious is found, not just print a warning that nobody reads. I also turn on scanning inside the image registry itself, and I re-scan images that are already running, since a new problem can get discovered after the image was first built.

</details>

---

### Q: How do you securely handle application secret management in cloud and Kubernetes environments?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Nothing sensitive ever goes into code, and nothing ever gets saved in Git. Real secrets live in a proper secrets manager, fully encrypted. One thing I always point out — a plain Kubernetes secret is only encoded, not actually encrypted, so anyone with access to it can read the real value in seconds. So instead, I use a tool that pulls secrets from the secrets manager directly into Kubernetes at run time, meaning the real value never touches a file or Git at any point.

</details>

---
