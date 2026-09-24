# Senior DevOps Interview Questions: Linux

### Q: How do you systematically troubleshoot network connectivity issues at the Linux host and container level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I troubleshoot this bottom-up, host first, then container. First I check the host's own external connectivity with `ping 8.8.8.8` — if that fails, it's an upstream or firewall issue that has nothing to do with containers yet. Then `dig google.com` to check whether DNS resolution itself is the problem, separate from raw IP connectivity, since those are two genuinely different failure modes.

If the host's fine, I exec into the container and repeat the same test — ping the raw IP first, then the domain. If IP ping works but domain ping fails, that's a container-level DNS issue, usually something in `/etc/resolv.conf`. If even the IP ping fails from inside the container, I check whether the host actually has IP forwarding enabled with `sysctl net.ipv4.ip_forward` — it needs to read `1` — and then inspect the IPTables NAT table with `iptables -t nat -L -n -v` to see if Docker's forwarding rules are actually present. If a firewall reload wiped those rules out, `systemctl restart docker` forces Docker to recreate its bridge network and repopulate them, which is usually the actual fix in that scenario.

</details>

---

### Q: How do Linux kernel Namespaces, Cgroups, and Capabilities isolate container processes on a host machine?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Containers aren't virtual machines, they're just regular Linux processes wrapped in three kernel features. **Namespaces** control what a process can actually *see* — its own PID tree, its own network interfaces, its own mounts — so a process inside a container has no visibility into the host's real process list or the host's real network. **Cgroups** control what a process can *use* — CPU, memory, disk I/O — so a runaway container can't just consume the entire host and take down every other workload on it with an OOM cascade.

**Capabilities** are the third piece — they break the monolithic root user down into individual permissions, like `CAP_NET_ADMIN` or `CAP_SYS_ADMIN`, so even a process technically running as UID 0 inside the container can be stripped of the specific kernel powers it doesn't actually need. In practice, setting `--memory="512m"` on a container writes that limit directly into the cgroup filesystem, something like `/sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes`, and running `ps aux` inside the container shows your process as PID 1, while the exact same process shows up under its real, much higher PID on the host — that's the namespace doing its job.

</details>

---

### Q: How do you inspect and manage container root user execution and drop Linux capabilities for host security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, containers run as root and hold onto a default set of Linux capabilities, and if that container ever gets compromised, an attacker with something like `CAP_SYS_ADMIN` can mount host filesystems or manipulate network interfaces — that's a real path to breaking out to the host, not just staying contained. So I avoid running as root in the first place wherever possible, and I strip capabilities aggressively as a second layer of defense.

At launch, I pass `--cap-drop=ALL` to remove every default capability, and then add back only the one thing that's actually needed — for a web server that has to bind to port 80 or 443, that's `--cap-add=NET_BIND_SERVICE`, nothing more: `docker run -d --name secure-web --cap-drop=ALL --cap-add=NET_BIND_SERVICE -p 80:80 nginx`. If I need to check what capabilities a running process actually has, `getpcaps <PID>` shows exactly that. This is straight-up least privilege applied at the container layer, and it's a cheap thing to get right that meaningfully shrinks the blast radius if something does go wrong.

</details>

---
