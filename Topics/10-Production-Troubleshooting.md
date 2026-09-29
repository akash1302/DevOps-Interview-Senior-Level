# Senior DevOps Interview Questions: Production Troubleshooting

### Q: How do you systematically troubleshoot a Docker container that cannot access the internet?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check the host machine first, then the container, so I don't waste time on the wrong problem. If the host itself can't reach the internet, that's the real issue, and it has nothing to do with Docker yet.

If the host is fine, I go inside the container and test the same thing — a plain IP address first, then a real website name, since those fail for different reasons. If IP works but the name doesn't, it's DNS. If even the IP fails, I check the host's network settings next.

For example, I had a case once where a firewall reset wiped out Docker's own network rules, and restarting Docker rebuilt them and fixed it.

</details>

---

### Q: How do you diagnose and resolve a container or pod trapped in a CrashLoopBackOff state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I check the logs from the last crash, since the app has already restarted with a clean slate. Then I check the exit code, because that tells me what actually happened.

A code of 137 almost always means it ran out of memory, so the fix is raising the memory limit. A code of 1 usually means the app hit an error on its own, like a missing setting — that's a code problem, not an infrastructure one.

If the logs don't explain anything, I'll change the startup command for a moment just to keep it running, so I can get inside and check things by hand.

</details>

---

### Q: How do you troubleshoot and recover from an AWS Terraform state corruption or accidental deletion?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the state file gets deleted and versioning is turned on for that storage bucket, recovery is quick — I just pull back the last good version. That's usually a two-minute fix, which is exactly why I always make sure versioning is on in the first place.

If there's no backup at all, it's a much longer job. I stop all changes, go through the real infrastructure one piece at a time, and rebuild the state by hand, checking after each step until nothing looks different.

</details>

---

### Q: Production suddenly gets very slow, but nothing actually crashed. How do you find the cause under pressure?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't start guessing, I check the basics first, in order. Is CPU or memory actually maxed out somewhere? Is the database slow, or is it the app itself? Is traffic higher than normal?

I also check what changed recently, since a slowdown right after a deploy almost always points back to that deploy.

If I can't find the cause quickly and the deploy is the obvious suspect, I'll just roll back first and investigate the real cause after — restoring the service matters more than proving what broke it in the moment.

</details>

---

### Q: An application keeps losing its connection to the database under normal load. How do you find out why?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is almost always a connection pool problem, not a network problem, so I check that first.

I look at how many connections the app is allowed to open, versus how many the database allows in total. If a lot of copies of the app are running at once, that math can easily add up past what the database allows.

I also check if connections are being closed properly after use, since a leak means the pool slowly fills up and never has room for new requests. Only after ruling those out would I actually suspect the network itself.

</details>

---

### Q: A third-party API your application depends on goes down. How do you stop that from taking your whole application down with it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never let one slow or dead dependency block everything else. I set a strict timeout on any call to an outside service, so my app doesn't just sit there waiting forever.

I also add a circuit breaker — after a certain number of failures in a row, the app stops even trying to call that service for a while and fails fast instead, which protects the rest of the app.

Wherever possible, I design that feature to degrade gracefully, like showing cached or default data, instead of the whole page failing just because one third-party service is having a bad day.

</details>

---

### Q: You get paged at 2 AM because CPU on your production EC2 web servers is stuck above 90%, and users are seeing timeouts. What's your step-by-step approach?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

My first move is always to stop the bleeding, not find the root cause. I'd bump up the Auto Scaling Group's desired capacity right away, so new instances come up and spread the load while I actually investigate.

While that's happening, I'm checking CloudWatch for CPU usage and failing health checks. Then I connect into one of the bad instances and run a live process monitor to see exactly what's eating the CPU — the app itself, a stuck database connection, or something that shouldn't be running at all.

I also check the app logs, to see if this lines up with a recent deploy or a real traffic spike. The order is always the same — restore service first, then dig into why it happened.

</details>

---

### Q: Your RDS Postgres database has been getting slower over time, with high CPU and slow reads. How do you find and fix the actual bottleneck?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

High CPU with slow reads usually means bad queries or missing indexes, not a broken database. So my first stop is RDS Performance Insights — it shows me exactly which queries are using up the database's time, ranked by load.

Once I find the worst one, I run it through `EXPLAIN ANALYZE` to see how it's actually being executed. Most of the time that shows a full table scan where an index should be doing the work instead, so the fix is adding an index on the right columns.

If the queries are already efficient and the load's just genuinely bigger than before, that's not a query problem anymore, it's a sizing problem, and I'd scale up the instance or add a read replica.

</details>

---
