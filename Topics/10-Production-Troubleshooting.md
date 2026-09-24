# Senior DevOps Interview Questions: Production Troubleshooting

### Q: How do you systematically troubleshoot a Docker container that cannot access the internet?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check the host machine first, then the container. This way I don't waste time on the wrong problem. If the host itself can't reach the internet, that's the real issue, and it has nothing to do with Docker yet. If the host is fine, I go inside the container and test the same thing — a plain IP address first, then a real website name, since those can fail for different reasons. If the IP works but the name doesn't, it's a DNS setting problem inside the container. If even the IP fails, I check the host's network settings next. One real case I've seen — a firewall reset wiped out Docker's own network rules, and restarting Docker rebuilt them and fixed it.

</details>

---

### Q: How do you diagnose and resolve a container or pod trapped in a CrashLoopBackOff state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First, I check the logs from the last time it crashed, since the app has already restarted with a clean slate. Then I check the exit code, because that tells me what actually happened. A code of `137` almost always means it ran out of memory — the fix is raising the memory limit. A code of `1` usually means the app hit an error on its own, like a missing setting — that's a code problem, not an infrastructure one. If the logs don't explain anything, I'll change the startup command for a moment, just to keep it running, so I can get inside and check things by hand.

</details>

---

### Q: How do you troubleshoot and recover from an AWS Terraform state corruption or accidental deletion?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the state file gets deleted, and we have versioning turned on for that storage bucket, recovery is quick. I just pull back the last good version and put it in place. That's usually a two-minute fix, which is exactly why I always make sure versioning is turned on. If there's no backup at all, it's a much longer job. I stop all changes, go through the real infrastructure one piece at a time, and rebuild the state by hand, checking after each step until nothing looks different anymore.

</details>

---

### Q: Production suddenly gets very slow, but nothing actually crashed. How do you find the cause under pressure?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't start guessing right away, I check the basics first, in order. Is CPU or memory actually maxed out somewhere? Is the database slow, or is it the app itself? Is there a spike in traffic, or did traffic stay normal? I also check what changed recently, since a slowdown right after a deploy almost always points back to that deploy, and that's usually faster to confirm than digging through metrics from scratch. If I can't find the cause quickly and the deploy is the obvious suspect, I'll just roll back first and investigate the real cause after, since restoring the service matters more than proving what broke it in the moment.

</details>

---

### Q: An application keeps losing its connection to the database under normal load. How do you find out why?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is almost always a connection pool problem, not a network problem, so I check that first. I look at how many connections the app is actually allowed to open, versus how many the database allows in total — if many copies of the app are all running at once, they can easily add up past what the database allows. I also check if connections are being closed properly after use, since a leak there means the pool slowly fills up over time and never has room for new requests. Only after ruling those out would I actually suspect the network itself.

</details>

---

### Q: A third-party API your application depends on goes down. How do you stop that from taking your whole application down with it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never let one slow or dead dependency block everything else. I set a strict timeout on any call to an outside service, so my app doesn't just sit there waiting forever. I also add a circuit breaker — after a certain number of failures in a row, the app stops even trying to call that service for a while, and fails fast instead, which protects the rest of the app from slowing down too. Wherever possible, I design the feature that depends on that API to degrade gracefully — like showing cached or default data — instead of the whole page failing just because one third-party service is having a bad day.

</details>

---

### Q: You get paged at 2 AM because CPU on your production EC2 web servers is stuck above 90%, and users are seeing timeouts. What's your step-by-step approach?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

My first move is always to stop the bleeding, not find the root cause. I'd bump up the desired capacity on the Auto Scaling Group right away, so new instances come up and spread the load while I actually investigate. That's a quick fix, not the real answer, but it buys time and gets users unblocked fast.

While that's happening, I'm in CloudWatch checking CPU usage and whether instances are failing health checks. Then I connect into one of the bad instances — using Session Manager, so I don't need to mess with SSH keys — and run a live process monitor to see exactly what's eating the CPU. Is it the app itself, a stuck database connection, or something that shouldn't be running at all, like a crypto-miner from a break-in? I also check the app logs around the same time, to see if this lines up with a recent deploy or a real traffic spike. Depending on what I find, the fix might be killing a bad process, rolling back a recent release, or scaling the database — but the order is always the same: restore service first, then dig into why it happened.

</details>

---

### Q: Your RDS Postgres database has been getting slower over time, with high CPU and slow reads. How do you find and fix the actual bottleneck?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

High CPU with slow reads usually means bad queries or missing indexes, not a broken database. So my first stop is RDS Performance Insights — it shows me exactly which queries are actually using up the database's time, ranked by load, instead of me guessing which one is the problem.

Once I find the worst query, I run it through `EXPLAIN ANALYZE` to see how the database is actually executing it. Most of the time this shows a full table scan where an index should be doing the work instead. So the fix is usually adding an index on the right columns — but I'm careful not to overdo this, since every index also slows down writes a little. If the queries are already efficient and the load is just genuinely bigger than before, then it's not a query problem anymore, it's a sizing problem — I'd either scale up the instance or add a read replica to take some of the read traffic off the main database. I'd also turn on the slow query log, so the next time this happens, I've already got the data instead of starting from zero.

</details>

---
