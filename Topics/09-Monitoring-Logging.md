# Senior DevOps Interview Questions: Monitoring & Logging

### Q: What are the practical operational challenges of scaling Prometheus in Kubernetes, and how does Thanos resolve them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Regular Prometheus keeps all its data on its own local disk. That causes problems as things grow — keeping old data around gets expensive, and there's no easy way to see all your clusters in one place. If that Prometheus server goes down and loses its disk, that history is just gone. Thanos fixes this by sending the data out to cheap, safe storage, and giving you one single place to look at metrics from every cluster at once. So instead of five separate dashboards, you get one place that shows everything, with the history kept safe.

</details>

---

### Q: How do AWS VPC Flow Logs enable network auditing, security monitoring, and traffic troubleshooting?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Flow Logs record every connection at the network level — what was allowed, and what was blocked. When something can't connect, like an app that can't reach a database, I search the logs for that traffic. If I see it being blocked, I know right away it's a firewall rule, not a bug in the app. These logs are also useful for security — they can show things like an unexpected connection that nobody approved. For a huge amount of log data, I'd use a proper search tool instead of looking through it by hand.

</details>

---

### Q: How do you monitor container resource utilization using Docker built-in tools and metrics?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For a quick check, I run a command that shows live CPU and memory use for every running container, so I can spot the problem fast. If a container dies suddenly, I check the event log to see if it actually ran out of memory. But for real, ongoing monitoring, I don't rely on checking things by hand. I turn on a metrics feature so our monitoring tool can pull that data automatically and send an alert, instead of someone having to notice a problem by luck.

</details>

---
