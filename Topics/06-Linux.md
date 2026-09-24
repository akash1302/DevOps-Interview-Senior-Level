# Senior DevOps Interview Questions: Linux

### Q: How do you systematically troubleshoot network connectivity issues at the Linux host and container level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always check the host first, then the container, in that order. First I ping a known IP, like `8.8.8.8`, from the host itself — if that fails, it's a host or firewall problem, nothing to do with containers yet. Then I check DNS separately with something like `dig google.com`, since that's a different kind of problem than raw connectivity. If the host is fine, I go into the container and repeat the same two checks. If the IP ping works but the domain doesn't, it's a DNS setting issue inside the container. If even the IP ping fails, I check if the host has packet forwarding turned on, and then check the firewall rules for anything blocking it.

</details>

---

### Q: How do Linux kernel Namespaces, Cgroups, and Capabilities isolate container processes on a host machine?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Containers aren't full virtual machines, they're just regular processes with some extra limits from the Linux kernel. **Namespaces** control what a process can actually see — its own process list, its own network, so it can't see the host's real processes. **Cgroups** control what it can actually use — CPU, memory — so one container can't eat up the whole machine and take everything else down with it. **Capabilities** break root's full power down into smaller pieces, so even a process running as root inside the container can be stripped of most of what real root access would normally allow.

</details>

---

### Q: How do you inspect and manage container root user execution and drop Linux capabilities for host security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, containers run as root and keep a set of Linux capabilities they usually don't need. If that container ever gets compromised, those extra capabilities give an attacker a real path to break out to the host. So I avoid root where I can, and on top of that, I strip every capability at startup and only add back the one thing that's actually needed — like allowing a web server to bind to a low port, nothing more. It's a small change that meaningfully limits the damage if something does go wrong.

</details>

---
