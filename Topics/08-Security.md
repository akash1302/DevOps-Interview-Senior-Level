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

### Q: How do you rotate secrets, like database passwords, without causing downtime for the application using them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never rotate a secret by just replacing the old one instantly everywhere, since anything still holding the old value in memory would suddenly fail. Instead, I use a secrets manager that supports rotation properly — it creates a new password, updates the actual database to accept both the old and new one for a short window, and then the app picks up the new value on its next normal refresh. Once I'm sure nothing is still using the old one, it gets fully disabled. The key idea is overlap — both values work for a short time, so nothing breaks in the gap.

</details>

---

### Q: What's the difference between encryption at rest and encryption in transit, and do you need both?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Encryption at rest protects data while it's sitting still, like in a database or on a disk — if someone steals the physical drive, they can't read it. Encryption in transit protects data while it's moving, like between a browser and a server — if someone's listening on the network, they can't read it either. These cover two completely different risks, so yes, I always want both. Having one without the other still leaves a real gap — say, data safely encrypted on disk, but sent across the network in plain text where anyone watching the traffic could read it.

</details>

---

### Q: What is the principle of least privilege, and how do you actually apply it in a real cloud environment, not just in theory?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Least privilege means giving someone, or some service, only the exact access they need, and nothing more. In practice, that means I never start with broad, full access and plan to restrict it later — I start with almost nothing, and add specific permissions only when something actually needs them and asks for it. I also review access regularly, since permissions tend to pile up over time as people change roles, and nobody ever goes back to remove the old ones unless it's a habit, not a one-time cleanup.

</details>

---

### Q: A developer accidentally pushes an AWS access key to a public GitHub repo, and it's been exposed for 6 hours. What's your emergency response?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The very first thing I do is kill the key, before I do anything else. I go straight into IAM, find that access key, and deactivate it, or just delete it outright. If it was tied to a role instead of a user, I'd detach the policy from that role so it instantly has no permissions left, even if someone still has the key.

Once the key is dead and can't be used anymore, I move to figuring out what actually happened during those 6 hours. I go through CloudTrail and look at every API call made using that key — what actions were taken, from which IP addresses, at what time. That tells me the real blast radius — did someone actually use it to spin up resources, read data they shouldn't have, or was it never actually touched by anyone. Based on what I find, I'd rotate anything else that key had access to, like database passwords, clean up any resources that shouldn't be there, and let the right people know what happened. After that, the real fix is prevention — adding a pre-commit check that blocks secrets from ever being pushed to Git in the first place, so this isn't just a one-time cleanup.

</details>

---
