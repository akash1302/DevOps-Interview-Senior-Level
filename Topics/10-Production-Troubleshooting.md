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
