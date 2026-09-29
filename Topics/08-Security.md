# Senior DevOps Interview Questions: Security

### Q: How do you implement defense-in-depth security for a production Kubernetes cluster?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I think of this as layers, since no single fix covers everything.

At the top, access to Kubernetes goes through real logins, not shared credentials. At the network layer, I block all traffic between pods by default and only open the specific paths that are actually needed. At the pod level, nothing runs as root, and every pod has a resource limit.

Underneath all of that, secrets are encrypted, and every image gets scanned for known problems before it's allowed to run.

</details>

---

### Q: How do container image vulnerability scanners integrate into DevSecOps CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Right after the image is built, I run a scanner against it, checking for known issues in both the OS packages and the app's own dependencies.

The important part is making the pipeline actually stop the build if something serious is found, not just print a warning that nobody reads.

I also turn on scanning inside the image registry itself, and I re-scan images that are already deployed, since a new issue can get discovered after the image was first built.

</details>

---

### Q: How do you securely handle application secret management in cloud and Kubernetes environments?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Nothing sensitive ever goes into code, and nothing gets saved in Git. Real secrets live in a proper secrets manager, fully encrypted.

One thing I always point out is that a plain Kubernetes secret is only encoded, not actually encrypted, so anyone with access to it can read the real value in seconds.

So instead, I use a tool that pulls secrets from the secrets manager directly into Kubernetes at run time, meaning the real value never touches a file or Git at any point.

</details>

---

### Q: How do you rotate secrets, like database passwords, without causing downtime for the application using them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never rotate a secret by just replacing it everywhere instantly, since anything still holding the old value in memory would suddenly break.

What I normally do is use a secrets manager that handles rotation properly — it creates a new password, updates the database to accept both the old and new one for a short window, and the app picks up the new value on its next normal refresh.

Once I'm confident nothing's still using the old value, it gets fully disabled. The key idea is that overlap window — both values work for a while, so nothing breaks in the gap.

</details>

---

### Q: What's the difference between encryption at rest and encryption in transit, and do you need both?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Encryption at rest protects data while it's sitting still, like on a disk — if someone steals the physical drive, they can't read it.

Encryption in transit protects data while it's moving, like between a browser and a server — if someone's watching the network, they can't read it either.

These cover two completely different risks, so yes, I always want both. Having one without the other leaves a real gap — data safely encrypted on disk, but sent across the network in plain text.

</details>

---

### Q: What is the principle of least privilege, and how do you actually apply it in a real cloud environment, not just in theory?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Least privilege means giving someone, or some service, only the exact access they need, and nothing more.

In practice, I never start with broad access and plan to restrict it later — I start with almost nothing, and add specific permissions only when something actually needs it and asks for it.

I also review access regularly, since permissions pile up over time as people change roles, and nobody removes the old ones unless it's an actual habit, not a one-time cleanup.

</details>

---

### Q: A developer accidentally pushes an AWS access key to a public GitHub repo, and it's been exposed for 6 hours. What's your emergency response?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The very first thing I do is kill the key, before anything else. I go into IAM, find it, and deactivate or delete it outright.

Once it's dead, I check CloudTrail for every API call made using that key during those 6 hours — what actions, from which IPs, at what time. That tells me the real blast radius.

Based on what I find, I rotate anything else that key had access to, clean up any resources that shouldn't be there, and let the right people know. After that, the real fix is prevention — adding a check that blocks secrets from ever being pushed to Git in the first place.

</details>

---
