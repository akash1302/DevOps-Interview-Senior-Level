# Senior DevOps Interview Questions: Linux

### Q: How do you systematically troubleshoot network connectivity issues at the Linux host and container level?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always check the host machine first, then the container. First I try to ping a known address from the host itself — if that fails, it's a host or firewall issue, nothing to do with the container yet. Then I check DNS separately, since that's a different kind of problem.

If the host is fine, I go inside the container and repeat the same two checks. If a plain IP works but a website name doesn't, it's a DNS setting problem inside the container. If even the IP fails, I check the host's network settings and firewall rules next.

</details>

---

### Q: How do Linux kernel Namespaces, Cgroups, and Capabilities isolate container processes on a host machine?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

A container isn't a full separate machine, it's just a regular process with some limits added by Linux.

Namespaces control what the process can actually see, so it can't see the host's real processes or network. Cgroups control what it can actually use, CPU and memory, so one container can't eat up the whole machine. Capabilities break root's full power down into small pieces, so even a process running as root inside the container is blocked from doing most of what real root access normally allows.

</details>

---

### Q: How do you inspect and manage container root user execution and drop Linux capabilities for host security?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, a container runs as root and keeps a set of permissions it usually doesn't need. If that container is ever compromised, those extra permissions give an attacker a real way to break out onto the host.

So I avoid using root where I can. On top of that, I remove every permission at startup and only add back the one thing the app actually needs, like the ability to use a low network port.

It's a small change, but it really limits the damage if something does go wrong.

</details>

---

### Q: A server's disk is almost full and you don't know why. How do you find out what's actually using the space?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I start at the top and work my way down instead of guessing. First I check which partition is actually full, since a server can have more than one. Then I check folder by folder from the root, drilling into the biggest directory until I find the actual large files.

For example, logs are the most common cause I run into — an app stuck in a retry loop can fill a log file with gigabytes of the same error in a short time. Once I find the cause, I clean it up, but I also fix why it grew that big in the first place, like adding log rotation, so it doesn't just fill up again.

</details>

---

### Q: How do you find out which process is using too much CPU or memory on a Linux server?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I use a live process monitor first, which shows every running process sorted by CPU or memory, updating in real time. That usually points straight at the problem.

Once I know which process it is, I check if it's actually doing real work or if it's stuck waiting on something that isn't responding. If it's a memory issue specifically, I check if usage keeps climbing over time and never comes back down, since that usually means a real leak, not just normal heavy use.

</details>

---

### Q: How do you make sure a script or service runs automatically every time a Linux server restarts?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't rely on someone remembering to start it manually after a reboot. I set it up as a real system service, so the operating system manages starting it, restarting it if it crashes, and stopping it cleanly on shutdown. That also gives me proper logs and a simple way to check if it's actually running.

For something that just needs to run on a schedule, like a nightly cleanup task, I use a scheduled job instead, since that's simpler and doesn't need to run in the background all the time.

</details>

---
