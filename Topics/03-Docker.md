# Senior DevOps Interview Questions: Docker

## Q1. How do multi-stage Docker builds optimize image size and security in production CI/CD pipelines?

### Answer
Multi-stage Docker builds utilize multiple `FROM` statements within a single `Dockerfile`. Each stage can use a distinct base image, allowing developers to compile code, download heavy dependencies, and build binaries in an early "builder" stage. Subsequent stages copy only the final compiled binaries or required artifacts into a minimal, clean runtime base image (e.g., Alpine Linux or Distroless). This drastically reduces image size and removes compilers, build tools, and source code from production images, minimizing the attack surface.

### Interview Answer
"In a single-stage build for Go or Java, you end up shipping SDKs, compilers, and source files, resulting in images over 1 GB. With multi-stage builds, I use a full Golang image as the builder stage to compile the binary, then copy just that single compiled binary into a minimal Alpine or Scratch base image. This shrinks the production image down to 20 MB, speeds up deployment pulls, and drastically reduces CVE vulnerability surfaces."

### Practical Example
Multi-stage `Dockerfile` for a Go application:
```dockerfile
# Stage 1: Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Stage 2: Minimal runtime stage
FROM alpine:3.19
WORKDIR /app
COPY --from=builder /app/main .
EXPOSE 8080
CMD ["./main"]
```

### Follow-up Questions
* What is the difference between `alpine` and `scratch` base images in Docker?
* How does caching work across multiple stages during `docker build` in CI pipelines?
* Why does excluding build tools like `gcc` or `git` improve production runtime security?

### Key Points
* Multi-stage builds use multiple `FROM` instructions to isolate build and runtime environments.
* Production images contain only compiled binaries and essential dependencies.
* Image sizes drop significantly (e.g., from 1GB to ~20MB), speeding up registry pushes and pod launch times.

---

## Q2. What are the security risks of running Docker containers as root, and how do you mitigate them?

### Answer
By default, Docker containers run their processes as the `root` user (`UID 0`). If a container vulnerability or runtime escape occurs, an attacker gain root-level host access, leading to host compromise. Root processes inside containers also retain Linux kernel capabilities (`NET_ADMIN`, `SYS_ADMIN`) and can tamper with mounted host volumes. Mitigation requires creating and switching to a non-root dedicated user in the `Dockerfile`, dropping unneeded kernel capabilities using `--cap-drop=ALL`, and enforcing non-root Execution policies via container orchestrators.

### Interview Answer
"Running as root exposes the underlying host to privilege escalation if a container breakout vulnerability occurs. To mitigate this, I create a dedicated system group and user in my Dockerfile and switch to it using the `USER` instruction. I also enforce running as non-root in Kubernetes security contexts and drop all default Linux kernel capabilities using `--cap-drop=ALL`, explicitly adding back only what's required like `NET_BIND_SERVICE`."

### Practical Example
Creating and using a non-root user in a `Dockerfile`:
```dockerfile
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```
CLI execution dropping capabilities:
`docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE my-app:v1`

### Follow-up Questions
* How does the Docker User Namespace (`userns-remap`) feature protect the host system?
* What happens if a non-root container user tries to bind to a low port (e.g., port 80)?
* How do Kubernetes Pod Security Standards enforce `runAsNonRoot` at cluster runtime?

### Key Points
* Running as root inside a container risks full host compromise upon container escape.
* Use the `USER` instruction in Dockerfiles to run processes under dedicated non-root users.
* Use Linux capability dropping (`--cap-drop=ALL`) to restrict kernel privilege access.

---

## Q3. How do you handle service dependency and startup readiness ordering in Docker Compose?

### Answer
In Docker Compose, the basic `depends_on` instruction only guarantees that dependency containers are *started*, not that the applications inside them are *ready* to accept network traffic. If a web application starts before its database completes initialization, the web app will crash. To enforce true readiness, Docker Compose uses `depends_on` combined with `condition: service_healthy` coupled to container `healthcheck` definitions, or shell startup scripts like `wait-for-it.sh` and application-level retry logic.

### Interview Answer
"Using plain `depends_on` only waits for the database container to launch, not for MySQL or Postgres to accept connections. To solve this, I define a `healthcheck` block in the database service—like running `pg_isready`—and set `depends_on: db: condition: service_healthy` on the web app service. In code, I also implement exponential backoff retry logic for database connections so the app resiliently handles temporary database startup delays."

