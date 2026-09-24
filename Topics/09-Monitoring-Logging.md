# Senior DevOps Interview Questions: Monitoring & Logging

### Q: What are the practical operational challenges of scaling Prometheus in Kubernetes, and how does Thanos resolve them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Standalone Prometheus stores everything on local disk, which creates real problems at scale — long-term retention gets expensive fast, there's no built-in way to get a unified view across multiple clusters, and if that Prometheus pod goes down and loses its disk, you've genuinely lost your metric history, not just visibility for a few minutes.

To fix that, I deploy Thanos alongside it. A Thanos Sidecar runs next to each Prometheus instance and ships historical metric blocks out to an S3 bucket for cheap, durable long-term storage. Then Thanos Querier sits on top and gives you one unified query interface across every cluster — so Grafana just points at the Querier as a single data source, and it transparently pulls live data from the Sidecars and historical data from the Store Gateway reading off S3. In practice that means Cluster A and Cluster B both run Prometheus plus a Sidecar uploading two-hour blocks to a shared bucket, and the Querier stitches it all together with deduplication, so you get true multi-cluster HA visibility instead of a pile of disconnected dashboards.

</details>

---

### Q: How do AWS VPC Flow Logs enable network auditing, security monitoring, and traffic troubleshooting?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

VPC Flow Logs capture every accept and reject decision happening at the network interface level — source IP, destination IP, ports, protocol, byte counts — and that's genuinely the fastest way to answer "why can't this app reach that database" when Security Groups or NACLs are the suspects. If I search the logs for the source and destination IP and see `REJECT` entries on port 5432, I know immediately it's a firewall rule dropping the packet, not an application bug.

I usually query this straight from CloudWatch Logs Insights, something like filtering `action = "REJECT"` and grouping by source IP and destination port to spot patterns fast — that same query is also how you catch something like a port scan or unauthorized outbound connection attempt, so it pulls double duty for security auditing, not just connectivity troubleshooting. For anything at real scale, petabytes of flow log data sitting in S3, I'd reach for Athena instead of CloudWatch, since that's built for querying that volume of data efficiently. And it's worth knowing this capture happens out-of-band at the network layer — it doesn't add latency or overhead to the actual instance traffic.

</details>

---

### Q: How do you monitor container resource utilization using Docker built-in tools and metrics?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For quick, live debugging on a host, `docker stats` is the first thing I run — it streams real-time CPU percentage, memory usage against the limit, and network and disk I/O for every running container, so I can immediately spot which one is actually the problem. If a container unexpectedly dies, `docker events --filter 'event=oom'` tells me right away whether it was actually killed by the Linux OOM killer, rather than crashing on its own.

Those CLI tools are great for one-off debugging, but for real production monitoring, I enable the Docker daemon's Prometheus-formatted metrics endpoint in `daemon.json`, something like setting `"metrics-addr": "127.0.0.1:9323"`, so the metrics collector can scrape it automatically instead of someone running `docker stats` by hand. In Kubernetes specifically, that same job is handled by cAdvisor, which collects per-container metrics directly on each node and feeds them into the same Prometheus pipeline — so the workflow stays consistent whether I'm debugging a single Docker host or a whole cluster.

</details>

---
