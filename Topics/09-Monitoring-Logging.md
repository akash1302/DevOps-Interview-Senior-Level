# Senior DevOps Interview Questions: Monitoring & Logging

### Q: What are the practical operational challenges of scaling Prometheus in Kubernetes, and how does Thanos resolve them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Regular Prometheus stores everything on its own local disk, which causes real problems as you grow — keeping old data around gets expensive, and there's no easy way to see metrics from multiple clusters in one place. If that Prometheus pod goes down and loses its disk, that history is just gone. Thanos fixes this by shipping the metric data out to cheap storage like S3, and giving you one single place to query across every cluster at once. So instead of five separate dashboards for five clusters, you get one unified view with full history kept safe.

</details>

---

### Q: How do AWS VPC Flow Logs enable network auditing, security monitoring, and traffic troubleshooting?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Flow Logs record every accepted and rejected connection at the network level — source, destination, ports, all of it. When something can't connect, like an app failing to reach a database, I search the logs for that specific traffic, and if I see it getting rejected, I know right away it's a firewall rule, not an app bug. It's also genuinely useful for security — the same logs help catch things like a port scan or some unexpected connection nobody approved. For huge amounts of log data, I'd use a proper query tool instead of scrolling through logs by hand.

</details>

---

### Q: How do you monitor container resource utilization using Docker built-in tools and metrics?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For quick checks, `docker stats` is the first thing I run — it shows live CPU, memory, and network use for every running container, so I can spot the problem one fast. If a container dies unexpectedly, I check the events log to see if it was actually killed for running out of memory. For real production monitoring though, I don't rely on manually running commands — I turn on the built-in metrics endpoint so our monitoring tool can pull that data automatically and alert on it, instead of someone having to notice a problem by chance.

</details>

---
