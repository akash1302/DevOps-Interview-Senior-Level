# Senior DevOps Interview Questions: AWS Scenario-Based

### Q: You are tasked with designing a scalable web application on AWS to handle fluctuating traffic. What services and architecture would you use to ensure both availability and cost-efficiency?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I'd start with a multi-AZ architecture, not a single big server.

For the application layer, an **ALB** in public subnets, running the app in **ECS Fargate** in private subnets across at least two AZs, so a single AZ failure doesn't take the app down. For the database, **RDS Multi-AZ**, with **ElastiCache** in front of it for anything read-heavy so the database isn't taking the full hit on every request.

Autoscaling is based on CPU or request count — more tasks come up during a spike and scale back down once traffic drops, so I'm not paying for peak capacity all day. Static content goes through **S3 and CloudFront** so it never even hits the app servers.

**Simple flow:** User → Route 53 → CloudFront → ALB → ECS → RDS (+ ElastiCache)

**Key point:** I don't just throw bigger servers at it, I make the app horizontally scalable so it absorbs spikes without keeping expensive idle capacity running the rest of the time.

</details>

---

### Q: Your application needs to process large volumes of data and convert it into various formats for downstream services. What AWS services would you use to automate and scale this process?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I land the raw files in **S3** first, since that gives durable storage and a natural trigger point.

An S3 event triggers processing. For light, fast conversions, that's a **Lambda** function. For heavier jobs — large files, longer processing time — I'd use **AWS Batch** instead, since it isn't time-limited the way Lambda is. If the pipeline has multiple steps, like validate, convert, then notify, I'd use **Step Functions** to orchestrate it properly instead of chaining Lambdas together by hand.

Converted output goes to a second S3 bucket, with **SQS** notifying whatever needs to pick it up, so a slow downstream consumer doesn't block the pipeline.

**Key point:** I split by workload size — Lambda for fast and small, Batch for heavy and long-running — instead of forcing everything through one tool.

</details>

---

### Q: Your organization has decided to use AWS for disaster recovery but wants to minimize costs. What disaster recovery architecture would you recommend?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

This really depends on the RTO and RPO the business is willing to give me — I never design a DR plan without those two numbers first.

If the business can tolerate a few hours of downtime, I'd go with a pilot light setup — the database replicates continuously to the second region, but the compute layer stays switched off until an actual disaster, then gets spun up from a template. That keeps ongoing cost low.

If they need faster recovery, a warm standby runs a smaller version of the full stack at all times, ready to scale up on failover — costs more, but recovers in minutes instead of from scratch.

**Key point:** Cost and recovery time are a direct trade-off, so I match the architecture to the real RTO/RPO instead of defaulting to the most expensive option.

</details>

---

### Q: You need to ensure secure and compliant data transfer between your on-premises data center and AWS. What would you recommend?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For anything beyond occasional small transfers, I'd recommend **AWS Direct Connect** — a dedicated physical link that bypasses the public internet entirely, giving consistent bandwidth and lower latency.

Direct Connect alone isn't encrypted by default though, and a lot of compliance requirements specifically ask for encryption in transit, so I'd run a VPN over it as well.

If Direct Connect isn't justified yet, like for lower volume or while it's still being provisioned, a **Site-to-Site VPN** with IPsec works fine as a primary path or a fallback, since it's encrypted by default and quick to stand up.

**Key point:** Direct Connect for the dedicated path, a VPN layered on top for encryption, and Site-to-Site VPN alone as a faster fallback for lower-volume needs.

</details>

---

### Q: You are managing a microservices-based application that is experiencing latency due to high database load. How would you optimize the architecture?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't jump straight to a bigger database instance, I check what's actually generating the load first, since scaling compute doesn't fix a bad query or a missing index.

If it's read-heavy, I add **ElastiCache** for frequently read data, and **RDS read replicas** for queries that don't need the absolute latest write, like reporting.

I also check connection handling, since a lot of microservices setups hit connection exhaustion because each instance holds its own pool. I'd put a connection pooler in front of the database so hundreds of app connections share a much smaller set of real database connections. For writes that don't need to happen synchronously, like sending a notification, I move those to a queue so the request returns fast.

**Key point:** I fix the actual bottleneck — reads, connections, or synchronous writes — instead of reflexively resizing the database.

</details>

---

### Q: You have a critical application that cannot tolerate any downtime. How would you architect this solution on AWS to achieve high availability?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Multi-AZ is the non-negotiable baseline — app servers across at least two AZs behind an ALB, and RDS Multi-AZ for the database.

If the tolerance is truly zero, I'd go further to multi-region — the app deployed in two regions, database replicating across regions, and Route 53 doing health-check-based failover so traffic shifts automatically if an entire region has a problem.

Deployments themselves also can't introduce downtime, so I use rolling or blue-green deployments, never a straight swap. And I make sure health checks are genuinely meaningful — checking the app can actually reach its dependencies, not just that the process is running.

**Key point:** True zero-downtime is a chain — multi-AZ or multi-region infrastructure, automated failover, and deployments that never drop capacity.

</details>

---

### Q: Your application requires secure storage of sensitive customer data, with auditing and monitoring of all access attempts. What would you do?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Data at rest gets encrypted with a customer-managed KMS key, not the default one, so I control rotation and can see exactly who used it in CloudTrail.

Access is locked down with a tight bucket policy and IAM policies scoped to exactly what each role needs, and I add a condition that denies any request not coming over TLS.

For auditing specifically, I turn on CloudTrail data events for that bucket, since management events alone don't cover object-level read and write access — that's the actual audit trail showing who read a specific object and when.

