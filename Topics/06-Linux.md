# Senior DevOps Interview Questions: Linux

## Q1. How do you systematically troubleshoot network connectivity issues at the Linux host and container level?

### Answer
Troubleshooting Linux host and container network connectivity involves systematically testing network layers from physical interfaces up to DNS resolution. On the host level, inspect interface status (`ip addr`, `ip link`), routing table configurations (`ip route`), and listening sockets (`ss -tulpn`). Test ICMP layer connectivity (`ping 8.8.8.8`) to verify IP connectivity, followed by DNS resolution tests (`dig google.com` or `nslookup`). At the container level, inspect network namespaces, bridge devices (`docker network inspect bridge`), and IPTables NAT forwarding rules (`iptables -L -n -v`).

### Interview Answer
"I troubleshoot systematically bottom-up. First, I test host external connectivity using `ping 8.8.8.8` to rule out upstream firewall issues, followed by `dig google.com` to check DNS resolution in `/etc/resolv.conf`. Next, I step into the container using `docker exec` or `busybox` debug containers to ping the host bridge gateway. If host connectivity works but the container fails, I check IPTables IP forwarding (`net.ipv4.ip_forward = 1`) and verify that Docker bridge subnet routing rules aren't being blocked by host firewalls like UFW or firewalld."

### Practical Example
Troubleshooting container internet loss:
1. Check host internet: `ping -c 2 8.8.8.8` (Success)
2. Check host IP forwarding: `sysctl net.ipv4.ip_forward` (Ensure value is `1`)
3. Inspect container IP and Gateway: `docker exec -it app_container ip route`
4. Inspect host IPTables forwarding rules: `iptables -t nat -L -n -v`
5. Restart Docker daemon to repair broken bridge interface bindings: `systemctl restart docker`

### Follow-up Questions
* What is the role of `net.ipv4.ip_forward` in Linux routing between network interfaces?
* How do you inspect listening ports and established sockets using `ss` versus `netstat`?
* How does `/etc/resolv.conf` handle DNS search domains inside Kubernetes pods?

### Key Points
* Isolate host-level network failure before diagnosing container-level network issues.
* Verify Linux kernel packet forwarding (`net.ipv4.ip_forward = 1`).
* Check IPTables NAT rules and bridge interface health when containers lose outbound access.

---

## Q2. How do Linux kernel Namespaces, Cgroups, and Capabilities isolate container processes on a host machine?

### Answer
Linux containers are not full virtual machines; they are isolated Linux processes governed by three kernel primitives: Namespaces, Control Groups (Cgroups), and Capabilities. **Namespaces** isolate what a process can *see*—providing virtualized views of process IDs (`pid`), networking (`net`), mount points (`mnt`), hostnames (`uts`), and user IDs (`user`). **Cgroups** limit and measure what a process can *use*—imposing resource caps on CPU, RAM, Disk I/O, and Network. **Capabilities** break down root privileges into distinct fine-grained units (e.g., `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`), allowing dropping unneeded root powers.

### Interview Answer
"Containers are fundamentally just Linux processes isolated by kernel features. Namespaces provide visibility isolation so a container process only sees its own PID tree, network interfaces, and mounts. Cgroups enforce resource quotas, preventing a buggy container from consuming 100% of the host CPU or memory and triggering OOM kills across other processes. Finally, Linux Capabilities decompose the monolithic root user into granular permissions, letting us restrict kernel calls even if the process runs as UID 0."

### Practical Example
* **Cgroups in action**: Setting Docker memory limits `--memory="512m"` writes restrictions directly to `/sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes`.
* **Namespaces in action**: Running `ps aux` inside a container shows PID 1, while running `ps aux` on the host machine shows the container process running under its real host PID.

### Follow-up Questions
* What happens when a container exceeds its Cgroup memory limit versus its CPU limit?
* How does Cgroups v2 improve upon resource management compared to Cgroups v1?
* How do `pid` namespaces enable sharing process trees between pause containers and application containers?

### Key Points
* Namespaces isolate process visibility (`pid`, `net`, `mnt`, `uts`, `user`).
* Cgroups restrict and account for system resource usage (CPU, RAM, I/O).
* Capabilities divide root privileges into granular operational permissions.

---

## Q3. How do you inspect and manage container root user execution and drop Linux capabilities for host security?

### Answer
By default, processes inside Docker containers run with elevated root privileges and retain a default set of Linux capabilities. If a container process is compromised, an attacker retaining capabilities like `CAP_SYS_ADMIN` or `CAP_NET_ADMIN` can manipulate kernel network interfaces, mount host file systems, or break out to the host. To secure host systems, containers should run under non-root UIDs, use read-only root filesystems, and drop default kernel capabilities via CLI flags (`--cap-drop=ALL`) or container security manifests.

### Interview Answer
"To prevent privilege escalation and container breakout attacks, I strictly avoid running containerized processes as root. In Dockerfiles, I explicitly create non-root service accounts. At container launch, I pass `--cap-drop=ALL` to strip away all kernel capabilities, then selectively add back only what's explicitly needed, such as `--cap-add=NET_BIND_SERVICE` for web servers binding to port 80/443. This adheres to the security principle of least privilege."

### Practical Example
Inspecting Linux capabilities of a process:
`getpcaps <PID>`
Executing a container dropping all capabilities except low-port network binding:
`docker run -d --name secure-web --cap-drop=ALL --cap-add=NET_BIND_SERVICE -p 80:80 nginx`

### Follow-up Questions
* What security risks arise if a container mounts `/var/run/docker.sock` as root?
* How do AppArmor and SELinux profiles add an extra layer of mandatory access control (MAC) to containers?
* What is `readOnlyRootFilesystem` in Kubernetes pod security contexts and why is it recommended?

### Key Points
* Default container root processes retain dangerous Linux kernel capabilities.
* Always drop all capabilities (`--cap-drop=ALL`) and selectively re-add mandatory ones.
* Running non-root users combined with capability stripping prevents container breakout attacks.


---
