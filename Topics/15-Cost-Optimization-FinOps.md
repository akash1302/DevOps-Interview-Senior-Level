# Senior DevOps Interview Questions: Cost Optimization / FinOps

### Q: Leadership wants to cut cloud spend by 30% this quarter without hurting reliability. How do you approach it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't start by cutting — I start by seeing where the money actually goes, ranked by dollar amount, not percentage change. A service that doubled but costs $50 a month matters far less than one that only grew 10% but is half the bill.

Then I split findings into two buckets. Waste — unattached EBS volumes, idle load balancers, dev environments left running over weekends, oversized instances nobody ever right-sized — doesn't touch reliability at all, it's just cleanup, and it's usually good for 10-15% on its own.

Real capacity is where I get careful. Moving steady, predictable workloads to Savings Plans is close to free money since it doesn't change how anything runs. Spot for fault-tolerant workloads cuts more, but only where the app can actually handle interruption gracefully — I don't force that everywhere just to hit a number.

*Rank spend by dollar amount, not percentage → clean up pure waste first, safe 10-15% → move steady workloads to Savings Plans, near-zero risk → Spot only for workloads that can tolerate interruption → track weekly, not just at quarter-end.*

I'd rather report 22% achieved safely than hit 30% and cause an outage that costs more than the savings.

</details>

---

### Q: You find several idle or forgotten resources that have been costing money for months. How do you build a process to catch this automatically going forward?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Finding it once is easy — the fix is not being the one who has to go looking again in six months.

I clean up the immediate waste first — unattached volumes, unused Elastic IPs, idle load balancers, old snapshots. Trusted Advisor and Cost Explorer's rightsizing recommendations catch most of this directly, so I don't reinvent that detection.

For prevention, tagging has to be enforced through the IaC modules teams actually use, not a policy doc — every resource gets an owner and environment tag automatically, and anything untagged is itself a signal something bypassed the normal process. Then a scheduled weekly check flags untagged resources, near-zero CPU utilization over two weeks, and non-prod resources running outside business hours — routed to a Slack channel the team actually reads, not a dashboard nobody opens.

For dev and staging specifically, I usually just automate a shutdown schedule outright — there's rarely a reason a dev box needs to run at 2 AM on a Saturday.

</details>

---

### Q: Data transfer costs are unexpectedly high this month. How do you diagnose and reduce them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Data transfer costs hide well — they're spread across small charges instead of one obvious resource, so I go looking for them specifically in Cost Explorer, filtered to data transfer.

First I check if it's cross-AZ, cross-region, or internet egress, since each has a different fix. Cross-AZ between chatty services — an app and a database in different AZs making thousands of small calls — adds up fast and is invisible until you filter specifically for it.

For cross-region, I check VPC Flow Logs for what's actually generating the traffic — I've found a service calling another service's public endpoint across regions purely because private connectivity between them was never set up. For internet egress, I check if a CDN could absorb it instead — putting CloudFront in front of static or cacheable content usually costs less and improves latency too.

*Filter Cost Explorer to data transfer → identify cross-AZ, cross-region, or egress → cross-AZ, check if chatty services can be co-located → cross-region, check Flow Logs for unnecessary cross-region calls → egress, check if a CDN can absorb it.*

I also check for missing VPC endpoints on S3/DynamoDB traffic from private subnets — going through a NAT Gateway instead of a direct endpoint is a common, easy-to-miss charge that has nothing to do with real internet transfer.

</details>

---

### Q: How do you decide whether moving workloads to Spot Instances or Graviton (ARM) is actually worth the operational risk?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I evaluate these separately, since they carry very different risk even though both get pitched as "cost optimization."

For Spot, the real question is whether the workload can handle an instance disappearing with two minutes of warning. Stateless services behind a load balancer are a clean fit — losing one out of twenty just means traffic routes around it. Anything stateful, or a long job that can't checkpoint, is a bad fit unless I'm willing to build real interruption handling for it.

For Graviton, the risk is almost entirely compatibility — a different CPU architecture, so compiled dependencies, native extensions, and non-multi-arch base images need actual testing, not an assumption. I've had a service pass CI cleanly and still hit a floating-point rounding difference between architectures that only showed up at production scale, so I test under real load in staging before trusting it.

*Spot — confirm stateless and horizontally scalable before considering it at all. Graviton — confirm dependency compatibility, test under real production-like load, not just CI → once confirmed, it's usually a safe, sizable discount with zero interruption risk.*

I'd never move a stateful, single-instance-critical workload to Spot just for the discount — that's a decision that looks great on a cost report and terrible on an incident postmortem.

</details>

---

### Q: How do you show engineering teams the actual cost of the services they own, so cost becomes part of their normal decision-making?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If cost only lives on a dashboard the platform team checks, engineering teams have no reason to think about it — the goal is getting it in front of the people actually making the decisions that drive it.

Tagging is the foundation, enforced through the same Terraform modules teams already use to provision things, not a policy doc. Untagged spend is invisible spend, and invisible spend never gets optimized.

Once tagging's solid, I build cost allocation reports by team and deliver them where the team already looks — I've had much better results posting a weekly "your service's cost, and what changed" straight into a team's own Slack channel than pointing them at a company dashboard. I also frame it in terms they already care about, like cost per request, rather than a raw dollar figure that's hard to reason about on its own.

*Enforce tagging through provisioning modules → build cost-by-team reports → deliver where the team already looks, not a separate dashboard → frame it as cost per request/customer, not a raw total → surface cost at PR/plan time where possible, when it's still cheap to change course.*

Where I can, I also surface cost estimates on a Terraform plan directly in the PR, since that's the moment an engineer can still change course cheaply, not a month later after it's already running.

</details>

---
