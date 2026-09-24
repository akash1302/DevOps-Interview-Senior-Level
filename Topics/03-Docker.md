# Senior DevOps Interview Questions: Docker

### Q: How do multi-stage Docker builds optimize image size and security in production CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

With a single-stage build for something like Go or Java, you end up shipping the compiler, the SDK, and all your source files in the final image, and that easily pushes you over a gigabyte. Multi-stage builds fix that by letting you use one `FROM` for a builder stage that compiles everything, and then a completely separate, minimal `FROM` for the final runtime image that only copies over the compiled binary.

For a Go app, that looks like using `golang:1.22-alpine` as the builder, running `go build`, and then in a second stage starting fresh from `alpine:3.19` and just doing `COPY --from=builder /app/main .`. That shrinks the production image down to something like 20MB instead of over a gig, which speeds up registry pulls and pod startup, and just as importantly, it strips out the compiler and any build tooling that would otherwise sit in the image as extra CVE surface nobody's actually using at runtime.

</details>

---

### Q: What are the security risks of running Docker containers as root, and how do you mitigate them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, a container's process runs as root, UID 0, and that's a real risk — if there's ever a container escape or runtime vulnerability, an attacker inherits root-level access on the host, not just inside the container. Root processes also keep a default set of Linux capabilities, like `NET_ADMIN` or `SYS_ADMIN`, which give a lot more power than most apps actually need.

To mitigate this, I create a dedicated non-root user right in the Dockerfile and switch to it with `USER`:

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --chown=appuser:appgroup . .
USER appuser
```

On top of that, at container launch I strip every capability with `--cap-drop=ALL` and only add back the exact one that's needed — for a web server binding to a low port, that's usually just `NET_BIND_SERVICE`, nothing else. That's the principle of least privilege applied directly at the container runtime level, and in Kubernetes I enforce the same thing cluster-wide through `runAsNonRoot` in the pod security context.

</details>

---

### Q: How do you handle service dependency and startup readiness ordering in Docker Compose?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

The thing people get wrong here is assuming `depends_on` on its own means the dependency is actually ready. It doesn't — it only guarantees the other container has *started*, not that Postgres inside it is actually accepting connections yet. If your web app starts before the database is truly ready, it just crashes on the first connection attempt.

The fix is combining `depends_on` with a real `healthcheck` and `condition: service_healthy`. On the database service I'd define something like `test: ["CMD-SHELL", "pg_isready -U postgres"]` with an interval and retry count, and then on the web service I'd set `depends_on: db: condition: service_healthy`. That way Compose actually waits for the health check to pass, not just for the process to launch. And even with that in place, I still build exponential backoff retry logic into the app itself for connecting to the database, since that safety net handles any timing edge case Compose's health check doesn't fully cover.

</details>

---

### Q: How do you safely perform Docker image and system cleanup in production without impacting running workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In production, blind cleanup is genuinely dangerous — running something like `docker system prune -a --volumes` can wipe out untagged images you'd want for a fast rollback, or worse, delete a volume holding real data. So I never run that blind. I start by actually checking `docker system df` to see where the disk is actually going before touching anything.

From there, the safe sequence is targeted: `docker image prune` to clear out dangling, untagged layers, and `docker container prune` for stopped containers that aren't coming back. I'll also run `docker builder prune` for stale build cache. But `docker volume prune` never runs automatically in my pipelines — volumes can hold offline database data, so I always inspect with `docker volume ls -f dangling=true` and confirm manually before removing anything there. Anything automated gets scheduled during a maintenance window, not fired off mid-deploy.

</details>

---

### Q: What is the technical difference between CMD and ENTRYPOINT in a Dockerfile, and how do they interact?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`ENTRYPOINT` is the fixed executable that always runs — think of it as making the container behave like a dedicated binary. `CMD` supplies the default arguments to that executable, and the difference that trips people up is how easy each one is to override: `CMD` gets replaced just by passing new arguments to `docker run`, but overriding `ENTRYPOINT` needs the explicit `--entrypoint` flag.

A simple example makes this click — `ENTRYPOINT ["ping"]` with `CMD ["localhost"]`. Running `docker run my-ping` executes `ping localhost`, but running `docker run my-ping google.com` swaps out the `CMD` portion and executes `ping google.com`, while `ENTRYPOINT` stays untouched. And this only works cleanly in exec form, the array syntax — shell form, like `CMD echo hello` instead of `CMD ["echo", "hello"]`, wraps the process in `/bin/sh -c`, which breaks signal handling since `SIGTERM` never reaches your actual application process. That's exactly why I always use the array syntax for both.

</details>

---