**Key point:** Encryption alone isn't the control, the audit trail is the real requirement here, so CloudTrail data events on that bucket are the piece I never skip.

</details>

---

### Q: Your team needs a continuous integration/continuous delivery (CI/CD) pipeline to automatically deploy applications in AWS. How would you set it up?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

Source triggers a build, the build produces an artifact, and the artifact gets deployed — that's the shape regardless of which tools are involved.

Source is Git-based, triggering on a merge to the target branch. The build step installs, tests, builds the image, and pushes it to a registry — I always run security scans and tests here, before anything gets near deployment, so a bad build fails fast and cheap.

Deployment goes through a rolling or blue-green rollout, and production is always gated behind manual approval, never auto-deployed straight from a merge. The pipeline authenticates using short-lived, tightly scoped credentials, not a long-lived access key stored in CI.

**Key point:** Production is always gated behind human approval, and the pipeline never holds a long-lived AWS credential.

</details>

---

### Q: Your web application experiences a large number of requests from a particular geographical region causing latency. How would you improve performance for users in that region?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

First I confirm it's actually distance-driven latency and not a backend bottleneck, by checking metrics broken down by region.

If it is distance, **CloudFront** in front of the app is the first move — its edge locations cache content close to users, so a lot of requests never reach the origin region at all.

For requests that genuinely need to hit the backend, I'd consider deploying a regional stack near that geography if traffic volume justifies it, with latency-based routing sending users to whichever region is fastest for them.

**Key point:** A CDN first for anything cacheable, since it's the cheapest and fastest fix — a full regional deployment is the next step up only if it's actually justified.

</details>

---

### Q: You need to implement fine-grained access control for a growing number of users accessing an AWS S3 bucket. How would you design this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I don't manage this with individual IAM users, that doesn't scale past a handful of people. I use IAM roles tied to groups, so access is managed by group membership, not per-person policy edits.

For real fine-grained control, like different teams needing access to different parts of the same bucket, I use **S3 Access Points** scoped per team, each with its own policy, rather than one giant bucket policy trying to express every team's rules.

Policies are scoped to the specific action and path a role actually needs, nothing broader, and I review access on a schedule, since permissions pile up over time as people change roles.

**Key point:** Access points plus tightly scoped IAM policies, not one broad bucket policy trying to cover every team's access pattern.

</details>

---

### Q: Your application is built on containers, and you need a solution that allows automatic scaling of containers without managing underlying servers. What AWS service would you recommend?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

**ECS with Fargate** is what I'd default to — containers run without me provisioning or patching any EC2 instances underneath at all.

Autoscaling is based on CPU, memory, or a custom metric like request count, and Fargate just launches more tasks to match, without me managing node capacity myself.

If the team's already committed to Kubernetes specifically, EKS with Fargate profiles gives the same benefit, though it does come with some real constraints, like no DaemonSets. For most teams not already deep into Kubernetes, I'd steer toward plain ECS Fargate, since it's simpler to operate day to day.

</details>

---

### Q: You need to archive data that is rarely accessed but must be retained for compliance reasons. How would you store this data in AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

S3 Glacier or Glacier Deep Archive, depending on how rarely it's accessed and how fast retrieval needs to be.

I don't move data there manually, I set up an S3 lifecycle policy so objects transition automatically after a defined age, so nobody has to remember to do it, and it happens consistently.

For compliance specifically, I'd also enable S3 Object Lock in compliance mode, so data genuinely can't be deleted or overwritten before the retention period ends, by anyone — that's usually the actual requirement compliance is asking for, not just cheap storage.

</details>

---

### Q: You need to optimize costs for a fleet of EC2 instances that run non-critical workloads during business hours. How can you achieve cost savings without sacrificing performance?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

If these only need to run during business hours, the biggest single win is not running them the rest of the time — a scheduled stop and start, so instances shut down every evening and weekend and start back up before the business day.

That alone is often a 60 to 70% reduction on that fleet's compute cost, and it doesn't touch performance during the hours anyone's actually using them.

On top of the schedule, I'd right-size the instances, checking real usage against what's provisioned, since non-critical workloads are often over-provisioned from an initial guess nobody revisited.

**Key point:** Scheduled shutdown outside business hours is the biggest lever by far, right-sizing is the next layer on top of that.

</details>

---

### Q: Your application handles sensitive data, and you need to ensure encryption of data in transit and at rest. How would you architect this solution on AWS?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

For data at rest, I enable KMS encryption across every storage layer that touches the data, using a customer-managed key so I control rotation and can see key usage in CloudTrail.

For data in transit, TLS is enforced end to end — a certificate on the ALB, with any plain HTTP request redirected to HTTPS rather than accepted. I also make sure this applies internally between services, not just at the public edge, since that's a common gap.

To make sure this isn't just a one-time setup, I enforce it structurally, like a bucket policy that denies any request not made over TLS, so it's not dependent on every engineer remembering to tick the box.

</details>

---

### Q: You need to migrate a large database from your on-premises environment to AWS with minimal downtime. How would you achieve this?

<details>
<summary><b>🔍 View Candidate's Answer</b></summary>

I'd use **AWS DMS** with continuous replication, not a one-shot export and import, since a straight dump-and-restore on a large database means real downtime for however long that transfer takes.

DMS does an initial full load into the target database, then switches to continuously replicating new changes from the source, so the target stays in sync while the source keeps running normally.

Cutover happens in a short, planned window — stop writes to the source, let DMS catch up the last few changes, point the app at the new database, and resume traffic. I always test the cutover against a staging copy first and check data consistency before the real cutover, not after.

**Key point:** Continuous replication is what turns a normally multi-hour migration into a cutover window measured in minutes.

</details>

---
