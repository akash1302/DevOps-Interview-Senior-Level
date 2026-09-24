# Senior DevOps Interview Questions: Linux

### Q: How do you systematically troubleshoot network connectivity issues at the Linux host and container level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always check the host machine first, then the container. First, I try to ping a known address from the host itself. If that fails, it's a host or firewall problem, and it has nothing to do with the container yet. Then I check DNS separately, since that's a different kind of problem. If the host is fine, I go inside the container and repeat the same two checks. If a plain IP address works but a website name doesn't, it's a DNS setting problem inside the container. If even the IP address fails, I check the host's network settings and firewall rules next.

</details>

---

### Q: How do Linux kernel Namespaces, Cgroups, and Capabilities isolate container processes on a host machine?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A container isn't a full separate machine, it's just a regular process with some limits added by Linux. **Namespaces** control what the process can actually see — so it can't see the host's real processes or the host's real network. **Cgroups** control what the process can actually use — CPU and memory — so one container can't eat up the whole machine. **Capabilities** break down root's full power into small pieces, so even a process running as root inside the container can be blocked from doing most of what real root access normally allows.

</details>

---

### Q: How do you inspect and manage container root user execution and drop Linux capabilities for host security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, a container runs as root and keeps a set of permissions it usually doesn't need. If that container is ever compromised, those extra permissions give an attacker a real way to break out onto the host machine. So I avoid using root where I can. On top of that, I remove every permission at startup, and only add back the one thing the app actually needs, like the ability to use a low network port. It's a small change, but it really limits the damage if something goes wrong.

</details>

---
