# Senior DevOps Interview Questions: Production Troubleshooting

### Q: How do you systematically troubleshoot a Docker container that cannot access the internet?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I check the host first, then the container, so I don't waste time chasing the wrong problem. If the host itself can't reach the internet, that's the real issue, nothing to do with Docker yet. If the host is fine, I go inside the container and test the same thing — a plain IP first, then a real domain name, since those can fail for different reasons. If the IP test works but the domain doesn't, it's a DNS setting problem inside the container. If even the IP test fails, I check if the host's forwarding setting is turned on, and check the firewall rules next. One real case — a firewall reload wiped out Docker's internal routing rules, and just restarting the Docker service rebuilt them and fixed it.

</details>

---

### Q: How do you diagnose and resolve a container or pod trapped in a CrashLoopBackOff state?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I check the logs from the last time it crashed, since the current instance has already restarted with a clean slate. Then I check the exit code, since that tells me what actually happened. A code of `137` almost always means it ran out of memory — the fix there is raising the memory limit. A code of `1` usually means the app itself hit an error, like a bad config or a missing secret — that's a code problem, not an infrastructure one. If the logs don't explain anything, I'll temporarily change the container's startup command to just keep it running, so I can get inside and check things by hand.

</details>

---

### Q: How do you troubleshoot and recover from an AWS Terraform state corruption or accidental deletion?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If the state file gets deleted and we've got versioning turned on for the backend bucket, recovery is quick — I just pull back the last good version from the bucket's history and put it back in place. That's usually a two-minute fix, which is exactly why I always make sure versioning is on in the first place. If there's no backup at all, it's a much longer job — I freeze all changes, go through the real infrastructure resource by resource, and rebuild the state manually with `terraform import`, checking after each one until nothing looks different anymore.

</details>

---
