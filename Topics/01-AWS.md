# Senior DevOps Interview Questions: AWS

### Q: How do you design a high-availability, multi-tier VPC architecture in AWS for web applications?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In production, I set up a custom VPC spanning at least two Availability Zones for redundancy, and I split it into three subnet tiers. Public subnets hold the Application Load Balancer and the NAT Gateways. Private subnets hold the application servers, EC2 or EKS nodes, and they route any outbound traffic through the NAT Gateway instead of getting a public IP. Database subnets sit one layer deeper, hosting RDS with no route to the internet at all.

So for a real deployment, that'd look like the ALB sitting in `10.0.1.0/24` and `10.0.2.0/24`, app servers in `10.0.10.0/24` and `10.0.20.0/24`, and a Multi-AZ RDS instance in `10.0.100.0/24` and `10.0.200.0/24`. The app servers can reach out to fetch API updates through the NAT Gateway, but nothing from the internet can ever reach them directly. On top of that, I restrict traffic with Security Groups at the instance level and NACLs at the subnet boundary, so even tier-to-tier communication is locked down to only what's actually needed.

</details>

---

### Q: How do you evaluate and choose between VPC Peering and AWS Transit Gateway for multi-VPC and hybrid connectivity?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

VPC Peering is great when you've only got a handful of VPCs — it's direct, there's no hourly gateway charge, and latency is low. The big catch is it doesn't support transitive routing, so if you've got N VPCs that all need to talk to each other, you end up needing a full mesh of N times N-minus-one over two peering connections, and that gets unmanageable fast.

That's exactly the wall I hit once we scaled past a dozen VPCs across multiple accounts with an on-prem Direct Connect requirement. We moved to Transit Gateway, which acts as a central hub — every VPC just connects to the hub once, and routing is managed centrally instead of pair by pair. A real example: an enterprise with fifty AWS accounts, each with dev, QA, and prod VPCs, connects everything to a central Shared Services VPC and the on-prem data center through Transit Gateway, instead of standing up thousands of individual peering connections. We also use AWS Resource Access Manager to share that Transit Gateway across accounts cleanly.

</details>

---

### Q: How do AWS Auto Scaling policies handle unpredictable traffic spikes using Dynamic Scaling versus Predictive Scaling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Dynamic Scaling reacts to real-time CloudWatch metrics — say, scaling out once CPU crosses 80%. It works, but it's reactive, and launching and bootstrapping new instances takes time, so there's always a lag between the spike hitting and capacity actually catching up.

For traffic that follows a predictable pattern, like a surge every morning at 8 AM, I pair that with Predictive Scaling, which uses machine learning on historical traffic data to pre-warm the Auto Scaling Group before the spike even hits. So on an e-commerce app, Predictive Scaling handles the expected 8 AM rush by scaling ahead of time, while Dynamic Scaling is still there to catch anything unexpected, like a flash sale nobody scheduled. Running both together gets you low latency during predictable peaks without over-provisioning and burning money the rest of the day.

</details>

---

### Q: What are EC2 Placement Groups, and how do you select between Cluster, Spread, and Partition placement strategies?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I pick a placement group based on what the workload actually needs from the underlying hardware. If I've got a high-performance computing job or tightly-coupled nodes needing single-digit-millisecond latency between them, I use **Cluster** placement, which packs instances close together in one AZ for maximum throughput. If instead I've got something like a small set of critical control plane nodes that absolutely can't all fail together on the same rack, I use **Spread** placement, which puts each instance on distinct underlying hardware.

For something like a distributed database — Cassandra, Kafka — I use **Partition** placement, which groups instances into logical partitions spread across separate racks, so a single rack failure only takes out one partition's worth of nodes, not the whole cluster. In practice, that might look like an HPC node cluster running in Cluster placement to hit 10Gbps-plus throughput between nodes, while a 3-node Kubernetes control plane runs in Spread placement so no two control plane nodes ever share the same physical host.

</details>

---

### Q: How do Amazon S3 VPC Endpoints differ from NAT Gateways when accessing S3 from private EC2 instances?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, if a private instance needs to reach S3, that traffic goes out through the NAT Gateway, which means it's routed over the internet path and racks up NAT data processing charges. Instead, I attach an S3 Gateway Endpoint directly to the private subnet's route table, which keeps all that traffic entirely inside the AWS network — no NAT Gateway involved at all, no hourly or per-GB charge for it, and it's actually faster.

We had a data pipeline reading and writing terabytes of logs to S3 daily from private EC2 instances, and just adding the Gateway Endpoint eliminated thousands of dollars a month in NAT data transfer fees. On top of the cost savings, I can attach an Endpoint Policy to lock down exactly which buckets that VPC is allowed to reach, which is a nice security tightening on top of the cost win. One thing worth knowing — this is different from an S3 Interface Endpoint using PrivateLink, which is a separate option with its own use case, but the Gateway Endpoint is what you want for standard private-subnet-to-S3 traffic.

</details>

---
