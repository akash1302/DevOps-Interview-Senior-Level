# Senior DevOps Interview Questions: Docker

### Q: How do multi-stage Docker builds optimize image size and security in production CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A normal build packs the compiler and all the source code into the final image, so it ends up huge, often over a gigabyte. A multi-stage build fixes this. One stage builds the app. A second, much smaller stage only copies the finished file, and nothing else. For a Go app, this can take the image from over a gigabyte down to about 20MB. It's faster to download, and it also removes tools that aren't needed in production and could be a security risk.

</details>

---

### Q: What are the security risks of running Docker containers as root, and how do you mitigate them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, a container runs as root. That's risky, because if someone ever breaks out of the container, they get root on the real machine too. So I create a normal, non-root user in the Dockerfile, and I run the app as that user instead. I also strip away all extra Linux permissions at startup, and only add back the one thing the app actually needs, like the permission to use a low network port. This way, even if something goes wrong, there's very little damage it can do.

</details>

---

### Q: How do you handle service dependency and startup readiness ordering in Docker Compose?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A common mistake is trusting `depends_on` alone. It only waits for the other container to *start*, not for it to actually be ready. So if the app starts before the database is really ready, it just crashes. The fix is adding a real health check to the database, and telling the app to wait for that health check to pass, not just for the container to start. I also add retry logic inside the app itself, just in case the timing is still a little off.

</details>

---

### Q: How do you safely perform Docker image and system cleanup in production without impacting running workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never run a full cleanup command blindly in production. It can delete images I might still need for a fast rollback, or worse, delete a volume with real data in it. Instead, I first check how much space is actually being used. Then I clean up in small, safe steps — old unused images, stopped containers, old build cache. I never delete volumes automatically. I always check them by hand first, since one of them might hold real database data.

</details>

---

### Q: What is the technical difference between CMD and ENTRYPOINT in a Dockerfile, and how do they interact?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`ENTRYPOINT` is the main command that always runs. `CMD` just gives it default settings, and those are easy to change when you start the container. So if `ENTRYPOINT` is `ping` and `CMD` is `localhost`, running it normally pings localhost. But I can pass a different address when I start it, and that only changes the `CMD` part — the `ENTRYPOINT` stays the same. One thing to always check — this only works right using the list format, like `["ping", "localhost"]`. Using plain text instead can break how the container shuts down cleanly.

</details>

---
