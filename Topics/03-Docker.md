# Senior DevOps Interview Questions: Docker

### Q: How do multi-stage Docker builds optimize image size and security in production CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

With a normal single-stage build, your final image ends up carrying the compiler and all the source code too, so it's huge, often over a gigabyte. Multi-stage builds fix that — you use one stage just to compile the code, and a second, much smaller stage that only copies over the final built file. For a Go app, that means building in a full Go image, then copying just the compiled binary into a tiny Alpine image for the final result. That can shrink the image from over a gigabyte down to around 20MB, which pulls faster and also removes a lot of unnecessary tools that could be a security risk sitting in production.

</details>

---

### Q: What are the security risks of running Docker containers as root, and how do you mitigate them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, containers run as root, and that's risky — if there's ever a container escape, the attacker gets root access on the real host, not just inside the container. To fix this, I create a normal, non-root user right in the Dockerfile and switch to it before the app runs. On top of that, I strip away all Linux capabilities at startup with `--cap-drop=ALL`, and only add back the one thing that's actually needed, like `NET_BIND_SERVICE` if the app needs to bind to port 80. That way, even if something goes wrong, the container has almost no extra power to do damage.

</details>

---

### Q: How do you handle service dependency and startup readiness ordering in Docker Compose?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The mistake people make is trusting `depends_on` alone — it only waits for the other container to *start*, not for the app inside it to actually be ready. So if your app starts before the database is really accepting connections, it just crashes right away. The fix is adding a real `healthcheck` to the database, and setting `condition: service_healthy` on the app that depends on it, so Compose actually waits for the database to pass its health check first. I also add retry logic in the app itself as a backup, just in case the timing is still off by a second or two.

</details>

---

### Q: How do you safely perform Docker image and system cleanup in production without impacting running workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never run a full `docker system prune` blindly in production — it can wipe out images I might need for a quick rollback, or worse, delete a volume holding real data. Instead, I check `docker system df` first to actually see what's using the space. Then I clean up in a targeted way — dangling images, stopped containers, old build cache. But volumes never get cleaned automatically in my scripts, I always check them by hand first, since one of them might be holding a database's actual data.

</details>

---

### Q: What is the technical difference between CMD and ENTRYPOINT in a Dockerfile, and how do they interact?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`ENTRYPOINT` is the main command that always runs. `CMD` just gives it default arguments, which are easy to override when you run the container. So if I set `ENTRYPOINT ["ping"]` and `CMD ["localhost"]`, running the container normally pings localhost, but passing a different address at run time overrides just the `CMD` part, while the `ENTRYPOINT` stays fixed. One thing I always check — this only works right using the array format, like `["ping", "localhost"]`. Using the plain string format instead wraps everything in a shell, and that actually breaks how the container receives shutdown signals.

</details>

---
