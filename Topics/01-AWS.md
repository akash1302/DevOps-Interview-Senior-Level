# Senior DevOps Interview Questions: AWS

### Q: How do you design a high-availability, multi-tier VPC architecture in AWS for web applications?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I always build the VPC across at least two Availability Zones, so one zone going down doesn't take the app with it. I split it into three layers — public subnets for the load balancer and NAT Gateway, private subnets for the app servers, and separate database subnets for RDS with no internet access at all. The app servers can go out to the internet through the NAT Gateway when they need to, but nothing from outside can ever reach them directly. I also lock things down with Security Groups on the instances and NACLs at the subnet level, so even traffic between layers is restricted to only what's needed.

</details>

---

### Q: How do you evaluate and choose between VPC Peering and AWS Transit Gateway for multi-VPC and hybrid connectivity?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

VPC Peering works fine for a small number of VPCs, it's simple and cheap. The big catch is it doesn't chain — if A is peered to B and B is peered to C, A still can't reach C. So once you're past a handful of VPCs, managing all those individual connections gets messy fast. That's why for anything bigger, like dozens of accounts each with their own VPCs, I use Transit Gateway instead. It works like a hub — every VPC connects to it once, and it handles the routing centrally, so you're not managing hundreds of point-to-point links by hand.

</details>

---

### Q: How do AWS Auto Scaling policies handle unpredictable traffic spikes using Dynamic Scaling versus Predictive Scaling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Dynamic Scaling reacts after the fact — it watches something like CPU usage and adds instances once a threshold is crossed. The problem is new instances take time to start up, so there's always a short lag. For traffic that follows a known pattern, like a daily rush every morning, I pair that with Predictive Scaling, which looks at past traffic and adds capacity ahead of time, before the spike even hits. So the predictable stuff gets handled in advance, and Dynamic Scaling is still there as a backup for anything unexpected, like a surprise sale.

</details>

---

### Q: What are EC2 Placement Groups, and how do you select between Cluster, Spread, and Partition placement strategies?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I pick this based on what actually matters for the workload. If I need really low latency between instances, like for big data or high-performance jobs, I use **Cluster** placement, which packs instances close together on the same hardware. If I have a small set of critical nodes that must never fail together, like a control plane, I use **Spread** placement, which puts each one on separate physical hardware. For something like Kafka or Cassandra, where I want failures isolated across groups, I use **Partition** placement, which splits instances into separate racks by group.

</details>

---

### Q: How do Amazon S3 VPC Endpoints differ from NAT Gateways when accessing S3 from private EC2 instances?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, private instances reach S3 through the NAT Gateway, and that costs money for every gigabyte that passes through it. Instead, I attach an S3 Gateway Endpoint straight to the private subnet's route table. That keeps all the S3 traffic inside AWS's own network, skips the NAT Gateway completely, and it's actually faster too. We did this on a pipeline moving terabytes of logs to S3 every day, and it saved real money on NAT charges. I also lock it down further with an endpoint policy, so that subnet can only reach specific buckets, not all of S3.

</details>

---
