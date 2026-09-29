# Senior DevOps Interview Questions: Docker

### Q: How do multi-stage Docker builds optimize image size and security in production CI/CD pipelines?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A normal single-stage build packs the compiler and all the source code into the final image, so it ends up huge, often over a gigabyte.

What I normally do is use a multi-stage build — one stage compiles the app, and a second, much smaller stage only copies over the finished file, nothing else.

For example, on a Go app, this took our image from over a gigabyte down to about 20MB. It's faster to pull, and it also removes build tools that shouldn't be sitting in a production image in the first place.

</details>

---

### Q: What are the security risks of running Docker containers as root, and how do you mitigate them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, a container runs as root, and that's risky, because if there's ever a container escape, whoever gets out has root on the real host too.

So what I normally do is create a non-root user in the Dockerfile and run the app as that user instead.

I also strip away extra Linux permissions at startup, and only add back the one thing the app actually needs, like permission to use a low network port. That way, even if something does go wrong, there's very little damage it can actually do.

</details>

---

### Q: How do you handle service dependency and startup readiness ordering in Docker Compose?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A common mistake is trusting `depends_on` alone. It only waits for the other container to start, not for it to actually be ready to accept connections.

So if the app starts before the database is really ready, it just crashes. The fix is adding a real health check to the database, and telling the app to wait for that health check to pass, not just for the container to start.

I also add retry logic inside the app itself, just in case the timing is still a little off even with the health check in place.

</details>

---

### Q: How do you safely perform Docker image and system cleanup in production without impacting running workloads?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never run a full cleanup command blindly in production. It can delete images I might still need for a quick rollback, or worse, delete a volume that has real data in it.

First I check how much space is actually being used. Then I clean up in small, safe steps — old unused images, stopped containers, old build cache.

I never delete volumes automatically. I always check them by hand first, since one of them might hold actual database data.

</details>

---

### Q: What is the technical difference between CMD and ENTRYPOINT in a Dockerfile, and how do they interact?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

`ENTRYPOINT` is the main command that always runs. `CMD` just gives it default arguments, and those are easy to override when you start the container.

For example, if `ENTRYPOINT` is `ping` and `CMD` is `localhost`, running it normally pings localhost. But I can pass a different address at run time, and that only changes the `CMD` part — the `ENTRYPOINT` stays the same.

One thing I always check is that this only works correctly using the array format, like `["ping", "localhost"]`. Using plain text instead can actually break how the container shuts down cleanly.

</details>

---

### Q: How does Docker's image layer caching work, and how do you write a Dockerfile that builds fast?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Docker builds an image line by line, and saves each line's result as a layer. If a layer hasn't changed since the last build, Docker just reuses it instead of redoing the work.

So the trick to a fast build is ordering the Dockerfile so the parts that change the least come first, and the parts that change the most, like the app code, come last.

What I normally do is copy dependency files and install dependencies before copying the rest of the source code. That way, changing one line of app code doesn't force Docker to redo the slow dependency install every time.

</details>

---

### Q: How do you set CPU and memory limits for containers, and what happens if a container goes over them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always set a memory limit on containers in production, never leave it unlimited.

If a container tries to use more memory than its limit, Linux kills the process outright. That's actually the safer outcome, since it fails fast and restarts, instead of slowly eating up all the memory on the host and taking other containers down with it.

CPU works differently — going over the limit doesn't kill the container, it just gets slowed down and shares the CPU more. So memory limits are really about survival, and CPU limits are more about being a good neighbor on a shared machine.

</details>

---

### Q: How do you handle logging for containers so logs don't fill up the disk and crash the host?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, Docker keeps writing container logs to a local file that can grow forever, and I've seen that fill up a disk and take down a whole host before.

So I set a log size limit and a rotation policy on the Docker daemon, so old logs get cleaned up automatically once they hit a certain size.

But really, in production, I don't rely on local log files at all. I ship logs out to a central logging system, so they're searchable in one place, and the local disk never becomes a single point of failure.

</details>

---
