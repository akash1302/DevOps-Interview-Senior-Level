# Senior DevOps Interview Questions: AWS

### Q: How do you design a high-availability, multi-tier VPC architecture in AWS for web applications?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I build the VPC across two or more zones, so if one zone goes down, the app still works. I use three layers of subnets. Public subnets hold the load balancer. Private subnets hold the app servers. A separate database subnet holds RDS, with no internet access at all. The app servers can go out to the internet through a NAT Gateway, but nothing from outside can come in to them directly. I also add Security Groups and NACLs, so even traffic between layers is controlled.

</details>

---

### Q: How do you evaluate and choose between VPC Peering and AWS Transit Gateway for multi-VPC and hybrid connectivity?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

VPC Peering connects two VPCs directly. It's simple and cheap for a few VPCs. But it does not pass through. If A is connected to B, and B is connected to C, A still cannot reach C. So with many VPCs, you need a lot of separate connections, and that gets hard to manage. Transit Gateway solves this. It works like a hub. Every VPC connects to the hub once, and the hub handles all the routing. I use Transit Gateway once we have more than a few VPCs to connect.

</details>

---

### Q: How do AWS Auto Scaling policies handle unpredictable traffic spikes using Dynamic Scaling versus Predictive Scaling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Dynamic Scaling watches live metrics, like CPU, and adds servers after usage goes up. It works, but new servers take a little time to start, so there is a short delay. Predictive Scaling is different. It looks at past traffic patterns and adds servers *before* the expected spike. So if traffic always goes up at 8 AM, Predictive Scaling adds capacity before 8 AM. I use both together — Predictive Scaling for traffic I can plan for, and Dynamic Scaling for anything sudden and unplanned.

</details>

---

### Q: What are EC2 Placement Groups, and how do you select between Cluster, Spread, and Partition placement strategies?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is about where AWS physically places your servers. If I need very fast, low-delay communication between servers, I use **Cluster** placement — it puts them close together on the same hardware. If I have a few critical servers that must never fail at the same time, I use **Spread** placement — it puts each one on separate hardware. If I'm running something like Kafka, where I want failures grouped and isolated, I use **Partition** placement — it splits servers into separate groups.

</details>

---

### Q: How do Amazon S3 VPC Endpoints differ from NAT Gateways when accessing S3 from private EC2 instances?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, a private server reaches S3 through the NAT Gateway, and that costs money for every bit of data sent. Instead, I add an S3 Gateway Endpoint to the subnet. This lets the server reach S3 directly, inside AWS's own network, skipping the NAT Gateway completely. It's cheaper and faster. We used this on a pipeline that moved a lot of data to S3 every day, and it saved real money. I also add a policy to the endpoint, so that subnet can only reach specific buckets, not all of S3.

</details>

---
