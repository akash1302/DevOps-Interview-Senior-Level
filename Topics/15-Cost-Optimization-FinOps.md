# Senior DevOps Interview Questions: Cost Optimization / FinOps

### Q: Leadership wants to cut cloud spend by 30% this quarter without hurting reliability. How do you approach it?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't start by cutting things — I start by seeing where the money actually goes, ranked by dollar amount, not percentage change. A service that doubled but only costs $50 a month matters far less than one that grew 10% but is half the bill.

Then I split findings into two buckets. Waste — unattached volumes, idle load balancers, dev environments left running over weekends, oversized instances — doesn't touch reliability at all, it's just cleanup, and it's usually good for 10 to 15% on its own.

Real capacity is where I get careful. Moving steady, predictable workloads to savings plans is close to free money since it doesn't change how anything runs. Spot instances cut more, but only for workloads that can actually handle interruption gracefully, so I don't force that everywhere just to hit a number. I'd rather report 22% achieved safely than hit 30% and cause an outage that costs more than the savings.

</details>

---

### Q: You find several idle or forgotten resources that have been costing money for months. How do you build a process to catch this automatically going forward?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Finding it once is easy, the real fix is not being the one who has to go looking again in six months.

First I clean up the immediate waste — unattached volumes, unused Elastic IPs, idle load balancers, old snapshots. AWS's own cost tools catch most of this directly, so I don't reinvent that detection.

For prevention, tagging has to be enforced through the infrastructure code teams actually use, not a policy document — every resource gets an owner and environment tag automatically. Then a scheduled weekly check flags untagged resources and anything sitting idle, routed to a Slack channel the team actually reads. For dev and staging specifically, I usually just automate a shutdown schedule outright, since there's rarely a reason a dev box needs to run at 2 AM on a Saturday.

</details>

---

### Q: Data transfer costs are unexpectedly high this month. How do you diagnose and reduce them?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Data transfer costs hide well, since they're spread across small charges instead of one obvious resource, so I go looking for them specifically in Cost Explorer.

First I check if it's cross-AZ, cross-region, or internet egress, since each has a different fix. Cross-AZ between chatty services, like an app and a database in different zones making thousands of small calls, adds up fast and is invisible until you filter for it specifically.

For cross-region traffic, I check VPC Flow Logs for what's actually generating it — I've found a service calling another service's public endpoint across regions purely because private connectivity was never set up. For internet egress, I check if a CDN could absorb some of it instead, since that's often both cheaper and faster for users.

</details>

---

### Q: How do you decide whether moving workloads to Spot Instances or Graviton (ARM) is actually worth the operational risk?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I evaluate these separately, since they carry very different risk even though both get pitched as cost optimization.

For Spot, the real question is whether the workload can handle an instance disappearing with two minutes of warning. Stateless services behind a load balancer are a clean fit. Anything stateful, or a long job that can't checkpoint its progress, is a bad fit unless I'm willing to build real interruption handling for it.

For Graviton, the risk is mostly compatibility — a different CPU architecture, so compiled dependencies and base images need actual testing, not an assumption. I'd test under real load in staging before trusting it in production, since something can pass CI cleanly and still behave differently at scale. I'd never move a stateful, business-critical workload to Spot just for the discount — that's a decision that looks great on a cost report and terrible on an incident postmortem.

</details>

---

### Q: How do you show engineering teams the actual cost of the services they own, so cost becomes part of their normal decision-making?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If cost only lives on a dashboard the platform team checks, engineering teams have no reason to think about it, so the goal is getting it in front of the people actually making the decisions that drive it.

Tagging is the foundation, enforced through the same infrastructure code teams already use, not a policy document, since untagged spend just becomes invisible spend that never gets optimized.

Once tagging's solid, I build cost reports by team and deliver them where the team actually looks — I've had much better results posting a weekly cost summary straight into a team's own Slack channel than pointing them at a company-wide dashboard. Where I can, I also surface a rough cost estimate right on the pull request for new infrastructure, since that's the moment an engineer can still change course cheaply.

</details>

---
