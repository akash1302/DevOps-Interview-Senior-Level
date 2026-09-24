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

### Q: What's the difference between RDS Multi-AZ and a Read Replica, and when would you use each?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Multi-AZ is for staying up if something breaks. It keeps a live copy of the database in a second zone, and if the main one fails, AWS switches over to the copy automatically. You never actually query that second copy directly, it just sits ready as a backup. A Read Replica is different — it's for handling more traffic, not for backup. It's a separate copy you can actually send read queries to, so heavy reporting or read-heavy traffic doesn't slow down the main database. In a real setup, I usually use both together — Multi-AZ for safety, and one or more Read Replicas to spread out read traffic.

</details>

---

### Q: How do you design IAM access for a growing company with many AWS accounts and many engineers?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never give engineers a personal AWS access key that sits around forever. Instead, I use AWS Organizations to keep separate accounts for each environment, like dev and prod, so a mistake in dev can't touch prod at all. For access, people log in once through a central identity system, and then assume a role that only lasts a short time, instead of having a permanent key. I also start every role with the least access it needs, and only add more if someone actually asks for it and it makes sense, instead of giving broad access by default and hoping nobody misuses it.

</details>

---

### Q: How would you reduce a company's AWS bill without hurting performance or reliability?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I start by actually looking at what's being paid for, using AWS's own cost tools, instead of guessing. A lot of savings come from easy wins first — deleting unused storage volumes, old snapshots nobody needs, and load balancers nobody's using anymore. Then I look at right-sizing — checking if servers are actually using the CPU and memory they're paying for, and downsizing the ones that aren't. For steady, predictable workloads, I'll buy savings plans or reserved capacity, since that's cheaper than paying full price all the time. I avoid cutting things that affect reliability just to save money — the goal is removing waste, not removing safety.

</details>

---

### Q: Finance tells you the AWS bill jumped 300% compared to last month, and you're asked to find out why. Where do you actually start?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I go straight to Cost Explorer first and group the spend by service, comparing this month against last month. That one view usually tells me immediately which service caused the jump — most of the time it's EC2, data transfer, or S3, so I don't waste time guessing across the whole account.

If it's EC2, I check for things like old dev instances nobody shut down, or an Auto Scaling policy that's too aggressive and keeps way more capacity running than it needs. If it's S3, I check whether there's a flood of tiny objects driving up request costs, or whether the lifecycle rules that should be moving old data to cheaper storage actually failed silently. And if it's data transfer, that's usually the sneaky one — I'd check VPC Flow Logs for large transfers going out to the internet or across regions that shouldn't be happening. Once I find the actual resource, I tag it properly so it's easy to track going forward, and I set up budget alerts and cost anomaly detection, so next time this happens, we get a warning within a day, not a surprise a month later.

</details>

---
