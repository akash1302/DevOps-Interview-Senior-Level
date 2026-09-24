# Senior DevOps Interview Questions: Monitoring & Logging

## Q1. What are the practical operational challenges of scaling Prometheus in Kubernetes, and how does Thanos resolve them?

### Answer
Prometheus is a powerful metrics collection system, but running a standalone instance presents architectural limitations: single-node storage bottlenecks, lack of long-term historical metric retention, and absence of built-in global multi-cluster monitoring views. If Prometheus restarts or loses its local disk, metric data is lost. **Thanos** resolves these challenges by transforming Prometheus into a highly available, distributed monitoring system. It uses a Sidecar component to ship metric blocks to cloud object storage (S3), uses Thanos Querier to provide a unified global query view across multiple clusters, and handles long-term storage and deduplication.

### Interview Answer
"Standalone Prometheus stores metrics locally on TSDB disk blocks, making long-term storage expensive and multi-cluster monitoring difficult. When Prometheus goes down, you lose visibility. To solve this, I deploy Thanos. A Thanos Sidecar runs alongside Prometheus, shipping historical metric blocks to an S3 bucket for cheap long-term storage. Thanos Querier aggregates metrics across all EKS clusters into a single Grafana dashboard, providing global HA deduplication and long-term retention."

### Practical Example
Thanos Multi-Cluster Monitoring Architecture:
1. **Cluster A & B**: Run Prometheus + Thanos Sidecar. Sidecar uploads 2-hour TSDB metric blocks to a shared AWS S3 bucket.
2. **Thanos Querier**: Queries active metrics from Thanos Sidecars and historical metrics from Thanos Store Gateway bound to S3.
3. **Grafana**: Points to Thanos Querier as a single unified Prometheus data source.

### Follow-up Questions
* How does sticky session load balancing apply when attempting native Prometheus HA without Thanos?
* What is the role of Thanos Compactor in downsampling historical metrics in S3?
* How does Prometheus scrape target endpoints versus push-based metric collection models?

### Key Points
* Standalone Prometheus lacks native long-term object storage and multi-cluster aggregation.
* Thanos Sidecar streams historical metric blocks directly to cloud object storage (S3).
* Thanos Querier provides global query views and deduplicates metrics across HA pairs.

---

## Q2. How do AWS VPC Flow Logs enable network auditing, security monitoring, and traffic troubleshooting?

### Answer
AWS VPC Flow Logs capture detailed IP traffic flow data passing through network interfaces (ENIs) in a VPC, subnet, or individual instance. Flow logs record accepted (`ACCEPT`) and rejected (`REJECT`) packet traffic along with source IP, destination IP, source port, destination port, protocol, byte count, and packet count. Flow log data is streamed to CloudWatch Logs or Amazon S3 for centralized analysis, enabling security auditing, malicious traffic detection, and network connectivity troubleshooting.

### Interview Answer
"VPC Flow Logs give complete visibility into network traffic moving through VPC interfaces. When troubleshooting why an application can't connect to a database or external API, I search CloudWatch Logs for the source and destination IP. If I see `REJECT` entries on port 5432, I immediately know a Security Group or NACL is dropping the packets. It's also vital for security auditing, allowing us to detect port scans or unauthorized outbound connection attempts."

### Practical Example
CloudWatch Logs Insights query analyzing rejected traffic:
```sql
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| stats count(*) by srcAddr, dstPort
| sort count(*) desc
| limit 20
```

### Follow-up Questions
* What is the difference between enabling Flow Logs at the VPC level versus the Subnet level?
* How do you analyze petabyte-scale VPC Flow Logs stored in S3 using Amazon Athena?
* Does capturing VPC Flow Logs introduce performance overhead or packet latency on EC2 instances?

### Key Points
* Flow Logs record `ACCEPT` and `REJECT` traffic metadata across VPC network interfaces.
* Crucial for diagnosing firewall drops (Security Group / NACL misconfigurations).
* Streams traffic data to CloudWatch Logs or S3 without impacting instance performance.

---

## Q3. How do you monitor container resource utilization using Docker built-in tools and metrics?

### Answer
Docker provides built-in Command Line Interface (CLI) utilities and API endpoints to monitor real-time container resource consumption. `docker stats` streams a live overview of CPU usage percentage, memory consumption and limits, network I/O, and block disk I/O across running containers. `docker events` streams real-time system events (container creation, start, die, OOM kill). For automated production monitoring, the Docker daemon exposes a Prometheus-formatted metrics endpoint (`/metrics`) that metrics collectors scrape directly.

### Interview Answer
"For real-time CLI debugging on a host, I run `docker stats` to immediately identify which container is consuming high CPU or hitting memory limits. If a container unexpectedly dies, I check `docker events` to see if an Out-Of-Memory (OOM) kill event occurred. In production environments, I configure `daemon.json` to expose Prometheus metrics, allowing our monitoring stack to automatically scrape container engine metrics."

### Practical Example
1. Streaming live resource metrics: `docker stats --format "table {{.Name}}	{{.CPUPerc}}	{{.MemUsage}}"`
2. Monitoring runtime engine events: `docker events --filter 'event=oom'`
3. Enabling Prometheus metrics in `/etc/docker/daemon.json`:
```json
{
  "metrics-addr": "127.0.0.1:9323",
  "experimental": true
}
```

### Follow-up Questions
* What exit code does Docker return when a container is terminated by the Linux OOM Killer?
* How does cAdvisor (Container Advisor) collect container metrics inside Kubernetes nodes?
* What is the difference between container memory limits and memory reservation flags?

### Key Points
* `docker stats` provides live streaming CPU, RAM, and I/O utilization metrics.
* `docker events` captures real-time lifecycle events including OOM container kills.
* Expose Docker daemon Prometheus metrics endpoints for production monitoring integration.


---
