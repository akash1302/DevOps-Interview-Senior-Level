# Senior DevOps Interview Questions: AWS

### Q: How do you design a high-availability, multi-tier VPC architecture in AWS for web applications?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

In my experience, I build the VPC across at least two availability zones, so if one zone goes down, the app still works.

I use three layers of subnets. Public subnets hold the load balancer. Private subnets hold the app servers. A separate database subnet holds RDS, with no direct internet access at all.

The app servers can reach out to the internet through a NAT Gateway when they need to, but nothing from outside can reach them directly.

For example, on a project I worked on, this setup meant when one AZ had an issue, traffic just shifted to the healthy AZ and nobody noticed. I also add Security Groups and NACLs so even traffic between layers is controlled, not just traffic coming from outside.

</details>

---

### Q: How do you evaluate and choose between VPC Peering and AWS Transit Gateway for multi-VPC and hybrid connectivity?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

VPC Peering connects two VPCs directly. It's simple and cheap when you only have a few VPCs.

The catch is it doesn't pass through — if A is peered to B, and B is peered to C, A still can't reach C. So once you have many VPCs, you end up managing a lot of separate connections by hand.

What I normally do once it gets past a handful of VPCs is move to Transit Gateway. It works like a hub — every VPC connects to it once, and the hub handles the routing centrally, so I'm not managing dozens of point-to-point connections myself.

</details>

---

### Q: How do AWS Auto Scaling policies handle unpredictable traffic spikes using Dynamic Scaling versus Predictive Scaling?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Dynamic Scaling watches a live metric, usually CPU, and adds servers after usage actually goes up. It works, but new servers take a little time to start, so there's always a short delay before capacity catches up.

Predictive Scaling is different — it looks at past traffic patterns and adds servers ahead of an expected spike, before it even hits.

What I normally do is run both together. Predictive Scaling handles traffic I can plan for, like a daily peak, and Dynamic Scaling is still there to catch anything sudden and unplanned, like a flash sale.

</details>

---

### Q: What are EC2 Placement Groups, and how do you select between Cluster, Spread, and Partition placement strategies?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This is about where AWS physically places your servers, and I pick based on what the workload actually needs.

If I need very fast, low-delay communication between servers, like for a big data job, I use Cluster placement — it packs them close together on the same hardware.

If I have a few critical servers that must never fail together, like a small control plane, I use Spread placement — each one goes on separate hardware.

For example, for something like Kafka, where I want failures grouped and isolated rather than spread randomly, I'd use Partition placement instead.

</details>

---

### Q: How do Amazon S3 VPC Endpoints differ from NAT Gateways when accessing S3 from private EC2 instances?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

By default, a private server reaches S3 through the NAT Gateway, and that costs money for every bit of data that passes through it.

What I normally do instead is add an S3 Gateway Endpoint to the subnet. That lets the server reach S3 directly inside AWS's own network, skipping the NAT Gateway completely.

For example, we had a pipeline moving a lot of data to S3 every day, and just adding the endpoint saved real money on NAT charges, and it was faster too. I also attach a policy to the endpoint so that subnet can only reach specific buckets, not all of S3.

</details>

---

### Q: What's the difference between RDS Multi-AZ and a Read Replica, and when would you use each?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Multi-AZ is for staying available if something breaks. It keeps a live standby copy of the database in a second zone, and if the main one fails, AWS switches over automatically. You never query that standby directly, it just sits ready.

A Read Replica is for a different problem — handling more traffic, not backup. It's a separate copy you can actually send read queries to, so heavy reporting doesn't slow down the main database.

In a real setup, I usually run both together — Multi-AZ for safety, and one or more Read Replicas to spread out read traffic.

</details>

---

### Q: How do you design IAM access for a growing company with many AWS accounts and many engineers?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I never give engineers a personal AWS access key that just sits around forever.

What I normally do is use AWS Organizations to keep separate accounts per environment, like dev and prod, so a mistake in dev can't touch prod at all.

For access, people log in once through the company's identity system and assume a role that only lasts a short time, instead of having a permanent key. I also start every role with the least access it actually needs, and only add more if someone specifically asks for it and it makes sense.

</details>

---

### Q: How would you reduce a company's AWS bill without hurting performance or reliability?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I check what's actually being paid for, using AWS's own cost tools, instead of guessing.

A lot of savings come from easy wins first — unused storage volumes, old snapshots nobody needs, load balancers nobody's using anymore. Then I look at right-sizing, checking if servers are actually using the CPU and memory they're paying for.

For steady, predictable workloads, I'll move to savings plans or reserved capacity, since that's cheaper than paying full price all the time. I'm careful not to cut anything that affects reliability just to save money — the goal is removing waste, not removing safety.

</details>

---

### Q: Finance tells you the AWS bill jumped 300% compared to last month, and you're asked to find out why. Where do you actually start?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I go to Cost Explorer and group the spend by service, comparing this month against last month. That usually tells me right away which service caused the jump, so I'm not guessing across the whole account.

If it's EC2, I check for things like old dev instances nobody shut down, or an autoscaling setting that's too aggressive. If it's S3, I check for a flood of small objects driving up request costs, or a lifecycle rule that quietly failed. If it's data transfer, I check for traffic going out to the internet or across regions that shouldn't be happening.

Once I find the actual resource, I tag it properly and set up a budget alert, so next time it's caught within a day, not as a surprise a month later.

</details>

---
