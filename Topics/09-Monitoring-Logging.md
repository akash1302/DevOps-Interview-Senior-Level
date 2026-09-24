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

### Q: What's the difference between an SLI, an SLO, and an SLA, and why does a platform team actually need all three?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

An SLI is just a real number you measure, like how many requests succeeded in the last hour. An SLO is the internal target you set for that number, like "99.9% of requests should succeed" — that's the goal the team actually works toward. An SLA is the promise made to the customer, usually with a real penalty attached if it's broken. I always set the SLO a bit stricter than the SLA, so the team gets an early warning and can react before we actually break the promise made to the customer, not after.

</details>

---

### Q: How do you design alerting so the on-call engineer doesn't get woken up for things that don't actually matter?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I only page a real person for things that need action right now, and would actually hurt the business if ignored — like the whole site being down. Anything that's just informational, or can wait until morning, goes to a dashboard or a non-urgent channel instead, never a phone alert. I also set alerts on trends, not just a single bad reading, since one slow request doesn't mean anything, but the error rate climbing steadily for ten minutes does. If the team starts ignoring alerts because there are too many false ones, that's a sign the alerting itself needs fixing, not a sign the team needs to try harder.

</details>

---

### Q: How do you set up centralized logging for an application running across many servers or containers?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never leave logs sitting only on the individual server that created them, since that server might get replaced or deleted at any time, and the logs would just disappear with it. Instead, every server and container ships its logs out to one central place in real time. I also make sure logs are structured, not just plain free text, so I can actually search and filter them properly — like finding every log line for one specific request, across every service it touched, instead of manually reading through raw text files one at a time.

</details>

---