### Practical Example
`docker-compose.yml` readiness enforcement:
```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secretpassword
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
```

### Follow-up Questions
* Why does `depends_on` with `condition: service_healthy` fail when running in Docker Swarm mode?
* How does application-level retry logic with exponential backoff prevent connection crash loops?
* What is the purpose of the `wait-for-it.sh` shell script pattern in containerized environments?

### Key Points
* Plain `depends_on` only tracks container process launch, not application health or port readiness.
* Combine `depends_on` with `service_healthy` and explicit `healthcheck` commands.
* Always build application-level retry mechanisms to handle asynchronous initialization.

---

## Q4. How do you safely perform Docker image and system cleanup in production without impacting running workloads?

### Answer
Over time, Docker environments accumulate stopped containers, unused networks, dangling build caches, and unreferenced images, consuming host disk space. Running aggressive commands like `docker system prune -a --volumes` in production is dangerous because it can destroy stopped containers, untagged images required for fast rollbacks, or orphan volumes storing persistent data. Safe production cleanup requires inspecting disk usage via `docker system df` and running targeted prune commands targeting dangling resources (`docker image prune`, `docker container prune`).

### Interview Answer
"In production, blind cleanup is dangerous. I start by auditing disk usage with `docker system df`. To clean up safely without deleting active images or persistent volumes, I run `docker image prune` to remove dangling `<none>` layers, and `docker container prune` to clear stopped containers. I never run `docker volume prune` automatically without filtering because it can delete offline database volumes. I automate safe dangling layer cleanup via cron jobs scheduled during maintenance windows."

### Practical Example
Step-by-step safe production cleanup sequence:
1. Inspect disk usage: `docker system df`
2. Remove dangling (untagged) images safely: `docker image prune`
3. Remove stopped containers: `docker container prune`
4. Filter and remove dangling build caches: `docker builder prune`
5. Inspect dangling volumes safely before removal: `docker volume ls -f dangling=true`

### Follow-up Questions
* What constitutes a "dangling" Docker image versus an "unused" Docker image?
* How can `--filter "until=24h"` be added to prune commands to prevent deleting recent image layers?
* What risks are associated with executing `docker volume prune` in production?

### Key Points
* Always inspect disk allocation first using `docker system df`.
* `docker image prune` safely removes untagged dangling build layers.
* Never execute `docker system prune --volumes` in production without manual volume checks.

---

## Q5. What is the technical difference between CMD and ENTRYPOINT in a Dockerfile, and how do they interact?

### Answer
`ENTRYPOINT` defines the fixed executable that will always run when the container starts, whereas `CMD` provides default arguments passed to that executable (or defines a default command if `ENTRYPOINT` is omitted). `CMD` parameters can be easily overridden from the command line interface during `docker run`, whereas `ENTRYPOINT` parameters require explicit flag syntax (`--entrypoint`) to override. When combined in exec form (`["executable", "param"]`), `ENTRYPOINT` acts as the command and `CMD` acts as default appendable arguments.

### Interview Answer
"`ENTRYPOINT` is for setting the main fixed executable—like `python` or `nginx`—making the container behave like a dedicated binary tool. `CMD` provides default arguments to that executable that users can override at runtime. When I combine them, I use `ENTRYPOINT ["nginx"]` for the binary and `CMD ["-g", "daemon off;"]` for the default flags. If a developer runs `docker run my-nginx -t`, Docker replaces `CMD` with `-t` while keeping `ENTRYPOINT` intact."

### Practical Example
Dockerfile definition:
```dockerfile
FROM alpine
ENTRYPOINT ["ping"]
CMD ["localhost"]
```
Behavior:
* `docker run my-ping` -> Executes: `ping localhost`
* `docker run my-ping google.com` -> Executes: `ping google.com` (overrides `CMD`)

### Follow-up Questions
* What is the difference between Shell form (`CMD echo hello`) and Exec form (`CMD ["echo", "hello"]`)?
* Why does Shell form prevent Linux signals (like `SIGTERM`) from reaching application child processes?
* How do you override `ENTRYPOINT` when executing `docker run`?

### Key Points
* `ENTRYPOINT` specifies the main immutable container binary executable.
* `CMD` defines default parameters that CLI arguments can easily override at launch.
* Always use Exec syntax `["executable", "param"]` to ensure PID 1 passes OS signals properly.


---
